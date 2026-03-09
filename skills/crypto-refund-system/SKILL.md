---
name: crypto-refund-system
description: Use when implementing refund flows for crypto deposits, building cancellation systems for on-chain games, or handling failed settlements. Covers refund triggers, claim flows, rent accounting, and atomic persistence for both EVM and Solana.
version: 0.1.0
tools: Read, Grep, Glob, Bash
---

# Crypto Refund System

Complete refund architecture for applications that take crypto deposits. Covers when refunds trigger, how they're signed and persisted, how users claim them on-chain, and the rent/fee edge cases that will burn you. Built from a production game where real user funds were at stake.

## When to Use

- Building refund flows for deposited crypto (ETH, SOL, tokens)
- Implementing game/session cancellation with fund return
- Handling failed settlements or stuck games
- Adding emergency withdrawal mechanisms
- Any system where users deposit funds and may need them back

## Architecture Principle

```
Refund = Cancellation Signature + On-Chain Claim

Server signs cancellation -> Persists signature -> Client claims on-chain
                                                -> Retry if needed
                                                -> Emergency cancel if server unreachable
```

**The refund is a pull system, not a push system.** The server never sends funds directly. It signs a cancellation message, and each player claims their own refund. This prevents the server from becoming a single point of failure for fund recovery.

## Refund Trigger Matrix

| Trigger | Phase | Cause | Action |
|---|---|---|---|
| Player leaves lobby | Lobby | User clicks leave | Sign cancellation for that player |
| Room deleted | Lobby | Last player leaves | Sign cancellation for all deposited |
| Disconnect timeout | Playing | Player offline >2 min | Bankrupt player, cancel if cascading |
| All players disconnect | Playing | Everyone goes offline | Auto-cancel for all |
| Bankruptcy error | Playing | Game state corruption | Force cancel as fallback |
| Stale room sweep | Playing | Inactive >2 hours | Force cancel, clean up |
| Emergency cancel | Any | Server down >24 hours | Permissionless on-chain cancel (no signature needed) |

## Server-Side: Signing Cancellations

### EVM (ECDSA + keccak256)

```typescript
import { keccak256, encodePacked, toBytes } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';

const signer = privateKeyToAccount(process.env.GAME_SIGNER_PRIVATE_KEY as `0x${string}`);

async function signCancellation(roomCode: string) {
  const gameId = keccak256(toBytes(roomCode));
  const nonce = `0x${randomBytes(32).toString('hex')}` as `0x${string}`;

  // Message: "CANCEL" || gameId || nonce || contractAddress || chainId
  const message = keccak256(
    encodePacked(
      ['string', 'bytes32', 'bytes32', 'address', 'uint256'],
      ['CANCEL', gameId, nonce, CONTRACT_ADDRESS, BigInt(CHAIN_ID)]
    )
  );

  const signature = await signer.signMessage({ message: { raw: toBytes(message) } });

  return { gameId, nonce, signature };
}
```

### Solana (Ed25519)

```typescript
import nacl from 'tweetnacl';
import bs58 from 'bs58';
import { randomBytes, createHash } from 'crypto';

function signSolanaCancellation(roomCode: string) {
  const keypair = loadSignerKeypair();
  const gameId = createHash('sha256').update(roomCode).digest();
  const nonce = randomBytes(32);

  // Message: "CANCEL" || gameId(32) || nonce(32) || programId(32)
  const message = Buffer.concat([
    Buffer.from('CANCEL'),
    gameId,
    nonce,
    PROGRAM_ID.toBuffer(),
  ]);

  const signature = nacl.sign.detached(message, keypair.secretKey);

  return {
    gameId: '0x' + gameId.toString('hex'),
    nonce: bs58.encode(nonce),
    signature: bs58.encode(signature),
  };
}
```

## Atomic Persistence

### Why Persistence Matters

The cancellation signature is the user's proof that they can claim a refund. If the server crashes between signing and sending to the client, the signature is lost. Persist before emitting.

### File-Based Atomic Writes

