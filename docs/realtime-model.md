# Realtime model

## Authority

The server owns round phase, the timer, submissions and scoring. Clients render
state and send intents. There is no client-side prediction.

That is a deliberate cost. Prediction would make submission feel instant, and it
is rejected because the thing being predicted cannot be predicted: whether an
answer scores full, half or zero depends on what every other player wrote and on
votes that have not happened yet. A client that guesses will be wrong regularly,
and correcting a visible score downward is worse than a small delay on submit.

Where responsiveness matters — the moment of typing an answer — the client is
free to reflect local input immediately, because that is display, not scoring.

## Room lifecycle

```mermaid
stateDiagram-v2
    [*] --> Lobby: room:create
    Lobby --> Lobby: player joins / proposes column / votes
    Lobby --> RoundActive: game:start (letter generated)
    RoundActive --> Review: timer expires or all submitted
    Review --> Review: round:voteInvalid
    Review --> Scoring: round:finalize
    Scoring --> Lobby: next round
    Scoring --> [*]: game ends
    RoundActive --> RoundActive: player reconnects
    Review --> Review: player reconnects
```

Reconnection is not a separate state. A returning player is resolved against the
room's current phase and re-sent the state for that phase including their own
prior submissions, rather than being treated as a new join. Treating reconnect as
a distinct flow means writing recovery logic per phase; resolving into the
existing phase means one path.

## State ownership

| State                                   | Home                        | Why                                         |
| --------------------------------------- | --------------------------- | ------------------------------------------- |
| Room, players, round, submissions       | Redis, via `RoomRepo`       | Shared across instances; naturally expiring |
| Socket-to-player mapping                | `socketRegistry` in process | Tied to the connection, dies with it        |
| Match history, seasons, stats, profiles | PostgreSQL                  | Durable, queried outside gameplay           |

The split is by lifetime. Anything whose life equals the room's belongs in Redis
with the room's TTL. Anything that outlives the room belongs in Postgres.
Anything that dies with the socket stays in process, because persisting it would
mean cleaning it up.

## Redis room store

`RedisRoomRepo` keys rooms by ID, maintains a separate code index so players can
join by short code, and holds a set of live room IDs. Multi-key updates — writing
the room and its code index together — go through `MULTI` so a crash cannot leave
a code pointing at a room that does not exist.

TTL is configured via `ROOM_TTL_SECONDS` and applied per room. Abandonment is the
normal end state for a casual lobby, and expiry handles it without needing to
distinguish "abandoned" from "everyone briefly disconnected" — which is precisely
the distinction that is hard to make correctly.

### Serialising round mutations

Room mutation is read-modify-write:

```
findById(roomId) → mutate in memory → save(room)
```

There is no `WATCH` and no version field on the store itself, so concurrent
writers would interleave and the later write would win wholesale. Rather than
version the store, the round transitions are serialised with a distributed mutex:

```ts
async acquire(roomId: string, ttlMs = 10_000): Promise<string | null> {
  const token = randomUUID();
  const res = await this.redis.set(this.key(roomId), token, { NX: true, PX: ttlMs });
  return res === 'OK' ? token : null;
}
```

Release runs a Lua script that deletes the key only if the stored token still
matches, so a holder whose lock has already expired cannot delete the lock a
different instance now owns. The TTL exists because a crashed holder must not
wedge a room permanently; the token exists because the TTL introduces exactly
that race.

`RoundLock` is an interface with `InMemoryRoundLock` and `RedisRoundLock`
implementations, chosen at boot on the same rule as the room repository.

**Coverage is partial.** The lock wraps `submit`, `vote`, `endRound`,
`stopRound`, `finalizeRound`, `pickLetterAndStartRound` and `sendReaction` — the
high-frequency paths where several players act at once. It does not wrap
`startGame`, `startNextRound`, `forceEndRound`, `endGame` or `cleanupRoom`, which
are host-initiated and rarer. Those remain exposed to the read-modify-write race
described above. Extending the lock to them is the next piece of work, and it is
mechanical rather than difficult.

## Transport

`@socket.io/redis-adapter` fans events out across instances via a pub/sub pair,
so a broadcast from any node reaches sockets connected to every other node.

Handlers are registered per lifecycle stage — `lobby`, `rooms`, `round`, `game`,
`disconnect` — and every one is wrapped by `wrapAck`, so a rejected or failed
intent returns `{ ok: false, error }` to the caller rather than disappearing.

Event contracts live in `@nome-terra/shared` as `ClientToServerEvents` and
`ServerToClientEvents`. Both the backend and the Next.js client import them, so
renaming an event breaks the build on both sides at once — which is the failure
mode you want, given the alternative is an emit that silently matches no handler.
