# PathType vs Unified Architecture Analysis

**Date:** 2025-09-06
**Context:** Post-refactoring analysis of image loading architecture in ViewSkater

## Background

During the refactoring to simplify `PathType::FileByte` complexity, we replaced the original `PathType` enum with a unified `Vec<PathBuf>` approach and moved all loading logic to `file_io::read_image_bytes()`. However, this architectural change introduced some performance and design trade-offs that warrant analysis.

## Original PathType Architecture

### Design
```rust
pub enum PathType {
    PathBuf(PathBuf),                           // Regular filesystem files
    FileByte(String, std::sync::OnceLock<Vec<u8>>) // Preloaded archive content
}
```

### Loading Logic
```rust
match path_type {
    PathType::PathBuf(path) => /* Direct filesystem read */,
    PathType::FileByte(_, lock) => /* Direct memory access via OnceLock */
}
```

## New Unified Architecture

### Design
```rust
pub image_paths: Vec<PathBuf>  // All paths as PathBuf
```

### Loading Logic
```rust
// Always executes this sequence:
1. Check archive_cache.preloaded_data (HashMap lookup)
2. Check path.exists() (filesystem syscall)
3. Try archive.read_from_archive() (if needed)
```

## Performance Analysis

| **Operation Type** | **PathType (Original)** | **Unified PathBuf (Current)** |
|-------------------|-------------------------|-------------------------------|
| **Regular files** | 1 operation (direct filesystem) | 2 operations (HashMap miss + filesystem hit) |
| **Archive content** | 1 operation (direct memory/archive) | 1-3 operations (HashMap check + potential filesystem + archive) |
| **Efficiency** | ✅ Zero unnecessary syscalls | ❌ Wasteful `path.exists()` calls for archive content |

## Pros & Cons Comparison

### PathType Approach (Original)

**Pros:**
- ✅ **Direct dispatch** - no unnecessary checks
- ✅ **Compile-time distinction** between file/memory sources
- ✅ **Type safety** - clear intent at type level
- ✅ **Zero wasteful syscalls** - immediate routing to correct handler
- ✅ **Performance** - always exactly 1 operation

**Cons:**
- ❌ Complex enum with `OnceLock` complexity
- ❌ Mixed concerns (path handling + memory management)
- ❌ Less maintainable due to enum complexity
- ❌ `OnceLock` lazy initialization overhead
- ❌ Harder to reason about memory lifecycle

### Unified PathBuf Approach (Current)

**Pros:**
- ✅ **Simple, unified interface** - single entry point
- ✅ **Clean separation** - file_io handles I/O, ArchiveCache handles archives
- ✅ **Standard library types** - familiar PathBuf usage
- ✅ **HashMap-based preloading** - more transparent than OnceLock
- ✅ **Maintainable** - clearer code organization

**Cons:**
- ❌ **Performance overhead** - always 2-3 operations vs 1
- ❌ **Runtime guessing** - blind checks without type-level knowledge
- ❌ **Wasteful syscalls** - `path.exists()` for archive content
- ❌ **Lost type safety** - no compile-time source distinction
- ❌ **Implicit context** - relies on `pane.has_compressed_file` flags

## Better Architectural Options

### Option 1: Lightweight Tagged PathBuf ⭐ **(Recommended)**
```rust
enum PathSource {
    Filesystem(PathBuf),    // Regular directory files
    Archive(PathBuf),       // Internal archive paths  
    Preloaded(PathBuf),     // Preloaded archive content
}
```
**Benefits:** Type safety + performance, no OnceLock complexity, clear intent

### Option 2: Context-Aware Loading
```rust
enum LoadContext {
    RegularDirectory,
    ZipArchive, 
    RarArchive,
    SevenZArchive,
}

fn read_image_bytes(path: &PathBuf, context: LoadContext, cache: &mut ArchiveCache)
```
**Benefits:** Explicit context, efficient dispatch, minimal type changes

### Option 3: Strategy Pattern
```rust
trait ImageLoader {
    fn load_image(&mut self, path: &PathBuf) -> Result<Vec<u8>, Error>;
}
```
**Benefits:** Pluggable loading strategies, clean abstraction

### Option 4: Enhanced Current with Hints
```rust
fn read_image_bytes(
    path: &PathBuf,
    archive_cache: Option<&mut ArchiveCache>, 
    hint: LoadHint  // filesystem_only, archive_only, auto_detect
)
```
**Benefits:** Minimal change, performance hints, backward compatible

## Core Issue Identified

The unified approach **lost type-level information** about data sources, forcing a "guess-and-check" pattern that:

1. **Wastes syscalls** on `path.exists()` for archive content that will never exist on filesystem
2. **Loses compiler assistance** in routing to correct handlers
3. **Creates implicit dependencies** on runtime flags like `pane.has_compressed_file`
4. **Reduces performance** from guaranteed 1-operation to 2-3 operations per load

## Recommendation

**Implement Option 1 (PathSource enum)** as it provides:
- **Best of both worlds** - clean architecture + performance + type safety
- **Minimal complexity** - simple enum without OnceLock overhead  
- **Clear intent** - explicit source type at call sites
- **Performance** - direct dispatch without wasteful checks
- **Migration path** - can build upon current HashMap-based preloading

This approach preserves the architectural improvements (clean separation of concerns, HashMap preloading) while restoring the performance characteristics and type safety of the original design.

## Next Steps

1. Decide implementation strategy: build on current changes vs revert to PathType base
2. Implement PathSource enum with three variants
3. Update all call sites to use explicit source types  
4. Benchmark performance improvements
5. Consider this pattern for future loading subsystems

## Lessons Learned

- **Type-level information** is valuable for performance-critical paths
- **Architectural purity** shouldn't come at significant performance cost  
- **Compile-time dispatch** is often superior to runtime guessing
- **Iterative refinement** can combine benefits of multiple approaches

