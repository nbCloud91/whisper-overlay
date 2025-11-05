# Dependency Update Notes

This document describes the recent dependency updates and steps to complete the build.

## Changes Made

### RealtimeSTT Update

Updated the RealtimeSTT fork reference to the latest commit:

- **Previous version**: 0.1.16 (unknown commit, was tracking `master`)
- **New version**: 0.1.17-unstable-2024-06-20
- **Commit**: `41830135f99d7426710aca7d11e2338b9632bb7a`
- **Changes in fork**:
  - New frame event to reduce realtime processing load
  - Ability to return transcribed segments on demand
  - Logging options
  - Shutdown fixes

### Why We Still Use the Fork

The upstream RealtimeSTT (KoljaB/RealtimeSTT) has progressed to v0.3.104, but it lacks features that whisper-overlay requires:

1. **Segment return with probability data**: The fork provides detailed segment information including word-level timestamps and probability scores, which the Rust client uses for display.
2. **Custom modifications**: The fork includes specific fixes for shutdown handling and performance optimizations.

## Steps to Complete the Update

### 1. Fix the Source Hash

The Nix package currently uses `lib.fakeHash` as a placeholder. To get the correct hash:

```bash
# Try building - it will fail with the correct hash in the error message
nix build .#realtime-stt-server

# Copy the correct hash from the error message and update:
# nix/packages/realtime-stt.nix line 22
```

Alternatively, use nix-prefetch:
```bash
nix-prefetch-url --unpack https://github.com/oddlama/RealtimeSTT/archive/41830135f99d7426710aca7d11e2338b9632bb7a.tar.gz
```

### 2. Update Flake Lock

Update all flake inputs to their latest versions:

```bash
nix flake update
```

### 3. Test the Build

Verify everything builds correctly:

```bash
# Build the server
nix build .#realtime-stt-server

# Build the overlay
nix build .#whisper-overlay

# Or build everything
nix flake check
```

### 4. Test Functionality

If you're on NixOS, test with the service:

```bash
# For home-manager users
systemctl --user restart realtime-stt-server
whisper-overlay overlay

# For NixOS service users
sudo systemctl restart realtime-stt-server
whisper-overlay overlay
```

## Future Considerations

### Migrating to Upstream RealtimeSTT

To migrate to the upstream version (currently v0.3.104), we would need to:

1. Modify `realtime-stt-server.py` to work without segment data, OR
2. Wait for upstream to add segment return functionality
3. Verify that word-level probability data is available
4. Test shutdown behavior

The upstream has significant new features including:
- Thread-safe inter-process communication
- Audio normalization options
- Asynchronous callback execution
- Enhanced VAD callbacks (`on_vad_start`, `on_vad_stop`)
- Improved WebSocket-based server checks

## Related Files

- `nix/packages/realtime-stt.nix` - Python package definition
- `nix/packages/realtime-stt-server.nix` - Server wrapper package
- `realtime-stt-server.py` - Server implementation (uses RealtimeSTT)
- `flake.nix` - Main flake configuration
- `flake.lock` - Locked dependency versions
