# 🔒 Validator Server Security Hardening Guide

**By [Echo Hound Labs](https://github.com/echohound-labs)**

If you're running a validator on a public server, bots are already trying to break in. Here's how many attempts our server had before we locked it down:

```bash
sudo grep "Failed password" /var/log/auth.log | wc -l
# Our result: 37,108 attempts
```

This guide covers the three essential layers of server security:
- **SSH Key Authentication** — stops unauthorized logins completely
- **UFW Firewall** — controls who can access your ports
- **Fail2ban** — bans IPs that hammer your server

---

## 🔑 Part 1 — SSH Key Authentication

The standard username/password login is vulnerable to brute force attacks. SSH keys replace passwords with a cryptographic key pair — no key file, no entry, period.

### Step 1 — Generate a Key Pair (on your LOCAL machine)

```bash
ssh-keygen -t ed25519 -C "your-validator-label"
```
- Press Enter to accept default location
- Set a strong passphrase when prompted

### Step 2 — Copy Your Public Key to the Server

```bash
ssh-copy-id user@your-server-ip
```
Enter your server password one last time.

### Step 3 — Test Key Login Works

Open a **new terminal** and connect:
```bash
ssh user@your-server-ip
```
It should ask for your **key passphrase**, not your server password.

> ⚠️ Do not continue until this works. Keep your old session open.

### Step 4 — Disable Password & Root Login

On the server:
```bash
sudo sed -i 's/.*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo sed -i 's/.*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/.*PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
```

Verify:
```bash
sudo grep -E "PasswordAuthentication|PubkeyAuthentication|PermitRootLogin" /etc/ssh/sshd_config
```

Expected output:
```
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
```

Restart SSH:
```bash
sudo systemctl restart ssh
```

### Step 5 — Final Test

Open a new terminal and SSH in to confirm it works. If it connects with your key passphrase, you're done with Part 1.

---

## 🛡️ Part 2 — UFW Firewall

UFW controls what ports are open and who can access them.

### Basic Setup

```bash
# Enable UFW
sudo ufw enable

# Allow SSH (do this FIRST or you'll lock yourself out)
sudo ufw allow 22

# Allow validator gossip/TPU ports (required for consensus)
sudo ufw allow 8000:8025/udp
sudo ufw allow 8000:8025/tcp
```

### Restrict RPC Ports

Your RPC ports (8899/8900) should only be accessible from your own IP — not the public internet:

```bash
# Allow your IP range on RPC ports
sudo ufw allow from YOUR.IP.0.0/16 to any port 8899
sudo ufw allow from YOUR.IP.0.0/16 to any port 8900

# Deny everyone else
sudo ufw deny 8899
sudo ufw deny 8900
```

> ⚠️ **Rule order matters** — ALLOW rules must appear BEFORE DENY rules for the same port. UFW processes top to bottom, first match wins.

### Check Your Rules

```bash
sudo ufw status numbered
```

### Common Pitfalls

| Mistake | Consequence |
|--------|-------------|
| Adding DENY before ALLOW | Your own IP gets blocked |
| Deleting rules low-to-high | Rule numbers shift, you delete the wrong ones |
| Restricting port 22 to a dynamic IP | IP changes, locked out |
| Leaving `8900/tcp ALLOW Anywhere` | RPC wide open to the internet |

Always delete UFW rules **highest number first** to avoid numbering shifts.

---

## 🚫 Part 3 — Fail2ban

SSH keys stop attackers from logging in, but bots can still hammer your server — filling logs, consuming CPU and connection slots. Fail2ban automatically bans IPs that repeatedly fail authentication.

### Install

```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### Verify It's Working

```bash
sudo fail2ban-client status sshd
```

You'll see currently banned IPs and total failed attempts. Ours had **9 IPs banned within minutes** of installing.

### Optional — Tighten the Settings

Create a local config (overrides defaults safely):
```bash
sudo nano /etc/fail2ban/jail.local
```

Add:
```ini
[sshd]
enabled = true
maxretry = 3
findtime = 600
bantime = 3600
```

This bans IPs after **3 failed attempts** within 10 minutes for **1 hour**. Restart to apply:
```bash
sudo systemctl restart fail2ban
```

---

## ✅ Full Security Checklist

```bash
# How many brute force attempts have you had?
sudo grep "Failed password" /var/log/auth.log | wc -l

# Is root login disabled?
sudo grep "PermitRootLogin" /etc/ssh/sshd_config

# Is password auth disabled?
sudo grep "PasswordAuthentication" /etc/ssh/sshd_config

# Are your firewall rules clean?
sudo ufw status numbered

# Is Fail2ban running and banning?
sudo fail2ban-client status sshd
```

---

## 🔑 Keep Your Keys Safe

Your private key lives at `~/.ssh/id_ed25519` on your local machine.

- Back it up somewhere secure
- Never share it
- Never copy it to the server
- If you lose it with no other access method, you'll need console/KVM access from your hosting provider

---

*Built by [Echo Hound Labs](https://github.com/echohound-labs) — securing the X1 Network*
