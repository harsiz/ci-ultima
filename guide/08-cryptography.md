# 08 Cryptography: The Rabbit Hole 🔐
### this is the part where things get genuinely fascinating

---

cryptography is one of those fields where the deeper you go, the more interesting it gets. it sits at the intersection of pure mathematics, computer science, and real-world security, and understanding it makes you a significantly better engineer.

we're going deep on this one. hashing, encryption, digital signatures, the math (conceptually, not formally), the broken stuff, the current state of things, and what you actually use in production.

fair warning: this chapter is dense. it's also probably the most interesting thing in this series.

---

## the fundamental distinction: hashing vs encryption

these get confused constantly, even by experienced developers.

**hashing** is a one-way function. you put data in, you get a fixed-size "digest" out. you *cannot* get the original data back from the hash. it's irreversible by design.

```
"hello world" ─── SHA-256 ──→ b94d27b9934d3e08a52e52d7da7dabfac484efe04294e576...
```

**encryption** is reversible. you put data in with a key, you get ciphertext out. with the right key, you can decrypt and get the original data back.

```
"hello world" ─── AES-256 (key) ──→ encrypted gibberish ─── AES-256 (key) ──→ "hello world"
```

this distinction has profound implications:
- passwords should be **hashed**, never encrypted. if you encrypt them, anyone with the key can decrypt them. if you hash them, you can only check if a given password produces the same hash.
- data you need to retrieve later (credit card numbers for recurring billing, medical records) must be **encrypted**, not hashed.
- file integrity verification uses **hashing**, compare the hash of the downloaded file to the published hash.

---

## cryptographic hash functions: what makes them "good"

a **cryptographic hash function** must satisfy:

**1. Deterministic:** same input always gives same output.
**2. Fast to compute:** hashing should be computationally cheap.
**3. Pre-image resistance (one-way):** given `H(x)`, it's computationally infeasible to find `x`.
**4. Second pre-image resistance:** given `x`, it's computationally infeasible to find `y ≠ x` such that `H(x) = H(y)`.
**5. Collision resistance:** it's computationally infeasible to find any `x` and `y` where `H(x) = H(y)`.

the word "computationally infeasible" is doing a lot of work. it means possible in theory but would take longer than the heat death of the universe with all the computers that exist. practically impossible.

---

## MD5: the broken one (like, genuinely dead)

MD5 (Message Digest Algorithm 5) was designed by Ron Rivest in 1991 and produced a 128-bit (32 hex character) hash.

```
"password123" → MD5 → 482c811da5d5b4bc6d497ffa98491e38
```

**it is broken. do not use MD5 for anything security-related.** not "old fashioned" or "weaker." *broken.* as in, collisions can be computed in seconds on modern hardware.

the timeline of MD5's death:
- **1996:** theoretical weaknesses found
- **2004:** full collision attack demonstrated (Wang et al.), two different inputs with the same MD5 hash found
- **2008:** X.509 certificate collision attack, researchers created a fraudulent CA certificate using an MD5 collision
- **2012:** Flame malware used MD5 collision to forge a Microsoft code-signing certificate, actual real-world exploitation

the Flame malware collision is worth understanding. Microsoft's certificate authority had been signing certificates with MD5. attackers found two certificate requests that had the same MD5 hash, got one legitimately signed, and used the signature for the other (malicious) certificate. this bypassed code signing completely. this was not theoretical. it happened.

