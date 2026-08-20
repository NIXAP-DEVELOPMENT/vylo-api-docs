---
---

# Vylo API

REST over HTTPS, JSON in and JSON out. Call and message history for your account, and sending.

| Host | Purpose |
|---|---|
| `https://history-api.voice.vylocloud.com` | History — conversations, calls, messages |
| `https://sms.voice.vylocloud.com` | Messaging — sending SMS and MMS |

Both accept the same token.

---

## 1. Authorization

We issue the token for your integration. Send it in the `Authorization` header on every request.

```http
GET /api/conversation_items HTTP/1.1
Host: history-api.voice.vylocloud.com
Authorization: Bearer <your token>
Accept: application/ld+json
```

```bash
curl "https://history-api.voice.vylocloud.com/api/conversation_items" \
  -H "Authorization: Bearer $VYLO_TOKEN" \
  -H "Accept: application/ld+json"
```

Always send `Accept: application/ld+json`. Plain `application/json` returns a bare array with no
envelope, which leaves you without the collection metadata.

### What the token covers

The token is bound to the phone numbers assigned to your integration. History returns only records
on those numbers — anything outside them simply does not appear in results. Tell us which numbers
the integration should cover and we will assign them.

### Errors

| Response | Meaning |
|---|---|
| `401 {"code":401,"message":"JWT Token not found"}` | No `Authorization` header on the request |
| `401 {"code":401,"message":"Invalid JWT Token"}` | Malformed token, or one we no longer accept |

A `401` on a request that previously worked means the token was replaced or revoked — contact us
for a new one rather than retrying.

> **Keep the token server-side.** It grants read access to your account's full call and message
> history. Never ship it to a browser, a mobile app, or anywhere a customer can reach it.

---

## 2. Conversations

A conversation is one thread between two numbers: yours (`ownerNumber`) and the other party's
(`peerNumber`). Calls and messages with the same pair share the same conversation, so the list
works as an inbox — one row per contact, most recent activity first.

```http
GET /api/conversations HTTP/1.1
Host: history-api.voice.vylocloud.com
Authorization: Bearer <your token>
Accept: application/ld+json
```

```json
{
  "@context": "/api/contexts/Conversation",
  "@id": "/api/conversations",
  "@type": "Collection",
  "member": [
    {
      "@id": "/api/conversations/90431",
      "@type": "Conversation",
      "id": 90431,
      "ownerNumber": "13125550100",
      "peerNumber": "13125550142",
      "callerName": "Northline Freight",
      "unreadCount": 2,
      "lastEvent": {
        "type": "sms",
        "direction": "inbound",
        "createdAt": "2026-08-19T14:41:03+00:00",
        "text": "Can we move the review call to Thursday?"
      }
    }
  ]
}
```

### Fields

| Field | Type | Description |
|---|---|---|
| `id` | integer | Conversation id. Use it to pull the thread's calls and messages. |
| `ownerNumber` | string | Your number. **Digits only, no leading `+`** — `13125550100`. |
| `peerNumber` | string | The other party's number, same format. |
| `callerName` | string \| null | Caller name for the peer, where the carrier supplied one. |
| `unreadCount` | integer | Unread items in this thread. |
| `lastEvent` | object \| null | Summary of the most recent activity — see below. |

There is exactly one conversation per `ownerNumber` + `peerNumber` pair, and its `id` is stable, so
you can map it to a thread in your CRM once and keep it.

### `lastEvent`

Shape depends on `type`. It is a summary for rendering a list — the authoritative record is the
conversation's items.

For a call:

```json
{ "type": "call", "direction": "inbound", "createdAt": "2026-08-19T14:22:07+00:00", "isMissed": false }
```

For a message:

```json
{ "type": "sms", "direction": "outbound", "createdAt": "2026-08-19T14:41:03+00:00",
  "text": "Confirmed for Thursday at 2pm.", "hasMedia": true, "mediaCount": 2 }
```

