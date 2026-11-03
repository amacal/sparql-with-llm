# Subqueries: Counting Two Multi-Valued Properties Without Join Inflation

## Overview

`agg-group-count.md` left one problem open: with two `rel:worksAt` and two `rel:employeeOf` values, both patterns in one `WHERE` grouped by person report 4 and 4 instead of 2 and 2. The fix it named, counting each property separately and joining the counts, is this exercise. In SPARQL that means subqueries: a complete `SELECT` nested inside another query's group pattern, evaluated on its own, with its result joined in like any other pattern. Inflation comes from counting after joining, so you count first and join one number per person to one number per person. Getting every person into the result with a real `0` takes one more idea, and that is where most of the work went.

The dataset is `data.ttl`, copied verbatim from `agg-group-count/data.ttl`: 18 triples about five named people. Bob has two values on each property, Alice one `rel:worksAt` only, Ben one `rel:employeeOf` only, Monica one of each (her `rel:employeeOf` a blank node), and James neither. The expected answer was worked out by hand before any query existed: (Bob, 2, 2), (Alice, 1, 0), (Ben, 0, 1), (Monica, 1, 1), (James, 0, 0), in no particular order. The target is `local-engine`. The primary engine was Apache Jena Fuseki 6.2.0, built from Apache's official Maven kit `jena-fuseki-docker-6.2.0.zip`. The plan agent's live lookup chose it because `arq.qparse --print=op` prints the algebra, which shows a subquery's own `project` as a barrier inside the outer `join`. A silent panel ran every query alongside it: Oxigraph 0.5.13, Eclipse RDF4J 6.1.0, QLever (digest-pinned), Virtuoso Open Source 7.2.17 (named graph plus `default-graph-uri`), and Tentris v1.5.0 (free for non-commercial use, proprietary).

## Join, and where inflation comes from

`union-alternatives.md` established that Join merges every compatible pair of left and right solutions. Two solutions are compatible when no variable bound in both has different values, and the merge is the union of the two mappings. With two solutions per person on each side, sharing only `?person`, all four pairs are compatible, so each `?a` value appears once per `?b` value. `COUNT(?a)` receives the multiset of `?a` evaluated over the group, as defined in `agg-group-count.md`, here `{"1", "1", "2", "2"}`, so the count is 4. Grouping and projection make no copies. The copies already exist once the join has run, and grouping just counts them.

## Subquery scope

The fetched §12 text says that subqueries "are evaluated logically first, and the results are projected up to the outer query", and that "only variables projected out of the subquery will be visible". In algebra terms, §18.2.2 translates a SubSelect into `ToMultiSet` of its own complete translation, so the subquery ends in its own Project. Project restricts each solution's domain to the listed variables. Every other variable is removed from the mapping, not hidden. As a result, two subqueries that both use `?employer` internally share nothing but what they both project, here `?person`. Nothing passes from the outer query down into a subquery. The subquery computes its answer over the whole dataset, and the outer query joins that answer in.

## Observable behavior

All results are Fuseki's JSON, and all six engines agreed on every query. `q1` joins two grouped subqueries, one per property, side by side. It returned two rows, Bob 2 and 2 and Monica 1 and 1, as `xsd:integer`, with Alice, Ben and James missing. `q2` added an outer required `?person rel:name ?name` and wrapped each subquery in `OPTIONAL`. It returned five rows, with no `employeeOf` key for Alice, no `worksAt` key for Ben, and only `person` for James, while `head.vars` still listed all three variables. `q3` moved the anchor inside each subquery, with `?person rel:name ?name` required and the counted property in an inner `OPTIONAL`, and joined the two subqueries directly with no outer anchor. It returned exactly the reference table, five rows with every cell bound and every count `xsd:integer`, including four real zeros. `q4` is `q3` plus an outer `FILTER(?worksAt > ?employeeOf)` and `ORDER BY DESC(?worksAt)`, and it returned one row, Alice 1 and 0. QLever's counts were typed `xsd:int` throughout, as in earlier sessions.

## Internal mechanism

This is `arq.qparse`'s tree for `q3`, as Fuseki printed it:

```
(project (?person ?worksAt ?employeeOf)
  (join
    (project (?person ?worksAt)
      (extend ((?worksAt ?.0))
        (group (?person) ((?.0 (count ?employer)))
          (leftjoin
            (bgp (triple ?person rel:name ?name))
            (bgp (triple ?person rel:worksAt ?employer))))))
    (project (?person ?employeeOf)
      (extend ((?employeeOf ?.0))
        (group (?person) ((?.0 (count ?employer)))
          (leftjoin
            (bgp (triple ?person rel:name ?name))
            (bgp (triple ?person rel:employeeOf ?employer)))))))
```

Read bottom-up: in each branch the `leftjoin` gives every named person at least one solution. Each person gets one group, `count` runs over that group, `extend` copies the hidden aggregate variable `?.0` to its alias, and `project` reduces each solution to `?person` plus one count. Both branches use `?name`, `?employer` and `?.0` internally. They are removed below the `join`, so the join compares only `?person`, and every person meets exactly one solution on each side. In `q1` the same shape had no anchor, so the `employeeOf` branch had no group for Alice. `agg-group-count.md`'s `q9` already showed that an explicit `GROUP BY` over zero solutions gives no groups. With no group, `count` was never called, no row existed, and the plain join had nothing to pair Alice with. `q2`'s outer `OPTIONAL` kept her anchor row but had nothing compatible to extend it with, so `?employeeOf` stayed unbound. No `0` was ever computed and lost; a `0` only appears when COUNT actually runs over a group whose multiset is empty, and `q3`'s inner anchor is what creates that group.

## Design rationale

The shorter alternative, `agg-group-count.md`'s `q6`, keeps the inflated join and uses `COUNT(DISTINCT ...)`, and it pays twice. It builds the product of each person's two value counts and then discards most of it in deduplication, where subqueries build the sum and finish with a one-to-one join. More seriously, DISTINCT compares values, so it silently undercounts when two different matches share one. Two distinct city resources both named "Springfield" count once under `COUNT(DISTINCT ?cityName)`, whereas a subquery still has `?city` in scope, keeps both solutions, and counts 2.

`q3` and `q2` with an outer COALESCE would give the same rows. The difference is that each `q3` subquery, run on its own, is already the complete answer for its property, zeros included, while the COALESCE version is only correct together with its outer anchor and patch. `q3`'s cost is evaluating the anchor once per subquery instead of once, which wastes a full scan of everyone when the properties are sparse. `q4`'s condition had to be an outer `FILTER`. Inside the `worksAt` subquery `?employeeOf` does not exist yet, and an outer `HAVING` would form one implicit group.

## Edge cases

Unbound and zero look different in JSON: an absent key in `q2`, `"0"^^xsd:integer` in `q3`. They also behave differently downstream. On `q2`'s output, `q4`'s filter would raise an error comparing Alice's `1` with her unbound `?employeeOf`, and `filter-unbound.md` showed that an error drops the solution. Monica's blank-node `rel:employeeOf` counts as 1, because a blank node is a bound term like any other. Both malformed versions of `q4`'s ordering, `?worksAt desc` and `(desc ?worksAt)`, were rejected with HTTP 400 by all six engines, because rule [24] requires `DESC` followed by a bracketed expression.

## Worked example

Trace a hypothetical sixth person, Zoe, with a name, three `rel:worksAt` values (`cmp:A`, `cmp:B`, `cmp:C`) and no `rel:employeeOf`, through `q3`'s tree; this was predicted in dialogue, never loaded or run. In the `worksAt` branch, the anchor matches Zoe once, and the left join extends that solution with each of her three companies, so three solutions reach the group. Her group's multiset of `?employer` values is `{A, B, C}`, and the count is 3. In the `employeeOf` branch, the anchor matches her once and the left join finds nothing compatible, so one solution with `?employer` unbound reaches the group. Evaluating `?employer` there is an error, the error is dropped, the multiset is empty, and the count is `0`. Each `project` leaves `{?person → Zoe, ?worksAt → 3}` on one side and `{?person → Zoe, ?employeeOf → 0}` on the other. They share only `?person`, they agree on it, and the join merges them into one row, (Zoe, 3, 0). By the same reasoning she would be missing entirely from `q1`, whose `employeeOf` subquery has no group for her, and in `q2` she would appear as (Zoe, 3) with `?employeeOf` unbound.
