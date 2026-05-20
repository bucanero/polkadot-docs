---
title: '@polkadot-apps/keys'
description: Hierarchical key derivation and session key management for Polkadot accounts, including KeyManager, SessionKeyManager, and seedToAccount utilities.
categories: dApps
---

# @polkadot-apps/keys

## Overview

`@polkadot-apps/keys` is a TypeScript package that provides hierarchical key derivation and session key management for Polkadot accounts. Starting from a single source of entropy — typically a wallet signature — the package lets you deterministically derive an unlimited number of purpose-specific child keys, Substrate accounts, and NaCl keypairs, all without ever exposing the source key material to the chain.

The package exposes three main building blocks:

- **[`KeyManager`](#keymanager)** — holds a 32-byte master key in memory and derives child keys via HKDF-SHA256. It does not persist anything; persistence is the caller's responsibility.
- **[`SessionKeyManager`](#sessionkeymanager)** — manages a BIP39 mnemonic-derived Sr25519 account with automatic persistence via a `KvStore`.
- **[`seedToAccount`](#seedtoaccount)** — a standalone function to derive a Polkadot account directly from a BIP39 mnemonic.

## When to Use It

Use `@polkadot-apps/keys` when your dApp needs to:

- **Derive per-user encryption keys from a wallet signature** without asking the user to manage additional secrets. A single signature generates a deterministic master key, and `KeyManager` derives scoped child keys for every purpose in your app (document encryption, HMAC signing, etc.).
- **Create proxy or voting accounts on the fly** without requiring the user to custody a new mnemonic. `km.deriveAccount(context)` produces a fully signable Sr25519 account from the master key for any purpose string.
- **Manage ephemeral session keys** in a browser or native container. `SessionKeyManager` auto-generates a BIP39 mnemonic on first use, persists it via `@polkadot-apps/storage`, and restores it on subsequent visits.
- **Derive NaCl keypairs for end-to-end encryption** (for example, encrypting user data before storing it on-chain or in IPFS) using the same deterministic master key.
- **Restore key material from storage** — export the raw master key bytes once, store them yourself, and reconstruct a `KeyManager` any time via `KeyManager.fromRawKey`.

Do **not** use this package when:

- You only need to connect a wallet and sign transactions — use [`@polkadot-apps/signer`](https://github.com/bucanero/polkadot-apps/tree/main/packages/signer){target=\_blank} instead.
- You need low-level HKDF, AES-GCM, or NaCl primitives directly — use [`@polkadot-apps/crypto`](https://github.com/bucanero/polkadot-apps/tree/main/packages/crypto){target=\_blank} instead.

## Core Concepts

### Master Key and HKDF

The foundation of `@polkadot-apps/keys` is a single **32-byte master key** held in memory by a `KeyManager` instance. All child keys are derived deterministically from this master key using [HKDF-SHA256](https://www.rfc-editor.org/rfc/rfc5869){target=\_blank} (HMAC-based Key Derivation Function).

HKDF takes three inputs:

| Input | Role | Value used by `KeyManager` |
|---|---|---|
| `IKM` (input key material) | The entropy source | The master key (or original signature) |
| `salt` | Domain separator for the extraction step | App-level salt (default: `"polkadot-apps-keys-v1"`) |
| `info` | Context string binding the output to its purpose | Caller-supplied context (e.g. `"document:abc123"`) |

Because derivation is deterministic, the same master key and context always produce the same child key — no matter when or where the derivation happens.

### KeyManager

`KeyManager` is the main class. It wraps a 32-byte master key and provides typed derivation methods:

- **`fromSignature`** — the most common entry point. Takes a wallet signature (≥ 32 bytes) and the signer's SS58 address; runs HKDF to produce the master key. The signature acts as high-entropy input key material that the user's wallet already controls.
- **`fromRawKey`** — restores a `KeyManager` from a previously exported 32-byte key, useful for rehydrating from encrypted storage between sessions.
- **`deriveSymmetricKey(context)`** — returns a 32-byte key suitable for symmetric encryption (e.g. AES-256-GCM or XChaCha20). Different contexts produce cryptographically independent keys.
- **`deriveAccount(context, ss58Prefix?)`** — derives an Sr25519 keypair and wraps it in a `DerivedAccount` with both SS58 and H160 addresses and a ready-to-use `PolkadotSigner`.
- **`deriveKeypairs()`** — derives Curve25519 and Ed25519 keypairs for NaCl Box (asymmetric encryption) and NaCl Sign (digital signatures).
- **`exportKey()`** — exports a copy of the master key as a `Uint8Array` for caller-managed persistence.

### SessionKeyManager

`SessionKeyManager` solves a common dApp pattern: you need a lightweight, auto-rotating Sr25519 account that persists between page loads without asking the user to sign each time.

On the first call to `getOrCreate()`, it generates a fresh BIP39 mnemonic using a cryptographically secure RNG, derives an Sr25519 keypair from it, and stores the mnemonic in a `KvStore`. On subsequent calls it loads the stored mnemonic and returns the same account. Mnemonics are isolated by a `name` option, so a single store can hold multiple independent session keys (for example a `"main"` and a `"burner"` account).

Key points:

- The session key is **only as secure as the underlying `KvStore`**. For sensitive operations, combine session keys with a user-controlled `KeyManager`.
- Call `clear()` to permanently delete the mnemonic from storage (e.g. on user logout).
- `fromMnemonic(mnemonic)` derives an account from an explicit mnemonic without touching storage — useful for import flows or testing.

### seedToAccount

`seedToAccount` is a convenience function for the common case of deriving a single Polkadot account from a BIP39 mnemonic. It is the same derivation used internally by `SessionKeyManager`, exposed as a standalone function for use in scripts, tests, or one-off account generation.

It supports both `sr25519` and `ed25519` key types and a configurable hard derivation path and SS58 prefix.

### DerivedAccount

Every account derivation method returns a `DerivedAccount` object with four fields:

| Field | Type | Description |
|---|---|---|
| `publicKey` | `Uint8Array` (32 bytes) | Raw public key |
| `ss58Address` | `SS58String` | SS58-encoded address (default: generic prefix 42) |
| `h160Address` | `` `0x${string}` `` | H160 EVM address derived via keccak256 |
| `signer` | `PolkadotSigner` | Ready-to-use signer for `polkadot-api` transactions |

### Context Strings and Key Isolation

Child keys derived with different context strings are cryptographically independent — knowledge of one child key does not help an attacker derive any other. This property lets you use a single master key for many purposes without cross-contamination:

```
masterKey
├── deriveSymmetricKey("document:abc123")  → 32-byte encryption key
├── deriveSymmetricKey("hmac:notifications") → 32-byte HMAC key
├── deriveAccount("voting-proxy")          → Sr25519 account
└── deriveKeypairs()                       → Curve25519 + Ed25519 keypairs
```

Use descriptive, namespaced context strings (e.g. `"myapp-v1:document:id"`) to avoid accidental collisions between features or app versions.

## User Journey

### Scenario 1: Per-User Document Encryption

A notes dApp wants to encrypt each user's documents with a key derived from their wallet, so that only the wallet owner can decrypt. No extra secrets need to be managed.

```typescript
import { KeyManager } from "@polkadot-apps/keys";
import { xchachaEncryptPacked, xchachaDecryptPacked } from "@polkadot-apps/crypto";

// 1. Ask the user's wallet to sign a known message
const signature = await wallet.sign("polkadot-apps-keys-v1");

// 2. Build the master key from the signature
const km = KeyManager.fromSignature(signature, userAddress);

// 3. Derive a per-document encryption key
const docKey = km.deriveSymmetricKey("notes:doc-42");

// 4. Encrypt content before storing
const packed = xchachaEncryptPacked(new TextEncoder().encode(content), docKey);

// 5. On reload: re-derive the same key from a fresh signature and decrypt
const loaded = xchachaDecryptPacked(packed, km.deriveSymmetricKey("notes:doc-42"));
console.log(new TextDecoder().decode(loaded));
```

### Scenario 2: Delegated Voting Proxy

A governance dApp needs to submit votes on behalf of the user without requiring them to approve every transaction individually. It derives a proxy account from the master key and funds it with a small amount.

```typescript
import { KeyManager } from "@polkadot-apps/keys";

const km = KeyManager.fromSignature(signature, userAddress);

// Derive a deterministic proxy account for governance
const proxy = km.deriveAccount("governance-proxy-v1", 0); // prefix 0 = Polkadot
console.log("Proxy address:", proxy.ss58Address);

// Fund the proxy account from the user's main account, then submit votes
// using proxy.signer with polkadot-api
```

### Scenario 3: Session Key for Lightweight Actions

A dApp needs to send frequent small transactions (e.g. pings, status updates) without asking the user to sign each one. A `SessionKeyManager` provides a persistent ephemeral account.

```typescript
import { SessionKeyManager } from "@polkadot-apps/keys";
import { createKvStore } from "@polkadot-apps/storage";

const store = await createKvStore({ prefix: "session" });
const skm = new SessionKeyManager({ store, name: "activity" });

// First load: generates and stores a new mnemonic
// Subsequent loads: restores the same account from storage
const { account } = await skm.getOrCreate();
console.log("Session account:", account.ss58Address);

// Use account.signer for lightweight transactions
// On logout:
await skm.clear();
```

### Scenario 4: End-to-End Encrypted Messaging

Two users want to send encrypted messages using public keys exchanged off-chain. Each derives their NaCl keypair from their `KeyManager` and shares only the public key.

```typescript
import { KeyManager } from "@polkadot-apps/keys";
import { nacl } from "@polkadot-apps/crypto";

// Alice derives her keypair
const aliceKm = KeyManager.fromSignature(aliceSig, aliceAddress);
const aliceKp = aliceKm.deriveKeypairs();

// Bob derives his keypair
const bobKm = KeyManager.fromSignature(bobSig, bobAddress);
const bobKp = bobKm.deriveKeypairs();

// Alice encrypts a message for Bob using Bob's public encryption key
const nonce = nacl.randomBytes(24);
const ciphertext = nacl.box(
  new TextEncoder().encode("Hello Bob!"),
  nonce,
  bobKp.encryption.publicKey,   // Bob's public key (shared)
  aliceKp.encryption.secretKey, // Alice's private key (local only)
);

// Bob decrypts the message
const plaintext = nacl.box.open(
  ciphertext,
  nonce,
  aliceKp.encryption.publicKey,  // Alice's public key (shared)
  bobKp.encryption.secretKey,    // Bob's private key (local only)
);
console.log(new TextDecoder().decode(plaintext!)); // "Hello Bob!"
```

### Scenario 5: Persisting and Restoring a Master Key

When a user closes the browser, the `KeyManager` (which holds the master key only in memory) is lost. Rather than asking for a new signature on every visit, the app exports and encrypts the master key in local storage.

```typescript
import { KeyManager } from "@polkadot-apps/keys";
import { xchachaEncryptPacked, xchachaDecryptPacked } from "@polkadot-apps/crypto";

// On first visit: derive from signature and persist
const km = KeyManager.fromSignature(signature, userAddress);
const raw = km.exportKey();

// Encrypt the raw key with a storage key derived from the same signature
const storageKey = km.deriveSymmetricKey("local-storage-lock");
const locked = xchachaEncryptPacked(raw, storageKey);
localStorage.setItem("masterKey", Buffer.from(locked).toString("base64"));

// On subsequent visits: restore without a new signature
const stored = Buffer.from(localStorage.getItem("masterKey")!, "base64");
// Derive the storage key from the user's signature (one sign per session)
const restoredKm = KeyManager.fromSignature(signature, userAddress);
const unlocked = xchachaDecryptPacked(stored, restoredKm.deriveSymmetricKey("local-storage-lock"));
const restoredFromStorage = KeyManager.fromRawKey(unlocked);
```

## API Reference

For the full, annotated source, see the [TypeScript source files on GitHub](https://github.com/bucanero/polkadot-apps/tree/main/packages/keys/src){target=\_blank}.

### Install

```bash
pnpm add @polkadot-apps/keys
```

### KeyManager

| Member | Signature | Returns | Description |
|---|---|---|---|
| `KeyManager.fromSignature` | `(signature: Uint8Array \| string, signerAddress: string, options?: { salt?: string })` | `KeyManager` | Create from a wallet signature. `signature` must be ≥ 32 bytes. Default salt: `"polkadot-apps-keys-v1"`. |
| `KeyManager.fromRawKey` | `(masterKey: Uint8Array)` | `KeyManager` | Restore from a stored 32-byte key. |
| `km.deriveSymmetricKey` | `(context: string)` | `Uint8Array` (32 bytes) | Derive a 32-byte symmetric key for `context`. |
| `km.deriveAccount` | `(context: string, ss58Prefix?: number)` | `DerivedAccount` | Derive an Sr25519 account. Default `ss58Prefix`: `42`. |
| `km.deriveKeypairs` | `()` | `DerivedKeypairs` | Derive Curve25519 + Ed25519 NaCl keypairs. |
| `km.exportKey` | `()` | `Uint8Array` (32 bytes) | Export a copy of the master key for caller-managed persistence. |

Source: [`src/key-manager.ts`](https://github.com/bucanero/polkadot-apps/blob/main/packages/keys/src/key-manager.ts){target=\_blank}

### SessionKeyManager

| Member | Signature | Returns | Description |
|---|---|---|---|
| `constructor` | `(options: { store: KvStore, name?: string })` | `SessionKeyManager` | Default `name`: `"default"`. |
| `skm.create` | `()` | `Promise<SessionKeyInfo>` | Generate a new mnemonic and persist it. |
| `skm.get` | `()` | `Promise<SessionKeyInfo \| null>` | Load an existing session key; `null` if none stored. |
| `skm.getOrCreate` | `()` | `Promise<SessionKeyInfo>` | Load existing or create and persist a new session key. |
| `skm.fromMnemonic` | `(mnemonic: string)` | `SessionKeyInfo` | Derive from an explicit mnemonic; no storage interaction. |
| `skm.clear` | `()` | `Promise<void>` | Delete the stored mnemonic. |

Source: [`src/session-key-manager.ts`](https://github.com/bucanero/polkadot-apps/blob/main/packages/keys/src/session-key-manager.ts){target=\_blank}

### seedToAccount

| Function | Signature | Returns | Description |
|---|---|---|---|
| `seedToAccount` | `(mnemonic: string, derivationPath?: string, ss58Prefix?: number, keyType?: "sr25519" \| "ed25519")` | `DerivedAccount` | Derive a `DerivedAccount` from a BIP39 mnemonic. Defaults: `derivationPath = "//0"`, `ss58Prefix = 42`, `keyType = "sr25519"`. |

Source: [`src/seed-to-account.ts`](https://github.com/bucanero/polkadot-apps/blob/main/packages/keys/src/seed-to-account.ts){target=\_blank}

### Types

Source: [`src/types.ts`](https://github.com/bucanero/polkadot-apps/blob/main/packages/keys/src/types.ts){target=\_blank}

```typescript
/** Derivation result for a Substrate/EVM account. */
interface DerivedAccount {
  /** Raw 32-byte public key. Sr25519 or Ed25519 depending on key type. */
  publicKey: Uint8Array;
  /** SS58-encoded address (generic prefix 42 by default). */
  ss58Address: SS58String;
  /** H160 EVM address derived via keccak256(publicKey). */
  h160Address: `0x${string}`;
  /** Ready-to-use PolkadotSigner for polkadot-api transactions. */
  signer: PolkadotSigner;
}

/** NaCl encryption + signing keypairs derived from a KeyManager. */
interface DerivedKeypairs {
  /** Curve25519 keypair for NaCl Box (asymmetric encryption). */
  encryption: { publicKey: Uint8Array; secretKey: Uint8Array };
  /** Ed25519 keypair for NaCl Sign (digital signatures). */
  signing: { publicKey: Uint8Array; secretKey: Uint8Array };
}

/** Returned by SessionKeyManager methods. */
interface SessionKeyInfo {
  /** The BIP39 mnemonic (the only value that needs persisting). */
  mnemonic: string;
  /** The derived account. */
  account: DerivedAccount;
}
```

## Related Packages

| Package | Description | Use together with `@polkadot-apps/keys` when… |
|---|---|---|
| [`@polkadot-apps/crypto`](https://github.com/bucanero/polkadot-apps/tree/main/packages/crypto){target=\_blank} | Symmetric encryption (AES-256-GCM, XChaCha20-Poly1305), HKDF, and NaCl primitives | You need to actually encrypt/decrypt data using the keys derived by `KeyManager`. |
| [`@polkadot-apps/address`](https://github.com/bucanero/polkadot-apps/tree/main/packages/address){target=\_blank} | SS58 and H160 address encoding, validation, and conversion | You need to validate, re-encode, or display the addresses returned in `DerivedAccount`. |
| [`@polkadot-apps/storage`](https://github.com/bucanero/polkadot-apps/tree/main/packages/storage){target=\_blank} | `KvStore` abstraction with browser/host/memory backends | You want `SessionKeyManager` to persist session keys across page loads. |
| [`@polkadot-apps/signer`](https://github.com/bucanero/polkadot-apps/tree/main/packages/signer){target=\_blank} | Multi-provider wallet signer (browser extension, Host API, dev accounts) | You need to connect a user's existing wallet to obtain the signature used in `KeyManager.fromSignature`. |
