# Resilient Remote Terminal Setup

A guide to building persistent, responsive terminal sessions across unreliable
networks — mobile hotspots, wifi roaming, VPN switches, or anything else that
drops connections.

The setup combines three layers:

| Layer         | Tool           | Role                                              |
|---------------|----------------|---------------------------------------------------|
| Networking    | **Tailscale**  | Stable IPs, NAT traversal, encrypted WireGuard    |
| Transport     | **SSH / Mosh** | SSH for full-featured, mosh for high-latency      |
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

Check connection type:

```bash
tailscale status
```

Look for `direct` vs a DERP relay next to each peer.

## tmux

tmux is a terminal multiplexer — it lets you run sessions that persist
independently of your terminal connection. If your SSH drops, everything keeps
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

## SSH + tmux

The baseline setup. SSH gives you a full-featured connection with true color
support, and tmux handles persistence.

Start a new session:

```bash
ssh user@<server-ip> -t "tmux new -s main"
```

Reattach later:

```bash
ssh user@<server-ip> -t "tmux attach -t main"
```

This is the recommended setup when you need true color (24-bit) — for example,
when running terminal apps like Claude Code that rely on it.

## Mosh

[Mosh](https://mosh.org) replaces the SSH transport with a UDP-based protocol.
It uses SSH for initial authentication, then hands off to a UDP channel. This
means your existing SSH keys and config just work — no separate auth setup. The
tradeoff: you get better responsiveness on bad connections, but lose true color
support (as of v1.3.2).

### Why mosh over SSH

- **Local echo** — keystrokes display instantly, even on high-latency connections
- **Roaming** — seamlessly survives wifi-to-mobile switches, sleep/wake, IP changes
- **UDP-based** — avoids TCP head-of-line blocking
- **Stateful sync** — only transmits screen diffs, not raw byte streams

### When to use mosh vs SSH

| Use case                          | Recommendation       |
|-----------------------------------|----------------------|
| Stable connection (LAN, wired)    | SSH + tmux           |
| True color apps (Claude Code)     | SSH + tmux           |
| Mobile / high latency / roaming   | Mosh + tmux          |
| No sudo on remote machine         | Mosh (from source)   |

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

**From source (no sudo):**

If you don't have root access on the remote machine, build mosh into a local
prefix:

```bash
./configure --prefix=$HOME/local
make
make install
```

Then connect with a custom server path:

```bash
mosh --server="LD_LIBRARY_PATH=\$HOME/local/lib:\$HOME/local/lib64 \$HOME/local/bin/mosh-server" user@<server-ip>
```

You can alias this in `~/.bashrc`:

```bash
alias mosh-custom='mosh --server="LD_LIBRARY_PATH=\$HOME/local/lib:\$HOME/local/lib64 \$HOME/local/bin/mosh-server" user@<server-ip>'
```

### Firewall

Mosh uses UDP ports 60000-61000. If you have a firewall:

```bash
sudo ufw allow 60000:61000/udp
```

Tailscale traffic bypasses this (it runs over the WireGuard tunnel), but this
is needed for direct connections.

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

### Mosh true color limitation

Mosh 1.3.2 (current stable) does not support true color (24-bit). Apps that
use true color will render incorrectly — missing colors, white text on white
background.

True color support was [merged into mosh's master branch](https://github.com/mobile-shell/mosh/pull/939)
in 2017, but no stable release has included it since. If you need true color
over mosh, you'll have to build from source (see the
[from source](#from-source-no-sudo) section above — same process, just clone
from `master`).

**Workaround (without rebuilding):** unset `COLORTERM` before launching the
app so it falls back to 256 colors, which mosh handles fine:

```bash
unset COLORTERM
```

If you need true color and don't want to build from source, use SSH + tmux
instead. Since tmux already handles session persistence, mosh's reconnection
benefit is less critical for stationary use.
