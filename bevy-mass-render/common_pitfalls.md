# Common Pitfalls

This document tracks recurring bugs and patterns that have caused issues multiple times in this codebase.

---

## 1. TurretRotatingAssembly Query Inclusion

**Frequency:** Very common (3+ occurrences)

**Problem:** Turret assemblies (the rotating gun part) have `BattleDroid` and `CombatUnit` components to reuse infantry targeting/firing logic. Any query for these components will unintentionally include turret assemblies alongside infantry units.

**Entity hierarchy:**
```
TurretBase (parent)
  ├─ TurretBase component
  ├─ Health component
  └─ TurretRotatingAssembly (child)
       ├─ BattleDroid component     <-- Causes query inclusion!
       ├─ CombatUnit component
       ├─ TurretRotatingAssembly component
       └─ MgTurret (optional)
```

**Symptoms:**
- Turrets appear in unit counts/lists
- Enemy units target assembly (child) instead of base (parent), then can't resolve position
- Features silently fail for turrets (queries expect MovementTracker which turrets lack)
- Unexpected entities in iteration loops

**Solution:** Always add `Without<TurretRotatingAssembly>` to infantry-only queries:

```rust
// WRONG - includes turret assemblies
mut combat_query: Query<(Entity, &GlobalTransform, &BattleDroid, &mut CombatUnit)>

// CORRECT - infantry only
mut combat_query: Query<
    (Entity, &GlobalTransform, &BattleDroid, &mut CombatUnit),
    Without<TurretRotatingAssembly>
>
```

**When writing turret-specific queries:**
```rust
// Query turret assemblies specifically
turret_query: Query<(&CombatUnit, Option<&MgTurret>), With<TurretRotatingAssembly>>

// Query turret bases (for position/health)
turret_base_query: Query<(&Transform, &Health), With<TurretBase>>
```

**Checklist for new systems:**
- [ ] Does this query `BattleDroid`? Add `Without<TurretRotatingAssembly>`
- [ ] Does this query `CombatUnit`? Consider if turrets should be included
- [ ] Does this expect `MovementTracker`? Turrets don't have it - use `Option<&MovementTracker>`

**Historical occurrences:**
1. `target_acquisition_system` - Enemies targeted assembly instead of base
2. `update_turret_details_ui` - Query required MovementTracker which turrets lack
3. Spatial grid updates - Turret assemblies were being tracked as units

---

## 2. Parent-Child Entity Resolution

**Problem:** When targeting a child entity (like TurretRotatingAssembly), damage/effects need to go to the parent (TurretBase).

**Solution pattern:**
```rust
// Get parent from child using ChildOf
if let Ok(child_of) = child_of_query.get(target_entity) {
    let parent = child_of.parent();
    // Apply effect to parent
}
```

---

## 3. Query Filter Order Matters for Performance

**Problem:** Bevy evaluates filters left-to-right. Put the most restrictive filter first.

```rust
// Less efficient - checks Without first, then With
Query<&Transform, (Without<Dead>, With<BattleDroid>)>

// More efficient - BattleDroid is rarer, checked first
Query<&Transform, (With<BattleDroid>, Without<Dead>)>
```

---

## Template for Adding New Pitfalls

```markdown
## N. [Pitfall Name]

**Frequency:** [Rare/Occasional/Common/Very common]

**Problem:** [What happens]

**Symptoms:**
- [Observable behavior 1]
- [Observable behavior 2]

**Solution:**
[Code example or explanation]

**Historical occurrences:**
1. [Where it happened]
```
