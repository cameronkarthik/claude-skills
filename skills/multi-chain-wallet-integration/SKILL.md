---
name: multi-chain-wallet-integration
description: Use when connecting wallets across EVM (MetaMask, Base) and Solana (Phantom, Solflare), implementing chain-scoped state, or building multi-chain authentication. Covers wallet adapters, signature verification, and chain-switching patterns.
version: 0.1.0
tools: Read, Grep, Glob, Bash
---

# Multi-Chain Wallet Integration

Patterns for connecting wallets across EVM (Base/Ethereum) and Solana chains in a single application. Covers wallet adapters, signature-based authentication, chain-scoped state management, and the bugs you will definitely encounter.

## When to Use

- Connecting wallets for both EVM and Solana in one app
- Building signature-based authentication (nonce + sign + verify)
- Implementing chain-scoped profiles, stats, or leaderboards
- Handling chain switching and wallet disconnection
- Verifying on-chain deposits from either chain
- Any app that supports multiple blockchains with user wallets

## Architecture Overview

```
MultiChainContext (unified interface)
  |
  +-- EVMWalletContext (wagmi + RainbowKit)
  |     |-- MetaMask, Phantom (EVM), Coinbase, WalletConnect
  |     |-- Chain: Base (8453) / Base Sepolia (84532)
  |     |-- Auth: EIP-191 personal_sign
  |
  +-- SolanaWalletContext (@solana/wallet-adapter)
  |     |-- Phantom, Solflare (auto-detected from browser)
  |     |-- Auth: Ed25519 signMessage
  |
  +-- AuthContext (unified session management)
        |-- Nonce-based challenge/response
        |-- HTTP-only session cookies (30 day expiry)
        |-- Chain stored per user in database
```

## Wallet Adapter Setup

### EVM (wagmi + RainbowKit)

```typescript
// context/EVMWalletContext.tsx
import { createConfig, http } from 'wagmi';
import { base, baseSepolia } from 'wagmi/chains';
import { connectorsForWallets, RainbowKitProvider } from '@rainbow-me/rainbowkit';
import {
  phantomWallet,
  metaMaskWallet,
  coinbaseWallet,
  walletConnectWallet,
} from '@rainbow-me/rainbowkit/wallets';

const RPC_URL = '/api/rpc'; // Proxy to avoid exposing API key

const config = createConfig({
  connectors: connectorsForWallets([
    {
      groupName: 'Recommended',
      wallets: [phantomWallet, metaMaskWallet, coinbaseWallet, walletConnectWallet],
    },
  ]),
  chains: [base, baseSepolia],
  transports: {
    [base.id]: http(RPC_URL),
    [baseSepolia.id]: http(RPC_URL),
  },
  ssr: true, // Required for Next.js
});

export function EVMWalletProvider({ children }: { children: React.ReactNode }) {
  return (
    <WagmiProvider config={config}>
      <QueryClientProvider client={queryClient}>
        <RainbowKitProvider>
          {children}
        </RainbowKitProvider>
      </QueryClientProvider>
    </WagmiProvider>
  );
}
```

**RPC Proxy:** Never expose your RPC API key in `NEXT_PUBLIC_*` env vars. Create an API route that proxies requests:

```typescript
// app/api/rpc/route.ts
export async function POST(req: Request) {
  const body = await req.json();
  const res = await fetch(process.env.ALCHEMY_RPC_URL!, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });
  return new Response(res.body, { headers: { 'Content-Type': 'application/json' } });
}
```

### Solana (@solana/wallet-adapter)

```typescript
// context/SolanaWalletContext.tsx
import { ConnectionProvider, WalletProvider } from '@solana/wallet-adapter-react';
import { WalletModalProvider } from '@solana/wallet-adapter-react-ui';

const RPC_URL = process.env.NEXT_PUBLIC_SOLANA_RPC_URL ?? 'https://api.mainnet-beta.solana.com';

export function SolanaWalletProvider({ children }: { children: React.ReactNode }) {
  return (
    <ConnectionProvider endpoint={RPC_URL}>
      <WalletProvider wallets={[]} autoConnect>
        <WalletModalProvider>
          {children}
        </WalletModalProvider>
      </WalletProvider>
    </ConnectionProvider>
  );
}
```

