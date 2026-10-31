# Limits, risks and the road to money

This page is the uncomfortable one. Read it before anyone proposes "just
let people buy coins".

## What this version does not do

| Missing | Effect today |
|:--|:--|
| Buying coins | Everyone has at most 100 per week. Nobody pays. |
| Withdrawing earnings | `earned` is a number on a screen and nothing more. |
| Push notifications for gifts | An author whose app is closed learns of a gift only the next time they open Notifications. |
| Showing a post's gifts by type | The API has `GET /feed/posts/:id/gifts`, but no screen uses it. |
| A wallet or history screen | The balance appears only inside the sheet. There is no list of gifts sent or received apart from Notifications (90 days). |
| Undo or refund | A gift cannot be taken back, by the sender or by support. |
| Gift names in other languages | Names come from the server in English. The surrounding strings are translatable. |
| Moderation of gifting | There are no limits per post or per pair of accounts, and no reporting of gift spam. Blocks are the only control. |

## Known gaps and inconsistencies

1. ~~**The landing burst ignored reduced motion.**~~ *Fixed in kyron PR #134
   (after v0.1.23).* It was a roughly 31 ms gold flash: Flutter shortens a
   controller to 5% of its length under reduced motion instead of skipping
   it. The landing is now not played at all.
2. ~~**Two stale comments said "emoji fallback".**~~ *Fixed in the same PR.*
   The catalogue comment, the artwork README and the cut tool (48 px tiles)
   now match the app.
3. **The server still sends `emoji`.** This is kept for app builds before
   v0.1.22. It can be removed once those builds are no longer supported.
4. **Gifting hides your like.** The gift replaces the heart icon, so whether
   you liked the post shows only in the colour of the count. Possible fix:
   a small red dot or ring on the gift when it is liked.
5. **`viewerGift` is your latest gift only.** Send a Fire and then a Spark,
   and the heart shows the Spark. That is intended, but a Rocket sender may
   not expect a later Spark to replace it.
6. **The week is UTC.** The refill arrives on Sunday evening in the
   Americas. Fine for a test; for money it needs a decision.
7. **`summary.total` and `byType` can disagree.** A type removed from the
   catalogue still counts in `total` but is not listed.
8. **The database tests do not run in CI.** A regression in the lock or the
   refill would pass CI. Add a Postgres service to the API job.

## Legal and app-store issues

This is engineering's reading of the problems, **not legal advice**. Take
it to counsel before anything here becomes money.

- **Apple (App Store Review Guidelines) and Google Play billing.** Coins
  that unlock anything digital inside the app must be sold through in-app
  purchase, with the store taking 15–30%. Guideline 3.1.1 explicitly allows
  IAP currencies to be used to "tip" digital content providers, which is
  this design. A separate rule (3.2.1) lets users send each other **money
  gifts outside IAP** only if 100% of the funds reach the recipient. That is
  a different design, and any margin for Kyron would rule it out. Check the
  current wording of both before choosing. **Payouts never go through the
  stores**: they need a separate payments provider.
- **Payouts mean money transmission.** Paying creators needs a provider that
  handles identity checks (KYC), tax forms (W-9/W-8BEN, 1099-K or local
  equivalents), sanctions screening and per-country rules. Kyron should not
  hold or move the money itself.
- **Virtual currency rules.**
  - Coins bought with money and not fully spent may be **unclaimed
    property** in some US states.
  - Consumer law in the EU and UK requires clear prices in real money, and
    refund rules for digital content.
  - Coins that can be **cashed out** are treated more strictly than coins
    that cannot.
  Keeping purchased coins and earnings in separate pools (as the ledger
  already does) helps.
- **Minors.** Spending by under-18s brings in parental-consent and
  child-protection rules (COPPA in the US, the UK Children's Code, the
  stores' own rules). Decide on an age gate for buying before launch.
- **Terms of service.** The live terms (linked from
  `app/lib/config/legal_links.dart`) need a virtual-goods section before
  coins are sold:
  - coins are a licence, not property;
  - no cash value except through the payout programme;
  - no transfer;
  - refunds;
  - what happens to coins and earnings when an account is banned or
    deleted;
  - Kyron's right to change prices and the catalogue.
  The test version relies on the notice in the sheet ("cannot be bought
  and have no cash value"). **The notice must stay until the terms cover
  coins.**
- **Fraud and laundering.** Paid gifts between colluding accounts are a
  well-known way to launder money and to cash out stolen cards. This needs
  velocity limits, holds on new earnings before payout, chargeback
  clawback (a negative entry in the ledger), and review of unusual pairs.
- **Artwork rights.** The gift pictures are AI-generated. Their copyright
  status is uncertain (see [catalogue](02-catalogue.md)). Commission
  redraws before they become paid products or brand assets.

## The road to money: what changes

The ledger was built so that these are additions, not rewrites.

| Step | Change |
|:--|:--|
| 1. Purchased coins | New `CoinKind` `PURCHASE` (positive), keyed by the store's transaction id: `purchase:<store>:<transactionId>`. The unique key makes a re-delivered receipt harmless. Receipts are verified server-side (App Store Server API, Play Developer API). |
| 2. Separate pools | Decide whether granted (free) coins and purchased coins are spent in an order (free first is common). The ledger can tell them apart by `kind`. |
| 3. Refunds and chargebacks | New kind `REVERSAL` (negative), keyed `reversal:<store>:<id>`, triggered by App Store Server Notifications and Play Real-time Developer Notifications. Decide what happens when the coins were already spent (a negative balance, or claw back from the recipient's earnings). |
| 4. Payouts | New kind `PAYOUT` (negative, on `earned`) keyed by the provider's transfer id. A minimum threshold and a holding period first. KYC must be complete before any payout. |
| 5. Value of earnings | Store a conversion rate per period (coins → currency), never computed from today's price. |
| 6. Ranking | Still none. Write it down again when money arrives, because the pressure to rank paid content will come. |
| 7. Push | Add `gift` to the push catalogue (`push/copy`) in every language, then send it with the socket event. |
| 8. CI | Add a Postgres service so `gifts.db.spec.ts` runs on every PR. |

## Open decisions

- Kyron's margin, if any, between what a buyer pays and what a creator
  earns, after the store's commission.
- Whether the weekly free coins continue once coins can be bought.
- Whether the refill should follow local time (it needs a time zone per
  account).
- Whether to show who gifted what on a post publicly, or only to its author.
- The age for buying coins.
