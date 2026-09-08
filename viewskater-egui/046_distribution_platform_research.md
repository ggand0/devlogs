# Distribution platform research

Date: 2026-09-08 and 09. Working notes, not a decision record. Every fee and
list below was read from the vendor's own page on the date given; re-check
before signing anything. Background: `docs/plans/2026-09-personal-project-
direction-analysis_claude-fable-5.1.md` (why sell at all) and
`~/ggando/plans/2026-09-viewskater-monetization-research_claude-fable-5.1.md`
(Steam, Microsoft Store, Flathub, Mac App Store, comparables). This entry
covers what those did not: direct-sales processors, the smaller storefronts,
and what an image viewer looks like on Steam.

## Outcome

- Sell the current build first, before any Pro feature. It tests the channel
  and the release pipeline, not demand. Expect single-digit sales from a
  build that is also free on GitHub.
- Direct sales from viewskater.com through a merchant of record is the
  primary channel for photographers. They buy from the developer's site
  (FastRawViewer, Photo Mechanic, XnView, Capture One all do this), want a
  key and an installer, and do not run a store client for a work tool.
- Merchant of record: try Polar first, Lemon Squeezy as fallback, Paddle
  out. Reasons in the sections below.
- Microsoft Store second (Windows is the largest platform, 15% cut, signed
  and auto-updating). Steam third (plumbing, wishlists, and a fit only if
  the app grows a VR or 3D modality).
- itch.io, Gumroad and Patreon add nothing over the above for a one-time
  desktop product.

## Channels

Fees read 2026-09-08 unless noted.

| Channel | Cut | Merchant of record | Auto-update | Audience |
|---|---|---|---|---|
| Steam | 30%, 100 USD per app refunded after 1,000 USD | yes | yes | gamers, developers, VR |
| Microsoft Store | 15% under 1M USD/yr, free to register | yes | yes | Windows users |
| itch.io | you choose, default 10%, plus ~3% processing | optional (collected-by-itch payout mode) | via itch app only | indie games, gamedev tools |
| Gumroad | 10% + 0.50 USD + processing; 30% on Discover | yes | no | creators |
| Patreon | 10% + processing, ~15-20% effective at small pledges | yes | no | recurring support, not sales |
| Paddle | 5% + 0.50 USD | yes | no | indie desktop software |
| Lemon Squeezy | 5% + 0.50 USD + 1.5% non-US card + 1.5% PayPal | yes | no | indie SaaS and creators |
| Polar | 5% + 0.50 USD + 1.5% international card | yes | no | indie developers |

Refunds and chargebacks: with a merchant of record the processor is the
legal seller, so refund requests and disputes go to them. You set a policy
and click approve. Stores handle it fully. Gumroad and bare Stripe leave it
to you, which is where the "handling refunds is annoying" reputation comes
from. Product support email is the same work on every channel.

## Image viewers on Steam (2026-09-08)

| App | Price | Released | Reviews |
|---|---|---|---|
| RGByte Image Viewer | ¥800 | 2025-02 | 0 |
| SuperIMGViewr | ¥2,500 | 2026-04 | 3 |
| Image Compressor | ¥235 | 2019-11 | 24, 70% positive |
| PHOSIMP (filters toy) | ¥120 | 2025-06 | 142, 97% positive |
| VR Photo Viewer, Witoo, MovingPictures, VR MEDIA VIEWER, AutoDepth | various | | the VR niche, real buyers |

Big non-game sellers for scale: DisplayFusion 1,077 reviews at 85%,
ShareX free with 2,778 reviews, Wallpaper Engine, Aseprite, Pixelorama,
Krita. Plain image viewers on Steam are a graveyard. The one image-viewing
job with a Steam audience is VR.

## Merchant of record: Paddle vs Lemon Squeezy vs Polar

A Grok prompt and response on Paddle vs Lemon Squeezy are in
`tmp/grok/2026-09-08_paddle_vs_lemonsqueezy.md` (not committed). Its
decisive claims were checked against vendor pages on 2026-09-09.

### Paddle: out

- Payout currencies (help center, 13 listed): AUD, GBP, CAD, CNY, CZK, DKK,
  EUR, HUF, PLN, ZAR, SEK, CHF, USD. No JPY. A payout in a currency other
  than the bank's country costs a 15 USD/EUR/GBP wire fee per payout, plus
  the receiving bank's fees. At tens of sales a month that exceeds the
  platform fee.
- Pricing page: "If you're selling products under $10 or require invoicing
  contact us for custom pricing." The launch price is in that range.
- Tax table (last updated 2025-08-01) lists Japan at 10% consumption tax,
  B2B and B2C. Qualified invoices under the invoice system: not verified.
