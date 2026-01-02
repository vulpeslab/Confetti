# Confetti Bug Fix Todo List

## Critical Priority

### 1. Fix pendingBlockEntities NPE in LevelChunk.postProcessGeneration (NEW)
- [ ] Change `pendingBlockEntities` from `HashMap` to `ConcurrentHashMap` in `ChunkAccess.java` line 83
- [ ] Review all usages of pendingBlockEntities for thread-safety:
  - Line 188: `getBlockEntityNbtForSaving()`
  - Line 472: `setBlockEntityNbt()`
  - Line 585-602: NBT serialization methods
  - Line 836, 844: `postProcessGeneration()`
- [ ] Commit and rebuild patches
- [ ] Test with 500+ bots

**Location:** `confetti-server/src/main/java/net/minecraft/world/level/chunk/ChunkAccess.java`

**Quick Fix:**
```java
// Before:
protected final Map<BlockPos, CompoundTag> pendingBlockEntities = Maps.newHashMap();

// After:
protected final Map<BlockPos, CompoundTag> pendingBlockEntities = new ConcurrentHashMap<>();
```

### 2. Fix OutOfMemoryError
- [ ] Profile memory usage with VisualVM or JFR during 500+ player test
- [ ] Check for ThreadLocal leaks in entity tracking (`updatingSectionStatus` field)
- [ ] Review ReentrantLock allocation per entity - consider using lock striping instead
- [ ] Check if `sectionLock` per entity is causing excessive object creation
- [ ] Review `entityLock` in ChunkEntitySlices for memory impact
- [ ] Test with increased heap size (4-8GB) to isolate memory issues from logic bugs
- [ ] Add `-XX:+HeapDumpOnOutOfMemoryError` to capture heap dump on crash

### 3. Fix Double Calling Chunk Load
- [ ] Add atomic boolean guard in `LevelChunk.loadCallback()` (line 661)
- [ ] Review `ChunkSystem.onChunkBorder()` (line 91) for thread-safety
- [ ] Check `NewChunkHolder.handleFullStatusChange()` (line 1281) for race conditions
- [ ] Add synchronization around chunk status transitions in `ChunkHolderManager.processPendingFullUpdate()`
- [ ] Ensure `loadCallback` is idempotent or properly guarded

**Quick Fix:**
```java
// Add field:
private final AtomicBoolean loadCallbackCalled = new AtomicBoolean(false);

// In loadCallback():
public void loadCallback() {
    if (!this.loadCallbackCalled.compareAndSet(false, true)) {
        return; // Already called
    }
    // ... existing code
}
```

---

## High Priority

### 4. Fix Entity Status Update Timeout
- [ ] Replace bounded spin-lock in `ChunkEntitySlices.startPreventingStatusUpdates()` with proper `ReentrantLock`
- [ ] Change from `AtomicBoolean` with spin-wait to blocking lock:
  ```java
  // Replace spin-lock pattern with:
  private final ReentrantLock statusUpdateLock = new ReentrantLock();
  public void lockStatusUpdates() { statusUpdateLock.lock(); }
  public void unlockStatusUpdates() { statusUpdateLock.unlock(); }
  ```
- [ ] Update `EntityLookup` to use new blocking lock instead of `startPreventingStatusUpdates()`
- [ ] Remove the 1000-iteration limit that causes timeouts

---

## Medium Priority

### 5. Fix Player Moved Wrongly After Teleport
- [ ] Review `ServerPlayer.teleportTo()` for position sync issues
- [ ] Ensure `lastGoodX/Y/Z` fields are updated after teleport
- [ ] Check if `absMoveTo()` properly resets movement validation state
- [ ] Verify teleport sets position on correct thread for cross-region teleports
- [ ] Review `ServerGamePacketListenerImpl.handleMovePlayer()` validation logic
- [ ] Consider adding teleport cooldown for movement validation

### 6. Improve Server Performance Under Load
- [ ] Profile lock contention with async-profiler
- [ ] Review if read-write locks can replace exclusive locks where possible
- [ ] Check for unnecessary synchronization in hot paths
- [ ] Consider lock-free data structures for frequently accessed collections
- [ ] Review GC settings - consider G1GC tuning or ZGC for lower latency

---

## Code Locations Reference

| Task | File | Line/Method |
|------|------|-------------|
| pendingBlockEntities NPE | `ChunkAccess.java` | line 83, declaration |
| pendingBlockEntities NPE | `LevelChunk.java` | `postProcessGeneration()` line 836 |
| Double chunk load | `LevelChunk.java` | `loadCallback()` line 661 |
| Double chunk load | `ChunkSystem.java` | `onChunkBorder()` line 91 |
| Double chunk load | `NewChunkHolder.java` | `handleFullStatusChange()` line 1281 |
| Status update lock | `ChunkEntitySlices.java` | `startPreventingStatusUpdates()` |
| Entity status | `EntityLookup.java` | `entityStatusChange()` |
| Teleport | `ServerPlayer.java` | `teleportTo()` |
| Movement validation | `ServerGamePacketListenerImpl.java` | `handleMovePlayer()` |

---

## Priority Order for Fixing

1. **TODO 1** - pendingBlockEntities NPE (quick fix, crashes server)
2. **TODO 3** - Double chunk load (quick fix, prevents corruption)
3. **TODO 4** - Status update timeout (medium effort, reduces errors)
4. **TODO 5** - Teleport movement validation (medium effort, improves UX)
5. **TODO 2** - OOM investigation (needs profiling tools)
6. **TODO 6** - Performance investigation (ongoing)

---

## Testing Checklist

After fixes, verify with:
- [ ] 100 bot players - no errors for 5 minutes
- [ ] 250 bot players - no errors for 5 minutes  
- [ ] 500 bot players - no errors for 5 minutes
- [ ] Mass teleport command (`/tp @a <player>`) - no "moved wrongly" warnings
- [ ] No "Double calling chunk load" errors
- [ ] No "Cannot update chunk status" errors
- [ ] Server maintains 20 TPS with 500 players
- [ ] No OOM with 4GB heap and 500 players for 10 minutes

---

## Completed Fixes (Reference)

- [x] Entity fluidHeight thread-safety (Patch 0052)
- [x] Thread-safe command execution - Kill/Give/Effect (Patch 0053)
- [x] NPE fix in getInBlockState (Patch 0054)
- [x] EntityList/ChunkEntitySlices concurrent access (Patch 0055)
