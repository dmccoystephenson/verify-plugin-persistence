# verify-plugin-persistence

Proves that a Dans-Plugins Minecraft plugin actually persists its data — that entities survive a write, a full server restart, and (where the plugin has more than one storage backend) a migration between them — by exercising the real code on a live Dockerized test server rather than trusting unit tests.

**Identity:** the kind of skill that refuses to call storage "working" on the strength of a green test suite, because the test suite is exactly where storage bugs hide.

---

## When to use this

- A PR adds or changes a storage backend, a repository, a schema, or a serialization path.
- A plugin gains a data migration command.
- You are about to merge something that writes player-visible state and you want evidence, not inference.

Do **not** use it as a substitute for unit tests. Use it as the thing that catches what unit tests structurally cannot: framework-object leakage, shaded-jar breakage, resource loading from inside the jar, and anything that only manifests on a real server.

---

## Why this exists

The failure that motivated this skill: a JSON storage backend passed 443 unit tests, a clean lint, and four rounds of automated review — and could not save a faction at all. Gson was handed whole domain objects that each held a `MedievalFactions` reference, so its reflective serializer walked into the plugin, down into the JDK internals behind `JavaPlugin`, and threw. The data file was never created.

The tests missed it because they mocked the plugin, and the mock's `getResource(...)` returned `null`, which made schema loading throw, which silently disabled validation. The only repositories under test were the three whose types happened to carry no plugin reference.

Every step below exists because some variant of that went undetected.

---

## Prerequisites

- The plugin repo has the standard Dans-Plugins test-server infrastructure: `compose.yml`, `Dockerfile`, `sample.env`, `up.sh` / `down.sh`.
- Docker is reachable (`docker version`). On WSL this needs Docker Desktop running with WSL integration enabled for the distro; if `docker` is missing but `/mnt/c/Program Files/Docker/Docker/resources/bin/docker.exe` exists, the daemon is simply not started — stop and tell the user rather than trying to work around it.
- A git worktree for the branch under test. Never verify on a dirty shared checkout.

---

## Steps

### 1 — Enumerate what must persist

Before running anything, list the entities the change touches and where each is stored. Read the repository interfaces, not the docs.

```bash
ls src/main/kotlin/**/[Ss]torage/**/*.kt src/main/java/**/repository/*.java 2>/dev/null
grep -rn "interface Mf.*Repository" src/main --include=*.kt --include=*.java
```

Write down, explicitly:

- Every entity type and its backing file/table.
- Which backends exist (`database`, `json`, flatfile, …) and the config key that selects them.
- Which entity is the **root** — the one everything else references. For Medieval Factions that is the faction. Verifying leaf entities while the root is broken proves nothing, and leaf entities will often appear to work.
- **Which types carry something a reflective serializer cannot handle** — a framework reference, `Instant`, `UUID`, a value class inside a generic collection, a sealed hierarchy. Grep the domain classes for those field types; each one is a candidate for exactly the bugs this skill exists to catch.

Then cover **every** repository, not a sample. Two separate write-only bugs in Medieval Factions PR #1944 were each found only by testing the specific repository that had none — the untested ones are untested precisely because nobody looked at them.

### 2 — Cheap proof first: exercise the real repositories in a JVM test

Do this **before** touching Docker. It is minutes instead of half an hour, and it localizes the bug.

Write a temporary probe test that constructs each domain object **fully populated** and round-trips it through the real repository. Two rules make or break this probe:

1. **Serve real resources.** If the plugin loads schemas or config from the jar, stub `getResource` to return the actual file — otherwise you are testing with validation switched off:

   ```kotlin
   `when`(plugin.getResource("schemas/factions.json"))
       .thenAnswer { java.io.File("src/main/resources/schemas/factions.json").inputStream() }
   ```

2. **Catch `Throwable`, don't assert.** The probe's job is to report what each repository does, not to stop at the first failure:

   ```kotlin
   private fun probe(name: String, block: () -> Unit) {
       try { block(); println("PROBE[$name] OK") }
       catch (e: Throwable) { println("PROBE[$name] THREW ${e::class.java.name}: ${e.message?.take(200)}") }
   }
   ```

Run one probe per repository, then list the files actually written:

```bash
./gradlew test --tests "*ProbeTest*" --console=plain
python3 -c "
import re,html
x=open('build/test-results/test/TEST-<pkg>.ProbeTest.xml',encoding='utf-8',errors='replace').read()
for m in re.findall(r'<system-out>(.*?)</system-out>', x, re.S): print(html.unescape(m))"
```

A repository that throws here is broken; a repository whose file never appears on disk is broken even if nothing threw. **Delete the probe before committing** — it is a diagnostic, not a test. Convert what it found into a real regression test instead (see Step 9).

### 3 — Confirm the shaded jar, not just the classpath

Unit tests run against the unshaded classpath. Production runs against the shadow jar with relocated packages and a hand-maintained `include(dependency(...))` list — a dependency that is on the test classpath may simply not be in the jar.

