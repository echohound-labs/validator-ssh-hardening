# 🔒 Secure Your Validator SSH — Key-Only Authentication

**By [Echo Hound Labs](https://github.com/echohound-labs)**

If you're running a validator on a public server, bots are already trying to break in. Here's how many attempts our server had before we fixed it:

```bash
sudo grep "Failed password" /var/log/auth.log | wc -l
# Our result: 37,108 attempts
```

This guide sets up SSH key authentication and disables password login entirely. No key file = no entry, period.

---

## Step 1 — Generate a Key Pair (on your LOCAL machine)

```bash
ssh-keygen -t ed25519 -C "your-validator-label"
```
- Press Enter to accept default location
- Set a strong passphrase when prompted — this protects your key file

---

## Step 2 — Copy Your Public Key to the Server

```bash
ssh-copy-id user@your-server-ip
```
Enter your server password one last time.

---

## Step 3 — Test Key Login Works

Open a **new terminal** and connect:
```bash
ssh user@your-server-ip
```
It should ask for your **key passphrase**, not your server password.

> ⚠️ Do not continue until this works. Keep your old session open.

---

## Step 4 — Disable Password & Root Login

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

---

## Step 5 — Final Test

Open a new terminal and SSH in one more time to confirm everything works. If it connects with your key passphrase, you're done.

Those 37,000+ brute force attempts are now completely useless. 🔐

---

*Built by [Echo Hound Labs](https://github.com/echohound-labs) — securing the X1 Network*
