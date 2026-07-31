# 0012 — Front line demoted to pure visualization (2026-07-09)

## Owner verdict

"The frontline is just a visualization / byproduct of colliding units. It
being an attractor is the strangest feature I have ever seen." Also:
probably pivoting to a Total War-style direction on the next branch —
keep the sim simple.

## The two symptoms and their shared root cause

1. **Streams of blobs on contact**: engaged steering sent every unit to its
   *nearest point on the contour* — a many-to-one map. Convex wiggles of
   the 8 m-grid contour captured fans of units (Voronoi collapse), the
   per-cell BFS lookup tore formations along cell boundaries, and bunching
   raised density which bulged the contour which attracted more units —
   self-reinforcing channels.
2. **Orders not reaching the clicked point**: within 60 m of any contour,
   line steering *replaced* the order (which degraded to a ≤90° direction
   hint) — so groups pressed wherever the line was, never at the ordered
   point, and passing groups got captured by flank battles.

Root cause of both (and of the earlier "units move without orders"): the
front was an **attractor with authority over movement**.

## The fix (net code deletion)

- **Movement = orders + collision. Nothing else.** Ordered units steer
  straight at the ordered point (arrive + slope); unordered units stand
  fast. Enemy contact is pure physics: cross-team separation blocks,
  crowd-density yield stops the shove, melee thins the block. The battle
  line is *where collisions happen*, not a thing units know about.
- frontline.rs is now a visualizer: density splat/blur + marching-squares
  contour (CONTACT_T back up to 0.5 so the drawn line only exists at real
  contact) + gizmos. The BFS nearest-front propagation, ENGAGE_DIST,
  per-cell normals, and group max_depth are deleted. Group `engaged` is
  now just "enemy blurred density at centroid > 0.8" (order-arrival
  bookkeeping + AI hooks).
- Stance (F) is inert for now; formation shapes return with the TW branch.
- Test script gained a drift watch: cut a group near the front with NO
  order → log its centroid drift over 20 s (should be ~0).

## Verified

Contact screenshot: both armies coherent (no streams), sharp interface,
single yellow line drawn exactly at the collision, 244 fps / 4.2 ms,
step 2.8 ms. Owner confirmed in play.

## Lesson (the big one)

Emergent-simulation features must not steal player authority. The moment
a field competes with an explicit order for control of units, the game
feels broken even if the simulation is "working". Orders are sacred;
fields may only *inform* (visuals, AI decisions) or *resist* (physics).
