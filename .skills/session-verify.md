Verify that the most recently written notes file, `.history/` entry, and `.index/sessions/<slug>/` directory conform to every rule in CLAUDE.md. Report each violation specifically — quote the offending text and cite the rule it breaks. No scripts — every check here is `Read`/`Grep`/`Glob` against the actual files, per `.index/schema.yml` and `.history/schema.yml`.

Your prompt should already include the write agent's report: every file it wrote/edited/removed, with paths, plus which concrete engine/format this session used. Use that to know what changed and where — don't re-derive it by globbing the whole `.index/`/`.history/` tree. Spot-check with targeted `Grep`/`Glob` calls (e.g. `grep -l 'title: "<Title>"' .index/sessions/*/meta.yml`, `grep -rl "<Title>" .index/sessions/*/prerequisites.yml`) rather than reading everything.

## What to check

**No hardcoded technology (never skip — the single highest-priority check in this repo):**

1. Scan every file the write agent touched for a specific engine, triple-store, or vendor name appearing as a default, a recommendation, or baked into CLAUDE.md/`.skills/*.md` wording — those files must stay engine-agnostic forever (a real name belongs only in a session's own notes file/`.history`/`.index` entry, recording what *that session* used).
2. Confirm the notes file's Overview (or a bespoke section) actually states which concrete engine/format this session used and a reason — its absence is itself a violation, not just a nice-to-have.
3. If the write agent's report shows a new target environment or engine category was added to `.index/schema.yml`'s enum, confirm that was a deliberate, explicitly-stated edit (per CLAUDE.md) — not a silent side effect.

**Exercise directory (also where other "Hard constraints" violations hide):**

1. Every query/dataset/conversion-script file reads as genuinely the user's own work (fresh, or carried over from their own earlier exercises) — flag anything that looks like a tutorial's bundled example query copied in wholesale rather than hand-written.
2. No query-builder/ORM/fluent-API artifact standing in for hand-written SPARQL text.
3. No plaintext secret (a password, an API token, a private key) anywhere in the exercise directory — flag immediately if found, regardless of any other check's outcome.
4. Query verification was actually performed — the write agent's report (or the notes file's Observable behavior section) states a hand-derived expected result compared against the actual one, and, for a `formats` exercise, a round-trip check — not just "it returned rows."

**Notes file** (skip entirely if this was a review session with no notes file — confirm that's actually why it's absent):

1. **Section order.** `# Title`, `## Overview`, bespoke theory/mechanism sections (any names), `## Observable behavior`, `## Internal mechanism`, `## Design rationale`, `## Edge cases` (optional), `## Worked example`. Flag missing/out-of-order/forbidden sections (`## Depends on`, `## Unlocks`).
2. **Prose only.** No bullets/numbered lists anywhere, including the worked example. Flag any `-`, `*`, `1.` list markers.
3. **Math formatting.** All math in `$$...$$` on its own line (rare here — only flag if one actually appears and is formatted wrong).
4. **Formula framing.** Every `$$` block needs a preceding sentence (why) and a following sentence (what it means). Flag any missing either.
5. **Paragraph length.** Flag standalone one-sentence paragraphs not attached to a neighbor.
6. **Word count.** `wc -w`. Flag if outside 600–1400 without an explicit justification in Overview or Design rationale.
7. **Worked example.** Non-trivial (not degenerate), mentally verifiable under a minute, entirely prose (no bulleted trace), concrete dataset triples/query/exact result.
8. **Citation vs. re-derivation.** A concept covered in a prior session's notes must be cited by filename, not re-derived. Flag any full re-derivation of a previously-covered result.
9. **No personal data.** Flag names, emails, or other personal information.

**`.history/` entry:**

