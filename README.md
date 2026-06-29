# dgx-spark-serving-mode

A small `serving-mode` script to flip a **DGX Spark** (or any Ubuntu box) between
its full desktop state and a **headless inference-serving** state — handing the
GPU as much memory as possible.

## Why this matters

GB10-based systems — the NVIDIA DGX Spark, ASUS Ascent GX10, and similar — are
uniquely suited to *serving* large MoE LLMs: 128 GB of unified memory per node,
and with ConnectX-7 you can cluster them (up to TP=8) to put even ~1T-parameter
models within reach. Once that's the box's job, serving is effectively its
primary purpose — and **every MB you free is more room for KV cache: more
concurrency and better prefix-cache reuse.**

The catch is that the 128 GB is *unified*, shared between the OS/desktop and the
GPU. Whatever the GNOME desktop, the display manager, snap, and assorted
maintenance daemons/timers hold (commonly **~10–15 GB**) is memory the vLLM KV
cache *doesn't* get. Dropping to `multi-user.target` (no GUI) and paring back
those services hands it back to the model — directly raising the usable KV pool,
context length, and concurrency.

If you're seeing a smaller-than-expected KV pool (e.g. `Maximum concurrency for
N tokens` below ~1×, or `Available KV cache memory` a dozen GiB lower than
someone else's on the same hardware), the desktop/extra services are the usual
cause.

This is the companion to the serving recipe at
**[Entrpi/qwen3.5-122B-A10B-on-spark](https://github.com/Entrpi/qwen3.5-122B-A10B-on-spark)**.

## Install

```bash
mkdir -p ~/bin
curl -fsSL https://raw.githubusercontent.com/Entrpi/dgx-spark-serving-mode/main/serving-mode -o ~/bin/serving-mode
chmod +x ~/bin/serving-mode
# optional: manage your own GPU sidecars / a model unit
cp serving-mode.conf.example ~/.config/serving-mode.conf   # then edit
```

## Use

```bash
~/bin/serving-mode status        # current target, MemAvailable, docker, units (no sudo)
sudo ~/bin/serving-mode on       # pare desktop + maintenance (incl. docker); multi-user.target
sudo ~/bin/serving-mode serve    # headless serve: pare desktop, KEEP docker, bring up model + user units
sudo ~/bin/serving-mode off      # restore the full graphical desktop and all services
```

- **`on`** — maximum memory freed for *manual* serving (you run the container
  yourself). Disables docker too; add `-u` to also stop your `--user` units.
- **`serve`** — the production state: desktop pared, **docker kept up**, the
  model unit + your user units (sidecars/agent) brought up, **linger enabled** so
  the whole stack survives logout/reboot.
- **`off`** — back to the normal graphical desktop.

Both `on` and `serve` `set-default multi-user.target`, so they **persist across
reboots**. `status` needs no sudo.

### What stays running (and why)

`serving-mode` pares back the desktop and *maintenance* layer only — the box
stays reachable and serving. These are **never touched** in any mode (all
verified `active` on a serving DGX Spark):

| Unit | Purpose | Why it stays on |
|---|---|---|
| `ssh.service` | SSH server | The only way into a headless box — stopping it locks you out |
| `NetworkManager` / `systemd-networkd` | Network & link management | One of them owns the SSH link; stopping it drops the network |
| `systemd-resolved` | DNS resolution | Name resolution for model / registry / package fetches |
| `systemd-timesyncd` | NTP clock sync | Correct time for TLS certs, logs, and scheduled jobs |
| `systemd-udevd` | Device manager | Enumerates the GPU and storage; the GPU may not initialize without it |
| `systemd-journald` | System logging | Captures vLLM / container logs for debugging |
| `systemd-logind` | Login & session manager | Sessions + `enable-linger`, so `serve`'s user units survive logout |
| `dbus.service` | IPC message bus | `systemctl --user`, logind, and NetworkManager all talk over it |
| `polkit.service` | Privilege authorization | `systemctl` / service actions need it to authorize |
| `nvidia-persistenced` | NVIDIA persistence daemon | Keeps the driver/GPU initialized between CUDA clients — avoids re-init latency/instability |
| `rasdaemon` | ECC / RAS error logging | Witnesses memory (ECC) errors under the heavy memory pressure of large-model serving |
| `getty@tty1` | Local console login | Recovery TTY if SSH / the network ever fails |
| `user@<uid>.service` | systemd `--user` manager | Hosts your `--user` units (sidecars/agent) and ssh-session scopes |
| `logrotate.timer` | Log rotation | Keeps journald / logs from filling the disk |
| `fstrim.timer` | Weekly SSD TRIM | Maintains SSD performance and longevity |

The desktop/maintenance units it *does* stop are listed in the `SERVICES`,
`DOCKER_SERVICES`, and `TIMERS` arrays at the top of the script.

## Optional: run vLLM as a managed service

[`examples/vllm-model.service`](examples/vllm-model.service) is a template that
runs the [qwen3.5-122B-A10B-on-spark](https://github.com/Entrpi/qwen3.5-122B-A10B-on-spark)
container under systemd. Install it, set `MODEL_UNIT="vllm-model.service"` in
`~/.config/serving-mode.conf`, and `serving-mode serve` will start it (and it
auto-restarts + survives reboot).

## Notes

- `serving-mode off` re-enables the **full** managed list — including anything
  you'd manually disabled before. Adjust the `SERVICES`/`TIMERS` arrays at the
  top of the script to taste.
- The script is conservative: each unit is toggled with `disable --now` /
  `enable --now` and failures are ignored, so a unit you don't have is a no-op.
- No secrets here. If you run GPU sidecars with their own tokens, keep those in
  your `systemd --user` unit files (referenced by name in `serving-mode.conf`),
  not in this repo.
