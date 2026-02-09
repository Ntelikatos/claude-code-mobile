# Claude Code Mobile

Run Claude Code from anywhere via SSH. Deploy on Railway and access it from your phone.

## Prerequisites

Before you begin, make sure you have:

1. **An SSH key pair** (Ed25519 recommended). Generate one if you don't have it:
   ```bash
   ssh-keygen -t ed25519 -C "your@email.com"
   ```
2. **A Railway account** at [railway.com](https://railway.com)
3. **A mobile SSH client**: [Termius](https://termius.com) (iOS/Android), [Blink Shell](https://blink.sh) (iOS), or [JuiceSSH](https://juicessh.com) (Android)

---

## Deploy to Railway (Step by Step)

### Step 1: Deploy the Template

[![Deploy on Railway](https://railway.app/button.svg)](https://railway.app/template/claude-code-mobile)

Click the button above, or deploy manually:
1. Create a new project on [railway.com](https://railway.com)
2. Select **"Deploy from GitHub repo"** and connect this repository

### Step 2: Set Your SSH Public Key

In Railway dashboard, go to your service **Variables** and add:

| Variable | Value |
|----------|-------|
| `SSH_PUBLIC_KEY` | Your public key (contents of `~/.ssh/id_ed25519.pub`) |

This is the **only required variable**. To find your key:
```bash
cat ~/.ssh/id_ed25519.pub
```

### Step 3: Enable TCP Proxy

1. Go to your service **Settings** > **Networking**
2. Under **TCP Proxy**, click **Enable** on port `22`
3. Railway will assign you a domain and port, e.g. `shuttle.proxy.rlwy.net:15140`

Save this — you'll need it to connect.

### Step 4: Create and Attach a Volume

1. In Railway dashboard, right-click on your project canvas and select **"Add Volume"** (or go to your service **Settings** > **Volumes**)
2. Create a new volume and attach it to your service
3. Set the **mount path** to `/data`
4. Redeploy the service if prompted

The volume persists your projects, SSH host keys, tmux sessions, fail2ban state, and Claude Code auth across redeployments. Without it, all data is lost when the container restarts.

### Step 5: Connect via SSH

From your terminal or mobile SSH client:

```bash
ssh claude@shuttle.proxy.rlwy.net -p 15140 -i ~/.ssh/id_ed25519
```

Replace the domain and port with the values from Step 3.

### Step 6: Authenticate Claude Code

Once connected via SSH, you need to authenticate Claude Code. Choose one method:

**Option A: Claude Account (Pro/Max subscription)**
```bash
claude login
```
This prints a URL. Open it in any browser (phone or laptop), sign in with your Claude account, and the CLI session is authorized. No API key needed.

**Option B: API Key**

Either pass it as an environment variable when deploying (add `ANTHROPIC_API_KEY` in Railway Variables), or set it manually inside the container:
```bash
export ANTHROPIC_API_KEY=sk-ant-your-key-here
echo 'export ANTHROPIC_API_KEY=sk-ant-your-key-here' >> ~/.bashrc
```

Your auth persists across container restarts via the `/data` volume.

### Step 7: Start Coding

```bash
# Start a persistent tmux session
tmux new -s dev

# Launch Claude Code
claude

# Detach (keep session running): Ctrl+B, then D
# Reattach later:
tmux attach -t dev
```

---

## Mobile Client Setup

### Termius (iOS / Android)

1. Open Termius > **Hosts** > **+** (new host)
2. **Hostname**: your Railway TCP domain (e.g. `shuttle.proxy.rlwy.net`)
3. **Port**: your assigned port (e.g. `15140`)
4. **Username**: `claude`
5. Go to **Keychain** > import your Ed25519 private key
6. Assign the key to the host
7. Enable **Keep connection alive** in advanced settings

### Blink Shell (iOS)

```bash
# In Blink, configure the host:
config

# Then connect:
ssh claude@shuttle.proxy.rlwy.net -p 15140
```

Import your private key in Blink's key management settings.

### JuiceSSH (Android)

1. **Identities** > add new > import your private key
2. **Connections** > add new > set host, port, username `claude`
3. Select your identity
4. Enable keep-alive in advanced settings

---

## Persistent Sessions with tmux

Your SSH connection may drop (switching WiFi, cellular, etc). tmux keeps your session alive:

```bash
# Create a named session
tmux new -s dev

# Detach: Ctrl+B, then D

# List sessions
tmux ls

# Reattach after reconnecting
tmux attach -t dev
```

One-liner to auto-attach on SSH connect:
```bash
ssh claude@<host> -p <port> -i ~/.ssh/id_ed25519 -t "tmux attach -t dev || tmux new -s dev"
```

---

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `SSH_PUBLIC_KEY` | Yes | — | Ed25519 public key(s) for SSH auth. Supports multiple keys, newline-separated. |
| `ANTHROPIC_API_KEY` | No | — | Claude API key. Alternative to `claude login`. |
| `TZ` | No | `UTC` | Timezone (e.g. `America/New_York`, `Europe/London`) |

---

## Local / VPS Deployment

### Local Testing

```bash
docker build -t claude-code-mobile .

docker run -d \
  -e SSH_PUBLIC_KEY="$(cat ~/.ssh/id_ed25519.pub)" \
  -p 2222:22 \
  -v claude-data:/data \
  --name claude-mobile \
  claude-code-mobile

ssh claude@localhost -p 2222 -i ~/.ssh/id_ed25519
```

### VPS with iptables Support

On a VPS (Hetzner, DigitalOcean, Oracle Cloud), add `--cap-add=NET_ADMIN` for full fail2ban iptables-based banning:

```bash
docker run -d \
  -e SSH_PUBLIC_KEY="$(cat ~/.ssh/id_ed25519.pub)" \
  -p 22:22 \
  -v claude-data:/data \
  --cap-add=NET_ADMIN \
  --name claude-mobile \
  claude-code-mobile
```

---

## Security

- **Key-only authentication** — passwords completely disabled
- **Modern ciphers** — ChaCha20-Poly1305, AES-256-GCM (AEAD only)
- **Modern key exchange** — Curve25519, DH Group 16/18
- **ETM-only MACs** — HMAC-SHA2-512-ETM, HMAC-SHA2-256-ETM
- **fail2ban** — 3 failed attempts = 1 hour ban, repeat offenders = 1 week ban
- **Non-root user** — all operations run as unprivileged `claude` user
- **DH moduli filtered** — only >= 3071-bit entries
- **sshaudit.com compliant** — passes audit with modern-only algorithms

---

## Architecture

```
s6-overlay (PID 1)
├── init-setup (oneshot)
│   └── Volume init, SSH keys, DH moduli, user setup
├── syslog (longrun, depends: init-setup)
│   └── rsyslogd for auth logging
├── sshd (longrun, depends: init-setup)
│   └── Hardened OpenSSH daemon
└── fail2ban (longrun, depends: syslog, sshd)
    └── Brute-force protection with hostsdeny
```

---

## Cost

| Platform | Cost | Notes |
|----------|------|-------|
| Railway Hobby | ~$5-10/mo | Pay-as-you-go, TCP proxy included |
| Hetzner CX22 | ~€4/mo | 2 vCPU, 4GB RAM, full iptables |
| Oracle Cloud Free | $0 | ARM A1, always-free tier |

---

## Troubleshooting

**Host key warning after redeploy**: The host key changed. Remove the old one:
```bash
ssh-keygen -R "[your-host]:port"
```

**Connection refused**: Ensure TCP Proxy is enabled in Railway or port is exposed in Docker.

**Permission denied (publickey)**: Verify `SSH_PUBLIC_KEY` matches your private key. Check with `cat ~/.ssh/id_ed25519.pub`.

**Claude Code not authenticated**: Run `claude login` inside the container for browser-based auth, or set `ANTHROPIC_API_KEY`.

**tmux session lost**: Sessions persist only while the container runs. If the container restarted, start a new session. Your files on `/data` are preserved.

---

## License

MIT
