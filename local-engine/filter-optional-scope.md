# FILTER Inside Versus Outside an OPTIONAL Group (local-engine)

## Overview

`filter-unbound.md` established that a `FILTER` restricts the solutions of the whole group it appears in, so moving it around *within* one group changes nothing. This exercise asks the next question: what happens when the same filter moves *between* groups, specifically from inside an `OPTIONAL { }` to the outer group that contains it. The answer is that the query changes meaning. Inside the OPTIONAL, the condition decides whether an optional value gets attached to a row. Outside it, the condition decides whether the row survives at all. The dataset is a hand-written eight-triple Turtle file about four people and their employers, `data.ttl`, queried by four user-written queries, `q1-inside.rq`, `q2-outside.rq`, `q3-outside-guarded.rq` and `q4-inside-outer-var.rq`. These were also the first queries in this repo to use SPARQL `PREFIX` declarations, which expand by the same plain string concatenation `rdf-turtle-abbreviations.md` established for Turtle.

The target is `local-engine` with Apache Jena Fuseki 6.2.0. There is no prebuilt official image, so it was built from the official Maven artifact `jena-fuseki-docker-6.2.0.zip`, then started in-memory with `--file /data/data.ttl /ds`. Jena was chosen, after a fresh live lookup, because its `qparse --print=op` command prints a query's algebra in the form the SPARQL specification defines. That makes the two filter placements directly visible. It is the third engine here, after Oxigraph and RDF4J; `arq.qparse` ran from the Fuseki server jar.

## How a group becomes algebra

Before evaluating a query, an engine translates every `{ }` group into a nested algebra expression. The spec's §18.2.2.2 and §18.2.2.7 say that each group's filters are collected and removed, the rest of the group is translated, and then the result is wrapped as `Filter(condition, rest)`. That is the formal version of the §5.2.2 rule. §18.2.2.6 adds one special case for `OPTIONAL { P }`. If `P` itself translated to `Filter(F, A2)`, the filter wrapper is removed and its condition becomes the third argument of the left join, giving `LeftJoin(G, A2, F)`, where `G` is everything earlier in the outer group. Otherwise the result is `LeftJoin(G, A, true)`, which is the form `optional-unbound.md` saw printed as `expression = true`.

What the third argument does is fixed by the §18.5 definition. `optional-unbound.md` wrote it only for the case where the expression is always true. The general definition splits the left join into two parts:

$$\mathrm{LeftJoin}(\Omega_1,\Omega_2,\mathit{expr}) = \mathrm{Filter}(\mathit{expr},\mathrm{Join}(\Omega_1,\Omega_2)) \ \cup\ \mathrm{Diff}(\Omega_1,\Omega_2,\mathit{expr})$$

The first part keeps every merge of a left solution with a compatible inner solution for which the expression is true. The second part, `Diff`, keeps a left solution *on its own* only if, for all inner solutions, it is either incompatible with them or the expression on the merge is false:

$$\mathrm{Diff}(\Omega_1,\Omega_2,\mathit{expr}) = \{\mu \in \Omega_1 \mid \forall \mu' \in \Omega_2:\ \mu,\mu'\ \text{not compatible, or compatible and}\ \mathit{expr}(\mathrm{merge}(\mu,\mu'))\ \text{is false}\}$$

The important consequence is that the condition is evaluated on the merged solution, so it can see variables from both sides. The "for all" also matters: the spec's written-out expansion of LeftJoin says "there exists", which W3C errata-query-7 corrects to "for all". Only the "there exists" reading would give a left solution with one passing and one failing match an extra unbound row.

## Observable behavior

The Graph Store Protocol dump after loading showed all eight triples, with the ages as `xsd:integer` and James's `[]` as the blank node `_:B335c74333fc08a8df62a8869442eae07`. `q1-inside.rq` returned four rows in JSON: James (16) and Ben (21) with no `employer` key, Bob (46) and Alice (20) with `cmp:Tentris`. `q2-outside.rq` returned two rows in TSV, Bob/Tentris and Alice/Tentris. `q3-outside-guarded.rq`, with the outer filter `!bound(?employer) || ?employer = cmp:Tentris`, returned three rows: Bob/Tentris, Ben with an empty TSV field, and Alice/Tentris. `q4-inside-outer-var.rq`, with `?age = 20` inside the OPTIONAL, parsed and returned four rows. Alice had 20 and Tentris, while James (16), Bob (46) and Ben (21) had an empty employer. Every result matched the hand-derived prediction; only row order differed, which is allowed without `ORDER BY`.

## Internal mechanism

`qparse --print=op` printed `q1` as `(leftjoin (bgp (triple ?person rel:aged ?age)) (bgp (triple ?person rel:worksAt ?employer)) (= ?employer cmp:Tentris))`. The condition is the left join's third argument, with no `filter` operator anywhere. `q2` printed as `(filter (= ?employer cmp:Tentris) (leftjoin (bgp …) (bgp …)))`, where the left join has no expression and the filter wraps its entire output. `q3` had the same shape with the guarded expression. `q4` printed as a `leftjoin` whose third argument is `(= ?age 20)`, which refers to a variable only the left side binds. That is legal precisely because the expression is evaluated on the merge. So in `q1` and `q4` the condition is tested once per merged left-plus-inner pair, and a left solution whose merges all fail comes out alone. In `q2` and `q3` it is tested once per finished row of the left join, and a failing row is simply removed.

## Design rationale

The placement is the query writer's way of resolving an ambiguity that English leaves open. "Every person with their age, and their employer only if it is Tentris" is `q1`: every left subject survives, and a non-matching optional value just does not bind. "People who work at Tentris" is `q2`. `q3` is "people who work at Tentris, plus people with no `rel:worksAt` triple in this dataset". That is deliberately not "unemployed": a missing triple means only that nothing was stated. The `!bound` guard cannot turn `q2` into `q1`. It rescues only rows where `?employer` was never bound; a filter can only keep or drop a row, never undo a binding the OPTIONAL already made.

## Edge cases

James and Ben arrive at the same `q1` output, an unbound employer, by different routes. Ben has no compatible inner solution, so the condition is never evaluated for him. James has one, and the condition evaluates to false on it. In `q2`, Ben is dropped by an error (an unbound comparison) and James by a false result (a blank node is a different term from `cmp:Tentris`). Bob is the case the "for all" in `Diff` exists for. Not explored this session: an expression that raises an *error* inside the left join, which `Diff`'s wording leaves open.

## Worked example

The two people worth tracing are written in `data.ttl` as `ppl:Bob rel:aged 46 ; rel:worksAt cmp:ID5 ; rel:worksAt cmp:Tentris .` and `ppl:James rel:aged 16 ; rel:worksAt [] .`, and `q1`'s WHERE clause is `?person rel:aged ?age . optional { filter (?employer = cmp:Tentris) . ?person rel:worksAt ?employer . }`. The other queries keep those two triple patterns and change only the filter: `q2` moves it to the outer group, `q3` does too as `filter (!bound(?employer) || ?employer = cmp:Tentris)`, and `q4` keeps it inside as `filter (?age = 20)`. Take Bob first, who has two `rel:worksAt` triples. The left side of every query gives one solution for him, `{?person → Bob, ?age → 46}`, and the inner pattern gives two, `{?person → Bob, ?employer → ID5}` and `{?person → Bob, ?employer → Tentris}`. Both are compatible, because they agree on `?person`, so there are two merges: `(Bob, 46, ID5)` and `(Bob, 46, Tentris)`.

In `q1` the condition `?employer = cmp:Tentris` is checked on each merge. `ID5` gives false and `Tentris` gives true, so the first part of the LeftJoin keeps `(Bob, 46, Tentris)`. For `Diff`, one compatible merge was true, so the "for all … false" test fails and the bare `(Bob, 46)` is not added. Bob produces exactly one row, as Fuseki returned. In `q4` the condition `?age = 20` is false on both merges, because both carry 46, so nothing passes and `Diff` adds Bob once, with the employer unbound, again as observed. In `q2` the left join has no condition, so both merges pass through it. The outer filter then drops `(Bob, 46, ID5)` as false and keeps `(Bob, 46, Tentris)`. `q3` behaves identically for Bob, because his `?employer` is bound in both rows and the guard's `!bound` half is false.

James shows the non-equivalence of `q1` and `q3`. His single merge is `(James, 16, _:b)`. In `q1` the condition is false on it, so `Diff` returns the bare `(James, 16)`. In `q3` the condition-free left join keeps `(James, 16, _:b)`. The outer filter then computes `false || false`, because the variable is bound and the blank node is not `cmp:Tentris`, and drops the row. That gives four rows for `q1` against three for `q3`, with James the only difference.
