---
title: Encryption
slug: encryption
order: 2
tags: [security, encryption, cryptography, aes-256]
---

# Encryption

DotEnv uses industry-standard encryption to protect your secrets at rest and in transit. This guide covers our encryption implementation, algorithms, and best practices.

## Encryption Overview

### Encryption Layers

```
┌─────────────────────────────────────────┐
│          User's Browser/CLI             │
│  ┌───────────────────────────────────┐  │
│  │   Optional Client Encryption      │  │ Layer 1
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
                    ↓ TLS 1.3
┌─────────────────────────────────────────┐
│            DotEnv Platform              │
│  ┌───────────────────────────────────┐  │
│  │    Server-Side Encryption        │  │ Layer 2
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│           Database Storage              │
│  ┌───────────────────────────────────┐  │
│  │    Storage-Level Encryption      │  │ Layer 3
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

## Encryption Algorithms

### AES-256-GCM

Our primary encryption algorithm:

```javascript
// Encryption parameters
const ALGORITHM = "aes-256-gcm";
const KEY_LENGTH = 32; // 256 bits
const IV_LENGTH = 12; // 96 bits for GCM
const TAG_LENGTH = 16; // 128 bits
const SALT_LENGTH = 32; // 256 bits

// Encryption process
function encrypt(plaintext, key) {
    const iv = crypto.randomBytes(IV_LENGTH);
    const cipher = crypto.createCipheriv(ALGORITHM, key, iv);

    const encrypted = Buffer.concat([
        cipher.update(plaintext, "utf8"),
        cipher.final(),
    ]);

    const tag = cipher.getAuthTag();

    // Combine IV + encrypted + tag
    return Buffer.concat([iv, encrypted, tag]);
}
```

### Key Derivation

```javascript
// PBKDF2 for key derivation
const ITERATIONS = 100000;
const DIGEST = "sha256";

function deriveKey(password, salt) {
    return crypto.pbkdf2Sync(password, salt, ITERATIONS, KEY_LENGTH, DIGEST);
}

// Argon2id for enhanced security (optional)
async function deriveKeyArgon2(password, salt) {
    return await argon2.hash(password, {
        type: argon2.argon2id,
        memoryCost: 2 ** 16,
        timeCost: 3,
        parallelism: 1,
        salt: salt,
        raw: true,
    });
}
```

## Encryption Modes

### 1. Server-Managed Encryption

Default mode with keys managed by DotEnv:

```yaml
# Project configuration
encryption:
    mode: server-managed
    algorithm: AES-256-GCM
    key_rotation:
        enabled: true
        frequency_days: 90
# How it works:
# 1. Unique key generated per project
# 2. Key encrypted with master key
# 3. Master key in HSM
# 4. Automatic rotation
```

#### Implementation

```javascript
class ServerManagedEncryption {
    constructor(projectId) {
        this.projectId = projectId;
        this.keyVersion = null;
    }

    async encrypt(plaintext) {
        // Get current encryption key
        const key = await this.getCurrentKey();

        // Encrypt data
        const encrypted = await crypto.encrypt(plaintext, key);

        // Add metadata
        return {
            version: key.version,
            algorithm: "AES-256-GCM",
            ciphertext: encrypted.toString("base64"),
            keyId: key.id,
        };
    }

    async decrypt(encryptedData) {
        // Get key by version
        const key = await this.getKey(encryptedData.version);

        // Decrypt data
        const ciphertext = Buffer.from(encryptedData.ciphertext, "base64");
        return await crypto.decrypt(ciphertext, key);
    }

    async rotateKey() {
        // Generate new key
        const newKey = await this.generateKey();

        // Re-encrypt all secrets
        await this.reencryptSecrets(newKey);

        // Mark new key as active
        await this.activateKey(newKey);
    }
}
```

### 2. Client-Managed Encryption

End-to-end encryption with keys never leaving the client:

```yaml
# Project configuration
encryption:
    mode: client-managed
    algorithm: AES-256-GCM
    key_storage: client-only
