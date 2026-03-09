---
name: socket-io-game-state
description: Use when building real-time multiplayer games with Socket.IO, managing room lifecycles, handling deposits and settlements, or implementing disconnect/reconnection logic. Covers the full game state machine pattern.
version: 0.1.0
tools: Read, Grep, Glob, Bash
---

# Socket.IO Real-Time Game State Management

Patterns for building real-time multiplayer games with Socket.IO. Covers room lifecycle, deposit verification, turn management, disconnect handling, and settlement flows. Built from a production game with real crypto stakes.

## When to Use

- Building a real-time multiplayer game with Socket.IO
- Managing room/lobby lifecycles (create, join, leave, destroy)
- Implementing deposit verification for on-chain games
- Handling player disconnects and reconnections mid-game
- Building turn-based game state machines
- Implementing settlement and refund flows

## Architecture Principle

```
Client (React + Socket.IO)
  -> Emits action events (room:join, game:roll, game:buy)
  -> Receives state events (room:state, game:state)
  -> Never trusts local state -- server is source of truth

Server (Node.js + Socket.IO)
  -> Validates every action (Zod schemas)
  -> Manages room state machine (lobby -> playing -> finished)
  -> Broadcasts sanitized state to all players
  -> Signs settlements/cancellations for on-chain claims
```

**The server is the single source of truth.** Clients emit intents, the server validates and broadcasts the resulting state. Never let clients modify game state directly.

## Room Lifecycle

### State Machine

```
LOBBY -> PLAYING -> FINISHED
  |         |
  |         +-> CANCELLED (force cancel)
  |
  +-> DESTROYED (all players leave)
```

### Room Creation

```typescript
socket.on('room:create', (data, cb) => {
  // 1. Validate input (Zod schema)
  const parsed = roomCreateSchema.safeParse(data);
  if (!parsed.success) return cb({ ok: false, error: 'Invalid input' });

  // 2. Rate limit (prevent room spam)
  if (isRateLimited(socket.id, 'create')) return cb({ ok: false, error: 'Too fast' });

  // 3. Auth check for on-chain rooms
  if (parsed.data.buyInEth && !sessionWallet) {
    return cb({ ok: false, error: 'Authentication required' });
  }

  // 4. Generate unique room code
  const code = generateRoomCode(); // 6-char alphanumeric, avoid O/I/L

  // 5. Create room
  const room: Room = {
    code,
    hostId: socket.id,
    players: [createPlayer(socket.id, parsed.data)],
    phase: 'lobby',
    maxPlayers: parsed.data.maxPlayers ?? 4,
    isOnChain: !!parsed.data.buyInEth,
    chain: parsed.data.chain ?? null,
    buyInEth: parsed.data.buyInEth,
    entryFeeLamports: parsed.data.entryFeeLamports,
  };

  roomManager.set(code, room);
  socket.join(code);

  cb({ ok: true, code });
  broadcastRoomState(code);
});
```

### Room Code Generation

```typescript
function generateRoomCode(): string {
  // Avoid confusing characters: O (zero), I (one), L (one)
  const chars = 'ABCDEFGHJKMNPQRSTUVWXYZ23456789';
  let code: string;
  do {
    code = Array.from({ length: 6 }, () =>
      chars[Math.floor(Math.random() * chars.length)]
    ).join('');
  } while (roomManager.has(code));
  return code;
}
```

### Slot Reservation (On-Chain Games)

For games with deposits, there's a race condition: a player validates their slot, starts an on-chain deposit (takes 5-30s), and another player takes their slot in the meantime.

```typescript
// Reserve a slot for 60 seconds before deposit
const roomReservations = new Map<string, SlotReservation[]>();

interface SlotReservation {
  socketId: string;
  walletAddress: string;
  expiresAt: number;
}

socket.on('room:validate-join', (data, cb) => {
  const room = roomManager.get(data.code);
  if (!room) return cb({ ok: false, error: 'Room not found' });

  // Check capacity including active reservations
  const activeReservations = (roomReservations.get(data.code) ?? [])
    .filter(r => r.expiresAt > Date.now());

  const effectiveCount = room.players.length + activeReservations.length;
  if (effectiveCount >= room.maxPlayers) {
    return cb({ ok: false, error: 'Room is full' });
  }

  // Reserve slot
  activeReservations.push({
    socketId: socket.id,
    walletAddress: data.walletAddress,
    expiresAt: Date.now() + 60_000,
  });
  roomReservations.set(data.code, activeReservations);

  cb({ ok: true });
});
```

