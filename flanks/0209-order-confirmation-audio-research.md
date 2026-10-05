# 0209: order confirmation audio, M2TW's three layers and the prompt sheet

Written by Claude Opus 5.5.

2026-10-05, main tree on `feat/soldier-scale` (no code touched). Plan 020, track G: "The order confirmation voice: a unit-wide shout when ordered." Done while the machine's CPU was busy with another project: no game, no build. The prompt sheet is `work/audio/order-confirm-prompts.md`.

## What M2TW does

From the vanilla text configs and the decoded events (devlog 0165). The samples were pulled from `Voice1.idx` and `Voice3.idx` with `work/research/m2tw-sound-loudness/sndpack.py` into `work/audio/listen/atmosphere/m2tw/18_order_confirm/` (12 group shouts, 213 English order lines) and 106 of them transcribed with ElevenLabs Scribe (`transcripts.md` there).

An order plays up to three layers:

1. The order line (`unit_voice`): one voice per unit by class (General, Heavy, Light) and accent, 0.25 s after the order, 0 dB, full level within 17 m, priority 200. Two to five lines per order: "Forward!", "Move out!", "March!", "Onward!", "Advance!" for a move; "Attack!", "Crush them!"; "Halt!", "Stop!"; "Form spearwall!"; "Fire at will!" / "Cease fire!"; "As you were!" for every state turned off. Heavy 1.0-2.2 s, Light shorter. A single unit's selection has no voice: the `Unit_Generic` files are named in the config and missing from the packs.
2. The group shout (`unit_confirm`): one syllable barked by the whole unit together, 0.53-0.61 s; Scribe hears "Hey!", "Oi!", "Hi!" or a grunt. 3 small, 3 medium, 6 large clips by the unit's size band, shared by every accent and class. -15 dB with up to 15 dB more cut at random, pitch 0.8-1.0, random delay 0-0.3 s, priority 170. It follows only orders whose vocal is marked `confirm` (move, attack, halt, formation and state orders).
3. One man's reply (`Individual_Confirm`): "Aye!", "Sire!", "Yes!" at a 10 % chance.

Not settled by the text: whether the shout waits for the order line to end. The bank comment "confirm sounds can't have stitching inside!" points that way. Check: record M2TW's output during a few orders and read the onsets.

## What flanks plays today

A UI click per action and a rate-limited war horn from the ordered regiments; halt and the formation keys play nothing; no voice anywhere. Regiments hold 200, 500 or 1000 men, so the size bands sit higher than M2TW's.

## The sheet

- 12 group shouts from the SFX model, unison asked for on purpose, with a fallback of stacked solo TTS barks (onsets within 60 ms) if the small takes come out as a pub.
- 23 heavy and 26 light order lines by TTS, one voice per class shared with the taunts, the `[shouting]` tag and the stressed word in capitals.
- 6 optional single-man replies.
- Gota's earlier takes in `assets_dev/sfx_dev/voice/` mapped to the rows they could fill.
- Wiring notes: player orders only, one order line per click however many regiments, group shouts from the few nearest regiments.
