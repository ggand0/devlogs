# Devlog 008: macOS 26 objc2 Crash Fix

**Date:** 2026-05-04
**Status:** In progress
**Issue:** https://github.com/ggand0/viewskater/issues/91

## Problem

viewskater panics on launch on macOS 26.2 (Tahoe) with:

```
invalid message send to -[...NSScreen_$ countByEnumeratingWithState:objects:count:]:
expected return to have type code 'q', but found 'Q'
```

The crash occurs during window initialization when winit enumerates `NSScreen` via `objc2-foundation` v0.2.2. macOS 26 enforces stricter Objective-C runtime type checking — the signed (`q`) vs unsigned (`Q`) return type mismatch was silently tolerated on earlier macOS versions.

Upstream tracking: https://github.com/madsmtm/objc2/issues/566

## Root Cause

The custom winit fork (`exp/upstream-crashfix-0.30.1-partial`, rev `7dea6466`) depends on `objc2 = "0.5.2"` without the `relax-sign-encoding` feature flag. This flag was added to objc2 0.5.x specifically to handle the signed/unsigned encoding relaxation that macOS 26 requires.

## Dependency Chain

```
viewskater
  -> iced (custom-0.13 fork)
    -> winit (custom fork, pinned rev 7dea6466)
      -> objc2 v0.5.2          <-- missing relax-sign-encoding
      -> objc2-foundation v0.2.2
```

## Fix Options Considered

1. **Upgrade winit to 0.31+ (objc2 v0.6.1):** Not viable — 0.31 changes the event loop system and causes performance regressions for this app.
2. **Upgrade winit fork to 0.30.13 base:** WIP branch `custom-dnd-0.30.13` exists but not ready. Note: upstream 0.30.13 still uses objc2 0.5.2 but *does* have `relax-sign-encoding` enabled.
3. **Minimal hotfix on current fork:** Create `hotfix/relax-sign-encoding` branch from current rev, add only the feature flag. **This is the chosen approach.**

## Fix Applied

### winit fork (`hotfix/relax-sign-encoding` branch)

Single change in `Cargo.toml`:

```toml
# Before
objc2 = "0.5.2"

# After
objc2 = { version = "0.5.2", features = ["relax-sign-encoding"] }
```

### iced fork (`custom-0.13` branch)

Added missing `reset_image_fps()` function (pre-existing build issue when using local path deps):
- `wgpu/src/image/mod.rs`: Added `reset_image_display_fps()` to clear all `ImageDisplayTracker` state
- `wgpu/src/lib.rs`: Added public `reset_image_fps()` wrapper

### viewskater

For local testing, switched `Cargo.toml` from git deps to local path deps (`../iced`, `../winit`). For release, the winit hotfix branch needs to be pushed and the iced fork's winit rev updated to point to it.

## Release Plan

1. Push `hotfix/relax-sign-encoding` branch to `ggand0/winit`
2. Update iced fork (`custom-0.13`) to reference the new winit rev (git, not path)
3. Push iced fork
4. viewskater Cargo.toml stays pointing at iced `custom-0.13` (no change needed for release)
5. Ask issue reporter to test

## Evidence That `relax-sign-encoding` Fixes the Error

The fix is verified directly in objc2's source (`crates/objc2/src/verify.rs`):

**Relaxation logic:** When `relax-sign-encoding` is enabled, unsigned integer type codes are normalized to their signed equivalents before comparison. `ULongLong` (type code `Q`) is treated as equivalent to `LongLong` (type code `q`):

```rust
fn relaxed_equivalent_to_box(encoding: &Encoding, expected: &EncodingBox) -> bool {
    if cfg!(feature = "relax-sign-encoding") {
        let actual_signed = match encoding {
            Encoding::UChar => &Encoding::Char,
            Encoding::UShort => &Encoding::Short,
            Encoding::UInt => &Encoding::Int,
            Encoding::ULong => &Encoding::Long,
            Encoding::ULongLong => &Encoding::LongLong,
            enc => enc,
        };
        // ... same for expected_signed
        if actual_signed == expected_signed {
            return true;
        }
    }
    encoding.equivalent_to_box(expected)
}
```

**Test case** asserting the exact error string from the issue:

```rust
let res = cls.verify_sel::<(), ffi::NSUInteger>(sel!(getNSInteger));
let expected = if cfg!(feature = "relax-sign-encoding") {
    Ok(())  // passes with the flag
} else if cfg!(target_pointer_width = "64") {
    Err("expected return to have type code 'q', but found 'Q'".to_string())
    // ^^^ exact error from issue #91
};
```

**Why macOS 26 triggers it:** Swift-backed ObjC classes (like the `NSScreen` array storage `_TtGCs23_ContiguousArrayStorageCSo8NSScreen_`) use `Int` (signed) for both `NSInteger` and `NSUInteger`. macOS 26 has more Swift-backed system classes, exposing the mismatch that older versions tolerated.

**ABI safety:** Confirmed by bjorn3 (Cranelift developer) in objc2#566 — on both x86_64 and arm64 Apple platforms, signed and unsigned integers of the same width are ABI-compatible.

## Cross-Ecosystem Confirmation

This is not unique to viewskater. The same crash affects the broader Rust GUI ecosystem:
- **Bevy:** https://github.com/bevyengine/bevy/issues/19567 — identical panic, same `objc2-foundation-0.2.2`, same `NSEnumerator.rs:7`, same macOS 26 trigger. Bevy also depends on winit with the same objc2 version.

## Notes

- The full 0.30.13 upgrade (`custom-dnd-0.30.13`) remains WIP and is a separate effort
- This hotfix is the minimum change to unblock macOS 26 users
- Confirmed compiles and runs on macOS (M1 MBA, opt-dev profile)
- Cannot verify the actual crash fix locally — need macOS 26 tester (the issue reporter)
