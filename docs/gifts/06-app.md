# App

Flutter, Riverpod 3, and the `kyron_design_system` tokens. Every file
involved:

| File | Role |
|:--|:--|
| `lib/models/gift.dart` | `GiftType`, `GiftWallet`, `GiftSheetData`, `SentGift`, `GiftNote` |
| `lib/models/feed_post.dart` | `gifts` (count) and `viewerGift` (your latest gift's type), in `fromJson` and `copyWith` |
| `lib/repositories/gifts_repository.dart` | `sheet()`, `send()`, `newRequestId()` |
| `lib/providers/gifts_provider.dart` | `giftsRepositoryProvider` |
| `lib/widgets/post_actions_row.dart` | the heart: hold-to-gift, lift, drawing the gift in its place, the landing burst |
| `lib/widgets/gift_sheet.dart` | `GiftSheet.show` / `.open`, the sheet, `GiftArt`, `CoinIcon`, `coinLabel`, the celebration |
| `lib/widgets/gift_motion.dart` | `GiftMotion` (idle motions), `giftPose`, `GiftBurstPainter`, `GiftFlight`, the trail, `sparklePath`, `giftSparkleColours` |
| `lib/widgets/like_burst.dart` | `LikeBurstPainter`, `likePunch`, `unlikePunch` (shared with likes) |
| `lib/widgets/post_card.dart`, `lib/screens/post_detail_screen.dart` | wire `onGift` to `GiftSheet.open`, and `onSent` to the providers |
| `lib/providers/feed_provider.dart` | `giftsChanged(postId, gifts, {type})` |
| `lib/providers/post_detail_provider.dart` | `giftsChanged(gifts, {type})`, which also updates the Recent feed's copy |
| `lib/models/notification_model.dart` | `NotificationType.gift`, `GiftNote` parsing |
| `lib/utils/notification_text.dart` | "sent a Fire to your post" |
| `lib/widgets/notification_item.dart` | the gift row: the PNG as a badge and as a trailing picture |
| `lib/repositories/notifications_repository.dart` | asks for `include=moderation,mentions,gifts` |
| `lib/providers/realtime_provider.dart` | `notification.new` refreshes the badge and the lists |
| `assets/gifts/*.png`, `pubspec.yaml` | the artwork, listed one file at a time |

---

## 1. Opening the sheet: holding the heart

The row keeps its five actions (reply, repost, like, save, share). There is
**no gift button**. The like is the way in.

| Gesture | Result | Haptic |
|:--|:--|:--|
| Tap | like or unlike, with the like punch and burst | `selectionClick` |
| Hold **350 ms** (`holdToGift`) | the heart lifts, and the sheet opens | `lightImpact` |
| Hold under 350 ms | nothing (a slow tap is not a gift) | |

- **Why 350 ms.** The platform's long press is 500 ms, which felt like a
  wait. Much shorter, and a slow like turns into a gift sheet. Tested at 400
  ms (opens) and 250 ms (does not open).
- **How.** A `RawGestureDetector` with its own
  `LongPressGestureRecognizer(duration: holdToGift)` wraps the button, so
  the InkWell's own long press (and Android's default buzz) never fires.
- **The lift.** A separate 320 ms controller moves the icon up by
  `2.5 × liftCurve(t)` px and scales it by `1 + 0.18 × liftCurve(t)`.
  `liftCurve`: ease-out-cubic up over the first 35%, held at 1 until 50%,
  then ease-in-out back to 0. It is exactly 0 at both ends, so the icon
  returns to where it was.
- **Tooltip.** On the holdable heart, the "Like" tooltip's trigger mode is
  `manual`, so it does not pop up over the sheet. A mouse hover still shows
  it.
- **Where gifts cannot be sent** (your own post, `onGift: null`), a long
  press is the tooltip's again.
- **Accessibility.** The detector has `excludeFromSemantics: true`, and a
  `Semantics` wrapper gives `onLongPress` with the hint **"Send a gift"**,
  so a screen reader offers the hold as an action named "Send a gift"
  (tested with `matchesSemantics`).
- **What the sheet is told.** `onGift` receives the heart **icon's** own
  `BuildContext` (via a `GlobalKey` on a `KeyedSubtree` around the icon),
  not the button's. That is the flight's target. The button includes the
  count.

## 2. The sheet