MD5 still appears in non-security contexts (checksums for data integrity verification *within a trusted system*, where collision attacks aren't relevant). `md5sum` for verifying a downloaded file is fine because you're checking against a published hash, an attacker would need to find a second input with the same hash *without controlling the original file*. but for passwords, signatures, certificates, absolutely not.

---

## SHA-1: also broken (but less obvious)

SHA-1 produces a 160-bit hash. it was the standard for SSL certificates, git's object hashing, and code signing for over a decade.

**SHAttered attack (2017):** Google and CWI Amsterdam demonstrated the first practical SHA-1 collision using 9 quintillion (9.2 × 10¹⁸) SHA-1 computations. they produced two different PDF files with identical SHA-1 hashes. cost: approximately $110,000 in cloud computing at the time. decreasing every year.

```
PDF file 1 (normal whitepaper) → SHA-1 → 38762cf7f55934b34d179ae6a4c80cadccbb7f0a
PDF file 2 (malicious content) → SHA-1 → 38762cf7f55934b34d179ae6a4c80cadccbb7f0a
```

same hash. different content. completely different purposes. this is devastating for digital signatures, you could get a certificate authority to sign document 1, and then that signature would be valid for document 2.

browsers stopped accepting SHA-1 certificates in 2017. Git is [migrating from SHA-1 to SHA-256](https://git-scm.com/docs/hash-function-transition/) as the object hashing algorithm (you can already opt in: `git init --object-format=sha256`).

---

## SHA-256 and the SHA-2 family: what you should actually use

SHA-256 (part of SHA-2, designed by the NSA, published 2001) produces a 256-bit hash.

```
"hello world" → SHA-256 → b94d27b9934d3e08a52e52d7da7dabfac484efe04294e576...
                                                (64 hex characters)
```

**current status:** no practical collision attacks exist. the full search space is 2^256 possible hashes. to get a sense of scale: there are approximately 2^266 atoms in the observable universe. finding a collision by brute force is literally astronomical.

the SHA-2 family:
- **SHA-224:** 224-bit output (SHA-256 truncated)
- **SHA-256:** 256-bit output (most common)
- **SHA-384:** 384-bit output (SHA-512 truncated)
- **SHA-512:** 512-bit output (faster than SHA-256 on 64-bit CPUs for large data)
- **SHA-512/256:** SHA-512 truncated to 256 bits (faster than SHA-256 on 64-bit, same security)

**SHA-3:** designed as a backup in case SHA-2 was ever broken (it wasn't). uses a completely different construction (Keccak sponge function vs SHA-2's Merkle-Damgård). SHA-3 is the standard, SHA-3-256 produces 256-bit output. both SHA-2 and SHA-3 are currently secure.

```go
import "crypto/sha256"

func hashData(data []byte) string {
    hash := sha256.Sum256(data)
    return hex.EncodeToString(hash[:])
}

// for streaming large data
func hashFile(path string) (string, error) {
    f, err := os.Open(path)
    if err != nil {
        return "", err
    }
    defer f.Close()
    
    h := sha256.New()
    if _, err := io.Copy(h, f); err != nil {
        return "", err
    }
    
    return hex.EncodeToString(h.Sum(nil)), nil
}
```

---

## HMAC: authenticated hashing (why you can't just hash a message for authentication)

here's a subtle but important concept. you want to verify that a message hasn't been tampered with. you might think: just hash the message and attach the hash.

**this doesn't work.** anyone who can see the message can also compute its hash. an attacker can modify the message and compute a new hash for it.

**HMAC (Hash-based Message Authentication Code)** mixes a secret key into the hash:

```
HMAC-SHA256(key, message) = SHA256(key XOR opad || SHA256(key XOR ipad || message))
```

you can't compute a valid HMAC without knowing the key. only parties who know the key can produce or verify HMACs.

```go
import "crypto/hmac"

func computeHMAC(key, data []byte) []byte {
    mac := hmac.New(sha256.New, key)
    mac.Write(data)
    return mac.Sum(nil)
}

func verifyHMAC(key, data, expectedMAC []byte) bool {
    actualMAC := computeHMAC(key, data)
    // use hmac.Equal for constant-time comparison
    // NEVER use bytes.Equal for MAC comparison (timing attacks!)
    return hmac.Equal(actualMAC, expectedMAC)
}
```

**timing attacks** are why `hmac.Equal` exists. a naive byte-by-byte comparison returns `false` as soon as a mismatch is found. by measuring how long the comparison takes, an attacker can learn how many bytes matched, and use this to forge MACs byte by byte. `hmac.Equal` compares all bytes in constant time regardless of when the first mismatch occurs.

this is the kind of subtle vulnerability that cryptography is full of. correct-looking code that's subtly broken in a way that takes a security PhD to spot.

---

## symmetric encryption: AES

**AES (Advanced Encryption Standard)** is the symmetric encryption standard. same key encrypts and decrypts. NSA approved it for top-secret data. practically every encrypted connection you make uses AES at some layer.

AES operates on 128-bit blocks and supports 128, 192, or 256-bit keys. **AES-256** is the standard for new applications.

**modes of operation:** AES-ECB, AES-CBC, AES-GCM. the mode determines how AES encrypts data larger than one block.

**never use ECB mode.** ECB (Electronic Code Book) encrypts each 128-bit block independently. identical plaintext blocks produce identical ciphertext blocks. this leaks structure:

```
[The most famous ECB vulnerability is the Linux penguin Tux encrypted with AES-ECB.
The output image still clearly shows the penguin's outline because
identical pixel blocks encrypt to identical ciphertext blocks.]
```

**use AES-GCM (Galois/Counter Mode):**
- encrypts the data (confidentiality)
- produces an authentication tag (integrity + authenticity)
- requires a unique nonce (number used once) per encryption
- GCM mode is authenticated encryption, you detect tampering

```go
import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "io"
)

func encrypt(key, plaintext []byte) ([]byte, error) {
    block, err := aes.NewCipher(key)  // key must be 16, 24, or 32 bytes
    if err != nil {
        return nil, err
    }
    
    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }
    
    // nonce must be unique per encryption with same key
    nonce := make([]byte, gcm.NonceSize())  // 12 bytes for GCM
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        return nil, err
    }
    
    // seal appends encrypted data to nonce
    // output: nonce || ciphertext || auth_tag
    ciphertext := gcm.Seal(nonce, nonce, plaintext, nil)
    return ciphertext, nil
}

func decrypt(key, ciphertext []byte) ([]byte, error) {
    block, _ := aes.NewCipher(key)
    gcm, _ := cipher.NewGCM(block)
    
    nonceSize := gcm.NonceSize()
    if len(ciphertext) < nonceSize {
        return nil, errors.New("ciphertext too short")
    }
    
    nonce, ciphertext := ciphertext[:nonceSize], ciphertext[nonceSize:]
    
    // open decrypts and verifies the authentication tag
    plaintext, err := gcm.Open(nil, nonce, ciphertext, nil)
    if err != nil {
        return nil, fmt.Errorf("decryption failed: %w", err)  // auth tag invalid = tampered
    }
    
    return plaintext, nil
}
```

the nonce must be unique for every encryption with the same key. reusing a nonce with the same key under GCM completely breaks confidentiality. use `crypto/rand` to generate random nonces, 12 bytes of random data gives you 2^96 possible nonces, making accidental collision astronomically unlikely.

---

## asymmetric (public-key) cryptography: RSA and elliptic curves

symmetric encryption has a fundamental problem: how do you share the key? if you can securely share a key, you could've just sent the message that way. **the key distribution problem**.

asymmetric cryptography solves this with key pairs:
- **public key:** can be shared with anyone
- **private key:** kept secret

what you encrypt with the public key can only be decrypted with the private key. what you sign with the private key can be verified by anyone with the public key.

### RSA

RSA (Rivest-Shamir-Adleman, 1977) is based on the hardness of factoring large numbers. multiplying two large primes is easy. factoring the product back into the primes is computationally infeasible for large enough numbers.

```
p = 61, q = 53           (in real RSA, these are 2048+ bit primes)
n = p × q = 3233          (public modulus)
e = 17                    (public exponent, typically 65537)
d = 2753                  (private exponent, computed from p, q, e)

encrypt:  c = m^e mod n
decrypt:  m = c^d mod n
```

RSA-2048 (2048-bit modulus) is still considered secure. RSA-1024 is deprecated. RSA-4096 exists but is overkill for most applications (4x slower than RSA-2048 with no practical security benefit given current factoring capabilities).

### elliptic curve cryptography (ECC)

ECC provides equivalent security to RSA with much smaller keys:

| Security level | RSA key size | ECC key size |
|----------------|-------------|--------------|
| 128-bit | 3072 bits | 256 bits |
| 192-bit | 7680 bits | 384 bits |
| 256-bit | 15360 bits | 521 bits |

ECC-256 provides the same security as RSA-3072 with a 12x smaller key. smaller keys = faster operations = less bandwidth.

**ECDSA (Elliptic Curve Digital Signature Algorithm)** and **Ed25519** are the current standards for digital signatures. Ed25519 (using the Edwards25519 curve) is the modern recommendation, faster than ECDSA, no parameter tuning required, no timing attack vulnerabilities.

```go
import (
    "crypto/ed25519"
    "crypto/rand"
)

// generate key pair
pub, priv, _ := ed25519.GenerateKey(rand.Reader)

// sign
message := []byte("important document")
signature := ed25519.Sign(priv, message)

// verify (anyone with the public key can do this)
valid := ed25519.Verify(pub, message, signature)
// valid == true
```

**X25519** (key exchange using Curve25519) is used for TLS key exchange. when you connect to `https://example.com`, X25519 is typically used to establish the symmetric encryption key for the session.

### Diffie-Hellman key exchange: the magic trick

here's one of the most elegant concepts in all of cryptography: two parties can agree on a shared secret over a completely public channel, without ever sending the secret.

```
Alice and Bob agree publicly on: prime p = 23, base g = 5

Alice picks secret a = 6:
  Alice computes: A = g^a mod p = 5^6 mod 23 = 8
  Alice sends 8 to Bob (public)

Bob picks secret b = 15:
  Bob computes: B = g^b mod p = 5^15 mod 23 = 19
  Bob sends 19 to Alice (public)

Alice computes: s = B^a mod p = 19^6 mod 23 = 2
Bob computes:   s = A^b mod p = 8^15 mod 23 = 2

Both got 2. Eve, watching the channel, only saw p=23, g=5, A=8, B=19.
She cannot compute 2 without solving the discrete logarithm problem.
```

this is MIND-BLOWING the first time you see it. two people who've never met, communicating over a line anyone can eavesdrop on, agree on a shared secret that nobody else knows. the mathematics of discrete logarithms is what makes it hard to reverse.

Modern implementations use X25519 (Diffie-Hellman over Curve25519) instead of this classical version, but the principle is identical.

---

## TLS: how HTTPS works

when you connect to `https://api.example.com`, a lot happens:

1. **TCP handshake** (connection established)
2. **TLS ClientHello:** client says "I support TLS 1.3, here are my cipher suites"
3. **TLS ServerHello:** server picks cipher suite, sends its certificate (public key + identity)
4. **Certificate verification:** client verifies the certificate is signed by a trusted Certificate Authority (CA) and the hostname matches
5. **Key exchange:** X25519 Diffie-Hellman establishes a shared session key (both sides compute it independently, nobody sends it)
6. **Symmetric encryption begins:** all further communication is AES-GCM encrypted with the session key

TLS 1.3 (2018) dropped many legacy cipher suites, made handshake faster (1-RTT vs 2-RTT in TLS 1.2), and enables **0-RTT resumption** for reconnecting clients (faster but with replay attack caveats).

**certificate authorities and the trust chain:**

you trust `api.example.com` because its certificate is signed by an intermediate CA, which is signed by a root CA, which is pre-loaded in your OS/browser. the root CAs are the ultimate source of trust. there are about 150 trusted root CAs worldwide.

this is why certificate pinning (your app only trusts specific certificates, not the whole CA system) was popular for mobile apps, if a CA is compromised or issues a fraudulent certificate, certificate pinning protects you. it's harder to maintain but provides additional security.

---

## the birthday paradox and collision probability

something counterintuitive: the probability of two people in a group sharing a birthday (365 options) reaches 50% with only **23 people**. with 70 people it's 99.9%.

this applies to hash collisions. for a hash function with `n` possible outputs, you need approximately `√n` random inputs to have a 50% chance of finding a collision. this is the birthday bound.

- MD5 has 2^128 outputs → collision expected after ≈ 2^64 attempts → achievable with modern hardware
- SHA-256 has 2^256 outputs → collision expected after ≈ 2^128 attempts → currently infeasible (2^128 is genuinely astronomical)

this is why MD5's 128-bit output is fundamentally weaker than it sounds, and why SHA-256 at 256 bits has strong collision resistance.

---

## go 1.26's new crypto: HPKE

Go 1.26 added the `crypto/hpke` package (Hybrid Public Key Encryption), which combines asymmetric and symmetric encryption efficiently:

1. generate an ephemeral key pair
2. derive a shared secret using X25519 (DH)
3. use the shared secret as a symmetric key for HKDF + AES-GCM

HPKE is used in ECH (Encrypted Client Hello) in TLS 1.3, which hides even the hostname you're connecting to from network observers. it's also used in MLS (Messaging Layer Security) for group messaging encryption.

```go
import "crypto/hpke"

// HPKE modes for different use cases
kem  := hpke.DHKEM_X25519_HKDF_SHA256  // key encapsulation
kdf  := hpke.HKDF_SHA256               // key derivation
aead := hpke.AES_128_GCM               // encryption

// sender
suite := hpke.NewSuite(kem, kdf, aead)
sender, encapsulated, _ := suite.NewSender(recipientPublicKey, nil)
ciphertext, _ := sender.Seal(plaintext, nil)

// recipient
receiver, _ := suite.NewReceiver(recipientPrivateKey, encapsulated, nil)
plaintext, _ := receiver.Open(ciphertext, nil)
```

---

## what NOT to use (the actual list)

| Algorithm | Status | Replace with |
|-----------|--------|-------------|
| MD5 | BROKEN (collisions in seconds) | SHA-256 |
| SHA-1 | BROKEN (collision demonstrated 2017) | SHA-256 |
| RC4 | BROKEN | AES-GCM |
| DES / 3DES | BROKEN / deprecated | AES-256 |
| RSA-1024 | deprecated | RSA-2048+ or ECDSA/Ed25519 |
| AES-ECB | fundamentally insecure | AES-GCM |
| Dual_EC_DRBG | backdoored (NSA) | ChaCha20-Poly1305 or use OS RNG |
| `math/rand` for security | not cryptographic | `crypto/rand` |

that last one is a classic Go mistake. `math/rand` is for simulations and games. it's deterministic given a seed. `crypto/rand` reads from the OS's cryptographically secure random number generator (CSPRNG). use `crypto/rand` for:
- generating passwords
- generating session tokens  
- generating nonces
- generating keys
- generating anything security-sensitive

```go
// WRONG: math/rand is predictable
import "math/rand"
token := rand.Int63()  // attacker can predict this

// RIGHT: crypto/rand is cryptographically secure
import "crypto/rand"
import "encoding/base64"

func generateToken(n int) (string, error) {
    bytes := make([]byte, n)
    _, err := rand.Read(bytes)
    if err != nil {
        return "", err
    }
    return base64.URLEncoding.EncodeToString(bytes), nil
}
```

---

*next: [09 Auth and Security: Passwords, OAuth2, and JWT](09-auth-and-security.md)*

*sources: [SHA-256 vs MD5 patsnap](https://eureka.patsnap.com/article/sha-256-vs-md5-which-hashing-algorithm-is-more-secure-in-2025) | [SHAttered attack](https://shattered.io/) | [Go crypto/hpke](https://pkg.go.dev/crypto/hpke) | [Aikido stop MD5 SHA-1](https://www.aikido.dev/code-quality/rules/stop-using-md5-and-sha-1-modern-hashing-for-security)*
