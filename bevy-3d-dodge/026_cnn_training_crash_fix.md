# Devlog 026: CNN Training Crash Fix - Connection Reset Bug

## Date: 2025-12-23

## Problem

CNN-based top-down image observation training randomly crashes with "Connection reset by peer" error. The training runs fine for some time (typically 50-60 episodes, ~2500 steps) then suddenly fails during an environment reset.

### Error Signature

```
requests.exceptions.ChunkedEncodingError: ("Connection broken:
ConnectionResetError(104, 'Connection reset by peer')", ConnectionResetError(104,
'Connection reset by peer'))

RuntimeError: Failed to reset environment: ("Connection broken:
ConnectionResetError(104, 'Connection reset by peer')", ...)
```

### Observations

1. **Only affects CNN setup** - MLP training with vector observations works fine
2. **Random timing** - Crashes after varying number of episodes (not deterministic)
3. **Server doesn't crash** - Bevy server continues running, can accept new connections
4. **No server-side errors** - Server logs show normal "Game Over!" messages, no panics
5. **Happens during reset** - Error occurs when SB3 auto-resets after episode ends

### Server Log Pattern

```
2025-12-22T23:47:11.980369Z  INFO bevy_3d_dodge::game::collision: Game Over!
2025-12-22T23:47:12.999429Z  INFO bevy_3d_dodge::game::collision: Game Over!
2025-12-22T23:47:14.002053Z  INFO bevy_3d_dodge::game::collision: Game Over!
2025-12-22T23:47:14.792350Z  INFO bevy_3d_dodge: Training mode DISABLED (headless)
```

The "Training mode DISABLED" appears because Python's `finally` block runs after the crash, successfully calling `end_training()`.

## Root Cause Analysis

### Key Differences: CNN vs MLP

| Aspect | MLP (works) | CNN (crashes) |
|--------|-------------|---------------|
| Response size | ~260 bytes | ~262KB (base64 image) |
| Per-frame allocation | Minimal | ~200KB Vec every frame |
| Lock frequency | Low | High (image buffer) |

### Hypothesis: Mutex Contention + Memory Churn

The crash is likely caused by a combination of:

1. **Mutex contention between game loop and HTTP handlers**
   - Game loop (Bevy thread): `blocking_lock()` to write image 120 times/sec
   - API handlers (Tokio threads): `lock().await` to read and clone image
   - Under load, handlers can block each other and the game loop

2. **Memory allocation pressure**
   - `generate_synthetic_topdown_image()` allocates new 196KB `Vec<u8>` every frame
   - Old buffer dropped every frame
   - At 120 FPS = ~24MB/sec allocation churn
   - Can cause GC pauses, fragmentation, or occasional allocation failures

3. **Lock held during large clone**
   - API handlers clone 200KB while holding lock
   - Blocks game loop from writing updates
   - Occasional timing issues cause HTTP connection to fail

### Why Connection Gets Reset

The HTTP response fails mid-stream because:
1. HTTP handler acquires read lock
2. Starts encoding/sending 262KB base64 response
3. Game loop blocks waiting for write lock
4. Timing cascade causes tokio task or connection to fail
5. TCP connection reset sent to client

## Solution

### 1. Switch from Mutex to RwLock

**File:** `src/rl/api.rs`

Changed `SharedEnvState` from `tokio::sync::Mutex` to `tokio::sync::RwLock`:

```rust
// Before
pub struct SharedEnvState {
    pub observation: Arc<Mutex<Vec<f32>>>,
    pub image_observation: Arc<Mutex<Vec<u8>>>,
    // ...
}

// After
pub struct SharedEnvState {
    pub observation: Arc<RwLock<Vec<f32>>>,
    pub image_observation: Arc<RwLock<Vec<u8>>>,
    // ...
}
```

**Benefit:** Multiple API handlers can read concurrently. Only blocks when game loop needs to write.

### 2. Use Read Locks in API Handlers

**File:** `src/rl/api.rs`

```rust
// Before
let image_bytes = state.shared_state.image_observation.lock().await.clone();

// After
let image_bytes = state.shared_state.image_observation.read().await.clone();
```

### 3. Use Write Locks in Game Loop

**File:** `src/main.rs`

```rust
// Before
*shared_state.image_observation.blocking_lock() = image_pixels;

// After
*shared_state.image_observation.blocking_write() = observation;
```

### 4. In-Place Image Generation (Eliminate Allocation)

**File:** `src/rl/image_observation.rs`

Added new function that writes directly to existing buffer:

```rust
/// Generate a synthetic top-down image into an existing buffer (avoids allocation)
pub fn generate_synthetic_topdown_image_into(
    pixels: &mut [u8],
    player_pos: Vec3,
    projectile_positions: &[(Vec3, Vec3)],
    thrower_pos: Option<Vec3>,
    arena_size: f32,
) {
    // ... writes directly to pixels buffer
}
```

