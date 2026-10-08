# Lesson 1 — WebSocket (card)

**What:** one open line between browser and server where both can send messages at any time.

**Why:** normal HTTP only works when the browser asks. A game server must say "your opponent moved" without being asked.

**How (4 steps):**
1. Browser sends a normal HTTP request: "upgrade me to WebSocket".
2. Server replies "OK" (status 101).
3. Same connection now carries small messages both ways, for as long as you want.
4. Server pings every ~30 s; no pong → phone is dead → close it.

**Everyday example:** HTTP is post: you write a letter, you get a reply. WebSocket is a phone call: both can talk any time until someone hangs up.

**Who uses it, when:** WhatsApp Web (messages appear instantly), Zerodha/Groww (live price ticks), Dream11 and Ludo King (live scores and moves), Google Docs (other people's cursors), Uber (driver moving on the map).

**When NOT:** pages, forms, normal REST APIs → plain HTTP is simpler and cheaper. One-way live updates only (a cricket score page) → Server-Sent Events is enough.

**In chaupar:** every move, dice roll and chat line travels on one WebSocket per player. The ping/pong is how we know a player dropped.

**One line to remember:** "HTTP is request–response. WebSocket is a persistent two-way TCP connection started by an HTTP Upgrade handshake (101), and it needs a heartbeat to detect dead clients."

---

## Quiz 1 (short answers, your own words)
1. In one line: what is a WebSocket, and what can it do that HTTP cannot?
2. Name one app you use daily that needs a WebSocket and one page that does not. Why?
3. How does a WebSocket start? Two steps.
4. A phone goes to sleep mid-game. How does the server find out, and why does it not find out by itself?

## Interview questions 1 (after the quiz)
1. HTTP vs long polling vs SSE vs WebSocket: when would you choose each?
2. What happens in the WebSocket handshake?
3. How do you detect a dead WebSocket client?
4. Node is single-threaded. How does it hold 10,000 connections? What breaks it?
5. What would you put in every message (an "envelope") and why?

## Task 1 (type it yourself, 60–90 min)
1. Install nvm, then Node 24: https://github.com/nvm-sh/nvm#installing-and-updating → `nvm install 24 && nvm use 24` → check `node -v`.
2. `mkdir chaupar && cd chaupar && git init && npm init -y`, set `"type": "module"` in package.json, `npm i ws`.
3. Read the `ws` README sections "Sending and receiving text data" and "How to detect and close broken connections".
4. Write `src/server.js`: WebSocket server on port 3000 that echoes every message back, logs connect/disconnect, pings every 30 s and terminates clients that do not pong. Read, then type; do not paste.
5. Test from the browser console: `const s = new WebSocket('ws://localhost:3000'); s.onmessage = e => console.log(e.data); s.send('hi')`.
6. Commit (Conventional Commit message), then run `/tutor review` and point me to `src/server.js`.
**Done when:** echo works from two tabs; a tab set to offline in DevTools gets terminated on the server within ~30 s.

---

## Optional deep dive (only if you want more)
HTTP is "ask, then answer". The browser asks, the server answers, then the line goes quiet. But in a game the server
must tell YOU something when the OTHER player moves, and you did not ask. HTTP has no way for the server to speak first.

## Four ways to solve it
| Way | How | Cost |
|---|---|---|
| Polling | Browser asks every second "anything new?" | 99 % of requests are wasted; moves arrive up to 1 s late |
| Long polling | Browser asks; server holds the request open until something happens, then answers; browser asks again | Works everywhere, but one HTTP round trip per event and awkward to code |
| SSE (Server-Sent Events) | One HTTP response that never ends; server streams events down it | Server → browser only. Browser must still use HTTP to send moves |
| **WebSocket** | One TCP connection, both directions, any time, tiny frames | Needs a server that speaks the protocol. This is what games use |

## How a WebSocket starts (the handshake)
It begins as a normal HTTP request:
```
GET /play HTTP/1.1
Host: chaupar.dev
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```
If the server agrees, it answers `101 Switching Protocols` with `Sec-WebSocket-Accept`, which is a hash of the key plus
a fixed magic string. From that moment the same TCP connection stops being HTTP and becomes a stream of **frames**.
The key/accept dance proves both sides really intended a WebSocket (a cache or proxy cannot fake it by accident).

## Frames
A frame = a small header + payload. The header has: FIN bit (last piece of a message?), an opcode (1 text, 2 binary,
8 close, 9 ping, 10 pong), the payload length, and a mask bit. Browser → server frames are always **masked** (XORed with
a random key); server → browser frames are not. Masking exists to stop old proxies from mistaking frame bytes for HTTP
and poisoning their caches. The `ws` library hides all of this; you only see `socket.send()` and `'message'` events.

## Dead connections (the part beginners miss)
When a phone goes to sleep or Wi-Fi drops, no close frame is sent. TCP keeps thinking the connection is alive for
minutes. So the server must **heartbeat**: send a ping every N seconds, expect a pong, and terminate sockets that stay
silent. Without this, a Ludo table fills with ghosts whose turns never end.

## One process, thousands of sockets
Node handles many sockets on one thread because waiting for the network is not work; the event loop just reacts when
bytes arrive. But CPU work on that thread (a bot thinking for 2 s) freezes every socket for 2 s. That is why bots run in
worker processes later (Phase 3).

## Server-authoritative (the rule that shapes everything)
The browser never says "I moved to cell 4 and now it's your turn". It says "I want to move to cell 4". The server checks
whose turn it is, whether the cell is free, applies the move, and broadcasts the new state to everyone. The client only
draws what the server says. Reasons: cheating, bugs, out-of-order messages, reconnects.

## Message design
Every message is JSON with an envelope: `{ t: 'move', seq: 17, payload: { cell: 4 } }`. `t` = type, `seq` = a counter
so the receiver can detect gaps and duplicates. Write every message type down in `docs/protocol.md` before coding.

## Where chaupar uses this
Every move, every dice roll, every chat line and the live feed are frames on one socket per player. Heartbeats drive
"player disconnected" handling and the auto-move timer. The envelope's `seq` is what makes reconnection (Phase 2) possible.