`GiftSheet.open(heart, post, onSent:)` calls `showAppSheet`
(`isScrollControlled: true`). On open it pre-caches all eight PNGs while
`GET /gifts` loads, so the grid arrives complete rather than filling in tile
by tile.

States: loading (260 px tall, with a spinner) → loaded, or failed (220 px,
the error text and a **Try again** button that reloads).

Layout, top to bottom, with 20 px side and bottom padding:

1. **Header row.** "Send a gift" (`fontSize6`, w700) on the left and the
   **balance pill** in the top-right corner: `CoinIcon(22)` and the number
   (`fontSize3`, w700, tabular figures), on `onSurface` at 6% alpha, fully
   rounded. Its screen-reader label is "You have 93 coins".
2. 16 px.
3. **Grid.** 4 columns, 8 px gaps, `childAspectRatio 0.82`, not
   scrollable. Each tile:
   - background `onSurface` 3.5%, or `primary` 10% with a 1.5 px `primary`
     border when picked, radius `radiusLg`, animated (`MotionTokens.fast`,
     ease-out-cubic);
   - the gift's `GiftArt` at **48 px**, inside its `GiftMotion`, scaled to
     **1.12** when picked (ease-out-back);
   - 8 px, then the price: `CoinIcon(14)`, 3 px, and the number
     (`fontSize1`, w600, 80% alpha, tabular);
   - **unaffordable** tiles are drawn at **40% opacity** but stay visible
     and tappable to pick, because knowing what a Rocket costs is how
     someone decides to save for one;
   - screen-reader label "Fire, 5 coins", a button, `selected` when picked.
4. 12 px.
5. **Notice, every time:** "Test coins: they cannot be bought and have no
   cash value. Refills Mon, Nov 2." (`fontSize1`, 55% alpha). The date is
   `DateFormat.MMMEd` in the reader's locale.
6. The error text, if the last send failed (`fontSize2`, `error`).
7. 16 px, then the **Send button** (`FilledButton`):

   | State | Label | Enabled |
   |:--|:--|:--|
   | nothing picked | Pick a gift | no |
   | picked, affordable | **Send Fire** · `CoinIcon(16)` · **5** (read as "Send Fire, 5 coins") | yes |
   | picked, not affordable | Not enough coins | no |
   | sending | an 18 px spinner | no |

**Two steps on purpose.** A tap picks and the button sends. If a tap spent
coins, a scroll that landed on a tile would spend them.

**Picking** another gift plays `selectionClick`, makes a new `requestId`
(`GiftsRepository.newRequestId()`: 16 bytes from `Random.secure()`, version
4 and RFC 4122 variant bits set) and clears the error. Re-picking the same
gift keeps the `requestId`.

**A failed send** keeps the sheet open with the error, and the same
`requestId`, so pressing Send again is the same request and never a second
gift.

## 3. Idle motion: `GiftMotion`

Each tile's picture moves while the sheet waits, for **3 rounds of 2.4 s
(7.2 s)**, then rests. **Picking replays it.** Gifts the wallet cannot cover
do not move. With reduced motion on (`MediaQuery.disableAnimations`), none
move.

- **Envelope.** The motion's strength is `min(1, min(v, 1−v) × 14)` over the
  whole run, so it eases in from rest and back to rest rather than
  snapping.
- **Phase offset** per gift, `(type.hashCode % 1000) / 1000`, so eight
  pictures do not move in step.
- **Why it stops.** A grid of eight things moving forever cannot be read,
  and a screen that never settles costs battery (and would never let a
  widget test settle).

`giftPose(type, phase, strength)`, with `w = 2π·phase` and `s = strength`:

| Gift | Motion | Transform |
|:--|:--|:--|
| spark | twinkle | rotate `0.14·sin(w)·s`, scale `1 + 0.10·sin(2w)·s` |
| rose | sway from the stem | rotate `0.11·sin(w)·s` about the bottom centre |
| fire | flicker from the base | `f = 0.6·sin(3w) + 0.4·sin(7w)`, scale x `1 − 0.04·f·s`, y `1 + 0.08·f·s`, bottom centre |
| star | turn | perspective `0.002`, rotateY `0.65·sin(w)·s` |
| gem | glint | `ShaderMask` (srcATop), a white diagonal band at alpha `0.6·s` sweeping once per round (`sweep = phase·1.6 − 0.3`, band ±0.12), and scale `1 + 0.04·sin(w)·s` |
| cake | hop and squash | up `3.5·|sin w|·s` px, squash `±0.04·(1−hop)·s` at the base |
| crown | tilt and bob | rotate `0.13·sin(w)·s`, up `1.5·sin(2w)·s` px |
| rocket | hover with engine shake | x `0.6·sin(11w)·s`, y `−3.5·sin(w)·s` px, rotate `0.05·sin(2w)·s` |
| any other | gentle bob | up `2.5·sin(w)·s` px |

