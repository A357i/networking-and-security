# Software Basics

> Covers: Data Representation, Data Encoding, Python Simple Demo, JavaScript Simple Demo, Database SQL Basics (Pre-Security Section 4)

---

## 1. Data Representation
- **Bit**: single 0/1. **Byte**: 8 bits.
- **Binary → Decimal → Hex**: same value, different base. Hex is used constantly in security (memory addresses, hashes, color codes in malware analysis, MAC addresses).
- Example: `01000001` (binary) = `65` (decimal) = `0x41` (hex) = `A` (ASCII).

## 2. Data Encoding
- **Encoding ≠ Encryption** — encoding (Base64, URL encoding, hex) is reversible with no key, meant for safe transport/representation, not confidentiality. Never mistake a Base64 blob for "encrypted."
- **ASCII**: maps numbers to characters (0–127).
- **Base64**: encodes binary data as printable text — extremely common in web traffic, auth headers, malware payloads. Learn to recognize it on sight (`=` padding, alphanumeric + `+/`).
- **URL encoding**: `%20` for space, etc. — relevant for crafting/reading web requests and payloads.

## 3. Python (Simple Demo)
Why it matters for you: Python is the default scripting language for exploit dev, automation, and most security tooling.
```python
# minimal example: read a wordlist and print each line
with open("wordlist.txt") as f:
    for line in f:
        print(line.strip())
```

## 4. JavaScript (Simple Demo)
Why it matters: runs client-side in the browser — foundational for understanding XSS, DOM manipulation, and how modern web apps behave.
```javascript
// minimal example: reading a cookie value
console.log(document.cookie);
```

## 5. Database / SQL Basics
```sql
SELECT username, password FROM users WHERE id = 1;
INSERT INTO users (username, password) VALUES ('test', 'pass123');
```
Core concepts: tables, rows, columns, primary keys. This is the foundation you'll need before SQL injection makes sense — SQLi works by breaking out of the intended query structure using unsanitized input.

---

## Open Questions / To Expand
- [ ] Add more Python once used for actual scripting in a room
- [ ] Add SQLi basics here or cross-link to Offensive-Security once covered
