# Coins and the ledger

## Three numbers per account

The API reports a wallet as:

| Field | Meaning | Computed as |
|:--|:--|:--|
| `coins` | What can be spent now | sum of `GRANT` + sum of `SPEND` (`SPEND` amounts are negative) |
| `earned` | What this account's posts have been given | sum of `EARN` |
| `weeklyCoins` | What the refill tops `coins` up to | the constant `WEEKLY_COINS` = 100 |
| `refillsAt` | When the next refill can happen | the Monday 00:00 UTC after now |

`earned` never joins `coins`. An account with 5 earned and 100 coins can
spend 100, not 105. This is tested ("does not let earnings be spent").

## The ledger: `CoinEntry`

Every movement of coins is one row, and rows are never updated. A balance
is not stored anywhere: it is computed with a `groupBy(kind)` / `SUM(amount)`
over the account's entries each time it is needed.

| `kind` | Sign | Written when | `key` (unique) |
|:--|:--|:--|:--|
| `GRANT` | ≥ 0 | the weekly refill, first wallet read or send of the week | `grant:<userId>:<YYYY-MM-DD of that Monday>` |
| `SPEND` | < 0 | a gift is sent | `gift:<giftId>:spend` |
| `EARN` | > 0 | a gift is received | `gift:<giftId>:earn` |

The unique `key` is the guarantee that nothing is ever recorded twice. If
two transactions both try to write this week's grant, or a gift's spend,
the second hits the unique index and its transaction fails.

## The weekly refill

When it runs: lazily, inside the same transaction as `GET /gifts` (reading
the wallet) and `POST …/gifts` (sending). Nothing runs on a schedule.

What it does:

1. Work out this week's key: `grant:<user>:<Monday date>`.
2. If an entry with that key already exists, stop.
3. Otherwise read the current `coins` and write a `GRANT` of
   `max(0, 100 − coins)`, **even when that is 0**.

Why a zero grant is written: without it, an account that started the week
full would get no grant entry, and its first gift would trigger the refill
again, topping it back to 100. That would repeat on every gift. The
zero-amount entry marks the week as done. This is the test "refills a full
wallet only once in a week, even after it spends".

### The week boundary

`weekStart(at)`:

```
day        = midnight UTC of `at`'s UTC date
sinceMonday = (day.getUTCDay() + 6) % 7      // Sunday 0 → 6, Monday 1 → 0
weekStart  = day − sinceMonday days
```

`nextRefill(at)` = `weekStart(at)` + 7 days.

| Instant (UTC) | Week starts | Next refill |
|:--|:--|:--|
| Wed 28 Oct 2026 12:00 | Mon 26 Oct 00:00 | Mon 2 Nov 00:00 |
| Sun 1 Nov 2026 23:59:59.999 | Mon 26 Oct 00:00 | Mon 2 Nov 00:00 |
| Mon 2 Nov 2026 00:00:00.000 | Mon 2 Nov 00:00 | Mon 9 Nov 00:00 |

The app shows `refillsAt` in the reader's local time and locale, for
example "Refills Mon, Nov 2." In UTC−5 that is a Sunday evening, because the
instant is the same everywhere.

## Spending

A send, in order, inside one transaction:

1. Lock the sender's `User` row.
2. If `(senderId, requestId)` already exists, return that gift (see the
   [API](05-api.md)). Nothing is charged.
3. Refill if a new week has started.
4. Compute `coins`. If it is below the price, refuse with "A Fire costs 5
   coins and you have 3. Your coins refill every Monday."
5. Create the `Gift` (with the price paid, `coins`).
6. Write two entries: `SPEND −price` for the sender and `EARN +price` for
   the author.
7. Increment `Post.giftCount`.
8. Read back the answer: the gift, the sender's new wallet, and the post's
   new gift count.

## Worked example

Wednesday 28 Oct, Ben opens the sheet for the first time:

| Step | Entry written | Ben `coins` | Ada `earned` |
|:--|:--|--:|--:|
| Ben opens the sheet | `GRANT +100` `grant:ben:2026-10-26` | 100 | 0 |
| Ben sends Fire to Ada's post | `SPEND −5` (Ben), `EARN +5` (Ada) | 95 | 5 |
| Ben retries the same request | nothing | 95 | 5 |
| Ben sends a Spark | `SPEND −1`, `EARN +1` | 94 | 6 |
| Ada opens the sheet | `GRANT +100` `grant:ada:2026-10-26` | (Ada: 100) | 6 |
| Tue 3 Nov, Ben opens the sheet | `GRANT +6` `grant:ben:2026-11-02` | 100 | 6 |

Cy with 100 coins sends ten Crowns (50 each) at the same instant. The lock
serialises them: two succeed, eight are refused, and Cy ends with 0. This
is tested against a real Postgres, and the test fails without the lock.

## What the economy deliberately is not

- **Not purchasable.** There is no store product, price or receipt check.
- **Not withdrawable.** `earned` is display-only.
- **Not transferable.** Coins move only as gifts, and gifts only to a
  post's author.
- **Not a ranking signal.** See the [overview](01-overview.md).
- **Not refundable.** There is no undo or refund. A gift on a post that is
  later deleted stays in the ledger, and so do both entries.

The [roadmap](08-risks-and-roadmap.md) covers what changes when coins
become money.
