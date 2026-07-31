# 0067: selectable enemy army styles

Branch: feat/army-picker, follow-up to 0066 after the owner's first
play-test ("looks good and works").

## The ask

The owner wanted the five random archetypes to be directly
selectable for the enemy, with Random staying as it was. The enemy
page's Random/Manual toggle became a chip row: Random, Balanced
Host, Iron Wall, Spear Hedge, Arrow Storm, Skirmish Horde, Manual.

## What changed

- BattleConfig.enemy_regs (Option) became the EnemyComp enum:
  Random, Style(idx), or Manual(counts). Army-size changes still
  reset it to Random.
- regiments.rs: the archetype math split into comp_from_fracs (the
  shared clamp ladder, also used by frac_comp), archetype_preview
  (un-jittered counts for the UI), and archetype_jittered (the
  battle-start draw). A chosen style is still jittered up to a fifth
  per kind each battle; Random picks the style uniformly first. The
  spawn logs the style either way.
- Picker: chips highlight the active choice (CustomStyled so the
  hover styler leaves them alone). Selecting a style hides the grid
  and shows the note pane with the style's representative counts at
  the current budget plus the jitter caveat; roster card counts and
  the info column show the same representative numbers. Manual and
  the player page behave exactly as before.

## Follow-up in the same session

The owner wanted the style preview as unit cards, not text, plus the
two small items from the feature-complete list:

- Selecting a style now fills the army list with its representative
  cards (read-only: the removal guard already no-ops outside the
  player page and Manual). The note pane shrinks to the one-line
  jitter caption under the grid; Random keeps the full note and
  hides the grid as before.
- A Default button sits after the team arrows and restores the
  classic split for whichever editable composition is on screen
  (player page, or enemy Manual). Hidden on Random/style pages.
- Right-clicking a roster card removes one regiment of that kind
  (Shift for 10), matching M2TW. Grid cells accept right-click for
  removal too. The hint strip mentions it.

## Cleanup pass

A pedantic clippy sweep over the branch surfaced nothing real
outside pre-existing codebase idiom; the applied cleanups were
deduplicating the three small-button spawn blocks (arrows, chips,
Default) into one helper, collapsing the chip-highlight check into a
single match, and deriving Eq on EnemyComp. PR draft written to
tmp/PR-army-picker.md.

## Checks

Build and clippy clean. FL_TEST_CHARGE and FL_TEST_FRONT runs spawn
with the classic split and no style roll (the scripted pinning from
0066 holds through the refactor). The chip row itself needs the
owner's live click-through: synthetic keypresses stopped landing in
this session (the window opens unfocused while the owner works, and
focus-stealing is off limits), so no screenshot of the enemy page.