```typescript
// server/refundStore.ts
import * as fs from 'fs';

const REFUND_PATH = 'pending-refunds.json';
const REFUND_TMP_PATH = 'pending-refunds.tmp.json';

interface PendingRefund {
  walletAddress: string;
  roomCode: string;
  gameId: string;        // 0x-prefixed hex
  nonce: string;         // hex (EVM) or base58 (Solana)
  signature: string;     // hex (EVM) or base58 (Solana)
  timestamp: number;
  reason: string;        // 'player_left_lobby' | 'disconnect-timeout' | etc.
  type?: 'settlement';   // Settlement vs cancellation
  chain?: string;        // 'base' | 'solana'
}

// Serialize writes to prevent corruption
let writeLock = Promise.resolve();

export function appendPendingRefunds(refunds: PendingRefund[]): Promise<void> {
  const thisWrite = writeLock.then(async () => {
    const existing = readPendingRefunds();
    existing.push(...refunds);
    // Atomic: write to temp file, then rename
    fs.writeFileSync(REFUND_TMP_PATH, JSON.stringify(existing, null, 2));
    fs.renameSync(REFUND_TMP_PATH, REFUND_PATH);
  });

  // Chain next write, swallow errors to keep lock alive
  writeLock = thisWrite.catch(err => console.error('[refundStore] Write failed:', err));
  return thisWrite;
}

export function readPendingRefunds(): PendingRefund[] {
  try {
    return JSON.parse(fs.readFileSync(REFUND_PATH, 'utf-8'));
  } catch {
    return []; // File missing or corrupted: safe default
  }
}
```

### Room-Level Mutual Exclusion

**Critical:** Settlement and cancellation must never both be issued for the same room. Use a room-level lock:

```typescript
const roomLocks = new Map<string, Promise<void>>();

export async function withRoomLock<T>(
  roomCode: string,
  fn: () => Promise<T>,
): Promise<T> {
  const prev = roomLocks.get(roomCode) ?? Promise.resolve();
  let resolve: () => void;
  const next = new Promise<void>(r => { resolve = r; });
  roomLocks.set(roomCode, next);

  await prev; // Wait for previous operation to complete

  try {
    return await fn();
  } finally {
    resolve!(); // Release lock for next waiter
  }
}

// Usage
async function forceGameCancellation(code: string, reason: string) {
  await withRoomLock(code, async () => {
    if (hasSettlement(code)) return; // Settlement already exists, skip
    if (hasCancellation(code)) return; // Already cancelled, skip

    const cancellation = signCancellation(code);
    await appendPendingRefunds(/* ... */);

    io.to(code).emit('game:cancellation:signature', cancellation);
  });
}
```

**Without this lock, a race condition can issue both a settlement AND a cancellation, allowing the winner to claim winnings AND all players to claim refunds -- draining the contract.**

## Client-Side: Claiming Refunds

### EVM Claim Flow

```typescript
async function claimRefund(
  walletClient: WalletClient,
  roomCode: string,
  refund: PendingRefund,
) {
  // 1. Check current on-chain state
  const gameState = await getOnChainGameState(roomCode);

  if (gameState !== 'CANCELLED') {
    // Game not cancelled yet -- submit the cancellation first
    await walletClient.writeContract({
      address: CONTRACT_ADDRESS,
      abi: GAME_ABI,
      functionName: 'cancelGame',
      args: [refund.gameId, refund.nonce, refund.signature],
    });

    // Wait for confirmation
    await new Promise(r => setTimeout(r, 2000));

    // Recheck state
    const recheck = await getOnChainGameState(roomCode);
    if (recheck !== 'CANCELLED') {
      throw new Error('Cancellation did not take effect');
    }
  }

  // 2. Simulate claim to check eligibility
  const canClaim = await simulateClaimRefund(roomCode, walletClient.account.address);
  if (!canClaim) {
    // Already claimed or not eligible -- treat as success
    return;
  }

  // 3. Execute claim
  const hash = await walletClient.writeContract({
    address: CONTRACT_ADDRESS,
    abi: GAME_ABI,
    functionName: 'claimRefund',
    args: [refund.gameId],
  });

  // 4. Wait for receipt
  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  if (receipt.status === 'reverted') {
    throw new Error('Refund claim reverted');
  }
}
```

### Solana Claim Flow

