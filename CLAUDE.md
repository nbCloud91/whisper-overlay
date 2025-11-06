# CLAUDE.md - Maintenance Context for whisper-overlay

This document provides essential context for maintaining this fork of whisper-overlay.

## Project Overview

**whisper-overlay** is a Wayland overlay that provides speech-to-text functionality for any application via a global push-to-talk hotkey. It uses CUDA-accelerated faster-whisper models for real-time transcription.

- **Architecture**: Server-client model (Python server + Rust client)
- **Original Author**: oddlama (last active ~June 2024)
- **Current Status**: Dormant/community maintenance
- **This Fork**: Personal maintenance for ongoing use

### Key Components

1. **realtime-stt-server.py** - Python server using RealtimeSTT library
2. **whisper-overlay** (Rust) - Wayland client with GTK4 overlay using layer-shell protocol
3. **Nix packaging** - NixOS module, Home Manager module, and packages

## The Double Fork Situation

### 1. whisper-overlay Fork Status

- **Upstream**: `github:oddlama/whisper-overlay` (dormant since ~June 2024)
- **This Fork**: Personal maintenance for version bumps and updates
- **Strategy**: Minimal maintenance - keep dependencies updated, fix breaking changes

### 2. RealtimeSTT Dependency Fork

This is the **critical dependency issue** that affects the entire project:

#### oddlama's Fork (Currently Used)
- **Repo**: `github:oddlama/RealtimeSTT`
- **Version**: 0.1.16-0.1.17 era (June 2024)
- **Commit**: `41830135f99d7426710aca7d11e2338b9632bb7a` (last known good)
- **Branch**: Tracks `master`
- **Status**: Outdated, no updates since June 2024

**Custom Features in Fork** (needed by whisper-overlay):
- **Segment return with probability data** - Returns word-level timestamps and probability scores
- **Frame event** - Reduces realtime processing load
- **Shutdown fixes** - Proper cleanup on termination
- **Logging options** - Better debugging

#### KoljaB's Upstream (Origin)
- **Repo**: `github:KoljaB/RealtimeSTT`
- **Version**: v0.3.104+ (active development until recently)
- **Status**: Now community-maintained (KoljaB stepped back due to time constraints)

**New Features in Upstream** (not in fork):
- Thread-safe inter-process communication
- Audio normalization options
- Asynchronous callback execution
- Enhanced VAD callbacks (`on_vad_start`, `on_vad_stop`)
- Improved WebSocket-based server checks
- Many bug fixes and performance improvements

**The Problem**: whisper-overlay's `realtime-stt-server.py` depends on fork-specific APIs that don't exist in upstream.

## Current Maintenance Goals

### Short-term (Immediate)
1. ✅ Update Nix flake dependencies (nixpkgs, Python 3.11 → 3.13)
2. ⏳ Fix `lib.fakeHash` in `nix/packages/realtime-stt.nix`
3. ⏳ Update Nix package format to modern `pyproject` style
4. ⏳ Test builds and runtime functionality
5. ⏳ Keep core functionality working with latest NixOS

### Medium-term (Nice to have)
1. Update oddlama's RealtimeSTT fork to incorporate upstream fixes
2. Consider maintaining our own RealtimeSTT fork if oddlama's remains dormant
3. Monitor upstream RealtimeSTT for API compatibility
4. Update other Rust dependencies (GTK4, etc.)

### Long-term (Potential Major Work)
1. **Migrate to upstream RealtimeSTT v0.3.x+** (breaking change)
   - Requires rewriting `realtime-stt-server.py` to use new APIs
   - Need to verify segment/probability data availability
   - Test shutdown behavior with new version
2. Port to X11 (partial branch exists, has issues)
3. Add multi-client support (currently single client only)

## Critical Dependencies

### System Requirements
- **Wayland compositor** (sway, hyprland, wlroots-based)
- **CUDA support** (highly recommended for performance)
  - Without GPU: ~1s latency for live, ~5s for final transcription
  - With GPU: Real-time performance
