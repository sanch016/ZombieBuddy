# Unofficial community fix for Project Zomboid 42.21

> **This is an unofficial fork.** MIT license and all original copyright notices
> (c) 2025-2026 Andrey "Zed" Zaikin are preserved. This build is **not** released
> by the original author.

## The bug

Since Project Zomboid **42.21.0** the game changed `ZomboidFileSystem.loadMods(...)`
to accept `java.util.List<String>` instead of `java.util.ArrayList<String>`:

```java
public void loadMods(List<String> mods, long ...) { ... }
```

ZombieBuddy's patch advice mapped `loadMods` with arguments declared as
`ArrayList<String>`. `PatchEngine` applies an advice only when the advice argument
type is assignable from the target method's argument type:

```java
if ( expected == Object.class || actual.isAssignableTo(expected) ) { ... }
```

`List.isAssignableTo(ArrayList)` is `false`, so the advice **silently skipped**. As
a result Java mods were never loaded: no `Exposing class to Lua` lines, 3D
mods (PZ3D, ATOMICSTEXTURES3D, and any other `javaJarFile` mod) stayed inactive.

## The fix

Declare the advice parameters as `List<String>` and cast back to
`ArrayList<String>` (in 42.21 the actual runtime argument is still a real
`ArrayList`, so the cast is always safe):

```java
@Patch.OnEnter
public static void enter(List<String> toLoad, @Patch.Local("t0") long t0) {
    ...
    Loader.maybeReorderMods((ArrayList<String>) toLoad);
    Loader.loadMods((ArrayList<String>) toLoad);
    ...
}
```

See `java/src/main/java/me/zed_0xff/zombie_buddy/patches/Patch_ZomboidFileSystem.java`.

## Prebuilt jar (recommended)

The release asset **`ZombieBuddy-2.3.2-patched.jar`** is the official ZB v2.3.2
(Workshop id 3619862853) jar with only the two `Patch_loadMods2` advice classes
replaced by the fixed versions.

Install:

1. Backup your original: `D:\steam\steamapps\common\ProjectZomboid\ZombieBuddy.jar` → `ZombieBuddy.jar.bak`.
2. Replace it with `ZombieBuddy-2.3.2-patched.jar` (rename to `ZombieBuddy.jar`).

Caveats:

- The original jar was JAR-signed; modifying any signed class breaks the digest, so
  this build is **unsigned** (manifest entries still say version 2.3.2).
- Because the version stays 2.3.2, ZombieBuddy's self-updater will not touch it
  (it only installs newer signed candidates). Remove the old `ZombieBuddy.jar.new`
  if present.
- Built against PZ **42.21.0**. It targets only the `loadMods` advice parameter
  type; everything else works as upstream.

Verify it works after launch (console shows):

```
[ZB] ZomboidFileSystem.loadMods(3 mods) ...
[ZB] Loader.loadMods() took ... ms
[ZB] Exposing class to Lua: Main as PZ3D
```