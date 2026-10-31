# API

Module: `api/src/modules/gifts/`.

| File | Role |
|:--|:--|
| `gifts.module.ts` | registers the controller and service. Imports `SupabaseTokenModule` so `AuthGuard` can be built |
| `gifts.controller.ts` | three routes, all behind `AuthGuard` |
| `gifts.service.ts` | wallet, refill, send, summary, lock |
| `gift-catalogue.ts` | `GIFT_CATALOGUE`, `GIFT_TYPES`, `giftType()`, `WEEKLY_COINS`, `weekStart()`, `nextRefill()` |
| `dto/send-gift.dto.ts` | validation of the send body |

Registered in `api/src/app.module.ts` as `GiftsModule`. Every route needs a
valid Supabase bearer token. Without one the answer is `401`.

The global rate limit (`@fastify/rate-limit`, per account when a bearer
token is present, otherwise per IP, over a 60-second window) applies. There
is no gift-specific limit.

---

## `GET /gifts`: the sheet in one call

It refills the wallet first if a new week has started. That is a write, in
the same locked transaction.

**200**

```json
{
  "catalogue": [
    { "type": "spark", "name": "Spark", "emoji": "✨", "coins": 1 },
    { "type": "rose",  "name": "Rose",  "emoji": "🌹", "coins": 3 },
    "… 8 entries, in price order"
  ],
  "wallet": {
    "coins": 93,
    "earned": 12,
    "weeklyCoins": 100,
    "refillsAt": "2026-11-02T00:00:00.000Z"
  }
}
```

---

## `POST /feed/posts/:id/gifts`: send one

`:id` must be a UUID (`ParseUUIDPipe`), otherwise `400`.

**Body**

| Field | Rule | Error message |
|:--|:--|:--|
| `type` | one of `GIFT_TYPES` | `type must be one of spark, rose, fire, star, gem, cake, crown, rocket.` |
| `requestId` | a version 4 UUID | `requestId must be a UUID.` |

**200**

```json
{
  "gift":   { "id": "…uuid…", "type": "fire", "coins": 5, "createdAt": "2026-10-28T12:00:00.000Z" },
  "wallet": { "coins": 95, "earned": 0, "weeklyCoins": 100, "refillsAt": "2026-11-02T00:00:00.000Z" },
  "gifts":  4
}
```

`gifts` is the post's gift count after this gift.

### Checks, in order

| # | Check | Answer |
|--:|:--|:--|
| 1 | `type` is in the catalogue (checked again by the service, after the DTO) | `400 There is no such gift.` |
| 2 | The post exists, `deletedAt` is null, and its author is not `BANNED` | `404 Post not found.` |
| 3 | The author is not the sender | `400 You can't send a gift to your own post.` |
| 4 | Neither has blocked the other | `404 Post not found.` (deliberately the same as a missing post) |
| — | *the transaction begins* | |
| 5 | Lock the sender's `User` row (`SELECT 1 … FOR UPDATE`) | waits for any other change to this wallet |
| 6 | `(senderId, requestId)` already exists, for the same post and type | `200` with that gift and the current wallet. **No charge** |
| 6b | It exists, but for a different post or type | `409 That request was already used for a different gift.` |
| 7 | Refill if a new week has started | |
| 8 | `coins ≥ price` | `400 A {Name} costs {price} coins and you have {coins}. Your coins refill every Monday.` |
| 9 | Create the `Gift`, `SPEND`, `EARN`, and `giftCount + 1` | |
| — | *commit* | |
| 10 | Only if the gift was newly created: socket event to the author | |

### Why the lock comes before everything else

The lock is taken before the idempotency check and the balance read. A
second request for the same wallet, whether a retry or a different gift,
waits at the lock and then reads what the first one wrote. Without the lock,
two concurrent sends can both read 100 and both spend. The test sends ten
Crowns at once from 100 coins: exactly two succeed, and it fails if the
lock is removed.

An earlier version of that test passed even without the lock. The ten
sends were racing to write the week's refill, nine lost on its unique key,
and the double spend was never exercised. The fix was to refill the wallet
on its own first, so the race is purely about spending.

### Idempotency in practice

The app creates one `requestId` when a gift is picked and keeps it until
the send succeeds. If the connection drops after the server committed but
before the answer arrived, pressing Send again returns the same gift and
the same wallet: one charge. Picking a different gift creates a new
`requestId`.

### The socket event

On a new gift (never on a replayed one):

```json
{ "type": "notification.new", "kind": "gift", "actorId": "<sender>", "postId": "<post>" }
```

It is sent to the author over the realtime socket (`DeliveryService.tell`).
The app's `realtime_provider.dart` invalidates the unread badge and refreshes
any Notifications list that is already built. **There is no push**: a push
needs its text in every language of the push catalogue (`push/copy`), which
the test version does not have.

---

## `GET /feed/posts/:id/gifts`: what a post was given

**200**

```json
{
  "total": 6,
  "byType": [
    { "type": "spark", "name": "Spark", "emoji": "✨", "coins": 1,  "count": 2 },
    { "type": "fire",  "name": "Fire",  "emoji": "🔥", "coins": 5,  "count": 1 },
    { "type": "gem",   "name": "Gem",   "emoji": "💎", "coins": 20, "count": 1 },
    { "type": "crown", "name": "Crown", "emoji": "👑", "coins": 50, "count": 2 }
  ]
}
```

- `total` is `Post.giftCount`. `byType` lists only types with a count above
  0, in catalogue order. A type no longer in the catalogue is counted in
  `total` but not listed.
- `404` if the post is deleted, its author is banned, or (when the viewer is
  not the author) either side has blocked the other.
- **The app does not call this yet.** It exists for a future breakdown on a
  post's own screen.

---

## Gifts in other responses

### Feed and post responses (`FeedPost`)

| Field | Type | Meaning |
|:--|:--|:--|
| `gifts` | number | `Post.giftCount` |
| `viewerGift` | string \| null | the catalogue `type` of the **latest gift this reader sent this post**, or null |

`viewerGift` is read in `FeedService.shapeFor(viewerId)` with a nested
select: the post's gifts where `senderId = viewer`, ordered by `createdAt
desc`, `take 1`. That is one row per post, using the `(postId, type)`
index. It is present on every endpoint that shapes posts through
`shapeFor`: the feeds and a single post among them. Each
reader sees only their own gift, and the author of a post sees null on it.
This is tested against a real database.

### Notifications (`GET /notifications`, `GET /notifications/unread`)

Gift rows appear only when the client asks for them with
`include=gifts`. The app sends `include=moderation,mentions,gifts`. This
opt-in exists because an app released before gifts would read an unknown
row as "Someone liked your post".

A gift row:

```json
{
  "id": "gift:<giftId>",
  "kind": "gift",
  "actor": { "id": "…", "name": "Ben", "…": "…" },
  "postId": "…",
  "postSnippet": "Something worth a gift",
  "content": null,
  "createdAt": "…",
  "unread": true,
  "gift": { "type": "fire", "coins": 5, "name": "Fire", "emoji": "🔥" }
}
```

Rules:

- **Window.** Only the last 90 days (`WINDOW_DAYS`). Each kind fetches up to
  the page limit (default 30) before the kinds are merged.
- **Excluded senders.** Rows are left out when `senderId` is null (the
  sender's account was deleted), when the sender is the reader, or when
  either side has blocked the other.
- **Removed types.** A type no longer in the catalogue is named "Gift" (and
  `emoji` falls back to 🎁 for older apps).
- **Unread** follows the existing `notificationsSeenAt` rule.
  `/notifications/unread?include=gifts` counts gifts since then, with the
  same exclusions.
