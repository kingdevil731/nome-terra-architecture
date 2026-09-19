# Nome Terra

Architecture showcase for a realtime multiplayer word game. The implementation is
private; this repository documents the system design.

**Live:** [nometerra.kingdevil731.dev](https://nometerra.kingdevil731.dev) (beta)

---

## The game

Players join a room, vote on which categories to play, receive a random letter,
and submit one answer per category under a timer. Everyone then reviews everyone
else's answers and votes out invalid ones. Scores account for unique answers,
duplicates between players, and entries voted invalid. Repeat across rounds and
seasons.

It is the Portuguese/Brazilian game _Stop!_ — known locally as Nome Terra — which
matters architecturally because the rules are fixed and well known. Players
notice immediately if scoring is wrong, and a scoring disagreement mid-round is
the failure that loses a lobby.

## Why the architecture is shaped this way

Three properties drive everything:

**Scoring must be identical for every player.** Duplicate detection is inherently
global — whether your answer scores full or half depends on what everyone else
wrote. No client can compute it correctly, so scoring is server-authoritative
with no exception.

**Rounds are timed, so latency is visible.** A player typing at 1.8 seconds
remaining expects their answer to count. The submission path is the only place in
the system where responsiveness is allowed to complicate the design.

**Players drop and return.** Mobile browsers background aggressively, and a
player who reconnects mid-round must land back in the correct phase with their
own submissions intact, not in a fresh lobby.

## Shape

```mermaid
flowchart TB
    subgraph nodes[Backend instances]
        N1[Node 1<br/>Socket.IO]
        N2[Node 2<br/>Socket.IO]
    end

    C1[Client] --> N1
    C2[Client] --> N2

    N1 <-->|@socket.io/redis-adapter<br/>pub/sub fanout| RD[(Redis)]
    N2 <-->|""| RD

    RD -.->|RoomRepo<br/>room state + TTL| RD

    N1 --> PG[(PostgreSQL<br/>history, seasons, stats)]
    N2 --> PG

    SH[["@nome-terra/shared<br/>socket contracts, DTOs, Zod"]]
    C1 --- SH
    N1 --- SH
```

pnpm workspace with three packages: `nome-terra-backend`, `nome-terra-web`
(Next.js/React), and `nome-terra-shared`.

## Three things worth reading about

### Room state lives in Redis behind a repository interface

Room state is not in process memory. `RoomRepo` is an interface with two
implementations — `InMemoryRoomRepo` for local development and `RedisRoomRepo`
keyed by room ID with a code index and a TTL — selected at boot.

The TTL is the part that earns its place. Abandoned rooms are the default outcome
in a casual multiplayer game: someone creates a lobby, nobody joins, they close
the tab. Without expiry that accumulates forever, and explicit cleanup means
detecting abandonment, which is exactly the thing that is hard when players
disconnect and return legitimately. Letting Redis expire the key sidesteps the
detection problem.

→ [Realtime model](docs/realtime-model.md)

### Production refuses to degrade silently

Both the socket adapter and the room repository check for `REDIS_URL` and take
different paths by environment:

```ts
if (!env.REDIS_URL) {
  if (env.isProduction) {
    console.error(
      "[repo] production fallback prevented: Redis room repo requires REDIS_URL",
    );
    throw new Error("REDIS_URL_REQUIRED_FOR_PRODUCTION_ROOM_REPO");
  }
  console.warn(
    "[repo] Redis room repo disabled; using InMemoryRoomRepo for non-production",
  );
  return new InMemoryRoomRepo();
}
```

The in-memory fallback is a genuine convenience locally — no Redis needed to run
the game. In production the same fallback is a trap: the server boots, works
correctly under one instance, and breaks in a way that looks like random room
loss the moment a second instance exists or the process restarts. Crashing at
boot is louder and cheaper than debugging that.

→ [Engineering decisions](docs/decisions.md)

### Every client-to-server event is acknowledged

Socket handlers are wrapped so a thrown error becomes a structured negative
acknowledgement rather than a lost event:

```ts
export function wrapAck<TPayload, TAck = unknown>(
  fn: (payload: TPayload, ack?: (res: TAck) => void) => Promise<void>,
) {
  return (payload: TPayload, ack?: (res: TAck) => void) => {
    fn(payload, ack).catch((e) => {
      ack?.({ ok: false, error: (e as Error).message || "UNKNOWN" } as TAck);
    });
  };
}
```

Fire-and-forget emits are the standard way realtime games become unexplainable.
A submission that vanishes with no error leaves the client showing a state the
server does not have, and the player finds out at scoring. Nine lines make every
failure addressable at the call site.

Socket event contracts (`ClientToServerEvents`, `ServerToClientEvents`) are typed
in `@nome-terra/shared` and imported by both sides, so an event name or payload
change is a compile error rather than a silent no-op.

## Structure

```
nome-terra-backend/src/
  realtime/
    socketServer.ts          adapter setup, boot
    gateways/gameGateway.ts
    gateways/handlers/       lobby · rooms · round · game · disconnect
    context/socketRegistry.ts
    redis/redisClient.ts
  modules/game/
    repositories/            roomRepo · redisRoomRepo · createRoomRepo
    services/                seasonService · playerSeasonStatsService
    history/ playerProfile/ stats/
  api/game/                  rooms · seasons · stats · social · history · profile
```

Handlers are split by lifecycle stage rather than kept in one connection handler,
because the round handlers are where the real complexity is and they should not
share a file with lobby chat.

Alongside the game: seasons, player profiles, per-season stats, match history,
social features, analytics, and an OpenAPI-documented REST surface for everything
that is not realtime.

## Known limitations

**Room writes are not version-guarded.** `RedisRoomRepo` uses `MULTI` for
atomic multi-key writes, but room mutation is read-modify-write with no `WATCH`
or version field. Two instances handling concurrent submissions to the same room
can lose an update. In practice a room's players are usually on one instance and
the window is small, but this is the thing that would need fixing before running
multiple instances under real load — and it is the one place where the
horizontal-scaling story is currently incomplete.

**Test coverage is uneven.** Auth, error handling, request logging, health check
and the OpenAPI router are tested; the scoring engine and round lifecycle are
not. That is backwards — scoring is the part players would notice being wrong.

## Roadmap

|                                                  | Status                      |
| ------------------------------------------------ | --------------------------- |
| Realtime rounds, voting, scoring, seasons, stats | Live in beta                |
| Redis adapter for multi-instance fanout          | Built                       |
| Redis-backed room state with TTL                 | Built                       |
| Optimistic concurrency on room writes            | Not built — see limitations |
| Scoring engine test suite                        | Not built                   |
| Matchmaking beyond room codes                    | Not started                 |

## Documents

|                                          |                                                                                     |
| ---------------------------------------- | ----------------------------------------------------------------------------------- |
| [Realtime model](docs/realtime-model.md) | Authority, room lifecycle, state ownership, the Redis store and its concurrency gap |
| [Decisions](docs/decisions.md)           | Each choice with the alternative rejected and the cost accepted                     |

---

## Repository note

This repository contains architecture documentation only. The implementation is
private.
