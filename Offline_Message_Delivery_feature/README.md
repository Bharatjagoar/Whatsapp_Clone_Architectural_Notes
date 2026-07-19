# ChatMeSerivice — Offline Message Delivery

Architecture notes for the offline-message-delivery feature in
[ChatMeSerivice](https://github.com/Bharatjagoar/ChatMeSerivice), a
WhatsApp-clone backend built on Node.js, Socket.IO, Redis, RabbitMQ, and
MongoDB, split across a main service and a message service.

This repo documents *why* the feature is built the way it is, not just
*what* it does. Each decision below is written as: the problem being
solved, the solution chosen, and — where relevant — the reasoning that
changed along the way, including wrong turns that were corrected.

## What this feature does

When a user is offline and receives a message, the message is stored with
`status: "sent"`. When they reconnect:

1. **Lane 1** — the main service asks the message service for that user's
   pending offline messages, over RabbitMQ.
2. **Lane 2** — the message service queries MongoDB and replies with the
   messages (or a bounded retry, then a clean failure signal, if MongoDB
   is briefly unreachable).
3. **Lane 3** — once messages are delivered to the client, the main
   service confirms delivery back to the message service, which updates
   MongoDB to `status: "delivered"`.
4. **Sender notification** — the original sender of a delayed-delivery
   message gets a live "delivered" event if they're still online, so
   their UI can flip from single-tick to double-tick.

## Files

| File | Service | Role |
|---|---|---|
| `processOfflineMessages.js` | Main | Sends the offline-messages request, waits for a reply, delivers to the client, confirms delivery |
| `MarkDelivery.js` | Message | Queries MongoDB for offline messages, replies over RPC, retries on transient DB failure |
| `updateDeliveryStatus.js` | Message | Marks messages delivered in MongoDB, groups by sender, dispatches sender notifications |
| `notifySenderDelivered.js` | Main | Looks up the sender's live socket via Redis, emits the delivery confirmation |

## Decisions documented

Full reasoning for each is in
[`ChatMeSerivice_Offline_Delivery_Architecture.txt`](./ChatMeSerivice_Offline_Delivery_Architecture.txt).

1. **Anonymous exclusive reply queues, one per request** — fixes a real
   race where a userid-keyed shared queue could hand one request's reply
   to a different concurrent request from the same user, silently losing
   messages.
2. **Removed `correlationId`** — was necessary defense under the old
   shared-queue design; became dead code once queues were made exclusive
   per-request, and was removed rather than kept as decoration.
3. **Explicit error signaling in reply payloads** (`{error, messages}`)
   — an empty reply used to mean either "no offline messages" or "we gave
   up after an outage," indistinguishably. Now it doesn't.
4. **Bounded in-process retry for transient MongoDB failures** — a real
   scenario (login immediately followed by a refresh, colliding with a
   brief DB blip) previously lost the request outright. Now retries for
   up to 10 seconds before giving up cleanly.
5. **Timeout on the waiting consumer** — without it, a crashed or
   never-replying message service would leave the main service's consumer
   hanging forever. Also a case study in the gap between "this code was
   written in conversation" and "this code is actually in the file" —
   caught late, after an earlier false "done."
6. **Removed a Dead Letter Queue; moved validation upstream** — a DLQ was
   built first as "the safer option," then found to add no real value
   once traced through: the malformed IDs it would catch can only
   originate from a bug in this service's own code, not external input,
   so nothing useful could be done with a DLQ entry. Validation was moved
   to the point where the data is generated instead.
7. **Sender delivery notification, fire-and-forget** — cross-process
   design (message service has no Socket.IO access, so it hands off to a
   consumer in the main service via RabbitMQ). Deliberately best-effort:
   no publisher confirms, no retry-if-sender-offline, because the
   underlying "delivered" status is already safely persisted in MongoDB
   before this step runs — a missed live notification is a delayed UI
   update, not a data-loss bug.

## Known open items

- Frontend has no real error handling on the offline-messages receive
  callback, and no UI logic yet for single/double-tick based on the new
  `messagesDelivered` event.
- Redis error handling inside `notifySenderDelivered.js` is currently
  coarse (any error discards without distinguishing transient vs.
  permanent) — deliberately deferred, not forgotten.
- No `prefetch` set on the message service's consumer channel — unrelated
  to this feature, but open.
- **An unresolved bug was found in this feature during testing, in the
  session this documentation was written from, and had not been diagnosed
  before the session ended.** Treat the design above as *agreed upon*,
  not yet *confirmed correct in practice*, until this is closed out.

## Why this repo exists

Written primarily as a record for myself — engineering decisions on a
solo project tend to get lost between sessions and across months. Kept
in prose with explicit "here's what I got wrong first" sections rather
than a polished retrospective, because the wrong turns are usually the
more useful part to remember.