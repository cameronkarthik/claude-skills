---
name: solana-browser-wallet
description: Use when building ephemeral browser-based Solana wallets, implementing client-side key management, or designing wallet persistence with IndexedDB vaults. Covers key generation, encrypted storage, and recovery patterns.
version: 0.1.0
tools: Read, Grep, Glob, Bash
---

# Solana Browser Wallet Patterns

Patterns for building ephemeral, browser-based Solana wallet systems. Designed for applications that need to generate and manage multiple keypairs client-side without ever sending private keys to a server.

## When to Use

- Building a multi-wallet tool that generates keypairs in the browser
- Implementing client-side key management for Solana
- Adding encrypted wallet persistence (IndexedDB vault)
- Designing recovery flows for browser-based wallets
- Any scenario where private keys must never leave the client

## Architecture Principles

1. **Keys live in memory.** Generate `Keypair` objects in a Zustand vanilla store (or equivalent). Never serialize to localStorage, cookies, or API calls.
2. **Persistence is opt-in.** Use an encrypted IndexedDB vault for users who need keys to survive page refreshes (e.g., holding positions overnight).
3. **Server sees public keys only.** API routes receive wallet addresses for tracking/display. Private keys stay in the browser runtime.
4. **Recovery is always available.** Export buttons, seed display, and sweep functions must be accessible at every stage -- especially error states.

## In-Memory Wallet Store

The core pattern uses a framework-agnostic store (Zustand vanilla) to hold keypairs:

```typescript
import { createStore } from 'zustand/vanilla';
import { Keypair } from '@solana/web3.js';

interface WalletState {
  wallets: { sol: Keypair; evm: { address: string; privateKey: string } }[];
  generate: (count: number) => void;
  clear: () => void;
}

const walletStore = createStore<WalletState>((set) => ({
  wallets: [],
  generate: (count) => {
    const wallets = Array.from({ length: count }, () => ({
      sol: Keypair.generate(),
      evm: generateEvmWallet(), // viem or ethers
    }));
    set({ wallets });
  },
  clear: () => set({ wallets: [] }),
}));
```

### Critical Rule: Never Clear on Error

```
WRONG: handleCancel() { reset(); walletStore.clear(); }
RIGHT: handleCancel() { try { sweep(); } finally { /* keys stay in memory */ } }
```