**Empty wallets array:** Pass `[]` for wallets. The adapter auto-detects installed browser extensions (Phantom, Solflare, etc.). Only add explicit adapters if you need custom behavior.

### Multi-Chain Context

```typescript
// context/MultiChainContext.tsx
interface MultiChainState {
  activeChain: 'solana' | 'base';
  connectedAddress: string | null;
  isAuthenticated: boolean;
  connectAndSign: () => Promise<void>;
  disconnect: () => Promise<void>;
}

export function MultiChainProvider({ children }: Props) {
  const [activeChain, setActiveChain] = useState<'solana' | 'base'>('solana');

  // Delegate to correct chain context
  const evmAuth = useEVMAuth();
  const solanaAuth = useSolanaAuth();

  const auth = activeChain === 'solana' ? solanaAuth : evmAuth;

  // Auto-detect chain from existing session
  useEffect(() => {
    async function detectChain() {
      const session = await fetch('/api/auth/me').then(r => r.json());
      if (session?.chain) {
        setActiveChain(session.chain);
      }
    }
    detectChain();
  }, []);

  return (
    <MultiChainContext.Provider value={{
      activeChain,
      connectedAddress: auth.address,
      isAuthenticated: auth.isAuthenticated,
      connectAndSign: auth.connectAndSign,
      disconnect: auth.disconnect,
    }}>
      {children}
    </MultiChainContext.Provider>
  );
}
```

## Authentication Flow

### Nonce-Based Challenge/Response

```
1. Client -> GET /api/auth/nonce
2. Server -> { nonce, message }  (nonce is one-time, expires in 5 min)
3. Client -> wallet.signMessage(message)
4. Client -> POST /api/auth/verify-wallet { signature, nonce, chain, walletAddress }
5. Server -> verifies signature, creates session cookie
6. Client -> authenticated (session cookie set)
```

### Server: Nonce Generation

```typescript
// server/auth.ts
const nonces = new Map<string, { message: string; walletAddress?: string; expiresAt: number }>();

export function generateNonce(): { nonce: string; message: string } {
  const nonce = randomBytes(32).toString('hex');
  const message = `Sign this message to connect to My App.\n\nNonce: ${nonce}`;

  nonces.set(nonce, {
    message,
    expiresAt: Date.now() + 5 * 60 * 1000, // 5 minute expiry
  });

  return { nonce, message };
}

export function consumeNonce(nonce: string): { message: string } | null {
  const entry = nonces.get(nonce);
  if (!entry || entry.expiresAt < Date.now()) {
    nonces.delete(nonce);
    return null;
  }
  nonces.delete(nonce); // One-time use
  return entry;
}
```

### Server: Signature Verification

```typescript
// server/routes/auth.ts
import { verifyMessage } from 'viem';
import nacl from 'tweetnacl';
import bs58 from 'bs58';

app.post('/api/auth/verify-wallet', async (req, res) => {
  const { signature, nonce, chain, walletAddress } = req.body;

  // Consume nonce (one-time)
  const nonceData = consumeNonce(nonce);
  if (!nonceData) return res.status(400).json({ error: 'Invalid or expired nonce' });

  let valid = false;

  if (chain === 'base') {
    // EVM: EIP-191 personal_sign verification
    valid = await verifyMessage({
      address: walletAddress as `0x${string}`,
      message: nonceData.message,
      signature: signature as `0x${string}`,
    });
  } else if (chain === 'solana') {
    // Solana: Ed25519 verification
    const msgBytes = new TextEncoder().encode(nonceData.message);
    const publicKey = bs58.decode(walletAddress);
    const sig = bs58.decode(signature);
    valid = nacl.sign.detached.verify(msgBytes, sig, publicKey);
  }

  if (!valid) return res.status(401).json({ error: 'Invalid signature' });

  // Create session
  const token = randomBytes(32).toString('hex');
  await createSession(token, walletAddress, chain);

  // Set HTTP-only cookie
  res.cookie('session', token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 30 * 24 * 60 * 60 * 1000, // 30 days
  });

  res.json({ ok: true });
});
```

