# Gifts: overview

## What a gift is

A gift is a small, visible thank-you that a reader sends to a post: a Spark,
a Rose, Fire and so on. It costs the sender coins and credits the post's
author with the same number of coins, as **earnings**. The author sees who
sent what in Notifications. The sender sees their gift take the place of the
heart on that post.

## The rules, in one place

| Rule | Value | Where it is enforced |
|:--|:--|:--|
| Who can send | Any signed-in account | `AuthGuard` on every gifts route |
| To whom | The author of a live post that is not the sender's | `GiftsService.send` |
| Not to | Your own post, a deleted post (`deletedAt` set), a post by a `BANNED` account, or an author either side has blocked | `GiftsService.send` |
| Gifts available | 8 types, fixed in code | `api/src/modules/gifts/gift-catalogue.ts` |
| Prices | 1, 3, 5, 10, 20, 30, 50, 100 coins | the same file |
| Starting coins | 100 | `WEEKLY_COINS` |
| Refill | Topped up to 100, once per week | `GiftsService.refill` |
| Week boundary | Monday 00:00:00.000 UTC, the same instant for everybody | `weekStart()` |
| Can coins be bought? | **No** (test version) | there is no purchase path |
| Can earnings be spent? | **No.** Earned and spendable coins are kept separate | `GiftsService.balances` |
| Can earnings be withdrawn? | **No** (test version) | there is no payout path |
| Retries | The same `requestId` returns the same gift, at no extra charge | unique `(senderId, requestId)` |
| Double spending | Impossible: each send locks the sender's row | `GiftsService.lock` |
| Effect on ranking | **None** | the feed's ranking never reads `giftCount` |
| Author notified | Socket event `notification.new` (kind `gift`) and a Notifications row | `DeliveryService.tell`, `NotificationsService` |
| Push notification | **None yet** | see [risks](08-risks-and-roadmap.md) |

## The flow, end to end

```
Reader holds ♥ (350 ms)
   │  lightImpact haptic, heart lifts
   ▼
Gift sheet ── GET /gifts ─────────────► catalogue + wallet (refilled first if a new week began)
   │  tiles animate; tap picks (selectionClick); "Send Fire 🪙 5"
   ▼
POST /feed/posts/:id/gifts {type, requestId}
   │   server: checks → lock sender row → same requestId? return it
   │           → refill → balance ≥ price? → Gift + SPEND + EARN → giftCount+1
   │           → socket "notification.new" to the author
   ▼
Sheet celebration (900 ms): spring, wobble, glow, sparkles   lightImpact
   ▼
Sheet closes; the gift flies (720 ms) along an arc onto the heart
   ▼
Lands: selectionClick, gold burst, the heart is now the Fire
   ▼
Author: badge count +1, a Notifications row "Ada sent a Fire to your post" with the Fire's picture
```

## Decisions and why they were made

**A ledger, not a balance column.** A balance stored in a column can drift,
be overwritten by a race, or be edited with no trace. A sum of immutable
entries cannot be wrong without a wrong entry, and every entry says where it
came from. This is what real money will need, and adding it later would be
the expensive part.

**A top-up refill, not an addition.** If every week added 100, an account
that sends nothing would hold 1,000 coins after ten weeks and could flood
one post at once. Topping up to 100 caps every wallet at one week's
allowance, which is exactly one Rocket. "One gift a week can be the big one"
is the intended scarcity.

**Monday 00:00 UTC for everyone.** Kyron does not store a time zone per
account, and a refill at local midnight would need one. A single global
instant is simple to reason about and to test. The cost is that the refill
arrives on Sunday evening in the Americas and Monday morning in Asia.

**Earnings are a separate number.** If earnings could be spent, two
accounts could pass coins back and forth to look generous or to farm
whatever a future payout pays for. Keeping them apart now means the paid
version decides what earnings are worth, without first unwinding a test
economy.

**Gifts do not move the feed.** The README of `kyron` promises a feed that
money cannot buy. Gifts are shown (the count on a post's own screen, the
heart, the notification) but `giftCount` is deliberately not an input to
ranking. This is written on the column itself in the Prisma schema.

**Blocks answer "not found".** Answering "you are blocked" would confirm
the post exists and that the block is there. The feed answers the same way.

**The catalogue lives in code, not a table.** Eight rows that change only
with a release do not need an admin screen, and a price change should be
visible in code review. `Gift.type` stores the key and `Gift.coins` stores
the price paid, so a later price change never rewrites history.

**There is no gift button. The heart is held.** An earlier version had a
dedicated gift button in the row. It was removed so the row kept its
original five actions, and gifting became a long press on the like: a light
haptic, a small lift of the heart, then the sheet.

**Pictures, never emoji.** Emoji look different on every phone and font, and
read as a placeholder. Every gift is a PNG made for Kyron. Where the app has
no picture for a type, it draws a gold present icon, not an emoji. The
server still sends an `emoji` field, but only for app builds released before
this change.

**A gift lands on the heart.** Sending changes the post visibly. The gift
flies out of the sheet onto the heart and replaces it, for that reader, from
then on.