### Joining with Deposit Verification

```typescript
socket.on('room:join', async (data, cb) => {
  const room = roomManager.get(data.code);

  // On-chain: verify deposit before allowing join
  if (room.isOnChain && data.txHash) {
    let verification;
    if (room.chain === 'solana') {
      verification = await verifySolanaDeposit(
        data.txHash, data.code, gameId, playerPubkey, room.entryFeeLamports
      );
    } else {
      verification = await verifyDeposit(
        data.txHash, data.code, walletAddress, room.buyInEth
      );
    }

    if (!verification.ok) {
      return cb({ ok: false, error: verification.error });
    }
  }

  // Add player to room
  room.players.push(createPlayer(socket.id, data));
  socket.join(data.code);

  // Clear reservation
  clearReservation(data.code, socket.id);

  cb({ ok: true });
  broadcastRoomState(data.code);
});
```

## State Broadcasting

### Sanitized Room State

Never send raw server state to clients. Sanitize per-player:

```typescript
function broadcastRoomState(code: string) {
  const room = roomManager.get(code);

  for (const player of room.players) {
    const socket = io.sockets.sockets.get(player.socketId);
    if (!socket) continue;

    socket.emit('room:state', {
      code: room.code,
      phase: room.phase,
      isOnChain: room.isOnChain,
      chain: room.chain,
      buyInEth: room.buyInEth,
      players: room.players.map(p => ({
        name: p.name,
        color: p.color,
        character: p.characterId,
        ready: p.ready,
        connected: p.connected,
        isHost: p.socketId === room.hostId,
        isYou: p.socketId === player.socketId,  // Per-player field
        deposited: p.deposited,
      })),
    });
  }
}
```

### Sanitized Game State

Remove sensitive data (deck contents, random seeds) before broadcasting:

```typescript
function broadcastGameState(code: string) {
  const room = roomManager.get(code);
  const state = room.gameState;

  const sanitized = {
    ...state,
    chanceDeck: undefined,       // Never expose deck order
    communityChestDeck: undefined,
    randomSeed: undefined,
  };

  io.to(code).emit('game:state', sanitized);
  room.lastActivity = Date.now();  // Track for stale detection
}
```

### Private State (Trades)

Some state should be visible only to involved players:

```typescript
function broadcastTradeState(code: string, tradeOffer: TradeOffer) {
  const room = roomManager.get(code);

  for (const player of room.players) {
    const socket = io.sockets.sockets.get(player.socketId);
    if (!socket) continue;

    const isInTrade = player.index === tradeOffer.fromIndex ||
                      player.index === tradeOffer.toIndex;

    socket.emit('game:state', {
      ...sanitizedState,
      tradeOffer: isInTrade ? tradeOffer : undefined,
    });
  }
}
```

## Input Validation

### Zod Schemas for Every Event

```typescript
import { z } from 'zod';

const roomCreateSchema = z.object({
  name: z.string().min(1).max(20),
  color: z.string(),
  characterId: z.string().optional(),
  maxPlayers: z.number().int().min(2).max(6).optional(),
  buyInEth: z.string().optional(),
  entryFeeLamports: z.number().optional(),
  chain: z.enum(['solana', 'base']).optional(),
});

const roomJoinSchema = z.object({
  code: z.string().length(6),
  name: z.string().min(1).max(20),
  color: z.string(),
  txHash: z.string().optional(),
  walletAddress: z.string().optional(),
});

const gameActionSchema = z.object({
  code: z.string().length(6),
});
```

**Every socket event gets validated with Zod.** No exceptions. Malicious clients can send anything.

### Rate Limiting

