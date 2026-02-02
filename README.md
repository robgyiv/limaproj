# limaproj (Lima bootstrap)

This setup keeps your source on the host and runs builds/tests inside a per-project Lima VM.
It mounts only the repo into the VM at `/workspace`.

## Prereqs

- Lima installed and working (`limactl` on PATH)
- Python 3 available on the host

## Install

Add the wrapper to your PATH (fish):

```fish
set -gx PATH $HOME/.lima/limaproj/bin $PATH
```

Reload:

```fish
source ~/.config/fish/config.fish
```

## Quick start (Apple silicon)

From inside your repo:

```bash
limaproj create proj-foo --policy untrusted \
  --image https://cloud.debian.org/cdimage/cloud/trixie/20260112-2355/debian-13-generic-arm64-20260112-2355.qcow2
```

Then:

```bash
limaproj start proj-foo
limaproj shell proj-foo
```

Inside the VM:

```bash
cd /workspace
```

## Policies

- **trusted**: repo mount + networking, installs build tools
- **untrusted**: repo mount + networking, minimal packages
- **airgap**: repo mount, no networking

Templates live in:

- `~/.lima/limaproj/templates/base.yaml`
- `~/.lima/limaproj/templates/trusted.yaml`
- `~/.lima/limaproj/templates/untrusted.yaml`
- `~/.lima/limaproj/templates/airgap.yaml`

## Notes

- The wrapper defaults to an amd64 Debian image; on Apple silicon, pass the arm64 image as above.
- You can stop and delete VMs with:

```bash
limaproj stop proj-foo
limaproj delete proj-foo
```
