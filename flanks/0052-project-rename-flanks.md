# 0052 — Project rename brainstorm: cascade -> flanks

**Date:** 2026-07-24 brainstorm, 2026-07-26 final decision
**Branch:** `main`

## Context

The project started as "cascade" when the concept was a swarm-vs-swarm war
of dots / M2TW hybrid. As it evolved into a proper Total War / UEBS-style
200k medieval battle sim, the name stopped fitting -- "cascade" sounds
artificial/synthetic, more suited to a particle sim than a battle game.

Goal: a name good enough for a public open-source GitHub repo. Not
necessarily final, but something that fits the game's identity.

## Requirements

- Evoke the scale (200k soldiers)
- Evoke "the line" -- the endless grinding frontline visible in large scenarios
- Era-agnostic (medieval now, but could expand to ancient, early modern, etc.)
- Memorable, not boring or overly literal
- Clean on GitHub (no major name collisions)
- Hard consonants (t, k, d, g, p) -- soft names like "amassed" or
  "cascade" don't match the weight of what's on screen

## Rejected directions

- **Military formation words**: battleline (too boring), frontline (sounds
  WW1), shieldwall (existing commercial game + 264-star firewall project)
- **Tide/flow metaphors**: wartide, irontide -- nothing felt right
- **Mass- compounds**: massfront, massfield ("alright but not memorable")
- **Made-up compounds**: fracline (forced), warline, ironline (all felt
  flat)
- **Tool/forge metaphors**: anvil ("not minecraft"), crucible, grindstone
  (too crafty)
- **Latin/Norse/archaic terms**: acies, othismos, fyrd, schiltron --
  interesting but too obscure or didn't click
- **Chaos/scale words**: tumult, havoc, throng, onslaught -- capture the
  wrong aspect (chaos, not the line)
- **"amass" / "amassed"**: right meaning (scale, gathered force) but
  "amass" killed by 15k-star OWASP tool; "amassed" sounds too soft --
  starts with open vowel, sibilant double-S
- **Formation-specific terms**: echelon (sounds good but names one
  specific formation, not the game), tercio, testudo, phalanx -- too
  era-locked or too specific
- **Unit/rank words**: cohort (good hard consonants but sounds like a
  Roman unit, not gameplay), regiment (organizational, not descriptive
  of gameplay), garrison (means stationed defense force)

## Winner: flanks

"Flanks" -- the sides of a formation, what you protect and what you
attack.

Why it works:
- **Describes gameplay**: flanking is the core tactical decision --
  protect yours, break theirs. Morale collapses from being outflanked.
- **Hard consonants**: FL-N-K-S. Punchy, aggressive, one syllable.
- **Era-agnostic**: flanks exist in every era of warfare
- **GitHub-clean**: zero game repos named "flanks"
- **No competing meanings**: plural kills the "flank steak" reading;
  native English speakers hear "flanks" and think battlefield
- **Scale-implicit**: the plural suggests flanks everywhere across a
  massive line, not one isolated maneuver

## Rename scope

- Cargo.toml: package name `frontline` -> `flanks`
- Window title: `"frontline"` -> `"flanks"`
- Settings config dir: `~/.config/frontline/` -> `~/.config/flanks/`
- README title and description
- `src/frontline.rs` and `FrontlinePlugin` are KEPT -- they name the
  frontline visualization mechanic, not the project
