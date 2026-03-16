---
name: fund-recovery
description: Use when funds are stuck in ephemeral wallets, bridge transactions failed, or a pipeline errored mid-execution. Emergency recovery procedures for Solana and EVM chains. Use this BEFORE panicking.
version: 0.1.0
tools: Read, Grep, Glob, Bash, WebFetch
---

# Fund Recovery

Emergency procedures for recovering funds from failed pipelines, stuck bridge transactions, or orphaned ephemeral wallets. Every step here exists because real money was lost when it wasn't followed.

## When to Use

- Pipeline errored and funds are in ephemeral wallets
- Bridge transaction failed mid-flight (funds sent but not received)
- User closed tab during active pipeline and needs to recover
- "Insufficient balance to consolidate" errors
- "Reached end of buffer unexpectedly" or similar bridge deserialization errors
- Funds visible on-chain in ephemeral wallet but app can't access them

## Triage: Where Are the Funds?

Before doing anything, determine where the funds actually are.

### Step 1: Check Solana Ephemeral Wallets

```typescript
// For each ephemeral wallet public key
const balance = await connection.getBalance(new PublicKey(walletPubkey));
console.log(`${walletPubkey}: ${balance / LAMPORTS_PER_SOL} SOL`);
```

Or via CLI:
```bash
solana balance <PUBKEY> --url https://solana-rpc.publicnode.com
```

### Step 2: Check for Pending Bridge Orders

If bridge-out was initiated, check order status:
```bash
curl "https://dln.debridge.finance/v1.0/dln/order/<ORDER_ID>/status"
```

Possible states:
- `Created`: order submitted but not yet filled. Funds are in escrow on source chain.
- `Fulfilled`: funds delivered to destination. Check destination wallets.
- `OrderCancelled`: order was cancelled. Funds should be back on source chain.
- No order found: bridge tx may have failed before creating an order. Funds are still in source wallet.

### Step 3: Check EVM Intermediary Wallets

If bridge-out succeeded (funds went Solana -> Base), check the EVM wallet:
```bash
# Using cast (foundry)
cast balance <EVM_ADDRESS> --rpc-url https://mainnet.base.org

# Or via curl
curl -X POST https://mainnet.base.org \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_getBalance","params":["<EVM_ADDRESS>","latest"],"id":1}'
```

## Recovery Procedures

### Scenario 1: Funds in Solana Ephemeral Wallets (Most Common)

The app still has keypairs in memory (tab not closed).

**Recovery**: Use the consolidation function to sweep all wallets back to parent.

```typescript
for (const wallet of ephemeralWallets) {
  const balance = await connection.getBalance(wallet.keypair.publicKey);
  const fee = 5000;
  const sendAmount = balance - fee;

  if (sendAmount <= 0) {
    console.log(`${wallet.publicKey}: balance too low (${balance} lamports)`);
    continue;
  }

  const tx = new Transaction().add(
    SystemProgram.transfer({
      fromPubkey: wallet.keypair.publicKey,
      toPubkey: parentWalletPubkey,
      lamports: sendAmount,
    })
  );

  const { blockhash, lastValidBlockHeight } = await connection.getLatestBlockhash();
  tx.recentBlockhash = blockhash;
  tx.feePayer = wallet.keypair.publicKey;
  tx.sign(wallet.keypair);

  const sig = await connection.sendRawTransaction(tx.serialize());
  await connection.confirmTransaction({ signature: sig, blockhash, lastValidBlockHeight });
  console.log(`Recovered ${sendAmount / 1e9} SOL from ${wallet.publicKey}: ${sig}`);
}
```

**Key rules**:
- Do NOT filter out wallets with errors. Attempt ALL wallets.
- Do NOT wipe keypairs from memory until recovery is confirmed.
- If a wallet has tokens (not SOL), sell them first via Jupiter, then sweep SOL.

### Scenario 2: Funds on EVM Chain (Bridge-Back Failed)

Bridge-out succeeded but bridge-back failed. Funds are on Base (or another EVM chain).

**If app still has EVM private keys in memory**:

Option A: Export keys and import into MetaMask manually.
```
1. Copy EVM private key from the emergency export panel
2. Open MetaMask -> Import Account -> paste private key
3. Switch to Base network
4. Send ETH back to your main wallet manually
```

