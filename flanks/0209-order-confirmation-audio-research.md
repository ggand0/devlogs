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

Revised 2026-10-06 against `refs/audio/elevenlabs-battle-sfx-notes.md`; the first version had the group shout generated at every size, which rule 4 (vocal counts under about 100 come out as a pub) rules out for the small band, and the order lines by TTS first, where rule 12 puts SFX with stress marks first.

- 16 solo barks from SFX ("Knight barking a one-word reply "HEY!" ...") and 12 group shouts stacked from them per size band with the notes' tested pitch, reverb and lowpass values, onsets within 80 ms to keep one hit; 8 generated medium and large takes ("Mass of knights shouting back a single "HEY!" in unison ...") to try against the stacks.
- 23 heavy and 26 light order lines from SFX in the notes' command-shout pattern (quoted, hyphenated, stressed syllable in capitals, "the last word loud", "one man, close"), TTS v3 through the class voices as the fallback. The archers never say "loose" for a formation, since "loose" means shoot.
- 6 optional single-man replies.
- Gota's earlier takes in `assets_dev/sfx_dev/voice/` mapped to the rows they could fill.
- Wiring notes: player orders only, one order line per click however many regiments, group shouts from the few nearest regiments.