`hasMedia` and `mediaCount` appear only when the message carries attachments. `text` may be `null`
on an attachment-only message.

### Query parameters

| Parameter | Description |
|---|---|
| `order[updatedAt]=desc` | Most recently active first. This is the useful ordering for an inbox. |
| `order[id]=asc` \| `desc` | By conversation id. Defaults to `desc`. |
| `search[ownerNumber][]=13125550100` | Only threads on this number of yours. Repeatable. |
| `search[peerNumber][]=13125550142` | Only threads with this external number. Repeatable. |
| `unreadCount[gt]=0` | Only threads with unread activity. Also `gte`, `lt`, `lte`, `between`. |
| `id=90431` | One conversation by id. `id[]=90431&id[]=90432` fetches several. |
| `id[lt]` / `id[gt]` | Range on id — used for paging, see below. |

Ordering by `updatedAt` is supported even though the field is not returned; take the timestamp from
`lastEvent.createdAt` instead.

```bash
curl "https://history-api.voice.vylocloud.com/api/conversations?order[updatedAt]=desc&unreadCount[gt]=0" \
  -H "Authorization: Bearer $VYLO_TOKEN" \
  -H "Accept: application/ld+json"
```

### Paging

Pages hold 30 conversations and the collection carries no total count. Two ways to walk it:

- `?page=2`, `?page=3` … — simplest, fine for a list a user browses.
- `?order[id]=desc&id[lt]=<lowest id on the last page>` — stable while new conversations arrive,
  which page numbers are not.

A page shorter than 30 records means you have reached the end.

### One conversation

```http
GET /api/conversations/{id}
```

Returns the same object as a list entry.

---

## 3. Conversation items

Every call and every message is a conversation item. Both live in one collection, separated by
`type`, so a single sync loop covers all of your history.

```http
GET /api/conversation_items HTTP/1.1
Host: history-api.voice.vylocloud.com
Authorization: Bearer <your token>
Accept: application/ld+json
```

### Filters

| Parameter | Description |
|---|---|
| `search[type][]=call` | `call` or `sms`. Repeatable; omit for both. |
| `search[direction][]=inbound` | `inbound` or `outbound`, relative to `ownerNumber`. Repeatable. |
| `search[conversation]=90431` | One thread, by conversation id. |
| `search[ownerNumber][]=13125550100` | Only items on this number of yours. Repeatable. |
| `search[missed]=true` | Calls that were never picked up. |
| `search[recordingType][]=voicemail` | `recording` or `voicemail`. Repeatable. |
| `search[userUuid][]=<uuid>` | Calls this user placed **or** answered. Repeatable. |
| `id[gt]` / `id[lt]` / `id[gte]` / `id[lte]` / `id[between]` | Range on the record id — used for paging and sync. |
| `order[id]=asc` \| `desc` | Defaults to `desc`, newest first. |
| `page=2` | Page number, as an alternative to the id cursor. |

Filters combine with AND; repeating one parameter matches any of its values.

```bash
# Missed calls on one number, oldest first
curl "https://history-api.voice.vylocloud.com/api/conversation_items\
?search[type][]=call&search[missed]=true&search[ownerNumber][]=13125550100&order[id]=asc" \
  -H "Authorization: Bearer $VYLO_TOKEN" \
  -H "Accept: application/ld+json"

# Everything in one thread, newest first
curl "https://history-api.voice.vylocloud.com/api/conversation_items?search[conversation]=90431" \
  -H "Authorization: Bearer $VYLO_TOKEN" \
  -H "Accept: application/ld+json"
```

### A call

