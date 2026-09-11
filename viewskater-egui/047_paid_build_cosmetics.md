# Paid build cosmetics

Update 2026-09-11: item 3 (accent presets) shipped as PR #35, with a
custom color picker on top. See devlog 048. The empty-state animation was
prototyped and parked on feat/official-empty-state.

Date: 2026-09-09. Ideas from a planning conversation, not decisions. The
constraint from the plan docs still applies: the store build is a thank-you
for buyers, not a reason to buy. Everything here must have a plain fallback
so a source build never looks broken, and nothing may depend on a server or
a license check in the app.

## Mechanism

- A Cargo feature `official` gates the cosmetics. `cargo build` without it
  is the free build and is what contributors run.
- The private part is an asset bundle (icon variants, About art), not code.
  The code that renders it is public. The release job for the store build
  fetches the bundle from a private location and the build embeds it with
  `include_bytes!` behind `cfg(feature = "official")`.
- Normal builds compile the same code paths with the asset absent and show
  today's UI.

## Ideas, in order of value per effort

1. App icon variant. The store build ships a different icon, for example the
   skater on a dark or gold badge. It sits in the taskbar, dock, and alt-tab
   all day, so it is the most visible signal for the least work. One image
   file per platform format.
2. About screen artwork. The About modal (`src/about.rs`) is text only
   today: name, version from build info, description, contributor links,
   learn-more link. The store build draws an image above the version line
   and a "Thank you for your purchase" line with the channel (Steam or
   Microsoft Store) and version. On Steam, optionally the buyer's name via
   the `steamworks` crate. Free builds show the current text-only modal.
3. Accent theme preset. (Correction 2026-09-11: the accent was never user
   editable; it was hardcoded in `UiTheme::teal_dark`, and `accent_slider`
   is the name of a slider widget.) The store build adds a setting with
   named presets that match the icon palette. Free builds stay teal.

Rejected:

- A full color scheme change. Screenshots, docs, and bug reports would then
  differ between free and store builds, and every "why does mine look
  different" email is a support cost. The accent preset gives the feeling
  without forking the look.
- A splash or launch animation. The whole pitch is that the app opens
  instantly.
- Anything that needs a server, such as a supporter wall with the buyer's
  name.

Maybe: a small skater easter egg on the empty-state screen or at the end of
a folder. Cosmetic, off the hot path, the kind of thing people screenshot.

## Collectible angle

Game cosmetics work through visibility and scarcity. Both are possible with
no server:

- Founder icon. Buyers of the first store release get an icon variant that
  later buyers never get. Each major version ships a new icon and About art,
  and the app keeps every past variant in a picker, so early buyers own the
  rarest one. Same trick as founder badges in early access games; costs one
  extra image per release.
- Store-specific variants. A Steam icon and a Microsoft Store icon that
  differ slightly.
- Steam Points Shop items: profile backgrounds, emoticons, badges tied to
  the app. That is the real collectible mechanism on Steam and costs only
  artwork. Trading cards depend on a Steam sales threshold, so do not count
  on them.

For a one-time-purchase utility the founder icon is the one most likely to
land.

## Artwork sourcing

First store release: self-made or generated art is fine. The asset is
outside the repo, so replacing it later is a file swap with no code change.
Steam's content survey asks whether the product or listing contains
AI-generated content and shows the answer on the store page. The Microsoft
Store has no equivalent disclosure as far as checked. Replace generated art
before the photographer launch.

Commissioning, from Japan:

- Skeb: artists list a price, one-shot request, no back and forth. Tags
  ロゴ, マスコット, ピクセルアート. A single mascot piece is a few thousand
  to a few tens of thousands of yen.
- pixiv Requests: same model, larger pool.
- X: search 依頼受付中 or "commissions open" plus the style. Indie Steam
  capsule artists post there, and capsule art is a needed format anyway.
- Credits: find an indie Steam app or GitHub project whose art fits, read
  the credits, message the artist.

One artist, one style, one brief: the skater mark as a character, a square
icon variant, a wide About header, and the Steam capsule sizes. Reused
across store pages, viewskater.com, and the app.

## Open

- Which store channels get which icon, and whether the free build ever gets
  a variant picker with only the default in it.
- Whether the Steam name line is worth the `steamworks` dependency for one
  label.
- Asset bundle location and how the release job authenticates to it.