**File:** `src/main.rs`

```rust
// Before: Allocate new Vec, then replace
let image_pixels = generate_synthetic_topdown_image(...);
*shared_state.image_observation.blocking_lock() = image_pixels;

// After: Update existing buffer in-place
let mut image_buffer = shared_state.image_observation.blocking_write();
generate_synthetic_topdown_image_into(&mut image_buffer, ...);
drop(image_buffer); // Release lock explicitly
```

**Benefit:** Eliminates ~24MB/sec allocation churn.

### 5. Increased HTTP Timeout

**File:** `python/bevy_dodge_env/environment.py`

```python
# Before
timeout: float = 5.0

# After
timeout: float = 30.0  # Increased from 5.0 for image observations
```

**Benefit:** More tolerance for occasional slow responses.

## Summary of Changes

| File | Change |
|------|--------|
| `src/rl/api.rs` | `Mutex` -> `RwLock` for SharedEnvState |
| `src/rl/api.rs` | `.lock().await` -> `.read().await` in handlers |
| `src/main.rs` | `.blocking_lock()` -> `.blocking_write()` |
| `src/main.rs` | Use in-place image generation |
| `src/rl/image_observation.rs` | Add `generate_synthetic_topdown_image_into()` |
| `python/bevy_dodge_env/environment.py` | Timeout 5s -> 30s |

## Phase 2: Additional Fixes (Still Occurring After Phase 1)

After the initial RwLock + in-place generation fix, the crash still occurred but ran longer (~21k steps vs ~2.6k steps before). This suggests the changes helped but didn't fully resolve the issue.

### Additional Changes Made

**1. Pre-allocated base64 buffer** (`src/rl/api.rs`)

Instead of letting BASE64.encode() allocate its own buffer:
```rust
// Before
Some(BASE64.encode(&image_bytes))

// After
let mut base64_string = String::with_capacity((image_bytes.len() * 4 / 3) + 4);
BASE64.encode_string(&image_bytes, &mut base64_string);
Some(base64_string)
```

**2. Connection pooling with Session** (`python/bevy_dodge_env/environment.py`)

Use a persistent HTTP session with connection pooling:
```python
self._session = requests.Session()
retry_strategy = Retry(
    total=3,
    backoff_factor=0.1,
    status_forcelist=[500, 502, 503, 504],
)
adapter = HTTPAdapter(max_retries=retry_strategy, pool_maxsize=10)
self._session.mount("http://", adapter)
```

Benefits:
- Reuses TCP connections (keep-alive)
- Built-in retry for transient errors
- Connection pooling reduces connection churn

**3. Manual retry logic for connection errors** (`python/bevy_dodge_env/environment.py`)

Added exponential backoff retry for ChunkedEncodingError and ConnectionError:
```python
for attempt in range(max_retries):
    try:
        response = self._session.post(url, json=data, timeout=self.timeout)
        ...
    except (ChunkedEncodingError, ConnectionError) as e:
        if attempt < max_retries - 1:
            wait_time = 0.1 * (2 ** attempt)  # 0.1s, 0.2s, 0.4s
            time.sleep(wait_time)
            continue
        raise
```

**4. Reduced headless FPS** (`src/main.rs`)

Changed default from 120 FPS to 60 FPS:
- At 120 FPS, game writes to shared state every 8.3ms
- API step handler waits 16ms, so 2 frames per step is redundant
- Lower frequency = less lock contention
- Training bottleneck is GPU/Python, not game FPS

**5. Increased reset wait time** (`src/rl/api.rs`)

Reset handler now waits 100ms instead of 50ms after sending reset command.

## Testing

Build successful. Changes need to be tested with actual training run to verify the fix.

```bash
# Terminal 1 - note: now defaults to 60 FPS
# No Vulkan vars needed for headless mode (no rendering)
cargo run --release -- --headless

# Terminal 2
uv run python python/train_sac.py --config python/configs/sac_topdown_cnn.yaml
```

## Lessons Learned

1. **Mutex vs RwLock matters** - When you have one writer and many readers, RwLock provides much better concurrency
2. **Allocation churn can cause subtle issues** - Even if allocations succeed, the memory management overhead can cause timing issues
3. **Large data transfer over HTTP needs care** - 262KB responses at high frequency stress the connection
4. **"Connection reset" often means server-side timing issue** - Not always a network problem

## Future Improvements

If issues persist:
1. Add connection pooling/keep-alive for HTTP
2. Implement proper async readback from GPU instead of CPU rendering
3. Consider WebSocket for lower-latency continuous streaming
4. Add metrics/logging around lock wait times
