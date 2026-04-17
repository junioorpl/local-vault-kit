# Threat model

## What this protects against

- **Lost / stolen laptop, powered off.** With FileVault on, full disk is encrypted. Sparse bundle adds nothing here, but doesn't hurt.
- **Lost / stolen laptop, powered on but locked.** FileVault still applies. If your `unmount` runs on lock (optional), bundle adds defense in depth.
- **Casual snooping when laptop is unlocked but you walk away.** Run `vault unmount <name>` in a hotkey or shortcut → folder disappears for the snooper, FileVault doesn't help here.
- **Malware that scans `$HOME` for files matching `*credential*` / `*.kdbx`.** When the vault is unmounted, the symlink dangles and the kdbx isn't on disk in plaintext form anywhere. This kills opportunistic malware that doesn't specifically target sparse bundles.
- **Backup leakage.** Cloud backup of `~/Vaults/*.sparsebundle` only ever exposes encrypted bands.

## What this does NOT protect against

- **Targeted attacker with code execution as your user while the vault is mounted.** They can read the mounted volume just like any other folder. Use `vault unmount` when not actively using secrets.
- **Keylogger / screen recorder.** Out of scope.
- **Physical evil-maid attacks** (firmware, bootloader). Use Apple's Secure Boot defaults; don't disable SIP.
- **Forgotten passphrase.** There is no recovery. The Keychain entry is the only copy. Back up Keychain (Time Machine includes it) or write the passphrase down once and store offline.
- **Nation-state forensics on RAM dumps while mounted.** Out of scope; consider `vault unmount` before sleep.

## Trust assumptions

- macOS Keychain is trusted to store the passphrase. ACL restricts read to `hdiutil` and `security` binaries.
- `hdiutil`'s AES-256 implementation is trusted (Apple ships it; FileVault uses the same primitives).
- The `vault` script is trusted. It runs as your user, never escalates.

## Recommended hardening on top

- FileVault: **on** (System Settings → Privacy & Security → FileVault).
- Firmware password: optional, breaks easy boot-disk swaps.
- `vault unmount` on screen lock: bind to a Shortcut + Stream Deck / hotkey.
- KeePassXC: enable "lock database after X minutes idle" + "clear clipboard after 10s".
- KeePassXC: add a YubiKey HMAC-SHA1 challenge-response factor when you get one.
- `vault destroy` if a vault was ever exposed in plaintext (e.g. during migration debugging).