- **User in `input` group** (for evdev hotkey detection)

### Python Dependencies (Server)
- **RealtimeSTT** (fork) - The critical dependency
- **faster-whisper** - Whisper model inference
- **torch/torchaudio** - PyTorch with CUDA
- **webrtcvad** - Voice activity detection
- **pyaudio** - Audio capture

### Rust Dependencies (Client)
- **gtk4** - UI framework
- **gtk4-layer-shell** - Wayland layer-shell protocol
- **evdev** - Global hotkey detection
- **tokio** - Async runtime

### Nix Flake Inputs
- **nixpkgs** - Main package set (currently outdated: June 2024)
- **flake-parts** - Flake organization
- **devshell** - Development environment
- **pre-commit-hooks** - Code quality

## Known Issues and Pitfalls

### 1. Nix Package Format Migration
**Problem**: Older Nix Python packages use deprecated attributes.

**Solution**: Update to modern format:
```nix
# OLD (deprecated)
nativeBuildInputs = [...];
propagatedBuildInputs = [...];

# NEW (correct)
pyproject = true;
build-system = [...];
dependencies = [...];
```

### 2. Hash Mismatch (lib.fakeHash)
**Problem**: `nix/packages/realtime-stt.nix:22` uses `lib.fakeHash` placeholder.

**Why**: The commit hash changed but the source hash wasn't updated.

**Solution**:
```bash
nix build .#realtime-stt-server  # Will fail with correct hash
# OR
nix-prefetch-url --unpack https://github.com/oddlama/RealtimeSTT/archive/<commit>.tar.gz
```

### 3. Python Version Lock-in
**Problem**: flake.lock pins old nixpkgs → old Python (3.11).

**Solution**: Always run `nix flake update` after checking out branches.

### 4. RealtimeSTT Fork Divergence
**Problem**: Upstream RealtimeSTT has moved far ahead (0.1.x → 0.3.x) with breaking changes.

**Risk**: Fork may become unmaintainable or miss critical bug fixes.

**Options**:
- A) Maintain own fork by cherry-picking upstream fixes
- B) Rewrite server to use upstream API (major work)
- C) Keep using old fork (security/bug risks)

### 5. Single Client Limitation
**Problem**: RealtimeSTT cannot process multiple requests simultaneously.

**Impact**: Only one user can transcribe at a time (network deployments affected).

**Workaround**: None currently - architectural limitation.

### 6. evdev Hotkey Detection
**Problem**: Using evdev instead of desktop portals requires `input` group membership.

**Why**: GlobalShortcuts portal doesn't work with layer-shell windows.

**Related**: https://github.com/bilelmoussaoui/ashpd/issues/213

### 7. CUDA Build Time
**Warning**: Enabling `nixpkgs.config.cudaSupport = true` triggers 2-3 hour build times.

**Reason**: Many packages must be rebuilt with CUDA support.

**Alternative**: Use docker-compose for server (pre-built CUDA images).

## Testing Checklist

When making updates, verify:

- [ ] `nix flake check` passes
- [ ] `nix build .#realtime-stt-server` succeeds
- [ ] `nix build .#whisper-overlay` succeeds
- [ ] Server starts without Python errors
- [ ] Hotkey detection works (Right Ctrl or configured key)
- [ ] Audio capture works
- [ ] Transcription appears in overlay
- [ ] Text is typed into focused window
- [ ] Waybar integration shows correct status
- [ ] NixOS/Home Manager modules still work

## File Reference

### Core Files
- `realtime-stt-server.py` - Server implementation (uses fork APIs)
- `src/main.rs` - Rust client entry point
- `flake.nix` - Nix flake definition
- `flake.lock` - Locked dependency versions (keep updated!)

