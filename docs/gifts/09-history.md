# History

How gifts were built, oldest first. Commits are on `KyronLabs/kyron`
`main`. Release notes are in its `CHANGELOG.md`.

| Date | Release | Change | Commit |
|:--|:--|:--|:--|
| 2026-10-29 | — | A security and legal audit of the whole project ran first. It shaped the gift design: RLS on the new tables, blocks answered as "not found", no ranking by money, and a test-only economy until the terms cover virtual goods. | — |
| 2026-10-29 | — | Design decision: payment through the app stores only. That works for **buying** coins (IAP), but the stores do not pay creators, so payouts need a separate provider. Hence a test version first. | — |
| 2026-10-29 | v0.1.20 | **Gifts on posts, with test coins.** The ledger, the lock, the weekly refill, four gifts (Spark 1, Fire 5, Gem 20, Crown 50), a gift button beside the like, the post's gift count, Notifications with `include=gifts`, socket notification. | `7415ea5` |
| 2026-10-29 | v0.1.20 | CI: the release build failed when the face-tracking plugin pulled a second NDK (26.3) that arrived corrupt. Fix: every plugin builds with the app's NDK 28.2, installed in its own step with retries. Then the dev build failed because that step ran from `app/`. Fix: run it from the repository root, and add a `check_script_paths` rule. | `7c296dc`, `760e0d5` |
| 2026-10-30 | v0.1.21 | **Gifts open from the heart.** The gift button removed and the row back to five actions. Long press on the like, a light haptic, the heart lifts, and the sheet shows prices and the balance. | PR #131 |
| 2026-10-30 | v0.1.22 | **Gifts look like gifts.** Eight AI-generated PNGs, with Rose 3, Star 10, Cake 30 and Rocket 100 added. A 4-column grid with prices and a coin under each gift, the balance top-right, a "sent" pop. The hold shortened from the platform's 500 ms to 350 ms. | `cd3d2c3`, PR #132 |
| 2026-10-30 | v0.1.23 | **Gifts land on the heart.** `viewerGift` on posts. Sparkle celebration, flight to the heart, the heart becomes the gift with a gold landing. Idle motion per gift. No emoji anywhere: the button, the fallback and notifications all use the PNGs. Haptics on pick, send and land. | `9d2d07a`, PR #133 |
| 2026-10-31 | — | This documentation. | — |

## Lessons recorded along the way

- **A concurrency test can pass for the wrong reason.** The first
  double-spend test passed with the lock removed, because the refill's
  unique key failed nine of the ten racing requests first. Always check
  that a safety test fails without the safety.
- **White on white cannot be cut out.** The first rocket had a white body.
  Generate artwork in colour on a white plate.
- **Listing an asset folder in `pubspec.yaml` ships everything in it**,
  README included. List the files.
- **Provenance has to be written down when the asset is made.** That is why
  `app/assets/gifts/README.md` exists.
