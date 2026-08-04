# ChatMeService — Offline Message Delivery

Architecture notes for the offline-message-delivery feature in
[ChatMeService](hhttps://github.com/Bharatjagoar/ChatMeService.git, a
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
[`WhatsApp_Offline_Delivery_Architecture.txt`](./WhatsApp_Offline_Delivery_Architecture.txt).

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
   hanging forever.
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
8. **Stale-socket check and payload validation in `notifySenderDelivered.js`**
   — a Redis entry existing doesn't mean the socket behind it is still
   connected. Added a live-socket check against Socket.IO's own registry
   instead of trusting Redis alone, plus array validation on the incoming
   payload so a malformed message can't get misclassified as a connection
   failure and silently discarded.
9. **Bounded retry on transient Redis failures** — same shape as decision
   4, applied to Redis instead of Mongo. A short Redis blip shouldn't cost
   a sender their live delivery notification; a genuinely dead lookup
   still fails fast.
10. **`clientMessageId`** — a client-generated UUID that gives the sender's
    own client a stable identity for a message before MongoDB's `_id`
    exists yet, so the delivery-notification event (decision 7) can be
    matched back to the right message in the sender's UI.
11. **Bounded retry on `updateDeliveryStatus.js`, with requeue instead of
    discard on final failure** — this consumer does multiple sequential
    DB operations per message, so a late failure can leave the delivered
    status already correctly written while only the notification step
    failed. Requeuing (rather than discarding) gives that last step
    another chance without any risk to already-correct data.

## Known open items

- Frontend has no delivery-tick UI polish yet — status-driven single/
  double-tick rendering exists, but the visual pass (icons, layout) isn't
  done.
- No `prefetch` set on any of the message service's consumer channels —
  unrelated to this feature specifically, but open.
- `updateDeliveryStatus.js`'s requeue-on-final-failure (decision 11) has
  no dead-letter mechanism behind it — a genuinely permanent failure
  there would requeue indefinitely rather than fail cleanly.

## Why this repo exists

Written primarily as a record for myself — engineering decisions on a
solo project tend to get lost between sessions and across months. Kept
in prose with explicit "here's what I got wrong first" sections rather
than a polished retrospective, because the wrong turns are usually the
more useful part to remember.