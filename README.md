# cf

`cf` is a single bash script. It is meant for hosts where Cuttlefish can't (or shouldn't) run on bare metal, for example Fedora, where the Google host packages are Debian-only.

```console
$ cf mount ~/aosp
$ cf start
$ cf launch ~/aosp/android14-release/out/target/product/vsoc_x86_64_only
```

## The problem this solves

The usual way of running Cuttlefish in a container from google has a lot of boilerplate repeated commands so this is a wrapper for setup and convenience.

The usual way to run Cuttlefish in a container (the official `cuttlefish-orchestration` image) runs `launch_cvd` as root. Everything it touches in your mounted build tree becomes root-owned, so the next `m` fails until you `chown` a multi-gigabyte `out/` back.

`cf` keeps the container's init as root (it needs that to create tap devices and bridges) but runs `launch_cvd` as **your own uid/gid**, exactly like a native Debian install would.

## How it works

- **A derived image.** On first start, `cf` builds `cuttlefish-local` on top of the official image. It adds a user with your username, uid and gid, and puts that user in the groups `launch_cvd` needs: the image's own `kvm`, `cvdnetwork`, `render` and `video` groups (by name, because `launch_cvd` checks membership by name) *and* the groups that own the device nodes on your host (by numeric gid, because a privileged container keeps the host's gids).
- **The container stays root.** The image's entrypoint is untouched. Only `docker exec -u <you>` runs as you.
- **Same-path mount.** Your mount directory is mounted at the same absolute path inside the container, so every path means the same thing on both sides and nothing needs translating.
- **The container is kept between runs.** `cf stop` stops it, `cf start` restarts it instantly. Setup only happens when the container is created.
- **State inside the container.** Cuttlefish runtime files live in `/tmp/cvd-runtime-<product>` inside the container, so `cf remove` discards them.

## Prerequisites

### Required

| What | Check / install |
|---|---|
| Linux host with KVM | `ls -l /dev/kvm` exists. Enable VT-x / AMD-V in firmware if not. |
| `vhost_vsock` kernel module | `sudo modprobe vhost_vsock`. To load it at boot: `echo vhost_vsock \| sudo tee /etc/modules-load.d/vhost_vsock.conf` |
| Docker Engine, usable **without sudo** | `sudo usermod -aG docker $USER`, then log out and back in. Rootless Docker is not supported (the container needs `--privileged` and host networking). |
| bash 4.4+ and coreutils | Standard on current Fedora, Debian and Ubuntu (`realpath`, `readlink`, `stat`) |
| A built Cuttlefish AOSP tree | An `aosp_cf_*` target, with the images in `out/target/product/<product>` and the host tools in `out/host/linux-x86/bin/launch_cvd` |

`cf` refuses to run as root or via `sudo`; that is the whole point.

### Optional: NVIDIA GPU acceleration

GPU passthrough is **optional**. `cf` uses it automatically when it detects a working NVIDIA setup and otherwise starts the container without it. Without passthrough, Cuttlefish's own GPU auto-detection picks the rendering mode (typically software rendering, which is slower).

To enable it:

1. **NVIDIA driver on the host.** `nvidia-smi` must work.
2. **NVIDIA Container Toolkit.** On Fedora / RHEL:
   ```bash
   curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo \
     | sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
   sudo dnf install -y nvidia-container-toolkit
   ```
   For Debian / Ubuntu and other distros, follow NVIDIA's
   [installation guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html).
3. **Register the runtime with Docker:**
   ```bash
   sudo nvidia-ctk runtime configure --runtime=docker
   sudo systemctl restart docker
   ```
4. **Verify:**
   ```bash
   docker run --rm --gpus all ubuntu nvidia-smi
   ```
5. **Recreate the container** so it picks the GPU up: `cf rebuild`. `cf` prints a `GPU passthrough:` line telling you what it decided.

Detection (`CF_GPUS=auto`, the default) looks for `nvidia-ctk` or `nvidia-container-runtime-hook` on your `PATH` **and** `/dev/nvidiactl`. You can override it:

| `CF_GPUS` | Effect |
|---|---|
| `auto` (default) | Pass the GPU through only if the toolkit and driver are detected |
| `none` | Never pass a GPU through |
| `all` or any Docker `--gpus` value (e.g. `device=0`) | Force it |

The setting is read when the container is created, so use `CF_GPUS=none cf rebuild` to change an existing container. AMD and Intel GPUs are not covered by this switch. A privileged container can see `/dev/dri`, so Mesa-based acceleration may work, but this is untested.

## Install

`cf` doesn't depend on where it lives. Copy it or symlink it onto your `PATH`:

```bash
chmod +x cf
mkdir -p ~/.local/bin
ln -sf "$(realpath cf)" ~/.local/bin/cf     # or: cp cf ~/.local/bin/cf
```

If `cf` isn't found afterwards, make sure `~/.local/bin` is on your `PATH` and open a new shell.

### Tab completion (bash)

For command tab completion

