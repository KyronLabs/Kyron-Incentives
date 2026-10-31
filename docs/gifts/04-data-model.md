# Data model

Migration: `api/prisma/migrations/20261104090000_gifts/migration.sql`.
Schema: `api/prisma/schema.prisma` (enum `CoinKind`, models `CoinEntry` and
`Gift`, and `Post.giftCount`).

The migration's timestamp (4 November) is later than the date it shipped
(29 October). That is deliberate: it was renamed to avoid colliding with
another migration and to sort after the migrations already on `main`.

## Enum `CoinKind`

| Value | Sign | Meaning |
|:--|:--|:--|
| `GRANT` | positive (or 0) | test coins from the weekly refill |
| `SPEND` | negative | coins spent on a gift |
| `EARN` | positive | what a post's author received for a gift |

## Table `CoinEntry`: the ledger

| Column | Type | Null | Notes |
|:--|:--|:--|:--|
| `id` | UUID | no | primary key |
| `userId` | UUID | no | whose wallet. FK → `User.id`, **ON DELETE CASCADE** |
| `kind` | `CoinKind` | no | |
| `amount` | INTEGER | no | signed, see the enum |
| `key` | TEXT | no | **UNIQUE**. `grant:<user>:<week>`, `gift:<id>:spend`, `gift:<id>:earn` |
| `giftId` | UUID | yes | FK → `Gift.id`, **ON DELETE SET NULL** |
| `createdAt` | TIMESTAMPTZ | no | default `CURRENT_TIMESTAMP` |

Indexes: `CoinEntry_pkey (id)`, `CoinEntry_key_key UNIQUE (key)`, and
`CoinEntry_userId_kind_idx (userId, kind)`. The last serves the balance
`groupBy`.

## Table `Gift`

| Column | Type | Null | Notes |
|:--|:--|:--|:--|
| `id` | UUID | no | primary key |
| `senderId` | UUID | **yes** | FK → `User.id`, **ON DELETE SET NULL**. Null once the sender deletes their account |
| `recipientId` | UUID | no | the post's author. FK → `User.id`, **ON DELETE CASCADE** |
| `postId` | UUID | no | FK → `Post.id`, **ON DELETE CASCADE** |
| `type` | TEXT | no | a catalogue key. Not an enum, so the catalogue can change without a migration |
| `coins` | INTEGER | no | the price paid, kept so later price changes do not rewrite history |
| `requestId` | UUID | no | made by the app per press |
| `createdAt` | TIMESTAMPTZ | no | default `CURRENT_TIMESTAMP` |

Indexes:

| Index | Columns | Serves |
|:--|:--|:--|
| `Gift_pkey` | `id` | |
| `Gift_senderId_requestId_key` UNIQUE | `(senderId, requestId)` | idempotent retries |
| `Gift_postId_type_idx` | `(postId, type)` | the per-post summary, and the reader's `viewerGift` lookup (by `postId`) |
| `Gift_recipientId_createdAt_idx` | `(recipientId, createdAt DESC)` | Notifications, newest first |

A Postgres detail: because `senderId` is nullable, the unique index treats
every `NULL` as distinct. Two gifts whose senders have both been deleted
never collide. That is harmless, because no new gift can be created with a
null sender.

## Column `Post.giftCount`

`INTEGER NOT NULL DEFAULT 0`. It is incremented by one per gift, in the same
transaction as the gift. It is shown on the post (as `gifts` in API
responses) and **deliberately not read by ranking**. It is never
decremented: there is no un-gift.

## Relations on `User`

`giftsSent` (relation "GiftSender"), `giftsReceived` (relation
"GiftRecipient"), and `coinEntries`.

## Who can read these tables

The migration ends with:

```sql
REVOKE ALL ON TABLE "CoinEntry" FROM anon, authenticated;
REVOKE ALL ON TABLE "Gift"      FROM anon, authenticated;
ALTER TABLE "CoinEntry" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "Gift"      ENABLE ROW LEVEL SECURITY;
```

The database is Supabase, whose anon and authenticated roles can query
tables directly with the public key. Without these lines, every account's
balance and every gift would be readable from the client. Wallets are
reachable only through the API, which connects as the owning role.

## What deletion does

| Deleted | `Gift` rows | `CoinEntry` rows | Effect |
|:--|:--|:--|:--|
| **Sender's account** | stay, `senderId` → NULL | the sender's own entries cascade away | The author keeps their earnings. Notifications drop the row because there is nobody to name. `summary` totals still count it. Tested. |
| **Recipient's account** | cascade away (by `recipientId`) | the recipient's entries cascade. The sender's `SPEND` stays, with `giftId` → NULL | The sender's spend stays spent. |
| **Post, hard-deleted** | cascade away | both entries stay, with `giftId` → NULL | Spend and earn both stand. |
| **Post, soft-deleted** (`deletedAt`) | stay | stay | No new gifts can be sent to it. Existing ones stand. |
| **Author banned** | stay | stay | No new gifts can be sent, and the summary answers 404. |

The invariant behind all of these: an entry leaves the ledger only with its
own account. One account's deletion never changes another account's
balance.