## 4. Sending: the celebration

On success:

1. `lightImpact`: the moment the coins are spent.
2. The sheet's content fades to **15%** opacity and ignores touches.
3. `_Sent` appears in the middle, a 224 px stage with the gift at **112
   px**, for **900 ms** (`_celebration`):
   - **Spring.** Scale `0.35 + 0.65 × elasticOut(t / 0.55)`: in with a
     bounce over the first 55%.
   - **Wobble.** Rotate `0.22 × sin(3πt) × (1 − t)`, which dies away.
   - **`GiftBurstPainter`** behind it:
     - **Glow.** A radial gradient of `#FFD66B` at up to 70% alpha. It
       rises fast (ease-out over the first 40%) and fades (ease-in), with a
       radius of `reach × (0.45 + 0.25t)`.
     - **14 sparkles.** Four-point stars (`sparklePath`, waist 0.28 of the
       radius) thrown out to `reach × (0.32 + 0.6·easeOutCubic(t))`, with
       angles jittered by `0.3·sin(7i)`. Radius `(6 or 10) × fade ×
       twinkle`, where `twinkle = 0.65 + 0.35·sin(14t + i)` and `fade = 1 −
       easeInQuad(t)`.
     - **18 motes.** Dots of radius `2.4 × (1 − late)` that start at 15%,
       travel to `reach × (0.5 + 0.48·drift)`, and fall 6 px.
     - **Colours** (`giftSparkleColours`): coin gold `#F2B233`, light gold
       `#FFD66B`, orange `#FF8A3D`, pink `#FF5C8A`. These are warm on
       purpose, so a burst never reads as an error.
   - The label **"Fire sent"** (`fontSize5`, w700) fades in between 25% and
     60%.
4. The sheet measures the 112 px picture's on-screen rectangle, hides it
   (`launched`), and closes with `_Outcome(sent, gift, from)`.

With reduced motion, `_Sent` is drawn still, at its final state, for the
same 900 ms.

## 5. The flight: `GiftFlight`

Inserted into the **root overlay**, so it passes over the closing sheet and
over whatever screen the post is on. Pointer events are ignored.

| Property | Value |
|:--|:--|
| Duration | **720 ms** |
| From | the centre of the sent picture's rectangle (where the sheet let go of it) |
| To | the centre of the heart icon (`localToGlobal` of its render box) |
| Path | quadratic Bézier, with the control point at `x = from + 15%·(to − from)`, `y = min(from, to) − 90 px`: thrown up and over, never slid |
| Position easing | ease-in-out-cubic |
| Size | from the picture's width (112) to **21 px**, ease-in-cubic, so it stays big and then shrinks late |
| Spin | `0.45 × sin(π·t)` rad: a turn that comes back upright |
| Fade | fully opaque until 92%, then out over the last 8%, as the heart takes its place |
| Trail | 7 sparkles at points `0.045` apart in time behind it. Radius `2 + 4 × (1 − (k+1)/8)` px, alpha shrinking toward the tail, the whole trail fading to 60% by the end |

On arrival (or if the overlay is torn down early) `onArrive` runs **exactly
once**: `selectionClick`, then `onSent(sent)`.

**No flight** (and `onSent` is called at once) when:

- reduced motion is on;
- the heart's context is no longer mounted (the screen was left while the
  sheet was up);
- its render box is missing or detached;
- the sheet could not measure where the picture was.

## 6. The heart becomes the gift

`onSent` calls the provider:

- **Feed card:** `notifier.giftsChanged(post.id, sent.gifts, type: sent.type)`.
- **Post screen:** `notifier.giftsChanged(sent.gifts, type: sent.type)`,
  which also updates the copy in the Recent feed.

Both set `FeedPost.gifts` and `FeedPost.viewerGift`. Nothing is predicted
and nothing is rolled back: the sheet answers only after the server has
committed.

