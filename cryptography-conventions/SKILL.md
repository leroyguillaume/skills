---
name: cryptography-conventions
description: >-
  Cryptography in the code a project ships, in any language: algorithms and
  key lengths, password hashing, random values, TLS and JWT verification.
  TRIGGER when: code hashes or checks a password; generates a token, session
  id, API key, salt, nonce or key; encrypts or signs data, or verifies a
  signature, a MAC or a tamper-proof digest; configures a TLS client or
  server or its certificate check; issues or validates a JWT; calls a random
  generator (`random`, `Math.random`, `rand`) or a crypto library (`secrets`,
  `hashlib`, `argon2`, `cryptography`, `ring`, `node:crypto`,
  `crypto.subtle`); user asks which algorithm, key size or password hash to
  use.
  SKIP when: a digest guards against nothing adversarial (cache key, ETag,
  dedup, hash map); TLS ends outside the code (ingress, load balancer,
  cert-manager), with no client or server configured in it.
---

# Cryptography conventions

These rules hold in every language, in a private repository as much as a
public one: they are about what the code does, and an attacker never checks
the repository's visibility. They cover the OpenSSF badge's `crypto_*`
criteria (`openssf-conventions`), but the reason to follow them is simpler —
a cryptographic mistake passes every test and only shows in front of an
attacker.

## Never write your own

- **Call a maintained library; never implement a primitive, a mode, a
  padding or a protocol.** Home-made crypto passes its tests and leaks
  through timing, padding errors or nonce handling that no test exercises.
- **Use the library's highest-level API**: a password-hashing API over a raw
  KDF, an AEAD over a cipher plus a MAC, a TLS stack over a socket wrapped by
  hand. A low-level ("hazmat") API leaves every choice to the caller, and
  every choice is a chance to get it wrong.
- **Only published, expert-reviewed algorithms, with a FLOSS
  implementation.** No XOR "encryption", no obfuscation sold as protection,
  no proprietary scheme that only a vendor's SDK can decrypt.

## Algorithms

