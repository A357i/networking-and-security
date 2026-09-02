# Password Attacks

## Authentication Factors
Authentication (proving identity) can be achieved through one or a combination of:
1. **Something you know** — a password or PIN code.
2. **Something you have** — a phone, security key, or smart card.
3. **Something you are** — a fingerprint or facial recognition.

This topic focuses on attacks against the "something you know" factor.

## Why Weak Passwords Persist
Based on the 150 million usernames/passwords leaked from the 2013 Adobe breach, top passwords included `123456`, `password`, `qwerty`, and similar obvious choices. Password habits haven't improved significantly — common passwords in the 2020s still include:
- `123456` and variants (`123456789`, `12345678`, `1234567890`)
- `password` and `Password1`
- `qwerty` and `qwerty123`
- Company/service names with numbers (`Summer2024`, `Welcome1`)
- Sports teams and pop culture references

The availability of massive breach databases makes password attacks more effective than ever — attackers cross-reference leaked credentials across services because many users reuse passwords.

## Types of Password Attacks
| Attack | Description |
|---|---|
| **Password Guessing** | Requires some knowledge of the target (pet's name, birth year, favourite team, children's names). Social media makes gathering this easier than ever. |
| **Dictionary Attack** | Attempts common words from a dictionary/wordlist. Effective because many users choose real words or simple variations. |
| **Brute Force Attack** | Tries all possible character combinations. Exhaustive and time-consuming, but effective against short passwords — search space grows exponentially with length. |
| **Credential Stuffing** | Uses username/password pairs leaked from previous breaches, tried against other services. Exploits password reuse; extremely effective. Automated tools test millions of credentials across many sites simultaneously. |
| **Password Spraying** | Tries one or two common passwords against thousands of accounts (rather than many passwords against one account). Evades account lockout mechanisms. |
| **Hybrid Attacks** | Combine dictionary words with common patterns — e.g. `Summer` + years appended (`Summer2023`, `Summer2024`) or common substitutions (`P@ssw0rd`, `Adm1n!`). |

## Wordlists and Breach Data
- **rockyou.txt** — classic wordlist of breached passwords, found on the AttackBox at `/usr/share/wordlists/rockyou.txt`.
- **SecLists** — collection of multiple wordlists for different purposes, at `/usr/share/seclists/` on many security distributions.
- **CrackStation** — wordlists optimised for password cracking.
- **Breach compilations** — contain billions of real passwords from various data breaches.

Wordlist choice should depend on knowledge of the target — e.g. a French user might use French words; a company's employees might use the company name with years/seasons appended. Custom, tailored wordlists are often more effective than generic ones.

## THC Hydra
Fast, flexible password cracking tool supporting many protocols (FTP, POP3, IMAP, SMTP, SSH, and all HTTP-related auth methods). Automates trying common passwords or wordlist entries against network services.

### General Syntax
```bash
hydra -l username -P wordlist.txt server service
```

### Examples
```bash
# Attack FTP with username mark
hydra -l mark -P /usr/share/wordlists/rockyou.txt 10.112.174.85 ftp

# Alternative syntax (equivalent to above)
hydra -l mark -P /usr/share/wordlists/rockyou.txt ftp://10.112.174.85

# Attack SSH with username frank
hydra -l frank -P /usr/share/wordlists/rockyou.txt 10.112.174.85 ssh

# Attack IMAP with username lazie
hydra -l lazie -P /usr/share/wordlists/rockyou.txt 10.112.174.85 imap

# Attack with a list of usernames (credential stuffing style)
hydra -L users.txt -P passwords.txt 10.112.174.85 ssh
```

### Useful Hydra Options
| Option | Description |
|---|---|
| `-l username` | Single username to attack |
| `-L users.txt` | File containing a list of usernames |
| `-p password` | Single password to try |
| `-P wordlist.txt` | File containing a list of passwords |
| `-s PORT` | Specify a non-default port |
| `-V` or `-vV` | Verbose output showing attempts |
| `-t n` | Number of parallel connections (threads) |
| `-d` | Debug mode for troubleshooting |
| `-f` | Stop after the first valid password found |
| `-w n` | Wait time between connections |

Once the password is found, issue `CTRL-C` to end the process. TryHackMe tasks expect attacks to finish in under five minutes; real-world scenarios take longer, where verbose/debug options help monitor progress.

## Other Password Attack Tools
1. **Medusa** — similar to Hydra but with a modular design; some find it more stable for certain protocols.
2. **Ncrack** — developed by the Nmap project, designed for high-speed parallel authentication testing.
3. **CrackMapExec (CME) / NetExec** — specialises in Windows/Active Directory environments; can spray passwords across SMB, WinRM, LDAP, and other protocols.
4. **Burp Suite Intruder** — useful for attacking web-based login forms where Hydra's HTTP modules may not work correctly.
5. **Hashcat and John the Ripper** — used for cracking password hashes offline rather than attacking live services. If password hashes are obtained (e.g. from a database breach), these tools recover plaintext passwords much faster than attacking a live service.

## Mitigating Password Attacks
| Defence | Notes |
|---|---|
| **Password Policies** | NIST SP 800-63B (U.S. government digital identity guideline) recommends focusing on password length over complexity rules, blocking known compromised passwords, and not requiring regular password changes unless there's evidence of compromise. |
| **Account Lockout** | Temporarily/permanently locks an account after failed attempts. Effective against brute force, but can be bypassed by password spraying or abused for denial of service. |
| **Throttling and Rate Limiting** | Delays responses to login attempts. Tolerable delay for legitimate users, severely hinders automated tools. Sophisticated implementations use exponential backoff. |
| **CAPTCHA** | Requires solving a challenge difficult for machines. Modern CAPTCHAs use behavioural analysis and risk scoring rather than just image recognition. |
| **Multi-Factor Authentication (MFA)** | Requires additional verification beyond the password (authenticator app code, SMS — less secure — or hardware security key). One of the most effective defences against password attacks. |
| **Passwordless Authentication** | Eliminates passwords entirely: **Passkeys** (FIDO2/WebAuthn) use cryptographic keys stored on devices with biometric/PIN verification; **Magic links** sent via email; **Hardware security keys** like YubiKeys. |
| **Breached Password Detection** | Checks passwords against known breach databases during registration/login (e.g. Have I Been Pwned API). |
| **Behavioural Analysis** | Detects anomalies — login attempts from unusual locations, impossible travel scenarios, patterns consistent with automated attacks. |
| **IP-based Controls** | Geofencing, blocking known malicious IPs, requiring additional verification for new devices/locations. |

Defence in depth combines multiple approaches above. For high-security environments, moving toward passwordless authentication eliminates many of these attack vectors entirely.
