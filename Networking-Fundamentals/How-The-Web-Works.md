# How The Web Works

> Covers: DNS in Detail, HTTP in Detail, How Websites Work, Putting it All Together (Pre-Security Section 6)
> Pairs with `Networking-Fundamentals.md` — that file covers DNS record types; this one covers the full resolution + request lifecycle.

---

## 1. DNS in Detail
Resolution order for a query like `example.com`:
1. Check local cache / `/etc/hosts` (see Networking-Fundamentals.md).
2. Query a **recursive resolver** (often your ISP or `8.8.8.8`).
3. Resolver queries a **root server** → get pointed to the `.com` **TLD server**.
4. TLD server points to the domain's **authoritative nameserver**.
5. Authoritative server returns the actual IP (`A` record).
6. Result is cached for the record's **TTL** (time-to-live) to avoid repeating this every time.

## 2. HTTP in Detail
- **Methods**: `GET` (retrieve), `POST` (submit data), `PUT`/`PATCH` (update), `DELETE` (remove).
- **Status codes**: `2xx` success, `3xx` redirect, `4xx` client error (`401` unauthorized, `403` forbidden, `404` not found), `5xx` server error.
- **Headers**: carry metadata — `Host` (which vhost, see Networking-Fundamentals.md), `User-Agent`, `Cookie`, `Content-Type`, `Authorization`.
- **HTTPS**: HTTP wrapped in TLS — encrypts the connection so headers/body aren't visible in transit, but the DNS query to resolve the domain typically still happens in the clear (unless DoH/DoT is used).

## 3. How Websites Work
A basic page load: browser resolves DNS → opens a TCP connection (3-way handshake) → (if HTTPS) TLS handshake → sends an HTTP `GET` request → server responds with HTML → browser parses HTML and fires further requests for CSS/JS/images → page renders.

## 4. Putting It All Together
End-to-end example, typing `https://example.com` into a browser:
1. Browser checks cache/`/etc/hosts`, then queries DNS for `example.com`'s IP.
2. TCP handshake (SYN → SYN-ACK → ACK) establishes a connection to that IP on port 443.
3. TLS handshake negotiates encryption.
4. Browser sends `GET / HTTP/1.1` with a `Host: example.com` header (this is what makes virtual hosting possible on shared IPs).
5. Server's web application processes the request, queries its database if needed, returns HTML.
6. Browser renders the page, issuing more requests for each embedded resource.

This full chain is exactly why DNS, TCP/IP, and HTTP knowledge all compound — a vhost fuzzing attack (Networking-Fundamentals.md) is really just manipulating step 4 while skipping step 1's normal DNS resolution.

---

## Open Questions / To Expand
- [ ] Add TLS handshake detail (cipher suites, certificate validation) once covered