```json
{
  "@id": "/api/conversation_items/4812337",
  "@type": "ConversationItem",
  "id": 4812337,
  "conversation": "/api/conversations/90431",
  "type": "call",
  "direction": "inbound",
  "ownerNumber": "13125550100",
  "peerNumber": "13125550142",
  "createdAt": "2026-08-19T14:22:07+00:00",
  "missed": false,
  "duration": 184,
  "waitSeconds": 12,
  "talkSeconds": 172,
  "recordingType": "recording",
  "recordingPath": "…/9f3a1c74-e2b8-4d51-9c3e-77b0a1d2e3f4.wav",
  "externalId": "9f3a1c74-e2b8-4d51-9c3e-77b0a1d2e3f4",
  "answeredByUser": "b4c1e9d2-3f77-4a10-9d21-6c5be0a4f118",
  "originatedByUser": null,
  "directExtension": "104",
  "message": null,
  "media": null,
  "status": null,
  "statusError": null
}
```

### A message

```json
{
  "@id": "/api/conversation_items/4812340",
  "@type": "ConversationItem",
  "id": 4812340,
  "conversation": "/api/conversations/90431",
  "type": "sms",
  "direction": "outbound",
  "ownerNumber": "13125550100",
  "peerNumber": "13125550142",
  "createdAt": "2026-08-19T14:41:03+00:00",
  "message": "Confirmed for Thursday at 2pm.",
  "media": [
    { "url": "https://fs-storage.voice.vylocloud.com/media/public/mms/…jpg",
      "contentType": "image/jpeg", "size": 184320 }
  ],
  "status": "delivered",
  "statusError": null,
  "externalId": "40017b9c-3d5e-4a21-8f60-1c9de4a7b302",
  "missed": null,
  "duration": null,
  "recordingType": null,
  "recordingPath": null
}
```

### Fields

Every field is present on every item; the ones that do not apply to that `type` are `null`.

| Field | Type | Applies to | Description |
|---|---|---|---|
| `id` | integer | both | Record id. Increases monotonically — this is the sync cursor. |
| `externalId` | string \| null | both | Call id, or the `messageId` returned when the message was sent. |
| `conversation` | string | both | Reference to the thread — `/api/conversations/{id}`. |
| `type` | string | both | `call` or `sms`. |
| `direction` | string | both | `inbound` or `outbound`, relative to `ownerNumber`. |
| `ownerNumber` | string | both | Your number. Digits only, no leading `+`. |
| `peerNumber` | string | both | The other party's number, same format. |
| `createdAt` | string | both | When it happened. UTC, ISO-8601. |
| `duration` | integer \| null | call | Total seconds, ring time included. |
| `waitSeconds` | integer \| null | call | Seconds spent ringing. |
| `talkSeconds` | integer \| null | call | Seconds actually on the call — the figure to report on. |
| `missed` | boolean \| null | call | `true` when the call was never picked up. |
| `recordingType` | string \| null | call | `recording`, `voicemail`, or `null` when nothing was recorded. |
| `recordingPath` | string \| null | call | Path to the audio. Opaque — concatenate, do not parse. |
| `answeredByUser` | string \| null | call | UUID of the user who answered. |
| `originatedByUser` | string \| null | call | UUID of the user who placed the call. |
| `directExtension` | string \| null | call | Extension the call was routed to. |
| `message` | string \| null | sms | Message body. `null` on an attachment-only message. |
| `media` | array \| null | sms | Attachments as `{ url, contentType, size }`. |
| `status` | string \| null | sms | Delivery state of an outbound message. `null` inbound. |
| `statusError` | string \| null | sms | Reason reported when delivery failed. |

`status` moves forward only — a late receipt never undoes an outcome you already recorded.

| `status` | Meaning |
|---|---|
| `queued` | Accepted, not yet handed to the carrier. |
| `sent` | Handed off, no handset confirmation yet. |
| `delivered` | Confirmed on the handset. Final. |
| `delivered_as_text` | An MMS the carrier would not take as media — delivered as text with links. Final. |
| `undelivered` | No confirmation either way. Treat as unknown, not as failure. |
| `failed` | Rejected; `statusError` carries the reason. Final. |

Items may also carry `transcript`, `comments` and `reactions`. Those are for use inside Vylo — ignore
them unless we have agreed otherwise.

### Paging

Pages hold 30 items and carry no total count. Page with the id cursor rather than `page=N`, so new
activity arriving mid-walk cannot shift rows between pages:

