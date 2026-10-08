# Backend Learning Roadmap — `chaupar` (multiplayer game server)

> A real-time, server-authoritative game platform in plain JavaScript. Starts as tic-tac-toe over raw
> WebSockets in week 1, grows into Connect Four with minimax/MCTS bots, then Ludo with provably fair
> dice, ratings, tournaments, and a server that holds 2,000 concurrent players on your laptop.
> Total cost to build and run: ₹0.

Prepared 7 Oct 2026. Time budget: 2–3 h weekdays, ~6 h each weekend day (≈ 24 h/week). 19 weeks + 2 buffer.
Working name `chaupar` (the ancient Indian dice game Ludo descends from) — rename freely.

---

## 0. What you are building and why

**The product:** a website where people log in, click "Find match", get paired with a human or a bot,
and play turn-based games (Connect Four, Ludo; chess optional) with timers, reconnection, ratings,
leaderboards and weekend tournaments. The frontend is deliberately thin (vanilla JS + canvas).
All the interesting work — rules engines, matchmaking, bots, timers, fairness, ratings, scaling — is on the server.

**Why it fits what you said:**
- "Simple, needs real maths and good backend concepts, not frontend." Game trees, alpha-beta pruning,
  UCB1 for Monte Carlo Tree Search, Glicko-2 ratings, Swiss pairings, provably fair dice, Little's law
  for capacity. Every one is real maths with a visible effect in the game.
- "Start from basics." Week 1 is a 200-line WebSocket server. Each phase adds one concept and ends
  with something playable.
- "Exciting, real world." You play it with friends on your phones from week 1 (via a free tunnel).
  Ludo is the most-played mobile game in India; a public, open-source server for it does not exist in Node.
- "No money." Free data (none needed), free hosting (Oracle Always Free or your own PC + Cloudflare
  Tunnel), free CI, free AI APIs. The optional domain is the only thing with a price, and it is optional.

**What exists already (honest):** Lichess (chess, Scala, enormous) and small Socket.IO tic-tac-toe
demos. No open-source Node platform with matchmaking, bots, Glicko-2, tournaments and fairness proofs.
Recruiters read a well-built real-time server (authoritative state, reconnection, horizontal scaling,
load-tested) as hard backend experience.

**Where the "AI" is:** the bots are classical AI (search + statistics) — the maths you asked for.
An LLM "coach" with a small retrieval index arrives in Phase 9. It is a supporting feature here, not
the core; if you want deep RAG later, it becomes its own small project on top of what you learn.

---

## 1. Ground rules

1. **Learn → build → write.** Read/watch the phase's "Learn first" list (≤ 5 h), build, then write
   10–20 lines in `docs/learning-log.md` on what surprised you.
2. **Every phase ends playable.** If it is not playable by a friend, the phase is not done.
3. **Commit every session, PR every week, release every phase.** Write PR descriptions as if a
   reviewer exists. Tag `v0.x.0` at each phase.
4. **One new concept per phase.** Ideas that don't fit go to §8 "After v1".
5. **Miss a week → shift a week.** Two buffer weeks exist. Never skip the learn step to catch up.
6. **Server is the source of truth.** Clients send intents ("drop in column 3"); the server validates,
   applies, and broadcasts. Never trust the client — this rule shapes everything you build.
7. **Plain JS, ESM, JSDoc + `// @ts-check`.** Write everything from scratch; no copying from work repos.
8. **AI is a tutor and a reviewer, never the author.** This is a learning project, so the rules for using AI (Claude, ChatGPT, Copilot) are:
   - *Explain first, then you type.* Ask for the concept, the algorithm, a worked example on paper. Then write the code yourself.
   - *Hint ladder when stuck:* (1) the idea in one sentence, (2) pseudo-code, (3) a snippet of at most 10 lines with every line explained. Full code only if you explicitly ask, and then you rewrite it in your own words before committing.
   - *Review mode:* open the PR, ask for a senior-engineer review: bugs, missing tests, "why did you do X?" questions you must answer.
   - *The one-line rule:* if you cannot explain a line you committed, delete it and rewrite it.
   - *Tests first for engines and algorithms:* write the property test, then make it pass.
   - *Boilerplate is allowed* (ESLint config, Compose files, CI YAML) but read it once line by line and note what each part does in the learning log.
   - *Interview check at every phase end:* have the AI act as an interviewer for 20 minutes on what you just built. If you cannot explain it, the phase is not done.
   - *Memory check:* one week after finishing an algorithm (minimax, Glicko-2, the Bayesian detector), re-implement its core from memory in `labs/`. If you cannot, revisit before moving on.

### Weekly rhythm (~24 h)

| Day | Time | What |
|---|---|---|
| Mon | 2–3 h | Learn: docs/video for the phase. Create the week's 3–5 GitHub issues. |
| Tue–Thu | 2–3 h each | Build. Commit at the end of every session. |
| Fri | 2 h | Tests, refactor, README, open the PR. |
| Sat | 6 h | The big build of the week. |
| Sun | 6 h | 2 h maths/DSA lab · 2 h finish & merge · 1 h learning log · 1 h plan next week. |

---

## 2. Stack decisions

| Decision | Choice | Why |
|---|---|---|
| Node | **24 LTS** via nvm (you have 22) | Active LTS until Apr 2028; `node:24-alpine` image. |
| HTTP | **Express 5** for REST (auth, profiles, history) | You know Express 4; learn v5 differences. |
| WebSockets | **`ws`** for Phases 0–1 (learn the protocol), then **Socket.IO 4** from Phase 2 (rooms, acks, connection-state recovery, Redis adapter) | Knowing both is the point; an ADR records what you had to hand-build with `ws`. |
| Database | **MySQL 8.4 LTS in Docker** (host port 3307; your local 8.0 stays on 3306) | Users, games, ratings, tournaments. |
| Migrations | **Knex** + raw SQL where it matters | Learn indexes, transactions, isolation levels by hand. |
| Live state | **Redis 8** (`redis:8` image, host port 6380) | Game state, presence, matchmaking pools, pub/sub, leaderboards, streams, Lua. |
| Jobs | **BullMQ 6** | Bot moves, post-game analysis, tournament rounds, cleanup. |
| Validation | **zod** on every REST body and every WebSocket message | Rule 6. |
| Passwords / tokens | **argon2id** + short-lived JWT for WS auth | Learn why bcrypt → argon2. |
| Logging / metrics | **pino**, **prom-client**, Grafana | Phase 7. |
| Tests | **Jest + supertest** + `fast-check` for property tests on rules engines | Rules engines are perfect for property-based testing. |
| Client | **Vanilla JS + canvas**, served statically; later on Cloudflare Pages (free) | Keep frontend to a minimum. |
| Hosting | **Oracle Always Free ARM VM** (₹0) or **your own PC + Cloudflare Tunnel** for play sessions | No VPS bill. |
| AI APIs | **Groq** or **Gemini** free tiers; **Transformers.js** local embeddings | Phase 9 only. |

---

## 3. Target architecture (v1.0)

