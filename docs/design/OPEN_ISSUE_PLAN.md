# Open-issue plan — the 36 open issues at `5511288` (2026-10-26)

Status: DRAFT dispatch map for user review.
Grounding: `PRIORITIES.md` (canonical), its P-4d cluster analysis (2026-10-18),
`VALUE_TYPES.md` (T1 landed, #1503), `ELEMENT_IDENTITY.md` (I1–I7 landed),
`EXPLICIT_SCOPE.md` / P-4c (bound plan is the migration spine), the standing
rules in `PRIORITIES.md` §1. This doc sequences the *current* 36 open issues
into waves; it does not replace `PRIORITIES.md` — a merged slice updates both.

Ledger: 36 open issues (#504 #1068 #1098 #1099 #1335 #1360 #1362 #1378 #1380
#1381 #1383 #1397 #1402 #1414 #1416 #1418 #1419 #1425 #1430 #1435 #1436 #1443
#1444 #1445 #1447 #1474 #1478 #1482 #1487 #1490 #1491 #1495 #1513 #1515 #1521
#1536). Fields: cluster (per P-4d), wave, effort (S ≈ 1 slice-day, M ≈ 2–3,
L ≈ a design cycle + slices), verification oracle.

Standing rules (abbreviated from §1):
- One slice per PR; Neo4j 5.26 oracle compares rows, never SQL text where a
  bound-plan path exists; goldens byte-identical or a justified regenerated
  diff; fmt / clippy -D warnings / full `cargo test` gate; `PRIORITIES.md`
  §2/§4 updated in the same PR; nightly is load-bearing; silent-wrong before
  loud-error.
- P-4c freeze still applies: no per-shape repairs in legacy composition
  (`with_to_cte`, `join_builder`, `filter_builder`, legacy `cte_extraction`,
  `plan_optimizer` repairs). Fixes route through the bound plan / lowering,
  or are loud-gated in legacy with the corpus entry recorded for the slice
  that will handle it.

## Wave 0 — standing hygiene (continuous, not a PR lane)

| Issue | Item | Effort | Note |
|---|---|---|---|
| #1098 | RUSTSEC-2026-0258 (h2 via chdb-rust's reqwest 0.11) | S (watch) | Blocked upstream: supplies nothing until chdb-rust drops reqwest 0.11. Keep the suppression, re-check on each published chdb-rust release; rotate the advisory note if the plate 0.3.x line patches. No main-branch work. |

## Wave 1 — quick wins (bounded single-fix slices; no cross-slice deps)

Ordered within the wave by soundness first, then noise-reduction.

| Issue | Slice | Effort | Verify |
|---|---|---|---|
| #1495 | `rand()` / `randomUUID()` in a CTE read twice draw different values (soundness: joins on the drawn value match nothing). Extend `Lowerer::stable` / `unwind::changed_ctes` from #1395: a CTE value drawn nondeterministically must be pinned once (materialized draw or deterministic reseed) so both reads agree. | M | Neo4j oracle repro (`social_integration`, OPTIONAL drive joining on `r`); mutation: reintroduce per-read draw → test red; corpus byte-identical. |
| #1515 | Directed shortestPath fails when the edge table is named `f` — alias collision with the search CTE. Found by the I1 oracle review; bound-plan path. | S | Oracle battery from I1 (the `f`-alias repro), corpus byte-identical. |
| #1444 | Databricks: dotted column names not quoted in the shortest-path search (`UNRESOLVED_COLUMN`). Quoting route through the dialect layer (`quote_identifier` semantics from the Path-C slice). | S | Spark scratch query battery; corpus `.databricks.sql` leak check. |
| #1414 | Databricks: `WHERE x = NULL` renders `WHERE NULL` (Spark `FILTER_NOT_BOOLEAN`). Cypher `= NULL` is never true — lower as `false`-folding, one place in the bound plan. | S | Oracle: Neo4j returns 0 rows, not an error; both dialects locked by one lowering test. |
| #1474 | Whole-number Float inside a node / relationship value returns as integer — `Map(String, Dynamic)` serializes `2.0` as `2` (lower/value.rs). Spell the float with a decimal-safe marker in value construction. | M | Round-trip battery (HTTP + Bolt): `2.0` stays Float over the wire; corpus value-mode goldens only. |
| #1521 | Relationship property named `from_id`/`to_id` collides with the structural end columns (Bolt/HTTP drop the property; one silently shadows the other). Key returned-row columns by ordinal, not column name, in `transform_to_relationship` / HTTP graph output. | M | Body asserts on `RETURN r` with `from_id` property (both transports); corpus byte-identical. |
| #1513 | Labels / types / element ids spelled into SQL as Cypher literal text — a backslash in a schema name / label is read as an escape (SQL injection-adjacent). Centralize identity/literal spelling through one escaped-literal helper (`string_value_to_sql_literal` precedent). | S | Fuzz (backslash, quote, newline, unicode in label/type names); corpus byte-identical on normal schemas. |
| #1418 | ClickHouse: `split()` with empty / multi-character / non-ASCII delimiter fails (`splitByChar` is 1-byte). Route through the dialect layer: `splitByString` + empty-delimiter (char-class) handling. | S | Dual-dialect golden battery; corpus byte-identical. |
| #1416 | Databricks: ORDER BY an aggregate of an impossible pattern (`UNSUPPORTED_EXPR_FOR_OPERATOR` on `CASE WHEN count(*) …` over FROM-less `WHERE false`). Order the constant-folded head or measure the aggregate once (CTE / alias) so Spark sees a plain reference. | M | `test_vlp_aggregation.py` shape on Spark; corpus `.databricks.sql` churn scoped to the repro family. |
| #1435 | ClickHouse: `RETURN DISTINCT p, count(*)` → `Code 184 ILLEGAL_AGGREGATION`. Cypher semantics: DISTINCT on an aggregating RETURN is a no-op — the grouping already makes rows distinct. Stop adding DISTINCT keys when the RETURN aggregates (bound-plan lowering). | M | Oracle against Neo4j (including the `p, count(*)` shape); corpus regeneration justified only in the affected lowering paths. |
| #1419 | ClickHouse UNWIND gaps: NULL element in a list, `size()` of a UTF-8 element, UNWIND of a scalar property. Hardening trio in unwind lowering, each an oracle case. | M | Neo4j oracle trio; corpus byte-identical. |

Exit: ~11 issues closed or reduced to test-locked residuals; quick-win lane
re-loads from Wave 2's tail.

## Wave 2 — value types track (cluster 1, the densest: 13 issues)

T1 (catalog column types) landed #1503. The T-track continues with
`VALUE_TYPES.md` slices; each issue below is one or two T-slices, sequenced
so the cheap value-serialization fixes unblock the loud-everywhere ones.

| Issue | Slice | Effort | Verify |
|---|---|---|---|
| #1478 | ClickHouse: `=` / `<>` / `IN` of lists holding NULL elements answer true/false where Cypher answers NULL. NULL-aware comparison semantics across list/array equality in lowering. | M | Oracle trio (=, <>, IN × NULL positions), both dialects. |
| #1482 | Grouped / DISTINCT / UNION map or list takes a Date inside it as the string of its text (ClickHouse map/list serialization). Type-aware spelling of nested values. | M | Oracle battery (date-in-map/list × grouped/DISTINCT/UNION). |
| #1491 | ORDER BY a mixed-type list fails (Code 44); on ClickHouse 25.8 any mixed-type list fails (Code 386). T4 continuation after #1396: common-type meeting for lists. | M | 25.8-specific battery (CI pins the server version or a compat matrix). |
| #1430 | A property of type Array / Map held by several sources of an embedded node fails the one-type guard. Common-type widening for embedded multi-source properties. | M | Embedded-node battery (multi-source `anyMerge` shape). |
| #1490 | ClickHouse 25.8: embedded node with Array/Map/Tuple property fails every query reading the node (`anyIfOrNull`). Highest blast radius in the cluster — version-coupled lowering rework. | L | Full embedded battery against 25.8; ship behind the CI matrix pin. |
| #1360 | Bound plan: text node keys miss equal values of differently typed columns (Date vs DateTime, Bool vs UInt8, FixedString vs String) — silent empty walks. Uses T1 types in the identity spelling; the identity cluster's bound-plan half. | M/L | Identity oracle battery (typed equality matrix), 100+ shapes vs Neo4j. |
| #1068 | Temporal output not ISO-8601: datetime T-separator, duration rendered as array instead of `PnDTnHnS`. Serialization-side; pairs with #1487. | M/L | Format battery vs Neo4j (HTTP + Bolt + Graph output). |
| #1487 | Temporal constructors fail: `localdatetime(string)` (`Code 43`), `date/datetime/localdatetime({map})`, `time()` (`Code 46`). Constructor lowering per dialect. | M | Constructor battery vs Neo4j on both dialects. |
| #1436 | Embedded: writable schema without property types stores every property as a string — `writes_e2e` fails on main. T1 wiring for chdb (column types from `system.columns`/DESCRIBE in embedded load path). | M | `writes_e2e` green on main without types; declared/derived types agree. |
| #1536 | Int128/UInt128/Int256 and wide Decimal come back as Floats over HTTP and Bolt (digits past double precision lost). The I7 residual: exact integer round-trip over both transports. | M | Round-trip battery at the exact-bit boundary (max i256 digits, big decimals); corpus byte-identical. |
| #504 | Coupled same-row OPTIONAL collapses to one row (array-valued `node_id` needs the design-cycled ARRAY-JOIN schema flag + wiring — documented as design-cycle in PRIORITIES). Keep parked until T-track types make array-ness knowable at plan time; then this becomes a bounded slice. | L (design) | Parked; revisit gate = T1 types proving array `node_id` at load. |

Exit: the silent-wrong cluster shrinks to its version-coupled tail; #1490
ships as its own release note.

## Wave 3 — lists of nodes (cluster 5): design doc first

PRIORITIES: "needs a design doc; depends on 1 and 2." Start it as soon as
Wave 2's T-slices for #1478/#1482/#1491 land (the list-meeting machinery is
the shared substrate).

