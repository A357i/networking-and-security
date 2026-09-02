# Transport Layer Security (TLS)

## A Brief History
- **SSL (Secure Sockets Layer)** originated with Netscape in 1994; SSL 3.0 released 1996.
- **TLS (Transport Layer Security)** introduced in 1999 as the successor to SSL.
- Version status:
  - SSL 2.0 / 3.0 — deprecated, insecure, never use.
  - TLS 1.0 / 1.1 — deprecated in 2021, no longer supported by major browsers.
  - TLS 1.2 (2008) — widely used, secure with modern cipher suites.
  - TLS 1.3 (2018) — current standard.
- "SSL" is still used colloquially (e.g. "SSL certificate"), but in practice all modern systems use TLS.

## Where TLS Fits in the Network Model
- Cleartext application-layer protocols (HTTP, SMTP, POP3, IMAP, etc.) expose all data to anyone with network access.
- TLS is conceptually added at the **presentation layer** (OSI layer 6), sitting below the application layer and encrypting its data.
- In practice, TLS operates between the transport and application layers rather than mapping cleanly to a single OSI layer — the OSI model is a simplification here.

## Upgrading Protocols with TLS

| Protocol | Default Port | Secured Protocol | Default Port with TLS |
|---|---|---|---|
| HTTP | 80 | HTTPS | 443 |
| FTP | 21 | FTPS | 990 |
| SMTP | 25 | SMTPS | 465 |
| POP3 | 110 | POP3S | 995 |
| IMAP | 143 | IMAPS | 993 |

- TLS is not limited to web/email. DNS can also be secured via TLS:
  - **DoT (DNS over TLS)** — wraps standard DNS traffic in a TLS connection, typically port 853.
  - **DoH (DNS over HTTPS)** — sends DNS queries as HTTPS requests on port 443.
  - Both prevent eavesdropping on DNS lookups (which otherwise reveal browsing activity).

## Implicit TLS vs STARTTLS
- **Implicit TLS**: uses a dedicated port; encryption begins immediately on connection (e.g. 443 for HTTPS, 993 for IMAPS).
- **STARTTLS**: client connects on the standard cleartext port (e.g. 25 for SMTP), then issues a `STARTTLS` command to upgrade the existing connection to TLS. Common for email protocols. For SMTP, port 587 (submission) with STARTTLS is the recommended client configuration.
- Implicit TLS is generally preferred — STARTTLS can be vulnerable to downgrade attacks if not properly implemented (an attacker performing MITM can strip the STARTTLS command, forcing the connection to stay in cleartext).

## How HTTPS Works
- Plain HTTP: (1) establish TCP connection, (2) send HTTP requests (GET, POST, etc.).
- HTTPS adds a step between TCP and HTTP: (1) establish TCP connection, (2) **establish TLS connection**, (3) send HTTP requests.

## The TLS Handshake (simplified TLS 1.2 / SSL RFC 6101 overview)
1. **ClientHello** — client sends supported TLS versions, cipher suites, and a random value.
2. **ServerHello + Certificate + ServerKeyExchange + CertificateRequest\* + ServerHelloDone** — server responds with chosen parameters, its certificate (signed by a CA), and its own random value.
3. **Key Exchange** — client and server derive the shared secret key. The exact mechanism depends on the chosen cipher suite (Certificate\*, ClientKeyExchange, CertificateVerify\*, [ChangeCipherSpec], Finished).
4. **[ChangeCipherSpec] / Finished** — both sides confirm the handshake completed and switch to encrypted communication.

### TLS 1.3 Improvements
- **Faster handshake**: 1-RTT (one round trip) vs 2-RTT for TLS 1.2; supports 0-RTT resumption for returning clients (with some security trade-offs).
- **Forward secrecy by default**: all TLS 1.3 cipher suites provide forward secrecy — a compromised private key in the future cannot decrypt past recorded sessions.
- **Simplified cipher suites**: outdated/insecure algorithms removed, eliminating weak configuration choices.
- **Encrypted handshake**: more of the handshake is encrypted, revealing less to observers.

Once the handshake completes, client and server share a secret key that a third party monitoring the channel cannot discover — all further communication is encrypted with it.

## Certificates and Trust
- TLS relies on public certificates signed by **Certificate Authorities (CAs)** trusted by the system.
- A certificate viewer shows:
  1. **To whom is the certificate issued?** (Common Name, Organization — the entity using the certificate)
  2. **Who issued the certificate?** (the CA that signed it)
  3. **Validity period** — an expired certificate should not be trusted.
- The browser handles chain-of-trust verification automatically, confirming the server's identity and that MITM has not occurred.

### Modern Certificate Ecosystem
- **Let's Encrypt** (launched 2015) provides free, automated TLS certificates — removed the cost barrier to HTTPS adoption. HTTPS traffic share grew from under 50% (2015) to over 95% today.
- **Certificate Transparency (CT)** requires CAs to log all issued certificates to public, auditable logs. Browsers check these logs and reject certificates not properly logged — makes fraudulent certificates much harder to obtain undetected.
- **Short-lived certificates** are increasingly common. Let's Encrypt certs are valid for only 90 days (some orgs use certs valid for hours/days), reducing the exposure window if a cert is compromised.
- **ACME (Automated Certificate Management Environment)** is the protocol used by Let's Encrypt and other CAs to automate issuance/renewal (e.g. via Certbot).

### Testing TLS Configurations
| Tool | Use case |
|---|---|
| `testssl.sh` | CLI tool checking supported protocols, cipher suites, vulnerabilities. Best for detailed assessments, especially internal/non-public systems. |
| `sslyze` | Python tool for analysing SSL/TLS configs; useful for automation and CI/CD integration. |
| SSL Labs (ssllabs.com) | Web-based analysis of public-facing HTTPS servers. Quickest option for a one-off assessment. |
| `nmap` ssl-enum-ciphers | Nmap script enumerating supported cipher suites as part of a broader port scan. |

Common issues to check for: support for deprecated protocols (TLS 1.0/1.1), weak cipher suites, missing forward secrecy, certificate problems (expired, misissued, untrusted CA).
