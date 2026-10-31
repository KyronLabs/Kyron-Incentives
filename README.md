# Kyron Incentives

The incentive systems in Kyron: how readers reward posts they think are worth
it, and how that reward is held, counted and shown.

The one that exists today is **Gifts**. A reader holds the heart on someone
else's post, picks a gift and spends coins on it. The post's author is told,
and the gift lands on the reader's heart.

> **Status: test version (live since app v0.1.20, 29–30 October 2026).**
> Coins cannot be bought: every wallet is topped up to 100 each Monday at
> 00:00 UTC. Earnings cannot be withdrawn. Nothing in this system is money
> yet. The ledger is built the way money will need it, so that purchases
> and payouts can be added to it later rather than replacing it.

The code lives in [`KyronLabs/kyron`](https://github.com/KyronLabs/kyron).
This repository holds the documentation for it.

## The documents

| Document | What it covers |
|:--|:--|
| [Overview](docs/gifts/01-overview.md) | What gifts are, the rules in one page, and the decisions behind them |
| [Catalogue and artwork](docs/gifts/02-catalogue.md) | The eight gifts and their prices, the coin, where the pictures came from and how they were cut |
| [Coins and the ledger](docs/gifts/03-economy.md) | Wallets, the weekly refill, spending and earning, why a balance is a sum, worked examples |
| [Data model](docs/gifts/04-data-model.md) | Tables, columns, keys, indexes, foreign keys, row-level security, what happens when things are deleted |
| [API](docs/gifts/05-api.md) | Every endpoint, request and response, error, the lock, idempotency, notifications, `viewerGift` |
| [App](docs/gifts/06-app.md) | The hold, the sheet, the send, the flight, the heart, the motions, haptics, accessibility, strings, every timing |
| [Testing](docs/gifts/07-testing.md) | Every test, what it proves, and how to run them, including against a real database |
| [Limits, risks and the road to money](docs/gifts/08-risks-and-roadmap.md) | What this version does not do, legal and app-store issues, known inconsistencies, open decisions |
| [History](docs/gifts/09-history.md) | How it was built, change by change |

## The system in twelve facts

1. **Eight gifts**, sorted by price: Spark 1, Rose 3, Fire 5, Star 10, Gem 20, Cake 30, Crown 50, Rocket 100.
2. Every wallet is **topped up to 100 coins** once a week, on **Monday at 00:00 UTC**. It is a top-up, not an addition, so coins never pile up.
3. **Earned coins are a separate number** and can never be spent.
4. A balance is never stored. It is the **sum of an append-only ledger** (`CoinEntry`).
5. Every ledger entry has a **unique key**, so the same event cannot be recorded twice.
6. Every send carries an **app-made UUID** (`requestId`), so a retried request returns the original gift and charges nothing more.
7. Each wallet change **locks the sender's `User` row** (`SELECT … FOR UPDATE`), so concurrent sends cannot spend the same coins twice.
8. You **cannot gift your own post**, a deleted post, a banned author's post, or anyone you have blocked or who has blocked you. A block is answered as "not found".
9. Gifts are **shown on the post but never ranked on**: the feed does not read `giftCount`.
10. The author learns of a gift **over the socket and in Notifications**. There is no push yet.
11. In the app, **holding the heart for 350 ms** opens the sheet. A tap is still a like.
12. Once you have sent a post a gift, **that post's heart becomes the gift**: on every screen and after a restart.
