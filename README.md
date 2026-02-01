# Resilient Remote Terminal Setup

A guide to building persistent, responsive terminal sessions across unreliable
networks — mobile hotspots, wifi roaming, VPN switches, or anything else that
drops connections.

The setup combines three layers:

| Layer         | Tool           | Role                                              |
|---------------|----------------|---------------------------------------------------|
| Networking    | **Tailscale**  | Stable IPs, NAT traversal, encrypted WireGuard    |
| Transport     | **SSH / Mosh** | SSH for simplicity, mosh for responsiveness and resilience |
| Persistence   | **tmux**       | Detach/reattach sessions, survives disconnects     |

You don't need all three — each section stands alone. But together they give
you a remote terminal that survives basically anything.

## Tailscale

[Tailscale](https://tailscale.com) assigns each device a stable IP on a
private WireGuard mesh network. This means you can SSH/mosh to a machine by
the same IP regardless of what network either device is on.

Tailscale establishes direct peer-to-peer connections when possible:

- **Same LAN** — traffic stays local, doesn't leave the network
- **Different networks** — uses NAT traversal for a direct connection, falls
  back to DERP relay servers only if that fails

## tmux

tmux is a terminal multiplexer — it lets you run sessions that persist
independently of your terminal connection. If your connection drops, everything keeps
running. You just reconnect and reattach.

### Key concepts

| Concept     | Description                                                        |
|-------------|--------------------------------------------------------------------|
| **Session** | A collection of windows managed by tmux. Persists after detaching. |
| **Window**  | A full-screen tab within a session.                                |
| **Pane**    | A split within a window (horizontal or vertical).                  |

### Common commands

```bash
tmux                    # Start a new session
tmux new -s main        # Start a named session
tmux attach -t main     # Reattach to a session
tmux ls                 # List sessions
```

### Keybindings (default prefix: Ctrl-b)

| Keybinding     | Action                  |
|----------------|-------------------------|
| `Ctrl-b d`     | Detach from session     |
| `Ctrl-b c`     | Create new window       |
| `Ctrl-b %`     | Split pane vertically   |
| `Ctrl-b "`     | Split pane horizontally |
| `Ctrl-b n`     | Next window             |
| `Ctrl-b p`     | Previous window         |
| `Ctrl-b [`     | Enter scroll/copy mode  |
| `Ctrl-c`       | Exit scroll/copy mode   |

## SSH + tmux

The baseline setup. SSH handles the connection, tmux handles persistence.

Start a new session:

```bash
ssh user@<server-ip> -t "tmux new -s main"
```

Reattach later:

```bash
ssh user@<server-ip> -t "tmux attach -t main"
```

This is a solid default for any use case, especially on stable connections.

## Mosh

[Mosh](https://mosh.org) replaces the SSH transport with a UDP-based protocol.
It uses SSH for initial authentication, then hands off to a UDP channel. This
means your existing SSH keys and config just work — no separate auth setup. It
does everything SSH does, plus:

- **Local echo** — keystrokes display instantly, even on high-latency connections
- **Roaming** — seamlessly survives wifi-to-mobile switches, sleep/wake, IP changes
- **UDP-based** — avoids TCP head-of-line blocking
- **Stateful sync** — only transmits screen diffs, not raw byte streams

**Important:** true color (24-bit) requires mosh 1.4.0+. Many distros still
ship 1.3.2 via `apt` — see [Fixing Colors](#fixing-colors) for how to get
1.4.0.

### Installation

**Debian/Ubuntu:**

```bash
sudo apt install mosh
```

**Termux (Android):**

```bash
pkg install mosh
```

If you get a linker error like `CANNOT LINK EXECUTABLE "mosh-client"`, update
all packages first:

```bash
pkg upgrade
```

**From source:**

If your distro doesn't ship 1.4.0+ or you don't have root access, build mosh
into a local prefix:

```bash
./configure --prefix=$HOME/.local
make
make install
```

Then connect with a custom server path:

```bash
mosh --server="LD_LIBRARY_PATH=\$HOME/.local/lib:\$HOME/.local/lib64 \$HOME/.local/bin/mosh-server" user@<server-ip>
```

You can alias this in `~/.bashrc`:

```bash
alias mosh-custom='mosh --server="LD_LIBRARY_PATH=\$HOME/.local/lib:\$HOME/.local/lib64 \$HOME/.local/bin/mosh-server" user@<server-ip>'
```

### Firewall

If you're connecting over Tailscale, the traffic runs through the WireGuard
tunnel and no firewall changes are needed. For direct connections, mosh uses
UDP ports 60000-61000:

```bash
sudo ufw allow 60000:61000/udp
```

## Mosh + tmux

The full stack — mosh for network resilience, tmux for session persistence.

Start a new session:

```bash
mosh user@<server-ip> -- tmux new -s main
```

Reattach later:

```bash
mosh user@<server-ip> -- tmux attach -t main
```

## Fixing Colors

### tmux color config

tmux defaults to `TERM=screen`, which breaks 256-color and true color. Add to
`~/.config/tmux/tmux.conf`:

```bash
set -g default-terminal "tmux-256color"
set -ga terminal-overrides ",*256col*:Tc"
```

Restart tmux for it to take effect (`tmux kill-server`).

To force a specific color mode per-session:

```bash
tmux -2 new -s main           # force 256-color
TERM=screen tmux new -s main  # force basic
```

### Mosh true color

Mosh supports true color (24-bit) since v1.4.0. However, some distros
(including Ubuntu) still ship v1.3.2 via `apt`, which doesn't handle true
color — apps will render with missing colors or white text on white background.

**Fix:** build mosh 1.4.0+ from source (see the
[from source](#from-source) section above).

**Workaround (without rebuilding):** unset `COLORTERM` before launching the
app so it falls back to 256 colors, which older mosh handles fine:

```bash
unset COLORTERM
```
