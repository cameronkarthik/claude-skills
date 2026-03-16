---
name: solana-jupiter-swap
description: Use when integrating Jupiter for Solana token swaps, building buy/sell flows, or debugging swap transaction failures. Covers quote fetching, transaction building, slippage, priority fees, and Token-2022 edge cases.
version: 0.1.0
tools: Read, Grep, Glob, Bash, WebFetch
---

# Jupiter Swap Integration

Production-tested integration guide for Jupiter Swap API v1. Built from implementing multi-wallet buy/sell flows with real SOL on mainnet.

## When to Use

- Integrating Jupiter for token swaps on Solana
- Building buy or sell flows for SPL tokens
- Debugging swap transaction failures or slippage issues
- Estimating swap output for UI display

## API Endpoints

| Purpose | Endpoint | Method |
|---|---|---|
| Get quote | `https://api.jup.ag/swap/v1/quote` | GET |
| Build swap tx | `https://api.jup.ag/swap/v1/swap` | POST |
| Token list | `https://tokens.jup.ag/tokens?tags=verified` | GET |

**All requests require the `x-api-key` header.** Get a key from the Jupiter developer portal.

## Authentication

```typescript
const JUPITER_BASE = "https://api.jup.ag";

function jupiterHeaders(apiKey: string): HeadersInit {
  return {
    "Content-Type": "application/json",
    "x-api-key": apiKey,
  };
}
```

## Quote Flow

```typescript
interface QuoteParams {
  inputMint: string;   // SOL = "So11111111111111111111111111111111111111112"
  outputMint: string;  // token contract address
  amount: string;      // in smallest unit (lamports for SOL)
  slippageBps?: number; // default 100 = 1%
}

async function getQuote(params: QuoteParams, apiKey: string) {
  const url = new URL(`${JUPITER_BASE}/swap/v1/quote`);
  url.searchParams.set("inputMint", params.inputMint);
  url.searchParams.set("outputMint", params.outputMint);
  url.searchParams.set("amount", params.amount);
  url.searchParams.set("slippageBps", String(params.slippageBps ?? 100));
  url.searchParams.set("restrictIntermediateTokens", "true");

  const res = await fetch(url.toString(), { headers: jupiterHeaders(apiKey) });
  if (!res.ok) throw new Error(`Jupiter quote failed: ${res.status}`);
  return res.json();
}
```

### Key Quote Response Fields

```typescript
interface QuoteResponse {
  inputMint: string;
  outputMint: string;
  inAmount: string;      // lamports
  outAmount: string;     // smallest token unit
  otherAmountThreshold: string; // minimum output accounting for slippage
  priceImpactPct: string;
  routePlan: Array<{
    swapInfo: { ammKey: string; label: string; inputMint: string; outputMint: string; };
    percent: number;
  }>;
}
```

## Building the Swap Transaction

```typescript
async function buildSwapTx(
  quoteResponse: QuoteResponse,
  userPublicKey: string,
  apiKey: string
) {
  const res = await fetch(`${JUPITER_BASE}/swap/v1/swap`, {
    method: "POST",
    headers: jupiterHeaders(apiKey),
    body: JSON.stringify({
      quoteResponse,
      userPublicKey,
      dynamicComputeUnitLimit: true,
      dynamicSlippage: true,
      prioritizationFeeLamports: {
        priorityLevelWithMaxLamports: {
          maxLamports: 1_000_000,
          priorityLevel: "veryHigh",
        },
      },
    }),
  });
  if (!res.ok) throw new Error(`Jupiter swap build failed: ${res.status}`);
  return res.json(); // { swapTransaction: string (base64), lastValidBlockHeight: number }
}
```

## Executing the Swap

```typescript
import { VersionedTransaction, Connection } from "@solana/web3.js";

async function executeSwap(
  swapTxBase64: string,
  keypair: Keypair,
  connection: Connection
): Promise<string> {
  const txBuf = Buffer.from(swapTxBase64, "base64");
  const vtx = VersionedTransaction.deserialize(txBuf);
  vtx.sign([keypair]);

  const sig = await connection.sendRawTransaction(vtx.serialize(), {
    skipPreflight: true,
    maxRetries: 3,
  });

  const { blockhash, lastValidBlockHeight } = await connection.getLatestBlockhash();
  await connection.confirmTransaction(
    { signature: sig, blockhash, lastValidBlockHeight },
    "confirmed"
  );

  // CRITICAL: confirmTransaction only checks inclusion, NOT success
  const txResult = await connection.getTransaction(sig, {
    commitment: "confirmed",
    maxSupportedTransactionVersion: 0,
  });
  if (txResult?.meta?.err) {
    throw new Error(`Swap failed on-chain: ${JSON.stringify(txResult.meta.err)}`);
  }

  return sig;
}
```

## Selling (Reverse Swap)

Selling is the same flow but with inputMint/outputMint reversed:

```typescript
// Buy: SOL -> Token
const buyQuote = await getQuote({
  inputMint: "So11111111111111111111111111111111111111112",  // SOL
  outputMint: tokenMint,
  amount: String(Math.floor(solAmount * 1e9)),  // lamports
}, apiKey);

// Sell: Token -> SOL
const sellQuote = await getQuote({
  inputMint: tokenMint,
  outputMint: "So11111111111111111111111111111111111111112",  // SOL
  amount: tokenBalance,  // raw token amount in smallest unit
}, apiKey);
```

### Getting Token Balance for Sell

