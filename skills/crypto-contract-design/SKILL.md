---
name: crypto-contract-design
description: Use when designing smart contracts as a settlement layer for off-chain applications, building on-chain game economies, or architecting deposit/payout systems with fee vaults. Covers the off-chain logic + on-chain settlement pattern.
version: 0.1.0
tools: Read, Grep, Glob, Bash, WebFetch
---

# Crypto Contract Design: Off-Chain Logic + On-Chain Settlement

Pattern for designing minimal smart contracts that serve as a settlement layer for off-chain applications (games, platforms, services). The core principle: keep complex logic off-chain, use contracts only for deposits, payouts, and fee collection.

## When to Use

- Designing smart contracts for an existing off-chain application (game, platform, etc.)
- Building deposit/payout systems with fee collection
- Architecting on-chain game economies
- Adding crypto payments to a web application
- Any scenario where the business logic is too complex or too fast for on-chain execution

## Architecture Principle

```
Off-Chain (fast, complex, free):
  Game logic, matchmaking, real-time state, UI, social features

On-Chain (slow, simple, trustless):
  Deposits, payouts, fee collection, emergency recovery
```

The contract is a vault, not a computer. It holds funds and releases them based on attestations from a trusted backend.

## Investigation Phase

Before designing any contracts, analyze the existing off-chain system:

### 1. Map the Fund Flow

Trace every point where money enters or exits the system:

- Where do users deposit? (join game, buy item, stake)
- Where do users withdraw? (win payout, refund, unstake)
- Where does the platform take fees? (per-game, per-trade, per-withdrawal)
- Are there time-locked funds? (staking periods, cooldowns)

### 2. Identify Trust Boundaries

What must be trustless vs. what can the backend attest to?

| Trustless (on-chain) | Trusted (backend attests) |
|---|---|
| "My deposit was received" | "Player X won the game" |
| "The fee is exactly Y%" | "The game lasted 45 minutes" |
| "I can withdraw after timeout" | "Player disconnected at round 3" |

### 3. Define the Minimum Contract Surface

The contract should handle ONLY:
- Accepting deposits (with validation: min/max amounts, player count)
- Releasing payouts (based on backend attestation)
- Collecting fees (isolated in a separate vault)
- Emergency recovery (timeout-based, no backend needed)

## Two-Contract Pattern

### Contract 1: Core Game/Platform Contract

Handles deposits, game lifecycle, and payouts.

```solidity
// Pseudocode -- adapt to your chain (Solidity, Anchor, etc.)
contract Platform {
    struct Session {
        address[] players;
        uint256[] deposits;
        uint256 totalPool;
        uint256 createdAt;
        bool settled;
    }

    mapping(bytes32 => Session) public sessions;
    address public signer;          // Backend attestation key
    uint256 public feeBps;          // Fee in basis points (e.g., 500 = 5%)
    uint256 public maxFeeBps;       // Hard cap (e.g., 1000 = 10%)
    uint256 public emergencyTimeout; // e.g., 24 hours

    function deposit(bytes32 sessionId) external payable {
        // Validate: session exists, not settled, player not already in, amount >= minimum
        // Add player and deposit to session
    }

    function settle(bytes32 sessionId, address winner, bytes calldata signature) external {
        // Verify backend signature over (sessionId, winner)
        // Calculate fee: totalPool * feeBps / 10000
        // Send fee to FeeVault
        // Send remainder to winner
        // Mark settled
    }

    function emergencyWithdraw(bytes32 sessionId) external {
        // Only callable after emergencyTimeout has passed
        // Only callable if session is NOT settled
        // Returns each player's original deposit (minus any fees)
        // No backend signature needed -- this is the trustless escape hatch
    }
}
```

### Contract 2: Fee Vault (Isolated)

Separating fee collection from the main contract reduces attack surface and simplifies accounting.

```solidity
contract FeeVault {
    address public owner;
    uint256 public lastSweep;
    uint256 public sweepCooldown; // e.g., 1 hour

    receive() external payable {
        // Accept fees from Platform contract
    }

    function sweep(address to) external {
        require(msg.sender == owner);
        require(block.timestamp >= lastSweep + sweepCooldown);
        lastSweep = block.timestamp;
        payable(to).transfer(address(this).balance);
    }
}
```

**Why isolate fees?**
- Main contract never holds platform funds -- only player deposits
- Fee accounting is trivially auditable (just check vault balance)
- Compromise of fee vault doesn't affect player deposits
- Sweep cooldown prevents instant drainage

