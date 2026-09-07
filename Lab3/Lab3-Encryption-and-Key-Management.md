# Lab 3: Data Protection — Encryption and Key Management

**Course:** IKB42603 Cloud Computing Security Essentials
**Lab:** Lab 3
**Topic:** At-rest and in-transit encryption, envelope encryption, cryptographic erasure and data integrity
**Environment:** OpenSSL, Docker, LocalStack KMS and AWS CLI v2

## Lab Summary

This lab demonstrates multiple data protection mechanisms used to protect confidentiality and integrity in cloud environments.

The lab begins with encryption fundamentals by demonstrating symmetric AES encryption for data at rest and asymmetric RSA encryption and digital signatures. TLS is then used to demonstrate encryption of data while it is transmitted over a network.

The second part of the lab focuses on cloud key management using LocalStack KMS. A KMS master key is used to generate a data encryption key, demonstrating envelope encryption. Separate keys are then used for different tenants, followed by cryptographic erasure through disabling a key.

Finally, SHA-256 hashing and a simple hash chain are used to demonstrate integrity verification and tamper-evident logging.

The lab demonstrates that encryption algorithms alone are not sufficient to protect data. Secure key management, key separation, controlled key access and secure key deletion are essential parts of cloud data protection.

---

## Task 1: Symmetric Encryption — Data at Rest

A sensitive patient record was created:

```bash
echo 'Patient: Ahmad, Diagnosis: confidential' > record.txt
```

The file was encrypted using AES-256-CBC:

```bash
openssl enc -aes-256-cbc -pbkdf2 -salt \
  -in record.txt \
  -out record.enc
```

The encrypted file was inspected:

```bash
cat record.enc
```

The ciphertext was not readable as the original plaintext patient record.

The file was then decrypted:

```bash
openssl enc -d -aes-256-cbc -pbkdf2 \
  -in record.enc \
  -out record.dec.txt
```

The original and decrypted files were compared:

```bash
diff record.txt record.dec.txt && \
echo 'MATCH: decryption successful'
```

### Result

The `MATCH: decryption successful` result confirms that the AES-encrypted data was successfully decrypted using the same secret.

Symmetric encryption uses one shared secret for both encryption and decryption. It is fast and suitable for protecting large amounts of data, but the shared key must be securely stored and distributed.

### Evidence

---

## Task 2: Asymmetric Encryption and Digital Signatures

An RSA private key was generated:

```bash
openssl genrsa -out private.pem 2048
```

The corresponding public key was extracted:

```bash
openssl rsa -in private.pem -pubout -out public.pem
```

The sensitive record was encrypted using the public key:

```bash
openssl pkeyutl -encrypt \
  -pubin \
  -inkey public.pem \
  -in record.txt \
  -out record.rsa
```

The encrypted data was decrypted using the private key:

```bash
openssl pkeyutl -decrypt \
  -inkey private.pem \
  -in record.rsa \
  -out record.rsa.txt
```

A digital signature was then created using the private key:

```bash
openssl dgst -sha256 \
  -sign private.pem \
  -out record.sig \
  record.txt
```

The signature was verified using the public key:

```bash
openssl dgst -sha256 \
  -verify public.pem \
  -signature record.sig \
  record.txt
```

### Result

The expected verification output is:

```text
Verified OK
```

This demonstrates the two different uses of asymmetric cryptography.

For confidentiality:

* The public key encrypts the data.
* The private key decrypts the data.

For digital signatures:

* The private key signs the data.
* The public key verifies the signature.

Digital signatures provide evidence of integrity and origin because modification of the signed data causes signature verification to fail.

### Evidence

---

## Task 3: Encryption in Transit Using TLS

A self-signed TLS certificate was generated:

```bash
openssl req -x509 -newkey rsa:2048 \
  -keyout key.pem \
  -out cert.pem \
  -days 7 \
  -nodes \
  -subj '/CN=localhost'
```