| Purpose | Use | Never |
| --- | --- | --- |
| Digest for integrity or signatures | SHA-256, SHA-384, SHA-512, SHA-3, BLAKE2, BLAKE3 | MD4, MD5, SHA-1 |
| Symmetric encryption | An AEAD: AES-GCM, ChaCha20-Poly1305; AES-GCM-SIV when a nonce may repeat | ECB; CBC or CTR without a MAC; DES, 3DES, RC4 |
| Message authentication | HMAC-SHA-256 | A bare hash of key and message |
| Signatures | Ed25519, ECDSA P-256, RSA-PSS | Anything keyed below the floor below |
| Key agreement | X25519, ECDH P-256, with ephemeral keys | Static keys only: a stolen long-term key then decrypts every past session |
| Passwords | Argon2id, [below](#passwords) | Any fast hash, any encryption |
| Random values | The OS CSPRNG, [below](#randomness) | Any general-purpose PRNG, Dual_EC_DRBG |

- **A weak algorithm only for interoperability**, when the most recent
  broadly supported version of a protocol requires it and offers nothing
  better. It is off by default wherever the protocol allows, and the
  documentation states the risk and its mitigation.
- **Leave a TLS library's default cipher suites and key-exchange groups
  alone** rather than pinning a list: the library retires algorithms as they
  age, a pinned list freezes them.

## Key lengths

The OpenSSF floor is NIST's 2030 minimum, 112 bits of security: symmetric
keys of 112 bits, RSA and DH of 2048, elliptic curves of 224, hashes of 224.
That floor runs out: [SP 800-57 Part 1](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
disallows 112-bit strength for protecting data from 2031, and
[NIST IR 8547](https://csrc.nist.gov/pubs/ir/8547/final) deprecates every
quantum-vulnerable public-key algorithm after 2030. What ships today is
still running then, so **defaults give 128 bits of security or more**:

| Kind | Default | Rejected below |
| --- | --- | --- |
| Symmetric | AES-256 or AES-128, ChaCha20 | 128 bits |
| RSA | 3072 bits | 2048 bits |
| Elliptic curve | P-256, Ed25519, X25519 | 224 bits |
| Hash | SHA-256 | 224 bits |

A key below the "rejected" column is refused outright. When a peer forces
one, accepting it takes an explicit setting, off by default: a downgrade the
software allows silently is one an attacker can ask for.

## Passwords

For the passwords of users the software authenticates. A credential the
software sends elsewhere — a database password, an upstream token — cannot be
hashed: it is a secret, [below](#keys-and-secrets).

- **Argon2id**, at least one of the
  [OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
  minimums: 19 MiB of memory, 2 iterations, parallelism 1, or 46 MiB, 1
  iteration, parallelism 1. Raise them as far as the server's login latency
  allows. Only when Argon2id is unavailable, in this order:
  - **scrypt**: N=2^17, r=8, p=1.
  - **bcrypt**, legacy only: cost 10 or more. It reads at most 72 bytes, so a
    longer password is rejected, never silently truncated.
  - **PBKDF2-HMAC-SHA256**, only when FIPS-140 is required: 600,000
    iterations.
- **Store the library's encoded string** (`$argon2id$v=19$m=19456,t=2,p=1$…`):
  it carries the algorithm, the parameters and a per-user salt the library
  drew from the CSPRNG. No separate salt column, no salt of your own.
- **Verify with the library's verify function, then rehash when the
  parameters are out of date** (`check_needs_rehash` and its equivalents),
  while the plaintext is at hand after a successful login. That is how a
  raised cost reaches existing accounts.
- **A pepper is optional**, and when used it is a secret kept out of the
  database, in the secret store.

## Randomness

- **Every key, nonce, IV, salt, token, session id, reset code and API key
  comes from the OS CSPRNG**, through the crypto API of the language. A
  general-purpose PRNG (`random`, `Math.random`, `SmallRng`, any seeded
  generator) produces values an attacker can predict from a few samples.
- **Tokens carry 32 bytes (256 bits) by default, never under 128 bits**,
  encoded as URL-safe base64 or hex. A UUID is an identifier, never a
  secret: a v4 holds 122 random bits, a v7 is mostly a timestamp.
- **Never reuse a nonce with the same key.** Let the library draw it when it
  offers to; otherwise draw a fresh one per message. Where nonces may repeat
  (many writers, a counter that can reset), use AES-GCM-SIV, which survives
  a repeat.
- **Compare a secret in constant time** — a token, a MAC, an API key — with
  the library's comparison. `==` returns at the first differing byte, and
  the timing gives the secret away a byte at a time.

## Keys and secrets

- **A key or credential is read from a file or a secret store** whose path
  is a setting (`config-conventions`), never compiled in, committed or baked
  into an image. Replacing it takes a restart, not a rebuild.
- **One key per purpose**: the encryption key never signs, and a JWT signing
  key never encrypts. A key reused across schemes lets an attack on one
  break the other.
- **Never logged**, not even truncated (`logging-conventions`).

## TLS

- **Certificate and hostname verification stay on**, in production code and
  in tests alike. A private CA is added as a trust anchor (the `caCerts`
  value of `helm-conventions`, a test CA in the test setup), never answered
  by turning verification off: `verify=False`,
  `tls_danger_accept_invalid_certs(true)`, `rejectUnauthorized: false`,
  `NODE_TLS_REJECT_UNAUTHORIZED=0`.
- **TLS 1.2 at least, 1.3 preferred**, through the library's defaults.
- **The secure protocol is the default** for everything the software speaks
  over a network: HTTPS, SSHv2, SFTP. A plaintext fallback takes an explicit
  setting.

## JWT

- **Verify against a fixed list of algorithms**, configured with the key,
  never read from the token's `alg` header
  ([RFC 8725 §2.1](https://www.rfc-editor.org/rfc/rfc8725#section-2.1)): a
  token that names its own algorithm can name `none`, or HMAC keyed with the
  public key.
- **Check `exp`**, and `aud` and `iss` whenever the issuer sets them.
- **Sign with EdDSA or ES256 when the verifier is not the issuer**: it then
  holds only a public key. An HS256 secret is 32 random bytes or more.

## Libraries

| Need | Rust | Python | TypeScript / JavaScript |
| --- | --- | --- | --- |
| Random bytes, tokens | `rand::rng()`, `rand::rngs::SysRng`; never `SmallRng` | `secrets.token_bytes()`, `secrets.token_urlsafe()`; never `random` | `crypto.getRandomValues()`, `crypto.randomBytes()` in Node; never `Math.random()` |
| Password hashing | `argon2` crate with its `getrandom` feature: `Argon2::default()` is Argon2id at 19 MiB, t=2, p=1; `hash_password`, `verify_password` | `argon2-cffi`: `PasswordHasher()` with `hash`, `verify`, `check_needs_rehash` | `argon2` package in Node; never in a browser |
| AEAD | `aes-gcm` (`Aes256Gcm`), `chacha20poly1305` | `cryptography`: `AESGCM`, `ChaCha20Poly1305`, `AESGCMSIV` | `crypto.subtle.encrypt` with `AES-GCM` |
| Constant-time comparison | `subtle` (`ConstantTimeEq`) | `secrets.compare_digest()` | `crypto.timingSafeEqual()` in Node |
| TLS | `rustls` | `ssl.create_default_context()` | Node's `https` and `tls` defaults |
| JWT | `jsonwebtoken`: `Validation::new(Algorithm::EdDSA)` | PyJWT: `jwt.decode(token, key, algorithms=["EdDSA"])` | `jose`: `jwtVerify(token, key, { algorithms: ["EdDSA"] })` |

A browser never hashes a password for storage and never holds a long-term
secret key: whatever it holds, its user and any injected script hold too.
That work happens on the server.

## Never

- Never implement a cryptographic primitive, mode or protocol.
- Never use MD5, SHA-1, DES, 3DES, RC4 or ECB in a security mechanism, nor CBC
  or CTR without a MAC.
- Never default to a key below 128 bits of security, nor accept one below the
  OpenSSF floor without an explicit setting.
- Never store a password with a fast hash or with encryption.
- Never draw a key, nonce, salt or token from a general-purpose PRNG.
- Never compare a secret with `==`.
- Never disable TLS certificate or hostname verification.
- Never take a JWT's algorithm from the token.
