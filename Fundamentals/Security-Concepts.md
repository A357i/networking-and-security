# Security Concepts

> Covers: The CIA Triad, Cryptography Concepts, Become a Hacker, Become a Defender (Pre-Security Section 7)

---

## 1. The CIA Triad
| Principle | Meaning | Attack that violates it |
|---|---|---|
| **Confidentiality** | Only authorized parties can access data | Data breach, eavesdropping |
| **Integrity** | Data isn't altered without authorization | Man-in-the-middle tampering, data corruption |
| **Availability** | Systems/data accessible when needed | DoS/DDoS |

Every security control (and every attack) can be framed as protecting or breaking one of these three. Useful lens when writing pentest reports — "this finding violates X" makes impact concrete.

## 2. Cryptography Concepts
- **Symmetric encryption**: same key encrypts and decrypts (AES). Fast, but key distribution is the hard problem.
- **Asymmetric encryption**: public/private key pair (RSA) — public key encrypts, private key decrypts. Solves key distribution, slower, used to exchange symmetric keys (as in TLS).
- **Hashing**: one-way, fixed-length output (SHA-256) — used for integrity checks and password storage (never store plaintext passwords, ideally salted hashes).
- **Encoding vs. Encryption vs. Hashing**: encoding = reversible, no key. Encryption = reversible, needs a key. Hashing = irreversible, no key needed to produce it. Mixing these up is a common early mistake — worth drilling.

## 3. Become a Hacker (Offensive Mindset)
- Curiosity-driven, methodology-first (don't randomly poke — follow recon → enumeration → exploitation → post-exploitation → reporting).
- Legal/ethical boundary: always have explicit written authorization (scope document) before testing anything.

## 4. Become a Defender (Defensive Mindset)
- Assume compromise will happen; focus on detection speed and containment, not just prevention.
- Defense in depth: layered controls so no single failure is catastrophic.

---

## Open Questions / To Expand
- [ ] Expand hashing section with salting/rainbow tables once covered in a cracking room