```
Browser / phone (canvas client)      Friends via Cloudflare Tunnel / Pages
        │  WebSocket (Socket.IO)              │
        ▼                                     ▼
 ┌──────────────── API instances (×N) ──────────────────┐
 │ Express 5 REST  ·  Socket.IO (Redis adapter)          │
 │ auth · lobby · matchmaking · game session handlers    │
 └───────────┬───────────────────────────┬───────────────┘
             │                           │
             ▼                           ▼
   Redis 8                            MySQL 8.4
   game state (hash+version)          users, games (move log), ratings history,
   presence (TTL keys)                tournaments, pairings, analysis
   matchmaking pool (ZSET + Lua)
   timers (ZSET of deadlines)
   leaderboards (ZSET)
   move stream (Redis Streams)
   pub/sub for fan-out
             │
             ▼
   Workers (BullMQ): bot moves (minimax / MCTS) · post-game analysis ·
                     tournament rounds · rating recompute · archive to S3
             │
   AWS (Phase 8): S3 replay archive · SES email · Lambda+EventBridge snapshots · SQS adapter
   Cloudflare (Phase 6/8): Tunnel · Pages · Turnstile · WAF · Durable Objects experiment
   Observability (Phase 7): pino → files · prom-client → Prometheus → Grafana
```

Core abstraction — a pure, synchronous **game engine interface** every game implements:

```js
// packages/engine/<game>/index.js
export const TicTacToe = {
  id: 'tictactoe', minPlayers: 2, maxPlayers: 2,
  initialState(players, seed) {},          // -> state (plain JSON)
  legalMoves(state, playerId) {},          // -> [move]
  applyMove(state, playerId, move) {},     // -> new state (no mutation) or throws
  status(state) {},                        // -> { finished, winner|draw, nextPlayer }
  view(state, playerId) {},                // -> what THIS player may see (hidden info)
};
```

Because engines are pure functions over JSON, the same code runs on the server, in bots, in tests,
and in replay — and property-based tests can hammer them with random legal move sequences.

Data model (MySQL):

| Table | Key columns |
|---|---|
| `users` | id, username, email, password_hash, created_at |
| `games` | id (ULID), game_type, status, seed, started_at, ended_at, winner_id, moves (JSON), final_state (JSON) |
| `game_players` | game_id, user_id or bot_level, seat, rating_before, rating_after |
| `ratings` | user_id, game_type, rating, rd, volatility (Glicko-2), updated_at |
| `rating_history` | user_id, game_type, game_id, rating, rd, created_at |
| `tournaments` | id, game_type, format (swiss/arena), starts_at, rounds, status |
| `tournament_players`, `pairings` | tournament_id, round, white_id/black_id (seat), result |
| `analysis` | game_id, move_no, eval, best_move, blunder (bool) |
| `flags` | user_id, kind (engine_match, rate_anomaly), score, details, created_at |

---

## 4. Phase plan

| Phase | Weeks | You will build | Playable result | Release |
|---|---|---|---|---|
| 0 | 1 | Raw WebSocket server + tic-tac-toe | Play a friend through a tunnel | v0.1.0 |
| 1 | 2–3 | Engine interface, Connect Four (bitboards), MySQL, auth, Docker Compose | Login, play, replay saved games | v0.2.0 |
| 2 | 4–5 | Redis: live state, presence, matchmaking, timers, reconnection; move to Socket.IO | "Find match", refresh mid-game and resume, lose on time | v0.3.0 |
| 3 | 6–7 | Bots: minimax/alpha-beta + MCTS in BullMQ workers; post-game analysis | Beat (or not) three bot levels; see your blunders | v0.4.0 |
| 4 | 8–9 | Ludo: 4 players, provably fair dice, spectators, chat, Ludo bot | 4 friends on phones; verify the dice yourself | v0.5.0 |
| 5 | 10–11 | Glicko-2 ratings, leaderboards, Swiss + arena tournaments | A weekend tournament with ratings | v0.6.0 |
| 6 | 12–13 | Production Docker, GitHub Actions CD, ₹0 hosting, Pages, Turnstile | A public URL that stays up | v0.7.0 |
| 7 | 14–15 | Scale: N instances, Redis adapter, load test 2,000 players, metrics, event sourcing | Grafana proving 2,000 concurrent bots | v0.8.0 |
| 8 | 16–17 | AWS (S3, SES, Lambda, SQS) + Cloudflare (Durable Objects, WAF) | Emails, archives, one room at the edge | v0.9.0 |
| 9 | 18 | AI coach (LLM + small retrieval index) + anti-cheat statistics | "Why did I lose?" explained | v0.10.0 |
| 10 | 19 | Hardening, docs, benchmarks, launch | Strangers can play | v1.0.0 |
| — | 20–21 | Buffer | | |

---

### Phase 0 — Week 1: WebSockets from scratch + tic-tac-toe

**Goal:** understand the WebSocket protocol and server-authoritative state with the simplest possible game.

