---
title: Key Management (Keys)
description: Hierarchical key derivation and session key management for Polkadot accounts.
---

# 🔑 Key Management (`@polkadot-apps/keys`)

Use this package to derive application-specific keys from a single user signature or to manage long-lived session keys with automatic storage.

## When to Use This Package

| **User Journey** | **Primary Class/Method** | **Scenario** |
| :--- | :--- | :--- |
| Derive deterministic keys for app features | `KeyManager.deriveAccount()` / `deriveSymmetricKey()` | You have a user's wallet signature and need to create sub-accounts (e.g., "voting", "gaming") or encryption keys for their private data. |
| Persist a user's session key | `SessionKeyManager` | Your dApp needs a consistent key representing the user across browser restarts without prompting them to sign every time. |
| Generate an account from a mnemonic | `seedToAccount()` | You are handling a traditional mnemonic phrase (e.g., from a wallet import) and need to derive a keypair. |

## Core Concepts

### 1. `KeyManager`: Stateless, Deterministic Derivation

The `KeyManager` holds a **master key** in memory. It never writes to disk. Given the same master key and context string, it always derives the same child key.

*   **From a signature (recommended):** Derives the master key from a wallet signature. This is secure because the signature proves the user controls their wallet. The master key is different for each `signerAddress` and `salt`.
    ```typescript
    const km = KeyManager.fromSignature(signature, userAddress);
    ```

*   **From a raw key:** Used when you have previously exported and persisted a master key (e.g., in the browser's `localStorage` via the `storage` package).
    ```typescript
    const persistedKey = await store.get("masterKey"); // Uint8Array
    const km = KeyManager.fromRawKey(persistedKey);
    ```

### 2. `SessionKeyManager`: Persistent, User-Specific Keys

This class manages a **BIP39 mnemonic** and its derived Sr25519 account, storing the mnemonic securely via a `KvStore` (from the `@polkadot-apps/storage` package). It's ideal for representing the "current user" of your dApp.

*   **`getOrCreate()`** is the primary method. It will either:
    1.  Load an existing mnemonic from the store and return its account.
    2.  Generate a new random mnemonic, save it, and return the new account.
*   Use the `name` parameter in the constructor to manage **multiple independent keys** (e.g., one for "main profile", one for "burner wallet") in the same storage backend.

## Complete User Journeys

### Journey 1: Build a "Proxy Voting" dApp (Using `KeyManager`)

**Goal:** A user signs a message once, and your app derives a unique "voting proxy" account for them on the Polkadot chain.

1.  **User connects wallet** (using `@polkadot-apps/signer`) and you obtain their `signerAddress`.
2.  **User signs a simple message** (e.g., "Authorize Voting App").
3.  **Derive the master key and then the voting account:**
    ```typescript
    import { KeyManager } from "@polkadot-apps/keys";

    // signature is the Uint8Array of the signed message
    const km = KeyManager.fromSignature(signature, signerAddress);
    const votingAccount = km.deriveAccount("voting-proxy", 0); // 0 = Polkadot prefix

    // Use votingAccount.signer with @polkadot-apps/tx to submit votes
    console.log(`Voting with account: ${votingAccount.ss58Address}`);
    ```
4.  **No storage needed:** Each time the user returns and signs the same message (or you cache the signature), `deriveAccount()` produces the same deterministic address.

### Journey 2: Add "End-to-End Encrypted Notes" (Using `deriveSymmetricKey`)

**Goal:** Encrypt user-specific notes so that only the user (and your app, with their consent) can decrypt them.

```typescript
// After obtaining a KeyManager (km) as above...
const noteEncryptionKey = km.deriveSymmetricKey("user-notes:v1");

// Encrypt a note (using @polkadot-apps/crypto or Web Crypto API)
const encryptedNote = await encrypt(noteText, noteEncryptionKey);
await saveToCloud(encryptedNote);

// To decrypt later, derive the same key again from the signature
const decryptionKey = km.deriveSymmetricKey("user-notes:v1");
const decrypted = await decrypt(encryptedNote, decryptionKey);
```

### Journey 3: Create a Persistent "Burner Wallet" (Using `SessionKeyManager`)

**Goal:** Your dApp creates a new, one-time-use account for a user, and remembers it across browser sessions.

```typescript
import { SessionKeyManager } from "@polkadot-apps/keys";
import { createKvStore } from "@polkadot-apps/storage";

// 1. Setup storage (host API will automatically pick the right backend)
const store = await createKvStore({ prefix: "my-dapp" });

// 2. Create a SessionKeyManager for a "burner" profile
const burnerManager = new SessionKeyManager({ store, name: "burner-wallet" });

// 3. Get or create the session key
const { mnemonic, account } = await burnerManager.getOrCreate();
console.log(`Burner address: ${account.ss58Address}`);

// 4. Use account.signer to sign transactions (e.g., for a faucet or low-value interactions)

// 5. Optional: Clear the key if the user wants to reset
// await burnerManager.clear();
```

## API Reference

For detailed method signatures and return types, refer to the [API documentation](https://paritytech.github.io/polkadot-apps/modules/_polkadot-apps_keys.html). Key interfaces are summarized below:

| Class | Key Methods |
| :--- | :--- |
| `KeyManager` | `fromSignature()`, `fromRawKey()`, `deriveAccount()`, `deriveSymmetricKey()`, `exportKey()` |
| `SessionKeyManager` | `getOrCreate()`, `fromMnemonic()`, `clear()` |
| Function | `seedToAccount()` |

## Related Packages

*   [`@polkadot-apps/signer`](https://github.com/paritytech/polkadot-apps/tree/main/packages/signer): For obtaining wallet signatures and managing signers.
*   [`@polkadot-apps/storage`](https://github.com/paritytech/polkadot-apps/tree/main/packages/storage): The `KvStore` used by `SessionKeyManager`.
*   [`@polkadot-apps/crypto`](https://github.com/paritytech/polkadot-apps/tree/main/packages/crypto): For encryption/decryption using derived keys.
*   [`@polkadot-apps/tx`](https://github.com/paritytech/polkadot-apps/tree/main/packages/tx): For submitting transactions using the derived `account.signer`.

---
