# Engineering decisions

## Server-authoritative scoring, no client prediction

**Chosen:** all scoring server-side, clients render only.

**Rejected:** optimistic local scoring with server correction. The value being
predicted depends on other players' answers and on invalid-votes that have not
been cast, so the prediction is wrong often. Revising a displayed score downward
is a worse experience than a brief wait.

**Cost:** submission confirmation is a round trip, in a timed round where that is
visible. Accepted because correctness in a well-known game is non-negotiable —
players know the rules and will not forgive wrong scores.

## Room state in Redis behind a repository interface

**Chosen:** `RoomRepo` interface, `InMemoryRoomRepo` for development,
`RedisRoomRepo` for production, chosen at boot by `createRoomRepo`.

**Rejected:** in-process room state, which is simpler and pins the system to one
instance permanently. Also rejected: rooms in PostgreSQL, which gives durability
nobody needs for a transient lobby and adds write load per submission.

**Cost:** every room read is a network hop and a deserialize, and the store
itself has no version field — the read-modify-write cycle is made safe by a
separate distributed lock rather than by the store. See
[realtime-model.md](realtime-model.md).

## TTL instead of explicit room cleanup

**Chosen:** rooms expire via Redis TTL.

**Why:** the common ending for a casual lobby is abandonment, not a clean exit.
Explicit cleanup requires distinguishing abandonment from temporary
disconnection, and that distinction is unreliable on mobile browsers that
background aggressively. Expiry needs no such judgement.

**Cost:** a genuinely long-running game could expire under a paused room. TTL is
configurable, and the mitigation is refreshing it on activity.

## Production refuses the development fallback

**Chosen:** throw at boot in production when `REDIS_URL` is absent; warn and fall
back to in-memory otherwise.

**Rejected:** a uniform fallback. The failure it produces is the expensive kind —
the server boots, appears healthy, works under a single instance, and loses rooms
on restart or the moment a second instance appears. Diagnosing that from a bug
report costs hours.

**Cost:** a misconfigured deploy fails to start rather than starting degraded.
That is the intended trade.

## A distributed lock rather than a versioned store

**Chosen:** serialise round mutations with `RedisRoundLock` — `SET NX PX` with a
random token, released by a Lua compare-and-delete.

**Rejected:** adding a `version` field to the room and retrying on conflict.
That is the textbook answer and it would work, but a round transition touches
several keys and involves timers; retrying the whole transition on a version
clash means re-deriving state that has already been partly acted on. A mutex
makes the critical section explicit instead.

**Why the token:** the TTL is there so a crashed holder cannot wedge a room
forever. But a TTL introduces its own race — a holder that has already expired
must not delete a lock a different instance now owns. Comparing the token before
deleting closes it.

**Cost:** a lock is a serialisation point, so the high-frequency paths are no
longer concurrent within a room. At the scale of one lobby that is free.
Coverage is also partial, which is stated in the limitations.

## Acknowledgements on every client-to-server event

**Chosen:** `wrapAck` wraps all handlers; failures return `{ ok: false, error }`.

**Rejected:** fire-and-forget emits, which is the Socket.IO default shape and
produces the worst class of realtime bug — a client whose state diverged from the
server with no error anywhere and no evidence of when it happened.

**Cost:** nine lines and an ack round trip per intent.

## Typed socket contracts in a shared package

**Chosen:** `ClientToServerEvents` and `ServerToClientEvents` in
`@nome-terra/shared`, imported by backend and web.

**Why:** socket event names are strings. A typo or a rename that misses one side
produces an emit that matches no handler, fails silently, and looks like a
network problem. Shared types turn that into a compile error.

## Handlers split by lifecycle stage

**Chosen:** separate handler modules for lobby, rooms, round, game and
disconnect.

**Why:** round handling carries almost all the complexity — timers, submission
windows, vote collection, finalisation. Keeping it in the same file as lobby
membership guarantees the file becomes unreadable.

## What I would change

**Extend the round lock to the remaining transitions.** `startGame`,
`startNextRound`, `forceEndRound`, `endGame` and `cleanupRoom` are not yet
serialised, so they remain exposed to the read-modify-write race that the lock
removes everywhere else.

**Test the scoring engine.** Current tests cover auth, error handling, logging,
health and OpenAPI routing — the infrastructure. Scoring and round lifecycle,
which are the parts players would notice being wrong, have none. Duplicate
detection across players and invalid-vote resolution are pure functions and are
straightforward to test; not having done so is the clearest gap in the project.