**Learn first (~3 h)**
- [MDN: Writing WebSocket servers](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_servers) (frames, masking, ping/pong) and the [`ws` README](https://github.com/websockets/ws).
- [Node.js event loop, timers, nextTick](https://nodejs.org/learn/asynchronous-work/event-loop-timers-and-nexttick).
- Skim [Gaffer on Games — What every programmer needs to know about game networking](https://gafferongames.com/post/what_every_programmer_needs_to_know_about_game_networking/) for the idea of an authoritative server (ignore the FPS parts).

**Build**
- [ ] Repo `chaupar` (MIT), ESM, ESLint + Prettier + husky + commitlint, GitHub Actions running lint + tests on every push (CI from day one; no Docker yet).
- [ ] `ws` server with a JSON message envelope `{ t: 'type', seq, payload }`, zod-validated; heartbeats with ping/pong and dead-connection cleanup.
- [ ] Rooms by 6-letter code (in memory for now); tic-tac-toe engine implementing the interface in §3; server validates every move, broadcasts state, rejects out-of-turn moves.
- [ ] Vanilla JS client on one HTML page (3×3 grid is fine). Serve it with Express.
- [ ] Unit tests for the engine; a WebSocket integration test that plays a full game with two clients.
- [ ] Play a friend on their phone via `cloudflared tunnel --url http://localhost:3000` (no account needed).

**Maths / DSA lab (Sun):** finite state machines (lobby → playing → finished) and win detection as line checks; Big-O refresher. Write `docs/protocol.md` describing every message type — your first protocol design.

**Playable:** two phones, one game. Tag `v0.1.0`.

---

### Phase 1 — Weeks 2–3: Engine interface, Connect Four, MySQL, auth, Docker Compose

**Goal:** a second game drops in without touching the server; games persist; players have accounts; the dev environment is reproducible.

**Learn first (~5 h)**
- [Pascal Pons — Solving Connect 4](http://blog.gamesolver.org/) parts 1–3 (bitboards, win detection with bit shifts). Then the [Chess Programming Wiki — Bitboards](https://www.chessprogramming.org/Bitboards) page for the general idea.
- Docker basics: [Docker Node.js guide](https://docs.docker.com/guides/nodejs/) + Compose intro. (TechWorld with Nana's 3-hour "Docker Tutorial for Beginners" on YouTube if you prefer video.)
- [Knex migrations](https://knexjs.org/guide/migrations.html); MySQL indexes, unique constraints, JSON columns, `EXPLAIN`.
- Password hashing: why argon2id ([OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)).

**Build — Week 2**
- [ ] Install Docker Engine + Compose plugin; nvm → Node 24. `compose.yaml` with `mysql:8.4` (port 3307, volume, healthcheck) and the API (bind mount + nodemon).
- [ ] Extract `packages/engine` (npm workspaces: `apps/server`, `packages/engine`, `apps/client`). Tic-tac-toe moves there.
- [ ] **Connect Four with bitboards**: two 64-bit masks (`BigInt`), win check via shifts, legal moves via column heights. Property-based tests with `fast-check` (random legal games never produce an illegal state; a win is always detected).
- [ ] Knex + migrations: `users`, `games`, `game_players`. Save every finished game with its move list.

**Build — Week 3**
- [ ] Auth: register/login (argon2id), short-lived JWT; WebSocket handshake authenticates with the token; rate-limit login attempts (in memory for now).
- [ ] Game history REST endpoints with keyset pagination; **replay**: rebuild any game by re-applying its move list through the engine (this is the seed of event sourcing).
- [ ] Client: login, game picker, Connect Four board, replay viewer with step buttons.
- [ ] CI now runs tests against MySQL via GitHub Actions `services:`.

**Maths / DSA lab:** bit manipulation (masks, shifts, popcount), **Zobrist hashing** (random 64-bit keys per cell/player, XOR to hash a position — you will need it in Phase 3), hash maps vs B-tree indexes.

**Playable:** login, play Connect Four with a friend, watch the replay. Tag `v0.2.0`.

---

### Phase 2 — Weeks 4–5: Redis — live state, presence, matchmaking, timers, reconnection

**Goal:** game state leaves process memory so the server can restart or scale; players get matched automatically; timers are enforced server-side; a refresh does not lose the game.

**Learn first (~5 h)**
- [Redis University RU102JS](https://university.redis.com/courses/ru102js/) (free) — data types, sorted sets, Lua, pub/sub.
- Redis patterns: [sorted-set leaderboards](https://redis.io/docs/latest/develop/), `SET NX PX` locks, keyspace TTLs for presence.
- [Socket.IO docs](https://socket.io/docs/v4/): rooms, acknowledgements, middleware auth, [connection state recovery](https://socket.io/docs/v4/connection-state-recovery).

**Build — Week 4**
- [ ] `redis:8` in Compose (port 6380). Game state in a Redis hash with a `version` field; moves applied with **compare-and-set** (WATCH/MULTI or a Lua script) so two instances can never double-apply.
- [ ] Presence: `presence:<userId>` keys with TTL refreshed by heartbeats; lobby shows who is online.
- [ ] **Matchmaking pool** as a sorted set keyed by rating (everyone starts at 1500); a Lua script atomically pops two players within a widening rating window; the window widens with wait time.
- [ ] **Server-side turn timers** that survive restarts: a sorted set of `deadline → gameId`, a ticker that pops expired deadlines and forfeits/auto-moves. (Not `setTimeout` — understand why.)

**Build — Week 5**
- [ ] Migrate transport to **Socket.IO** (keep the `ws` version tagged `v0.2.0` for reference): rooms per game, acks for moves, auth middleware, connection-state recovery. ADR: what `ws` made you build by hand, what Socket.IO gives you, what it costs.
- [ ] **Reconnection**: resume token, server sends full state + last `seq`, client discards stale messages. Test by killing the tab mid-game and by restarting the server mid-game.
- [ ] Rate limiting per socket (token bucket in Redis), spectator mode (read-only join).

**Maths / DSA lab:** sorted sets and **skip lists**; **timer wheels** vs sorted deadlines; optimistic concurrency (compare-and-set) vs locks; a 1-page note on matchmaking as a queue: arrival rate, wait time, window widening.

**Playable:** open two browsers, click "Find match", get paired; refresh mid-game and continue; let the clock run out and lose. Tag `v0.3.0`.

---

### Phase 3 — Weeks 6–7: Bots — minimax, alpha-beta, MCTS, in BullMQ workers

**Goal:** real game AI, computed off the event loop, with difficulty levels and post-game analysis.

**Learn first (~6 h)**
- [Sebastian Lague — Algorithms Explained: minimax and alpha-beta pruning](https://www.youtube.com/watch?v=l-hh51ncgDI) (YouTube, 11 min), then Chess Programming Wiki pages: [Minimax](https://www.chessprogramming.org/Minimax), [Alpha-Beta](https://www.chessprogramming.org/Alpha-Beta), [Iterative Deepening](https://www.chessprogramming.org/Iterative_Deepening), [Transposition Table](https://www.chessprogramming.org/Transposition_Table).
- [Pascal Pons — Solving Connect 4](http://blog.gamesolver.org/) parts 4–8 (move ordering, transposition table, bitboard tricks).
- [MCTS beginner's guide (int8.io)](https://int8.io/monte-carlo-tree-search-beginners-guide/) — UCB1, selection/expansion/simulation/backpropagation.
- [BullMQ docs](https://docs.bullmq.io/): workers, concurrency, priorities, retries, `QueueEvents`; Node `worker_threads` vs separate processes.

**Build — Week 6**
- [ ] `packages/bots`: **minimax with alpha-beta**, iterative deepening with a time budget, move ordering (centre columns first), **transposition table** keyed by Zobrist hash. Three levels = three time budgets (50 ms / 500 ms / 3 s).
- [ ] Bots run in a **separate worker process via BullMQ** (`bot-move` queue); the game handler enqueues, the worker computes and applies the move through the same engine; the API never blocks. Measure event-loop lag with and without this (the key Node lesson).
- [ ] "Play vs bot" in matchmaking when no human appears within N seconds.

**Build — Week 7**
- [ ] **MCTS** bot with UCB1; a `bot-vs-bot` job that runs 1,000 games minimax-vs-MCTS at equal time budgets and stores win rates — put the table in the README.
- [ ] **Post-game analysis** job: evaluate every move, mark blunders (eval drop > threshold), store in `analysis`; the replay viewer shows an eval bar and blunder markers.
- [ ] Graceful shutdown for workers (`SIGTERM` → finish current job → exit); retries + dead-letter queue for crashed analyses; bull-board at `/admin/queues`.

**Maths / DSA lab:** game trees and why alpha-beta prunes (best case O(b^(d/2))); the UCB1 formula and exploration vs exploitation; heaps (priority queue for move ordering); hash tables under load (transposition table replacement schemes).

**Playable:** three bot levels; a blunder-marked replay of your loss. Tag `v0.4.0`. **Write-up #1:** "Minimax vs MCTS at Connect Four: 1,000 games, with numbers".

---

### Phase 4 — Weeks 8–9: Ludo — the flagship

**Goal:** a four-player game with hidden randomness done honestly, spectators, chat, and a probability-based bot.

**Learn first (~4 h)**
- Provably fair randomness: commit-reveal with a hashed server seed, a client seed and a nonce — read any two "provably fair" explainers and the Node `crypto` docs (`randomBytes`, `createHmac`).
- Standard Ludo rules (write `docs/ludo-rules.md` yourself first; rule ambiguities are where bugs live).
- Probability basics: distributions of one die, "at least one six in N turns", expected value.

**Build — Week 8**
- [ ] **Ludo engine**: board as a graph (52 shared squares + 4 home columns), tokens, safe squares, captures, extra turn on six, three-sixes forfeit, exact-roll finish. Property tests: any legal sequence keeps every token on a valid square; a finished game has exactly one winner order.
- [ ] **Provably fair dice**: at game start publish `sha256(serverSeed)`; each roll = `HMAC(serverSeed, clientSeed:nonce)` → 1–6; at game end reveal `serverSeed` and expose a verify endpoint + a "verify this game" button. A player can prove the server did not cheat.
- [ ] 4-player rooms, seat assignment, turn timers with auto-move, disconnect handling (after 2 missed turns a bot takes the seat).

**Build — Week 9**
- [ ] **Ludo bot**: expected-value heuristic (capture probability, risk of being captured, progress, home-stretch priority). Compare against a random bot over 1,000 games.
- [ ] Spectators with fan-out through Redis pub/sub; in-game chat with rate limiting and a **trie-based profanity filter**; emotes.
- [ ] Client: Ludo board on canvas (keep it plain), mobile layout, sounds optional.
- [ ] **Shakuni mode v1** (the USP, see Appendix A): hidden role assigned by the server, 3 wish tokens, one Sabha vote per game, sealed wish log revealed with the server seed at game end. Balance N and the vote rules with bot-vs-bot simulations from Phase 3.

**Maths / DSA lab:** probability and expected value (do the dice maths by hand, then verify with 1M simulated rolls); graphs (board path, BFS for "distance to home"); tries; HMAC and commitment schemes.

**Playable:** four friends on phones, one of them secretly Shakuni, and a dice-fairness proof they can check. Tag `v0.5.0`. **Write-up #2:** "Provably fair dice for Ludo in 60 lines of Node".

---

### Phase 5 — Weeks 10–11: Ratings, leaderboards, tournaments

**Goal:** real competitive infrastructure: a statistically sound rating system, fast leaderboards, and tournaments that run themselves.

**Learn first (~5 h)**
- [Glicko-2 paper (Mark Glickman)](http://www.glicko.net/glicko/glicko2.pdf) — 6 pages; it has a worked example you will turn into a test.
- MySQL transactions and isolation levels, `SELECT … FOR UPDATE`, deadlocks (Hussein Nasser's [channel](https://www.youtube.com/@hnasr) has short videos on each).
- Swiss-system pairing rules (Wikipedia is enough); note that perfect pairing is maximum-weight matching (Edmonds' blossom) — you will implement a constrained greedy version.

**Build — Week 10**
- [ ] **Glicko-2** in `packages/rating` with the paper's example as a unit test; per-game-type ratings; rating updates inside a MySQL transaction (`FOR UPDATE` on both players, retry on deadlock); `rating_history`.
- [ ] **Leaderboards** in Redis sorted sets: global and weekly (key per ISO week; a repeatable job rotates and persists the weekly top 100 to MySQL). Rank, neighbours, pagination.
- [ ] Matchmaking now uses Glicko-2 rating and rating deviation for the window.

**Build — Week 11**
- [ ] **Tournaments**: Swiss (fixed rounds, constrained greedy pairing with no-rematch rule, byes) and Arena (continuous pairing for a time window, lichess-style). Rounds are scheduled with BullMQ delayed jobs; a tournament lobby socket room shows standings live.
- [ ] Tie-breaks (Buchholz), standings, result reporting from game end events.
- [ ] Email reminders stubbed to the log (SES arrives in Phase 8).

**Maths / DSA lab:** the Glicko-2 derivation at a high level (why rating deviation shrinks with games); sorted-set ranking in O(log n); matching in graphs (why greedy Swiss pairing can fail and what blossom fixes); Buchholz as a weighted sum.

**Playable:** a weekend Ludo arena with friends and rating changes you can explain. Tag `v0.6.0`. **Write-up #3:** "Implementing Glicko-2 from the paper in JavaScript".

---

### Phase 6 — Weeks 12–13: Production Docker, CI/CD, hosting for ₹0

**Goal:** a public URL that stays up, deployed automatically from a release tag, without a hosting bill.

**Learn first (~5 h)**
- Multi-stage Dockerfiles, non-root user, `.dockerignore`, `HEALTHCHECK`: [Docker Node.js + GitHub Actions guide](https://docs.docker.com/guides/nodejs/configure-github-actions/).
- [GitHub Actions: publishing Docker images](https://docs.github.com/en/actions/publishing-packages/publishing-docker-images) to GHCR.
- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/), [Cloudflare Pages](https://developers.cloudflare.com/pages/), [Turnstile](https://developers.cloudflare.com/turnstile/).
- [The Twelve-Factor App](https://12factor.net/).

**Build — Week 12**
- [ ] Production `Dockerfile` (multi-stage, prod deps, non-root, healthcheck) — one image, two commands (server, worker). `compose.prod.yaml`: server, worker, mysql, redis, named volumes, restart policies, log rotation.
- [ ] CI: PR → lint + tests (MySQL + Redis services). Push to `main` → build multi-arch image (amd64 + arm64, needed for Oracle ARM) → GHCR with `sha`/`latest`/semver tags.
- [ ] CD: on release tag → SSH → `docker compose pull && up -d` → smoke test. Secrets in a GitHub Environment.

**Build — Week 13**
- [ ] Hosting, pick one: **Oracle Always Free** ARM VM (₹0; retry regions if "out of capacity") — Ubuntu, Docker, `ufw` (SSH only), fail2ban. **Or** run on your own PC and expose it with a **named Cloudflare Tunnel** only when you want to play; document both in `docs/deploy.md`.
- [ ] Static client on **Cloudflare Pages** (free, `*.pages.dev`), API behind the Tunnel; CORS and WebSocket origin checks.
- [ ] **Turnstile** on signup; nightly `mysqldump` + a restore drill you actually run.

**Maths / DSA lab:** graphs again — Compose `depends_on` and CI job graphs are DAGs; implement topological sort.

**Playable:** a link anyone can open any time. Tag `v0.7.0`.

---

### Phase 7 — Weeks 14–15: Scale, load, observability, event sourcing

**Goal:** prove the design holds thousands of concurrent players and know *why* when it doesn't.

**Learn first (~5 h)**
- [Socket.IO Redis adapter](https://socket.io/docs/v4/redis-adapter/) and [multiple nodes](https://socket.io/docs/v4/using-multiple-nodes/) (sticky sessions vs recovery).
- [Artillery docs](https://www.artillery.io/docs) (Socket.IO engine) — or write your own bot clients, which you already have from Phase 3.
- Node performance: `node --cpu-prof`, event-loop lag, [Clinic.js](https://clinicjs.org/).
- [Martin Fowler — Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html); Redis Streams + consumer groups (RU202 first module).

**Build — Week 14**
- [ ] Run 3 server instances behind nginx with the Redis adapter; verify a game whose two players are on different instances works; verify reconnection to a different instance.
- [ ] **Load test**: 2,000 simulated players (bot clients) playing Connect Four continuously on your laptop. Record msgs/sec, p95 move latency, event-loop lag, Redis ops/sec, memory per connection.
- [ ] Fix the top 3 bottlenecks you find (expect: JSON serialisation, Redis round-trips → pipelining, per-move DB writes → batching).

**Build — Week 15**
- [ ] Metrics with prom-client: connected sockets, moves/sec, matchmaking wait p95, bot queue depth, event-loop lag. Prometheus + Grafana in Compose; commit the dashboard JSON.
- [ ] **Event sourcing**: every move appended to a Redis Stream per game; MySQL `games.moves` becomes a projection; a consumer group builds stats (openings, average game length) and can rebuild any game. Document snapshots vs replay.
- [ ] Graceful rolling deploys: drain (stop accepting new games, finish or hand off running ones — state is in Redis so this is possible) — then deploy with zero dropped games.

**Maths / DSA lab:** **Little's law** (concurrent players = arrival rate × session length — size your matchmaking and capacity with it); ring buffers and backpressure; percentiles (how p95 is computed; t-digest idea).

**Playable:** 2,000 bots and a Grafana dashboard. Tag `v0.8.0`. **Write-up #4:** "2,000 concurrent Ludo players on a laptop: what broke and what fixed it".

---

### Phase 8 — Weeks 16–17: AWS and Cloudflare, both at ₹0

**Goal:** use the cloud services every job posting lists, each for one real feature, without a bill.

> **Open the AWS account now, not earlier.** New accounts get a Free plan with $100 credit (+$100 for activities) that **auto-closes after 6 months**. Opening it in week 16 keeps it alive well past launch. Set a Budget alert at $1 on day one; never upgrade to the Paid plan for this project.

**Learn first (~6 h)**
- [AWS Skill Builder](https://skillbuilder.aws/) "Cloud Practitioner Essentials" (free): IAM, S3, SQS, Lambda modules. [SDK for JavaScript v3 guide](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/).
- [Cloudflare Durable Objects](https://developers.cloudflare.com/durable-objects/) (on the free plan) and the WebSocket hibernation API.

**Build — Week 16 (AWS)**
- [ ] IAM: one user per feature, hand-written least-privilege policies, MFA on root, Budget alert.
- [ ] **S3**: nightly job archives finished games older than 30 days (gzip NDJSON per day), MySQL keeps metadata + `s3_key`; replay viewer fetches via presigned GET. Lifecycle rule → Glacier IR.
- [ ] **SES**: signup verification and tournament reminders (sandbox + verified addresses is fine).
- [ ] **Lambda + EventBridge Scheduler**: weekly leaderboard snapshot and rating-deviation decay as a scheduled Lambda; CloudWatch logs, metric filter, alarm.
- [ ] **SQS adapter** behind a `QueueAdapter` interface for the analysis queue; ADR: BullMQ vs SQS.

**Build — Week 17 (Cloudflare)**
- [ ] **Durable Objects experiment**: implement one Connect Four room as a Durable Object (WebSocket hibernation, SQLite storage). Compare latency, cost model and complexity with your Redis design in an ADR. Keep whichever you prefer for "quick play" rooms.
- [ ] WAF rate-limit rules on auth routes; KV for feature flags; a Workers cron that pings `/health`.

**Maths / DSA lab:** **consistent hashing** (route game rooms to instances; virtual nodes) and rendezvous hashing; why Durable Objects are "one object = one serialised actor".

**Playable:** same game, now with emails, archives and one room at the edge. Tag `v0.9.0`.

---

### Phase 9 — Week 18: AI coach and anti-cheat

**Goal:** turn the analysis data into something a player learns from, and detect bad actors with statistics.

**Learn first (~3 h)**
- [Hack your own RAG stack in under an hour](https://hackernoon.com/hack-your-own-rag-stack-in-under-an-hour) (JS, Transformers.js) — just the retrieval mechanics.
- z-scores and hypothesis testing basics (any statistics primer; Khan Academy is fine).

**Build**
- [ ] **Coach**: after a game, take the three worst blunders from `analysis`, retrieve 2–3 relevant passages from a small index of strategy notes (Connect Four strategy articles + Ludo tactics you write yourself; embedded with Transformers.js, stored in Redis 8 vector index), and ask Groq/Gemini (free tier) to explain each mistake in plain language with the engine's numbers. Stream the answer over the existing socket. Cache per position hash.
- [ ] A 10-question eval of retrieval (hit@3) so the index is measured, not assumed.
- [ ] **Anti-cheat**: engine-match rate (fraction of a player's moves equal to the bot's best move) compared with peers at the same rating as a z-score; move-time anomalies; multi-account heuristics. Flags land in an admin queue with the evidence.

**Maths / DSA lab:** cosine similarity and top-K with a heap; z-scores; false-positive rates and why thresholds need base rates.

**Playable:** "Why did I lose?" answered with numbers and words. Tag `v0.10.0`.

---

### Phase 10 — Week 19: Hardening, docs, launch v1.0

**Build**
- [ ] Security pass: origin checks on WebSocket upgrade, zod on every message (already), `helmet`, auth rate limits, JWT rotation, gitleaks in CI, CodeQL, `SECURITY.md`, dependency audit.
- [ ] README final: 60-second GIF, architecture diagram, how the dice proof works, bot win-rate table, load-test numbers, Glicko-2 notes, one-command `docker compose up`, OpenAPI for REST (`swagger-jsdoc`), ADR index.
- [ ] `v1.0.0` release with CHANGELOG. Share on r/node, r/gamedev, dev.to, LinkedIn, and with your friends' WhatsApp groups (they are the real users).

**Maths / DSA lab:** mock-interview day — explain where the project uses hash maps, heaps, tries, graphs, sorted sets, bit tricks, probability, statistics.

**Playable:** strangers can sign up and play. Tag `v1.0.0`. **Write-up #5:** the launch post.

---

## 5. Maths & DSA thread at a glance

| Week | Concept | Where the game uses it |
|---|---|---|
| 1 | State machines, Big-O | Game lifecycle, protocol |
| 2 | Bit manipulation, bitboards | Connect Four engine |
| 3 | Zobrist hashing, hash maps | Position keys, transposition table |
| 4 | Sorted sets / skip lists, timer wheels | Matchmaking pool, server timers |
| 5 | Optimistic concurrency, queues | Compare-and-set state, matchmaking as a queue |
| 6 | Game trees, minimax, alpha-beta, heaps | Bots, move ordering |
| 7 | UCB1 / MCTS, hash table replacement | MCTS bot, transposition table |
| 8 | Probability, expected value, HMAC commitments | Dice, fairness proof |
| 9 | Graphs (board path, BFS), tries | Ludo board, profanity filter, Ludo bot |
| 10 | Glicko-2 (statistics), O(log n) ranking | Ratings, leaderboards |
| 11 | Graph matching, weighted tie-breaks | Swiss pairing, Buchholz |
| 12–13 | DAGs, topological sort | Compose / CI dependencies |
| 14 | Little's law, percentiles | Capacity planning, p95 |
| 15 | Ring buffers, backpressure, event sourcing | Load, move streams |
| 16–17 | Consistent / rendezvous hashing, actors | Room routing, Durable Objects |
| 18 | Cosine similarity, z-scores | Coach retrieval, anti-cheat |
| 19 | Review | Interview prep |

---

## 6. GitHub profile plan (alongside)

- Profile README: who you are, "building chaupar — a real-time multiplayer game server", stack badges, links to the five write-ups. Pin `chaupar`.
- Work like a team of one: issues, milestones per phase, PRs with real descriptions, Conventional Commits, releases with changelogs, CI + CodeQL badges, ADRs in `docs/adr/`.
- The README's numbers are the hook: bot win rates, 2,000-player load test, dice proof. Numbers and diagrams are what recruiters remember.
- Natural open-source contributions after Phase 3 and Phase 7: Socket.IO, BullMQ, or `ws` docs/issues you now understand.

---

## 7. Accounts and costs

| When | Account | Cost |
|---|---|---|
| Week 1 | GitHub (have), Cloudflare free (quick tunnels need no account) | ₹0 |
| Week 12 | Oracle Cloud Always Free **or** nothing (own PC + named Tunnel) | ₹0 |
| Week 12 | Cloudflare Pages, Turnstile, named Tunnel (needs a free Cloudflare account; a domain is optional — `*.pages.dev` and `*.trycloudflare.com` work) | ₹0 (domain optional ≈ ₹800/yr) |
| **Week 16** | **AWS Free plan** (6-month auto-close; Budget alert at $1) | ₹0 |
| Week 18 | Groq / Google AI Studio free tiers | ₹0 |

---

## 8. After v1.0 (one at a time)

- **Chess** via `chess.js` for rules while you focus on clocks, Stockfish-in-Docker analysis, puzzles mined from blunders.
- **Mobile**: a PWA with push notifications for "your turn" in correspondence games.
- **Kafka** (Redpanda) for the move stream; **PostgreSQL** port; **TypeScript** migration module by module; **Kubernetes** (k3s) with a Helm chart; **Terraform** for the AWS pieces.
- A real **RAG** project: index Lichess-style opening explorers or strategy books and build a coach that is the core, not a feature.

---

## 9. Sources

- WebSockets & Node: [MDN WebSocket servers](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_servers); [`ws`](https://github.com/websockets/ws); [Node event loop](https://nodejs.org/learn/asynchronous-work/event-loop-timers-and-nexttick); [Socket.IO docs](https://socket.io/docs/v4/); [Gaffer on Games](https://gafferongames.com/).
- Game AI: [Chess Programming Wiki](https://www.chessprogramming.org/); [Solving Connect 4 (Pascal Pons)](http://blog.gamesolver.org/); [MCTS beginner's guide](https://int8.io/monte-carlo-tree-search-beginners-guide/); [Sebastian Lague minimax video](https://www.youtube.com/watch?v=l-hh51ncgDI).
- Ratings & tournaments: [Glicko-2 paper](http://www.glicko.net/glicko/glicko2.pdf); [Lichess source (lila)](https://github.com/lichess-org/lila) for how a real platform structures arenas.
- Redis / queues: [Redis University](https://university.redis.io/); [Redis docs](https://redis.io/docs/latest/develop/); [BullMQ](https://docs.bullmq.io/); [bull-board](https://github.com/felixmosh/bull-board).
- Docker / CI / hosting: [Docker Node guide](https://docs.docker.com/guides/nodejs/); [GitHub Actions Docker publish](https://docs.github.com/en/actions/publishing-packages/publishing-docker-images); [Oracle Always Free](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm); [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/); [Pages](https://developers.cloudflare.com/pages/); [Durable Objects](https://developers.cloudflare.com/durable-objects/); [Workers pricing/free limits](https://developers.cloudflare.com/workers/platform/pricing/).
- AWS: [Free Tier](https://aws.amazon.com/free/) and [FAQ](https://aws.amazon.com/free/free-tier-faqs/); [Skill Builder](https://skillbuilder.aws/); [SDK v3 guide](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/).
- Scale & observability: [Artillery](https://www.artillery.io/docs); [Clinic.js](https://clinicjs.org/); [prom-client](https://github.com/siimon/prom-client); [Event Sourcing (Fowler)](https://martinfowler.com/eaaDev/EventSourcing.html).
- AI / stats: [Hack your own RAG stack](https://hackernoon.com/hack-your-own-rag-stack-in-under-an-hour); [Transformers.js](https://huggingface.co/docs/transformers.js); [Groq limits](https://console.groq.com/docs/rate-limits).
- DSA: [ThePrimeagen — The Last Algorithms Course You'll Need (free)](https://master.dev/courses/algorithms/); [System Design Primer](https://github.com/donnemartin/system-design-primer); [Hussein Nasser](https://www.youtube.com/@hnasr).

---

## Appendix A — Shakuni mode (the USP), design draft v1

**Premise (from the Sabha Parva):** Shakuni's dice obeyed his wish. In this mode one of the four players is secretly Shakuni
and can, a limited number of times, decide what the dice show. The others must work out who he is before he wins.
Social deduction (the Among Us idea) layered on Ludo, with the maths you already build for fair dice and anti-cheat.

**Roles (v1):** 4 players, 1 Shakuni + 3 Pandavas. Server assigns secretly at game start.
**Wish tokens:** Shakuni has N = 3. Before his roll he may spend one and pick the face. Nobody can tell a wished roll
from a fair one during the game. All other rolls, his included, come from the provably fair HMAC chain.
**Sabha vote:** any player may call a Sabha once per game. Everyone votes on who is Shakuni (30-second timer).
- Majority correct → Shakuni is exposed: loses remaining wishes, his token furthest ahead returns to base. Game continues.
- Wrong or no majority → the caller loses the next turn and Shakuni gains one wish token.
**Win conditions:** a Pandava wins by finishing first (and the Pandavas also share a win if Shakuni was exposed before the
end). Shakuni wins by finishing first, or by being unexposed when the game ends while in the top two.
**Reveal:** at game end the server publishes the server seed and the sealed wish log (turn numbers where a wish was used),
so everyone can verify: exactly N wishes, all other rolls fair. Same commit-reveal as the base game, one extra sealed field.
**Deduction aids (v2):** a "suspicion meter" computed by the server from roll statistics (how unlikely each player's sequence
is under fair dice, Bayesian posterior over who is Shakuni). This is the Phase 9 anti-cheat maths turned into gameplay.
**Extra roles (v2, only after v1 is balanced):** Duryodhana (knows Shakuni, shares his win), Vidura (may inspect one roll
per game and learn whether it was wished), Krishna (one re-roll for any player).
**Balance levers (so neither side has it easy — the Among Us principle):**
- *Shakuni is alone against three.* In a plain race 1 v 3 he loses most games; the wishes only compensate. The simulation
  decides how many wishes make it close, not a guess.
- *Every wish is evidence.* Roll history is public. A wished six at exactly the capture distance looks like luck once and
  like Shakuni twice. The v2 suspicion meter makes this maths visible; in v1 humans do it by eye.
- *Wish cooldown:* no two wishes on consecutive turns (Among Us kill cooldown). Shakuni must plan, not spam.
- *Whisper:* when a wish is used there is a chance p (start at 25 %) that the Sabha hears "a wish was used this round",
  without saying by whom (Among Us "body reported"). Tension rises without giving the answer.
- *Exposure hurts:* exposed Shakuni loses all wishes, his lead token goes home, and his remaining rolls are marked so the
  three can coordinate captures against him.
- *False accusation hurts too:* a wrong Sabha costs the caller a turn and hands Shakuni a wish, and each player may call
  only once. The Pandavas must gather evidence and agree, not vote on a hunch.
- *Shakuni's win is strict:* he must finish first (unexposed or not). "Survive until the end" alone is not a win in v1.
- *Turn cap:* if nobody finishes within T turns, the side that is behind on the simulation table wins the tie. The cap
  stops stalling by either side.
- *Equal seats:* seat order and start positions are symmetric so the only asymmetry is the role.
- *Target numbers:* Shakuni wins 40–50 % of balanced bot games, each Pandava about 17–20 %, and a Sabha is called in
  most games (if it is never called, the mode is boring; if it is always right, Shakuni is too weak). Tune N, cooldown,
  p and T on the simulation table, then re-check with ten human games.

**Bots:** Shakuni bot uses expected value plus deception (does not always wish a six; saves wishes for captures and the
home stretch). Pandava bots accuse when the posterior for one player exceeds a threshold.
**Balance:** run 1,000 bot games per rule set; target a Shakuni win rate between 40 and 50 percent. Put the table in the README.
**Tone:** the Mahabharata is sacred to many players. Keep the depiction respectful, keep the humour on Shakuni's cunning,
never on deities, and never attach real money to this mode.
**Name:** "Shakuni" has 8,100 searches/month in India (SEO difficulty 33) versus 2,900 for "chaupar", and it describes the
game rather than a Jaipur metro station. shakuni.in, shakuni.dev, shakuni.app, shakunidice.com and playshakuni.com were
available on 7 Oct 2026; shakuni.com and shakuni.games are taken. Either keep `chaupar` as the platform and "Shakuni" as
the flagship mode, or brand the whole product "Shakuni". Both are fine; the data favours the second for recall and search.

---

## Appendix B — Bot ladder and Shakuni-mode AI

Bots are not optional: they fill empty seats, let one person play alone, and run the simulations that balance every rule.
Build them as a ladder; each rung is a difficulty level and a measurable opponent for the next.

### B.1 Ludo bot ladder (Phase 4, extends the Connect Four bots of Phase 3)

| Level | Decision method | Maths / DSA | Difficulty levers |
|---|---|---|---|
| 0 Random | Any legal move | — | Baseline for measuring everything else |
| 1 Greedy | Highest immediate gain (capture > enter > home > progress) | Scoring function, sorting | Add noise: pick a random move with probability ε |
| 2 Expected value | Score each move by progress + capture bonus − risk, where risk = probability an opponent piece can hit the landing square next turn (count pieces 1–6 behind, dice distribution; two dice chances if a six repeats) | Probability, expected value, hash map of square → pieces | Weight tuning; ε noise; ignore risk at lower levels |
| 3 Expectimax | 1–2 ply search: my move → chance node over 6 dice faces → each opponent's best reply by level-2 scoring | Expectiminimax (minimax with chance nodes), memoisation keyed by Zobrist hash | Depth, time budget |
| 4 Monte Carlo | For each legal move, play N random-ish playouts with level-2 bots to the end; pick the move with the best win rate; later MCTS with UCB1 and chance nodes | Monte Carlo simulation, UCB1, law of large numbers | N playouts / time budget (50 ms, 500 ms, 3 s) |

Every bot is a pure function `(state, playerId, rng) → move` using a **seeded PRNG**, so a game is reproducible in tests.
Run a round-robin of 1,000 games per pair to get a win-rate matrix; rate the bots with Glicko-2 (Phase 5) and name the
difficulty levels by rating ("Easy ≈ 1200, Normal ≈ 1500, Hard ≈ 1800"). Difficulty becomes a measured number, not a label.

### B.2 Shakuni mode needs two more brains

**The detector (every Pandava bot, and the v2 suspicion meter):**
- Keep a posterior `P(player i is Shakuni | everything seen so far)` for each seat, starting at 1/3 each.
- Each roll updates it with a likelihood ratio: `P(outcome | i is Shakuni) / P(outcome | fair)`. Under fair dice every face
  is 1/6. Under "Shakuni with wishes left" the model says high-value outcomes (a six, an exact capture, an exact home
  entry) are more likely, by how often a sensible Shakuni would wish in that spot. Work in log-probabilities to avoid underflow.
- Whisper events ("a wish was used this round") update all three posteriors by who could have benefited that round.
- **Decision to call a Sabha** is expected-value: call if `P(top suspect) × gain_if_right > (1 − P) × cost_if_wrong`, with
  the one-call-per-game budget making the threshold rise early and fall late. Easy detectors just count sixes; Normal uses
  the Bayesian update; Hard also models Shakuni's strategy (he avoids obvious wishes), which sharpens the likelihoods.

**The deceiver (the Shakuni bot):**
- Before each roll, compute `gain(face)` for each of the six faces using the level-2 scoring, and `expected_gain` of a fair roll.
- Compute the **suspicion cost** of each face by running the detector's own update on itself: "how much would this
  outcome raise their posterior on me?" Shakuni uses the same maths as his hunters, pointed at himself.
- Spend a wish only when `gain(face) − expected_gain` exceeds `λ × suspicion_cost + value_of_saving_the_wish`.
  The saving value comes from a small dynamic program over (wishes left, turns left).
- Randomise: sometimes wish a four instead of a six when the four is nearly as good, and sometimes skip a clearly good
  wish, so the pattern is not predictable (a mixed strategy, the game-theory answer to being read).
- Difficulty: Easy Shakuni wishes greedily (gets caught), Normal uses the suspicion cost, Hard also baits a false accusation
  by leaving another player's rolls looking lucky (plays near them so their captures look suspicious).

### B.3 Using the bots to balance the mode (this is the loop)

1. Fix a rule set (wishes N, cooldown, whisper chance p, turn cap T, exposure penalty).
2. Run 1,000 games: 1 Shakuni bot vs 3 Pandava bots at matched levels, seeds logged.
3. Record Shakuni win rate, Sabha call rate, correct-call rate, average turn of the call, game length.
4. Adjust one lever, repeat. Target: Shakuni 40–50 %, a Sabha in most games, correct calls 50–65 %.
5. Freeze the table in the README. Re-check with ten human games; humans are worse detectors than Normal bots at first,
   so the shipped default may need N = 2 for public lobbies and N = 3 for friends who know the mode.

**Where this lives in the roadmap:** Phase 3 builds the search and Monte Carlo machinery on Connect Four; Phase 4 ports
it to Ludo and adds the detector and deceiver; Phase 5 rates the bots; Phase 9 reuses the detector as the suspicion meter
and as anti-cheat for human players.

---

## Appendix C — Closing the gaps to "Flipkart / Uber / BookMyShow" scale concepts

The core roadmap already covers: horizontal scaling behind a load balancer, stateless API instances with shared state in
Redis, pub/sub fan-out, queues and workers, idempotency, optimistic concurrency, rate limiting, cache-aside, event
sourcing, archival, observability, graceful deploys, load testing. Those are the backbone of every large consumer backend.
These additions cover what a game server would otherwise miss. Each is one to three evenings and slots into an existing phase.

| # | Concept (where big companies use it) | What you add to chaupar | Slots into |
|---|---|---|---|
| C1 | **High-contention booking, no oversell** (BookMyShow seats, Flipkart flash sales) | Tournaments get limited seats (say 100). Seat *hold* in Redis with a TTL (`SET NX PX`), *confirm* in a MySQL transaction with `SELECT … FOR UPDATE`, hold expiry job releases seats. Load-test with 2,000 bots hitting "join" at once; prove zero oversell and show the numbers. | Phase 5 (tournaments) + Phase 7 (load test) |
| C2 | **Ledger and payments without money** (Flipkart/PhonePe wallets) | A virtual-coin economy: entry fees, prizes, daily bonus. Double-entry ledger table (every coin movement is two rows that sum to zero), idempotency keys on every credit/debit, balance = sum of entries, reconciliation job that proves the books balance. | Phase 5 |
| C3 | **Reliable events between a DB and a queue** (every order pipeline) | The **outbox pattern**: when a game finishes, write the result row and an outbox row in the same MySQL transaction; a relay publishes outbox rows to Redis/BullMQ and marks them sent. No lost or duplicated events on crash. | Phase 7 |
| C4 | **Large database operations** (any table with hundreds of millions of rows) | MySQL **partitioning** of `games` and `analysis` by month; a **read replica** in Compose (MySQL async replication) with read/write splitting in the app; measure replica lag; slow-query log review; archival to S3 (already in Phase 8). | Phase 7–8 |
| C5 | **A real cloud load balancer** (every production deployment) | One evening: AWS **Application Load Balancer** + target group + health checks + two EC2 instances in an Auto Scaling group, sticky sessions for WebSockets, connection draining on deploy. Tear it down the same night (ALB is paid; a few hours fits in the free-plan credits). Write up ALB vs your nginx. | Phase 8 |
| C6 | **Splitting a monolith** (microservices, with the pain) | Move matchmaking into its own service with an internal HTTP (later gRPC) API, its own deploy, a health check, and a timeout + retry + circuit breaker in the caller. Feel what you gain (independent scaling) and what you pay (latency, failure modes). | Phase 7 |
| C7 | **Geo** (Uber, Swiggy, Zomato) — the one thing a game server never needs | Optional one-week feature: "play with people nearby" using Redis GEO (`GEOADD`, `GEOSEARCH`) and a geohash explainer in the README. If skipped, cover it in the paper-design thread below. | After v1 or Phase 2 stretch |

### System-design study thread (runs alongside, 2 h every second Sunday)

Building gives intuition; interviews also ask you to design systems you have not built. Every second Sunday, design one
classic on paper (one page: requirements, estimates, data model, APIs, bottlenecks, trade-offs), then map every box to
something in chaupar:

| Week | Design on paper | Maps to |
|---|---|---|
| 4 | Rate limiter as a service | Your token bucket in Redis |
| 6 | URL shortener at scale | ULIDs, keyset pagination, caching, archival |
| 8 | WhatsApp / real-time chat | Socket.IO rooms, Redis adapter, presence, delivery receipts |
| 10 | BookMyShow seat booking | C1 holds and confirms |
| 12 | Flipkart flash sale / inventory | C1 + C2 + queues as a buffer |
| 14 | Uber matching and ETA | C7 geo, matchmaking as a queue, Little's law |
| 16 | Payments / wallet (PhonePe) | C2 ledger, idempotency, C3 outbox |
| 18 | Leaderboards and fantasy sports (Dream11 on IPL night) | Redis sorted sets, event sourcing, your 2,000-player load test |

Resources: [System Design Primer](https://github.com/donnemartin/system-design-primer), [ByteByteGo](https://www.youtube.com/@ByteByteGo),
[Hussein Nasser](https://www.youtube.com/@hnasr), Gaurav Sen and Arpit Bhayani on YouTube (both Indian, both explain the
designs companies here actually ask about). Write each one-pager into `docs/system-design/` — recruiters read those.

---

## Appendix D — Where players and stars actually come from (₹0)

Search data says "chaupar" brings no relevant traffic and the Ludo phrases are small. That is normal: multiplayer games
grow by invitation, not by Google. There are two audiences and they arrive by different doors.

**Audience 1 — players (for fun, feedback and the load graphs)**
- *Room links are the growth loop.* Every game needs three more people. One tap shares `chaupar.pages.dev/r/ABC123` on
  WhatsApp. Make joining need zero signup (guest name, upgrade later). This is how Skribbl, Gartic and Codenames Online grew.
- *Shakuni mode is the story people tell.* "One of us was secretly Shakuni" is a message people forward; plain Ludo is not.
- *Telegram as a second client (Phase 3 stretch).* A bot with inline keyboards puts the game inside groups that already
  play Ludo bots daily. Telegram bots are free and discoverable inside Telegram's own search.
- *Communities, in order of fit:* college and office WhatsApp groups, Telegram Ludo groups, r/india and r/IndianGaming
  threads around IPL time, Discord servers of Indian dev communities, Product Hunt once v1.0 has a GIF.
- *Realistic numbers:* 20–200 real players in the first months from friends and communities. That is enough to find
  bugs, balance Shakuni mode with humans, and produce a real Grafana screenshot. Bots fill every other seat.

**Audience 2 — developers (the GitHub stars and the recruiters)**
- They never come from Google either. They come from: Show HN ("Show HN: open-source Ludo server with provably fair dice
  and a hidden-role mode"), r/node, r/selfhosted, r/gamedev, dev.to and Hashnode write-ups (the five planned), the
  awesome-nodejs and awesome-selfhosted lists, and LinkedIn posts with one diagram and one number each.
- *What makes developers star it:* the fairness proof, the bot win-rate table, the 2,000-player load test, the Glicko-2
  implementation, a one-command install. Numbers and diagrams, not features.
- *Realistic numbers:* 50–300 stars in the first year if the write-ups are good. For a backend portfolio that is plenty;
  recruiters read the README and the commits, not the star count.

**Search, kept in proportion**
- Title the landing page and README for the phrases people do type: "Play Ludo online with friends — no download, open
  source" and "multiplayer Ludo website". Difficulty is low on the long-tail ones; expect a trickle after months, not a flood.
- Do not buy a domain for SEO. Buy one (shakuni.in or chaupar.dev) only when you want a cleaner link to share.

**Measure it (and learn one more backend tool):** self-host Umami or Plausible in Compose for page and room analytics,
and log "room created → 4 players joined → game finished" as a funnel. That funnel is your real metric, not keyword volume.
