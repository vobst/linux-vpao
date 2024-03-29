# linux-vpao

Arch Linux package for building the mainline kernel with my configuration.

Based on the [linux package](https://gitlab.archlinux.org/archlinux/packaging/packages/linux).

## Usage

### Module Signatues

This kernel enforces module signatures, i.e., `CONFIG_MODULE_SIG_FORCE=y`. Thus, you need to sign your out-of-tree modules and bake the corresponding pubkey into the kernel. Do this by setting `CONFIG_SYSTEM_TRUSTED_KEYS=/path/to/cert.pem`.

#### DKMS

If you use dkms for managing out-of-tree modules, edit `/etc/dkms/framework.conf` and set `mok_signing_key=/path/to/cert.key` and `mok_certificate=/path/to/cert.pub`.

### UKI

It is trendy to use UKIs. These steps assume that you use `mkinitcpio` to build the initrd.

These notes are based on the following articles:
- [Unified Extensible Firmware Interface/Secure Boot](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot)
- [Unified kernel image](https://wiki.archlinux.org/title/Unified_kernel_image)

#### Kernel Command Line

Create files `/etc/cmdline.d/*.conf` with all of the kernel command line parameters you would like to bake into the UKI.

#### initrd Generation

Instruct `mkinitcpio` to generate the UKI. Edit `/etc/mkinitcpio.d/linux-vpao.preset`:

- change `ALL_kver="/root/.local/boot/vmlinuz-linux-vpao"` (we don't want a bzImage in /boot)
- comment out `PRESET_image` (we don't want the stand alone initrd in /boot)
- uncomment and set `PRESET_uki=/boot/EFI/Linux/arch-linux-vpao-PRESET.efi`

Pacman will automatically run mkinitcpio when the kernel package is changed.

#### microcode

The CPU microcode updates are baked into the UKI and applied on boot. Thus, we need to regenerate the UKI when the microcode gets updated. Create `/etc/pacman.d/hooks/ucode.hook` with the following contents:

```console
cat << EOF > /etc/pacman.d/hooks/ucode.hook
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=intel-ucode
Target=linux-vpao

[Action]
Description=Update Microcode module in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case $trg in linux-vpao) exit 0; esac; done; /usr/bin/mkinitcpio -P'
EOF
```

By default, the microcode is stored in the `/boot` partition, i.e., it is not protected from tampering, and its integrity is not verified when building a new UKI. To change the install location:

- remove the Arch package `pacman -Rsu intel-ucode`
- get the PKGBUILD form [Gitlab](https://gitlab.archlinux.org/archlinux/packaging/packages/intel-ucode/-/tree/main?ref_type=heads)
- change the install command, e.g., to `/root/.local/boot/`, which is also where we put the vmlinuz images
- change the mkinitcpio preset, `ALL_microcode=(/root/.local/boot/*-ucode.img)`
- build and install the package
- (automatically) check for updates from time to time, especially when some new CPU bug came out...

#### Signature

The UKI will be signed by sbctl.

```console
cat << EOF > /etc/initcpio/post/uki-sbctl
#!/usr/bin/env bash
sbctl sign-all -g
EOF
chmod +x /etc/initcpio/post/uki-sbctl
```

Make sure to sign them manually once such that they get added to the list of files that will be signed by `sign-all`.

```console
sudo sbctl sign -s /boot/EFI/Linux/arch-linux-vpao.efi
sudo sbctl sign -s /boot/EFI/Linux/arch-linux-vpao-fallback.efi
```

#### Grub

If you really want to use Grub for chainloading the UKI, add the following menu entries to `/etc/grub.d/40_custom`:

```console
echo 'if [ ${grub_platform} == "efi" ]; then
  menuentry "Arch Linux VPAO UKI" {
	insmod fat
	insmod chain
	search --no-floppy --set=root --fs-uuid 1204-FAF1
	chainloader /EFI/Linux/arch-linux-vpao.efi
  }
  menuentry "Arch Linux VPAO UKI [fallback]" {
	insmod fat
	insmod chain
	search --no-floppy --set=root --fs-uuid 1204-FAF1
	chainloader /EFI/Linux/arch-linux-vpao-fallback.efi
  }
fi' >> /etc/grub.d/40_custom
```

To select the UKI by default, edit `/etc/default/grub` and set `GRUB_DEFAULT="Arch Linux VPAO UKI"`.

#### Efi Boot

Use `efibootmgr` to create the boot entries:

```console
sudo efibootmgr --create --disk /dev/sda --part 1 --label linux-vpao-uki -l '\EFI\Linux\arch-linux-vpao.efi' --unicode
sudo efibootmgr --create --disk /dev/sda --part 1 --label linux-vpao-uki-fallback -l '\EFI\Linux\arch-linux-vpao-fallback.efi' --unicode
```

Then you maybe want to change the boot order

```console
sudo efibootmgr -o 000X,000Y,...
```

and remove some unneded boot entries (like Grub):

```console
sudo efibootmgr -b 000Z -B
```

When you are sure that everything is stable, remove the bootloader and all non-UKI kernels as well as their initramfss from the `/boot` partition.

#### fwupd

I hope that you are using fwupd to keep your firmware up to date. To use it with secure boot enabled sign the EFI application

```console
sbctl sign -s /usr/lib/fwupd/efi/fwupdx64.efi -o /usr/lib/fwupd/efi/fwupdx64.efi.signed
```

and tell fwupd to use it (append to `/etc/fwupd/fwupd.conf`)

```console
[uefi_capsule]
DisableShimForSecureBoot=true
```

also add a pacman hook to resign the updater when needed

```console
cat << EOF > /etc/pacman.d/hooks/sign-fwupd-secureboot.hook
[Trigger]
Operation = Install
Operation = Upgrade
Type = Path
Target = usr/lib/fwupd/efi/fwupdx64.efi

[Action]
When = PostTransaction
Exec = /usr/bin/sbctl sign /usr/lib/fwupd/efi/fwupdx64.efi -o /usr/lib/fwupd/efi/fwupdx64.efi.signed
Depends = sbctl
EOF
```

#### Intel ME

Do you like opaque binary blobs that run on a permanently powered coprocessor
and are said to have almost god-like control over your system? Then the Intel
ME is just your. There isn't much that you can do about it, but we at least
disable the OS support for IME features. If you can't control the modules
included in your distribution you can at least blacklist them:

```console
cat << EOF > /etc/modprobe.d/mei.conf
blacklist mei
blacklist mei_hdcp
blacklist mei_me
blacklist mei_pxp
blacklist mei_wdt
EOF
```
