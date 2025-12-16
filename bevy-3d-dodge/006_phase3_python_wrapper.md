# Phase 3: Python Gymnasium Wrapper

## Goal

Create a Python Gymnasium environment that interfaces with the Bevy game through the HTTP API, enabling RL training with Stable-Baselines3 and other frameworks.

## Tasks

### 1. Project Setup
- Initialize Python project with `uv` package manager
- Create `pyproject.toml` with dependencies
- Set up package structure under `python/bevy_dodge_env/`

### 2. Core Environment Implementation
- `BevyDodgeEnv` class inheriting from `gym.Env`
- HTTP client for API communication
- Proper space definitions matching Bevy backend
- Error handling and timeout management

### 3. Vectorization Support
- Wrapper for parallel environment instances
- Multi-process support via `SubprocVecEnv`
- Port management for concurrent Bevy instances

### 4. Testing & Validation
- Unit tests for environment functionality
- Integration tests with running Bevy server
- Random agent stress testing

## Timeline

Should take ~4 hours to complete initial implementation and testing.

## Success Criteria

- Can create environment and connect to API
- Random agent runs without errors
- Spaces match backend specifications
- Vectorized environments work correctly
