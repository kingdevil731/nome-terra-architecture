# Engineering decisions

Each entry records what was chosen, what was rejected, and what it cost.

## One rules layer, imported by server and clients

**Chosen:** `game-core` — a pure package with zero runtime dependencies holding
every rule: normalization, validation, grouping, scoring, review resolution,
letter selection, stop conditions, blocklist. Server, web and mobile all import
it. Nothing in it may be reimplemented elsewhere.

**Rejected:** server-only rules with the client rendering whatever it is told.
Simpler, and it makes every score display wait on a round trip. Also rejected:
client-side prediction, where the client implements an approximation and gets
corrected — the approximation is wrong often enough to be visible, and revising
a displayed score downward is worse than a brief wait.

**Why a third option exists:** with one implementation, client and server cannot
disagree about what a score _should_ be. They can only disagree about what inputs
existed, which is exactly what the server is authoritative for. The category of
bug where two sides slowly diverge is removed rather than managed.

**Cost:** the package has to stay genuinely pure. No `Date.now`, no
`Math.random`, no I/O, no framework imports — time and randomness are injected.
That is a real constraint on anyone adding a rule, and it is enforced by
convention and review rather than by tooling.

## Randomness is injected, not called

**Chosen:** a round's letter derives from a seed that is persisted with the
round.

**Rejected:** calling `Math.random` where the letter is needed. Obvious and
untestable.

**Why:** a disputed round can be replayed exactly. "The game scored that wrong"
becomes a claim you can check rather than an argument, which matters in a game
whose rules players already know by heart.

## A distributed lock rather than a versioned store

**Chosen:** serialise room writes with `RedisRoundLock` — `SET NX PX` with a
random token, released by a Lua compare-and-delete.

**Rejected:** a `version` field on the room with retry-on-conflict. That is the
textbook answer and it would work, but a round transition touches several keys
and involves timers; retrying the whole transition on a version clash means
re-deriving state that has already been partly acted on. A mutex makes the
critical section explicit instead.

**Why the token:** the TTL is there so a crashed holder cannot wedge a room
forever. A TTL introduces its own race — a holder that has already expired must
not delete a lock a different instance now owns. Comparing the token before
deleting closes it.

**Cost:** writes to a room are serialised, which at one-lobby scale is free. And
correctness now depends on every write path taking the lock, which is a
convention rather than a property of the store.

## Lock ownership is encoded in method names

**Chosen:** entry points call `acquireRoundLockWaiting`; internals that assume
the lock is held are named `...WithLockHeld` — nine of them.

**Rejected:** passing a lock token as a typed parameter, which the compiler could
check. Considered and not done: it threads a token through every signature for a
guarantee the naming already makes hard to violate accidentally.

**Cost:** a convention, not a compiler guarantee. The mitigation is
`roomLock.race.test.ts`, which asserts that `RoomService`, `LobbyService` and
`PresenceService` all take the same lock — because an unlocked write that read
before a locked one saved puts the old room back.

## Idempotency on repeated intents

**Chosen:** `IdempotencyStore.once(key, fn)` runs an action once and replays its
result for the same key.

**Why:** acknowledgements tell a client that an intent failed so it can retry.
Retrying is only safe if the server can tell a retry from a new intent. Without
this, acks would make the system _less_ correct, not more.

**Cost:** keys have to be generated well client-side, and the store is per
instance rather than shared.

## Acknowledgements on every client-to-server event

**Chosen:** `wrapAck` wraps all handlers; failures return `{ ok: false, error }`.

**Rejected:** fire-and-forget emits, the Socket.IO default shape, which produces
the worst class of realtime bug — a client whose state diverged with no error
anywhere and no evidence of when it happened.

**Cost:** nine lines and an ack round trip per intent.

## Room state in Redis behind a repository interface

**Chosen:** `RoomRepo` interface, `InMemoryRoomRepo` for development,
`RedisRoomRepo` for production, selected at boot.

**Rejected:** in-process state, which pins the system to one instance forever.
Also rejected: rooms in PostgreSQL, which buys durability nobody needs for a
transient lobby and adds write load per submission.

**Cost:** every room read is a network hop and a deserialize, and the store has
no version field — safety comes from the lock instead.

## TTL instead of explicit room cleanup

**Chosen:** rooms expire via Redis TTL.

**Why:** the common ending for a casual lobby is abandonment, not a clean exit.
Explicit cleanup means distinguishing abandonment from temporary disconnection,
and that distinction is unreliable on mobile browsers that background
aggressively. Expiry needs no such judgement.

**Cost:** a long paused game could expire. TTL is configurable and the mitigation
is refreshing it on activity.

## Production refuses the development fallback

**Chosen:** throw at boot in production when `REDIS_URL` is absent; warn and fall
back to in-memory otherwise.

**Rejected:** a uniform fallback. The failure it produces is the expensive kind —
the server boots, appears healthy, works under one instance, and loses rooms on
restart or the moment a second instance appears.

**Cost:** a misconfigured deploy fails to start rather than starting degraded.
That is the intended trade.

## Guardrail tests for conventions

**Chosen:** `prismaImports.guardrail.test.ts` and `theme.guardrail.test.ts` fail
the build when someone imports around a boundary rather than through it.

**Why:** the conventions that matter most here — the rules layer is the only
implementation, database access goes through one path — are exactly the ones a
unit test cannot express. A test that greps the source is inelegant and it is the
only thing that actually holds the line.

## What I would change

**Bound the lock wait.** `acquireRoundLockWaiting` retries until it succeeds, so
a pathologically slow holder delays callers instead of failing them. Fine while
critical sections stay short; the thing to watch if they stop being short.

**Bring mobile into the workspace.** `nome-terra-mobile` is commented out of
`pnpm-workspace.yaml`, so it is not typechecked or tested alongside the rest —
which means the shared socket contracts are not being verified against it.

**Share the idempotency store.** It is per instance. Two instances can each run
the same keyed action once.