An Nginx container was started to serve the sensitive record over HTTPS:

```bash
docker run --rm -d \
  --name tls \
  -p 8443:443 \
  -v $(pwd)/cert.pem:/etc/nginx/cert.pem \
  -v $(pwd)/key.pem:/etc/nginx/key.pem \
  -v $(pwd)/record.txt:/usr/share/nginx/html/record.txt \
  nginx
```

The HTTPS service was accessed using:

```bash
curl -k https://localhost:8443/record.txt
```

### Result

The connection uses HTTPS, which protects data while it is transmitted across the network.

The `-k` option allows `curl` to accept the self-signed certificate used in the lab.

Without TLS, data transmitted using HTTP could potentially be read by an attacker monitoring the network. TLS encrypts the communication channel so intercepted traffic is not readable as plaintext.

### Evidence

---

# Task 4: KMS Master Key Management

LocalStack was used to simulate AWS Key Management Service.

The LocalStack endpoint variable was configured:

```bash
EP='--endpoint-url=http://localhost:4566'
```

A KMS master key for Tenant A was created:

```bash
aws $EP kms create-key \
  --description 'CCSE tenant-A master key'
```

The returned KeyId was stored:

```bash
KEY_A=<PASTE_KEYID>
```

A small secret was encrypted using KMS:

```bash
aws $EP kms encrypt \
  --key-id $KEY_A \
  --plaintext "$(echo -n 'hello' | base64)" \
  --query CiphertextBlob \
  --output text
```

### Result

The KMS master key provides centralized management for encryption keys.

The key is identified using its KeyId and is used to protect cryptographic operations without requiring the application to directly manage the underlying master key material.

### Evidence

---

# Task 5: Envelope Encryption

Envelope encryption was implemented using a KMS master key and a separate data encryption key.

A data key was generated using KMS:

```bash
aws $EP kms generate-data-key \
  --key-id $KEY_A \
  --key-spec AES_256 \
  --query '[Plaintext,CiphertextBlob]' \
  --output text
```

The command returned two values:

1. A plaintext data key.
2. A KMS-encrypted version of the data key.

The values were stored as:

```text
datakey.b64
datakey.enc
```

The plaintext key was converted into binary form:

```bash
base64 -d datakey.b64 > datakey.bin
```

The data key was used to encrypt the sensitive record locally:

```bash
openssl enc -aes-256-cbc -pbkdf2 \
  -in record.txt \
  -out record.env.enc \
  -pass file:./datakey.bin
```

After encryption, the plaintext copies of the data key were removed:

```bash
rm datakey.bin datakey.b64
```

The remaining protected key was confirmed:

```bash
echo 'Only the KMS-wrapped data key (datakey.enc) remains.'
```

### Result

Envelope encryption separates bulk data encryption from master key protection.

The process can be summarised as:

```text
record.txt
    ↓
Encrypted using
plaintext Data Encryption Key
    ↓
record.env.enc

Data Encryption Key
    ↓
Wrapped by
KMS Master Key
    ↓
datakey.enc
```

The data encryption key encrypts the large data locally, while the KMS master key protects the smaller data key.

Only the KMS-wrapped data key needs to be stored after encryption.

### Evidence

---

# Task 6: Per-Tenant Keys and Cryptographic Erasure

A separate KMS key was created for Tenant B:

```bash
aws $EP kms create-key \
  --description 'CCSE tenant-B master key'
```

The returned KeyId was stored:

```bash
KEY_B=<PASTE_KEYID>
```

Tenant A's key was scheduled for deletion:

```bash
aws $EP kms schedule-key-deletion \
  --key-id $KEY_A \
  --pending-window-in-days 7
```

The key was then disabled immediately:

```bash
aws $EP kms disable-key \
  --key-id $KEY_A
```

An attempt was made to decrypt Tenant A's wrapped data key:

```bash
aws $EP kms decrypt \
  --ciphertext-blob fileb://datakey.enc \
  2>&1 | head -3
```

### Result

The decrypt operation should fail because the Tenant A master key has been disabled.

This demonstrates cryptographic erasure.

The encrypted data and wrapped data key may still physically exist, but without access to the master key, the data encryption key cannot be recovered.

Therefore:

```text
KMS Master Key unavailable
        ↓
Cannot unwrap Data Encryption Key
        ↓
Cannot decrypt record.env.enc
        ↓
Encrypted data is unrecoverable
```

Per-tenant keys also improve isolation because one tenant's key material is separate from another tenant's key.

### Evidence

---

# Task 7: Integrity and Tamper-Evidence

## SHA-256 Integrity Verification

The SHA-256 hash of the original file was generated:

```bash
sha256sum record.txt
```

A copy of the file was created and modified:

```bash
cp record.txt tampered.txt
echo 'x' >> tampered.txt
```

The hashes of both files were compared:

```bash
sha256sum record.txt tampered.txt
```

### Result

The original and modified files produce different SHA-256 hashes.

This demonstrates integrity verification because even a small modification changes the resulting hash.

### Hash Chain

A simple tamper-evident hash chain was created:

```bash
PREV=0

for line in 'login ok' 'file read' 'export data'; do
  PREV=$(echo -n "$PREV$line" | sha256sum | cut -d' ' -f1)
  echo "$line | $PREV"
done
```

The hash chain works as follows:

```text
PREV = 0

login ok
    ↓
Hash 1

Hash 1 + file read
    ↓
Hash 2

Hash 2 + export data
    ↓
Hash 3
```

### Result

Each hash depends on the previous hash and the current log entry.

If an earlier entry is modified, its hash changes. This causes the hashes of subsequent entries to no longer match the original chain.

The hash chain does not prevent modification, but it makes unauthorised modification detectable.

### Evidence

---

# Verification Commands

The following commands were used to verify the cryptographic controls.

## Verify KMS Keys

```bash
aws --endpoint-url=http://localhost:4566 kms list-keys
```

This verifies that the KMS keys have been created.

## Verify Digital Signature

```bash
openssl dgst -sha256 \
  -verify public.pem \
  -signature record.sig \
  record.txt
```

Expected output:

```text
Verified OK
```

---

# Short-Answer Questions

## Q1. Compare symmetric and asymmetric encryption: speed, key distribution, and typical use.

Symmetric encryption uses one shared key for both encryption and decryption, while asymmetric encryption uses a public key and a private key.

Symmetric encryption is generally faster and is suitable for encrypting large amounts of data, such as data at rest. However, it has a key distribution problem because the same secret key must be securely shared with every authorised party.

Asymmetric encryption is generally slower but simplifies some key distribution problems because the public key can be shared openly while the private key remains secret. It is commonly used for secure key exchange, digital signatures, PKI and TLS.

In this lab, AES was used for symmetric encryption, while RSA was used for public/private key encryption and digital signatures.

---

## Q2. Why is key management described as the weakest link, not the algorithm?

Strong encryption algorithms cannot protect data if the encryption keys are poorly managed.

If a key is exposed, stolen, incorrectly stored, accessed by an unauthorised user or not properly deleted, an attacker may be able to decrypt the protected data.

Therefore, encryption security depends not only on the algorithm but also on:

* Where keys are stored.
* Who can access the keys.
* How keys are protected.
* How keys are rotated.
* How keys are separated between tenants.
* How keys are securely disabled or deleted.

This is why key management is often considered the most important part of an encryption system.

---

## Q3. Explain envelope encryption and why only the master key needs hardware-grade protection.

Envelope encryption uses two levels of encryption keys.

A Data Encryption Key (DEK) encrypts the actual data locally. The master key stored and managed by KMS encrypts, or wraps, the data key.

The encrypted data and wrapped data key can then be stored together.