# How it works:
# 1. Key generated on client
# 2. Key never sent to server
# 3. All encryption/decryption client-side
# 4. Zero-knowledge architecture
```

#### Browser Implementation

```javascript
// Browser-based encryption
class ClientEncryption {
    constructor() {
        this.algorithm = {
            name: "AES-GCM",
            length: 256,
        };
    }

    async generateKey() {
        return await crypto.subtle.generateKey(
            this.algorithm,
            true, // extractable
            ["encrypt", "decrypt"],
        );
    }

    async encrypt(plaintext, key) {
        const iv = crypto.getRandomValues(new Uint8Array(12));
        const encoded = new TextEncoder().encode(plaintext);

        const ciphertext = await crypto.subtle.encrypt(
            {
                name: "AES-GCM",
                iv: iv,
            },
            key,
            encoded,
        );

        // Combine IV and ciphertext
        const combined = new Uint8Array(iv.length + ciphertext.byteLength);
        combined.set(iv);
        combined.set(new Uint8Array(ciphertext), iv.length);

        return btoa(String.fromCharCode(...combined));
    }

    async decrypt(encryptedData, key) {
        const combined = Uint8Array.from(atob(encryptedData), (c) =>
            c.charCodeAt(0),
        );

        const iv = combined.slice(0, 12);
        const ciphertext = combined.slice(12);

        const decrypted = await crypto.subtle.decrypt(
            {
                name: "AES-GCM",
                iv: iv,
            },
            key,
            ciphertext,
        );

        return new TextDecoder().decode(decrypted);
    }
}
```

#### CLI Implementation

```go
// CLI-based encryption
package encryption

import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "encoding/base64"
)

type ClientEncryption struct {
    key []byte
}

func (c *ClientEncryption) Encrypt(plaintext string) (string, error) {
    block, err := aes.NewCipher(c.key)
    if err != nil {
        return "", err
    }

    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", err
    }

    nonce := make([]byte, gcm.NonceSize())
    if _, err := rand.Read(nonce); err != nil {
        return "", err
    }

    ciphertext := gcm.Seal(nonce, nonce, []byte(plaintext), nil)
    return base64.StdEncoding.EncodeToString(ciphertext), nil
}

func (c *ClientEncryption) Decrypt(encrypted string) (string, error) {
    ciphertext, err := base64.StdEncoding.DecodeString(encrypted)
    if err != nil {
        return "", err
    }

    block, err := aes.NewCipher(c.key)
    if err != nil {
        return "", err
    }

    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", err
    }

    nonceSize := gcm.NonceSize()
    nonce, ciphertext := ciphertext[:nonceSize], ciphertext[nonceSize:]

    plaintext, err := gcm.Open(nil, nonce, ciphertext, nil)
    if err != nil {
        return "", err
    }

    return string(plaintext), nil
}
```

## Key Management

### Key Generation

```javascript
// Cryptographically secure key generation
class KeyGenerator {
    generateKey() {
        return crypto.randomBytes(32); // 256 bits
    }

    generateKeyWithMetadata() {
        const key = this.generateKey();
        const keyId = crypto.randomBytes(16).toString("hex");

        return {
            id: keyId,
            key: key,
            algorithm: "AES-256-GCM",
            created_at: new Date().toISOString(),
            version: 1,
            status: "active",
        };
    }

    deriveKeyFromPassword(password, salt) {
        return crypto.pbkdf2Sync(
            password,
            salt,
            100000, // iterations
            32, // key length
            "sha256", // digest
        );
    }
}
```

### Key Storage

```javascript
// Secure key storage patterns
class KeyStorage {
    // Server-side HSM storage
    async storeInHSM(key, keyId) {
        const hsm = await this.connectHSM();

        await hsm.importKey({
            keyId: keyId,
            keyMaterial: key,
            keySpec: "AES_256",
            keyUsage: "ENCRYPT_DECRYPT",
        });
    }