```typescript
const rateLimits = new Map<string, number[]>();

function isRateLimited(socketId: string, action: string, limit = 10, window = 1000): boolean {
  const key = `${socketId}:${action}`;
  const now = Date.now();
  const timestamps = (rateLimits.get(key) ?? []).filter(t => t > now - window);

  if (timestamps.length >= limit) return true;

  timestamps.push(now);
  rateLimits.set(key, timestamps);
  return false;
}

// Apply to all events
socket.on('game:roll', (data, cb) => {
  if (isRateLimited(socket.id, 'action')) return;
  // ... handle action
});

// Stricter limit for chat
socket.on('chat:send', (data, cb) => {
  if (isRateLimited(socket.id, 'chat', 2, 1000)) return; // 2 msgs/sec
  // ... XSS sanitize + broadcast
});
```

## Disconnect Handling

### The 2-Minute Reconnection Window

```typescript
const DISCONNECT_TIMEOUT_MS = 120_000; // 2 minutes
const disconnectTimers = new Map<string, NodeJS.Timeout>();

socket.on('disconnect', () => {
  const code = roomManager.getCodeBySocket(socket.id);
  if (!code) return;

  const room = roomManager.get(code);
  const player = room.players.find(p => p.socketId === socket.id);
  if (!player) return;

  if (room.phase === 'finished') {
    // Game over: just mark disconnected, preserve winner access
    player.connected = false;
    return;
  }

  if (room.phase === 'playing') {
    // Mid-game: start reconnection timer
    player.connected = false;
    io.to(code).emit('player:disconnected', { playerIndex: player.index });

    disconnectTimers.set(socket.id, setTimeout(() => {
      handleDisconnectTimeout(code, socket.id);
    }, DISCONNECT_TIMEOUT_MS));
  }

  if (room.phase === 'lobby') {
    // Lobby: remove player immediately
    removePlayerFromLobby(code, socket.id);
  }
});
```

### Reconnection

```typescript
socket.on('room:reconnect', (data, cb) => {
  const room = roomManager.get(data.code);
  if (!room) return cb({ ok: false, error: 'Room not found' });

  // Find player by wallet address (socket ID changed)
  const player = room.players.find(p => p.walletAddress === data.walletAddress);
  if (!player) return cb({ ok: false, error: 'Not in this room' });

  // Cancel disconnect timer
  const timer = disconnectTimers.get(player.socketId);
  if (timer) {
    clearTimeout(timer);
    disconnectTimers.delete(player.socketId);
  }

  // Update socket mapping
  player.socketId = socket.id;
  player.connected = true;
  socket.join(data.code);

  io.to(data.code).emit('player:reconnected', { playerIndex: player.index });

  cb({ ok: true });
  broadcastRoomState(data.code);
  broadcastGameState(data.code);
});
```

### Disconnect Timeout Handler

```typescript
async function handleDisconnectTimeout(code: string, socketId: string) {
  const room = roomManager.get(code);
  const player = room.players.find(p => p.socketId === socketId);
  if (!player || player.connected) return; // Reconnected in time

  // Bankrupt the disconnected player
  try {
    declareBankruptcy(room.gameState, player.index);
  } catch (err) {
    // Bankruptcy failed -- force cancel for refunds
    await forceGameCancellation(code, 'disconnect-bankruptcy-error');
    return;
  }

  // Check if only one player remains
  const remaining = room.players.filter(p => !p.bankrupt);
  if (remaining.length === 1) {
    // Last player standing wins
    forceWinForRemaining(code, remaining[0]);
  }

  broadcastGameState(code);
}
```

### All-Disconnected Auto-Cancel

If every player disconnects from an on-chain game, auto-cancel for refunds:

```typescript
function checkAllDisconnected(code: string) {
  const room = roomManager.get(code);
  if (!room.isOnChain) return;

  const allDisconnected = room.players.every(p => !p.connected);
  if (allDisconnected) {
    forceGameCancellation(code, 'stale-sweep-all-disconnected');
  }
}
```

## Turn Management

### Turn Timer

