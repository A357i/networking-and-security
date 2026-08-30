# Operating Systems Basics

> Covers: Operating Systems Introduction, Windows Basics, Linux CLI Basics, Windows CLI Basics, Operating System Security (Pre-Security Section 3)

---

## 1. OS Introduction
The OS manages hardware resources (CPU, memory, storage, I/O) and provides the interface between hardware and applications. Core jobs: process scheduling, memory management, file system management, permission enforcement.

## 2. Windows Basics
- **File structure**: `C:\Windows`, `C:\Users`, `C:\Program Files`.
- **Registry**: hierarchical config database (`HKLM`, `HKCU`, etc.) — a huge target for persistence mechanisms and forensic artifacts.
- **Services**: background processes managed via `services.msc` / `sc` — common privilege-escalation surface (misconfigured service permissions).

## 3. Linux CLI Basics
| Command | Purpose |
|---|---|
| `ls -la` | List files incl. hidden, with permissions |
| `cd` / `pwd` | Navigate / show current dir |
| `cat` / `less` | View file contents |
| `ps aux` | List running processes |
| `chmod` / `chown` | Change permissions / ownership |
| `grep` | Search text/output |
| `man <cmd>` | Manual page for a command |

Filesystem Hierarchy: `/etc` (config, includes `/etc/hosts` — see Networking notes), `/home` (user dirs), `/var/log` (logs), `/bin`/`/usr/bin` (binaries).

## 4. Windows CLI Basics
| Command (CMD) | PowerShell equivalent | Purpose |
|---|---|---|
| `dir` | `Get-ChildItem` | List directory |
| `ipconfig` | `Get-NetIPAddress` | Network config |
| `tasklist` | `Get-Process` | Running processes |
| `whoami` | `whoami` | Current user |
| `net user` | `Get-LocalUser` | User accounts |

PowerShell is scriptable and object-oriented (vs. plain text in CMD) — worth prioritizing since most modern Windows attack tooling is PowerShell-based.

## 5. Operating System Security
- **Users & groups**: principle of least privilege — accounts should have only the access they need.
- **Permissions**: Linux uses `rwx` for owner/group/other; Windows uses ACLs (more granular, per-user).
- **Patching**: unpatched OSes are still one of the most common initial-access vectors in real engagements.
- **Hardening**: disabling unused services, enforcing strong auth, logging — the defensive counterpart to everything you'll try to bypass offensively.

---

## Open Questions / To Expand
- [ ] Expand Windows Registry hive-by-hive once covered in a room
- [ ] Add Linux permission edge cases (SUID/SGID/sticky bit) when relevant to privesc
