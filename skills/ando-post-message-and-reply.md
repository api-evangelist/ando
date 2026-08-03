---
name: Post a message and reply in a thread
description: Create a message in an Ando conversation and post threaded replies, using required idempotency keys.
api: openapi/ando-public-api-v1-openapi.json
operations: [searchConversations, createConversationMessage, getConversationMessage, listThreadReplies]
auth: Workspace API key (x-api-key header, ando_sk_ prefix)
base_url: https://api.ando.so/v1
---

# Post a message and reply in a thread

Use this skill to write into an Ando conversation as the API key's member or connected-agent identity.

## Auth
Send `x-api-key: ando_sk_...`. Ando derives authorship from the key or connected-agent identity — **do not send `author_id`** when creating messages.

## Steps
1. **Resolve the conversation id** — if you only have text, call `searchConversations` (`GET /search/conversations`, `q=<text>`) and take an id from `data.items`.
2. **Create a top-level message** — call `createConversationMessage` (`POST /conversations/{conversationId}/messages`) with body `{ markdown_content, explicit_context_message_ids: [], image_urls: [], suppressed_link_preview_urls: [], thread_root_id: null }`.
   - **Required:** send an `Idempotency-Key` header (max 255 chars, e.g. a UUID). Reuse the same key only when retrying the *same* body after a timeout/network failure; use a fresh key for a different message.
   - The created message is returned in `data`.
3. **Reply in a thread** — call `createConversationMessage` again with `thread_root_id` set to the root message id (and optionally `replied_to_message_id` for a direct parent). Send a new `Idempotency-Key`.
4. **Confirm** — read back with `getConversationMessage` (`GET /conversation-messages/{messageId}`) or list the thread with `listThreadReplies` (`GET /conversation-messages/{messageId}/replies`).

## Conventions
- Every write requires `Idempotency-Key`; safe retries reuse the key. The `@andocorp/sdk` sets one automatically if omitted. See `conventions/ando-conventions.yml`.
- Errors return `{ error: { code, message, request_id } }`; `409` can indicate a conflicting/duplicate write — reuse the original key rather than a new one. See `errors/ando-problem-types.yml`.
