---
name: solana-anchor-deployment
description: Use when deploying Anchor programs to Solana devnet or mainnet, initializing PDAs, managing keypairs, or configuring RPC providers. Covers the full deployment lifecycle from build to GlobalConfig initialization.
version: 0.1.0
tools: Read, Grep, Glob, Bash, WebFetch
---

# Solana Anchor Deployment

Battle-tested deployment guide for Anchor programs on Solana. Built from deploying a multiplayer game contract across devnet and mainnet, debugging PDA initialization failures, managing keypairs securely, and fighting RPC configuration issues.

## When to Use

- Deploying an Anchor program to devnet or mainnet
- Initializing GlobalConfig or other singleton PDAs after deployment
- Managing server-side signing keypairs (Ed25519)
- Configuring Helius or other RPC providers
- Migrating from devnet to mainnet
- Debugging deployment failures (insufficient funds, wrong wallet, wrong network)

## Pre-Flight Checklist

Before deploying anything:

1. Which cluster? (`devnet` vs `mainnet-beta`) -- confirm in `Anchor.toml`
2. Which wallet is paying? (`~/.config/solana/id.json` or custom path)
3. Does the wallet have enough SOL? (program deploy costs 2-5 SOL depending on size)
4. Is the program ID correct? (must match `declare_id!()` in `lib.rs` AND `Anchor.toml`)
5. Are all PDAs derivable? (seeds must match between program and client code)

## Anchor.toml Configuration

```toml
[features]
seeds = false
skip-lint = false

[programs.devnet]
my_program = "YourProgramId111111111111111111111111111111"

[programs.mainnet]
my_program = "YourProgramId111111111111111111111111111111"

[provider]
cluster = "devnet"
wallet = "~/.config/solana/id.json"

[scripts]
test = "anchor test"
```

**Critical:** The program ID in `[programs.devnet]` and `[programs.mainnet]` MUST match the `declare_id!()` in your Rust code. If they differ, deployment succeeds but all PDA derivations will fail silently.

## Deployment Steps

### 1. Build

```bash
cd solana-program
anchor build
```

This generates:
- `target/deploy/my_program-keypair.json` (program keypair)
- `target/deploy/my_program.so` (compiled BPF)
- `target/idl/my_program.json` (IDL for client)
- `target/types/my_program.ts` (TypeScript types)

### 2. Verify Program ID

```bash
solana address -k target/deploy/my_program-keypair.json
```

This MUST match your `declare_id!()`. If it doesn't:

```bash
# Option A: Update declare_id to match the keypair
# Option B: Generate a new keypair and rebuild
solana-keygen new -o target/deploy/my_program-keypair.json --force
# Then update declare_id!() and Anchor.toml with the new address
anchor build  # Rebuild with new ID
```

### 3. Fund the Deploy Wallet

```bash
# Check balance
solana balance

# Devnet: airdrop
solana airdrop 5

# Mainnet: transfer from exchange or another wallet
```

**Cost estimates:**
- Small program (~50KB): ~1 SOL
- Medium program (~200KB): ~3 SOL
- Large program (~500KB): ~5+ SOL
- Each redeployment (upgrade): similar cost

### 4. Deploy

```bash
# Devnet
anchor deploy --provider.cluster devnet

# Mainnet
anchor deploy --provider.cluster mainnet
```

