<div align="center">
  <img src="assets/hostrail.svg" width="112" alt="Hostrail icon">

# Hostrail

**Your infrastructure. Your terminal. Your Vault.**

A local-first SSH, SFTP, terminal, and port-forwarding workspace for macOS,
Windows, Linux, and Android.

[![Latest release](https://img.shields.io/github/v/release/Raterfy/hostrail-releases?style=flat-square&label=latest)](https://github.com/Raterfy/hostrail-releases/releases/latest)
[![Platforms](https://img.shields.io/badge/platforms-macOS%20%7C%20Windows%20%7C%20Linux%20%7C%20Android-2e7cf6?style=flat-square)](#download-hostrail)
[![Local first](https://img.shields.io/badge/data-local--first-35c48d?style=flat-square)](#privacy-and-security)
[![No account](https://img.shields.io/badge/account-not%20required-38d1be?style=flat-square)](#privacy-and-security)

[**Download the latest release**](https://github.com/Raterfy/hostrail-releases/releases/latest) ·
[Features](#what-hostrail-does) ·
[Security](#privacy-and-security) ·
[Install](#installation-notes)
</div>

---

## Download Hostrail

The current release is **Hostrail 1.6.0**.

| Platform | Download | Package |
|---|---|---|
| macOS | [**Universal app**](https://github.com/Raterfy/hostrail-releases/releases/download/v1.6.0/Hostrail-1.6.0-macos-universal.zip) | Apple Silicon + Intel |
| Windows | [**Windows x64**](https://github.com/Raterfy/hostrail-releases/releases/download/v1.6.0/Hostrail-1.6.0-windows-x64.zip) | Complete portable bundle |
| Linux | [**Linux x64**](https://github.com/Raterfy/hostrail-releases/releases/download/v1.6.0/Hostrail-1.6.0-linux-x64.tar.gz) | GTK desktop bundle |
| Android | [**arm64 APK**](https://github.com/Raterfy/hostrail-releases/releases/download/v1.6.0/Hostrail-1.6.0-android-arm64.apk) | Most recent phones and tablets |
| Android | [**armv7 APK**](https://github.com/Raterfy/hostrail-releases/releases/download/v1.6.0/Hostrail-1.6.0-android-armv7.apk) | Older 32-bit ARM devices |
| Android | [**x86_64 APK**](https://github.com/Raterfy/hostrail-releases/releases/download/v1.6.0/Hostrail-1.6.0-android-x86_64.apk) | Emulators and x86_64 devices |

Every release includes `SHA256SUMS.txt`. After downloading it beside an
artifact, verify the file before installation:

```bash
shasum -a 256 -c SHA256SUMS.txt --ignore-missing
```

## What Hostrail does

### SSH workspace

- Save and organize SSH hosts with groups, tags, search, and grid or list views.
- Open multiple live SSH and local-shell terminals in browser-style tabs.
- Split sessions left, right, top, or bottom into persistent workspaces.
- Use Focus mode, terminal search, command snippets, replay, and guarded
  broadcast input.
- Connect through jump-host chains with explicit host-key verification.

### SFTP and forwarding

- Browse local and remote files side by side.
- Upload and download files or complete folders with conflict handling,
  cancellation, retry, and partial-safe transfers.
- Create local, remote, and SOCKS5 port-forwarding rules with live counters.
- Manage remote files without leaving the active terminal workspace.

### Encrypted Vault

- Protect local data with AES-256-GCM and the operating system's secure storage.
- Optionally require a dedicated Vault passphrase at Hostrail startup.
- Generate or import SSH keys without storing plaintext credentials in host
  profiles.
- Export portable encrypted backups protected with Argon2id + AES-256-GCM.

### Self-hosted synchronization

Hostrail 1.6.0 can synchronize the encrypted Vault through an SSH/SFTP server
you already control. There is no Hostrail sync server to deploy and no hosted
control plane:

1. choose one of your saved SSH hosts;
2. choose the remote encrypted archive path;
3. enter a shared sync passphrase;
4. Hostrail downloads, authenticates, merges, re-encrypts, and atomically
   uploads the Vault archive.

Synchronization can run manually, at startup, and after debounced local Vault
changes. Automatic sync protects the sync passphrase in the operating system's
secure storage and uses an already-saved SSH credential without opening a
background prompt. The passphrase is never sent to the VPS.

An expiring atomic SFTP lock prevents simultaneous devices from overwriting the
same archive. Active locks return a stable busy status, while abandoned locks
expire after 15 minutes.

## Docker SFTP setup for Hostrail sync

Docker can host a dedicated **SFTP endpoint** for the encrypted Vault archive. It
does not run Hostrail itself and does not provide a browser-accessible Hostrail
web interface. Install the native Hostrail application on every device that
needs access.

### 1. Prepare the VPS

Install Docker Engine and the Compose plugin, then create persistent directories:

```bash
mkdir -p ~/hostrail-sftp/{keys,storage}
cd ~/hostrail-sftp
sudo chown -R 1001:1001 storage
chmod 700 keys
```

### 2. Generate a key on a trusted client

Run this on your computer, not on the VPS:

```bash
mkdir -p ~/.config/hostrail-sync
ssh-keygen -t ed25519 \
  -f ~/.config/hostrail-sync/id_ed25519 \
  -C hostrail-sync
```

Copy only the public key to the VPS:

```bash
scp ~/.config/hostrail-sync/id_ed25519.pub \
  your-admin-user@your-vps:~/hostrail-sftp/keys/hostrail.pub
ssh your-admin-user@your-vps \
  'chmod 600 ~/hostrail-sftp/keys/hostrail.pub'
```

Never place `id_ed25519` in the Docker directory or on the VPS. Import that
private key into Hostrail's protected Keychain on each trusted device.

### 3. Create `compose.yml`

Save the following as `~/hostrail-sftp/compose.yml`:

```yaml
services:
  hostrail-sftp:
    image: atmoz/sftp:alpine
    container_name: hostrail-sftp
    restart: unless-stopped

    ports:
      - "2222:22"

    command: hostrail::1001:1001

    volumes:
      - ./storage:/home/hostrail/data
      - ./keys/hostrail.pub:/home/hostrail/.ssh/keys/hostrail.pub:ro

    security_opt:
      - no-new-privileges:true

    cap_drop:
      - ALL

    cap_add:
      - AUDIT_WRITE
      - CHOWN
      - DAC_OVERRIDE
      - FOWNER
      - KILL
      - NET_BIND_SERVICE
      - SETGID
      - SETUID
      - SYS_CHROOT
```

The `hostrail` user has no password. The encrypted archive persists in
`./storage`, independently from the container lifecycle.

### 4. Start and test SFTP

On the VPS:

```bash
cd ~/hostrail-sftp
docker compose up -d
docker compose ps
docker compose logs --tail=100 hostrail-sftp
```

From the trusted client:

```bash
sftp -P 2222 -i ~/.config/hostrail-sync/id_ed25519 \
  hostrail@your-vps.example.com
```

Run `ls data`, then exit with `quit`.

### 5. Configure Hostrail

Create a saved host in Hostrail with:

```text
Address: your-vps.example.com
Port: 2222
Username: hostrail
Credential: the imported client key
Remote path: data/vault-v1.enc
```

Connect once and verify the SSH host-key fingerprint. Then open
**Settings → Sync**, select the saved host, enter `data/vault-v1.enc`, and use
**Sync now**. Enter the same sync passphrase on every device, or enable
**Automatic sync** to protect it in the operating system secure store.

Hostrail sends only the Argon2id + AES-256-GCM encrypted archive. The SFTP
container never receives the sync passphrase or cleartext Vault. Automatic sync
uses only an SSH credential already stored in Hostrail.

### 6. Firewall or VPN

Prefer WireGuard, Tailscale, or another private network. If port `2222` must be
public, restrict it to trusted source addresses. Example with UFW:

```bash
sudo ufw allow from 203.0.113.10 to any port 2222 proto tcp
sudo ufw status
```

Replace the example address with the trusted client's public IP. With a VPN, bind
the Compose mapping to the VPS VPN address, for example
`100.64.0.10:2222:22`, instead of every interface.

### 7. Back up and restore

Create a backup on the VPS:

```bash
cd ~/hostrail-sftp
tar -czf "hostrail-sftp-$(date +%Y%m%d-%H%M%S).tar.gz" storage
```

Copy the backup away from the VPS. To restore it:

```bash
cd ~/hostrail-sftp
docker compose down
tar -xzf /path/to/hostrail-sftp-backup.tar.gz
sudo chown -R 1001:1001 storage
docker compose up -d
```

### 8. Update the container

```bash
cd ~/hostrail-sftp
docker compose pull
docker compose up -d
docker image prune -f
```

`docker compose down` keeps `./storage`. Do not delete that directory unless the
archive is backed up and permanent deletion is intended.

## Privacy and security

Hostrail is designed to work without surrendering control of your infrastructure:

- **no Hostrail account or subscription;**
- **no analytics, advertising SDK, or tracking identifier;**
- **no proprietary backend or Hostrail-operated cloud sync;**
- credentials remain protected by the native Keychain, Credential Manager,
  Secret Service, or Android Keystore;
- unknown and changed SSH host keys require explicit confirmation;
- activity logs do not store host addresses, passwords, or private keys;
- update checks read this repository's public GitHub Releases API at most once
  every 24 hours when enabled.

## Installation notes

### macOS

Extract the ZIP and move `Hostrail.app` to **Applications**. The public build is
universal (`arm64` + `x86_64`) but is not Apple-notarized. If Gatekeeper blocks
the first launch, approve Hostrail in **System Settings → Privacy & Security**.

### Windows

Extract the complete ZIP before launching `Hostrail.exe`. Keep the bundled DLLs
and `data/` directory beside the executable.

### Linux

Extract the archive and keep `hostrail`, `lib/`, and `data/` together. GTK 3 and
a working Secret Service/libsecret provider are required.

### Android

Download the APK matching the device architecture and allow installation from
the browser or file manager when Android asks. APKs are signed with Hostrail's
stable update certificate. Play Store distribution is not configured.

## Updates

Hostrail can notify you when this repository publishes a newer compatible
release. **Download update** opens the matching release asset in your browser;
installation remains manual, so Hostrail never silently replaces the installed
application.

## About this repository

`Raterfy/hostrail-releases` is Hostrail's public binary distribution and update
feed. It contains release notes, packaged applications, and checksums. It is not
the application source repository and never receives Vault data, credentials,
host inventories, or analytics events.