```bash
./gradlew shadowJar --console=plain
JAR=$(ls build/libs/*-all.jar)
unzip -l "$JAR" | grep -E "schemas/|<your-lib>"          # resources and libs actually shipped
```

If the change loads resources or uses a library reflectively, write a throwaway class that does exactly that against `-cp "$JAR"` alone. Relocated names apply:

```java
import com.dansplugins.factionsystem.shadow.org.everit.json.schema.loader.SchemaLoader;
```

This is the only way to catch "works in tests, `NoClassDefFoundError` on the server".

### 4 — Build and start the test server

```bash
cp sample.env .env
# Trim optional plugins to cut build time and log noise:
sed -i 's/^CURRENCIES_ENABLED=.*/CURRENCIES_ENABLED=false/; s/^DYNMAP_ENABLED=.*/DYNMAP_ENABLED=false/; s/^PLACEHOLDER_API_ENABLED=.*/PLACEHOLDER_API_ENABLED=false/' .env
docker compose build      # 20-30 min: installs two JDKs, builds Ponder, runs BuildTools for Spigot
docker compose up -d
```

Run the build with `run_in_background: true`. If a stale container holds the name, `docker rm <name>` first — check it is `Exited` before removing.

Wait for readiness, and **match on a marker that cannot be satisfied by a previous startup**. Counting occurrences is the reliable form:

```bash
until [ "$(docker logs <container> 2>&1 | grep -c 'Done (')" -ge 1 ]; do sleep 10; done
```

Start on the plugin's **default** backend first and confirm it still works. That is your regression baseline — a refactor of the storage-selection code can break the path nobody is looking at.

**The container builds its own jar, and the entrypoint overwrites yours.** `post-create.sh` copies the image's jar into `plugins/` on every start, so a jar you `docker cp` in — or drop into the bind-mounted `plugins/` directory — is silently replaced, and you end up verifying the code as of the last image build. After changing plugin code you must `docker compose build` again. The Spigot/BuildTools layers are cached, so a rebuild is minutes rather than the original half hour. Confirm which code you are actually running by checking for a log line or behaviour unique to your change; do not assume.

**Note also that `docker compose up -d` recreates the container when the compose config changed**, which resets the log. A cumulative wait like `grep -c 'Done (' -ge 5` will then never be satisfied. Re-count from zero after any recreate, or scope the wait with `docker logs --since`.

### 5 — Open a console channel

`compose.yml` sets neither `stdin_open` nor `tty`, so there is no console to type into. Enable RCON.

```bash
docker compose down
sed -i 's/enable-rcon=false/enable-rcon=true/; s/^rcon.password=$/rcon.password=<password>/' testmcserver/server.properties
cat > compose.override.yml <<'EOF'
services:
  testmcserver:
    ports:
      - "25575:25575"
EOF
```

`compose.override.yml` is picked up automatically and must be **deleted during cleanup** — never commit it.

Use a minimal RCON client (packet: `<int32 len><int32 id><int32 type><body>\0\0`; type 3 = auth, 2 = command):

```python
import socket, struct
def pack(i, t, b): 
    p = struct.pack("<ii", i, t) + b.encode() + b"\x00\x00"
    return struct.pack("<i", len(p)) + p
```

Write it to the scratchpad, not the repo.

### 6 — Drive a write from the console

Most gameplay commands require a `Player` sender. Find one that does not — read the command class and check whether it uses `sender` generically rather than casting to `Player`.

For Medieval Factions: `/f admin create <name>` works from console, needs `mf.admin.create` (console has every permission), and requires `factions.allowLeaderlessFactions: true` in the plugin config. It calls `factionService.save(...)`, which is the full repository write path.

Configure the plugin for the backend under test and restart:

```bash
sed -i "s/^  type: database/  type: json/" testmcserver/plugins/<Plugin>/config.yml
```

**Async commands return before they finish.** RCON will show the "starting…" message and nothing else. Never treat empty or optimistic command output as success — verify against the log and the filesystem.

### 7 — The four-part persistence proof

All four parts are required. Each catches something the others miss.

| Part | Check | Catches |
|---|---|---|
| **a. Write** | The file or row now exists and is non-trivially sized | Serialization throwing; silently swallowed failures |
| **b. Content** | Every expected field is present, and **no framework object leaked in** | Domain objects dragging plugin/server refs into the output |
| **c. Reload** | Restart, then confirm the startup log reports the right count | Deserialization failures; write-only formats |
| **d. Dereference** | A startup task that reads a *derived* property runs clean | Framework refs deserialized as `null` — the file looks perfect and every read NPEs |

Part (c) is worth doing per entity type, not just for the root. A repository that swallows read failures and reports "file is empty" will pass (a) and (b) while being completely unreadable — and will then overwrite the stored data on the next write. If a `catch` in the load path returns an empty collection, treat that entity type as unverified until a read has actually returned something.

For (b), assert the negative explicitly:

```bash
python3 - <<'EOF'
import json
d = json.load(open('testmcserver/<data-dir>/<entity>.json'))
e = d['<entities>'][0]
print("keys:", sorted(e.keys()))
print("LEAKED FRAMEWORK REF:", '"plugin"' in open('testmcserver/<data-dir>/<entity>.json').read())
EOF
```

