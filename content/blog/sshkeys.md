---
title: "ssh key management"
date: 2026-04-30T08:05:28+10:00
draft: false
tags: ["Linux", "ssh"]
categories: ["Linux", "Security"]
featured_image: ""
summary: "SSH Key Management Best Practices"
---

# SSH Key Management Best Practices

## Core Principle — One Key Per Client Machine

A key represents **you on a specific machine**. Each machine should have its own unique keypair. The private key never leaves the machine it was created on.

```
Desktop  → has its own key → published to GitHub + laptop server
Laptop   → has its own key → published to GitHub + any other servers
```

### Why This Approach

- If one machine is **compromised or lost**, you only revoke that one key
- You can see exactly **which machine** made a connection in server logs
- GitHub and servers can identify **where** you connected from
- You never need to copy private keys between machines

---

## Key Distribution Map

```
Desktop keypair (id_ed25519_desktop)
 ├── Public key → GitHub (Settings → SSH Keys)
 ├── Public key → Laptop server (~/.ssh/authorized_keys)
 └── Private key → stays on desktop ONLY

Laptop keypair (id_ed25519_laptop)
 ├── Public key → GitHub (Settings → SSH Keys)
 ├── Public key → Desktop (~/.ssh/authorized_keys) if needed
 └── Private key → stays on laptop ONLY
```

---

## Generating Keys

Give keys descriptive names to distinguish them:

```bash
# On desktop
ssh-keygen -t ed25519 -C "github-desktop" -f ~/.ssh/id_ed25519_desktop

# On laptop server
ssh-keygen -t ed25519 -C "github-laptop" -f ~/.ssh/id_ed25519_laptop
```

---

## Copying Public Keys to Remote Servers

From the **client machine**, copy the public key to the target server:

```bash
ssh-copy-id username@server-ip
```

This appends the public key to `~/.ssh/authorized_keys` on the server.

### Verify the Key Was Copied

On the **server**, confirm the key is present:

```bash
cat ~/.ssh/authorized_keys
```

Cross-check on the **client** that it matches:

```bash
cat ~/.ssh/id_ed25519_desktop.pub
```

The output should be identical.

---

## Adding Keys to GitHub

GitHub supports multiple SSH keys per account — one per machine. Both keys authenticate as the same GitHub user, so commits show the same username regardless of which machine you push from.

1. Go to **GitHub → Settings → SSH and GPG Keys → New SSH key**
2. Add the public key from each machine with a descriptive label:
   - `Desktop - Ubuntu 24.04`
   - `Laptop Server - Ubuntu 26.04`

---

## SSH Config File

Create `~/.ssh/config` on each machine to manage multiple keys and define shortcuts:

```
# Desktop ~/.ssh/config

Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_desktop

Host laptop-server
    HostName 192.168.10.x
    User yourusername
    IdentityFile ~/.ssh/id_ed25519_desktop
```

With this config you can connect using the alias instead of the full address:

```bash
# Connect to laptop server
ssh laptop-server

# Git works normally
git clone git@github.com:yourrepo.git
```

---

## Disabling Password Authentication (After Key Setup)

Once SSH key login is confirmed working, disable password authentication on the server:

```bash
sudo nano /etc/ssh/sshd_config
```

Set the following:

```
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```

Restart SSH to apply:

```bash
sudo systemctl restart ssh
```

> **Warning:** Always confirm key-based login works before disabling password authentication, otherwise you risk locking yourself out.

---

## What You Should Never Do

- Never copy a **private key** from one machine to another
- Never share one private key across multiple machines
- Never store private keys in a repository or cloud storage
- Never disable password auth before verifying key login works

---

## Key File Reference

| File | Location | Description |
|------|----------|-------------|
| `id_ed25519` | `~/.ssh/` | Private key — never share |
| `id_ed25519.pub` | `~/.ssh/` | Public key — safe to distribute |
| `authorized_keys` | `~/.ssh/` | Public keys of machines allowed to connect in |
| `config` | `~/.ssh/` | SSH client configuration and host aliases |
| `known_hosts` | `~/.ssh/` | Fingerprints of servers you have connected to |