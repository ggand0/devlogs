# Devlog 029: gRPC Implementation for RL Communication

## Overview

Replaced HTTP communication with gRPC to improve CNN training performance. The main goals were:
- Eliminate Base64 encoding overhead for image observations (~33% data reduction)
- Use Unix domain sockets for lower latency IPC
- Maintain backward compatibility with HTTP transport

## Performance Results

### After Step Counter Synchronization (v2)

| Transport | Image Size | FPS | Notes |
|-----------|------------|-----|-------|
| gRPC (Unix socket) | 256x256 grayscale | ~236 FPS | With `--fps 240` |
| gRPC (Unix socket) | 64x64 grayscale | ~236 FPS | Same ceiling |
| Vector observations | 65-dim float | ~225 FPS | Standard mode |

**4.4x faster than initial gRPC implementation!**

The bottleneck was artificial sleep delays, not the transport. Key optimizations:
1. Replaced 16ms sleep in `step()` with step counter synchronization
2. Reduced reset sleep from 100ms to 20ms
3. Game loop FPS is now the ceiling (use `--fps 240` or higher)

### Initial gRPC Implementation (v1)

| Transport | Image Size | FPS |
|-----------|------------|-----|
| gRPC (Unix socket) | 256x256 grayscale | ~54-56 FPS |
| HTTP | 256x256 grayscale | ~50 FPS |

The initial 10-12% improvement came from:
- Raw bytes instead of Base64 encoding (saves ~33% bandwidth)
- Protocol buffers (more efficient serialization than JSON)
- Unix sockets have lower latency than TCP

## Implementation Details

### Rust Side

**New Files:**
- `proto/rl_env.proto` - Protocol buffer definitions for all messages and service
- `build.rs` - tonic-build configuration for proto compilation
- `src/rl/grpc_api.rs` - gRPC server implementation (~450 lines)

**Modified Files:**
- `Cargo.toml` - Added tonic, prost, tokio-stream dependencies
- `src/main.rs` - Added CLI flags for transport selection
- `src/rl/mod.rs` - Export grpc_api module

**Dependencies Added:**
```toml
tonic = "0.12"
prost = "0.13"
tokio-stream = { version = "0.1", features = ["net"] }
hyper-util = { version = "0.1", features = ["tokio"] }
async-stream = "0.3"

[build-dependencies]
tonic-build = "0.12"
```

**CLI Flags:**
- `--socket-path <PATH>` - Unix socket path (default: `/tmp/bevy_rl.sock`)
- `--grpc-port <PORT>` - Use TCP instead of Unix socket
- `--http` - Fall back to HTTP transport

### Python Side

**New Files:**
- `python/bevy_dodge_env/grpc_client.py` - Low-level gRPC client
- `python/bevy_dodge_env/rl_env_pb2.py` - Generated protobuf code
- `python/bevy_dodge_env/rl_env_pb2_grpc.py` - Generated gRPC stubs
- `python/compile_proto.sh` - Script to regenerate proto files

**Modified Files:**
- `python/bevy_dodge_env/environment.py` - Added transport abstraction
- `python/bevy_dodge_env/vec_env.py` - Added gRPC support for vectorized envs
- `pyproject.toml` - Added grpcio dependencies

**Dependencies Added:**
```
grpcio>=1.60.0
grpcio-tools>=1.60.0
protobuf>=4.25.0
```

## Key Technical Decisions

### 1. Transport Selection
Both HTTP and gRPC are supported. gRPC is the default, but HTTP can be enabled with `--http` flag for debugging or compatibility.

### 2. Unix Socket vs TCP
Unix domain sockets are used by default for gRPC because:
- Lower latency than TCP (no network stack overhead)
- Simpler configuration (no port conflicts)
- TCP option available via `--grpc-port` for cross-machine scenarios

### 3. Image Observation Format
gRPC sends raw bytes directly in the `image_observation` field, eliminating the need for Base64 encoding that HTTP requires. This saves ~33% bandwidth for image data.

## Step Counter Synchronization

The gRPC handler needs to wait for fresh observations from the game loop. Initially this used a 16ms sleep, which capped FPS at ~60.

**Solution**: Added a step counter that the game loop increments after updating observations. The gRPC handler waits for the counter to change before returning.