| Issue | Slice | Effort | Verify |
|---|---|---|---|
| — | `docs/design/NODE_LISTS.md`: one canonical representation for a list of nodes (element list vs value list), lowering per operation (collect/UNWIND/subscript/slice/aggregation over it), and the union rule for mixed physical forms. | L (design) | Prototype matched to Neo4j 5.26 on ≥6 shapes (P-4c precedent). |
| #1362 | UNION of a literal list of nodes and `nodes(p)` fails `Code 386`. | L | Oracle on the union shape; corpus churn scoped. |
| #1380 | Lists mixing labels or types (`[a, x]`, `collect(nodes(p))`) unsupported. | L | Oracle matrix (same-label, cross-label, node+value mixes). |
| #1378 | `count` / `CASE` / `slice` / `min` of node lists + pattern comprehensions unavailable. | L | Each operation an oracle case; single lowering reused. |
| #1425 | `collect(x)` + `[n]` of a node outside the grouping fails (node columns not in GROUP BY). | M | Oracle; the GROUP-BY exclusion list is the precise fix site. |
| #1383 | Bolt UNION: `id()` returns raw keys (Post and User collide) — the non-HTTP `id()` arm of I5b; plus `reduce()`/`label()` of non-node list elements loud gaps. | M | Bolt battery (two labels, one key text) vs `elementId` semantics. |
| #1335 | DISTINCT / grouping by path values holds `any(value)` per group → 39 GiB for 2.9M paths (perf + structural: paths need the identity key, not `any`). | L | LDBC-scale benchmark in the PR (rows + bytes), memory before/after. |

