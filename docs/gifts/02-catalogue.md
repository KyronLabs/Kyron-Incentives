# Catalogue and artwork

## The eight gifts

Source of truth: `api/src/modules/gifts/gift-catalogue.ts`, `GIFT_CATALOGUE`.

| # | `type` (key) | Name | Price (coins) | Picture | Motion in the sheet | Shipped |
|--:|:--|:--|--:|:--|:--|:--|
| 1 | `spark` | Spark | 1 | four-point sparkle with two small sparkles | twinkles | v0.1.20 |
| 2 | `rose` | Rose | 3 | one rose in bloom on a short stem with a leaf | sways | v0.1.22 |
| 3 | `fire` | Fire | 5 | stylised bubbly flame | flickers | v0.1.20 |
| 4 | `star` | Star | 10 | puffy five-point star with an orbiting ring | turns (3D) | v0.1.22 |
| 5 | `gem` | Gem | 20 | brilliant-cut gemstone | catches the light | v0.1.20 |
| 6 | `cake` | Cake | 30 | two-tier cake, cherry, one candle | bobs | v0.1.22 |
| 7 | `crown` | Crown | 50 | crown with pearl points and band gems | tilts | v0.1.20 |
| 8 | `rocket` | Rocket | 100 | toy rocket, porthole, puffy flame, coloured body | hovers | v0.1.22 |

Rules the catalogue must keep. Each is enforced by a test in
`gifts.service.spec.ts`:

- **Sorted by price.** The array order is the order the sheet shows them:
  two rows of four.
- **Gifts already sent keep their key and price.** Spark 1, Fire 5, Gem 20
  and Crown 50 shipped first and must stay as they are, so every stored
  `Gift.type` still names a catalogue entry.
- **Nothing costs more than a week's coins.** The dearest gift (Rocket, 100)
  equals `WEEKLY_COINS`.
- **Every key is unique.**

The server also carries an `emoji` per gift (✨ 🌹 🔥 ⭐ 💎 🎂 👑 🚀). Since
v0.1.23 **the app never shows it**. It stays in API responses only for app
builds older than v0.1.22, which used it as their fallback.

### Adding or changing a gift

1. Add an entry to `GIFT_CATALOGUE` at the right place in price order. A new
   key needs **no migration**: `Gift.type` is a plain string.
2. Never change the key or the price of a gift that has been sent. Stored
   gifts keep the price they were sent at (`Gift.coins`). Changing a price
   changes only future gifts, but the "keeps the gifts that have already
   been sent" test pins the original four.
3. Add `app/assets/gifts/<type>.png` (see below), list it in
   `app/pubspec.yaml`, and add the key to `GiftArt.drawn` in
   `app/lib/widgets/gift_sheet.dart`.
4. Optionally add an idle motion case in `giftPose()`
   (`app/lib/widgets/gift_motion.dart`). Unknown types get a gentle bob.
5. An app released before the picture existed draws a gold present icon for
   the new gift. It never draws a blank or an emoji.
6. Removing a gift from the catalogue: stored gifts and notifications keep
   working. A notification row for an unknown type is named "Gift".

## The coin

`app/assets/gifts/coin.png`: a round gold coin with an embossed four-point
sparkle. It appears beside the balance (22 px), under every tile's price
(14 px), and in the Send button (16 px). If the file fails to load, the
fallback is `Iconsax.coin_1` in `#F2B233`.

## The artwork

Files: `app/assets/gifts/{coin,spark,rose,fire,star,gem,cake,crown,rocket}.png`.
Each is listed individually in `app/pubspec.yaml`, not as the folder, so the
folder's `README.md` does not ship inside the app.

### Origin and rights

- **Generated for Kyron** on 30 October 2026 with Figma's image generation
  (`generate_image`, model `gpt-image-2.5-flare`), in the Figma file "Kyron
  gifts — artwork" on Kyron's own team. They are used under Figma's terms
  for AI output.
- **Not drawn from anyone's work.** Screenshots of other apps' gift
  catalogues were used only as a mood board (glossy, soft 3D, iridescent),
  never as a source.
- **Rights caveat.** AI-generated images without enough human authorship
  may not be protected by copyright in the US and elsewhere. Kyron may use
  them freely but may not be able to stop others using them. If a gift
  becomes part of the brand, have an illustrator redraw it and record that
  in `app/assets/gifts/README.md`.

### The shared prompt

Every image used one prompt so that the set reads as one family:

> A single cute glossy 3D app icon of **[subject]**, soft clay-like 3D
> rendering, **[colours]**, glossy highlights, soft studio lighting,
> **[view]**, centred, isolated on a plain flat pure white background, no
> shadow, no text, no border.

The rocket was generated twice. The first had a white body, and cutting
away the white plate removed part of the rocket. The second was generated
with a coloured body.

### How they were cut: `app/tools/cut_gift_art.py`

```
python3 tools/cut_gift_art.py SOURCE.png assets/gifts/NAME.png [--size 256]
```

1. **Remove only the plate.** It flood-fills from the image border through
   near-white pixels: the worst channel is within `PLATE = 28` of 255. A
   white highlight inside the object is not connected to the border, so it
   survives. A colour key would have punched those highlights out.
2. **Soft edge.** Alpha ramps by distance from white between `EDGE_FROM = 6`
   and `EDGE_TO = 48`, so the outline is antialiased.
3. **No white halo.** Each edge pixel's colour is solved for what it would
   be without white behind it (decontamination) before its alpha is
   applied. Without this, a pale rim shows on the dark sheet.
4. **Crop, centre, and write at 256 × 256 px.** The sheet draws a tile at
   48 logical px (112 px when sent), so 256 px covers a 4× screen at tile
   size and stays small enough to ship nine files.

The 1024 px sources are in the Figma file.

### Checking them visually

`app/test/render_gift_sheet.dart` is a rendering tool, not a test. It draws
the sheet to PNGs, light and dark, before and after a pick, plus three
frames of the send burst:

```
flutter test test/render_gift_sheet.dart --dart-define=shots=/some/dir
```

Text in the Send button draws as grey boxes in those images. That is a
font-loading quirk of the tool, not of the app.