```typescript
import { PublicKey } from "@solana/web3.js";
import { getAssociatedTokenAddress } from "@solana/spl-token";

async function getTokenBalance(
  connection: Connection,
  walletPubkey: PublicKey,
  tokenMint: PublicKey
): Promise<string> {
  const ata = await getAssociatedTokenAddress(tokenMint, walletPubkey);
  try {
    const balance = await connection.getTokenAccountBalance(ata);
    return balance.value.amount; // raw amount string
  } catch {
    return "0"; // no ATA = no balance
  }
}
```

## Critical Gotchas

### 1. SOL mint is NOT the System Program address
- SOL mint for Jupiter: `So11111111111111111111111111111111111111112` (Wrapped SOL)
- System Program: `11111111111111111111111111111111` (NOT for swaps)
- These are different. Using the wrong one silently returns empty quotes.

### 2. Reserve 10M lamports for Token-2022 swaps
Breakdown: WSOL ATA ~2.04M + Token-2022 ATA ~2.5M + tx fee 5K + priority fee ~3.5M + Jupiter internal ~2M. For standard SPL tokens, 5M lamports is usually enough.

### 3. `skipPreflight: true` is required
Jupiter transactions use address lookup tables. Preflight simulation often fails even when the transaction would succeed on-chain. Always skip it.

### 4. Dynamic slippage > fixed slippage
Setting `dynamicSlippage: true` in the swap request lets Jupiter calculate optimal slippage per route. Better than a fixed `slippageBps` for most cases.

### 5. Priority fees matter for execution speed
Without priority fees, swaps can sit in the mempool during congestion. The `prioritizationFeeLamports` config with `veryHigh` priority and a 1M lamport cap covers most situations without overpaying.

### 6. Quote amounts are strings, not numbers
All amounts in the Jupiter API are string representations of integers. Never do arithmetic on them without converting: `BigInt(quoteResponse.outAmount)`.

### 7. `restrictIntermediateTokens: true` prevents routing through illiquid pairs
Without this flag, Jupiter may route through obscure intermediate tokens that can fail or have high slippage.

### 8. Swap transactions are VersionedTransaction, not legacy Transaction
Always deserialize with `VersionedTransaction.deserialize()`, never `Transaction.from()`. The wrong deserializer produces "Versioned messages must be deserialized with VersionedMessage" errors.

### 9. Rate limits per API key
Jupiter applies per-key rate limits. If you're doing multi-wallet swaps, stagger the requests slightly (100-200ms between each) or batch quote requests before building transactions.

### 10. Vitest environment for Solana tests
Tests that use `@solana/web3.js` must run in Node environment, not jsdom. Add `// @vitest-environment node` at the top of test files or configure per-file in vitest.config.ts.

## Multi-Wallet Buy Pattern

For buying across N wallets (bundler-style):

```typescript
async function buyOnAllWallets(
  wallets: Array<{ publicKey: string; keypair: Keypair }>,
  tokenMint: string,
  amountsLamports: number[],
  connection: Connection,
  apiKey: string,
  onProgress: (pubkey: string, status: { stage?: string; error?: string; txSig?: string }) => void
) {
  const results = [];

  for (let i = 0; i < wallets.length; i++) {
    const w = wallets[i];
    try {
      onProgress(w.publicKey, { stage: "quoting" });

      const quote = await getQuote({
        inputMint: "So11111111111111111111111111111111111111112",
        outputMint: tokenMint,
        amount: String(amountsLamports[i]),
      }, apiKey);

      onProgress(w.publicKey, { stage: "building_tx" });

      const swapResult = await buildSwapTx(quote, w.publicKey, apiKey);

      onProgress(w.publicKey, { stage: "signing" });

      const sig = await executeSwap(swapResult.swapTransaction, w.keypair, connection);

      onProgress(w.publicKey, { stage: "complete", txSig: sig });
      results.push({ publicKey: w.publicKey, sig, success: true });
    } catch (err) {
      const msg = err instanceof Error ? err.message : String(err);
      onProgress(w.publicKey, { error: msg });
      results.push({ publicKey: w.publicKey, error: msg, success: false });
    }

    // Stagger to avoid rate limits
    if (i < wallets.length - 1) {
      await new Promise(r => setTimeout(r, 150));
    }
  }

  return results;
}
```

## API Proxy Route (Next.js)

When building a web app, proxy Jupiter calls through your API to avoid exposing the API key:

```typescript
// app/api/swap/quote/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function GET(req: NextRequest) {
  const { searchParams } = new URL(req.url);
  const url = new URL("https://api.jup.ag/swap/v1/quote");

  // Forward all query params
  searchParams.forEach((value, key) => url.searchParams.set(key, value));

  const res = await fetch(url.toString(), {
    headers: {
      "Content-Type": "application/json",
      "x-api-key": process.env.JUPITER_API_KEY!,
    },
  });

  const data = await res.json();
  return NextResponse.json(data, { status: res.status });
}
```

## Integration Checklist

- [ ] Obtain Jupiter API key and store in `.env`
- [ ] Set up API proxy routes to avoid exposing key in browser
- [ ] Use `So11111111111111111111111111111111111111112` for SOL (NOT the System Program address)
- [ ] Use `restrictIntermediateTokens: true` in quote requests
- [ ] Configure `dynamicSlippage: true` and priority fees in swap requests
- [ ] Deserialize with `VersionedTransaction.deserialize()` (not `Transaction.from()`)
- [ ] Send with `skipPreflight: true`
- [ ] Confirm with blockhash form, then verify `meta.err` via `getTransaction()`
- [ ] For multi-wallet: stagger requests 100-200ms apart
- [ ] For sell flows: get token balance via `getTokenAccountBalance` before quoting
- [ ] Reserve 5-10M lamports per wallet beyond swap amount for fees/ATAs
- [ ] Add `// @vitest-environment node` to test files using `@solana/web3.js`