## Wave 4 — walk SQL as text / Spark recursion limits (cluster 6)

| Issue | Slice | Effort | Verify |
|---|---|---|---|
| #1402 | Edge constraints: single hops read the constraint as Cypher, walks and unions as SQL text — one semantics de-tangling per attachment site. | L | Constraint battery across single-hop, walk, union; oracle rows. |
| #1443 | Databricks: shortestPath + a second recursive relation → `INVALID_RECURSIVE_REFERENCE` (Spark: one recursive step per query). Options: single-recursion constraint with a loud refusal, or a union decomposition. | M | Spark scratch battery; loud gate documented in wiki if we choose refusal. |
| #1445 | Databricks: shortestPath over several relationship types with a length condition fails on Spark. | M | Same harness as #1443. |
| #1099 | Expand bounded / exact-length variable-length paths per level instead of one recursive CTE (performance). Design first (LDBC benchmark gate), then staged implementation; complements #1443 (fewer recursive steps on Spark too). | L (design + M implementation) | LDBC SF1 benchmark: latency + bytes on the bounded/exact family; result rows identical (oracle). |

## Wave 5 — front-door and grammar remainders (clusters 4 + 3 tails)

| Issue | Slice | Effort | Verify |
|---|---|---|---|
| #1447 | Embedded: a write in a later UNION arm, or a `COUNT { }` in a write, never reaches the write path. Completes cluster 4 (#1504 landed the HTTP/UNION half there; this is the embedded + COUNT-in-write dispatch). | M | Embedded write battery (later-arm write, COUNT{}-in-write) vs HTTP behavior; part of the embedded write lane with #1436. |
| #1381 | `reduce()` with an empty-list accumulator (`Array(Nothing)` unification) + parser gaps (index-then-property, quoted comprehension variables). Completes cluster 3 (S4 did COUNT{}/arity; this is the listed remainder). | M | Empty-accumulator oracle case; parser round-trip tests; corpus byte-identical. |
| #1397 | `collect()` after UNWIND of unordered rows carries per-element sort keys — 4.5 GB / 0.4–0.5 s on 60M elements. Performance follow-up to #1395's `fixed_order`: carry sort keys only where order is load-bearing (single-pass `groupArray` + append-order preservation), or slim the tuple. | M | Benchmark harness from the issue (3M × range(1,20)): bytes + time vs valid-order corpus goldens. |

## Order, capacity, and gates

- Suggested throughput (one agent + occasional subagents): Waves 1 → 2 → 3
  design doc → its slices; Waves 4/5 interleaved when unblocked. Two lanes:
  one quick-win lane (Wave 1 then Wave 5 tails), one T-track lane (Wave 2
  then #1397); #1098 is pure watch. If two agents: agent A = Wave 1 then
  Wave 4; agent B = Wave 2 then Wave 3.
- Gate per PR (unchanged): fmt / clippy -D warnings / full `cargo test`
  (optionally via the ratchet tool; on a worktree with subagent review), the
  Neo4j oracle harness for any lowering change, corpus byte-identical or
  justified regenerated golden, nightly green after landing.
- Ledger discipline: closing an issue updates `PRIORITIES.md` §2/§4 and this
  doc's wave table in the same PR. New silent-wrong findings are filed and
  slotted into the owning wave by cluster, not patched ad hoc.