```typescript
async function claimSolanaRefund(
  wallet: AnchorWallet,
  roomCode: string,
  refund: PendingRefund,
) {
  const gameId = roomCodeToGameId(roomCode);
  const gamePda = getGamePda(gameId);
  const game = await readGameAccount(gamePda);

  if (game.state === 'Active') {
    // Check if emergency cancel is available (24h timeout)
    const elapsed = Date.now() / 1000 - Number(game.startedAt);
    if (elapsed < 86400) {
      throw new Error('Emergency timeout not reached. Please wait.');
    }
    // Emergency cancel (no signature needed)
    await emergencyCancelOnSolana(wallet, roomCode);
  } else if (game.state === 'Waiting') {
    // Cancel with server signature
    const signerPubkey = await getGameSignerPubkey();
    await cancelGameOnSolana(
      wallet, roomCode,
      refund.nonce, refund.signature, signerPubkey,
    );
  }
  // If already Cancelled: skip cancel, go to claim

  // Claim refund
  await claimRefundOnSolana(wallet, roomCode);

  // Best-effort: reclaim account rent
  fetch('/api/contracts/solana/close-game', {
    method: 'POST',
    body: JSON.stringify({ roomCode }),
  }).catch(() => {}); // Fire and forget
}
```

### Batch Claiming

```typescript
async function batchClaimRefunds(refunds: PendingRefund[]) {
  let claimed = 0;
  let skipped = 0;

  // Only batch CANCELLED games (skip WAITING/ACTIVE to avoid signature overhead)
  const cancelledRefunds = refunds.filter(r => r.onChainState === 'Cancelled');

  for (const refund of cancelledRefunds) {
    try {
      if (refund.chain === 'solana') {
        await claimSolanaRefund(wallet, refund.roomCode, refund);
      } else {
        await claimRefund(walletClient, refund.roomCode, refund);
      }
      claimed++;
    } catch (err: any) {
      if (err.message?.includes('rejected') || err.message?.includes('denied')) {
        break; // User rejected wallet prompt -- stop batch
      }
      skipped++; // Other error -- continue batch
    }
  }

  return { claimed, skipped };
}
```

## On-Chain Contracts

### EVM Contract

```solidity
contract Game {
    enum GameState { WAITING, ACTIVE, SETTLED, CANCELLED }

    struct GameData {
        GameState state;
        uint256 buyIn;
        uint256 pot;
        uint256 startedAt;
    }

    mapping(bytes32 => GameData) public games;
    mapping(bytes32 => mapping(address => bool)) public deposited;
    mapping(bytes32 => mapping(address => bool)) public refunded;

    uint256 public constant EMERGENCY_TIMEOUT = 24 hours;

    function cancelGame(
        bytes32 gameId,
        bytes32 nonce,
        bytes calldata signature
    ) external {
        GameData storage g = games[gameId];
        require(g.state == GameState.WAITING || g.state == GameState.ACTIVE);

        // Verify server signature
        bytes32 message = keccak256(abi.encodePacked(
            "CANCEL", gameId, nonce, address(this), block.chainid
        ));
        require(verifySignature(message, signature), "Invalid signature");

        g.state = GameState.CANCELLED;
    }

    function emergencyCancel(bytes32 gameId) external {
        GameData storage g = games[gameId];
        require(g.state == GameState.WAITING || g.state == GameState.ACTIVE);
        require(deposited[gameId][msg.sender], "Not a player");
        require(
            block.timestamp >= g.startedAt + EMERGENCY_TIMEOUT,
            "Too early"
        );
        g.state = GameState.CANCELLED;
    }

    function claimRefund(bytes32 gameId) external nonReentrant {
        GameData storage g = games[gameId];
        require(g.state == GameState.CANCELLED, "Not cancelled");
        require(deposited[gameId][msg.sender], "Not deposited");
        require(!refunded[gameId][msg.sender], "Already refunded");

        refunded[gameId][msg.sender] = true;
        payable(msg.sender).transfer(g.buyIn);

        emit RefundClaimed(gameId, msg.sender, g.buyIn);
    }
}
```

### Solana Program

```rust
pub fn claim_refund(ctx: Context<ClaimRefund>) -> Result<()> {
    let game = &mut ctx.accounts.game;
    let player = &ctx.accounts.player;

    require!(game.state == GameState::Cancelled, MyError::GameNotCancelled);

    let player_key = player.key();
    let idx = game.players.iter()
        .position(|p| *p == player_key)
        .ok_or(MyError::NotAPlayer)?;

    require!(game.deposited[idx], MyError::NotDeposited);
    require!(!game.refunded[idx], MyError::AlreadyRefunded);

    // Rent protection: ensure PDA retains minimum balance
    let game_lamports = game.to_account_info().lamports();
    let rent = Rent::get()?.minimum_balance(GameAccount::SIZE);
    require!(
        game_lamports >= game.buy_in.checked_add(rent).ok_or(MyError::Overflow)?,
        MyError::InsufficientBalance
    );

    game.refunded[idx] = true;

    // Direct lamport transfer (no CPI needed for native SOL)
    **game.to_account_info().try_borrow_mut_lamports()? -= game.buy_in;
    **player.try_borrow_mut_lamports()? += game.buy_in;

    Ok(())
}
```