```typescript
const TURN_TIMEOUT_MS = 60_000; // 60 seconds per turn
const turnTimers = new Map<string, NodeJS.Timeout>();

function startTurnTimer(code: string) {
  // Clear existing timer
  const existing = turnTimers.get(code);
  if (existing) clearTimeout(existing);

  const room = roomManager.get(code);
  const currentPlayer = room.gameState.players[room.gameState.currentTurn];

  // Broadcast countdown to clients
  io.to(code).emit('turn:timer', {
    playerIndex: room.gameState.currentTurn,
    timeoutMs: TURN_TIMEOUT_MS,
  });

  turnTimers.set(code, setTimeout(() => {
    handleTurnTimeout(code);
  }, TURN_TIMEOUT_MS));
}

function handleTurnTimeout(code: string) {
  const room = roomManager.get(code);
  const state = room.gameState;

  // Track idle warnings
  const player = state.players[state.currentTurn];
  player.idleWarnings = (player.idleWarnings ?? 0) + 1;

  if (player.idleWarnings >= 4) {
    // 4th timeout: force bankruptcy
    declareBankruptcy(state, state.currentTurn);
  } else {
    // Auto-advance: skip their turn
    advanceTurn(state);
  }

  broadcastGameState(code);
}
```

## Settlement Flow

### Winner Determination

```typescript
function handleGameOver(code: string) {
  const room = roomManager.get(code);
  const state = room.gameState;

  room.phase = 'finished';

  if (!room.isOnChain) return; // Free play: no settlement needed

  const winner = state.players[state.winner];

  // Sign settlement for on-chain claim
  if (room.chain === 'solana') {
    const settlement = signSolanaSettlement(code, winner.walletAddress);
    io.to(code).emit('game:settlement:signature', {
      chain: 'solana',
      ...settlement,
    });
  } else {
    const settlement = await signSettlement(code, winner.walletAddress);
    io.to(code).emit('game:settlement:signature', {
      chain: 'base',
      ...settlement,
    });
  }

  // Persist for offline claiming
  persistSettlement(code, winner.walletAddress, settlement);
}
```

### Force Cancellation (Refund Path)

```typescript
async function forceGameCancellation(code: string, reason: string) {
  // Mutual exclusion: prevent both settlement AND cancellation
  await withRoomLock(code, async () => {
    // Skip if settlement already exists
    if (hasSettlement(code)) return;
    // Skip if cancellation already exists
    if (hasCancellation(code)) return;

    const room = roomManager.get(code);

    let cancellation;
    if (room.chain === 'solana') {
      cancellation = signSolanaCancellation(code);
    } else {
      cancellation = await signCancellation(code);
    }

    // Persist for all deposited players
    const refunds = room.players
      .filter(p => p.deposited)
      .map(p => ({
        walletAddress: p.walletAddress,
        roomCode: code,
        gameId: cancellation.gameId,
        nonce: cancellation.nonce,
        signature: cancellation.signature,
        timestamp: Date.now(),
        reason,
        chain: room.chain,
      }));

    await appendPendingRefunds(refunds);

    room.phase = 'finished';
    io.to(code).emit('game:cancellation:signature', cancellation);
  });
}
```

## Client-Side Socket Integration

### Connection Setup

```typescript
import { io, Socket } from 'socket.io-client';

let socket: Socket | null = null;

export function getSocket(): Socket {
  if (!socket) {
    socket = io({
      autoConnect: true,
      reconnection: true,
      reconnectionAttempts: 10,
      reconnectionDelay: 1000,
    });
  }
  return socket;
}
```

### React Context

```typescript
function SocketProvider({ children }: { children: React.ReactNode }) {
  const [roomState, setRoomState] = useState<RoomState | null>(null);
  const [gameState, setGameState] = useState<GameState | null>(null);
  const [lastGameStateAt, setLastGameStateAt] = useState<number>(0);

  useEffect(() => {
    const socket = getSocket();

    socket.on('room:state', setRoomState);
    socket.on('game:state', (state) => {
      setGameState(state);
      setLastGameStateAt(Date.now());
    });
    socket.on('player:disconnected', ({ playerIndex }) => {
      setDisconnectedPlayers(prev => new Set([...prev, playerIndex]));
    });
    socket.on('player:reconnected', ({ playerIndex }) => {
      setDisconnectedPlayers(prev => {
        const next = new Set(prev);
        next.delete(playerIndex);
        return next;
      });
    });

    return () => {
      socket.off('room:state');
      socket.off('game:state');
      socket.off('player:disconnected');
      socket.off('player:reconnected');
    };
  }, []);

  return (
    <SocketContext.Provider value={{ roomState, gameState, socket: getSocket() }}>
      {children}
    </SocketContext.Provider>
  );
}
```