```bash
# newest first: take the lowest id from the page you just read
?order[id]=desc&id[lt]=4812337

# oldest first: take the highest id
?order[id]=asc&id[gt]=4812337
```

A page shorter than 30 records means you have reached the end.

### One item

```http
GET /api/conversation_items/{id}
```

Returns the same object. Use it to re-check the `status` of a message you sent.

---

## 4. Sending SMS and MMS

Sending runs on the messaging host. Same `Authorization: Bearer` token as history.

```http
POST /send-sms HTTP/1.1
Host: sms.voice.vylocloud.com
Authorization: Bearer <your token>
Content-Type: application/json
```

| Field | Type | Description |
|---|---|---|
| `from` | string | **Required.** One of your numbers. `+13125550100` or `13125550100` — both accepted. |
| `to` | string | **Required.** Destination number, either form. |
| `text` | string | Message body. Required unless `mediaUrls` is given. |
| `mediaUrls` | array | Up to 10 attachment URLs from `/mms-attachment`. Turns the message into an MMS. |

```bash
curl -X POST https://sms.voice.vylocloud.com/send-sms \
  -H "Authorization: Bearer $VYLO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "+13125550100",
    "to": "+13125550142",
    "text": "Your appointment is confirmed for Thursday at 2pm."
  }'
```

```json
{ "success": true, "messageId": "40017b9c-3d5e-4a21-8f60-1c9de4a7b302" }
```

`messageId` is the message's identity. It appears in history as `externalId`, which is how you match
a send to its record and follow its delivery `status` — see [section 3](#3-conversation-items).

> **No idempotency key.** A retried send is a second message. If your client retries on timeout,
> keep your own guard so a network blip does not text a customer twice.

### Sending attachments

Attachments have to be uploaded to us first; a link to your own storage is refused. Upload each
file, then pass the returned URLs to `/send-sms`.

```http
POST /mms-attachment HTTP/1.1
Host: sms.voice.vylocloud.com
Authorization: Bearer <your token>
Content-Type: multipart/form-data
```

```bash
curl -X POST https://sms.voice.vylocloud.com/mms-attachment \
  -H "Authorization: Bearer $VYLO_TOKEN" \
  -F "file=@invoice.pdf"
```

```json
{
  "success": true,
  "url": "https://fs-storage.voice.vylocloud.com/media/public/mms/…pdf",
  "fileName": "…pdf",
  "contentType": "application/pdf",
  "size": 184320
}
```

Then send:

```bash
curl -X POST https://sms.voice.vylocloud.com/send-sms \
  -H "Authorization: Bearer $VYLO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "+13125550100",
    "to": "+13125550142",
    "text": "Invoice attached.",
    "mediaUrls": ["https://fs-storage.voice.vylocloud.com/media/public/mms/…pdf"]
  }'
```

`text` may be omitted for an attachment-only message.

### Attachment limits

| Type | Per file |
|---|---|
| JPEG, PNG, BMP, WebP, MP4, 3GPP | 5 MB |
| GIF, MP3, WAV, OGG, AMR, PDF, vCard, plain text | 600 KB |

Ten files per message, 10 MB in total. Anything larger, or a type not listed, is rejected on upload.
Not every number can send attachments — we will tell you which of yours can.

### Errors

| Response | Body |
|---|---|
| `400` | ``{"error":"Parameters `from` and `to` are required"}`` |
| `400` | ``{"error":"Either `text` or `mediaUrls` is required"}`` |
| `400` | `{"error":"At most 10 media attachments per message"}` |
| `400` | `{"error":"Media must be uploaded via /mms-attachment"}` |
| `401` | `{"error":"Unauthorized"}` — missing or invalid token |
| `403` | `{"error":"Forbidden"}` — `from` is not one of your numbers |
| `413` | Attachment exceeds the limit for its type |

A `400` is a problem with the request; do not retry it unchanged. Retry only on `5xx`, with backoff.
# vylo-api-docs
