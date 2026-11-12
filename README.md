# verify-plugin-persistence

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A Claude Code skill that proves a Dans-Plugins Minecraft plugin actually persists its data. It writes entities on a live Dockerized test server, restarts the server, and checks that the plugin reads them back. Where the plugin has more than one storage backend, it also checks a migration between them. It does not rely on unit tests for any of this.

The whole skill is one file: [`verify-plugin-persistence.md`](verify-plugin-persistence.md).

## Why

A green test suite is where storage bugs hide. The skill was written after a JSON storage backend passed 443 unit tests and could not save a faction at all. The tests had mocked away exactly the parts that failed on a real server. The ["Why this exists"](verify-plugin-persistence.md#why-this-exists) section of the skill has the full story.

## When to use it

- A PR adds or changes a storage backend, a repository, a schema, or a serialization path.
- A plugin gains a data migration command.
- A change writes player-visible state and needs evidence before merge.

It does not replace unit tests. It catches what they structurally cannot: framework objects leaking into serialized data, broken shaded jars, resources that fail to load from inside the jar, and anything else that only shows up on a real server.

## What it does

The skill walks through ten steps:

1. List every entity that must persist, and where each one is stored.
2. Round-trip each repository in a throwaway JVM probe test before touching Docker.
3. Check that the shaded jar really contains the resources and libraries the change needs.
4. Build and start the test server.
5. Open an RCON console channel.
6. Drive a write from the console.
7. Run the four-part persistence proof: write, content, reload, and dereference.
8. Run a migration round trip, if the plugin has more than one backend.
9. Turn each defect found into a regression test, and prove the test fails without the fix.
10. Clean up and report what was covered and what was not.

A **Failure modes catalogue** maps the symptoms you will see to their causes. A **Self-audit** section lists every plugin-specific assumption the steps rely on, so you can check the skill against the plugin when it drifts.

Medieval Factions is the reference plugin. Details that apply only to it are marked "For Medieval Factions:" in the skill.

## Requirements

The skill's [Prerequisites](verify-plugin-persistence.md#prerequisites) section lists them in full:

- A plugin repo with the standard Dans-Plugins test-server infrastructure: `compose.yml`, `Dockerfile`, `sample.env`, `up.sh` / `down.sh`
- A reachable Docker daemon
- A git worktree for the branch under test

## Usage

Install `verify-plugin-persistence.md` as a Claude Code skill. Then invoke `/verify-plugin-persistence` from inside the plugin repo you want to verify.

## License

MIT. See [LICENSE](LICENSE).
