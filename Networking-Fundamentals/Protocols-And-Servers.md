# Protocols and Servers

> Covers: TryHackMe "Protocols and Servers" room (in progress — Telnet, HTTP, FTP done; SMTP/POP3/IMAP still to come per room description).
> Focus of this room: talking to application-layer protocols manually (via Telnet) instead of through a GUI, to see what's actually happening under the hood — and the cleartext insecurities that come with it.

---

## 0. Why Learn These (Often Decades-Old) Protocols
1. **Still in use** — most public-facing services now run encrypted versions (HTTPS, SFTP, IMAPS), but the underlying protocol mechanics are identical. The commands sent over HTTPS are the same HTTP commands, just wrapped in TLS.
2. **You'll encounter cleartext versions in the field** — legacy systems, internal networks, IoT devices, and misconfigured services often still run unencrypted protocols. Recognizing and exploiting that is a core pentesting skill.
3. **Understanding the protocol explains the attacks** — knowing how SMTP works explains why email spoofing is possible; knowing HTTP makes web app vulnerabilities click.

**Core insecurity theme**: when credentials are sent in cleartext, anyone with access to the network traffic can capture them. Acceptable when these protocols were designed for trusted academic networks decades ago — a serious vulnerability on modern networks. (Follow-up room "Protocols and Servers 2" covers securing these with TLS, plus sniffing/MITM/password attacks in more depth.)

---

## 1. Telnet
- **Application-layer protocol** for connecting to a virtual terminal on another computer — log in remotely, get a terminal/console, run programs and admin tasks.
- Flow: connect → prompted for username → prompted for password → correct auth grants a remote shell. **All of this is unencrypted.**
- **Telnet today**: almost entirely replaced by **SSH** for interactive remote access. You're unlikely to find it on a modern, properly-configured system — but it still turns up in legacy systems/older network gear (routers, switches, industrial controllers), embedded/IoT devices, internal networks where security was never prioritized, and misconfigured systems. **Finding an open port 23 during a pentest is a significant finding** — it signals a legacy system or a security misconfiguration.
- **Telnet client as a testing tool**: even though telnet *servers* are rare now, the telnet *client* is still genuinely useful — because it operates over TCP, it can connect to any TCP port and let you manually interact with text-based protocols (this is exactly the banner-grabbing technique from `Offensive-Security/Reconnaissance.md`, reused here to speak HTTP/FTP by hand).
- **Why it's insecure, concretely**: even though your password isn't echoed to the screen while typing, that protection is cosmetic — the credentials still travel the network in cleartext. A packet capture (Wireshark "Follow TCP Stream") on a telnet session shows the username and password in plain ASCII, fully readable. Anyone on the same network segment, anyone who's compromised a router/switch on the path, a malicious insider, or a successful MITM attacker can read it.
- **Secure alternative**: **SSH** — encrypts all traffic including credentials, and has been the standard for remote CLI access for over two decades.
- Default port: **23**.

