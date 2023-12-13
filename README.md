# linux-vpao

Arch Linux package for building the mainline kernel with my configuration.

Based on https://gitlab.archlinux.org/archlinux/packaging/packages/linux

## Usage

### Module Signatues

This kernel enforces module signatures. Thus, you need to sign out-of-tree modules and bake the pubkey into the kernel by setting `CONFIG_SYSTEM_TRUSTED_KEYS` to a pem-encoded certificate.

#### DKMS

Set `mok_signing_key` and `mok_certificate` to point to the cryptographic material that you want to use for signing.
