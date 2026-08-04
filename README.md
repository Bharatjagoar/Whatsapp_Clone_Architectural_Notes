# Architecture Decisions

Engineering decision logs for ChatMeService, a WhatsApp-clone backend
(Node.js, Socket.IO, Redis, RabbitMQ, MongoDB). Each feature gets its own
folder: a short README summarizing the decisions, and a full write-up of
the reasoning behind each one — including the wrong turns that got
corrected along the way.

## Features

- **[Offline Message Delivery](./Offline_Message_Delivery_feature/README.md)**
  — how messages sent to an offline user get queued, retried, delivered on
  reconnect, and confirmed back to the sender. Covers RabbitMQ RPC
  patterns, race conditions in reply-queue design, and why a
  dead-letter-queue was built and then removed.

More features will be added here as they're built.