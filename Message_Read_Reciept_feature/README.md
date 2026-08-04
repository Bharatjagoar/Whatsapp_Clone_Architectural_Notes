# ChatMeService — Message Read Status

Architecture notes for the read-status (blue tick) feature in
[ChatMeService](https://github.com/Bharatjagoar/ChatMeService.git), a
WhatsApp-clone backend built on Node.js, Socket.IO, Redis, RabbitMQ, and
MongoDB, split across a main service and a message service.

This repo documents *why* the feature is built the way it is, not just
*what* it does. Each decision below is written as: the problem being
solved, the solution chosen, and — where relevant — the reasoning that
changed along the way, including wrong turns that were corrected.

## What this feature does

When a user opens a chat, every message in that conversation still marked
`"sent"` or `"delivered"` is marked `"read"`:

1. **Lane 1** — the client sends a mark-as-read request to the main
   service on chat open (and again if new messages arrive while the chat
   stays open). Main service responds immediately; it doesn't wait on
   anything downstream.
2. **Lane 2** — the message service updates MongoDB for the whole
   conversation, then, only if anything actually changed, fires a
   notification for the original sender.
3. **Lane 3** — the main service looks up the sender's live socket via
   Redis and emits a "read" event if they're still connected, flipping
   their tick from grey double-check to blue.

## Files

| File | Service | Role |
|---|---|---|
| `messagesController.js` (`MarkAsRead`) | Main | Receives the mark-as-read request, fires the update, responds immediately |
| `updateReadStatus.js` | Message | Marks the conversation's messages read in MongoDB, dispatches the sender notification if anything changed |
| `notifySenderRead.js` | Main | Looks up the sender's live socket via Redis, emits the read confirmation |

## Decisions documented

Full reasoning for each is in
[`Message_Read_Status_feature.txt`](./Message_Read_Status_feature.txt).

1. **Read scope: whole conversation, not individual messages** — unlike
   delivered-status, read isn't tracked per message ID. A live-received
   message never has a real Mongo `_id` on the client to begin with, so
   the update is scoped by `chatId` + `receiverId` instead, sidestepping
   the missing-ID problem rather than working around it.
2. **Transport: direct request, not RabbitMQ RPC** — delivered-status
   needs a queue because of real timing coordination around offline
   reconnects; read-status has no such coordination need, since the user
   is already connected by definition. A missed read update is also
   cosmetic (self-corrects next time the chat is opened), unlike a missed
   delivered update, so the retry/ack machinery delivered-status uses
   wasn't justified here.
3. **Trust boundary: client-supplied ID for routing, server-verified ID
   for authorization** — the request carries two identities. The one that
   decides whose messages get mutated (`receiverId`) comes only from the
   server-verified JWT. The one used purely to route a live notification
   (`otherUserId`) is accepted from the client, because getting it wrong
   only misroutes a notification — it can't mutate data on someone else's
   behalf.

## Known open items

- Read-status intentionally has no dead-letter or requeue behavior on
  permanent failure (unlike offline-delivery's Decision 11) — a dropped
  update here just waits for the next chat-open to self-correct, so the
  asymmetry is deliberate, not an oversight.
- No multi-instance handling for the live-socket check in
  `notifySenderRead.js` — same accepted limitation as delivered-status's
  Decision 8, unaddressed for the same reason (single-instance scale
  currently, no production load requiring it yet).

## Why this repo exists

Written primarily as a record for myself — engineering decisions on a
solo project tend to get lost between sessions and across months. Kept
in prose with explicit "here's what I got wrong first" sections rather
than a polished retrospective, because the wrong turns are usually the
more useful part to remember.