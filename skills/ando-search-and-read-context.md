---
name: Search and read Ando workspace context
description: Find messages and conversations in an Ando workspace by text/semantic search, then read history and thread replies.
api: openapi/ando-public-api-v1-openapi.json
operations: [searchMessages, searchConversations, listConversationMessages, getConversationMessage, listThreadReplies]
auth: Workspace API key (x-api-key header, ando_sk_ prefix)
base_url: https://api.ando.so/v1
---

# Search and read Ando workspace context

Use this skill to pull relevant context out of an Ando workspace before acting. All operations here are read-only and idempotent.

## Auth
Send a workspace API key on every request: `x-api-key: ando_sk_...` (or `Authorization: Bearer ando_sk_...`). Each key is scoped to one workspace; you only see conversations, messages, calls, tasks, and members visible to that key.

## Steps
1. **Find the conversation** — call `searchConversations` (`GET /search/conversations`) with `q=<text>`. Read results from `data.items`.
2. **Find messages by topic** — call `searchMessages` (`GET /search/messages`) with `q=<text>`, optionally scoping with `conversation`, `author`, `thread`, `mode` (`semantic`), and `limit`. Read `data.items`.
3. **Read recent history** — when you already have a conversation id, call `listConversationMessages` (`GET /conversations/{conversationId}/messages`) with `limit` and `before=<cursor-or-iso-timestamp>` to page backward. Read `data.items` and `data.page_info`.
4. **Fetch one message** — call `getConversationMessage` (`GET /conversation-messages/{messageId}`).
5. **Read a thread** — call `listThreadReplies` (`GET /conversation-messages/{messageId}/replies`); confirm the root via `data.thread_root_id`.

## Conventions
- Pagination is cursor-based: read `data.items` + `data.page_info`, page with `before`/`after`. Prefer the `data` envelope over the legacy top-level `items`/`pageInfo` aliases.
- Errors return `{ error: { code, message, request_id } }`. On `429`/`5xx`, back off and retry; capture `error.request_id` for support. See `errors/ando-problem-types.yml`.