Clearing wallets on error/cancel means losing access to funded wallets. Only clear after confirmed successful consolidation (sweep all funds back to the user's main wallet).

## Encrypted IndexedDB Vault

For applications where users may hold positions for hours/days, implement an encrypted vault:

### Key Derivation

Derive an AES-GCM encryption key from a wallet signature (the user's connected Phantom/Solflare wallet signs a deterministic message):

```typescript
const VAULT_MESSAGE = 'Sign to unlock your wallet vault';

async function deriveVaultKey(walletAdapter: WalletAdapter): Promise<CryptoKey> {
  // User signs a fixed message with their main wallet
  const signature = await walletAdapter.signMessage(
    new TextEncoder().encode(VAULT_MESSAGE)
  );

  // Use signature as key material
  const keyMaterial = await crypto.subtle.importKey(
    'raw', signature, 'HKDF', false, ['deriveKey']
  );

  return crypto.subtle.deriveKey(
    { name: 'HKDF', hash: 'SHA-256', salt: new Uint8Array(32), info: new TextEncoder().encode('vault') },
    keyMaterial,
    { name: 'AES-GCM', length: 256 },
    false,
    ['encrypt', 'decrypt']
  );
}
```

### Vault Operations

```typescript
async function saveToVault(wallets: WalletData[], vaultKey: CryptoKey): Promise<void> {
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const plaintext = new TextEncoder().encode(JSON.stringify(
    wallets.map(w => ({
      solSecret: Array.from(w.sol.secretKey),
      evmKey: w.evm.privateKey,
    }))
  ));

  const ciphertext = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv }, vaultKey, plaintext
  );

  // Store in IndexedDB (not localStorage -- survives larger payloads)
  await idbSet('wallet-vault', { iv: Array.from(iv), data: Array.from(new Uint8Array(ciphertext)) });
}

async function loadFromVault(vaultKey: CryptoKey): Promise<WalletData[]> {
  const stored = await idbGet('wallet-vault');
  if (!stored) return [];

  const plaintext = await crypto.subtle.decrypt(
    { name: 'AES-GCM', iv: new Uint8Array(stored.iv) },
    vaultKey,
    new Uint8Array(stored.data)
  );

  const parsed = JSON.parse(new TextDecoder().decode(plaintext));
  return parsed.map((w: any) => ({
    sol: Keypair.fromSecretKey(new Uint8Array(w.solSecret)),
    evm: { privateKey: w.evmKey, address: deriveAddress(w.evmKey) },
  }));
}
```

### Vault Lifecycle

```
User connects wallet
  -> Sign vault message (derive AES key)
  -> Check IndexedDB for existing vault
    -> Found: decrypt and restore to memory store
    -> Not found: generate fresh wallets, encrypt and save

On wallet generation:
  -> Auto-save to vault after every generation

On consolidation (sweep complete):
  -> Wipe vault from IndexedDB
  -> Clear memory store

On disconnect / page close:
  -> Keys remain in IndexedDB (encrypted)
  -> Memory store is garbage collected
```

## What Destroys the Vault

| Action | Vault Safe? |
|---|---|
| Page refresh | Yes (IndexedDB persists) |
| Browser restart | Yes |
| Code deploy / git pull | Yes (IndexedDB is separate from app code) |
| `Clear site data` in browser | **NO** -- vault is destroyed |
| Different browser | **NO** -- IndexedDB is per-browser |
| Incognito mode | **NO** -- IndexedDB is ephemeral |

Always warn users about these edge cases in the UI.

## Recovery Patterns

### Per-Wallet Key Display

Every wallet row in the UI must have a "Reveal Key" button that shows the base58-encoded secret key:

```typescript
function revealKey(keypair: Keypair): string {
  return bs58.encode(keypair.secretKey);
}
```

### Export All Keys

A single button that exports all wallet keys as a downloadable JSON:

```typescript
function exportAllKeys(wallets: WalletData[]): void {
  const data = wallets.map((w, i) => ({
    index: i,
    solana: { address: w.sol.publicKey.toBase58(), secretKey: bs58.encode(w.sol.secretKey) },
    evm: { address: w.evm.address, privateKey: w.evm.privateKey },
  }));

  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  // Trigger download
  const a = document.createElement('a');
  a.href = url;
  a.download = `wallet-export-${Date.now()}.json`;
  a.click();
  URL.revokeObjectURL(url);
}
```

### Emergency Sweep

When things go wrong (bridge errors, stuck transactions), sweep all remaining balances back to the user's main wallet:

```typescript
async function emergencySweep(wallets: WalletData[], destination: PublicKey, connection: Connection): Promise<string[]> {
  const signatures: string[] = [];

  for (const wallet of wallets) {
    const balance = await connection.getBalance(wallet.sol.publicKey);
    if (balance <= 5000) continue; // Skip empty wallets (5000 = tx fee)

    const tx = new Transaction().add(
      SystemProgram.transfer({
        fromPubkey: wallet.sol.publicKey,
        toPubkey: destination,
        lamports: balance - 5000, // Leave enough for the fee
      })
    );

    const sig = await sendAndConfirmTransaction(connection, tx, [wallet.sol]);
    signatures.push(sig);
  }

  return signatures;
}
```

### Recovery Button Visibility

The recovery/sweep button must be visible at ALL pipeline stages, not just specific ones:

```
WRONG: {stage === 'BUYING' && <RecoveryButton />}
RIGHT: {stage !== 'IDLE' && stage !== 'COMPLETE' && <RecoveryButton />}
```

Users can lose funds at any stage. Recovery must always be one click away.

## Security Checklist

- [ ] Private keys never sent to API routes (verify with network tab)
- [ ] No `console.log` of secret keys in production
- [ ] Vault encryption uses wallet-derived key (not hardcoded)
- [ ] `clear()` only called after confirmed successful sweep
- [ ] Export button accessible at every non-idle stage
- [ ] IndexedDB vault wiped on successful consolidation
- [ ] CSP headers block inline scripts in production
- [ ] No `Buffer.from` in browser code (use Uint8Array)

## Browser Compatibility Notes

- `crypto.subtle` requires HTTPS in production (works on localhost in dev)
- IndexedDB has a ~50MB limit per origin in most browsers
- Safari has stricter IndexedDB eviction policies -- warn users
- Web Workers can access `crypto.subtle` and IndexedDB for isolated key operations
