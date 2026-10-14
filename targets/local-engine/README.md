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
checked its own docs for its actual load mechanism and default port. Every
name below carries an engine number `<n>` — `1` for the exercise's engine,
`2` and up for any further engine brought up in the same sitting (a
cross-engine check of the same query, see CLAUDE.md's "Cross-engine and
cross-format comparison") — so two engines never collide and teardown can
remove all of them by pattern:

Not every engine publishes a prebuilt image. When the chosen one doesn't,
build it locally from the engine's own official build kit (its published
Dockerfile or equivalent artifact, fetched fresh — never a third-party
image), unpacked under `.tmp/`, and tag it after the exercise directory
(`<dir>-engine-<n>-image`) so teardown knows exactly what to remove:

```bash
docker build -t <dir>-engine-<n>-image .tmp/<engine-build-kit>/
```

Otherwise, pull the published image:

```bash
docker pull <chosen-engine-image>

# most engines expose either a bulk-load step at container startup (reading
# a mounted dataset file or directory) or an HTTP load endpoint once running
# — confirm which from that engine's own docs before assuming either;
# --init runs a tiny init as PID 1 that forwards signals and reaps child
# processes, so the engine's server process can't linger as a zombie that
# makes `docker rm -f` fail at teardown
docker run -d --init --name <dir>-engine-<n> \
  -v "$HOST_WS/local-engine/<dir>/<dataset-file>:/<engine-expected-path>:ro" \
  -p 0:<engine-sparql-port> <chosen-engine-image>

# the address to query (SPARQL 1.1 Protocol, usually an HTTP GET/POST):
docker inspect -f '{{.NetworkSettings.IPAddress}}' <dir>-engine-<n>
docker port <dir>-engine-<n>
```

A bind mount isn't the only way in, and isn't always the simplest one: an
engine with an HTTP load endpoint can take the dataset over the wire once
it's running, and an engine whose own CLI subcommand (a bulk load, a
query-plan/explain mode) needs files inside the container can get them via
a named volume instead. Many engine images are minimal and ship no shell,
so `docker exec ... sh` is not available to stage files — create the
volume, attach it to a short-lived `docker create`d container, `docker cp`
the dataset and query files into it, remove that container, then run each
engine subcommand as its own `docker run --rm -v <dir>-engine-<n>-vol:/<path>`
against the same volume:

```bash
docker volume create <dir>-engine-<n>-vol
docker create --name <dir>-engine-<n>-cp -v <dir>-engine-<n>-vol:/in <chosen-engine-image> <any-no-op-arg>
docker cp local-engine/<dir>/<dataset-file> <dir>-engine-<n>-cp:/in/
docker rm <dir>-engine-<n>-cp
docker run --rm -v <dir>-engine-<n>-vol:/in <chosen-engine-image> <engine-subcommand> ...
```

A persistent store inside such a volume keeps whatever was loaded into it,
so load the dataset into it exactly once — a second load of the same file
is not a no-op when the file contains blank nodes (each load mints fresh
ones), and silently changes what every later query returns. When the
exercise's dataset file changes mid-sitting, the same reasoning means
starting over rather than loading on top: remove the volume, recreate it,
and load the new file into the empty store. Some engines make this
unavoidable anyway — instead of loading into a running server, they build
an offline index from the dataset file ahead of time and the server only
reads that index, so restarting the server alone would keep answering from
the old data; for those, an edited dataset always means rebuilding the
index from scratch before the next query.

The devcontainer deliberately ships no RDF parser of its own (that would be
a default tool by the back door), so the engine's loader *is* the dataset's
parse check: a syntax error in the exercise's dataset surfaces as that
loader's error message. Paste it into chat verbatim, line number included,
and remember that a parser reports where it got stuck — often a line or
two after the actual mistake. Reload only once the file is fixed, clearing
whatever a failed load may have partially stored first.

Confirm the container is actually ready to accept queries (a log line, a
health endpoint, a simple `ASK` query that doesn't depend on the exercise's
own data) before trusting any result from it — a container that's still
loading a dataset can otherwise return a misleadingly empty result that
looks like a real answer.

Then confirm what the engine actually stored, separately from anything a
query returns: dump the loaded graph back out (most engines support the
SPARQL 1.1 Graph Store HTTP Protocol's `GET` on the default graph, or an
equivalent export/dump subcommand — confirm from the engine's docs) and
compare it against the dataset file the exercise wrote, triple count and
terms both. An engine is free to canonicalize literals, regenerate blank
node labels, or collapse triples that become identical on load, and a
query result alone can't distinguish "the engine stored something
different" from "the engine matched differently" — the dump is what
separates the two.

## Query it

The SPARQL 1.1 Protocol is itself part of what this repo teaches (see
CLAUDE.md's "Scope") — a plain HTTP GET with the query URL-encoded in a
`query` parameter, or a POST with the appropriate content type, against
whatever path the chosen engine exposes its SPARQL endpoint at (confirm
from its docs rather than assuming a path). `curl` against that endpoint,
requesting whichever result format the exercise is using via the `Accept`
header, is the generic mechanism — no engine-specific client library needed
for this part.

The Depth principle's internal-mechanism level asks what the engine
actually did, not only what it returned, so at spin-up also check the
chosen engine's docs for whether it can report how it evaluated a query —
an `EXPLAIN`-style plan, a query-execution tree carried in a dedicated
result format, a request parameter, or a separate CLI subcommand that
prints the algebra. When it can, fetch that alongside each query's results
and print it in chat the same way as the results themselves; when it
can't, say so plainly rather than inferring the plan from output order or
timing alone.

Some engines only load into a named graph, and what such an engine treats
as the default graph when a query names none (empty, or the union of every
graph) is its own choice, not the standard's. Until named graphs and `FROM`
have been taught in a session of their own, keep the exercise's query file
free of them and let the protocol request carry the dataset instead: the
SPARQL 1.1 Protocol's `default-graph-uri` parameter names the graph to use
as the default graph for that one request, shown in the command printed in
chat so the mechanism is visible rather than hidden:

```bash
curl -s http://<engine-ip>:<port>/<endpoint-path> \
  -H 'Accept: <result-format>' \
  --data-urlencode 'default-graph-uri=<graph-iri-the-dataset-was-loaded-into>' \
  --data-urlencode 'query@local-engine/<dir>/<query-file>.rq'
```

## Tear down

```bash
# every engine of the sitting, whatever <n> reached
docker rm -f $(docker ps -aq --filter 'name=^<dir>-engine-')
docker volume rm $(docker volume ls -q --filter 'name=^<dir>-engine-')  # only if named volumes were used
docker rmi $(docker images -q '<dir>-engine-*-image')  # only if images were built locally
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