## 2. HTTP
- The protocol used to transfer web pages: browser → server for HTML/images/files, plus form submissions and uploads. Any time you browse the web, you're using HTTP.
- **HTTP vs HTTPS**: HTTP is cleartext — anyone with network access can read everything transferred, including credentials and personal data. HTTPS wraps HTTP in TLS. Modern browsers flag plain HTTP as "Not Secure" and block some features (geolocation, camera) on non-HTTPS sites entirely. Still worth understanding HTTP directly because: the commands/structure are identical under HTTPS (just encrypted); you'll hit it on internal pentests and legacy systems; it underpins understanding web vulnerabilities; and tools like Burp Suite decrypt HTTPS specifically to show you this raw HTTP underneath.
- **Manually sending HTTP requests** (since it's cleartext, Telnet or Netcat can act as a crude "browser"):
  1. Connect to port 80: `telnet <target> 80`
  2. Send a request line: `GET /index.html HTTP/1.1` (or `GET / HTTP/1.1` for the default page)
  3. Provide a Host header: `host: <value>`, then press Enter **twice**
  - The response includes the page content *plus* information a normal browser hides from view.
- **Information revealed in headers**: a `Server: nginx/1.18.0 (Ubuntu)` header reveals both the web server software+version **and** the OS. Valuable during recon because specific versions may have known CVEs, OS info helps tailor further attacks, and even just the web server family narrows down likely attack vectors. Security-conscious admins configure servers to suppress this — finding detailed version info in headers during a pentest is worth flagging.
- **Web servers**: **Nginx** (now the most widely used, known for performance under concurrent connections, free/open-source), **Apache** (still extremely common, highly configurable, huge module ecosystem, free/open-source), **IIS** (Microsoft's server, common in Windows enterprise environments, requires a Windows Server license). Also notable: LiteSpeed, Caddy (automatic HTTPS built in), Node.js (for JS-based apps).
- **Browsers**: Chrome (dominant market share), Safari (default on macOS/iOS), Edge (Chromium-based, replaced IE), Firefox (open-source, privacy-focused — often preferred for pentesting/security research due to its DevTools and add-on ecosystem).
- **HTTP versions**: **HTTP/1.1** — the text-based workhorse for decades, and the most accessible for manual/Telnet-based testing since it's human-readable. **HTTP/2** — adds multiplexing (multiple requests over one connection), header compression, server push; binary rather than text, so it can't be manually driven over Telnet. **HTTP/3** — runs on **QUIC** (built on UDP instead of TCP), better performance especially on unreliable networks, increasingly common on major sites.

## 3. FTP (File Transfer Protocol)
- One of the earliest internet protocols, built for efficient file transfer between different systems. Still around today, though replaced by secure alternatives for most modern use.
- **Modern replacements** (because plain FTP sends credentials and data in cleartext):
  - **SFTP** (SSH File Transfer Protocol) — runs over SSH, port 22, encrypts everything. The most common replacement.
  - **FTPS** (FTP Secure) — adds TLS to FTP: implicit TLS on port 990, or STARTTLS on port 21.
  - **SCP** (Secure Copy Protocol) — also over SSH, being deprecated in favor of SFTP.
- **Where plain FTP still shows up**: legacy systems/older apps, anonymous FTP servers for public distribution, internal networks that never implemented encryption, embedded/network devices with limited capability, misconfigured servers. **Finding an FTP server — especially one allowing anonymous login — is a common, significant pentest finding.**
- **Manually interacting with FTP** (same cleartext-exploiting logic as HTTP): FTP listens on **port 21** by default, so point Telnet there instead of port 23.
  - Login: `USER <username>` → server asks for password → `PASS <password>` → `230 Login successful.`
  - `STAT` — additional server status info (also explicitly confirms **"Control connection is plain text"** and **"Data connections will be plain text"** — i.e. everything, including credentials, is unencrypted).
  - `SYST` — shows the System Type of the target (e.g. `215 UNIX Type: L8`).
  - `PASV` — switches to passive mode.
- **Active vs. Passive mode**:
  - **Active**: data sent over a separate channel *originating from the server's port 20*; the server initiates the data connection back to the client. Often fails when the client is behind a firewall/NAT.
  - **Passive**: data sent over a channel originating from a client port above 1023; the **client** initiates both connections. More firewall-friendly — the default for most modern FTP clients.
- `TYPE A` switches file transfer to ASCII mode; `TYPE I` switches to binary. **A file transfer can't actually complete over a simple client like Telnet** — FTP opens a *second, separate* TCP connection just for the data transfer, and Telnet has no way to handle that second channel.
- **How the file transfer actually works**: the FTP client connects to the server on **TCP 21** (the control channel — all commands go here). When the client requests a file, a **second TCP connection** is opened just for that transfer (the data channel). This dual-connection architecture is exactly why FTP is often problematic with firewalls.
- **Using a real FTP client**: after logging in you get an `ftp>` prompt — `ls` lists files, `ascii` switches to ASCII mode (for text files), `get <filename>` downloads a file by opening the separate data channel automatically.
- **Anonymous FTP**: some servers allow login with username `anonymous` (or `ftp`) and *any* string (often an email-looking one) as the password — historically used for public file distribution. **Always try anonymous login when you find an FTP server during a pentest.** Anonymous FTP can expose accidentally-shared sensitive files, config backups, or — if write access is misconfigured — a place to upload malicious files.
- **Common FTP servers**: **vsftpd** (Very Secure FTP Daemon — one of the most common on Linux), **ProFTPD** (highly configurable/modular), **Pure-FTPd** (security/simplicity focused), IIS (includes FTP on Windows). **Clients**: the console client on Linux, or GUI tools like **FileZilla** — note that major browsers have dropped FTP support entirely in recent years.
- **Security implications**: because FTP sends credentials, commands, *and* file contents in cleartext, anyone capturing traffic sees usernames/passwords, transferred file contents, directory structure, and the commands themselves. If FTP must be used at all, restrict it to isolated networks or use FTPS — for most purposes, SFTP over SSH is the recommended replacement.

## 4. SMTP (Simple Mail Transfer Protocol)
- **Email delivery components**: **MUA** (Mail User Agent — the email client, e.g. Thunderbird, Outlook, webmail) → **MSA** (Mail Submission Agent — receives mail from the MUA, checks for errors, forwards it) → **MTA** (Mail Transfer Agent — routes and delivers mail between servers) → **MDA** (Mail Delivery Agent — stores the email in the recipient's mailbox for retrieval).
- **The five-step flow**: (1) MUA connects to the MSA to submit a message; (2) MSA checks for errors, forwards to the MTA (often hosted on the same server); (3) the sending MTA delivers to the recipient's MTA; (4) that MTA commonly also functions as the MDA; (5) the recipient retrieves the mail from the MDA using their own MUA (via POP3 or IMAP).
- **Postal analogy**: you (MUA) hand mail to a post office employee (MSA) who checks it before your local post office (MTA) accepts it; the local post office sends it to the destination country's post office (MTA); that office delivers to the recipient's mailbox (MDA); the recipient (MUA) checks their mailbox periodically.
- **Email protocols**: **SMTP** for *sending*; **POP3** or **IMAP** for *receiving*.
- **SMTP ports and encryption**:
  - **Port 25** — traditional server-to-server port (MTA to MTA). Often blocked by ISPs for residential connections to prevent spam. Encryption is optional here, negotiated via `STARTTLS`.
  - **Port 587** — the **submission port**, used by mail clients (MUA) to submit messages to their mail server (MSA). The recommended port for sending email; requires authentication; TLS negotiated via `STARTTLS`.
  - **Port 465** — originally for SMTPS (implicit TLS), deprecated, then reinstated. TLS begins immediately on connection here, no negotiation needed.
- **Manually sending email with Telnet** (plain port 25, to see cleartext protocol commands):
  ```
  telnet <target> 25
  helo <hostname>     (or: ehlo <hostname> for Extended SMTP)
  mail from: <sender>
  rcpt to: <recipient>
  data
  <message body>
  .                    (a period alone on its own line ends the message)
  quit
  ```
- **Email spoofing and why it works**: the `mail from:` address is specified manually by the client, and the server accepts it **without verifying the sender actually controls that address**. SMTP was designed in an era of trusted networks and has no built-in sender-identity verification — this is exactly why phishing emails can convincingly claim to be from a legitimate address. The protocol itself doesn't stop anyone from claiming to be anyone.
- **Security implications**: email remains the primary vector for phishing; misconfigured mail servers can become open relays for spam; cleartext SMTP exposes email content and credentials to sniffing; understanding SMTP is what makes email-header analysis during incident response make sense. During pentests: test for open-relay configuration, attempt email spoofing to assess security awareness, and analyze headers to trace the origin of suspicious messages. Modern mitigations — **SPF, DKIM, DMARC** — exist specifically to combat spoofing (covered in "Protocols and Servers 2").

## 5. POP3 (Post Office Protocol v3)
- Protocol used to **download** email messages from an MDA server: the client connects, authenticates, downloads new messages, then (optionally) deletes them from the server.
- **Ports and encryption**: **Port 110** — default, cleartext (some servers support upgrading via `STLS`, the POP3 equivalent of SMTP's `STARTTLS`). **Port 995** — POP3S, implicit TLS, encrypted from the start. Most providers today require/strongly encourage 995; plaintext POP3 still turns up on internal networks, legacy systems, or misconfigured servers.
- **Manually interacting via Telnet** (port 110):
  ```
  telnet <target> 110
  USER <username>
  PASS <password>
  STAT              → +OK <n> <size>   (n = message count, size in octets — per RFC 1939)
  LIST              → lists messages on the server
  RETR 1            → retrieves message 1
  QUIT
  ```
- **Common commands**: `USER`/`PASS` (identify/authenticate), `STAT` (message count + total size), `LIST` (list messages with sizes), `RETR n` (retrieve message n), `DELE n` (mark n for deletion), `RSET` (unmark deletions), `QUIT` (end session, apply deletions).
- **"Download and delete" behavior** (POP3's default): emails end up stored **locally**, not on the server — once downloaded, a message is only accessible from that device; if the device is lost, the email is gone unless backed up elsewhere; server-side storage stays minimal. This default can be changed to leave copies on the server, but multiple clients accessing the same account over POP3 is inherently awkward — there's no sync, so read/unread status and message state diverge across devices.
- **POP3 vs. IMAP — when POP3 still makes sense**: accessing mail offline / with unreliable connectivity, minimizing server-side storage, single-device access, or local archiving. For everything else, IMAP has largely replaced it because of synchronization.
- **Security implications**: finding an exposed POP3 server (especially on 110) during a pentest is an opportunity — credentials sent over cleartext POP3 are capturable via network sniffing, password attacks can target POP3 auth directly, and mailbox access can reveal sensitive info, reused credentials, or password-reset links. Capturing the `USER`/`PASS` commands in traffic = valid, potentially-reusable credentials.

## 6. IMAP (Internet Message Access Protocol)
- More sophisticated than POP3 — keeps email **synchronized across multiple devices/clients**. Mark a message read on your phone, and that state is saved on the IMAP server (the MDA) and reflected on your laptop next sync.
- **Why IMAP became the standard**: people check email from phones, laptops, tablets, and browsers, switching constantly. IMAP's server-side model handles this naturally — emails remain on the server and are accessible from any device; read/unread status, folders, and flags stay synced everywhere; deleting on one device removes it everywhere; search happens server-side without downloading everything. This is the direct opposite of POP3's per-device, download-and-delete model.
- **Ports and encryption**: **Port 143** — default, cleartext (many servers support upgrading via `STARTTLS`). **Port 993** — IMAPS, implicit TLS from the start. Most providers require IMAPS today — Gmail, Outlook, and Yahoo have disabled plaintext IMAP entirely — but plaintext IMAP still turns up on internal mail servers or legacy systems.
- **Manually interacting via Telnet** (port 143): every IMAP command is preceded by a **unique tag** (`c1`, `c2`, `c3`...) so the client can match each reply to its request.
  ```
  telnet <target> 143
  c1 LOGIN <username> <password>
  c2 LIST "" "*"          → lists mailbox folders (INBOX, Trash, Drafts, Sent, ...)
  c3 EXAMINE INBOX        → opens INBOX read-only, shows FLAGS/EXISTS/RECENT/UIDVALIDITY
  c4 LOGOUT
  ```
- **Reading the server's initial response**: it advertises a `CAPABILITY` list — useful recon info. `IMAP4rev1` = the IMAP version; `STARTTLS` = server supports upgrading to an encrypted connection; `IDLE` = server can push new-mail notifications; `ACL` = access-control-list support. The `LIST` response reveals the folder structure and confirms successful auth.
- **Common commands**: `LOGIN user pass` (authenticate), `LIST "" "*"` (list folders), `SELECT folder` (open read/write), `EXAMINE folder` (open read-only), `FETCH n BODY[]` (retrieve message n), `SEARCH criteria` (server-side search), `STORE n +FLAGS (\Seen)` (mark n as read), `LOGOUT` (end session).
- **IMAP vs. webmail**: web interfaces (Gmail, Outlook.com) use HTTPS to secure the browser↔provider connection, but the underlying mail storage still runs on IMAP concepts — and plenty of users run a traditional mail client alongside webmail. IMAP stays relevant because enterprise environments often run their own mail servers with IMAP access, pentesters may encounter IMAP services directly during assessments, and mobile/desktop clients still use IMAP extensively.
- **Security implications**: IMAP sends login credentials in cleartext — a captured `LOGIN <user> <pass>` command is a fully valid credential pair. Compromised IMAP access is *particularly* valuable to an attacker beyond just the credentials: **persistent access** (unlike POP3, mail stays server-side, so stolen creds keep working going forward); **historical data** (the entire mailbox history — potentially years of sensitive communication — is accessible); **password-reset abuse** (search the mailbox for reset emails to pivot into other accounts); **business email compromise** (invoice fraud, impersonation, data theft); **lateral movement** (mail often contains credentials and internal documentation useful for further attacks).

---

## 7. Room Summary / Protocol Reference
**Key takeaway**: every protocol in this room transmits data in cleartext by default, including authentication credentials — if traffic isn't encrypted, anyone with network access can capture usernames, passwords, email content, and file transfers. The Telnet *client* isn't used as a server protocol anymore, but it remains a genuinely practical tool for manually interacting with any text-based protocol on any TCP port — used here to demonstrate HTTP, FTP, SMTP, POP3, and IMAP directly. For every cleartext protocol here, a secure alternative exists and should be used in modern deployments.

Other protocols exist that attackers also care about (e.g. **SMB** — shared file/printer access — is a common attractive target) but weren't this room's scope.

| Protocol | TCP Port | Purpose | Data Security | Secure Alternative | Secure Port |
|---|---|---|---|---|---|
| FTP | 21 | File Transfer | Cleartext | FTPS or SFTP | 990 (FTPS), 22 (SFTP) |
| HTTP | 80 | Worldwide Web | Cleartext | HTTPS | 443 |
| IMAP | 143 | Email (MDA) | Cleartext | IMAPS | 993 |
| POP3 | 110 | Email (MDA) | Cleartext | POP3S | 995 |
| SMTP | 25 | Email (MTA) | Cleartext | SMTPS or SMTP+STARTTLS | 465 (SMTPS), 587 (Submission) |
| Telnet | 23 | Remote Access | Cleartext | SSH | 22 |

---

## Open Questions / To Expand
- [x] Room complete — Telnet, HTTP, FTP, SMTP, POP3, IMAP all covered
- [ ] "Protocols and Servers 2" (next room) covers attacks against these protocols/servers (sniffing, MITM, password attacks) + TLS mitigations (SPF/DKIM/DMARC for SMTP) — add once covered