`PostActionsRow` then draws the like with `art: GiftArt(type: viewerGift,
size: 21)` in place of the 17 px heart icon:

- A **tap is still a like.** The count beside it is still the like count,
  and its colour still shows liked (red) or not.
- **The landing.** When `artKey` (the type) changes on a live widget, the
  like's 620 ms `_play` controller runs the `likePunch` curve (dip to 0.84,
  overshoot, settle) with the like burst painted in **gold `#F2B233`**
  instead of the like's red, spilling 15 px past the icon.
- **No false landings.** The like action is keyed `like-<postId>`. A list
  that reuses a row for a different post builds it fresh, so no landing is
  played for a gift that never flew. A rebuild with the same gift does not
  play it either (tested).
- **It persists.** `viewerGift` comes from the server on every feed and
  post response, so the gift shows on every screen, on every device, and
  after a restart. It is **your** latest gift. Other readers see their own,
  or the heart.

The post's own screen also spells out the count, "4 gifts", beside likes
and replies once it is above 0. The feed card shows no gift count.

## 7. Notifications

- The list and the unread count ask for `include=moderation,mentions,gifts`.
- On a socket `notification.new`, the badge refreshes, and so does every
  Notifications list already built.
- A gift row reads **"Ben sent a Fire to your post"**, or "sent a gift to
  your post" if the row carried no gift. **No emoji.**
- **The badge on the avatar** is the gift's own PNG at 16 px on a
  `surface`-coloured circle with 2 px padding, in place of the coloured
  circle with an icon that other kinds use.
- **The gift again at the end of the row,** 32 px, before the timestamp.
- Tapping opens the post.

## 8. No emoji, anywhere

| Place | Before v0.1.23 | Now |
|:--|:--|:--|
| Send button | "Send 🔥 Fire · 5 coins" | "Send Fire" · coin · 5 |
| Unknown gift type | its emoji | `Iconsax.gift` in `#F2B233` |
| Notification text | "sent a 🔥 Fire to your post" | "sent a Fire to your post" |
| Notification badge | `Iconsax.gift_copy` on amber | the gift's PNG |
| `GiftArt` API | `GiftArt(type, emoji, size)` | `GiftArt(type, size)` |

The models still parse `emoji` (default 🎁) because the server sends it,
but no widget reads it. The test "names no gift by its emoji, before or
after sending" scans the sheet's text.

## 9. Haptics, all of them

| Moment | Haptic | Why this one |
|:--|:--|:--|
| Tap the heart (like) | `selectionClick` | the lightest the platform has: a picker's notch |
| Hold the heart (sheet opens) | `lightImpact` | one step above a tap, so holding feels different, and far short of Android's long-press buzz |
| Pick a different gift | `selectionClick` | a notch as the choice moves |
| Gift sent | `lightImpact` | the moment coins are spent |
| Gift lands on the heart | `selectionClick` | felt, not announced |

The full sequence (hold, pick, send, land) is `lightImpact, selectionClick,
lightImpact, selectionClick`. This is asserted exactly by a test. Nothing
heavier is used anywhere.

## 10. Strings

All go through `l10n.ui(key, fallback)`. The fallbacks below are the
English text.

| Key | English |
|:--|:--|
| `ui_gift_title` | Send a gift (the sheet title, and the hold hint on the heart) |
| `ui_gift_balance` | You have $coins |
| `ui_gift_one_coin` / `ui_gift_coins` | 1 coin / $count coins |
| `ui_gift_test_notice` | Test coins: they cannot be bought and have no cash value. |
| `ui_gift_refills_on` | Refills $date. |
| `ui_gift_pick` | Pick a gift |
| `ui_gift_send_named` | Send $gift |
| `ui_gift_not_enough` | Not enough coins |
| `ui_gift_sent_named` | $gift sent |
| `ui_notif_gift` | sent a gift to your post |
| `ui_notif_gift_named` | sent a $gift to your post |
| `ui_retry` | Try again (shared) |

Gift **names** ("Fire") come from the server's catalogue and are English
only for now.

## 11. Reduced motion, summarised

With the system's "remove animations" setting on:

- the tiles do not move;
- the celebration is shown still for 900 ms;
- there is no flight, and the heart changes at once;
- **the landing punch and gold burst on the heart still play.** They reuse
  the like's animation, which does not check reduced motion. This is a gap;
  see [risks](08-risks-and-roadmap.md).
