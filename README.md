# Multiproject Minecraft Mod Template

A bare Gradle scaffold for developing multiple NeoForge mods (Minecraft
1.21.1) side by side, where each mod is still a fully independent,
standalone-buildable project. There are no mods in here yet — this is the
starting point. See "Adding a mod" below.

## Requirements

- JDK 21
- Git (recommended)

You don't need Gradle installed separately — use the included wrapper
(`gradlew` / `gradlew.bat`) in whichever folder you're building from.

## How this template works

The core idea: every mod you add lives in its own folder as a **fully
self-contained Gradle project** — own `settings.gradle`, `build.gradle`,
`gradle.properties`, and `gradlew` wrapper. Copy any one of those folders out
on its own (a separate repo, a separate zip) and it still builds, compiles
and runs with no knowledge of this root folder or any other mod.

This root folder (`settings.gradle`, `build.gradle`, `gradle.properties`)
doesn't contain any mod code itself. It just:

1. **Includes** each mod folder (`settings.gradle`'s `include(...)`) so you
   can build/run them together and IDEs see the whole thing as one project.
2. Optionally **substitutes** Maven-coordinate dependencies between mods
   for live source (root `build.gradle`) so you're not publishing to
   `mavenLocal()` on every change while co-developing.
3. Optionally defines a **combined run** (`runAllClient`/`runAllServer` in
   root `build.gradle`) that launches every mod in the `modProjects` list
   together in one game instance.

None of that is required for any individual mod to build — it's purely
convenience for when you're working on several of them together.

## Adding a mod

There are three situations you might be in, depending on where the mod
comes from.

### 1. Generating a fresh mod

1. Use the [NeoForge mod generator](https://neoforged.net/) (or clone the
   [MDK for the ModDevGradle plugin](https://github.com/NeoForgeMDKs)) to
   generate a new mod for Minecraft 1.21.1. Pick the same
   Minecraft/NeoForge/Parchment versions as this template's root
   `gradle.properties`, so everything lines up if you build it alongside
   other mods later.
2. Move the generated project into a new folder here, e.g. `mymod/`.
3. Follow "A project that already matches this template's shape" below — a
   freshly generated mod already has that shape (its own `settings.gradle`,
   `build.gradle`, `gradle.properties` and `gradlew`), it just isn't wired
   into this root yet.

   Note: the MDK's own `build.gradle` ships a commented-out
   `tasks.named('wrapper', Wrapper).configure { ... }` block, which some
   versions of the generator uncomment automatically the first time you run
   `./gradlew wrapper` inside the generated project standalone. Normally that
   line would fail once the mod is included here (`Task with name 'wrapper'
   not found`), since Gradle only auto-creates a `wrapper` task on the actual
   root of the build. This template's root `build.gradle` pre-registers a
   `wrapper` task on every included subproject for exactly this reason, so
   you can drop the generated mod in unmodified — no need to comment
   anything out.

### 2. A project that already matches this template's shape

If it already has its own `settings.gradle`, `build.gradle`,
`gradle.properties` and `gradlew` (for example, a mod you previously split
out of a multiproject folder like this one), wiring it in is just:

1. Copy (or `git clone`) its folder into this root folder, e.g. `mymod/`.
2. Add `'mymod'` to the `include(...)` line in the root `settings.gradle`.
3. If you want it in the combined launch, add `"mymod"` to the root
   `build.gradle`'s `modProjects` list.
4. If it depends on another mod here via a Maven coordinate, add a matching
   `substitute(module('group:artifact')).using(project(':othermod'))` line
   to the root `build.gradle` (see "Cross-mod dependencies" below).

Nothing else needs to change — `mymod`'s own `build.gradle`/`gradle.properties`
stay as they are, and it remains independently buildable on its own.

### 3. A NeoForge project from somewhere else (a different template, the vanilla MDK, a solo repo)

This needs a bit more care, since the toolchain and plugin setup have to
line up with everything else in this build:

1. Copy the project's folder in as-is (e.g. `mymod/`).
2. Check its `gradle.properties` against the root `gradle.properties`:
   `minecraft_version`, `neo_version`, `parchment_minecraft_version` and
   `parchment_mappings_version` all need to match — a single Gradle build
   can't mix Minecraft/NeoForge versions across subprojects. Bump whichever
   side is behind.
3. Check which NeoForge Gradle plugin it uses. This template uses
   ModDevGradle (`net.neoforged.moddev` version `2.0.144`). If the incoming
   project uses NeoGradle (`net.neoforged.gradle...`) or a much older
   ModDevGradle version, either upgrade its `build.gradle` to match, or
   leave it building standalone only (don't add it to `include(...)`) and
   consume it as a published jar dependency instead (see "Cross-mod
   dependencies" below).
4. If it doesn't already have its own `settings.gradle` and `gradlew`, add
   them (see the template below) so it stays standalone-buildable like any
   other mod folder here.
5. Make sure its `mod_id` and `mod_group_id` don't collide with any other
   mod you've already added. If you want it covered by the generic
   dependency-substitution rule (see "Cross-mod dependencies" below),
   update its `mod_group_id` to follow the `mod_group_id_base` convention
   from the root `gradle.properties` (`<mod_group_id_base>.<mod_id>`).
6. Add its folder name to the root `settings.gradle`'s `include(...)`, and
   optionally to `modProjects` in the root `build.gradle` for the combined
   launch.
7. If it should depend on another mod here, add
   `implementation 'group:artifact:version'` to its `dependencies { }`
   (plus `mavenLocal()` to its `repositories { }`), and a substitution line
   in the root `build.gradle` if you want live source while co-developing.

If anything's misaligned, `./gradlew :mymod:build` from the root (or
`:mymod:tasks` first) will surface version/plugin conflicts quickly rather
than failing deep into compilation.

A minimal standalone `settings.gradle` for a mod folder that doesn't have
one yet:

```groovy
pluginManagement {
    repositories {
        gradlePluginPortal()
    }
}

plugins {
    id 'org.gradle.toolchains.foojay-resolver-convention' version '0.8.0'
}

rootProject.name = 'mymod'
```

Copy `gradlew`, `gradlew.bat` and `gradle/wrapper/` from this root folder
into `mymod/` too, so it has its own wrapper.

## Cross-mod dependencies

If one mod depends on another (say `feature` depends on `core`), declare it
as a normal Maven coordinate in `feature/build.gradle` — **not** a Gradle
`project(':core')` reference, since that only makes sense when both mods
live inside the same Gradle build, which breaks the "copy this folder out
and it still builds" goal:

```groovy
repositories {
    mavenLocal()
}

dependencies {
    implementation 'com.example.core:core:1.0.0'
}
```

Built standalone, `feature` resolves `core` from `mavenLocal()` (your local
`~/.m2` repository) — so you'd run `./gradlew publishToMavenLocal` inside
`core/` before building `feature/` alone.

While co-developing everything together through *this* root project, you
don't want to re-run `publishToMavenLocal` every time you touch `core`'s
code. Add a dependency substitution rule to the root `build.gradle` for that
dependency:

```groovy
allprojects {
    configurations.configureEach {
        resolutionStrategy.dependencySubstitution {
            substitute(module('com.example.core:core')).using(project(':core'))
        }
    }
}
```

This transparently swaps that Maven coordinate for the live `:core`
subproject when building through this root folder, so `feature` always
compiles and runs against `core`'s current source — no publish step needed.
Standalone (no root `build.gradle` present), this rule doesn't exist, so the
dependency resolves normally from `mavenLocal()`.

**Doing this for every mod automatically.** Writing one substitution line per
cross-mod dependency is fine for a couple of mods, but gets repetitive as the
project grows. If every mod's `mod_group_id` follows the `mod_group_id_base`
convention set in the root `gradle.properties` (group is
`<mod_group_id_base>.<mod_id>`, artifact id is `<mod_id>`), its Maven
coordinate becomes fully predictable from just its folder name — so one
generic rule can substitute every cross-mod dependency at once, with nothing
to add as you bring in more mods:

```groovy
allprojects { proj ->
    proj.configurations.configureEach {
        resolutionStrategy.dependencySubstitution {
            rootProject.subprojects.each { sub ->
                if (sub != proj) {
                    substitute(module("${rootProject.mod_group_id_base}.${sub.name}:${sub.name}"))
                        .using(project(sub.path))
                }
            }
        }
    }
}
```

Both versions (the explicit one-liner and this generic loop) are included,
commented out, in the root `build.gradle` — use whichever fits: the explicit
version is easier to read at a glance ("these two mods are linked"), the
generic version needs no upkeep as the project grows but relies on every mod
actually following the group-id convention.

The depended-on mod (`core` in this example) doesn't need any special wiring
to load alongside `feature` in the dev environment: once it's a real
dependency (project-substituted or a published jar, either way), NeoForge
finds its `neoforge.mods.toml` on the runtime classpath and loads it
automatically, the same way it would pick up any other real mod dependency
(e.g. JEI).

## Combined "launch everything" run

The root `build.gradle` defines `runAllClient` and `runAllServer` tasks that
launch every mod in one list together in a single game instance — useful
once you have more than one mod and want to test them interacting, not just
each one on its own. It's driven by a single list near the bottom of the
file:

```groovy
def modProjects = []
```

Add a mod's folder name to that list (it must match a subproject name
declared in `settings.gradle`) and run:

```
./gradlew runAllClient
```

Both the `mods { }` registration and `loadedMods` are generated from that
one list, so there's nothing else to keep in sync. It's empty by default —
nothing to launch until you've added at least one mod.

## Project structure

```
multiproject/
├── build.gradle       # dependency substitution examples + the combined "runAllClient" run
├── settings.gradle     # include(...) list of mod folders (empty for now)
├── gradle.properties   # Minecraft/NeoForge/Parchment versions - copy into new mods' own gradle.properties
├── gradlew / gradlew.bat / gradle/wrapper/
└── (add your mod folders here, e.g. core/, mymod/ - see "Adding a mod" above)
```

## Notes

- **Version bumps**: NeoForge/Minecraft/Parchment versions should stay in
  sync across the root `gradle.properties` and every mod folder's own
  `gradle.properties`, since each mod stands alone. Versions pinned here
  (NeoForge `21.1.248`, Parchment `2024.11.17`) were current as of this
  template being generated — check
  https://projects.neoforged.net/neoforged/neoforge and
  https://parchmentmc.org/docs/getting-started for newer releases.
