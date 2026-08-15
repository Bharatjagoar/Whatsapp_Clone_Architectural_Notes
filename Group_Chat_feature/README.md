# ChatMeService — Group Chat

Architecture notes for the group-chat feature in
[ChatMeService](https://github.com/Bharatjagoar/ChatMeService.git), a
WhatsApp-clone backend built on Node.js, Socket.IO, Redis, RabbitMQ, and
MongoDB, split across a main service and a message service.

This repo documents *why* the feature is built the way it is, not just
*what* it does. Each decision below is written as: the problem being
solved, the solution chosen, and — where relevant — the reasoning that
changed along the way, including wrong turns that were corrected.

## What this feature does

1. **Creation** — a user creates a group with a name and a member list.
   Main service receives the request and relays it to message service over
   RabbitMQ RPC, which persists the `Group` document and replies with it.
2. **Presence** — on socket connect, main service asks message service
   (via RPC) which groups the connecting user belongs to, then writes that
   user's ID into a Redis Set per group (`group:<groupId>:online`).
3. **Send** — a sender's client emits a group message. Main service
   requests an atomically-assigned sequence number and persistence from
   message service (RPC, waits for reply), then fans the message out to
   every online member using the group's Redis presence set plus the
   existing per-user socket registry.
4. **Read tracking** — as messages enter a member's viewport, their client
   fires a fire-and-forget event that advances a per-(user, group) read
   pointer (`lastReadSeq`) in MongoDB.

## Files

| File | Service | Role |
|---|---|---|
| `groupController.js` / `groupRoute.js` | Main | `POST /group/create`, publishes the create-group RPC request |
| `CreateGroup.js` | Message | Persists the `Group` document, replies over RPC |
| `processGroupMemberships.js` | Main | On connect, requests this user's group memberships, writes Redis presence |
| `GetUserGroups.js` | Message | Answers the membership RPC by querying `Group.members` |
| `GroupSeqCounter.js` | Message | Atomic per-group `$inc` counter handing out message sequence numbers |
| `SendGroupMessage.js` | Message | Assigns `seq`, persists the message, replies over RPC |
| `sendGroupMessage` handler (`socket.js`) | Main | Requests seq+persist via RPC, fans out to online members on reply |
| `GroupReadState.js` | Message | Schema + `markReadUpTo` — direct-set read pointer per (user, group) |
| `markGroupRead` handler (`socket.js`) | Main | Fire-and-forget relay from client viewport events to the read-pointer write |

## Decisions documented

Full reasoning for each is in
[`Group_Chat_Architecture.txt`](./Group_Chat_Architecture.txt).

1. **Atomic per-group sequence counter, not Mongo `_id` or timestamp** —
   `_id`/timestamp ordering can disagree with what a given client actually
   rendered under concurrent sends. A `$inc`-based counter gives every
   message a single, race-safe position every client and the server agree
   on.
2. **Pointer-based read model (`lastReadSeq`), not per-message read
   records** — one document per (user, group) instead of one per (user,
   message). Named, accepted tradeoff: cannot represent "read with holes"
   (a skipped-then-later message can get marked read without ever being
   seen), permanent once the pointer passes it, not self-healing on
   re-render. Accepted because the failure case is rare and low-stakes for
   a chat app, same tradeoff WhatsApp/Telegram make.
3. **Read-mark transport: fire-and-forget, not RPC** — unlike group send,
   the marking client needs nothing back to render correctly. A dropped
   read-mark write self-corrects the next time the user reads further,
   since the pointer only ever advances forward — so the retry/ack
   machinery group send needs wasn't justified here.
4. **Group send: RPC-and-wait, not fire-and-forget like 1:1** — 1:1 send
   can fire-and-forget because nothing about the emit depends on
   persistence. Group send can't: every recipient needs a message that
   already carries its correct `seq`, and `seq` only exists after a
   cross-service atomic increment completes. Considered and rejected
   optimistic-emit-then-reconcile: it reopens the exact ordering problem
   decision 1 solves, plus adds visible message reordering on-screen.
5. **Isolated the DB write from the reply-send in retry logic** — both
   `CreateGroup.js` and `SendGroupMessage.js` originally retried the whole
   operation (including the write) if the reply-send failed after a
   successful write, risking duplicate documents — and for
   `SendGroupMessage.js`, a duplicated message plus a burned `seq` number.
   Split into a write-only retry path and a never-retried, best-effort
   reply path.
6. **Group schemas live in message service, not main service** — `User`
   lives in main service, but `GroupSeqCounter` needs to be updated in
   close lockstep with `GroupMessage`, so both (and `Group`,
   `GroupReadState`) live in message service's Mongo connection instead,
   matching the existing plain-`ObjectId`-no-`ref` convention already used
   for cross-service user references.
7. **Redis presence: Set of member IDs, looked up alongside the existing
   socket registry, not merged into one structure** — storing socket IDs
   in two places would require every reconnect/disconnect to update both
   to stay correct, risking silent staleness. Two O(1) lookups per online
   member costs the same complexity class as one merged lookup would (both
   are O(N) for N online members) — not worth the correctness risk to save
   a constant factor.

## Known open items

- No frontend UI for group chat yet — creation, send, and read-marking are
  all verified working via a standalone Socket.IO test script
  (`testGroupSend.js`), not through the actual client.
- `markGroupRead`'s consumer (`MarkGroupRead.js`) does not retry on
  transient DB failure, unlike `CreateGroup.js`/`SendGroupMessage.js` —
  deliberate, per decision 3, not an oversight.

## Why this repo exists

Written primarily as a record for myself — engineering decisions on a
solo project tend to get lost between sessions and across months. Kept
in prose with explicit "here's what I got wrong first" sections rather
than a polished retrospective, because the wrong turns are usually the
more useful part to remember.