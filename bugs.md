# Confetti Bug Tracker

This document tracks known bugs and issues identified during testing with ~500 bot players.

---

## Critical Issues

### 1. NullPointerException in LevelChunk.postProcessGeneration (NEW - Server Crash)
**Severity:** Critical  
**Status:** Open  
**First Seen:** 2026-01-02

**Description:**  
Server crashes with NPE when iterating over `pendingBlockEntities.keySet()` because a null key was inserted during concurrent access.

**Error:**
```
java.lang.NullPointerException: at index 3
    at com.google.common.collect.ObjectArrays.checkElementNotNull(ObjectArrays.java:232)
    at com.google.common.collect.ImmutableList.copyOf(ImmutableList.java:265)
    at net.minecraft.world.level.chunk.LevelChunk.postProcessGeneration(LevelChunk.java:836)
    at ca.spottedleaf.moonrise.common.util.ChunkSystem.onChunkTicking(ChunkSystem.java:111)
    at ca.spottedleaf.moonrise.patches.chunk_system.scheduling.NewChunkHolder.handleFullStatusChange(NewChunkHolder.java:1281)
```

**Root Cause:**
`pendingBlockEntities` in `ChunkAccess.java:83` is a regular `HashMap` being accessed concurrently by multiple threads, causing null keys to appear.

**Location:**
- `ChunkAccess.java:83` - `protected final Map<BlockPos, CompoundTag> pendingBlockEntities = Maps.newHashMap();`
- `LevelChunk.java:836` - `ImmutableList.copyOf(this.pendingBlockEntities.keySet())`

**Fix Required:**
Replace `HashMap` with `ConcurrentHashMap` in ChunkAccess.

---

### 2. OutOfMemoryError - Server Crash
**Severity:** Critical  
**Status:** Open  
**First Seen:** 2026-01-02

**Description:**  
Server runs out of memory when 500+ players are online, causing complete server crash.

**Error:**
```
Exception: java.lang.OutOfMemoryError thrown from the UncaughtExceptionHandler in thread "Watchdog Thread"
Exception: java.lang.OutOfMemoryError thrown from the UncaughtExceptionHandler in thread "JNA Cleaner"
Exception: java.lang.OutOfMemoryError thrown from the UncaughtExceptionHandler in thread "Server console handler"
./start.sh: line 1:  3917 Killed: 9               java -Xmx2G -jar server.jar --nogui
```

**Possible Causes:**
- Memory leak in entity/chunk handling
- Per-entity locks creating too many ReentrantLock objects
- ThreadLocal objects not being cleaned up
- 2GB heap may be insufficient for 500+ players

**Suggested Investigation:**
- Profile memory usage with VisualVM/JFR
- Check for ThreadLocal leaks in entity tracking
- Review lock object allocation patterns
- Consider increasing heap size for testing

---

### 3. Double Calling Chunk Load
**Severity:** High  
**Status:** Open  
**First Seen:** 2026-01-02

**Description:**  
`LevelChunk.loadCallback()` is being called multiple times for the same chunk, indicating a race condition in chunk loading.

**Error:**
```
[ConfettiTickThread-5/ERROR]: Double calling chunk load!
java.lang.Throwable: null
    at net.minecraft.world.level.chunk.LevelChunk.loadCallback(LevelChunk.java:661)
    at ca.spottedleaf.moonrise.common.util.ChunkSystem.onChunkBorder(ChunkSystem.java:91)
    at ca.spottedleaf.moonrise.patches.chunk_system.scheduling.NewChunkHolder.handleFullStatusChange(NewChunkHolder.java:1281)
    at ca.spottedleaf.moonrise.patches.chunk_system.scheduling.ChunkHolderManager.processPendingFullUpdate(ChunkHolderManager.java:1375)
    at ca.spottedleaf.moonrise.patches.chunk_system.scheduling.ChunkHolderManager.processTicketUpdates(ChunkHolderManager.java:1359)
```

**Location:**
- `LevelChunk.java:661` - loadCallback()
- `ChunkSystem.java:91` - onChunkBorder()
- `NewChunkHolder.java:1281` - handleFullStatusChange()

**Occurs:** Multiple times during gameplay, from various ConfettiTickThreads

**Suggested Fix:**
- Add atomic flag to prevent double loading
- Review `handleFullStatusChange` for thread-safety
- Add synchronization around chunk status transitions

---

### 4. Cannot Update Chunk Status for Entity
**Severity:** High  
**Status:** Open  
**First Seen:** 2026-01-02

**Description:**  
EntityLookup fails to update entity chunk status because the chunk is already receiving an update. This indicates the status update spin-lock is timing out.

**Error:**
```
[ConfettiTickThread-6/ERROR]: [EntityLookup] Cannot update chunk status for entity ServerPlayer['Bot94'/1779...] 
since entity chunk (5,1) is receiving update
```

