# Secure Shell (SSH)

## Overview
SSH was created to provide a secure way to perform remote system administration. The "S" (secure) guarantees:
1. You can confirm the identity of the remote server.
2. Exchanged messages are encrypted and can only be decrypted by the intended recipient.
3. Both sides can detect any modification in the messages.

These three guarantees map to **confidentiality** and **integrity**, achieved through proper use of encryption algorithms.

SSH completely replaced Telnet for interactive remote access due to its security guarantees. SSH server listens on **port 22** by default.

## SSH Authentication Methods
| Method | Notes |
|---|---|
| Password authentication | Simplest method; username/password sent over the encrypted SSH tunnel. Protected in transit, but vulnerable to brute force if weak passwords are used. |
| Public key authentication | Recommended for regular use. Key pair: private key (kept secret locally) + public key (placed on servers). Server challenges the client to prove possession of the private key without transmitting it. |
| Certificate-based authentication | Used in larger organisations. An SSH Certificate Authority signs user/host keys, removing the need to distribute public keys to every server individually. Scales better; supports key expiration and revocation. |
| Multi-factor authentication (MFA) | Combines multiple methods — many orgs now require both a key and a one-time password from an authenticator app. |

## Connecting via SSH
```bash
ssh username@10.112.174.85
```
Connects to the server at the specified IP with the given login name. If listening on the default port, the client is prompted for a password (or uses configured key auth). Once authenticated, all subsequent commands run over an encrypted channel.

## Host Key Verification
On first connection to a system, SSH prompts to confirm the fingerprint of the server's public key — this guards against MITM attacks. There is usually no third party verifying the key automatically, so this must be done manually or through out-of-band verification (e.g. asking the server admin, checking a secure configuration management system).

```
The authenticity of host '10.112.174.85' can't be established.
ED25519 key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Once accepted, the host key is stored in `~/.ssh/known_hosts`. If the key ever changes unexpectedly, SSH warns you — this could indicate a MITM attack or that the server was reinstalled.

## SSH Key Generation
```bash
# Generate an Ed25519 key (recommended for modern systems)
ssh-keygen -t ed25519 -C "your_email@example.com"

# For systems that don't support Ed25519, use RSA with 4096 bits
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```
- **Ed25519** keys are shorter, faster, and considered more secure than RSA — current recommendation.
- RSA keys should use at least 4096 bits if Ed25519 is unavailable.
- Private key stored in `~/.ssh/id_ed25519` (or `~/.ssh/id_rsa`) — protect with a strong passphrase.
- Public key stored with a `.pub` extension — safe to share.

To enable key-based login on a server, copy your public key to the remote's authorized keys:
```bash
ssh-copy-id mark@10.112.174.85
```
This appends your public key to `~/.ssh/authorized_keys` on the remote system.

## Useful SSH Options
```bash
# Connect on a non-standard port
ssh -p 2222 mark@10.112.174.85

# Use a specific private key
ssh -i ~/.ssh/custom_key mark@10.112.174.85

# Jump through a bastion/jump host to reach an internal server
ssh -J bastion.example.com mark@internal-server

# Local port forwarding (access remote service through local port)
ssh -L 8080:localhost:80 mark@10.112.174.85

# Dynamic port forwarding (SOCKS proxy)
ssh -D 9050 mark@10.112.174.85

# Run a single command without an interactive shell
ssh mark@10.112.174.85 "cat /etc/passwd"
```

## SSH Config File
For frequent connections, create shortcuts in `~/.ssh/config`:
```
Host webserver
    HostName 10.112.174.85
    User mark
    Port 22
    IdentityFile ~/.ssh/id_ed25519

Host internal
    HostName 10.10.10.50
    User admin
    ProxyJump bastion.example.com
```
With this configuration, `ssh webserver` connects using the full parameters automatically.

## Secure File Transfer
| Method | Notes |
|---|---|
| **SFTP** (SSH File Transfer Protocol) | Runs over SSH (port 22). Recommended method for interactive file transfers today — familiar FTP-like interface over an encrypted SSH connection. |
| **SCP** (Secure Copy Protocol) | Traditionally used for simple file copies. Deprecated by OpenSSH in favour of SFTP; still works but shows a deprecation warning on newer systems. |
| **rsync over SSH** | Preferred for transferring large amounts of data or syncing directories — only transfers changed portions of files. |

```bash
# SFTP
sftp mark@10.112.174.85

# SCP: copy from remote to local
scp mark@10.112.174.85:/home/mark/archive.tar.gz ~/

# SCP: copy from local to remote
scp backup.tar.bz2 mark@10.112.174.85:/home/mark/

# rsync
rsync -avz -e ssh /local/directory/ mark@10.112.174.85:/remote/directory/
```

### SSH vs FTPS vs SFTP
- **SFTP** runs over SSH (port 22) — most common choice for secure file transfers today.
- **FTPS** is FTP secured with TLS (port 990 for implicit TLS) — a different protocol from SFTP despite the similar name.
- **SCP** also runs over SSH but is being phased out in favour of SFTP.
- SFTP is generally recommended for most use cases: same authentication and encryption as SSH, simplifying management.

## SSH Hardening Considerations
Settings live in `/etc/ssh/sshd_config`:
- **Disable password authentication** (`PasswordAuthentication no`) — after setting up key-based auth.
- **Disable root login** (`PermitRootLogin no`) — forces users to authenticate as regular users first.
- **Use `AllowUsers` / `AllowGroups`** to restrict which accounts can log in via SSH.
- **Change the default port** — reduces automated scanning noise (security through obscurity, not a strong control on its own).
- **Enable fail2ban** (or similar) to block repeated failed authentication attempts.
- **Use modern key exchange and cipher algorithms** by configuring `KexAlgorithms`, `Ciphers`, and `MACs`.