```bash
mkdir -p ~/.local/share/bash-completion/completions
cf completion > ~/.local/share/bash-completion/completions/cf
```

Open a new shell to pick it up. This needs the `bash-completion` package (`sudo dnf install bash-completion` / `sudo apt install bash-completion`), and it completes the command name `cf`, so `cf` needs to be on your `PATH`. Re-run the command above after updating `cf` if new commands were added. Alternatively, add `source <(cf completion)` to `~/.bashrc`.

## Quick start

```bash
# 1. Tell cf which directory to mount. Do this once, before the first `cf start`.
cf mount ~/aosp

# 2. (optional) save a default images dir so `cf launch` needs no arguments
cf default ~/aosp/android14-release/out/target/product/vsoc_x86_64_only

# 3. create + start the container, then boot the device
cf start
cf launch
```

Then:

- Web UI: <https://localhost:1080>
- ADB: `adb connect 127.0.0.1:6520`

### Which directory should `cf mount` point to?

The directory that **contains everything you'll pass to `cf launch`**. Both the images dir (`.../out/target/product/<product>`) and the host tools dir (`.../out/host/linux-x86`) must be inside it.

- **A parent holding several checkouts** (`~/aosp` with `android14-release/`, `13/`, ...) is the most flexible: you can switch builds with `cf launch <dir>` and never touch the mount again.
- **A single checkout** works too, but any other tree is unreachable until you change the mount and run `cf rebuild`.
- If your `out/` is redirected elsewhere (`OUT_DIR`), the mount must cover that location too.

Avoid mounting something as broad as your home directory: the container is privileged and runs as root.

The mount is only read when the container is **created** (first `cf start`, or after `cf remove` / `cf rebuild`). It is saved, so you run `cf mount` once; a plain `cf start` on an existing container never looks at it. To change it later: `cf mount <new dir>` then `cf rebuild`.

## Commands

| Command | What it does |
|---|---|
| `cf mount [dir \| --reset]` | Show, set or clear the directory mounted into the container |
| `cf start` | Create the container (first time) or start the existing one |
| `cf stop` | Stop the container (it is kept) |
| `cf rebuild` | Remove the container, rebuild the image, recreate the container |
| `cf remove` | Remove the container (the image is kept) |
| `cf launch [images_dir] [tools_dir]` | Run `launch_cvd` in the container as you. Starts the container if needed. |
| `cf shell` | Open a shell in the container as you |
| `cf default [dir \| --reset]` | Show, set or clear the default images dir |
| `cf completion` | Print the bash completion script |
| `cf help` | Show usage |

`tools_dir` defaults to `<out>/host/linux-x86`, derived from the standard AOSP layout `<out>/target/product/<product>`. Pass it explicitly if yours differs.

## Configuration

Settings are stored in `~/.config/cf/` (or `$XDG_CONFIG_HOME/cf/`).

**Images dir** for `launch`, in order of precedence. There is no built-in default, and `launch` errors if none is set:

1. an argument: `cf launch <dir>`
2. the environment: `CF_IMAGES_DIR=<dir> cf launch`
3. the saved default: `cf default <dir>`

**Mount dir**, read at container creation: `CF_MOUNT_DIR` (environment), then the saved value from `cf mount`.

**GPU:** `CF_GPUS`, described [above](#optional-nvidia-gpu-acceleration).

## Troubleshooting

**`User must be a member of kvm` / `Validation of user configuration failed`**
The container was created from an older image without the right group memberships. Run `cf rebuild`.

**`/dev/vhost-vsock not found`**
`sudo modprobe vhost_vsock` (see [Prerequisites](#required)).

**`could not select device driver "" with capabilities: [[gpu]]`**
Docker was asked for a GPU but the NVIDIA runtime isn't registered. Follow the [GPU steps](#optional-nvidia-gpu-acceleration), or run `CF_GPUS=none cf rebuild`.

**`images dir isn't inside the container's mount`**
The container only sees the directory it was created with. Run `cf mount <parent dir>` then `cf rebuild`.

**`out/` already contains root-owned files** (from an earlier root-based setup)
Fix once with `sudo chown -R "$USER:" <aosp>/out`. With `cf` nothing new should become root-owned.

**SELinux alert about `systemd-coredump` and the `kill` capability (Fedora)**
This shows up when `run_cvd` crashes; the coredump handler is reacting to the abort, not causing it. Fix the crash (the launch output says why) rather than adding a policy module.

**`Failed to load library: libEGL.so` at launch**
These lines can be printed while Cuttlefish probes for GPU support; the launch continues.

## Security notes

The container runs with `--privileged` and `--network host`. The image's init creates tap devices and bridges in the host's network namespace, and the container has access to every host device. Only run this on a machine and network you're comfortable giving that access to, and only mount what you need.

## Uninstall

```bash
cf remove
docker rmi cuttlefish-local:latest
rm -rf ~/.config/cf
rm -f ~/.local/bin/cf ~/.local/share/bash-completion/completions/cf
```
