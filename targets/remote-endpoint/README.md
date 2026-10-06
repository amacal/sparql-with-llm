# remote-endpoint etiquette

Shared guidance, not an exercise itself. `remote-endpoint/` exercises query a
real, publicly-hosted SPARQL endpoint over the standard SPARQL 1.1 Protocol
— genuinely different from `local-engine/` because the dataset, the scale,
and the network are all real and shared with other users, not a disposable
container this repo controls.

Which public endpoint is appropriate is decided the same deliberate,
looked-up-fresh way as a `local-engine` exercise's engine (see CLAUDE.md's
"No hardcoded technology") — this file names none, only the etiquette that
applies whichever one is chosen:

- Always set a sane `LIMIT` while developing a query against unfamiliar,
  real-scale data — an unbounded query against a public endpoint can be slow
  for you and discourteous to everyone else sharing it.
- Respect whatever rate-limit or usage-policy page the endpoint publishes
  (fetch it live and actually read it, don't assume); back off and wait
  rather than retrying aggressively on a timeout or a `429`.
- A `SERVICE` call federating out to a *second* public endpoint from within
  a query compounds both endpoints' load and latency — worth doing
  deliberately for what it teaches about federation cost, never by
  accident.
- Treat an exercise's recorded result as a snapshot: real public datasets
  change over time, so a notes file's worked example should say when it was
  run if the exact numbers could plausibly drift later.
- No credentials are ever needed for this target — if an endpoint requests
  one, it isn't the kind of public endpoint this target is for; that
  belongs under a managed/cloud-hosted target instead (see CLAUDE.md's
  "Target environments").
