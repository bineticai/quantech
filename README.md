# Quantech Drive

**Quantech Drive** is a cross-platform encrypted virtual drive that lets you create secure, encrypted storage on your existing hardware. Store up to 5TB of encrypted data with military-grade encryption.

## Features

- **AES-256-GCM Encryption**: Military-grade encryption for all data
- **Argon2id Key Derivation**: Secure password-based key derivation
- **BIP39 Recovery**: 24-word mnemonic for account recovery
- **Cross-Platform**: Linux and macOS support (Windows coming soon)
- **FUSE Integration**: Seamless filesystem mounting

## Installation

```bash
go install github.com/binetic-ai/quantech-drive/cmd/quantech@latest
```

## Quick Start

### Initialize a new drive

```bash
quantech init
```

### Mount the drive

```bash
quantech mount /path/to/mount/point
```

### Unmount the drive

```bash
quantech umount /path/to/mount/point
```

### Check drive status

```bash
quantech status
```

## Commands

- `init` - Create and initialize a new encrypted drive
- `mount` - Mount an encrypted drive to a filesystem location
- `umount` - Unmount an encrypted drive
- `status` - Display drive status and statistics

## Security

Quantech Drive uses:
- **AES-256-GCM** for authenticated encryption
- **Argon2id** for key derivation from passwords
- **BIP39 mnemonics** for secure recovery phrases

## License

Proprietary. All rights reserved. (c) 2026 Binetic.ai

## Support

For issues and feature requests, visit the project repository.
