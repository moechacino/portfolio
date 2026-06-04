# Project API B: Chatting App

> API documentation sample for a messaging backend.

Project API B is a chatting backend concept that supports user accounts, conversation lists, message history, and real-time message delivery.

## Highlights

<div class="badge-row">
  <span class="badge">Users</span>
  <span class="badge">Conversations</span>
  <span class="badge">Messages</span>
  <span class="badge">Realtime</span>
</div>

## Core Features

| Module | Description |
| --- | --- |
| Auth | Login, logout, identity verification |
| Contacts | User discovery and contact state |
| Conversations | Direct and group conversation metadata |
| Messages | Send, read, edit, delete, and paginate messages |
| Presence | Online state and typing indicators |

## Example Endpoints

```http
POST /api/auth/login
GET /api/conversations
POST /api/conversations
GET /api/conversations/:id/messages
POST /api/conversations/:id/messages
PATCH /api/messages/:id/read
```

## Suggested Stack

* Runtime: Node.js, Go, Laravel, or equivalent backend framework
* Database: PostgreSQL or MongoDB
* Realtime: WebSocket, Socket.IO, or server-sent events
* Cache: Redis for presence, fan-out, and ephemeral state

## Implementation Notes

Message pagination should use cursor-based queries so older messages load consistently even when new messages arrive during the session.
