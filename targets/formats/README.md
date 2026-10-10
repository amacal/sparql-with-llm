# formats tooling

Shared infrastructure, not an exercise itself — and, like
`targets/local-engine/README.md`, this file deliberately names no concrete
parser, serializer, or library. Per CLAUDE.md's "No hardcoded technology",
which tool parses or serializes the format(s) a given `formats/<dir>/`
exercise covers is chosen fresh, by the plan agent, at session-planning
time, from an actual live look-it-up — never carried over from a prior
exercise out of habit, and never written into this file or baked into the
devcontainer image. What's described here is the *mechanism* every
`formats/<dir>/` exercise follows, whichever concrete tool it ends up using
this time.

This file is infrastructure Claude maintains, the same tier as
`.devcontainer/` — not the learning material itself. Everything inside a
`formats/<dir>/` exercise directory (the dataset file(s), any conversion
script) is written by you, guided Socratically (see CLAUDE.md's "Exercise
files" and "Hard constraints").

## Where the tool runs

No server is needed, but the chosen tool still has to run somewhere other
than the devcontainer image itself. Two options, picked per tool:

- **A venv under `.tmp/`** — when the tool is a Python package. The image
  ships `python3` and `python3-venv`, so `python3 -m venv .tmp/venv` gives
  a working `pip` without touching the image.
- **A disposable container** — when the tool needs a runtime the image
  doesn't have, or ships as an image of its own. Name it `<dir>-tool`.

## Getting the dataset into a container

The devcontainer talks to the host's Docker daemon (Docker-outside-of-Docker),
so a `-v` bind-mount source of `/workspaces/...` doesn't exist on the host —
Docker silently creates an *empty* directory there instead, and the tool then
reports the dataset file as missing. Either mount the host-side path, looked
up fresh each sitting exactly as `targets/local-engine/README.md` describes
(`HOST_WS`), or copy the file in, which also gives a snapshot of exactly the
version being verified:

```bash
docker exec <dir>-tool mkdir -p /work
docker cp formats/<dir>/<dataset-file> <dir>-tool:/work/<dataset-file>
```

Re-copy after every edit you make to the dataset — a copied file does not
follow later changes the way a mount would.

## Round-trip check

Every `formats` exercise ends with the round-trip check from CLAUDE.md's
"Query verification discipline": parse, serialize, reparse, and compare
the two graphs as graphs (isomorphism, if the tool offers a check for it),
stated alongside — never instead of — what changed at the byte level.
Intermediate outputs go under `/work/` in the container or `.tmp/` in the
workspace, never into a tracked path.

## Tear down

```bash
docker rm -f <dir>-tool   # or: rm -rf .tmp/venv
```

Removed at the end of the sitting that created it, per CLAUDE.md's
"Session closing ritual" — the exercise's own files persist; the tool's
environment never needs to.
