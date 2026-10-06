# local-engine containers

Shared infrastructure, not an exercise itself — but unlike a sibling repo's
own shared managed-node images, this file deliberately names no concrete
image. Per CLAUDE.md's "No hardcoded technology", which SPARQL engine backs
any given exercise is chosen fresh, by the plan agent, at session-planning
time, from an actual live look-it-up (WebFetch/WebSearch) for what's
currently mainstream, actively maintained, and freely available — never a
name carried over from a prior exercise out of habit, and never written
into this file. What's described here is the *mechanism* every
`local-engine/<dir>/` exercise follows, whichever concrete image it ends up
using this time.

This file is infrastructure Claude maintains, the same tier as
`.devcontainer/` — not the learning material itself. Everything inside a
`local-engine/<dir>/` exercise directory (the query file(s), any dataset
authored for the exercise) is written by you, guided Socratically — fresh,
or carried over from one of your own earlier exercises (see CLAUDE.md's
"Exercise files"). Claude never writes or completes any of it (see
CLAUDE.md's "Hard constraints").

## Spin up a node

The devcontainer talks to the host's Docker daemon (Docker-outside-of-Docker),
so a `-v` bind-mount source must be the *host*-side path of this workspace,
not `/workspaces/...` (which doesn't exist on the host — Docker would
silently create an empty directory there instead). Look it up fresh each
sitting, from inside the devcontainer:

```bash
HOST_WS=$(docker inspect "$(hostname)" \
  --format '{{range .Mounts}}{{if eq .Destination "'"$PWD"'"}}{{.Source}}{{end}}{{end}}')
```

The general shape, once the plan agent has named a concrete engine image and
checked its own docs for its actual load mechanism and default port:

```bash
docker pull <chosen-engine-image>

# most engines expose either a bulk-load step at container startup (reading
# a mounted dataset file or directory) or an HTTP load endpoint once running
# — confirm which from that engine's own docs before assuming either
docker run -d --name <dir>-engine \
  -v "$HOST_WS/local-engine/<dir>/<dataset-file>:/<engine-expected-path>:ro" \
  -p 0:<engine-sparql-port> <chosen-engine-image>

# the address to query (SPARQL 1.1 Protocol, usually an HTTP GET/POST):
docker inspect -f '{{.NetworkSettings.IPAddress}}' <dir>-engine
docker port <dir>-engine
```

Confirm the container is actually ready to accept queries (a log line, a
health endpoint, a simple `ASK` query that doesn't depend on the exercise's
own data) before trusting any result from it — a container that's still
loading a dataset can otherwise return a misleadingly empty result that
looks like a real answer.

## Query it

The SPARQL 1.1 Protocol is itself part of what this repo teaches (see
CLAUDE.md's "Scope") — a plain HTTP GET with the query URL-encoded in a
`query` parameter, or a POST with the appropriate content type, against
whatever path the chosen engine exposes its SPARQL endpoint at (confirm
from its docs rather than assuming a path). `curl` against that endpoint,
requesting whichever result format the exercise is using via the `Accept`
header, is the generic mechanism — no engine-specific client library needed
for this part.

## Tear down

```bash
docker rm -f <dir>-engine
```

Every container is ephemeral: destroyed at the end of the sitting that
created it, regardless of whether the exercise concluded (the exercise's
*files* — the query, the dataset — persist; the running container never
needs to). A node that has already answered a few queries doesn't look any
different from a fresh one for most exercises here (querying doesn't
mutate a dataset the way Ansible's package installs mutate a managed node),
but an exercise that uses SPARQL Update should still recreate the
container before claiming a from-scratch result, for the same reason any
other repo in this family insists on a clean starting state before
verifying a claim.

The bridge IP and exact port are ephemeral (valid only for that container's
lifetime) — write them into that exercise's own notes file if they matter
to the worked example, never assume they'll be the same next time.