### Client: EVM Signing

```typescript
import { useSignMessage } from 'wagmi';

function useEVMAuth() {
  const { signMessageAsync } = useSignMessage();
  const { address } = useAccount();

  async function connectAndSign() {
    // 1. Get nonce
    const { nonce, message } = await fetch('/api/auth/nonce').then(r => r.json());

    // 2. Sign with wallet (EIP-191)
    const signature = await signMessageAsync({ message });

    // 3. Verify on server
    await fetch('/api/auth/verify-wallet', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ signature, nonce, chain: 'base', walletAddress: address }),
    });
  }

  return { connectAndSign, address, isAuthenticated };
}
```

### Client: Solana Signing

```typescript
import { useWallet } from '@solana/wallet-adapter-react';

function useSolanaAuth() {
  const { publicKey, signMessage, connected } = useWallet();

  async function connectAndSign() {
    if (!publicKey || !signMessage) throw new Error('Wallet not connected');

    // 1. Get nonce
    const { nonce, message } = await fetch('/api/auth/nonce').then(r => r.json());

    // 2. Sign with wallet (Ed25519)
    const messageBytes = new TextEncoder().encode(message);
    const signatureBytes = await signMessage(messageBytes);
    const signature = bs58.encode(signatureBytes);

    // 3. Verify on server
    await fetch('/api/auth/verify-wallet', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        signature,
        nonce,
        chain: 'solana',
        walletAddress: publicKey.toBase58(),
      }),
    });
  }

  return { connectAndSign, address: publicKey?.toBase58(), isAuthenticated };
}
```

## Chain-Scoped State

### Database Schema

```sql
CREATE TABLE users (
  wallet_address TEXT PRIMARY KEY,
  chain TEXT NOT NULL DEFAULT 'solana',  -- 'solana' or 'base'
  display_name TEXT,
  character_id TEXT,
  created_at INTEGER DEFAULT (unixepoch()),
  last_seen INTEGER DEFAULT (unixepoch())
);

-- Profile is global (same name/avatar across chains)
-- Stats are per-chain

CREATE TABLE game_history (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  chain TEXT NOT NULL,                    -- 'solana' or 'base'
  room_code TEXT NOT NULL,
  wallet_address TEXT NOT NULL,
  winner_wallet TEXT,
  entry_fee TEXT,                         -- lamports or wei
  winner_payout TEXT,
  players TEXT,                           -- JSON array
  created_at INTEGER DEFAULT (unixepoch())
);
```

### Chain-Filtered Queries

```typescript
// Stats per chain
function getPlayerStats(walletAddress: string, chain?: string) {
  if (chain) {
    return db.prepare(`
      SELECT
        COUNT(*) as gamesPlayed,
        SUM(CASE WHEN winner_wallet = ? THEN 1 ELSE 0 END) as gamesWon,
        SUM(CASE WHEN winner_wallet = ? THEN winner_payout ELSE 0 END) as totalEarned
      FROM game_history
      WHERE wallet_address = ? AND chain = ?
    `).get(walletAddress, walletAddress, walletAddress, chain);
  }
  // No chain: aggregate across all chains
  return db.prepare(`SELECT ... FROM game_history WHERE wallet_address = ?`).get(walletAddress);
}

// Leaderboard per chain
function getLeaderboard(limit: number, chain?: string) {
  const whereClause = chain ? 'WHERE chain = ?' : '';
  return db.prepare(`
    SELECT winner_wallet, COUNT(*) as wins, SUM(winner_payout) as earnings
    FROM game_history
    ${whereClause}
    GROUP BY winner_wallet
    ORDER BY earnings DESC
    LIMIT ?
  `).all(...(chain ? [chain, limit] : [limit]));
}
```

### Client-Side Chain Awareness

```typescript
// Always pass chain when fetching stats
const { data: stats } = useSWR(
  activeChain ? `/api/stats/me?chain=${activeChain}` : null
);

// Show correct currency
function formatAmount(amount: string, chain: string) {
  if (chain === 'solana') {
    return `${(Number(amount) / 1e9).toFixed(4)} SOL`;
  }
  return `${(Number(amount) / 1e18).toFixed(4)} ETH`;
}
```

