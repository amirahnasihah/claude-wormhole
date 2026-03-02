# Deploy claude-wormhole on Linux (Hetzner, VPS, etc.)

Run claude-wormhole on the **same machine** where you already run Claude Code (e.g. via SSH/Termius). Then access the web UI from your phone or laptop over Tailscale — no public ports.

**Why this works:** Wormhole must run where tmux and Claude Code run, because it uses `node-pty` to attach to tmux sessions. If you SSH into a Hetzner server to run `cld` / Claude Code, install wormhole on that server.

## Architecture

```
Phone / Laptop (browser or PWA)
  │
  │  Tailscale (private)
  │
Hetzner server
  ├── tmux + Claude Code sessions  (same as today via Termius)
  ├── wormhole (Next.js + WebSocket)
  └── tailscale serve → https://your-server.tailnet.ts.net:3100/
```

## 1. Prerequisites on the server (Ubuntu)

SSH into your Hetzner server, then:

```bash
# Node.js 20+ (via NodeSource or nvm)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Or with nvm (recommended if you already use it):
# curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
# source ~/.bashrc && nvm install 20

# tmux, jq
sudo apt-get update
sudo apt-get install -y tmux jq

# Claude Code CLI (same account you use in Cursor)
npm install -g @anthropic-ai/claude-code
```

Optional: [TPM](https://github.com/tmux-plugins/tpm) for tmux-resurrect/continuum:

```bash
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
```

Copy the project’s tmux config if you want:

```bash
cp /path/to/claude-wormhole/scripts/tmux.conf ~/.tmux.conf
# Then inside tmux: prefix + I to install plugins
```

## 2. Install Tailscale on the server

So your phone/laptop can reach the server without opening public ports:

```bash
# Ubuntu/Debian
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
# Log in with the same Tailscale account as your phone/laptop
```

Enable Tailscale serve (HTTPS) for port 3100:

```bash
tailscale serve --bg --https 3100 http://127.0.0.1:3100
```

Your UI will be at: `https://<your-server-name>.tailnet-name.ts.net:3100/`

## 3. Deploy claude-wormhole

On the server (as the same user you use for SSH and Claude Code):

```bash
cd ~
git clone https://github.com/ssv445/claude-wormhole.git
cd claude-wormhole

npm install
npm run build
```

Create a `.env.local` if you want a different port (optional):

```bash
echo "PORT=3100" > .env.local
```

**Run once manually to confirm:**

```bash
./bin/wormhole start
# In another SSH session:
./bin/wormhole status   # should show Web server: healthy
curl -s http://127.0.0.1:3100/api/sessions
```

Then Ctrl+C the server and run it as a service (next step).

## 4. Run as a service (systemd, Linux)

`install.sh` and `wormhole service` are written for **macOS (launchd)**. On Linux use systemd.

**Option A – User service (recommended)**  
Runs as your SSH user, so it sees your tmux sessions and `claude` CLI.

```bash
mkdir -p ~/.config/systemd/user
# Replace /home/YOUR_USER/claude-wormhole with your actual path
export WORMHOLE_ROOT="$HOME/claude-wormhole"
sed "s|__WORMHOLE_ROOT__|$WORMHOLE_ROOT|g" "$WORMHOLE_ROOT/scripts/claude-wormhole.service" \
  > ~/.config/systemd/user/claude-wormhole.service

systemctl --user daemon-reload
systemctl --user enable claude-wormhole
systemctl --user start claude-wormhole
systemctl --user status claude-wormhole
```

**Option B – Keep it simple (no systemd)**  
Use a terminal multiplexer so the process survives disconnect:

```bash
cd ~/claude-wormhole
nohup ./bin/wormhole start >> /tmp/wormhole.log 2>&1 &
# Or run inside tmux/screen so you can attach and see logs
```

**Tailscale serve** must be enabled after reboot (or add it to a startup script):

```bash
tailscale serve --bg --https 3100 http://127.0.0.1:3100
```

## 5. Create a session and open the UI

On the server (via Termius or SSH):

```bash
cd ~/claude-wormhole   # or any project
./bin/wormhole cld      # or: wormhole cld if you symlinked
# Or: tmux new -s my-app && claude
```

On your phone (Tailscale app connected): open  
`https://<your-server>.tailnet-name.ts.net:3100/`  
You should see your tmux sessions; tap one to open the terminal.

## 6. Symlink `wormhole` (optional)

```bash
sudo ln -sf "$HOME/claude-wormhole/bin/wormhole" /usr/local/bin/wormhole
# Then: wormhole status, wormhole cld, etc.
```

## Security notes

- **Do not expose port 3100 to the public internet.** Use Tailscale (or a VPN) so only your devices can reach the UI.
- Tailscale serve gives you HTTPS and ties access to your Tailnet.
- If you ever expose the app without Tailscale, put it behind a reverse proxy with authentication (e.g. nginx + basic auth or OAuth).

## Troubleshooting

| Issue | Check |
|-------|--------|
| `wormhole status` → Web server down | `systemctl --user status claude-wormhole` or run `./bin/wormhole start` in foreground and see errors. |
| Can’t connect from phone | Tailscale app connected? Same account as server? `tailscale serve --bg --https 3100 http://127.0.0.1:3100` run on server? |
| No sessions in UI | Create a tmux session on the server (e.g. `wormhole cld` or `tmux new -s foo`). |
| node-pty / spawn errors | Install build tools: `sudo apt-get install -y build-essential python3`; then `npm rebuild` in the repo. |

## Alternatives

- **Keep using only Termius:** You can keep using SSH + tmux + Claude Code without the web UI; wormhole just adds a browser/phone UI and optional push notifications.
- **Claude Code “remote control” (Pro):** If that feature becomes available for your account, it’s an official option; wormhole is a self-hosted alternative that works today with Tailscale.