### Game Stuck Detection

Detect when the game state hasn't updated in too long (server crash, network partition):

```typescript
useEffect(() => {
  if (roomPhase !== 'playing' || gameState?.gameOver) return;

  const interval = setInterval(() => {
    const elapsed = Date.now() - lastGameStateAt;
    if (elapsed > 90_000) { // 90 seconds without update
      setGameStuck(true);
    }
  }, 10_000);

  return () => clearInterval(interval);
}, [roomPhase, lastGameStateAt]);
```

## Socket Event Reference

### Client -> Server

| Event | Purpose | Callback |
|---|---|---|
| `room:create` | Create new room | `{ ok, code?, error? }` |
| `room:validate-join` | Reserve slot before deposit | `{ ok, error? }` |
| `room:join` | Join room (with optional txHash) | `{ ok, error? }` |
| `room:leave` | Leave room | `{ ok }` |
| `room:ready` | Toggle ready status | `{ ok }` |
| `room:start` | Start game (host only) | `{ ok, error? }` |
| `room:reconnect` | Reconnect after disconnect | `{ ok, error? }` |
| `room:deposit` | Verify Solana deposit | `{ ok, error? }` |
| `room:base-deposit` | Verify Base deposit | `{ ok, error? }` |
| `game:roll` | Roll dice | - |
| `game:buy` | Purchase property | - |
| `game:decline` | Decline purchase | - |
| `game:bankruptcy` | Declare bankruptcy | - |
| `chat:send` | Send chat message | - |

### Server -> Client

| Event | Purpose | Payload |
|---|---|---|
| `room:state` | Full room state | `{ code, phase, players[] }` |
| `room:error` | Error message | `string` |
| `game:state` | Full game state (sanitized) | `GameState` |
| `player:disconnected` | Player went offline | `{ playerIndex }` |
| `player:reconnected` | Player came back | `{ playerIndex }` |
| `player:deposited` | Deposit verified | `{ playerIndex }` |
| `game:settlement:signature` | Winner can claim | `{ nonce, signature, gameId }` |
| `game:cancellation:signature` | Players can refund | `{ nonce, signature, gameId }` |
| `turn:timer` | Turn countdown | `{ playerIndex, timeoutMs }` |
| `chat:message` | Chat message | `{ from, text, timestamp }` |

## Stale Room Cleanup

Periodically sweep for abandoned rooms:

```typescript
setInterval(() => {
  const now = Date.now();

  for (const [code, room] of roomManager.entries()) {
    // Inactive for 2+ hours
    if (now - room.lastActivity > 2 * 60 * 60 * 1000) {
      if (room.isOnChain && room.phase === 'playing') {
        forceGameCancellation(code, 'stale-sweep-inactive-2h');
      }
      roomManager.delete(code);
    }
  }
}, 60_000); // Check every minute
```

## Common Mistakes

1. **Trusting client state.** Never accept game state modifications from clients. The server computes all state transitions.
2. **Broadcasting raw server objects.** Always sanitize before sending (remove deck order, random seeds, other players' hidden info).
3. **No reconnection window.** Without a timer, disconnected players lose immediately. 2 minutes is a good default.
4. **No slot reservation.** For on-chain games, the deposit takes time. Reserve the slot first.
5. **No mutual exclusion on settlement/cancellation.** Both can be triggered simultaneously by race conditions. Use room-level locks.
6. **No stale room cleanup.** Rooms accumulate in memory. Sweep inactive rooms periodically.
7. **No rate limiting.** Malicious clients can spam events. Rate limit everything.
8. **No idle detection.** AFK players block the game. Use turn timers with escalating consequences.