## Rent Accounting (Solana-Specific)

### The Rent Problem

Solana accounts must maintain a minimum balance (rent exemption). When the last player claims their refund, the PDA must still have enough lamports for rent. If not, the transaction reverts and the last player can never claim.

### Solution: Rent-Safe Subtraction

```rust
// In settlement: deduct rent before calculating payout
let available = game_lamports.saturating_sub(rent);
let fee = (available as u128 * fee_bps as u128 / 10000) as u64;
let payout = available - fee;

// In refund: verify balance covers refund + rent
require!(game_lamports >= buy_in + rent, InsufficientBalance);
```

### Rent Recovery

After all refunds are claimed, close the game account to recover rent:

```rust
pub fn close_game(ctx: Context<CloseGame>) -> Result<()> {
    let game = &ctx.accounts.game;

    // Must be settled or cancelled
    require!(
        game.state == GameState::Settled || game.state == GameState::Cancelled,
        MyError::GameNotFinalized
    );

    // If cancelled: all deposited players must have refunded
    if game.state == GameState::Cancelled {
        for i in 0..game.players.len() {
            if game.deposited[i] {
                require!(game.refunded[i], MyError::RefundsNotComplete);
            }
        }
    }

    // Anchor's `close` constraint transfers remaining lamports to recipient
    Ok(())
}
```

**The authority wallet receives ~0.003 SOL per closed game.** This self-funds future close_game transactions. Run close_game as a fire-and-forget cleanup after refund claims.

## Refund Discovery (Client-Side)

Users need to find their unclaimed refunds, even if they missed the real-time socket event.

### Multi-Source Scan

```typescript
async function discoverRefunds(walletAddress: string, chain: string): Promise<RefundItem[]> {
  const refunds: RefundItem[] = [];

  // Source 1: Persistent refund store (server knows about these)
  const serverRefunds = await fetch(`/api/refunds/${walletAddress}`).then(r => r.json());
  for (const r of serverRefunds) {
    if (r.chain === chain) refunds.push(r);
  }

  // Source 2: Game history (find games where player deposited but didn't win)
  const history = await fetch(`/api/stats/me?chain=${chain}`).then(r => r.json());
  for (const game of history.recentGames) {
    if (!refunds.some(r => r.roomCode === game.roomCode)) {
      // Check on-chain state
      const onChainState = await getOnChainGameState(game.roomCode, chain);
      if (onChainState === 'Cancelled') {
        const alreadyClaimed = await checkIfRefunded(game.roomCode, walletAddress, chain);
        if (!alreadyClaimed) {
          refunds.push({ ...game, source: 'history' });
        }
      }
    }
  }

  // Source 3 (EVM only): On-chain transfer scan via Alchemy
  if (chain === 'base') {
    const transfers = await alchemy.getAssetTransfers({
      fromAddress: walletAddress,
      toAddress: CONTRACT_ADDRESS,
      category: ['external'],
    });
    // Cross-reference with known games, check if refundable
  }

  return refunds;
}
```

### On-Chain State Checks

```typescript
// EVM: read game state from contract
async function getOnChainGameState(roomCode: string): Promise<string> {
  const gameId = keccak256(toBytes(roomCode));
  const result = await publicClient.readContract({
    address: CONTRACT_ADDRESS,
    abi: GAME_ABI,
    functionName: 'games',
    args: [gameId],
  });
  return ['WAITING', 'ACTIVE', 'SETTLED', 'CANCELLED'][Number(result.state)];
}

// Solana: read game PDA
async function getSolanaGameState(roomCode: string): Promise<string> {
  const gameId = roomCodeToGameId(roomCode);
  const gamePda = getGamePda(gameId);
  const game = await readGameAccount(gamePda);
  return game?.state ?? 'NOT_FOUND';
}
```

## Emergency Cancel (No Server Needed)