    // Client-side browser storage
    async storeInBrowser(key, keyId) {
        // Convert to storable format
        const keyData = await crypto.subtle.exportKey("jwk", key);

        // Store in IndexedDB (not localStorage!)
        const db = await this.openDB();
        const tx = db.transaction("keys", "readwrite");

        await tx.objectStore("keys").put({
            id: keyId,
            key: keyData,
            created: Date.now(),
        });
    }

    // Client-side file storage
    async storeInFile(key, keyId) {
        const keyData = {
            version: "1.0",
            id: keyId,
            algorithm: "AES-256-GCM",
            key: key.toString("base64"),
            created: new Date().toISOString(),
        };

        // Encrypt the key file itself
        const passphrase = await this.getPassphrase();
        const encrypted = await this.encryptKeyFile(keyData, passphrase);

        fs.writeFileSync(
            `~/.dotenv/keys/${keyId}.key`,
            encrypted,
            { mode: 0o600 }, // Read/write for owner only
        );
    }
}
```

### Key Rotation

```javascript
// Automated key rotation
class KeyRotation {
    async rotateKeys(projectId) {
        console.log(`Starting key rotation for project ${projectId}`);

        try {
            // 1. Generate new key
            const newKey = await this.generateNewKey();

            // 2. Get all secrets
            const secrets = await this.getProjectSecrets(projectId);

            // 3. Decrypt with old key, encrypt with new
            const reencrypted = await this.reencryptSecrets(
                secrets,
                this.currentKey,
                newKey,
            );

            // 4. Atomic update
            await this.db.transaction(async (trx) => {
                // Update secrets
                await this.updateSecrets(reencrypted, trx);

                // Update key metadata
                await this.activateNewKey(newKey, trx);

                // Archive old key
                await this.archiveOldKey(this.currentKey, trx);
            });

            // 5. Verify
            await this.verifyRotation(newKey);

            console.log("Key rotation completed successfully");
        } catch (error) {
            console.error("Key rotation failed:", error);
            await this.rollbackRotation();
            throw error;
        }
    }

    async reencryptSecrets(secrets, oldKey, newKey) {
        return Promise.all(
            secrets.map(async (secret) => {
                const decrypted = await this.decrypt(secret.value, oldKey);
                const encrypted = await this.encrypt(decrypted, newKey);

                return {
                    ...secret,
                    value: encrypted,
                    key_version: newKey.version,
                };
            }),
        );
    }
}
```

## Transport Encryption

### TLS Configuration

```nginx
# Nginx TLS configuration
server {
    listen 443 ssl http2;
    server_name api.dotenv.cloud;

    # TLS 1.3 only
    ssl_protocols TLSv1.3;

    # Strong ciphers only
    ssl_ciphers TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256;

    # Enable OCSP stapling
    ssl_stapling on;
    ssl_stapling_verify on;

    # HSTS
    add_header Strict-Transport-Security
        "max-age=31536000; includeSubDomains; preload" always;

    # Certificate pinning
    add_header Public-Key-Pins
        'pin-sha256="base64=="; max-age=31536000' always;
}
```

### API Client Configuration

```javascript
// Secure API client
class SecureAPIClient {
    constructor() {
        this.agent = new https.Agent({
            // Minimum TLS version
            minVersion: "TLSv1.3",

            // Certificate pinning
            checkServerIdentity: (hostname, cert) => {
                const pin = crypto
                    .createHash("sha256")
                    .update(cert.pubkey)
                    .digest("base64");

                if (!VALID_PINS.includes(pin)) {
                    throw new Error("Certificate pin mismatch");
                }
            },
        });
    }

    async request(method, path, data) {
        const response = await fetch(`https://api.dotenv.cloud${path}`, {
            method,
            agent: this.agent,
            headers: {
                "Content-Type": "application/json",
                "X-API-Version": "1.0",
            },
            body: JSON.stringify(data),
        });

        return response.json();
    }
}
```

## Encryption Best Practices

### 1. Key Management

```javascript
// ✅ Good: Separate keys per environment
const keys = {
    development: generateKey(),
    staging: generateKey(),
    production: generateKey(),
};