## Design Parameters

Configure these for your specific use case:

| Parameter | Example | Notes |
|---|---|---|
| `minDeposit` | 0.001 ETH / 0.01 SOL | Prevent dust spam |
| `maxPlayers` | 2-6 | Keep gas/compute manageable |
| `feeBps` | 500 (5%) | Basis points. 100 = 1%. |
| `maxFeeBps` | 1000 (10%) | Hard cap, immutable after deploy |
| `emergencyTimeout` | 24 hours | Long enough for normal games, short enough for stuck funds |
| `sweepCooldown` | 1 hour | Prevents rapid fee extraction |
| `signer` | Single EOA | Backend attestation key. Upgradeable to multisig later. |

## Backend Attestation

The backend signs a message attesting to the outcome. The contract verifies this signature on-chain:

```
Message format: keccak256(abi.encodePacked(sessionId, winner, chainId, contractAddress))
```

Include `chainId` and `contractAddress` to prevent replay attacks across chains/deployments.

### Signer Key Management

**Development:** Single EOA, private key in env var.
**Production:** Multisig or threshold signature scheme. The signer key is the single point of trust -- if compromised, the attacker can settle games fraudulently. Plan for key rotation from day one.

## Testing Strategy (Foundry / Anchor)

### Core Test Scenarios

1. **Happy path:** Deposit -> Settle -> Winner receives pool minus fee -> Fee vault receives fee
2. **Emergency withdrawal:** Deposit -> Wait timeout -> Emergency withdraw -> All players refunded
3. **Fee bounds:** Attempt to set fee above maxFeeBps -> Revert
4. **Double settle:** Attempt to settle same session twice -> Revert
5. **Invalid signature:** Settle with wrong signer -> Revert
6. **Deposit validation:** Below minimum -> Revert. Session full -> Revert. Already deposited -> Revert.
7. **Emergency before timeout:** Attempt emergency withdraw before timeout -> Revert
8. **Sweep cooldown:** Sweep vault -> Attempt immediate re-sweep -> Revert
9. **Zero-player session:** Settle with no deposits -> Handle gracefully
10. **Multi-player payout:** Verify correct split for games with multiple winners
11. **Reentrancy:** Malicious winner contract attempts reentrant call -> Protected
12. **Signer rotation:** Update signer -> Old signatures rejected, new signatures accepted

### Foundry Example

```solidity
function test_settlePaysFeeToVault() public {
    bytes32 sessionId = keccak256("game-1");

    // Two players deposit 1 ETH each
    vm.deal(player1, 1 ether);
    vm.deal(player2, 1 ether);

    vm.prank(player1);
    platform.deposit{value: 1 ether}(sessionId);

    vm.prank(player2);
    platform.deposit{value: 1 ether}(sessionId);

    // Backend signs attestation that player1 won
    bytes memory sig = signSettle(signerKey, sessionId, player1);

    uint256 balanceBefore = player1.balance;
    platform.settle(sessionId, player1, sig);

    // Player1 receives 2 ETH minus 5% fee = 1.9 ETH
    assertEq(player1.balance - balanceBefore, 1.9 ether);

    // Fee vault received 0.1 ETH
    assertEq(address(feeVault).balance, 0.1 ether);
}
```

## Scaling Considerations

### Phase 1: MVP
- Single signer EOA
- Fixed fee percentage
- Native token only (ETH/SOL)
- 2-player sessions

### Phase 2: Growth
- ERC-20/SPL token support
- Variable player counts (2-6)
- Fee tiers based on deposit size
- Event indexer for analytics

### Phase 3: Decentralization
- Multisig or threshold signer
- On-chain dispute resolution (optimistic, with challenge period)
- DAO governance for fee parameters
- Cross-chain session support

## Common Mistakes

1. **Putting game logic on-chain.** Gas costs scale with complexity. A turn-based game with 10 actions per turn across 20 turns = 200 on-chain transactions. Keep it off-chain.
2. **No emergency timeout.** If the backend goes down, player funds are locked forever. Always have a permissionless escape hatch.
3. **Fee vault in the main contract.** Mixing player deposits with platform revenue makes accounting and security harder.
4. **Hardcoded signer with no rotation.** The signer key WILL need to be rotated eventually. Build the upgrade path from day one.
5. **Missing reentrancy protection.** Any contract that sends ETH to arbitrary addresses needs reentrancy guards. Use checks-effects-interactions or OpenZeppelin's ReentrancyGuard.
