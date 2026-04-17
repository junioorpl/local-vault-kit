# local-vault-kit

Encrypted local folders on macOS without the VeraCrypt/macFUSE pain.

A thin bash wrapper around **APFS encrypted sparse bundles** + **macOS Keychain** + **launchd**, plus **KeePassXC** for password management. Designed to coexist with FileVault, Obsidian vaults, AI agents (Claude Code, Cursor), and incremental backups.

> **Threat model:** "I want laptop-loss / cold-boot / casual-snoop protection on a few sensitive folders without giving up day-to-day convenience or breaking my tools." See [`docs/threat-model.md`](docs/threat-model.md).

## Why not VeraCrypt?

| Concern | VeraCrypt | This kit (APFS sparse bundle) |
|---|---|---|
| Kernel extension on Apple Silicon | needs macFUSE/FUSE-T | none — native APFS |
| Backup / cloud sync | monolithic blob, full re-sync on any change | sparse bundle = ~8 MB bands, incremental |
| macOS Keychain auto-unlock | no | yes |
| Mount as native disk | yes | yes (`/Volumes/<name>`) |
| Cross-platform | yes (Linux/Win/Mac) | macOS only |
| Maintenance burden | upstream forks, signing issues | zero — Apple ships `hdiutil` |

If you need cross-platform → use [Cryptomator](https://cryptomator.org). If macOS-only → this kit is simpler.

## Install

```bash
git clone https://github.com/<you>/local-vault-kit.git
ln -s "$PWD/local-vault-kit/bin/vault" /opt/homebrew/bin/vault   # or any dir on $PATH
brew install --cask keepassxc                                    # for password management
```

## Quickstart

```bash
# 1. create encrypted bundle (you set the passphrase; it's stored in Keychain)
vault init my-secrets --size 200m

# 2. move a sensitive folder into it (leaves a symlink in original location)
vault migrate my-secrets ~/work/secrets
vault migrate my-secrets ~/work/credentials

# 3. auto-mount on every login
vault autounlock my-secrets --enable

# done. open KeePassXC, point its database at /Volumes/my-secrets/keepass.kdbx
```

## Commands

| Command | Description |
|---|---|
| `vault init <name> [--size SIZE]` | Create encrypted sparse bundle, stash passphrase in Keychain, mount it |
| `vault mount <name>` | Mount (auto-unlocks via Keychain) |
| `vault unmount <name>` | Eject |
| `vault status [name]` | Show mount state of one or all bundles |
| `vault migrate <name> <src-dir>` | Move folder into vault, replace source with symlink, leave timestamped backup |
| `vault autounlock <name> --enable\|--disable` | Toggle login-time auto-mount LaunchAgent |
| `vault list` | List bundles in `$VAULT_HOME` |
| `vault destroy <name>` | Wipe bundle + Keychain entry + LaunchAgent (asks for confirmation) |

## How it works

1. `hdiutil create -encryption AES-256 -type SPARSEBUNDLE` → encrypted disk image that grows on demand. Stored in `~/Vaults/<name>.sparsebundle` by default.
2. Passphrase saved to login Keychain via `security add-generic-password`. Only `hdiutil` and `security` are ACL'd to read it.
3. `vault mount` retrieves the passphrase and pipes it to `hdiutil attach -stdinpass`. The volume mounts at `/Volumes/<name>`.
4. `vault migrate` `rsync`s the source folder into the vault, renames original to `<src>.pre-vault-<timestamp>`, and creates a symlink at the original path. **Tools that read the original path keep working** when the vault is mounted, and **fail closed** (broken symlink) when unmounted.
5. `vault autounlock --enable` writes a LaunchAgent `~/Library/LaunchAgents/com.local-vault-kit.<name>.plist` that runs `vault mount <name>` at login.

## Why the symlink-back pattern?

You want sensitive folders encrypted **and** addressable by their original paths so existing tools (Obsidian, Claude Code, scripts, IDEs) keep working. The migrate flow makes `~/work/secrets/` a symlink to `/Volumes/secret-vault/secrets/`. When the vault is locked, the symlink dangles → fail-closed.

## KeePassXC integration

Place your `*.kdbx` database **inside** a vault:

```bash
vault init my-secrets --size 100m
# in KeePassXC: Database > New > save to /Volumes/my-secrets/keepass.kdbx
```

Now KeePassXC autofill works (foreground convenience) AND the database file lives on encrypted media (defense-in-depth on top of FileVault).

## Caveats

- **macOS only.** Linux/Windows users: use Cryptomator or LUKS/BitLocker.
- **FileVault is still your primary line of defense.** This kit adds a second factor for individual folders. If FileVault is off, turn it on first.
- **Keychain auto-unlock = "if you can log in, you can read the vault."** That's what most people want. If you need post-login challenge (passphrase on every mount), don't use `autounlock`.
- **Don't put a vault inside iCloud Drive's `Mobile Documents` path** — iCloud doesn't handle sparse bundles cleanly. For sync, point Syncthing or `rsync` at the bundle directory directly.
- **Time Machine** handles sparse bundles fine (band-level diffs).
- **No multi-user support** — Keychain entries are per-user.

## Troubleshooting

```bash
# bundle won't mount?
hdiutil attach -verbose ~/Vaults/<name>.sparsebundle

# auto-mount not firing?
tail -f /tmp/local-vault-kit.<name>.log /tmp/local-vault-kit.<name>.err
launchctl list | grep local-vault-kit

# forgot passphrase?
# you didn't, you saved it in Keychain. open Keychain Access, search for "local-vault-kit:".
```

## License

MIT — see [`LICENSE`](LICENSE).
