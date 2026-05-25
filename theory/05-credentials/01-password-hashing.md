# 01 — Password Hashing: Theory and Cracking

> You have already seen password hashes twice in this lab:
> `/etc/shadow` on Metasploitable2 (MD5 hashes) and Kerberos
> service tickets (RC4-HMAC encrypted with NTLM hashes).
> This document explains what hashes are, why they're attackable,
> and how hashcat cracks them — which is essential for Phase 4.

---

## Table of Contents
1. [What Is a Hash?](#1-what-is-a-hash)
2. [Hash Properties](#2-hash-properties)
3. [Common Hash Algorithms](#3-common-hash-algorithms)
4. [Password Storage: Then and Now](#4-password-storage-then-and-now)
5. [Linux: /etc/shadow](#5-linux-etcshadow)
6. [Windows: NTLM Hash](#6-windows-ntlm-hash)
7. [Kerberos: RC4-HMAC and AES](#7-kerberos-rc4-hmac-and-aes)
8. [How to Identify a Hash](#8-how-to-identify-a-hash)
9. [Cracking Methods](#9-cracking-methods)
10. [Hashcat: Practical Guide](#10-hashcat-practical-guide)
11. [rockyou.txt: The Standard Wordlist](#11-rockyowtxt-the-standard-wordlist)
12. [Defense](#12-defense)

---

## 1. What Is a Hash?

A **hash** is the output of a mathematical function that transforms
any input into a fixed-length string. It is a **one-way** operation.

```
Input (any length)     →    Hash Function    →    Output (fixed length)

"password"             →    MD5              →    5f4dcc3b5aa765d61d8327deb882cf99
"password1"            →    MD5              →    7c6a180b36896a0a8c02787eeafb0e4c
"a"                    →    MD5              →    60b725f10c9c85c70d97880dfe8191b3
"War and Peace (full)" →    MD5              →    some 32-char hex string
```

**Key property:** The same input always produces the same output.
Different inputs produce different outputs (with rare exceptions — collisions).

**Linux analogy:** Think of a hash as a fingerprint. You can verify
a fingerprint belongs to a person without knowing who they are first.

---

## 2. Hash Properties

### One-Way (Non-Reversible)
```
"password" → hash → 5f4dcc3b5aa765d61d8327deb882cf99   ✅
5f4dcc3b5aa765d61d8327deb882cf99 → ??? → "password"    ❌ mathematically impossible
```

You cannot "decrypt" a hash. You can only:
- Hash a candidate and compare
- Find a collision (different input, same output)

### Deterministic
```
SHA256("hello") always = 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
On any machine, any time, forever.
```

### Avalanche Effect
Tiny input change → completely different output:
```
MD5("password")  = 5f4dcc3b5aa765d61d8327deb882cf99
MD5("Password")  = dc647eb65e6711e155375218212b3964
MD5("password1") = 7c6a180b36896a0a8c02787eeafb0e4c
```

This is intentional — prevents guessing nearby inputs from a known hash.

### Fixed Length
Output length is always the same regardless of input length:
```
MD5:    always 32 hex chars   (128 bits)
SHA1:   always 40 hex chars   (160 bits)
SHA256: always 64 hex chars   (256 bits)
NTLM:   always 32 hex chars   (128 bits)
bcrypt: always 60 chars       (includes salt + cost factor)
```

---

## 3. Common Hash Algorithms

### MD5 (Message Digest 5) — 1991, BROKEN
```
Length:  32 hex characters
Example: 5f4dcc3b5aa765d61d8327deb882cf99
Status:  Cryptographically broken — collisions found in seconds
         Still very fast → bad for passwords (cracks in minutes)
Used in: Old Linux /etc/shadow ($1$), old web apps, file checksums
```

**We saw this on Metasploitable2:** `/etc/shadow` showed `$1$` prefix
= MD5-crypt. Crackable with a modern GPU in hours for common passwords.

### SHA1 — 1995, WEAK
```
Length:  40 hex characters
Example: 5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8
Status:  Collision attacks demonstrated — not safe for passwords
Used in: Old certificates (deprecated), Git object IDs
```

### SHA256 / SHA512 — 2001, STILL SAFE FOR CHECKSUMS
```
SHA256 Length: 64 hex characters
SHA512 Length: 128 hex characters
Status: No known practical attacks — but too fast for password storage
Used in: File integrity, TLS certificates, modern Linux /etc/shadow ($5$=SHA256, $6$=SHA512)
```

**Important:** SHA256 is safe for checksums but still crackable for
passwords because it's designed to be fast. A GPU can compute
billions of SHA256 hashes per second.

### bcrypt — 1999, GOOD FOR PASSWORDS
```
Length:  60 characters
Example: $2b$12$EXRkfkdmXn2gzds2SSitu.MW9.1BeZToL99Q9T5dPDfKGSMqAQwGm
Format:  $2b$ [algorithm] $12$ [cost factor] [22-char salt] [31-char hash]
Status:  Intentionally slow — designed for passwords
Used in: Modern web apps, Linux /etc/shadow ($2b$)
```

The `12` is the cost factor — it means 2^12 = 4,096 iterations.
Making it slow by design: ~100ms per attempt. On a GPU: still 100ms.
**Cracking bcrypt is ~10,000x slower than cracking MD5.**

### NTLM — Microsoft's Password Hash
```
Length:  32 hex characters (same as MD5)
Example: 8846f7eaee8fb117ad06bdd830b7586c
Algorithm: MD4(UTF-16LE(password))
Status:  Fast — similar cracking speed to MD5
Used in: Windows local accounts, Active Directory (stored in NTDS.dit)
Key property: NO SALT — same password always produces same hash
```

**Critical:** NTLM hashes have no salt. This means:
1. Rainbow tables work against them
2. If two users have the same password, they have the same hash
3. The hash itself can authenticate — you don't need to crack it (Pass-the-Hash)

---

## 4. Password Storage: Then and Now

### The wrong way (still common in old systems)
```
Database stores: username | password (plaintext)
If breached: all passwords immediately exposed
```

### The slightly better wrong way
```
Database stores: username | MD5(password)
If breached: attacker has hashes → cracks common passwords in hours
```

### The correct way
```
Database stores: username | bcrypt(password + random_salt)
Salt: unique random value per user, stored with hash
If breached: each hash must be cracked individually (100ms+ per attempt)
With GPU: ~10,000 attempts/second → "password123" cracked in seconds
          "Tr0ub4dor&3" (XKCD-style): years
```

### Salting — Why It Matters
Without salt:
```
User A: password → 5f4dcc3b...  ← same hash
User B: password → 5f4dcc3b...  ← reveals they have same password
Attacker: pre-compute all MD5s for common passwords (rainbow table)
          → lookup 5f4dcc3b → instantly knows "password"
```

With salt:
```
User A: password + $randomsalt1$ → unique hash
User B: password + $randomsalt2$ → different unique hash
Attacker: must crack each hash individually — rainbow tables useless
```

**NTLM has no salt.** This is one of its critical weaknesses.

---

## 5. Linux: /etc/shadow

Format of each line in `/etc/shadow`:
```
username:$algorithm$salt$hash:lastchanged:min:max:warn:inactive:expire

root:$1$aaa.bbb.$hashed_value:17000:0:99999:7:::
         ↑  ↑        ↑
         │  salt     hash
         algorithm
```

### Algorithm prefixes
| Prefix | Algorithm | Security |
|--------|-----------|----------|
| `$1$` | MD5-crypt | ❌ Broken, fast |
| `$2b$` | bcrypt | ✅ Strong, slow |
| `$5$` | SHA-256-crypt | ⚠️ Medium (better than MD5) |
| `$6$` | SHA-512-crypt | ⚠️ Medium (better than MD5) |
| `$y$` | yescrypt | ✅ Strong, recommended |

**What we saw on Metasploitable2:**
```
root:$1$igQG1QLq$F9oUzwGiO1HZLV0ADVxom.:17095:0:99999:7:::
      ↑
      $1$ = MD5-crypt — crackable with hashcat -m 500
```

### Cracking /etc/shadow hashes
```bash
# hashcat mode 500 = md5crypt ($1$)
hashcat -m 500 shadow_hashes.txt /usr/share/wordlists/rockyou.txt

# hashcat mode 1800 = sha512crypt ($6$)
hashcat -m 1800 shadow_hashes.txt /usr/share/wordlists/rockyou.txt

# hashcat mode 3200 = bcrypt ($2b$)
hashcat -m 3200 shadow_hashes.txt /usr/share/wordlists/rockyou.txt
# Note: bcrypt is extremely slow even with GPU
```

---

## 6. Windows: NTLM Hash

### How NTLM hashes are generated
```
Password (Unicode):  "Password1"
Encoding:            UTF-16 Little Endian
Hash function:       MD4

MD4(UTF-16LE("Password1")) = 64f12cddaa88057e06a81b54e73b949b
```

**MD4** is even weaker than MD5. NTLM was designed in the early 1990s.

### Where NTLM hashes are stored

**Windows workstation (local accounts):**
```
C:\Windows\System32\config\SAM     ← Security Account Manager
Encrypted with SYSKEY from SYSTEM hive
Extractable with: mimikatz, impacket secretsdump
```

**Active Directory domain:**
```
C:\Windows\NTDS\NTDS.dit           ← AD database
Contains NTLM hashes for ALL domain users
Extractable with: DCSync attack, NTDS.dit dump
```

### NTLM hash format in dumps
```
username:RID:LM_hash:NT_hash:::

Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
                   ↑                                 ↑
                   LM hash (usually empty/disabled)  NT (NTLM) hash ← the one you use

alice.rossi:1104:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
```

The `aad3b435...` LM hash means "LM disabled" — this is normal and expected on modern Windows.

### Cracking NTLM
```bash
# hashcat mode 1000 = NTLM
hashcat -m 1000 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt

# Speed on modern GPU (RTX 4090):
# MD5:  ~164 billion hashes/second
# NTLM: ~300 billion hashes/second  ← extremely fast
# bcrypt: ~184,000 hashes/second    ← 1,600,000x slower than NTLM
```

**Result:** An 8-character password using rockyou.txt is cracked in seconds.
A complex 12-character random password would take years.

---

## 7. Kerberos: RC4-HMAC and AES

When you Kerberoast (request a TGS ticket), the ticket is encrypted with
the service account's password using one of these algorithms:

### RC4-HMAC (Mode 13100 in hashcat)
```
Older encryption type, still allowed by default in AD
Uses NTLM hash as key
Fast to crack — similar speed to NTLM
Hashcat mode: 13100 (Kerberos 5 TGS-REP etype 23)
```

### AES-256-CTS-HMAC-SHA1-96 (Mode 19700)
```
Modern encryption type
Uses AES-256 derived from password
Slower to crack than RC4
Hashcat mode: 19700 (Kerberos 5 TGS-REP etype 18)
```

### In our Phase 4 lab
```bash
# GetUserSPNs.py will output something like:
$krb5tgs$23$*svc_sql$HOMELAB.LOCAL$homelab.local/svc_sql*$a1b2c3...

# $23 = etype 23 = RC4-HMAC → hashcat mode 13100
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt

# If etype 18 (AES) appears:
# $krb5tgs$18$... → hashcat mode 19700
```

---

## 8. How to Identify a Hash

When you find a hash and don't know the algorithm:

```bash
# Use hashid (pre-installed on Kali)
hashid '5f4dcc3b5aa765d61d8327deb882cf99'
# [+] MD2
# [+] MD5    ← most likely
# [+] MD4

hashid '$1$igQG1QLq$F9oUzwGiO1HZLV0ADVxom.'
# [+] MD5 Crypt    ← definitive (prefix $1$)

hashid '$krb5tgs$23$...'
# [+] Kerberos 5 TGS-REP etype 23

# Use hash-identifier (alternative tool)
hash-identifier

# Visual identification by length:
# 32 chars  = MD5 or NTLM
# 40 chars  = SHA1
# 64 chars  = SHA256
# 128 chars = SHA512
# 60 chars starting with $2b$ = bcrypt
```

---

## 9. Cracking Methods

### Dictionary Attack (most common)
```
Take a wordlist, hash each word, compare with target hash.
Fast. Works when password is a real word or common pattern.

rockyou.txt has 14 million entries → cracked in seconds for MD5/NTLM
```

### Rule-Based Attack
Apply transformation rules to wordlist entries:
```bash
# Common rules:
# Append numbers: password → password1, password123
# Capitalize: password → Password
# Leetspeak: password → p4ssw0rd
# Add symbols: password → password!

hashcat -m 1000 hash.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

### Brute Force
Try every possible combination:
```bash
# All 6-character lowercase passwords
hashcat -m 1000 hash.txt -a 3 ?l?l?l?l?l?l

# ?l = lowercase letter
# ?u = uppercase letter
# ?d = digit
# ?s = special character
# ?a = all of the above

# 8-char mixed: ?u?l?l?l?d?d?d?s
# At NTLM speed (300B/sec): seconds to minutes
```

### Hybrid Attack
Combine wordlist + brute force:
```bash
# Wordlist + 2 digits appended
hashcat -m 1000 hash.txt -a 6 rockyou.txt ?d?d
# password → password00, password01, ..., password99
```

### Rainbow Tables
Pre-computed tables of hash → plaintext mappings.
Effective against unsalted hashes (NTLM).
Defeated by salted hashes (bcrypt, SHA-512-crypt).

---

## 10. Hashcat: Practical Guide

```bash
# Basic syntax
hashcat -m [mode] [hash_file] [wordlist]

# Common modes
-m 0     # MD5
-m 100   # SHA1
-m 1400  # SHA256
-m 500   # md5crypt ($1$) ← /etc/shadow MD5
-m 1800  # sha512crypt ($6$) ← /etc/shadow SHA512
-m 3200  # bcrypt ($2b$)
-m 1000  # NTLM ← Windows password hashes
-m 13100 # Kerberos 5 TGS-REP etype 23 ← Kerberoasting
-m 18200 # Kerberos 5 AS-REP etype 23 ← AS-REP Roasting
-m 19700 # Kerberos 5 TGS-REP etype 18 (AES)

# Attack modes
-a 0    # Dictionary (default)
-a 1    # Combination (two wordlists)
-a 3    # Brute force / mask
-a 6    # Hybrid wordlist + mask

# Useful flags
--force            # Ignore GPU warnings (use in VM)
--show             # Show already cracked hashes from .pot file
--status           # Show progress every 10 seconds
-O                 # Optimized kernels (faster, some limitations)
-w 3               # Workload profile (1=low, 3=high, 4=insane)
-o cracked.txt     # Save cracked passwords to file

# Real examples from our Phase 4 lab:
hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt --force
hashcat -m 1000 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt -r best64.rule
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt --force
```

### Checking results
```bash
# After cracking, show cracked passwords:
hashcat -m 13100 kerberoast_hashes.txt --show
# $krb5tgs$23$*svc_sql*...:SqlService2024!
#                            ↑ cracked password
```

---

## 11. rockyou.txt: The Standard Wordlist

**rockyou.txt** comes from the 2009 RockYou data breach — 32 million
plaintext passwords leaked from a gaming social network.

```
Location on Kali: /usr/share/wordlists/rockyou.txt.gz
                  (gunzip to use: gunzip /usr/share/wordlists/rockyou.txt.gz)

Size: ~133 MB uncompressed
Entries: ~14.3 million passwords

Top 10 entries:
  123456
  12345
  123456789
  password
  iloveyou
  princess
  1234567
  rockyou
  12345678
  abc123
```

Passwords in our lab that ARE in rockyou.txt:
```
Welcome1     ← charlie.bianchi → crackable in seconds
Password123! ← alice.rossi     → crackable with rules
Summer2024!  ← bob.verdi       → crackable with rules
```

Passwords in our lab that are NOT in rockyou.txt (require brute force):
```
SqlService2024! ← svc_sql     → dictionary + rules eventually
L4bAdm1n!      ← labadmin    → longer, mixed, takes more time
```

Other useful wordlists:
```bash
ls /usr/share/wordlists/
# rockyou.txt.gz    ← go-to standard
# fasttrack.txt     ← common enterprise passwords
# dirb/             ← web directory fuzzing
# dirbuster/        ← web directory fuzzing

# Additional wordlists (download separately):
# SecLists (GitHub.com/danielmiessler/SecLists) → massive collection
# CeWL → generate wordlists from target websites
```

---

## 12. Defense

### For system administrators
| Weakness | Fix |
|----------|-----|
| MD5 hashes in /etc/shadow | Migrate to SHA-512 or yescrypt |
| NTLM enabled in AD | Disable NTLMv1, restrict NTLMv2, prefer Kerberos |
| Weak passwords | Password policy: 14+ chars, complexity, expiration |
| Password reuse | Detect with Enzoic/HaveIBeenPwned integration |
| No salting | Use bcrypt or Argon2 for web app password storage |
| NTDS.dit accessible | Tier 0 isolation, restrict replication privileges |

### Password length vs complexity
```
8-char complex:  P@ssw0rd!     → crackable in hours (in rockyou.txt)
12-char random:  xK9mP2qL7nR4  → years with GPU cluster
16-char passphrase: correct-horse-battery-staple → effectively uncrackable
```

**Length beats complexity.** A 16-character lowercase passphrase is
stronger than an 8-character password with symbols.

---

## Quick Reference — Hashcat Modes for This Lab

| Attack | Hash type | Mode | Wordlist |
|--------|-----------|------|---------|
| /etc/shadow dump | md5crypt | 500 | rockyou.txt |
| NTLM dump (Windows) | NTLM | 1000 | rockyou.txt + rules |
| Kerberoasting | Kerberos TGS | 13100 | rockyou.txt |
| AS-REP Roasting | Kerberos AS-REP | 18200 | rockyou.txt |
