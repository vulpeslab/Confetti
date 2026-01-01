# Confetti - AI Agent Context

## Architecture
Multi-threaded 1.20.6 Purpur fork enabling vertical scaling. Chunks are grouped into **regions** (default 8x8 chunks), and regions are locked during ticking to prevent race conditions between threads. Code executes on chunk's thread, not a main thread.

## Critical Build Commands
```bash
./gradlew applyPatches              # Patch upstream Paper
./gradlew shadowjar createMojmapPaperclipJar  # Build final jar
./gradlew publishToMavenLocal      # Publish API locally
./gradlew rebuildPatches            # Update patches after code changes
```
Output jar: `build/libs/confetti-paperclip-*-mojmap.jar`

## Project Structure
- `patches/api/` - API patches (branding, Folia compatibility checks)
- `patches/server/` - Server implementation patches (region scheduler, thread safety)
- `confetti-api/` - Generated API module
- `confetti-server/` - Generated server module
- `confetti.yml` - Runtime configuration (region-size, thread-count)

## Multithreading Core Concepts
1. **Region locking**: When ticking a chunk, its region and 8 neighboring regions are locked. See `patches/server/0005-Add-multithreading-region-scheduler.patch` for `RegionPos` implementation.
2. **Thread execution**: Use `Bukkit.getRegionScheduler().run(plugin, location, task)` for block operations, `entity.getScheduler().run(plugin, task, null)` for entity operations.
3. **Thread-safety**: Use `ConcurrentHashMap` instead of `HashMap`, synchronized blocks for atomic operations. Never read mutable collections while another thread modifies them.

## Plugin Compatibility
Folia-supported plugins work without changes. Detection: check `Bukkit.class.getMethod("getRegionScheduler")`, not `RegionizedServer` class. Set `run-unsupported-plugins-in-sync: true` in config for unsupported plugins.

## Configuration Must-Knows
- `region-size`: Power of 2, min 8 recommended (lightning rods have 8-chunk radius)
- `thread-count`: `-1` = CPU cores - 1, `1` = disable multithreading
- `allow-unsupported-plugins-to-modify-chunks-via-global-scheduler`: Only works with single-threaded regions

## Version Info
Maven coordinates: `com.vulpeslab.confetti:confetti-api:1.20.6-R0.1-SNAPSHOT` from Clojars. Base Purpur commit in `gradle.properties`.
