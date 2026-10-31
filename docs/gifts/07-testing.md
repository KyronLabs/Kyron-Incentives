# Testing

## API unit tests: `api/src/modules/gifts/gifts.service.spec.ts`

These run in CI on every PR, against a mocked Prisma client.

| Group | Test | What it proves |
|:--|:--|:--|
| sending a gift | refuses a gift that is not in the catalogue | 400 before any read |
| | refuses a post that is gone, or whose author is banned | 404 |
| | refuses a gift to your own post | 400 |
| | answers as if the post were not there when either has blocked the other | 404, not "blocked" |
| the weekly refill | starts the week on Monday, 00:00 UTC | `weekStart` |
| | counts a Sunday as the end of its week, not the start of the next | Sunday 23:59:59.999 belongs to the earlier week |
| | counts the first instant of Monday as the new week | boundary inclusive |
| the catalogue | is in price order, which is the order the sheet shows | |
| | keeps the gifts that have already been sent, at their prices | spark 1, fire 5, gem 20, crown 50 pinned |
| | has nothing a week of coins cannot buy | max price ≤ `WEEKLY_COINS` |
| | names every gift once | unique keys |

## API database tests: `api/src/modules/gifts/gifts.db.spec.ts`

These run against a **real, migrated Postgres**, and only when
`MODERATION_TEST_DATABASE_URL` is set. **CI has no database, so CI skips
them.** Run them locally before changing anything in the ledger, the lock or
the refill.

The scenario is sequential. Ada writes a post. Ben and Cy send her gifts.
The clock is fixed at Wednesday 28 Oct 2026 and then Tuesday 3 Nov 2026.

| # | Test | What it proves |
|--:|:--|:--|
| 1 | fills a new wallet to 100 coins, once | first read grants 100, and a second read grants nothing (exactly one `GRANT`) |
| 2 | moves coins from sender to author, and tells the author | Fire: Ben 95, Ada earned 5, socket event `{notification.new, gift, actorId, postId}` |
| 3 | does not let earnings be spent | Ada: coins 100, earned 5 |
| 4 | treats a retried request as the same gift | same id, same wallet, same count |
| 5 | refuses to reuse a request for a different gift | 409 |
| 6 | never spends the same coins twice, however many gifts arrive at once | 10 concurrent Crowns from 100, exactly 2 succeed, 0 left. **Fails without the lock.** The wallet is refilled on its own first, so the race is about spending and not about the refill's unique key |
| 7 | refuses a gift the wallet cannot cover, and says when it refills | message ends "Your coins refill every Monday." |
| 8 | refills once the week turns: up to 100, not on top of what is left | |
| 9 | refills a full wallet only once in a week, even after it spends | the zero-amount `GRANT` marker |
| 10 | counts each kind of gift a post was given | `{spark: 2, fire: 1, gem: 1, crown: 2}`, total 6 |
| 11 | tells the author, on the notifications screen of an app that asked | 6 gift rows with `include=gifts`, none without, unread ≥ 6 |
| 12 | tells each reader which gift they last sent a post, and nobody else's | `viewerGift`: Ben's latest, Cy `gem`, Ada (the author) null |
| 13 | keeps the author's earnings when a sender deletes their account | earned unchanged, 6 gifts remain, the deleted sender is not named, 3 rows left |

### Running them locally

1. Start a disposable Postgres 16. Port 55432 and a socket directory under
   `/var/tmp` worked in development.
2. Create the roles the migrations expect, from `infra/postgres/init/*.sql`.
   The gifts migration revokes from `anon` and `authenticated`, so those
   roles must exist.
3. Migrate:
   `DATABASE_URL=… npx prisma migrate deploy` (in `api/`).
4. Run:

```
MODERATION_TEST_DATABASE_URL='postgresql://postgres@localhost:55432/kyron_test?host=/var/tmp/kyron-pg' \
  npx jest src/modules/gifts/gifts.db.spec.ts
```

5. Throw the database away afterwards. The tests delete their own users,
   but nothing depends on keeping it.

`docs/MODERATION.md` in `kyron` describes the same setup for the moderation
database tests, which share the variable.

## App tests: `app/test/gifts_test.dart`

39 tests. All run in CI.

| Group | Tests |
|:--|:--|
| reading what the server sends | `viewerGift` parsed and kept by `copyWith`. `SentGift.type` parsed. Gift count defaults to 0. A gift notification row parses |
| what a gift row says | "sent a Fire to your post" (no emoji). Still makes sense without a gift |
| request ids | version 4 UUIDs. 1,000 generated, none repeated |
| holding the heart | five actions and no gift button. A hold opens with `lightImpact` and no like. No "Like" tooltip over the sheet. A tap still likes with `selectionClick`. The lift goes up and comes back to exactly 1×. `liftCurve` is 0 at both ends and peaks at 1. The screen reader hint is "Send a gift". A post you cannot gift is only a like |
| the sheet | the test-coins notice. The balance in the top right. The price under each gift. Artwork in four columns. Nothing sent until picked and pressed, and the Send button shows "Send Fire" with a coin. No emoji before or after sending. A sparkle burst (`GiftBurstPainter`) as it goes. Motions only on affordable gifts, and the sheet settles. Will not send what the wallet cannot cover. Shows "Fire sent", then closes with the result. A retry is the same request. A different gift is a new request |
| the artwork | every drawn type, and the coin, is in the bundle (> 1 KB). An unknown type draws a present, no text, no image |
| how long the heart is held | opens at 400 ms, not at 250 ms |
| the heart, once a gift is sent | draws the gift instead of the heart. Lands with a **gold** burst when it changes. No landing at all with reduced motion (checked inside the first 10 ms, because Flutter's shortened flash would be gone by 120 ms). No landing on a rebuild with the same gift |
| sending from the heart | **end to end**: hold → pick → send → the heart is still a heart mid-flight → it becomes the Fire, the count is 4, and the haptics are exactly `[lightImpact, selectionClick, lightImpact, selectionClick]`. With reduced motion, the heart changes with no flight |
| a gift notification | the PNG twice (badge and trailing), no gift icon, no emoji, "sent a Fire to your post" |

Run:

```
cd app && flutter test test/gifts_test.dart
```

## The rendering tool

`app/test/render_gift_sheet.dart` is not a test (its name does not end in
`_test`). It writes PNGs of the sheet, light and dark, before and after a
pick, and three frames of the send burst (120, 320 and 620 ms after the
send). Use it to review visual changes:

```
flutter test test/render_gift_sheet.dart --dart-define=shots=/tmp/shots
```

## What is not tested automatically

- **The flight on a real device**, especially for a post near the bottom of
  the screen, where the arc is shortest, and on a tablet.
- **The gift database tests in CI.** They need a Postgres, which CI does not
  have.
- **Load.** The lock serialises one account's sends. Different accounts do
  not contend. There is no load test.
- **The 900 ms celebration and the motions on low-end Android.** They are
  only checked in widget tests and rendered frames.