Option B: Build a recovery transaction programmatically.
```typescript
import { createWalletClient, http, parseEther } from "viem";
import { base } from "viem/chains";
import { privateKeyToAccount } from "viem/accounts";

async function recoverFromEvm(
  evmPrivateKey: `0x${string}`,
  destinationAddress: `0x${string}`
) {
  const account = privateKeyToAccount(evmPrivateKey);
  const client = createWalletClient({
    account,
    chain: base,
    transport: http("https://mainnet.base.org"),
  });

  // Get balance
  const balance = await client.getBalance({ address: account.address });
  const gasEstimate = 21000n * 100000000n; // ~0.0021 ETH buffer
  const sendAmount = balance - gasEstimate;

  if (sendAmount <= 0n) {
    throw new Error(`Balance too low: ${balance} wei`);
  }

  const hash = await client.sendTransaction({
    to: destinationAddress,
    value: sendAmount,
  });

  return hash;
}
```

**If app lost EVM private keys (tab was closed)**:

If keys were saved to encrypted IndexedDB vault:
1. Connect the same wallet that was used during the pipeline
2. Sign the vault unlock message
3. Retrieve keys from vault
4. Follow Option A or B above

If keys were NOT saved anywhere: **funds are lost.** This is why the `beforeunload` warning and vault backup exist.

### Scenario 3: Bridge Order in "Created" State (Stuck in Escrow)

The bridge order was created but never fulfilled. Funds are locked in deBridge escrow.

```bash
# Check order status
curl "https://dln.debridge.finance/v1.0/dln/order/<ORDER_ID>/status"
```

If status is `Created` for more than 10 minutes:
1. The order may be unfillable (amount too small, destination chain congested)
2. deBridge has auto-cancellation for orders older than ~30 minutes
3. After cancellation, funds return to the source wallet automatically
4. Monitor the source wallet balance -- it should increase when the order cancels

If status is `OrderCancelled`:
- Funds should already be back in the source wallet
- Check the source wallet balance to confirm

### Scenario 4: Tab Closed, No Keypairs, Funds on Solana

If you know the public keys of the ephemeral wallets (from logs, Supabase trade records, or browser console history):

**You cannot recover without the private keys.** The keypairs were generated in memory and are gone.

**Prevention for next time**:
- Use encrypted IndexedDB vault with wallet-signature-derived key
- The vault persists across page refreshes
- On reconnect, sign the unlock message to restore keypairs

### Scenario 5: Token Balance in Ephemeral Wallet (Not SOL)

The wallet has SPL tokens that need to be sold before consolidating SOL.

```typescript
// 1. Get token balance
const ata = await getAssociatedTokenAddress(tokenMint, wallet.keypair.publicKey);
const tokenBalance = await connection.getTokenAccountBalance(ata);

// 2. Sell via Jupiter
const quote = await getQuote({
  inputMint: tokenMint.toBase58(),
  outputMint: "So11111111111111111111111111111111111111112",
  amount: tokenBalance.value.amount,
});
const swapResult = await buildSwapTx(quote, wallet.publicKey, apiKey);
await executeSwap(swapResult.swapTransaction, wallet.keypair, connection);

// 3. Now consolidate SOL
// ... standard consolidation flow
```

### Scenario 6: "Insufficient Balance to Consolidate"

The wallet has SOL but less than the transaction fee (5000 lamports / 0.000005 SOL).

```
Balance: 4,500 lamports
Fee:     5,000 lamports
Result:  Cannot send (negative amount)
```

**Resolution**: This amount is dust. It's not recoverable. At current SOL prices this is fractions of a cent. Log it and move on.

If multiple wallets have small balances that together are meaningful, you would need to fund one wallet with enough to pay its own fee, then daisy-chain transfers. Usually not worth the effort.

## Prevention Checklist

After any recovery, verify these are in place to prevent recurrence:

- [ ] Cancel button visible at ALL pipeline stages (never disabled)
- [ ] Recovery attempts ALL wallets, not just non-errored ones
- [ ] State (keypairs) preserved until recovery confirms
- [ ] beforeunload warning active during pipeline
- [ ] Emergency key export panel visible on ERROR state
- [ ] Encrypted IndexedDB vault for persistence across page refresh
- [ ] Trade records created AFTER funding confirmed (not before)
- [ ] Bridge reserve is 35M+ lamports (0.035 SOL) per wallet
- [ ] confirmTransaction followed by getTransaction + meta.err check
- [ ] Bridge tx deserialized with hexToBytes (not Buffer.from base64)

## Post-Recovery Audit

After recovering funds:

1. **Check total recovered vs total deposited**. Account for tx fees, bridge fees, and any swap slippage.
2. **Document what failed and why** in the project's CLAUDE.md or memory.
3. **Add a test case** that reproduces the failure condition.
4. **Verify the fix** by running a small test transaction (0.01 SOL) through the full pipeline.
