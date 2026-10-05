Buddies
=======

A remake of the classic PlayStation game, Team Buddies. Development originally began in Java on the Spout voxel game engine, then moved to C++ with custom engines.

This repository is the **Java/Spout prototype**. It is a Spout plugin, not a standalone game.

> **Status: archived upstream (2019), unmaintained.** This fork (`DarkTama/Buddies`) only keeps the prototype buildable. It is alpha-quality 2013 code and has only been verified to build and load on a headless Spout server. Gameplay and the client have not been verified.

## Dependency state

Everything this plugin needs was hosted on infrastructure that no longer exists, so the stock `pom.xml` could not build as originally written.

| Dependency | Original source | Today |
|---|---|---|
| `org.spout:spout-api` (Spout engine) | `nexus.spout.org` | Host is dead. Source survives at [spoutdev/Spout](https://github.com/spoutdev/Spout), which has not seen real development since Dec 2013 and has no tags or releases. Must be built locally. |
| Spout's own libraries: `cereal`, `react`, `simplenbt`, `jlibnoise` | `nexus.spout.org` | Dead host. Sources survive on GitHub (see below); must be built locally. |
| `org.spout:shared-parent` (parent POM) | `nexus.spout.org` | Dead host. Source on GitHub, must be built locally. |
| `org.fourthline.cling:cling-support:2.0-alpha2` | `4thline.org/m2` | Still online, but **HTTP only**. Maven 3.8+ blocks plain-HTTP repos, so a mirror override is required. |
| `commons-math3` 3.2 | Maven Central | Fine. |

### Spout version matters

Spout's API changed quickly in 2013, and this plugin matches one narrow window of it. It needs **both** of these, and no Spout commit satisfies them outside this window:

- the old `org.spout.api.math` package (removed on 2013-07-30 in `790d2afd8`, later replaced by the separate `spout-math` library), and
- `NetworkComponent.setObserver(...)` and `getNetwork().setSyncDistance(...)` (added around 2013-07-29).

Use Spout **`3755babf5`** (2013-08-31). It is the newest `master` commit with both. Newer commits fail with hundreds of missing-symbol errors, older ones lack the networking methods.

The one remaining API drift is already fixed in this repo: `BuddiesBiomeGenerator.generate` now uses Spout's `generate(buffer, world)` signature and reads the chunk Y from `blockData.getBaseChunkY()`.

### Dependency source pins

All are under `https://github.com/spoutdev/`. Check out the commit nearest 2013 so the coordinates match what Spout expects.

| Repo | Commit | Installed as |
|---|---|---|
| `shared-parent` | `c021267` (2013-07-25) | `org.spout:shared-parent:1` |
| `Cerealization` | `daa5c8b` (2013-09-15) | `org.spout:cereal:1.0.0-SNAPSHOT` |
| `React` | `49bb83e` (2013-12-03) | `org.spout:react:1.0.0-SNAPSHOT` |
| `SimpleNBT` | `a6fd207` (2013-09-15) | `org.spout:simplenbt:1.0.5-SNAPSHOT` |
| `JLibnoise` | `19370c7` (2013-09-15) | `net.royawesome:jlibnoise:dev-SNAPSHOT` (re-installed under this name, see below) |

## Building

### Requirements

- **JDK 8** (tested with Temurin 8). Newer JDKs will not work: Spout and its libraries are Java 6/7 era.
- **Maven 3.9.x.** Not on winget. Download the zip from `archive.apache.org/dist/maven/maven-3/` and add `bin` to `PATH`.
- Git.

Run all Maven commands with `JAVA_HOME` pointing at JDK 8. PowerShell:

```powershell
$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-8.0.504.1-hotspot'
$env:Path = "$env:JAVA_HOME\bin;$env:Path"
```

Quote `-D` flags in PowerShell (`'-DskipTests'`), or it splits them at the dots.

### 1. Allow the 4thline HTTP repo

Create `~/.m2/settings.xml`:

```xml
<settings>
  <mirrors>
    <mirror>
      <id>4thline-http</id>
      <mirrorOf>4thline-cling</mirrorOf>
      <url>http://4thline.org/m2</url>
    </mirror>
  </mirrors>
</settings>
```

### 2. Build the Spout libraries

For each of `shared-parent`, `Cerealization`, `React`, `SimpleNBT`, `JLibnoise`: clone, check out the pinned commit, then run:

```powershell
mvn -B install '-DskipTests' '-Dgpg.skip=true' '-Dmaven.javadoc.skip=true' '-Dlicense.skip=true'
```

Then re-install JLibnoise under the coordinates the pinned Spout expects (`net.royawesome`, not `org.spout`):

```powershell
mvn install:install-file '-Dfile=target\jlibnoise-1.0.0-SNAPSHOT.jar' '-DgroupId=net.royawesome' '-DartifactId=jlibnoise' '-Dversion=dev-SNAPSHOT' '-Dpackaging=jar' '-DgeneratePom=true'
```

(Adjust the jar name if the built file differs.)

### 3. Build Spout

Clone `spoutdev/Spout` and check out `3755babf5`. Its old `maven-shade-plugin` pins (2.0 / 2.1) crash under Maven 3.9 with `NoClassDefFoundError: org/sonatype/aether/...`. Bump both to 3.2.4:

- `api/pom.xml`: `maven-shade-plugin` version `2.0` to `3.2.4`
- `engine/pom.xml`: `maven-shade-plugin` version `2.1` to `3.2.4`

Then:

```powershell
mvn -B install '-DskipTests' '-Dgpg.skip=true' '-Dmaven.javadoc.skip=true' '-Dlicense.skip=true' '-Daether.enhancedLocalRepository.trackingFilename=x'
```

The last flag stops Maven refusing locally cached artifacts (cling and its transitive dependencies) because they were originally fetched through a different repository id. If you still see "present, but unavailable" errors, delete the `_remote.repositories` files under `~/.m2/repository` and retry.

Result: `engine/target/spout-1.0.0-SNAPSHOT.jar` (the runnable engine).

### 4. Build Buddies

```powershell
mvn -B clean package
```

Output: `target/buddies-0.0.1-SNAPSHOT.jar`. (The `dependency-reduced-pom.xml` the shade plugin leaves behind is a build artifact; ignore it.)

## Running

Make a run directory containing the engine jar and a `plugins` folder:

```
run/
  spout.jar          <- copy of engine/target/spout-1.0.0-SNAPSHOT.jar
  plugins/
    buddies-0.0.1-SNAPSHOT.jar
```

From `run/`, with JDK 8:

```powershell
java -Xmx1G -jar spout.jar --platform SERVER   # server (headless)
java -Xmx1G -jar spout.jar --platform CLIENT   # client, connects to localhost:13756
```

A healthy server start logs `Enabling Buddies v0.0.1-SNAPSHOT`, `Buddies v0.0.1-SNAPSHOT enabled`, and `Done Loading, ready for players`.

## Known issues

- The client has not been verified to render. It is 2013 LWJGL 2 / OpenGL code on a modern driver stack.
- The server logs a UPnP error (`ConflictInMappingEntry`, HTTP 500) at startup on some routers. It did not stop the server from loading Buddies.
- Spout is alpha software and unmaintained. Do not expect bug fixes upstream.

## Project layout

- `src/` is the plugin source, under `me.man_cub.buddies`.
- `resources/` holds game assets bundled into the jar.
- `properties.yml` is the Spout plugin descriptor, filtered by Maven.
