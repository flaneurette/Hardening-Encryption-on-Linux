# Hardening with GnuPG on Linux

Below are three practical scenarios for hardening secure backups with GnuPG.

## How this protects against quantum decryption

Quantum computers can threaten **asymmetric cryptography** (RSA, ECC) but do not efficiently break symmetric encryption like AES.

- AES-256 is reduced to roughly AES-128 security against quantum attacks (via Grover's algorithm)  
- Brute-forcing 2^128 possibilities is still completely infeasible 
- Your GPG-encrypted backup uses AES to encrypt data, so even if RSA/ECC is broken in the future, the bulk data remains secure

**Key points:**
- Symmetric encryption is resistant to quantum attacks  
- Strong passphrases and offline keys are essential  
- The main risk is key compromise, not the encryption algorithm  

> Even if an attacker stores your encrypted backup today and waits decades for a quantum computer, the backup remains secure if your key/passphrase is strong and protected.
> In essence, symmetric encryption isn't perfectly 100% immune but is far more resilient, requiring simple key-length increases rather than a complete algorithmic overhaul like asymmetric cryptography.
---

## Scenario 1 - Pure symmetric encryption with offline Key

**Goal:** Protect against “store now, decrypt later” by keeping the key offline.

### Steps

1. Generate a random key (256-bit):
```bash
gpg --gen-random 2 32 > backup.key
```

**Notes:**
- Never transmit this key over the network
- Keep a backup of the key in a safe location
- Use a key per backup for best security
  
2. Encrypt backup using that key:
```bash
gpg --symmetric \
    --cipher-algo AES256 \
    --batch \
    --passphrase-file backup.key \
    backup.tar
```

3. Decrypt later:
```bash
gpg --decrypt \
    --batch \
    --passphrase-file backup.key \
    backup.tar.gpg > backup.tar
```

**Properties:**
- Immune to quantum attacks
- Intercepted backups are useless
- No private key compromise risk
- USB key must be protected

---

## Scenario 2 - Split Key (Two secrets required)

**Goal:** Require both offline and remote secrets to decrypt.

### Steps

1. Generate symmetric key:
```bash
gpg --gen-random 2 32 > full.key
```

2. Split into two parts:
```bash
split -b 16 full.key keypart_
```

- `keypart_aa` → offline USB  
- `keypart_ab` → will be encrypted

3. Encrypt second half with GPG:
```bash
gpg --encrypt -r YOURKEYID keypart_ab
```

4. Encrypt backup with full key:
```bash
gpg --symmetric \
    --cipher-algo AES256 \
    --batch \
    --passphrase-file full.key \
    backup.tar
```

5. Restore:
```bash
gpg --decrypt keypart_ab.gpg
cat keypart_aa keypart_ab > full.key
gpg --decrypt --batch --passphrase-file full.key backup.tar.gpg > backup.tar
```

**Properties:**
- Needs **both** offline + encrypted GPG part
- Resistant to interception
- Slightly more operational complexity

---

## Scenario 3 - Key-encrypted key (professional pattern)

**Goal:** Industry-standard approach separating data and encryption key.

### Steps

1. Generate data encryption key (DEK):
```bash
gpg --gen-random 2 32 > dek.key
```

2. Encrypt backup with DEK:
```bash
gpg --symmetric \
    --cipher-algo AES256 \
    --batch \
    --passphrase-file dek.key \
    backup.tar
```

3. Encrypt DEK with your GPG key:
```bash
gpg --encrypt -r YOURKEYID dek.key
```

4. Delete plaintext DEK:
```bash
shred -u dek.key
```

5. Restore:
```bash
gpg --decrypt dek.key.gpg > dek.key
gpg --decrypt --batch --passphrase-file dek.key backup.tar.gpg > backup.tar
```

**Properties:**
- Clean separation of key and data
- Easy to rotate GPG keys later
- Used in professional backup workflows
  
---

## Recommendations

| Threat model | Best choice |
|--------------|------------|
| Maximum paranoia, long-term secrecy | Scenario 1 |
| Physical + cryptographic separation | Scenario 2 |
| Clean, maintainable, professional | Scenario 3 |

> Scenario 1 + good operational hygiene is already extremely strong for personal or small organizational backups.

---

## Notes

- Protect offline keys carefully (USB, paper, or hardware device)  
- Use strong passphrases and modern algorithms (AES-256, Ed25519)  
- Regularly rotate keys if possible  
- For long-term sensitive data, monitor post-quantum crypto developments
