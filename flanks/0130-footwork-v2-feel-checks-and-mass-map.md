Written by Claude Fable 5.1

# 0130: Footwork v2 under Gota's feel checks: facing, the last man, the enemy crowd

Branch `feat/melee-footwork`. Follows 0129 (125f66c). Gota's rule for this phase: once a fix is built he plays it first, no scenario batches or rebuilds of his binary until he says so (memory feel-check-before-batches). Test builds now go to a separate target directory (`CARGO_TARGET_DIR=target-agent`, excluded locally via .git/info/exclude) and are run with `FL_BIN=target-agent/opt-dev/flanks work/scripts/pile-wide.sh`; his `target/opt-dev/flanks` changes only when he says so. Launchers: work/scripts/pile-wide.sh (the 100-file line vs the 12-file block; `-c` locks the camera, `-t 60` runs muted and prints the roll-up table) and work/scripts/pile-two.sh (two on one).

## Feel check 1 on 125f66c (wide line): three observations

Screenshots resources/footwork/debug_092526/debug14 to 17. Gota: overall still better than main.

1. Men who followed comrades toward the centre stood in loose ranks facing forward, not the enemy mass. Cause: a man out of formation who stands falls through to the formed man's rule (keep the line facing), which only yields to a target within 6 m, and v2's targets come from the 2 m box. Fix: a committed man with no enemy in reach faces where he is going (memo_dir: the man he sees, else his goal), the formed man's rule applies to men in formation only.
2. Some men went back and forth between their slot and the approach. Not pinned; the pile log's "running back" count was 1 to 3 per five seconds. Likely a man in formation with an enemy in reach behind him (no swing without the forward half-plane) lunging and being pulled back until his reaction roll commits him.
3. One man on the far flank always left behind, joining by patience. Cause: the start event fired on a man's speed crossing 1.0 m/s, and a man setting off behind a comrade walks at the advance pace, 1.06 m/s, which with any slope or crowd factor he never crosses; the tail of a line got no cue. Fix: the event fires at 0.5 m/s (SET_OFF_SPEED) and repeats every half second while he keeps going, so the men he leaves behind have him in view until he is 6 m away, as the old runner look did.

## The face rectangle, built and replaced the same night

The fight face of 125f66c (the nearest point of the enemy block's rectangle) was first built from the men's extents; one man chasing out onto a wing stretched it over the field and men heading for its "face" stopped on empty ground (the wide line's "away from the fight" count went 314/269/241/228 at 20/30/40/50 s against bee555e's 259/196/152/122). Rebuilt from the slot extents with a one-rank inset: 295/234/195/151. Then Gota's feel check 2 (debug18, 19): wing men facing completely sideways, away from the mass. Cause: the slot rectangle centred on the men's centroid; a 42-rank block pressed forward fills half its slots, so the rectangle stuck 20 to 30 m past the men onto the orange side. Gota: "a bunch of blind men; is the radius only 2 m?" Yes: the enemy sight was the touch box, and any rectangle is a stand-in for where the crowd is.

## The mass map (committed after this entry's first half)

Picture work/notes/vis/013-mass-map.png. Each regiment engaged or in melee bins its living men 3 x 3 over its footprint every tick in update_groups' existing per-unit loop (bins of a third of last tick's slot extents around the centroid, men beyond them in the edge bins; GroupData::mass, nine centroids and counts). A man of a fighting regiment with nobody in his box reads the map of the regiment his own fights (fight_target): out of formation he faces and heads for the nearest non-empty bin, every tick; in formation, on his look tick, a bin within FL_SEEK_R (15 m) is an enemy in sight (SIGHT_MASS, sharing bit 6 with the polled perception's far-runner bit, which v2 never sets) and he reacts in his own time, which restores the case v2 had given up. Cost: nine distance checks, no scanning; the binning adds about a third to a loop that already runs, for engaged regiments only.

Gota's feel check 3 on the mass map (wide line): "much better, engagement is more interesting, good and satisfactory for this scenario." Committed as the second commit of the branch's v2 series (see git log after 125f66c).

## Feel check 4: a 5 v 5 line battle (debug20)

Two problems, both understood, neither built yet:

1. **Three-part blobs.** Every engaged pair curled into a round blob and the front became three lumps. First explanation (wing men aiming at their target's side bins, ignoring the enemy regiment ahead; fix by reading the nearby enemy regiments' maps) was wrong: the picture work/notes/vis/014-blobs.png showed the arrows unchanged, since in an equal-width line a wing man's nearest bin is still his own target's outer bin. The real cause: men head for bin *centroids*, three points across a regiment's front, so the ranks behind converge on three spots and the wings pull inward toward the outer centroids. Proposed fix: each bin also keeps the box of its men, so the map is the crowd's outline; a man heads for the nearest point of the boxes (straight at the crowd in front, spread evenly along its edge; the wrap only where nothing stands ahead), reads the two nearest other enemy regiments' maps too, refreshes his goal on his look tick and keeps it in a per-man column between looks. About 300 cycles per look, under ~40 cycles per soldier on average.
2. **No charge impact; a unit stops the moment it touches the enemy.** Three causes: the charge flag turns off at engagement (main's code), the contact frame freezes at the first wind-ups (bee555e) or at the first men in reach (this branch's clock change), and v2's wave commits the ranks behind within two seconds, which drops their slot pull. M2TW runs the charge task until most of the unit has charged (finish proportion 0.75, devlog 0120). Proposed fix, a crash phase: from the moment a charging regiment engages until its block stops advancing (centroid forward speed under 0.3 m/s smoothed, 8 s cap), the frame keeps moving, the charge boost stays, the slot pull keeps pushing the ranks in, fighters fight as on main, and the melee clock waits; then the frame freezes where the fight line is and the melee behavior takes over. Defenders' clocks start at contact as now; an attacker walking in stalls at once. Gota agreed to this one.

## Statistics gathered between feel checks (deterministic scenarios, bee555e baselines)

- Polled gates (FL_FAR_LOOK=1) bit-identical to bee555e after every change so far.
- Wide line, out of formation at 10/15/20/25/30/35 s: bee555e 74/270/357/390/401/394; v2 with resolve events 64/451/462/441/428/407 (memory 15, 30 or 60 ticks made no difference); v2 with swing/set-off events and the slot face 51/418/464/449/435/412. The wave is about twice bee555e's speed; not tuned yet, Gota has not complained about it.
- Two on one, victim alive at 15/25/35/45 s: bee555e 459/376/304/244; v2 with the slot face 467/405/354/251.
- DIR: damage per hit by sector unchanged; the lone victim dies faster (35 vs 92 alive at 38 s).
- Cost ABAB at 200k: not yet run (feel checks first; the cycle-counted binaries need rebuilding from the current source).

## Open

- Build the two fixes above (boxes + neighbor maps + stored goal; the crash phase), Gota's feel check on the 5 v 5 and the wide line, then the roll-up tuning, then the cost ABAB against bee555e and main, then new baselines and the movement.rs refactor.
- The back-and-forth of item 2 above, once the rest settles.