```rust
// In SharedEnvState:
pub step_counter: Arc<RwLock<u64>>,

// In main.rs game loop, after updating observation:
*shared_state.step_counter.blocking_write() += 1;

// In grpc_api.rs step():
let current_step = *self.shared_state.step_counter.read().await;
self.command_tx.send(command)?;

// Wait for counter to increment (max 100ms timeout)
loop {
    let new_step = *self.shared_state.step_counter.read().await;
    if new_step > current_step {
        break;
    }
    tokio::task::yield_now().await;
}
```

This eliminates artificial delays and allows FPS to match the game loop rate.

## Unix Socket Bug Fix

Encountered `RST_STREAM with error code 1` when connecting Python gRPC client to tonic server via Unix socket.

**Root Cause:** Python grpc 1.57+ percent-encodes the authority header in Unix socket paths, causing protocol errors with some servers.

**Fix:** Set `grpc.default_authority` option in Python client:
```python
self.channel = grpc.insecure_channel(
    f"unix://{socket_path}",
    options=[("grpc.default_authority", "localhost")],
)
```

**References:**
- https://github.com/hyperium/tonic/issues/742
- https://github.com/grpc/grpc/issues/34305

## Usage Examples

### Starting the Game Server

```bash
# gRPC with Unix socket (default) - use --fps for higher training throughput
./target/release/bevy_3d_dodge --headless --fps 240 --socket-path /tmp/bevy_rl.sock

# gRPC with TCP
./target/release/bevy_3d_dodge --headless --fps 240 --grpc-port 50051

# HTTP (fallback)
./target/release/bevy_3d_dodge --headless --http --port 8000
```

### Python Client

```python
from bevy_dodge_env.environment import BevyDodgeEnv

# gRPC with Unix socket (default)
env = BevyDodgeEnv()

# gRPC with custom socket path
env = BevyDodgeEnv(socket_path="/tmp/my_socket.sock")

# HTTP fallback
env = BevyDodgeEnv(transport="http", port=8000)
```

### Vectorized Environments

```python
from bevy_dodge_env.vec_env import make_vec_env

# gRPC (default) - requires multiple game instances with different sockets
vec_env = make_vec_env(
    n_envs=4,
    socket_base="/tmp/bevy_rl",  # Creates /tmp/bevy_rl_0.sock, etc.
    config_kwargs={'observation_mode': 'topdown'}
)

# HTTP
vec_env = make_vec_env(
    n_envs=4,
    start_port=8000,
    transport="http",
    config_kwargs={'observation_mode': 'topdown'}
)
```

## Proto Schema

The proto file defines all message types for the RL environment API:

```protobuf
service RlEnvironment {
  rpc GetObservationSpace(ObservationSpaceRequest) returns (ObservationSpace);
  rpc GetActionSpace(ActionSpaceRequest) returns (ActionSpace);
  rpc Reset(ResetRequest) returns (ResetResponse);
  rpc Step(StepRequest) returns (StepResponse);
  rpc Configure(ConfigureRequest) returns (ConfigureResponse);
  rpc SetLevel(SetLevelRequest) returns (SetLevelResponse);
  rpc StartTraining(StartTrainingRequest) returns (StartTrainingResponse);
  rpc EndTraining(EndTrainingRequest) returns (EndTrainingResponse);
}
```

Key message types:
- `StepRequest/StepResponse` - Core RL step with action and observation
- `ConfigureRequest` - Game configuration (level, observation mode, image size, etc.)
- `ResetResponse` - Initial observation after environment reset

## Regenerating Proto Files

If the proto schema changes:

```bash
# Rust (automatic via build.rs)
PROTOC=~/.local/bin/protoc cargo build

# Python
cd python && ./compile_proto.sh
```

Note: The `compile_proto.sh` script includes a fix for relative imports in the generated Python code.

## Future Improvements

1. ~~**Streaming RPC**: Could use bidirectional streaming for even lower latency~~ - Not needed, step counter sync achieves 236 FPS with unary calls.

2. **Higher FPS**: Game loop can run faster than 240 FPS with `--fps` flag. At some point, GPU rendering becomes the bottleneck for image observations.

3. **Shared Memory**: For even higher throughput, could use shared memory (mmap) instead of Unix sockets to eliminate data copying entirely. Would require significant refactoring.
