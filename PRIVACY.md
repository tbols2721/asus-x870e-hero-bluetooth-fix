# Privacy and diagnostic reports

Keep diagnostic reports limited to the information needed to reproduce a
problem. Hardware model names, USB vendor/product IDs, kernel versions,
software revisions, and firmware checksums identify a configuration without
requiring a device's unique Bluetooth address.

## Information to keep local

Do not publish Bluetooth addresses, device serial numbers, personalized device
names, usernames, hostnames, absolute home-directory paths, credentials, pairing
keys, or microphone recordings. Replace identifying fields consistently with
labels such as `<CONTROLLER_ADDRESS>`, `<DEVICE_ADDRESS>`, `<USER>`, and `<HOST>`
when a small diagnostic excerpt is necessary.

BlueZ pairing state under `/var/lib/bluetooth` contains keys and unique device
identifiers. Do not upload or commit that directory. Inspect screenshots as
well as text: terminal prompts, Bluetooth menus, and desktop notifications can
reveal personal information.

Raw `bluetoothctl`, `btmon`, `journalctl`, PulseAudio, and installer output can
contain private fields. Git ignore rules do not filter terminal output, issue
comments, release assets, or files deliberately added with `git add -f`.

## Preparing a useful report

Include the hardware model, USB product identity, distribution, running kernel,
relevant software versions, exact command or control that failed, and observed
behavior. State whether initialization, pairing, playback, or microphone input
is affected. Prefer a short redacted excerpt to a complete log archive.

`./scripts/check-system` reports board model, kernel, module path and
srcversion, firmware integrity, USB product identity, and controller health.
It omits addresses, device names, hostnames, and home-directory paths. Review
its output before sharing, particularly if you have modified the script.
The installation script's full output is not redacted.

## Before a contribution

Review both staged content and commit identity:

```bash
git diff --cached --check
git diff --cached
git status --short
git var GIT_AUTHOR_IDENT
git var GIT_COMMITTER_IDENT
```

Use a GitHub noreply address if you do not want a personal email in public
commit metadata. Ignored `local/` and `build/` directories, downloads, compiled
modules, logs, keys, and common capture formats should remain untracked.
Deleting a file in a later commit does not remove it from earlier Git history.
