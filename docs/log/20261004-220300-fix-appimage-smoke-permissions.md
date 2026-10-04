# Fix Linux AppImage Smoke: AppRun EACCES for non-root user

Status: in progress (fix committed on `fix/appimage-smoke-permissions`, CI run pending)

## Why
`.github/workflows/linux-appimage-smoke.yml` (scheduled) failed every day from
at least 2026-09-27 to 2026-10-04 (8 consecutive runs). Run 37212431646:

```
env: 'squashfs-root/AppRun': Permission denied
AppRun failed with status 126
```

Also failing on upstream (`gh run list --repo mediar-ai/screenpipe --workflow linux-appimage-smoke.yml`), so not fork-specific.

## Pre-flight findings
```
gh run view 37212431646 --repo pkhanpara/screenpipe          # only "Smoke AppImage on Debian 13" red; build+repack green
gh run download 37212431646 -n linux-appimage-smoke
unsquashfs -o $(./x.AppImage --appimage-offset) -ll x.AppImage
drwx------ root/root  squashfs-root
-rwxr-xr-x root/root  squashfs-root/AppRun
-rwxrwx--x root/root  squashfs-root/AppRun.wrapped
drwx------ root/root  squashfs-root/usr
drwx------ root/root  squashfs-root/usr/bin
```
Directories are stored 0700 root in the shipped image. The smoke extracts as
root, then runs `su smoke-user -c ... squashfs-root/AppRun` -> cannot traverse.
Everything earlier in the step runs as root, hence only the last command fails.

Repro in `debian:trixie` (root-owned copy, `su u -c "env sr/AppRun"`):
before: `Permission denied`, rc=126; after `chmod -R u+rwX,go+rX,go-w`: AppRun
executes (then stops on missing libpulse, which CI installs).

## Design
Normalize modes in the repack step right before `appimagetool`, so the shipped
artifact is fixed, not just the test.
Rejected: chmod only inside the smoke container (hides a real shipped-artifact
defect); run the smoke as root (loses the non-root coverage that caught this).

## What was done
- Added `chmod -R u+rwX,go+rX,go-w squashfs-root` before `appimagetool` in
  `linux-appimage-smoke.yml`.

## Still to do
- `release-app.yml` (~l.957-977) and `release-enterprise.yml` (~l.729-748) use
  the same repack pattern and likely ship 0700 dirs too. Left untouched: release
  workflows, see `docs/human-only-app-publication.md`. Needs a deliberate change.
- Arch smoke step has been skipped since the Debian step started failing; it may
  surface its own failure once this passes.

## Gotchas
- My local host umask is 002, but squashfs stores explicit modes, so extraction
  preserved 0700. A bind-mounted `chmod` inside docker modifies the host tree:
  re-extract before re-running a "before" repro.