**Affected Entities:** ServerPlayer, Rabbit (and likely all entity types)

**Location:**
- `EntityLookup.java` - entity status change handling
- `ChunkEntitySlices.java` - `startPreventingStatusUpdates()` spin-lock

**Root Cause:**
The `startPreventingStatusUpdates()` method uses a bounded spin-lock (1000 iterations) that times out when contention is high, causing status updates to be skipped.

**Suggested Fix:**
- Replace bounded spin-lock with proper ReentrantLock with blocking
- Increase spin count or add exponential backoff
- Review why chunks are held in "receiving update" state for long periods

---

## Medium Issues

### 5. Player Moved Wrongly / Moved Too Quickly
**Severity:** Medium  
**Status:** Open  
**First Seen:** 2026-01-02

**Description:**  
Players are being flagged for invalid movement after teleportation. This causes rubber-banding and incorrect position validation.

**Errors:**
```
[Server thread/WARN]: Bot483 moved wrongly!, (0.14523200451660045)
[Server thread/WARN]: Bot20 moved wrongly!, (-0.010000000000005116)
[Server thread/WARN]: Bot213 moved too quickly! -150.77,-13.77,176.50
```

**Context:**  
Occurs after `/tp @a InfinityBytes` command that teleported 501 entities.

**Root Cause:**
After teleportation, the server's expected player position may not be synchronized correctly with the actual teleported position, causing movement validation to fail.

**Suggested Fix:**
- Ensure teleportation properly updates all position tracking variables
- Reset movement validation state after teleport
- Check if position is being set on correct thread after cross-region teleport

---

### 6. Server Can't Keep Up
**Severity:** Medium  
**Status:** Open  
**First Seen:** 2026-01-02

**Description:**  
Server is consistently falling behind on ticks when many players are online.

**Errors:**
```
[Server thread/WARN]: Can't keep up! Is the server overloaded? Running 2601ms or 52 ticks behind
[Server thread/WARN]: Can't keep up! Is the server overloaded? Running 8446ms or 168 ticks behind
```

**Context:**  
- ~500 players online
- Server falling up to 168 ticks (8.4 seconds) behind
- Eventually leads to OOM and crash

**Possible Causes:**
- Lock contention in entity/chunk systems
- Too many synchronization points slowing down parallel execution
- Memory pressure causing GC pauses

---

## Low Priority / Informational

### 7. Watchdog Thread Dump
**Severity:** Low (Informational)  
**Status:** N/A

**Description:**  
Server not responding for 10+ seconds, watchdog creating thread dump.

**Error:**
```
[Watchdog Thread/ERROR]: --- DO NOT REPORT THIS TO CONFETTI - THIS IS NOT A BUG OR A CRASH ---
[Watchdog Thread/ERROR]: The server has not responded for 10 seconds! Creating thread dump
```

**Context:**  
This is a symptom of the above issues (OOM, lock contention) rather than a bug itself.

---

## Summary Table

| # | Issue | Severity | Component | Status |
|---|-------|----------|-----------|--------|
| 1 | NPE in postProcessGeneration | Critical | ChunkAccess | Open |
| 2 | OutOfMemoryError | Critical | Memory/Heap | Open |
| 3 | Double Chunk Load | High | ChunkSystem | Open |
| 4 | Entity Status Update Timeout | High | EntityLookup | Open |
| 5 | Player Moved Wrongly | Medium | Teleportation | Open |
| 6 | Server Can't Keep Up | Medium | Performance | Open |
| 7 | Watchdog Thread Dump | Low | Monitoring | N/A |

---

## Testing Environment

- **Version:** Confetti 1.21-DEV-e83d259
- **Java:** OpenJDK 21.0.9
- **Heap Size:** 2GB (`-Xmx2G`)
- **Players:** ~500 bots
- **Date:** 2026-01-02

---

## Related Patches

The following patches were recently applied and may need review:

1. **0052** - Entity fluidHeight thread-safety (Object2DoubleMaps.synchronize)
2. **0053** - Thread-safe command execution (ensureSync for Kill/Give/Effect)
3. **0054** - NPE fix in getInBlockState concurrent access
4. **0055** - EntityList/ChunkEntitySlices concurrent access crash fix

---

## Next Steps

1. **Fix pendingBlockEntities** - Replace HashMap with ConcurrentHashMap in ChunkAccess
2. **Memory Profiling** - Run with JFR/VisualVM to identify memory leaks
3. **Fix Double Chunk Load** - Add atomic guard in `LevelChunk.loadCallback()`
4. **Fix Status Update Lock** - Replace spin-lock with proper blocking lock
5. **Teleport Validation** - Review teleport code for position sync issues
6. **Increase Test Heap** - Try with 4-8GB heap to separate memory issues from logic bugs