## Chain Switching

### EVM: Wrong Chain Detection

```typescript
import { useChainId, useSwitchChain } from 'wagmi';

function useChainGuard() {
  const chainId = useChainId();
  const { switchChainAsync } = useSwitchChain();
  const targetChainId = process.env.NODE_ENV === 'production' ? 8453 : 84532; // Base

  async function ensureCorrectChain() {
    if (chainId !== targetChainId) {
      try {
        await switchChainAsync({ chainId: targetChainId });
      } catch {
        throw new Error('Please switch your wallet to Base');
      }
    }
  }

  return { ensureCorrectChain, isWrongChain: chainId !== targetChainId };
}
```

### Solana: No Runtime Switching

Solana doesn't have a chain-switching concept like EVM. The RPC URL determines the network. There's no wallet prompt needed.

```typescript
// Client always connects to the configured RPC
const RPC_URL = process.env.NEXT_PUBLIC_SOLANA_RPC_URL!;
// Devnet and mainnet use different URLs, not different chain IDs
```

## Deposit Verification

### EVM (Event Log Verification)

```typescript
async function verifyEVMDeposit(
  txHash: string,
  roomCode: string,
  expectedWallet: string,
  expectedBuyIn: string,
): Promise<{ ok: boolean; error?: string }> {
  // 1. Get receipt
  const receipt = await publicClient.getTransactionReceipt({ hash: txHash as `0x${string}` });
  if (!receipt || receipt.status !== 'success') {
    return { ok: false, error: 'Transaction failed' };
  }

  // 2. Verify destination
  if (receipt.to?.toLowerCase() !== CONTRACT_ADDRESS.toLowerCase()) {
    return { ok: false, error: 'Wrong contract' };
  }

  // 3. Decode event logs
  const playerJoinedLog = receipt.logs.find(log => {
    // Match PlayerJoined event topic
    return log.topics[0] === PLAYER_JOINED_TOPIC;
  });

  if (!playerJoinedLog) return { ok: false, error: 'Event not found' };

  // 4. Verify fields
  const decoded = decodeEventLog({
    abi: GAME_ABI,
    data: playerJoinedLog.data,
    topics: playerJoinedLog.topics,
  });

  const expectedGameId = keccak256(toBytes(roomCode));
  if (decoded.args.gameId !== expectedGameId) {
    return { ok: false, error: 'Wrong game ID' };
  }

  if (decoded.args.player.toLowerCase() !== expectedWallet.toLowerCase()) {
    return { ok: false, error: 'Wrong player' };
  }

  // 5. Dedup
  markDepositVerified(txHash, roomCode, expectedWallet);
  return { ok: true };
}
```

### Solana (PDA State Verification)

```typescript
async function verifySolanaDeposit(
  txSignature: string,
  roomCode: string,
  playerPubkey: string,
  expectedBuyInLamports: number,
): Promise<{ ok: boolean; error?: string }> {
  // 1. Dedup
  if (isDepositVerified(txSignature)) return { ok: true };

  // 2. Read GameAccount PDA
  const gameId = roomCodeToGameId(roomCode); // SHA-256
  const gamePda = getGamePda(gameId);
  const accountInfo = await connection.getAccountInfo(gamePda);

  if (!accountInfo?.data) return { ok: false, error: 'Game account not found' };

  // 3. Deserialize (manual: skip discriminator, parse fields)
  const game = deserializeGameAccount(Buffer.from(accountInfo.data));

  // 4. Verify buy-in
  if (Number(game.buyIn) !== expectedBuyInLamports) {
    return { ok: false, error: 'Buy-in mismatch' };
  }

  // 5. Find player and check deposited flag
  const playerKey = new PublicKey(playerPubkey);
  const idx = game.players.findIndex(p => p.equals(playerKey));
  if (idx < 0) return { ok: false, error: 'Player not found' };
  if (!game.deposited[idx]) return { ok: false, error: 'Not deposited' };

  // 6. Dedup
  markDepositVerified(txSignature, roomCode, playerPubkey);
  return { ok: true };
}
```

