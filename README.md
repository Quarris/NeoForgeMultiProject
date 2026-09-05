# Multiproject Minecraft Mod Template

Bare Gradle scaffold for developing multiple NeoForge mods (Minecraft
1.21.1) side by side, each one a fully independent, standalone-buildable
project. No mods included yet - see "Adding a mod" below.

## Requirements

- JDK 21
- Git (recommended)

Use the included wrapper (`gradlew`/`gradlew.bat`) - no separate Gradle install needed.

## How this template works

Each mod lives in its own folder as a fully self-contained Gradle project
(own settings.gradle, build.gradle, gradle.properties, gradlew). Copy any
mod folder out and it still builds with no knowledge of this root or any
other mod.

This root folder holds no mod code. It only:

1. `include(...)`s each mod folder in settings.gradle, so they build/run together and IDEs see one project.
2. Optionally substitutes Maven-coordinate dependencies between mods for live source (build.gradle), skipping `mavenLocal()` publishing while co-developing.
3. Optionally defines a combined run (`runAllClient`/`runAllServer`) launching every mod in `modProjects` together.

None of this is required for any individual mod to build.

## Adding a mod

### Option A: generate a fresh mod and drop it in

1. Generate a mod for MC 1.21.1 via the [NeoForge generator](https://neoforged.net/)
   (or the [ModDevGradle MDK](https://github.com/NeoForgeMDKs)), matching this
   template's Minecraft/NeoForge/Parchment versions.
2. Move it into a new folder here, e.g. `mymod/`.
3. Continue from step 4 under "Bringing in an existing project" below.

### Bringing in an existing project

#### Already matches this template's shape

Has its own settings.gradle/build.gradle/gradle.properties/gradlew already:

1. Copy (or clone) its folder in, e.g. `mymod/`.
2. Add `'mymod'` to `include(...)` in root settings.gradle.
3. Add `"mymod"` to `modProjects` in root build.gradle if you want it in the combined launch.
4. If it depends on another mod here, add a substitution line (see "Cross-mod dependencies").

Nothing else changes - it stays independently buildable.

#### A NeoForge project from somewhere else (different template, vanilla MDK, solo repo)

Needs the toolchain/plugin setup aligned first:

1. Copy the folder in as-is.
2. Match `minecraft_version`, `neo_version`, `parchment_minecraft_version`,
   `parchment_mappings_version` to the root gradle.properties - a build can't
   mix MC/NeoForge versions across subprojects.
3. Check its NeoForge plugin - this template uses ModDevGradle
   (`net.neoforged.moddev` 2.0.144). NeoGradle or an old ModDevGradle version:
   upgrade it, or keep it standalone-only and consume it as a published jar instead.
4. Add its own `settings.gradle`/`gradlew` if missing (template below).
5. Make sure `mod_id`/`mod_group_id` don't collide with existing mods; follow
   the `mod_group_id_base` convention if you want the generic substitution rule to pick it up.
6. Add it to `include(...)`, and to `modProjects` if wanted in the combined launch.
7. For a dependency on another mod here: `implementation` coordinate + `mavenLocal()`
   in its build.gradle, plus a substitution line at root.

`./gradlew :mymod:build` surfaces version/plugin mismatches quickly.

A minimal standalone `settings.gradle` for a mod folder that doesn't have one yet:

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

Copy `gradlew`, `gradlew.bat`, `gradle/wrapper/` from root into `mymod/` too.

## Cross-mod dependencies

Declare it as a normal Maven coordinate, not `project(':core')` (breaks standalone builds):

```groovy
repositories {
    mavenLocal()
}

dependencies {
    implementation 'com.example.core:core:1.0.0'
}
```

Standalone, this resolves from `mavenLocal()` - run `publishToMavenLocal` in
`core/` first. Through this root, substitute it for the live project instead:

```groovy
allprojects {
    configurations.configureEach {
        resolutionStrategy.dependencySubstitution {
            substitute(module('com.example.core:core')).using(project(':core'))
        }
    }
}
```

For more than a couple of mods, use the generic version instead (also in root
build.gradle, commented out) - substitutes every cross-mod dependency automatically
as long as each mod's `mod_group_id` follows the `<mod_group_id_base>.<mod_id>` convention:

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

`core` needs no special wiring to load alongside `feature` - NeoForge picks up
its `neoforge.mods.toml` from the classpath like any other mod dependency.

## Combined "launch everything" run

`runAllClient`/`runAllServer` launch every mod in `modProjects` (root build.gradle)
together in one game instance:

```groovy
def modProjects = []
```

Add mod folder names (must match `settings.gradle`'s `include(...)`), then:

```
./gradlew runAllClient
```

Both `mods { }` and `loadedMods` are generated from that one list.

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

- **Version bumps**: keep NeoForge/Minecraft/Parchment versions in sync across
  root and every mod's own gradle.properties. Current pins: NeoForge `21.1.248`,
  Parchment `2024.11.17` - check for newer releases at
  https://projects.neoforged.net/neoforged/neoforge and
  https://parchmentmc.org/docs/getting-started.