Part (d) is the one people skip. A reflective deserializer will happily produce an object whose framework field is `null`; the data file is flawless and the plugin NPEs the moment anything reads a derived property. Pick a startup task that touches one — for Medieval Factions, "Disabling neutrality for existing factions" reads `faction.flags[plugin.flags.neutral]` and therefore proves the plugin reference was re-attached. Then confirm the log is clean:

```bash
docker logs --since 5m <container> 2>&1 | grep -icE "exception|severe"   # expect 0
```

To read one specific startup out of an accumulating log:

```bash
docker logs <container> 2>&1 | awk '/Enabling <Plugin>/{n++} n==2'
```

### 8 — Migration round trip (only if the plugin has more than one backend)

A migrator that reads A and writes B exercises both paths at once. Run the full circuit, restarting into each backend so the data is *read back by the plugin* rather than merely written:

1. Backend A → create data → verify (Step 7).
2. `/<cmd> migrate toB` → expect a non-zero migrated count for the root entity.
3. Switch config to B, restart → the count must match.
4. **Delete backend B's data**, then `migrate toA` → confirm it is recreated.
5. Switch to A, restart → count matches and content is intact.

Step 4 matters: without clearing first, an unchanged file proves nothing, because the migrator may have written nothing at all.

### 9 — Turn the finding into a permanent test

A live verification that is not encoded in a test will be re-broken. For every defect this skill finds, add a regression test **and prove it has teeth** by reverting the fix and watching it fail:

```bash
# neutralize the fix, run the new test, expect FAILED, then restore
git stash && ./gradlew test --tests "*NewRegressionTest*"; git stash pop
```

A test that passes both with and without the fix is not a regression test.

### 10 — Clean up and report

```bash
docker compose down
rm -f compose.override.yml
git status --short          # must be clean of test scaffolding
```

Report as a table of steps with concrete evidence (byte counts, loaded counts, migration counts, exception counts). State plainly what was **not** covered — console-driven verification cannot exercise claiming, gates, locks, chat, or anything needing a player in the world. Do not let a green table imply coverage you did not achieve.

---

## Failure modes catalogue

Consult when something behaves oddly.

| Symptom | Cause |
|---|---|
| `JsonIOException: Failed making field 'java.util.ResourceBundle#parent' accessible` | A domain object holds a plugin/server reference and was handed to a reflective serializer. Persist via DTOs instead. |
| Tests pass but validation never runs | `getResource` was mocked to `null`, so schema loading threw and validation was silently disabled. |
| Data file written but every read NPEs | Framework field deserialized as `null`; it must be re-attached during mapping. |
| Entity type is write-only: file looks correct, reads return nothing | A read-side deserializer is missing an adapter the write side has (e.g. `Instant`), and the repository's `catch` reports the file as empty. The next write then persists only the new entity and drops the rest. |
| `NoClassDefFoundError` on the server only | Transitive dependency missing from the shadow jar's `include(dependency(...))` list. |
| Wait loop returns instantly | The readiness marker matched a *previous* startup. Count occurrences instead of grepping. |
| Wait loop never returns | The container was recreated and the log reset, so a cumulative count can no longer be reached. |
| Fix appears to have no effect on the server | The entrypoint copied the image's jar over yours. Rebuild the image; do not hot-copy. |
| RCON command produces no output | The command is async; check the server log, not the response. |
| `Conflict. The container name "…" is already in use` | Stale exited container. `docker rm <name>`. |
| Gradle: `Unexpected lock protocol found in lock file` | Corrupt cache. `./gradlew --stop && rm -rf ~/.gradle/caches/<version>/javaCompile`. |
| `docker system df` errors with "too many levels of symbolic links" | Docker Desktop/WSL overlay quirk; harmless, does not indicate a failing build. |

---

## Self-audit

Run this section when the skill may have drifted from reality — e.g. after the test-server infrastructure changed, commands started failing, or results are consistently wrong.

1. Read this skill file from top to bottom.
2. For each command, path, or assumption, verify it is still correct:
   - `compose.yml`, `Dockerfile`, `sample.env`, `up.sh` still exist and carry the documented variables
   - The console-drivable command named in Step 6 still exists and still accepts a non-player sender
   - The startup-task marker named in Step 7d still appears in the log and still reads a derived property
   - Config keys (`storage.type`, `factions.allowLeaderlessFactions`) still match the plugin's config
   - The shadow-jar relocation prefixes in Step 3 still match `build.gradle`
3. For each problem found, open a GitHub issue:
   ```bash
   gh issue create --repo dmccoystephenson/verify-plugin-persistence \
     --title "<problem summary>" \
     --body "$(cat <<'EOF'
   **Section:** <which step or section is wrong>
   **Problem:** <what is incorrect>
   **Expected behavior:** <what it should do instead>
   EOF
   )"
   ```
4. Report a summary: how many issues were filed, or confirm the skill is up to date.
