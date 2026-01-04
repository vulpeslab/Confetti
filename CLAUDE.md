# Confetti - AI Agent Context

## Project Overview
**Confetti** is a high-performance Minecraft server fork (based on Purpur 1.21.1) that implements vertical scaling through multithreaded region-based ticking. Similar to Folia but uses a different approach to chunk management and locking.

## Architecture
Multi-threaded Purpur fork enabling vertical scaling. Chunks are grouped into **regions** (default 8x8 chunks), and regions are locked during ticking to prevent race conditions between threads. Code executes on chunk's thread, not a main thread.

## CRITICAL: Patch Workflow (DO NOT SKIP!)
**Changes made to `confetti-server/` or `confetti-api/` will be LOST if `applyPatches` is run!**

After making code changes, you MUST:
1. **Commit changes to subrepo first:**
   ```bash
   cd confetti-server && git add -A && git commit -m "Description of changes"
   ```
2. **Then rebuild patches:**
   ```bash
   cd .. && ./gradlew rebuildPatches
   ```
3. **Commit the new patch files:**
   ```bash
   git add patches/ && git commit -m "Update patches"
   ```

**Why?** The paperweight-patcher system generates `confetti-server/` from `patches/server/`. Running `applyPatches` regenerates the entire directory from patches, destroying any uncommitted work.

## Build Commands
```bash
./gradlew applyPatches                        # Patch upstream Paper (WARNING: overwrites confetti-server!)
./gradlew shadowjar createMojmapPaperclipJar  # Build final jar
./gradlew publishToMavenLocal                 # Publish API locally
./gradlew rebuildPatches                      # Update patches after code changes (commit to subrepo first!)
```
Output jar: `build/libs/confetti-paperclip-*-mojmap.jar`

## Test Server Environment
- `test-server/server.jar` is a **symlink** to the build output. No need to copy JARs manually after building.
- Launch with `cd test-server && ./start.sh`.
- Check `test-server/logs/latest.log` and `test-server/crash-reports/` for runtime issues.

## Project Structure
- `patches/api/` - API patches (branding, Folia compatibility checks)
- `patches/server/` - Server implementation patches (region scheduler, thread safety)
- `confetti-api/` - Generated API module
- `confetti-server/` - Generated server module
- `confetti.yml` - Runtime configuration (region-size, thread-count)

## Technologies
- **Language:** Java 21+
- **Build System:** Gradle (using the Paperweight framework)
- **Base Upstream:** Purpur -> Paper -> Spigot -> Bukkit
- **Platform:** Minecraft 1.21.1

### Prerequisites
- JDK 21 or higher
- Git (configured with user name and email)
- (macOS) `diffutils` installed via Homebrew (`brew install diffutils`)

## Multithreading Core Concepts
1. **Region locking**: When ticking a chunk, its region and 8 neighboring regions are locked. See `patches/server/0005-Add-multithreading-region-scheduler.patch` for `RegionPos` implementation.
2. **Thread execution**: Use `Bukkit.getRegionScheduler().run(plugin, location, task)` for block operations, `entity.getScheduler().run(plugin, task, null)` for entity operations.
3. **Thread-safety**: Use `ConcurrentHashMap` instead of `HashMap`, synchronized blocks for atomic operations. Never read mutable collections while another thread modifies them.

## Thread-Safety Patterns Used
- `SimpleStampedLock` - Optimistic read lock with write lock fallback (see `com.vulpeslab.confetti.util.SimpleStampedLock`)
- `synchronized` methods for atomic operations on shared state
- `volatile` fields for visibility across threads
- `AtomicBoolean`, `AtomicLong` for lock-free atomic updates
- Snapshot iteration: copy collections before iterating when modifications may occur
- ConcurrentLinkedDeque with poll-drain pattern for thread-safe queue processing

## Plugin Compatibility
Folia-supported plugins work without changes. Detection: check `Bukkit.class.getMethod("getRegionScheduler")`, not `RegionizedServer` class. Set `run-unsupported-plugins-in-sync: true` in config for unsupported plugins.

## Configuration Must-Knows
- `region-size`: Power of 2, min 8 recommended (lightning rods have 8-chunk radius)
- `thread-count`: `-1` = CPU cores - 1, `1` = disable multithreading
- `allow-unsupported-plugins-to-modify-chunks-via-global-scheduler`: Only works with single-threaded regions

## Development Guidelines

### Multithreading Conventions
- **Thread Safety:** Always assume code can be executed concurrently unless it's explicitly on the Global Scheduler.
- **Data Structures:** Prefer `ConcurrentHashMap` or synchronized collections for shared state.
- **Task Scheduling:** Use `Bukkit.getRegionScheduler()` or `entity.getScheduler()` instead of `Bukkit.getScheduler()`.
- **Async Operations:** Use `teleportAsync` instead of `teleport`.

### Important Files
- `README.md`: General project info and build instructions.
- `HOW_IT_WORKS.md`: Deep dive into the region locking architecture.
- `CONFETTI_YAML.md`: Documentation for `confetti.yml` configuration.
- `DEVELOPING_A_MULTITHREAD_PLUGIN.md`: Guide for plugin developers.
- `todos.md` & `bugs.md`: Current development focus and known critical issues.

## Testing Server with Bots
To test under load, use the Minecraft Stress Test tool:
```bash
java -Dbot.count=150 -Dbot.ip=127.0.0.1 -jar minecraft-stress-test-1.0.0-SNAPSHOT-jar-with-dependencies.jar
```

## Release Workflow
To release a new version via GitHub Actions:

```bash
# For subsequent releases (increments build number automatically):
gh workflow run Release --ref <branch>

# For FIRST release of a new MC version (use existing build.1 in gradle.properties):
gh workflow run Release --ref <branch> -f bump_version=false
```

**Important:** The workflow defaults to `bump_version=true`. For initial releases where `gradle.properties` already has `build.1`, you MUST use `-f bump_version=false` to avoid skipping to `build.2`.

## Version Info
Maven coordinates: `com.vulpeslab.confetti:confetti-api:1.21.1-R0.1-SNAPSHOT` from Clojars. Base Purpur commit in `gradle.properties`.
