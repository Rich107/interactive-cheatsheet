# openssl certificates & keys

The commands you Google three times a year — generating keys, building CSRs, inspecting certs,
verifying chains, probing TLS endpoints, converting between PEM/DER/PKCS#12, and hashing files.
All single-binary `openssl` invocations.

## 1. Generate a private key

Generate a 4096-bit RSA private key (modern form, encrypted output via `-pass` if you add it):

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out key.pem
```

Older but still common form:

```bash
openssl genrsa -out key.pem 4096
```

Add `-aes256` to `genrsa` (or `-pass pass:secret` to `genpkey`) to encrypt the resulting key at rest.

## 2. Generate an EC key

Elliptic-curve keys are smaller and faster than RSA at equivalent security levels. P-256 is the
common default:

```bash
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out ec.key
```

Swap `P-256` for `P-384` or `P-521` for higher security; `secp256k1` for the Bitcoin/Ethereum curve.

## 3. Generate a CSR

Create a Certificate Signing Request from an existing key, with the subject embedded inline so the
command doesn't prompt:

```bash
openssl req -new -key key.pem -out req.csr -subj "/CN=example.com"
```

Add SANs with `-addext "subjectAltName=DNS:example.com,DNS:www.example.com"`.

## 4. Self-signed cert

One-shot: generate a fresh RSA key and a self-signed cert in a single command — useful for local
TLS, internal services, or test fixtures:

```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes -subj "/CN=example.com"
```

`-nodes` ("no DES") writes the key unencrypted. Drop it for an interactive passphrase prompt.

## 5. Inspect a cert

Full human-readable dump:

```bash
openssl x509 -in cert.pem -noout -text
```

Just the bits you usually want:

```bash
openssl x509 -in cert.pem -noout -subject -issuer -dates -fingerprint
```

`-checkend 86400` exits non-zero if the cert will expire in the next 24 hours — perfect for
monitoring scripts.

## 6. Inspect a CSR

Decode a CSR and verify its self-signature in one go:

```bash
openssl req -in req.csr -noout -text -verify
```

## 7. Inspect a private key

Confirm the algorithm, curve/key size, and modulus of a key on disk:

```bash
openssl pkey -in key.pem -noout -text
```

Add `-pubout` (drop `-noout`) to extract the matching public key.

## 8. Verify a chain

Check that `cert.pem` is signed by something in `chain.pem` (which should contain the issuing
intermediate and root concatenated together):

```bash
openssl verify -CAfile chain.pem cert.pem
```

Exit 0 + `OK` on success. Use `-untrusted intermediates.pem -CAfile root.pem` when the
intermediates aren't trust anchors themselves.

## 9. Test a TLS endpoint

Print the certificate chain a server presents over TLS, with SNI set explicitly (most modern
servers require it). The `< /dev/null` closes stdin so the command returns instead of hanging:

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts < /dev/null
```

Pipe into `openssl x509 -noout -dates` to extract just the validity window of the leaf cert.

## 10. Convert formats

PEM (base64) to DER (binary):

```bash
openssl x509 -in cert.pem -outform der -out cert.der
```

Bundle a key + cert (+ optional chain) into a PKCS#12 / `.p12` / `.pfx` file:

```bash
openssl pkcs12 -export -inkey key.pem -in cert.pem -out bundle.p12 -name "example.com"
```

Pull the key back out of a PKCS#12 (use `-clcerts -nokeys` to extract only the certificate):

```bash
openssl pkcs12 -in bundle.p12 -nocerts -nodes -out key.pem
```

## 11. Hash & encode

SHA-256 a file:

```bash
openssl dgst -sha256 file
```

Base64-encode stdin (decode with `-d`):

```bash
openssl base64
```

Generate 32 cryptographically-random bytes as base64 — a solid one-liner for secrets, tokens, or
session keys:

```bash
openssl rand -base64 32
```