When the data is needed:

1. The wrapped data key is sent to KMS.
2. KMS decrypts or unwraps the data key.
3. The plaintext data key decrypts the actual data.
4. The plaintext data key is discarded after use.

This is efficient because the master key does not encrypt every large file directly. Only the smaller master key requires the strongest hardware-grade protection, while temporary data keys perform the bulk encryption.

---

## Q4. How does cryptographic erasure achieve provable deletion where overwriting cannot in the cloud?

Cryptographic erasure makes encrypted data unrecoverable by destroying, disabling or otherwise permanently removing access to the encryption key required to decrypt it.

In this lab, Tenant A's master key is scheduled for deletion and disabled. The encrypted data and wrapped data key may still exist, but without the master key, the data key cannot be recovered.

Therefore, the encrypted record cannot be decrypted.

This is especially useful in cloud environments because customers generally do not control the underlying physical storage. They cannot reliably overwrite every physical block, replica, backup or storage copy.

Destroying the encryption key provides a practical method of making encrypted data unrecoverable even when physical storage copies may remain.

---

## Q5. How does a hash chain make a log tamper-evident?

A hash chain links each log entry to the hash of the previous entry.

The lab starts with:

```text
PREV = 0
```

Each new hash is calculated using:

```text
Previous Hash + Current Log Entry
```

For example:

```text
Entry 1
    ↓
Hash 1

Entry 2 + Hash 1
    ↓
Hash 2

Entry 3 + Hash 2
    ↓
Hash 3
```

If an attacker modifies an earlier log entry, its hash changes. This also causes the following hashes to no longer match the original chain.

A hash chain does not prevent someone from changing a log, but it makes modification detectable. Therefore, it provides tamper-evidence and supports tamper-evident logging.

---

# Security Best-Practices Checklist

* [x] Sensitive data is encrypted at rest using AES.
* [x] AES decryption is verified using the `MATCH` confirmation.
* [x] RSA public and private keys are used for asymmetric cryptography.
* [x] Digital signatures are verified using the public key.
* [x] Data is protected in transit using TLS.
* [x] KMS is used for centralized key management.
* [x] Envelope encryption separates data encryption from master key protection.
* [x] Plaintext data keys are removed after encryption.
* [x] Separate keys are used for different tenants.
* [x] Cryptographic erasure is demonstrated by disabling the required decryption key.
* [x] SHA-256 hashing is used to verify data integrity.
* [x] A hash chain is used to demonstrate tamper-evident logging.

---

# Cleanup and Teardown

After saving all evidence, the lab environment can be cleaned up.

Stop the TLS container:

```bash
docker stop tls 2>/dev/null
```

Remove generated cryptographic files:

```bash
rm -f record.* \
  private.pem \
  public.pem \
  key.pem \
  cert.pem \
  datakey.* \
  tampered.txt
```

Stop and remove LocalStack:

```bash
docker stop localstack && docker rm localstack
```

---

# Conclusion

This lab demonstrated multiple techniques for protecting data confidentiality and integrity in cloud environments.

AES symmetric encryption was used to protect data at rest, while RSA asymmetric cryptography demonstrated public/private key encryption and digital signatures. TLS was used to protect data in transit.

The lab then demonstrated that key management is a critical part of encryption security. LocalStack KMS was used to create master keys and generate data encryption keys for envelope encryption. The plaintext data key encrypted the actual record, while the KMS master key protected the data key.

Separate keys were created for different tenants to provide stronger cryptographic separation. Disabling Tenant A's master key demonstrated cryptographic erasure because the encrypted data could no longer be decrypted.

Finally, SHA-256 hashing and a hash chain demonstrated data integrity and tamper-evident logging.

Overall, the lab shows that effective cloud data protection requires more than selecting a strong encryption algorithm. Secure systems must also protect data in transit and at rest, manage keys securely, separate tenant keys, support secure deletion and verify data integrity.