- Onboarding reviews the domain and product; rejections for unfinished
  sites are common in seller reports.

### Lemon Squeezy: fallback

- Japan is on the bank payout list. Payouts twice monthly (created 1st and
  15th, paid 14th and 28th) after a 13-day hold, converted to JPY at
  mid-market. 1% fee for non-US banks. Minimum payout 50 USD, rolls over
  until met. This is on the official Getting Paid page; Grok had marked it
  unverified.
- Platform fee is charged on the tax-inclusive total: 0.50 USD + 5% + 1.5%
  outside the US + 1.5% PayPal. Their own example: 20 USD product, 20% VAT,
  fee 2.06 USD, net 17.94 USD. Dispute 15 USD plus the refund. Platform fee
  kept on refunds.
- Built-in file hosting with signed, versioned download URLs, customer
  portal, license keys with activate/validate/deactivate API. File size
  limit not found in docs.
- Risk: Stripe bought it in 2024. CEO post 2026-01-28 admits slower support
  and fewer updates while the team built Stripe Managed Payments, and says
  the goal is to migrate Lemon Squeezy users over. No shutdown date. Stripe
  Managed Payments lists Japan as a supported business location (with AU,
  HK, SG), so the migration is open to us, but Managed Payments has no
  file hosting or license keys of its own.
- Seller reports (r/SaaS post ~2026-02, Trustpilot reviews through
  2026-06): identity verification stuck for weeks or months, rejections
  with no reason, a Stripe account left in restricted status for months
  with payouts silently failing. Consistent with the CEO's own admission.
  For a first product, being stuck in verification with no support is the
  worst outcome, so this is not the first choice.

### Polar: first choice

- Fees (docs, 2026-09-09): Starter 5% + 0.50 USD, no monthly fee. +1.5%
  international cards. Dispute 15 USD. Payouts via Stripe Connect: 2 USD
  per active payout month plus 0.25% + 0.25 USD per payout. Organizations
  created before 2026-05-27 keep the old 4% + 0.40 USD; the Reddit posts
  quoting that rate predate the change. Pro plan 3.8% + 0.40 at 20 USD/mo
  if volume ever justifies it.
- Japan is on the supported payout countries list. Identity verification
  is Stripe's own Japanese KYC through Connect, not a Polar queue.
- File downloads up to 10 GB per file, license keys, customer portal. The
  license validate endpoint needs no API key, so an optional cosmetic
  unlock later is a plain HTTPS call from the app.
- Organization profile has its own "Public support email" separate from
  the login email, editable after signup (confirmed in
  `server/polar/organization/schemas.py` of polarsource/polar; the docs
  pages were 404 that day). Login email: `polar@<domain>`. Buyer-facing
  email: a stable `support@` or `hello@` that survives a processor switch.
- Risk: founded 2023, merchant of record since 2024, open source, already
  raised its free-tier price once. Younger than the others.

### Consumption tax, unresolved

Both Lemon Squeezy and Polar are foreign merchants of record. Best reading
from the Grok response, not verified: the buyer's contract is with the
processor, the payout to us is a supply to a non-resident, so likely
輸出免税 for 消費税 and ordinary income for 所得税. Neither processor is
confirmed to issue 適格請求書. Questions for a 税理士 are listed at the end
of the Grok file: treatment of the payout, whether Steam and Microsoft
Store proceeds get the same treatment, whether to register for qualified
invoices below the ¥10M threshold, FX booking, 開業届 and 青色 timing.

## Still to verify before launch

- Polar: confirm Stripe Japan verification completes for an individual
  and a test purchase pays out.
- Polar or Lemon Squeezy in writing: JCT on Japanese buyers, qualified
  invoice issuance, installer file size limit.
- Windows code signing before the direct page goes live. An unsigned exe
  from a store is tolerable; from a direct page it looks like malware.
- Update check in the app pointing at the download page, since direct
  sales have no auto-update.

## Sources

Paddle: /help/manage/get-paid/can-i-be-paid-in-my-local-currency,
/help/manage/get-paid/is-there-a-fee-taken-for-payouts, /pricing,
/help/sell/tax/which-countries-does-paddle-charge-sales-tax-or-vat-for.
Lemon Squeezy: docs /help/getting-started/{supported-countries,getting-paid,
fees}, blog /2026-update, trustpilot.com/review/lemonsqueezy.com.
Polar: polar.sh/docs/merchant-of-record/{fees,supported-countries},
polar.sh/blog/introducing-polar-plans, github.com/polarsource/polar.
Stripe: docs.stripe.com/payments/managed-payments/eligibility.
Steam store pages for the apps listed. itch.io creator FAQ, Gumroad and
Patreon fee pages via third-party summaries dated 2026.