**If deployment fails with "insufficient funds":**
- The error shows the exact amount needed
- Fund the wallet with that amount + 0.5 SOL buffer
- Retry the deploy (it's idempotent)

**If deployment hangs:**
- RPC might be congested. Switch to a paid RPC (Helius, QuickNode)
- Set priority fees: `solana config set --url YOUR_RPC_URL`

### 5. Copy IDL to Client

```bash
cp target/idl/my_program.json ../lib/solana-program/idl.json
```

The IDL is the contract between your program and client code. Always copy after build.

## GlobalConfig Initialization

Most Anchor programs need a one-time initialization of a singleton config PDA.

### Pattern

```rust
#[account]
pub struct GlobalConfig {
    pub authority: Pubkey,      // Who can update config
    pub game_signer: Pubkey,    // Ed25519 key for backend attestation
    pub fee_vault: Pubkey,      // Fee collection wallet
    pub fee_bps: u16,           // Fee in basis points (500 = 5%)
    pub bump: u8,               // PDA bump
}

impl GlobalConfig {
    pub const SIZE: usize = 8 + 32 + 32 + 32 + 2 + 1; // 107 bytes
}
```

### Initialization Script

```typescript
// scripts/initialize.ts
import { Connection, Keypair, PublicKey } from '@solana/web3.js';
import { AnchorProvider, Program, Wallet } from '@coral-xyz/anchor';
import * as fs from 'fs';

const PROGRAM_ID = new PublicKey(process.env.SOLANA_PROGRAM_ID!);
const RPC_URL = process.env.SOLANA_RPC_URL ?? 'https://api.devnet.solana.com';

async function main() {
  // Load deploy wallet
  const walletKeyfile = process.env.SOLANA_WALLET_PATH ??
    `${process.env.HOME}/.config/solana/id.json`;
  const secretKey = Uint8Array.from(
    JSON.parse(fs.readFileSync(walletKeyfile, 'utf-8'))
  );
  const wallet = Keypair.fromSecretKey(secretKey);

  const connection = new Connection(RPC_URL, 'confirmed');
  const provider = new AnchorProvider(
    connection,
    new Wallet(wallet),
    { commitment: 'confirmed' }
  );

  // Load IDL
  const idl = JSON.parse(
    fs.readFileSync('target/idl/my_program.json', 'utf-8')
  );
  const program = new Program(idl, PROGRAM_ID, provider);

  // Derive config PDA
  const [configPda] = PublicKey.findProgramAddressSync(
    [Buffer.from('config')],
    PROGRAM_ID
  );

  // Check if already initialized
  const existing = await connection.getAccountInfo(configPda);
  if (existing) {
    console.log('Config already initialized at:', configPda.toBase58());
    return;
  }

  // Initialize
  const gameSigner = new PublicKey(process.env.SOLANA_GAME_SIGNER_PUBKEY!);
  const feeVault = new PublicKey(process.env.SOLANA_FEE_VAULT_PUBKEY!);
  const feeBps = parseInt(process.env.SOLANA_FEE_BPS ?? '500');

  const tx = await program.methods
    .initialize(gameSigner, feeVault, feeBps)
    .accounts({
      config: configPda,
      authority: wallet.publicKey,
      systemProgram: SystemProgram.programId,
    })
    .rpc();

  console.log('Initialized config:', tx);
  console.log('Config PDA:', configPda.toBase58());
}

main().catch(console.error);
```

**Run:**
```bash
SOLANA_PROGRAM_ID=... \
SOLANA_GAME_SIGNER_PUBKEY=... \
SOLANA_FEE_VAULT_PUBKEY=... \
npx ts-node scripts/initialize.ts
```

### Common Initialization Failures

| Error | Cause | Fix |
|---|---|---|
| `Account already exists` | Config PDA already initialized | Safe to skip (idempotent) |
| `Insufficient funds` | Wallet can't pay rent | Fund wallet with ~0.01 SOL |
| `Program not found` | Wrong PROGRAM_ID or not deployed | Verify program ID matches deployment |
| `Invalid seeds` | PDA derivation mismatch | Check seed bytes match exactly between program and script |

## PDA Design Patterns

### Singleton Config

```rust
seeds = [b"config"]
// One per program. Stores global settings.
```

### Per-Entity Account

```rust
seeds = [b"game", game_id.as_ref()]
// One per game/session. game_id is typically a hash of a room code.
```

### Nonce Account (Replay Prevention)

```rust
seeds = [b"nonce", nonce.as_ref()]
// One per signature. Initialized on first use, prevents replay.
```

### Game ID Derivation

```typescript
// Solana uses SHA-256 (NOT keccak256 like EVM)
import { createHash } from 'crypto';

export function roomCodeToGameId(roomCode: string): Buffer {
  return createHash('sha256').update(roomCode).digest();
}

export function getGamePda(gameId: Buffer): PublicKey {
  const [pda] = PublicKey.findProgramAddressSync(
    [Buffer.from('game'), gameId],
    PROGRAM_ID
  );
  return pda;
}
```

**Critical difference from EVM:** Solana uses SHA-256, not keccak256. If your EVM contract uses `keccak256(roomCode)` for game IDs, your Solana program MUST use SHA-256. These produce different hashes. Client code must use the right hash for each chain.

## Ed25519 Signing (Backend Attestation)

### Keypair Generation

```bash
# Generate a new Ed25519 keypair for server signing
node -e "
const nacl = require('tweetnacl');
const bs58 = require('bs58');
const kp = nacl.sign.keyPair();
console.log('Public key:', bs58.encode(kp.publicKey));
console.log('Secret key:', JSON.stringify(Array.from(kp.secretKey)));
"
```

Store the secret key in environment or secure key file. The public key goes into GlobalConfig as `game_signer`.

### Server-Side Signing

```typescript
import nacl from 'tweetnacl';
import bs58 from 'bs58';
import { randomBytes } from 'crypto';
import { PublicKey } from '@solana/web3.js';

function loadSignerKeypair(): nacl.SignKeyPair {
  const raw = process.env.SOLANA_GAME_SIGNER_SECRET!;
  let secretKey: Uint8Array;
  // Support both JSON array and base58 formats
  if (raw.startsWith('[')) {
    secretKey = Uint8Array.from(JSON.parse(raw));
  } else {
    secretKey = bs58.decode(raw);
  }
  return nacl.sign.keyPair.fromSecretKey(secretKey);
}

function signSettlement(roomCode: string, winnerPubkey: string) {
  const keypair = loadSignerKeypair();
  const gameId = roomCodeToGameId(roomCode);
  const nonce = randomBytes(32);
  const winner = new PublicKey(winnerPubkey);

  // Message format: gameId(32) || winner(32) || nonce(32) || programId(32)
  const message = Buffer.concat([
    gameId,
    winner.toBuffer(),
    nonce,
    PROGRAM_ID.toBuffer(),
  ]);

  const signature = nacl.sign.detached(message, keypair.secretKey);

  return {
    nonce: bs58.encode(nonce),
    signature: bs58.encode(signature),
    gameId: '0x' + gameId.toString('hex'),
  };
}
```

### On-Chain Verification

The program verifies Ed25519 signatures using the Ed25519 precompile:

```rust
fn verify_ed25519_signature(
    ix_sysvar: &AccountInfo,
    expected_signer: &Pubkey,
    expected_message: &[u8],
) -> Result<()> {
    // Search instructions sysvar for Ed25519 precompile
    let mut ed25519_ix: Option<Instruction> = None;
    for i in 0u16..10 {
        match load_instruction_at_checked(i as usize, ix_sysvar) {
            Ok(ix) if ix.program_id == ed25519_program::ID => {
                ed25519_ix = Some(ix);
                break;
            }
            Ok(_) => continue,
            Err(_) => break,
        }
    }
    let ix = ed25519_ix.ok_or(MyError::InvalidEd25519Program)?;

    // Parse header: num_sigs(1) | pad(1) | offsets...
    require!(ix.data.len() >= 16, MyError::InvalidEd25519Data);
    require!(ix.data[0] == 1, MyError::InvalidEd25519Data); // exactly 1 signature

    let sig_ix_index = u16::from_le_bytes([ix.data[4], ix.data[5]]);
    let pubkey_offset = u16::from_le_bytes([ix.data[6], ix.data[7]]) as usize;
    let pubkey_ix_index = u16::from_le_bytes([ix.data[8], ix.data[9]]);
    let msg_offset = u16::from_le_bytes([ix.data[10], ix.data[11]]) as usize;
    let msg_size = u16::from_le_bytes([ix.data[12], ix.data[13]]) as usize;
    let msg_ix_index = u16::from_le_bytes([ix.data[14], ix.data[15]]);

    // CRITICAL: All ix_index must be 0xFFFF (self-referencing)
    // Prevents attackers from referencing arbitrary instruction data
    require!(
        sig_ix_index == 0xFFFF && pubkey_ix_index == 0xFFFF && msg_ix_index == 0xFFFF,
        MyError::InvalidEd25519Data
    );

    // Verify public key matches expected signer
    let ix_pubkey = &ix.data[pubkey_offset..pubkey_offset + 32];
    require!(ix_pubkey == expected_signer.as_ref(), MyError::InvalidSigner);

    // Verify message matches expected
    let ix_message = &ix.data[msg_offset..msg_offset + msg_size];
    require!(ix_message == expected_message, MyError::InvalidMessage);

    Ok(())
}
```

### Client-Side Transaction Construction

```typescript
import { Ed25519Program } from '@solana/web3.js';

async function claimWinnings(
  wallet: AnchorWallet,
  roomCode: string,
  nonceB58: string,
  signatureB58: string,
  signerPubkeyB58: string
): Promise<string> {
  const nonce = bs58.decode(nonceB58);
  const signature = bs58.decode(signatureB58);
  const signerPubkey = new PublicKey(signerPubkeyB58);

  const gameId = roomCodeToGameId(roomCode);
  const message = Buffer.concat([
    gameId,
    wallet.publicKey.toBuffer(),
    Buffer.from(nonce),
    PROGRAM_ID.toBuffer(),
  ]);

  // Ed25519 precompile instruction MUST be before the claim instruction
  const ed25519Ix = Ed25519Program.createInstructionWithPublicKey({
    publicKey: signerPubkey.toBytes(),
    message,
    signature,
  });

  const claimIx = await program.methods
    .claimWinnings(Array.from(nonce) as any)
    .accounts({ /* ... */ })
    .instruction();

  const tx = new Transaction();
  tx.add(ComputeBudgetProgram.setComputeUnitLimit({ units: 300_000 }));
  tx.add(ComputeBudgetProgram.setComputeUnitPrice({ microLamports: 50_000 }));
  tx.add(ed25519Ix);  // MUST be before claim instruction
  tx.add(claimIx);

  const signed = await wallet.signTransaction(tx);
  return await connection.sendRawTransaction(signed.serialize());
}
```

**Ed25519 instruction MUST appear before the instruction that verifies it.** The program searches the instructions sysvar by index. If your wallet prepends ComputeBudget instructions, search up to index 10 (not just index 0).

## Keypair Management

### Secure Key File Pattern

```typescript
import * as fs from 'fs';
import * as os from 'os';
import * as path from 'path';

const KEYS_PATH = path.join(os.homedir(), '.config', 'myapp', 'keys.json');

interface SecureKeys {
  SOLANA_GAME_SIGNER_SECRET?: string;
  SOLANA_AUTHORITY_SECRET?: string;
}

export function loadSecureKeys(): SecureKeys {
  const stat = fs.statSync(KEYS_PATH);
  // Production: enforce 0o600 permissions (owner-only read/write)
  if ((stat.mode & 0o777) !== 0o600) {
    console.warn('WARNING: Key file permissions too permissive');
  }
  return JSON.parse(fs.readFileSync(KEYS_PATH, 'utf-8'));
}
```

**Key Format Support:**
- JSON array: `[u8; 64]` -- `Uint8Array.from(JSON.parse(raw))`
- Base58: Solana standard -- `bs58.decode(raw)`

**Never:**
- Commit keys to git (add to `.gitignore`)
- Store in localStorage or client-side code
- Log key material (even in debug mode)
- Use the deploy wallet as the signing key (separate concerns)

### Environment Variables

```bash
# .env (server-side only, never in NEXT_PUBLIC_*)
SOLANA_PROGRAM_ID="YourProgramId..."
SOLANA_GAME_SIGNER_SECRET='[1,2,3,...,64]'  # Ed25519 keypair
SOLANA_AUTHORITY_SECRET='[1,2,3,...,64]'     # Close game authority
SOLANA_RPC_URL="https://mainnet.helius-rpc.com/?api-key=YOUR_KEY"
SOLANA_FEE_VAULT_PUBKEY="VaultAddress..."
```

## RPC Configuration

### Helius (Recommended)

| Network | URL Pattern |
|---|---|
| Mainnet | `https://mainnet.helius-rpc.com/?api-key=YOUR_KEY` |
| Devnet | `https://devnet.helius-rpc.com/?api-key=YOUR_KEY` |

### Client vs Server RPC

```typescript
// Server-side: use private env var
const RPC_URL = process.env.SOLANA_RPC_URL ?? 'https://api.devnet.solana.com';

// Client-side: use public env var (API key visible but rate-limited)
const RPC_URL = process.env.NEXT_PUBLIC_SOLANA_RPC_URL ?? 'https://api.mainnet-beta.solana.com';
```

**The default public RPCs (`api.mainnet-beta.solana.com`) rate-limit aggressively.** Always use a paid RPC for production. Helius free tier gives 100K requests/day which is enough for early-stage apps.

### Commitment Levels

| Level | Speed | Safety | Use Case |
|---|---|---|---|
| `processed` | Fastest | May be rolled back | UI display only |
| `confirmed` | ~5s | 1+ block confirmation | **Default for everything** |
| `finalized` | ~15s | 32+ blocks | High-value settlements |

Always use `'confirmed'` unless you have a specific reason not to.

## Devnet to Mainnet Migration

### Checklist

- [ ] Program ID is the same (deploy to mainnet with same keypair)
- [ ] `Anchor.toml` cluster updated to `mainnet`
- [ ] RPC URLs updated (devnet -> mainnet Helius)
- [ ] GlobalConfig re-initialized on mainnet (it's a different PDA on a different cluster)
- [ ] Fee vault address set to mainnet wallet
- [ ] Game signer keypair is the same (or update GlobalConfig if rotated)
- [ ] Client env vars updated (`NEXT_PUBLIC_SOLANA_RPC_URL`)
- [ ] Server env vars updated (`SOLANA_RPC_URL`)
- [ ] Airdrop code removed (only works on devnet)
- [ ] Minimum deposit amounts adjusted for mainnet SOL prices
- [ ] Priority fees configured for mainnet congestion

### Common Migration Failures

| Problem | Cause | Fix |
|---|---|---|
| PDA not found | Config not initialized on mainnet | Run initialize script against mainnet |
| Wrong game signer | Config initialized with wrong pubkey | Redeploy config (or add update_config instruction) |
| Transactions fail silently | Wrong RPC URL (still pointing to devnet) | Verify RPC URL in both server and client env |
| Deposit verification fails | Server reads devnet, client sends to mainnet | Ensure all RPC URLs point to same cluster |

## Account Closure (Rent Recovery)

After a game settles or cancels and all refunds are claimed, close the game account to recover rent:

```typescript
async function closeGame(roomCode: string): Promise<string> {
  const authority = loadAuthorityKeypair();
  const gamePda = getGamePda(roomCodeToGameId(roomCode));
  const configPda = getConfigPda();

  const tx = new Transaction().add(new TransactionInstruction({
    programId: PROGRAM_ID,
    keys: [
      { pubkey: gamePda, isSigner: false, isWritable: true },
      { pubkey: configPda, isSigner: false, isWritable: false },
      { pubkey: authority.publicKey, isSigner: false, isWritable: true },
      { pubkey: authority.publicKey, isSigner: true, isWritable: false },
    ],
    data: CLOSE_GAME_DISCRIMINATOR,
  }));

  return await sendAndConfirmTransaction(connection, tx, [authority]);
}
```

**On-chain requirements:**
- Game state must be `Settled` or `Cancelled`
- If `Cancelled`: all deposited players must have claimed refunds
- Caller must be `config.authority`
- Recovered rent (~0.003 SOL) goes to authority wallet

## Testing Strategy

### Local Validator

```bash
# Start local validator
solana-test-validator

# Deploy locally
anchor deploy --provider.cluster localnet
```

### Anchor Tests

```typescript
describe('my_program', () => {
  it('initializes config', async () => {
    await program.methods
      .initialize(gameSigner, feeVault, 500)
      .accounts({ config: configPda, authority: wallet.publicKey })
      .rpc();

    const config = await program.account.globalConfig.fetch(configPda);
    assert.equal(config.feeBps, 500);
  });

  it('creates and joins game', async () => {
    const gameId = Array.from(roomCodeToGameId('TEST01'));
    await program.methods
      .createGame(gameId, 2, new BN(100_000_000)) // 0.1 SOL
      .accounts({ game: gamePda, host: player1.publicKey })
      .signers([player1])
      .rpc();

    const game = await program.account.gameAccount.fetch(gamePda);
    assert.equal(game.players.length, 1);
    assert.equal(game.deposited[0], true);
  });
});
```

### Devnet Testing

Always test the full flow on devnet before mainnet:
1. Deploy program
2. Initialize config
3. Create game (deposit)
4. Join game (deposit)
5. Settle game (Ed25519 signature)
6. Claim winnings
7. Close game (rent recovery)
8. Cancel game (separate test)
9. Claim refund
10. Emergency cancel (wait 24h or reduce for testing)

## References

- [Anchor Book](https://www.anchor-lang.com/)
- [Solana Cookbook](https://solanacookbook.com/)
- [Helius RPC](https://www.helius.dev/)
- [Ed25519 Precompile](https://docs.solana.com/developing/runtime-facilities/programs#ed25519-program)