// ❌ Bad: Shared key across environments
const sharedKey = generateKey();

// ✅ Good: Regular key rotation
schedule.every("90 days").do(async () => {
    await rotateEncryptionKeys();
});

// ❌ Bad: Never rotating keys
const PERMANENT_KEY = "never-changing-key";
```

### 2. Secure Storage

```javascript
// ✅ Good: Encrypted key storage
const encryptedKey = await encryptWithMasterKey(projectKey);
await secureStorage.store(encryptedKey);

// ❌ Bad: Plain text key storage
localStorage.setItem("encryption_key", key);

// ✅ Good: Memory protection
sodium.memzero(key); // Clear key from memory after use

// ❌ Bad: Leaving keys in memory
let globalKey = key; // Key remains in memory
```

### 3. Error Handling

```javascript
// ✅ Good: Secure error handling
try {
  const decrypted = await decrypt(ciphertext, key);
  return decrypted;
} catch (error) {
  logger.error('Decryption failed', {
    error: error.message,
    // Don't log sensitive data
  });
  throw new Error('Unable to decrypt data');
}

// ❌ Bad: Exposing sensitive information
catch (error) {
  console.error('Decryption failed:', {
    key: key,
    ciphertext: ciphertext,
    error: error
  });
}
```

## Compliance Requirements

### FIPS 140-2 Compliance

```javascript
// FIPS-compliant configuration
const fipsConfig = {
    algorithms: {
        encryption: "AES-256-GCM",
        hashing: "SHA-256",
        signing: "RSA-PSS-SHA256",
        keyDerivation: "PBKDF2-HMAC-SHA256",
    },

    minimumKeyLengths: {
        symmetric: 256,
        rsa: 2048,
        ecc: 256,
    },

    randomNumberGeneration: "DRBG-CTR-AES-256",
};
```

### Quantum-Resistant Future

```javascript
// Preparing for post-quantum cryptography
const postQuantumConfig = {
    current: {
        algorithm: "AES-256-GCM",
        keyExchange: "ECDHE",
    },

    future: {
        algorithm: "AES-256-GCM", // Still quantum-resistant
        keyExchange: "Kyber1024", // NIST PQC winner
        signature: "Dilithium3", // NIST PQC winner
    },

    hybrid: {
        // Use both classical and PQC during transition
        keyExchange: ["ECDHE", "Kyber1024"],
        signature: ["RSA-PSS", "Dilithium3"],
    },
};
```

## Troubleshooting

### Common Issues

1. **Decryption Failures**

    ```javascript
    // Check key version mismatch
    if (data.keyVersion !== currentKeyVersion) {
        const historicalKey = await getKeyByVersion(data.keyVersion);
        return decrypt(data.ciphertext, historicalKey);
    }
    ```

2. **Performance Issues**

    ```javascript
    // Use streaming for large data
    const stream = crypto.createDecipheriv(algorithm, key, iv);
    inputStream.pipe(stream).pipe(outputStream);
    ```

3. **Cross-Platform Compatibility**
    ```javascript
    // Ensure consistent encoding
    const encrypted = Buffer.from(data).toString("base64");
    const decrypted = Buffer.from(encrypted, "base64");
    ```

## Resources

- [Encryption Standards](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.197.pdf)
- [Key Management Best Practices](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf)
- [DotEnv Security Whitepaper](https://dotenv.cloud/security/whitepaper)
- [Quantum-Safe Cryptography](https://www.nist.gov/pqcrypto)

## Next Steps

- [Configure access controls](/documentation/v1/security-compliance/access-control)
- [Enable audit logging](/documentation/v1/security-compliance/audit-logging)
- [Review compliance requirements](/documentation/v1/security-compliance/compliance)
- [Implement security best practices](/documentation/v1/security-compliance/security-best-practices)