### Nix Packages
- `nix/packages/realtime-stt.nix` - Python package for RealtimeSTT fork
- `nix/packages/realtime-stt-server.nix` - Wrapper for server script
- `nix/packages/whisper-overlay.nix` - Rust client package

### Nix Modules
- `nix/modules/nixos.nix` - NixOS service module
- `nix/modules/home-manager.nix` - Home Manager service module

### Documentation
- `README.md` - User documentation
- `UPDATING.md` - Temporary update instructions (created during updates)
- `CLAUDE.md` - This file

## RealtimeSTT Fork Maintenance Strategy

If oddlama's fork remains dormant, consider:

### Option A: Maintain Our Own Fork
1. Fork `oddlama/RealtimeSTT` to personal account
2. Cherry-pick relevant commits from `KoljaB/RealtimeSTT`
3. Update `nix/packages/realtime-stt.nix` to point to our fork
4. Document why each upstream feature is/isn't included

### Option B: Contribute to Community Maintenance
1. Since KoljaB's upstream is now community-maintained, engage with maintainers
2. Propose adding segment/probability APIs to upstream
3. Once available, migrate whisper-overlay to use upstream

### Option C: API Compatibility Layer
1. Keep using old fork for now
2. Create adapter layer in `realtime-stt-server.py` to abstract RealtimeSTT APIs
3. Makes future migration easier

**Recommendation**: Start with Option C (adapter layer) while monitoring upstream for API additions.

## Useful Commands

### Nix Operations
```bash
# Update all flake inputs
nix flake update

# Check flake validity
nix flake check

# Build specific package
nix build .#whisper-overlay
nix build .#realtime-stt-server

# Enter development shell
nix develop

# Run without installing
nix run . -- overlay --hotkey KEY_F12
```

### Testing
```bash
# Test NixOS module
sudo nixos-rebuild test --flake .#

# Test Home Manager module
home-manager switch --flake .#

# Manual server test
./result/bin/realtime-stt-server.py --debug

# Manual client test
./result/bin/whisper-overlay overlay --hotkey KEY_RIGHTCTRL
```

### Maintenance
```bash
# Check for outdated inputs
nix flake metadata

# See what would be updated
nix flake lock --update-input nixpkgs --dry-run

# Update specific input only
nix flake lock --update-input nixpkgs

# Prefetch new source hash
nix-prefetch-url --unpack <url>
```

## Links

- **This Project**: `github:oddlama/whisper-overlay`
- **RealtimeSTT Fork**: `github:oddlama/RealtimeSTT` (used by this project)
- **RealtimeSTT Upstream**: `github:KoljaB/RealtimeSTT` (now community-maintained)
- **faster-whisper**: `github:SYSTRAN/faster-whisper`
- **Layer Shell**: Protocol for Wayland overlays
- **evdev crate**: `docs.rs/evdev/latest/evdev/struct.Key.html`

## Version History

- **June 2024**: oddlama's last activity on both repos
- **November 2025**: Personal fork maintenance began
  - Updated flake.lock (Python 3.11 → 3.13)
  - Migrated to modern Nix Python package format
  - Fixed build issues

## Future Considerations

### When Upstream RealtimeSTT Adds Needed Features
If community maintainers add segment/probability APIs to upstream:

1. Test compatibility with our server
2. Create migration branch
3. Update `nix/packages/realtime-stt.nix` to use upstream
4. Update documentation
5. Announce to any users

### If This Fork Gets More Users
Consider:

- Setting up GitHub Issues/Discussions
- Creating release tags
- Adding CI/CD (GitHub Actions with Nix)
- Publishing to nixpkgs (upstream acceptance)
- Writing migration guide from oddlama's original

### If Wayland Protocols Change
Monitor for:

- layer-shell-protocol updates
- virtual-keyboard-v1 changes
- New global shortcuts portal support (would be better than evdev)

---

**Last Updated**: 2025-11-06
**Maintainer**: Personal fork for continued use
**Status**: Active minimal maintenance