**Key difference:** EVM verifies via transaction logs (event emitted). Solana verifies via PDA state (account data). Both need dedup (same tx can't count twice).

## Session Middleware for Sockets

```typescript
// Authenticate socket connections via session cookie
io.use(async (socket, next) => {
  const cookie = socket.handshake.headers.cookie;
  if (!cookie) return next(); // Allow unauthenticated for free-play

  const session = await getSessionFromCookie(cookie);
  if (session) {
    (socket as any).walletAddress = session.wallet_address;
    (socket as any).chain = session.chain;
  }

  next();
});

// In event handlers: trust session, not client input
socket.on('room:join', (data, cb) => {
  const walletAddress = (socket as any).walletAddress; // From session
  // NOT from data.walletAddress (client-controlled, untrusted)
});
```

## Common Bugs and Their Fixes

### 1. Session/Wallet Mismatch

**Bug:** User connects wallet A, session is for wallet A. User switches to wallet B in their extension. Session still says wallet A.

**Fix:** Check on every action:
```typescript
const sessionWallet = (socket as any).walletAddress;
const connectedWallet = data.walletAddress; // From client

if (room.isOnChain && sessionWallet !== connectedWallet) {
  return cb({ ok: false, error: 'Wallet mismatch. Please reconnect.' });
}
```

### 2. Chain Mismatch in Stats

**Bug:** User connects with Solana, wins a game. Stats page shows ETH amounts because it's not filtering by chain.

**Fix:** Always pass `chain` parameter:
```typescript
// BAD: shows mixed chain data
const stats = getPlayerStats(walletAddress);

// GOOD: chain-scoped
const stats = getPlayerStats(walletAddress, user.chain);
```

### 3. EVM Address Case Sensitivity

**Bug:** Address stored as `0xAbCd...` but compared against `0xabcd...`. Match fails.

**Fix:** Always lowercase EVM addresses:
```typescript
// Storage
await upsertUser(walletAddress.toLowerCase(), chain);

// Comparison
if (address.toLowerCase() === expectedAddress.toLowerCase()) { ... }

// Solana: exact match (base58 is case-sensitive)
if (publicKey.toBase58() === expectedAddress) { ... }
```

### 4. Phantom on Both Chains

**Bug:** User has Phantom installed. It appears as both an EVM wallet (via RainbowKit) and a Solana wallet (via wallet-adapter). User accidentally connects the wrong one.

**Fix:** The MultiChainContext `activeChain` selector determines which auth flow runs. Make the chain selector prominent in the UI before wallet connection.

### 5. Wallet Disconnects Without Event

**Bug:** Wallet extension disconnects (user revokes permission) but no disconnect event fires. App thinks wallet is still connected.

**Fix:** Periodic healthcheck:
```typescript
useEffect(() => {
  const interval = setInterval(() => {
    if (activeChain === 'base' && !evmIsConnected) {
      handleDisconnect();
    }
    if (activeChain === 'solana' && !solanaConnected) {
      handleDisconnect();
    }
  }, 5000);
  return () => clearInterval(interval);
}, []);
```

### 6. Wrong Game ID Hash

**Bug:** EVM uses `keccak256(roomCode)` for game IDs. Solana uses `SHA-256(roomCode)`. Developer uses the wrong hash for the chain and deposits go to the wrong PDA.

**Fix:** Chain-aware game ID derivation:
```typescript
function getGameId(roomCode: string, chain: string): Buffer {
  if (chain === 'solana') {
    return createHash('sha256').update(roomCode).digest();
  }
  return Buffer.from(keccak256(toBytes(roomCode)).slice(2), 'hex');
}
```

## Security Checklist

- [ ] Nonces are one-time use and expire after 5 minutes
- [ ] Session cookies are httpOnly, secure, sameSite=strict
- [ ] Wallet addresses from sessions, not from client input (for sensitive ops)
- [ ] EVM addresses compared case-insensitively
- [ ] Chain parameter validated (only 'solana' or 'base' accepted)
- [ ] RPC API keys not exposed in NEXT_PUBLIC_ vars (use proxy)
- [ ] Deposit verification includes dedup (same tx can't count twice)
- [ ] Socket auth middleware runs on every connection
- [ ] Signature verification uses the correct algorithm per chain
