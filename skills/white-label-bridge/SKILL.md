---
name: white-label-bridge
description: Use when building a standalone cross-chain bridge widget, creating a white-label bridge product with fee collection, or scaffolding a bridge frontend. Covers fee-first architecture, deBridge integration, and branding removal.
version: 0.1.0
tools: Read, Grep, Glob, Bash, WebFetch
---

# White-Label Bridge Widget

Pattern for building standalone, white-labeled cross-chain bridge products with platform fee collection. One-click bridge with your branding, your fee, powered by deBridge (or similar) under the hood.

## When to Use

- Building a standalone bridge product / widget
- Adding a bridge feature with platform fee collection
- White-labeling an existing bridge protocol
- Creating a simple SOL-to-EVM (or reverse) bridge frontend

## Architecture

```
User connects wallet
  -> Enters amount
  -> UI shows: bridge amount, platform fee, estimated output
  -> Click "Bridge"
  -> Transaction 1: Platform fee transfer (verified on-chain)
  -> Transaction 2: Bridge transaction via deBridge
  -> Poll for completion
  -> Show success with destination tx link
```

### Why Fee-First?

The platform fee is a separate transaction that executes BEFORE the bridge. The API route verifies the fee landed on-chain before returning the bridge transaction. This ensures:

1. Fee collection is atomic -- if the bridge fails, the fee is already collected
2. No smart contract needed -- just a standard transfer to a fee wallet
3. On-chain verifiable -- anyone can audit fee payments

## Stack

| Component | Choice | Why |
|---|---|---|
| Framework | Next.js (App Router) | Server-side API routes for fee verification |
| Wallet | `@solana/wallet-adapter-react` | Standard Solana wallet connection |
| Bridge API | deBridge DLN | Best cross-chain liquidity, API-first |
| Styling | Tailwind CSS | Fast iteration |

## Implementation

### 1. Fee Configuration

```typescript
// lib/config.ts
export const BRIDGE_CONFIG = {
  feeRecipient: process.env.NEXT_PUBLIC_FEE_WALLET!,    // Your platform wallet
  feeBps: 50,                                            // 0.5% fee (50 basis points)
  minBridgeAmount: 0.1,                                  // SOL minimum
  sourceChain: 7565164,                                  // Solana
  destChain: 8453,                                       // Base
  nativeToken: '11111111111111111111111111111111',        // SOL
  destNativeToken: '0x0000000000000000000000000000000000000000', // ETH
};
```

### 2. Fee Calculation

```typescript
export function calculateFee(amountLamports: bigint, feeBps: number) {
  const fee = (amountLamports * BigInt(feeBps)) / 10000n;
  const bridgeAmount = amountLamports - fee;
  return { fee, bridgeAmount };
}
```

### 3. API Route: Verify Fee + Return Bridge Tx

```typescript
// app/api/bridge/route.ts
export async function POST(req: Request) {
  const { amount, feeTxSignature, destinationAddress } = await req.json();

  // 1. Verify fee transaction landed on-chain
  const connection = new Connection(process.env.SOLANA_RPC_URL!);
  const feeTx = await connection.getTransaction(feeTxSignature, {
    maxSupportedTransactionVersion: 0,
  });

  if (!feeTx || feeTx.meta?.err) {
    return Response.json({ error: 'Fee transaction failed or not found' }, { status: 400 });
  }

  // Verify fee went to the right wallet and amount is correct
  // (check postBalances - preBalances for fee recipient)

  // 2. Get bridge transaction from deBridge
  const { fee, bridgeAmount } = calculateFee(BigInt(amount), BRIDGE_CONFIG.feeBps);

  const bridgeRes = await fetch('https://dln.debridge.finance/v1.0/dln/order/create-tx', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      srcChainId: BRIDGE_CONFIG.sourceChain,
      srcChainTokenIn: BRIDGE_CONFIG.nativeToken,
      srcChainTokenInAmount: bridgeAmount.toString(),
      dstChainId: BRIDGE_CONFIG.destChain,
      dstChainTokenOut: BRIDGE_CONFIG.destNativeToken,
      dstChainTokenOutRecipient: destinationAddress,
      senderAddress: feeTx.transaction.message.accountKeys[0].toBase58(),
    }),
  });

  const bridgeData = await bridgeRes.json();
  return Response.json(bridgeData);
}
```

### 4. Live Quote Display

Show the estimated output amount in real-time as the user types:

```typescript
// hooks/useBridgeQuote.ts
export function useBridgeQuote(amountSol: number) {
  const [quote, setQuote] = useState<Quote | null>(null);

  useEffect(() => {
    if (amountSol < BRIDGE_CONFIG.minBridgeAmount) {
      setQuote(null);
      return;
    }

    const timeout = setTimeout(async () => {
      const amountLamports = Math.floor(amountSol * 1e9);
      const { bridgeAmount } = calculateFee(BigInt(amountLamports), BRIDGE_CONFIG.feeBps);

      const res = await fetch(
        `https://dln.debridge.finance/v1.0/dln/order/quote?` +
        `srcChainId=${BRIDGE_CONFIG.sourceChain}&` +
        `srcChainTokenIn=${BRIDGE_CONFIG.nativeToken}&` +
        `srcChainTokenInAmount=${bridgeAmount}&` +
        `dstChainId=${BRIDGE_CONFIG.destChain}&` +
        `dstChainTokenOut=${BRIDGE_CONFIG.destNativeToken}`
      );

      const data = await res.json();
      setQuote({
        estimatedOutput: data.estimation?.dstChainTokenOut?.amount,
        bridgeFee: data.estimation?.costsDetails,
      });
    }, 500); // Debounce 500ms

    return () => clearTimeout(timeout);
  }, [amountSol]);

  return quote;
}
```

### 5. Frontend Flow

```typescript
async function handleBridge() {
  // Step 1: Send fee transaction
  const feeTx = new Transaction().add(
    SystemProgram.transfer({
      fromPubkey: wallet.publicKey,
      toPubkey: new PublicKey(BRIDGE_CONFIG.feeRecipient),
      lamports: fee,
    })
  );
  const feeSig = await wallet.sendTransaction(feeTx, connection);
  await connection.confirmTransaction(feeSig, 'confirmed');

  // Step 2: Get bridge tx from API (which verifies the fee)
  const res = await fetch('/api/bridge', {
    method: 'POST',
    body: JSON.stringify({
      amount: amountLamports.toString(),
      feeTxSignature: feeSig,
      destinationAddress: destAddress,
    }),
  });
  const bridgeData = await res.json();

  // Step 3: Deserialize and send bridge tx
  // See debridge-integration skill for hex deserialization details
  const txBytes = hexToBytes(bridgeData.tx.data);
  const vtx = VersionedTransaction.deserialize(txBytes);

  // Refresh blockhash (API blockhash can be stale)
  const { blockhash, lastValidBlockHeight } = await connection.getLatestBlockhash();
  vtx.message.recentBlockhash = blockhash;

  const bridgeSig = await wallet.sendTransaction(vtx, connection, { skipPreflight: true });

  // Step 4: Poll for bridge completion
  // Extract orderId from bridge response or tx logs
  await waitForBridgeOrder(bridgeData.orderId);
}
```

## Branding Checklist

When white-labeling, strip all third-party branding:

- [ ] Remove "Powered by deBridge" or similar attribution (check footer, tooltips, meta tags)
- [ ] Replace bridge protocol logos with your own
- [ ] Custom domain (no `dln.trade` or `debridge.finance` visible to users)
- [ ] Custom loading states and success messages
- [ ] Custom error messages (don't expose raw API errors from the bridge protocol)
- [ ] OpenGraph / meta tags use your branding
- [ ] Favicon and manifest use your assets

## Error Handling

### Wallet Rejection

Users can reject any transaction in their wallet. Handle gracefully:

```typescript
try {
  const sig = await wallet.sendTransaction(tx, connection);
} catch (e: any) {
  if (e.message?.includes('User rejected')) {
    setStatus('Transaction cancelled by user');
    return; // Don't show error state, just reset
  }
  throw e; // Re-throw actual errors
}
```

### RPC Rate Limiting

Default Solana RPCs (`api.mainnet-beta.solana.com`) rate-limit aggressively. Use a dedicated RPC:

```typescript
// Good options:
// - Helius (helius.dev)
// - QuickNode
// - Triton (triton.one)
// - Public fallback: solana-rpc.publicnode.com
```

### Bridge Failure

If the bridge transaction fails after the fee was collected, the user has already paid. Options:

1. **Retry the bridge** (recommended) -- fee is already paid, just re-fetch and re-send the bridge tx
2. **Refund** -- send the fee back from the fee wallet (requires backend logic)
3. **Credit** -- track failed bridges and credit on next use

## Deployment Checklist

- [ ] Fee wallet address set in environment variables (not hardcoded)
- [ ] RPC URL set to a reliable provider (not default mainnet-beta)
- [ ] HTTPS enabled (required for wallet adapters in production)
- [ ] Fee verification logic handles edge cases (tx not found yet, wrong amount, wrong recipient)
- [ ] Bridge amount is calculated AFTER fee deduction
- [ ] Quote display matches actual execution (same fee calculation)
- [ ] All third-party branding removed
- [ ] Error states show actionable messages
- [ ] Mobile-responsive wallet connection flow