The emergency cancel is the trustless escape hatch. If the server goes down permanently, users can still recover their funds after 24 hours.

### EVM

```solidity
function emergencyCancel(bytes32 gameId) external {
    require(deposited[gameId][msg.sender], "Not a player");
    require(block.timestamp >= games[gameId].startedAt + 24 hours, "Too early");
    require(games[gameId].state != GameState.SETTLED, "Already settled");
    require(games[gameId].state != GameState.CANCELLED, "Already cancelled");
    games[gameId].state = GameState.CANCELLED;
}
```

### Solana

```rust
pub fn emergency_cancel(ctx: Context<EmergencyCancel>) -> Result<()> {
    let game = &mut ctx.accounts.game;
    let player = &ctx.accounts.player;

    require!(
        game.state == GameState::Waiting || game.state == GameState::Active,
        MyError::GameAlreadySettled
    );

    let player_key = player.key();
    require!(
        game.players.iter().any(|p| *p == player_key),
        MyError::NotAPlayer
    );

    let now = Clock::get()?.unix_timestamp;
    require!(
        now - game.started_at >= 86400, // 24 hours
        MyError::EmergencyCancelTooEarly
    );

    game.state = GameState::Cancelled;
    Ok(())
}
```

**No signature needed.** Any player in the game can call this after 24 hours. This is the last resort when the server is unreachable.

## Error Handling

### Idempotent Claims

`AlreadyRefunded` errors should be treated as success:

```typescript
try {
  await claimRefund(gameId);
} catch (err: any) {
  if (err.message.includes('AlreadyRefunded') || err.message.includes('6010')) {
    // Already claimed -- this is fine
    return { ok: true, alreadyClaimed: true };
  }
  throw err; // Real error
}
```

### User Rejection vs Real Errors

```typescript
try {
  const hash = await walletClient.writeContract(/* ... */);
} catch (err: any) {
  if (err.message?.includes('rejected') || err.message?.includes('denied')) {
    setStatus('Transaction cancelled');
    return; // Don't show error UI
  }
  setError(`Refund failed: ${err.message}`);
}
```

### Transient Failures (Solana RPC)

```typescript
async function withRetry<T>(fn: () => Promise<T>, retries = 3): Promise<T> {
  for (let i = 0; i < retries; i++) {
    try {
      return await fn();
    } catch (err: any) {
      if (i === retries - 1) throw err;
      if (err.message?.includes('blockhash') || err.message?.includes('timeout')) {
        await new Promise(r => setTimeout(r, 2000 * (i + 1)));
        continue;
      }
      throw err; // Non-transient error
    }
  }
  throw new Error('Unreachable');
}
```

## Testing Checklist

- [ ] Happy path: deposit -> cancel -> claim refund -> balance restored
- [ ] Double claim: claim twice -> second attempt handled gracefully
- [ ] Emergency cancel: wait 24h -> emergency cancel -> claim refund
- [ ] Settlement/cancellation mutual exclusion: can't issue both
- [ ] Atomic persistence: crash between sign and emit -> refund recoverable
- [ ] Rent protection (Solana): last player can still claim
- [ ] Account closure: all refunded -> close game -> rent recovered
- [ ] Batch claiming: multiple refunds -> sequential claim -> user rejection stops batch
- [ ] Stale room cleanup: inactive 2h -> auto-cancel -> refunds available
- [ ] All-disconnected: everyone leaves mid-game -> auto-cancel
- [ ] Wrong chain: Solana user can't claim Base refund and vice versa

## Common Mistakes

1. **Push-based refunds.** Never send funds from the server. Users claim their own refunds. The server only signs.
2. **No persistence.** If the server crashes after signing but before the client receives the signature, the user loses access to their refund forever. Persist before emitting.
3. **No mutual exclusion.** Without a room-level lock, settlement and cancellation can both be issued, allowing double-withdrawal.
4. **Ignoring Solana rent.** The last refund claim will revert if it would leave the PDA below rent-exempt minimum. Always check `lamports >= buyIn + rent`.
5. **No emergency cancel.** If the server goes down permanently, user funds are locked forever. The 24-hour timeout is the escape hatch.
6. **Case-sensitive EVM addresses.** Always lowercase before comparison.
7. **Not treating AlreadyRefunded as success.** Users will retry. The contract rejecting a double-claim is correct behavior, not an error.