1. **Placement and naming.** File exists at exactly `.history/<YYYY-MM>/<YYYY-MM-DD>-<slug>.yml`; its own `date:` field matches the filename's date; its own `title:` field, run through `.index/schema.yml`'s `slug_algorithm`, matches the filename's slug. Flag any mismatch.
2. **Schema completeness.** `date`, `title`, `session` block with every fixed field present: `file`, `status`, `attempted`, `explored`, `tried`, `corrections`, `bugs_found`, `completed`, `not_completed`, `open_questions`, `notes`. Flag any missing field, even when empty.
3. **Empty-field convention.** Empty = exactly `—`, never an empty list, never omitted. Flag any list mixing `—` with real entries, and flag `bugs_found` not `—` on a review entry (no new code, no new defects possible).
4. **No `Depends on`/`Unlocks`.** Flag any such field or resemblance anywhere — belongs solely in `.index/`.
5. **No algebra/mechanism exposition.** Flag any full query-algebra description or correctness argument (belongs in the notes file). A short factual reference naming a result ("Compared the observed result ordering against the engine's documented guarantees") is fine; reproducing the derivation is not.
6. **No personal/psychological content.** Flag personal info, personality description, psychological interpretation, inferred learning style, subjective judgment of intelligence/ability/motivation/behavior. `corrections` must be factual before/after about an assumption or prediction, never an evaluation of the person.
7. **Field discipline.** Spot-check: `completed` items are observable outcomes, not vague ("understood X," "gained insight"); `not_completed` isn't silently empty when the session mentions deferred work; `open_questions` isn't populated with an invented "natural next step" that wasn't actually raised.
8. **No retrospective edits to other entries.** Confirm (via the write agent's file list, or `git status .history/` if needed) that no `.history/` file *other than* the one new entry was touched this close, unless it's one of the immutability-rule exceptions (factual error, formatting, filename/date correction, genuinely-omitted same-session item) — and that exception is explicitly stated in the write agent's report. A change for a newly-discovered dependency/reuse/future-target is a violation — belongs in `.index/` instead.

**`.index/` — consistency and incremental-update discipline:**

1. **Directory shape.** `ls .index/sessions/<slug>/` is exactly the 10 fixed files `.index/schema.yml` enumerates (`meta.yml`, `summary.txt`, and the 8 relationship lists), plus `cross_repo_links.yml` if and only if this session actually cited/bridged a sibling repo's concept — none missing, none extra beyond that one optional file (in particular, no `reuses_code` file should exist).
2. **New entry correctly classified.** `Read` each relationship file and verify it was classified honestly against CLAUDE.md's "Fast index (`.index/`)" tests, comparing against the sibling sessions it actually cites (`Read` those specific files, not the whole tree) — e.g. flag a `prerequisites.yml` entry that's really just chronology, or a harder pattern listed as prerequisite of a simpler one. Flag a missing cross-engine/cross-format `related_to` link if this session's title names an engine/format already paired with an equivalent pattern elsewhere.
3. **`prerequisites.yml`/`uses_concepts.yml` resolve.** For each title listed, `grep -l 'title: "<ref>"' .index/sessions/*/meta.yml` finds exactly one file.
4. **`concepts.yml`/`capabilities.yml` reachable.** For each tag, `grep -rl "<tag>" .index/sessions/*/concepts.yml` (or `capabilities.yml`) includes this session's directory.
5. **Date consistency.** `meta.yml`'s `date` matches the `.history/` entry's `date` exactly.
6. **`target` validity.** `meta.yml`'s `target` field is one of the enum currently listed in `.index/schema.yml`, and matches the target implied by the `file` field's directory (e.g. `local-engine/...` implies `local-engine`).
7. **`future_targets` resolved structurally.** If this session's own title was previously a `.index/future-targets/<slug>.yml` file, confirm it's now gone (`rm`'d) — can't be both a completed session and a future target. Flag any bare string flag baked into an identifier — that belongs in the future-target file's `status: not-completed` field.
8. **No unwarranted full re-derivation.** Confirm the write agent's report lists only the new session's own files plus any explicitly-justified touches — not a sweep implying every existing session was re-read or rewritten.
9. **Retrospective revisions reflected correctly, and only in `.index/`.** If an earlier session's file was revised, confirm the write agent's report states a specific reason tied to this new session's evidence, confirm only that one file changed (not a cascade), and confirm it was **not** mirrored into that session's `.history/` entry.

10. **`status` field discipline.** `meta.yml`'s `status` is exactly `demonstrated` or `retained` — flag anything else. A `retained` status is only ever legitimate as the direct result of a `.index/cold-runs/<slug>.yml` for that exact title resolving to `status: passed` this session; flag a `retained` status with no corresponding passed cold-run as a violation (a false "proof of competence," exactly the failure mode this whole mechanism exists to prevent).

11. **Retention-check discipline.** The write agent's report must state the retention-check decision explicitly, one way or the other (scheduled / resolved passed / resolved failed / explicitly declined and why) — flag silence on this as a violation, never assume "no mention means no decision was needed." If a check was scheduled, confirm `.index/cold-runs/<slug>.yml` actually exists with a sensible `due_date` and `status: pending`. If one was resolved, confirm it was edited (never deleted — see `.index/schema.yml`'s `cold_runs.lifecycle`) and that a `failed` outcome was *not* silently accompanied by a `retained` promotion.

12. **Cross-repo link honesty.** If `cross_repo_links.yml` exists, confirm its `evidence_status_there` is specific (`demonstrated`/`retained`), not assumed, and that the chosen `relationship` (`cites`/`bridges`) is actually justified by what's written in this session's own query/dataset work rather than just asserted — a `cites` relationship standing in for work that was actually redone from scratch anyway is a violation, not a harmless label.

Flag as a violation any of the above left stale or inconsistent.

## Output format

List each violation as: **[File] Rule violated — quoted offending text**. If none found, say so explicitly and give the notes file's word count (or state "review session, no notes file" if applicable).
