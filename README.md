# ZipTag

*A tiny, schema-less, graph-flavored data store for anything texty.*
Status: **experimental / pre-1.0**

## Why ZipTag?

Most app data is small, text-ish, and doesn’t need heavy infra. **ZipTag** gives you:

* **Schema-less** storage that still feels like a sparse, dynamic schema.
* **Simple graph**: untyped, bidirectional links between tags.
* **Make your own indices** by tagging; query from any angle.
* **Fast in-memory queries** with plan simplification and smart traversal.
* **What-if overlays**: run queries against *hypothetical* changes without committing.
* **Transactional packages**: apply a set of adds/deletes/links as one atomic revision.
* **Composable query variables** and pluggable match functions.
* Focus on **lightweight code** and **ever-improving efficiency**.

North star: keep the model simple, keep the ops small, keep the queries snappy.

## Caveats

* **Memory-first**: queries are optimized for RAM, not disk. Keep spare RAM proportional to live data.
* **Eventual durability** by default: memory updates are journaled and fsynced in batches.
* **Tag limits**:

  * Tag types: lower-case (hyphens allowed), max 64 characters
  * **Tag values: max 1,024 characters** (store larger payloads behind a `file-ref`)
* **Identity is immutable**: `(ttype, val)` is identity; **no renames**.
* **Security**: none built-in yet.

  * In memory: not encrypted
  * No access controls
  * Not encrypted at rest by default (opt-in planned)
* **Single node** only.
* **Performance is on you**: custom match functions define your search cost.
* **Big blobs**: not searchable unless you pre-tag attributes.

## How it works

* **All state lives in memory** for simplicity and speed.
* **Durability**: minimal append-only **WAL** + periodic **snapshots**.
* **Two primitives**:

  * **ttype** (tag type)
  * **tag** (typed value + untyped links)
* **Identity**: a tag is uniquely and immutably identified by **(ttype, val)**.
* **Handle**: `tref` is a compact int64 for fast references (not identity).
* **Links** are untyped and bidirectional; express relationship semantics via *tags* (relation-as-tag).
* **Revisions**: every committed transaction produces a new monotonic **rev**.

  * You can run queries at a chosen `rev` or against **overlay layers** (hypothetical packages) before commit.

## Data model

```
ttype
 └─ identifier : str   # lower-case, hyphens allowed, ≤ 64 chars

tag
 ├─ tref  : int64                      # stable handle (not identity)
 ├─ val   : str                        # ≤ 1,024 characters
 └─ ttype : ttype

links
 └─ bidirectional adjacency between tags (untyped)

versioning (in-memory metadata)
 ├─ create_rev : u64
 └─ delete_rev : u64 | ∞               # tombstone when deleted in a later rev
```

### Tag object (runtime)

```python
tag.tref   # int
tag.val    # str
tag.ttype  # str
tag.links  # list[Tag] (may be lazy / view-like)
```

## Install

ZipTag isn’t on PyPI yet. Use it as a local dependency.

```bash
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -U pip
pip install -e .
```

Python 3.9+ recommended.

## Quick start

```python
from ziptag import ZipTagStore

store = ZipTagStore(path="./zdata")  # data dir; created if missing

# Create / fetch tags (idempotent by (ttype, val))
ada     = store.add_tag("person",   "ada lovelace")
python  = store.add_tag("language", "python")
speaks  = store.add_tag("rel",      "speaks")  # relation-as-tag convention

# Untyped, bidirectional links
store.link(ada.tref,   speaks.tref)
store.link(speaks.tref, python.tref)

# Languages Ada speaks
q = '| person == "ada lovelace" > rel == "speaks" > language'
langs = store.fetch(q)                              # -> list[Tag]
print([t.val for t in langs])                       # ['python']
```

> `add_tag(ttype, val)` is **get-or-create** by `(ttype, val)`.

---

# Transactions, overlays & revisions

**Goal:** represent a *package of change* (adds/deletes/links/unlinks, and optional ttype declarations) that can be:

* previewed as a **hypothetical overlay** in queries (no commit),
* **committed atomically** to produce a new **revision**,
* **resumed** safely after a crash (idempotent, WAL-backed).

## Mental model

* A **Tx** is a small **delta layer**: `(Δtags, Δlinks, Δtombstones)`.
* A **Hypothetical** query layers one or more Tx deltas **above** a base revision.
* A **Commit** merges the Tx into the base and advances `rev`.
* Operations inside Tx are **idempotent** (e.g., `add_tag` by `(ttype,val)`, `link` deduped).

## API sketch

```python
# Read the current committed revision
rev0 = store.current_rev()

# Create a transaction (parent defaults to current committed rev)
tx = store.begin_tx(parent_rev=rev0)

# Staged operations (idempotent within the tx)
p = tx.add_tag("provider", "azure")
s = tx.add_tag("slot", "2025-11-10T18:00Z/PT4H")
r = tx.add_tag("rel", "has-slot")
tx.link(p, r); tx.link(r, s)

# Preview (what-if) without committing
cands = store.fetch('| provider > rel == "has-slot" > slot',
                    overlays=[tx])

# Commit atomically: returns new revision
rev1 = tx.commit()

# Queries can target a revision explicitly
store.fetch(q, rev=rev1)
```

### Tx operations

```
tx.add_tag(ttype, tval) -> Tag
tx.get_tag(ttype, tval) -> Tag|None
tx.rem_tag(ttype, tag_or_ref) -> None
tx.link(a, b) -> int             # 1 if added, 0 if existed
tx.unlink(a, b) -> int
tx.declare_ttype(name: str)      # optional, for metadata/back-compat
tx.drop_ttype(name: str)         # only if empty; otherwise error
tx.abort()                        # discard delta
tx.commit() -> rev                # atomic merge into base
```

* **`overlays=[...]`** accepts Tx objects *or* persisted Tx IDs (if you store staged packages).
* New ephemeral tags inside overlays use temporary negative `tref` under the hood; callers see normal `Tag`.

### Isolation & layering

* Default reads: **READ_COMMITTED** (latest committed `rev`).
* **SNAPSHOT(rev)**: consistent read at a past revision.
* **OVERLAY**: `rev` + ordered overlays (last wins). Deletes (tombstones) in an overlay mask base entities.

### Backward compatibility

Legacy single-op methods (`add_tag`, `link`, …) remain and internally use short auto-commit Tx.

---

## Queries

The query language is designed for clarity over untyped links with strong set semantics.

* A query starts with `|`.
* `>` means “follow links” (untyped). Then **filter by ttype and value**.
* Steps alternate between traversals and filters. Single input and output ttypes keep plans simple.
* Named variables splice reusable subpaths.
* Functions plug into filters.
* **Overlays** are passed via API (`overlays=[...]`) or an optional DSL prefix.

### Grammar quick-ref

```
query        := ['with' overlay_block] '|' step { '>' step } ;
overlay_block:= '{' overlay_stmt { ';' overlay_stmt } '}' ;
overlay_stmt := add_tag | rem_tag | link | unlink ;
add_tag      := '+tag' '(' IDENT ',' STRING ')' ['as' NAME] ;
rem_tag      := '-tag' '(' IDENT ',' STRING ')' ;
link         := 'link' '(' ref ',' ref ')' ;
unlink       := 'unlink' '(' ref ',' ref ')' ;
ref          := NAME | '(' IDENT ',' STRING ')' ;

step         := type_filter [ value_filter ] | any | varref ;
type_filter  := IDENT
any          := '*'
value_filter := compare | funcall | group ;
compare      := '==' STRING | '!=' STRING ;
group        := '(' disjunction ')' ;
disjunction  := conjunction { 'or' conjunction } ;
conjunction  := predicate { 'and' predicate } ;
predicate    := compare | funcall ;
funcall      := NAME '(' args? ')' ;
args         := expr { ',' expr } ;
expr         := STRING | NUMBER | NAME | funcall ;
varref       := NAME
```

**Note:** The DSL overlay block is sugar for API overlays; it does **not** persist.

### Determinism and ordering

Results are **sets**. Order is unspecified unless you apply an ordering function (`top`, `sort(by=...)` if/when provided). Tie-breakers default to **`tref` ascending**.

### Built-in functions

```
startswith(s)
regex(expr)
num(s)
>=, >, <=, <
all, any
top(n, by=func?)
match(func)
match_first(keyfunc)
exclude(func)
len(s)
lower(s), upper(s)
```

Examples:

```
# People who speak Python
| person > rel == "speaks" > language == "python"

# Documents that mention any language starting with "p"
| doc > rel == "mentions" > language > startswith("p")

# Numeric comparisons via num(val)
| build > version > match(lambda t: num(t.val) >= 3)
```

---

## Durability & recovery

### Minimal WAL (transaction-aware)

WAL is append-only with transaction boundaries:

```
TXBEGIN(txid:u64, parent_rev:u64, ts:u64)
TXOP   (txid:u64, index:u32, kind:enum, payload:bytes)   # idempotent op
TXCOMMIT(txid:u64, new_rev:u64)
```

* **Apply order**: on normal operation, Tx ops update an **overlay**; commit merges into base then appends `TXCOMMIT`.
* **Batched fsync**: governed by `flush_interval_ms`; `sync()` forces flush+fsync.
* **Recovery**:

  1. Scan WAL once; collect `TXCOMMIT` set.
  2. For each committed `txid`, replay its `TXOP`s (in index order) against the base snapshot, then finalize `new_rev`.
  3. Ignore `TXBEGIN/TXOP` without `TXCOMMIT`.

Because `add_tag` and `link` are idempotent, replays are safe even if some ops were already applied before a crash.

### Snapshots

* Periodically write a compact **snapshot** at cutoff `rev=C`.
* On startup: load snapshot (state ≤ C), replay WAL for `rev > C`.

### Guarantees

| Operation     | Atomic  | Visible when | Durable when    |
| ------------- | ------- | ------------ | --------------- |
| add_tag       | yes     | on return    | after WAL flush |
| link/unlink   | yes     | on return    | after WAL flush |
| rem_tag       | yes     | on return    | after WAL flush |
| **tx.commit** | **yes** | new `rev`    | after WAL flush |
| bulk variants | no      | stepwise     | after WAL flush |

---

## Identity, uniqueness, and deletes

* **Identity**: `(ttype, val)` is immutable and unique (per `ttype`). `add_tag` is idempotent.
* **`tref`**: monotonic int64 handle assigned at creation.
* **Deletes**:

  * `rem_tag` removes the tag and all incident links (records a tombstone with `delete_rev`).
  * Empty `ttype` buckets are pruned from in-memory maps.
* **Duplicates and inconsistencies**:

  * Duplicate links are ignored and logged.
  * Invalid queries raise an exception with a reason and a suggested fix.

---

## Performance notes

* Planner favors **small → big** traversals and prunes redundant filters.
* Memoization: sub-traversals may be cached by `(input_set_digest, subquery_digest)`.
* Built-in indices:

  * per-ttype map `(val -> tref)` for get-or-create
  * adjacency sets for links
* `startswith` and `regex` are scan-based today; add match functions as needed.

---

## Examples: COD Fish (contract engine)

**COD Fish** is a marketplace for compute. Buyers and providers trade time-bounded compute slots. This exercises constraints and **what-if overlays**.

### Core ttypes

* Actors & assets: `buyer`, `provider`, `slot`, `region`, `country`
* Capabilities: `gpu`, `cpu-cores`, `ram-gb`, `storage`, `security`
* SLO/SLA: `sla-uptime`, `sla-latency-p95-ms`
* Policies: `policy`, `use-case`
* Calendars: `holiday`
* Relations (`rel` values): `has-slot`, `has-cap`, `requires-cap`, `requires-slo`, `blocks-country`, `blackout`, `resells`, `excludes-provider`, `contract`

> For large docs, store a `file-ref` and link to it.

### Seed (Python)

```python
S = ZipTagStore("./zdata")
azure = S.add_tag("provider","azure")
john  = S.add_tag("provider","johns-garage-rig")
has_slot = S.add_tag("rel","has-slot"); has_cap = S.add_tag("rel","has-cap")
slot2 = S.add_tag("slot","2025-11-10T18:00Z/PT4H")
S.link(azure, has_slot); S.link(has_slot, slot2)
for c in ["rtx-5090","32","128","us-west","ssd","99.9","200"]:
    # map to ttypes: gpu, cpu-cores, ram-gb, region, storage, sla-uptime, sla-latency-p95-ms
    pass
```

### Query: buyer needs `us-west`, ≥32 cores, ≥128 GB, SSD, latency ≤200 ms

```
| slot > overlaps("2025-11-10T14:00Z","2025-11-10T20:00Z")
> rel == "has-cap" > region == "us-west"
> rel == "has-cap" > cpu-cores > num(val) >= 32
> rel == "has-cap" > ram-gb    > num(val) >= 128
> rel == "has-cap" > storage   == "ssd"
> rel == "has-cap" > sla-latency-p95-ms > num(val) <= 200
> rel == "has-slot" > provider
```

### What-if overlay: buyer excludes a provider; provider adds a new slot

**API overlay**

```python
tx = S.begin_tx()
excl = tx.add_tag("rel","excludes-provider")
tx.link(tx.add_tag("buyer","acme-ai"), excl)
tx.link(excl, tx.add_tag("provider","azure"))
# propose new slot for John's rig
s2 = tx.add_tag("slot","2025-11-10T22:00Z/PT2H")
tx.link(tx.add_tag("provider","johns-garage-rig"), tx.add_tag("rel","has-slot"))
tx.link(tx.add_tag("rel","has-slot"), s2)

cands = S.fetch(q, overlays=[tx])   # overlay affects only this call
```

**DSL overlay**

```
with {
  +tag(rel, "excludes-provider") as EX;
  link( (buyer,"acme-ai"), EX );
  link( EX, (provider,"azure") );
  +tag(slot,"2025-11-10T22:00Z/PT2H") as S2;
  link( (provider,"johns-garage-rig"), (rel,"has-slot") );
  link( (rel,"has-slot"), S2 );
}
| slot > rel == "has-slot" > provider
```

### Buying (contract as tag)

```python
tx = S.begin_tx()
c   = tx.add_tag("contract", "acme-2025-11-10-azure-slot2")
r   = tx.add_tag("rel","contract")
tx.link(c,r); tx.link(r, tx.add_tag("buyer","acme-ai"))
tx.link(c,r); tx.link(r, S.add_tag("provider","azure"))
tx.link(c,r); tx.link(r, S.add_tag("slot","2025-11-10T18:00Z/PT4H"))
rev = tx.commit()
```

Find future contracts for a buyer:

```
| contract > rel == "contract" > buyer == "acme-ai" > rel == "contract" > slot
> overlaps("2025-11-01T00:00Z","2025-12-01T00:00Z")
```

---

## Planned telemetry

Telemetry is **off by default**. When enabled:

* **Counters**: ops by type (`add_tag`, `link`, `fetch`), plan cache hits/misses, adjacency inserts/removes, WAL appends/fsyncs, snapshot builds, tx begin/commit/abort.
* **Gauges**: tag count, link count, per-ttype counts, WAL queue depth, bytes since last snapshot, current `rev`.
* **Latency**: P50/P95/P99 for `fetch`, `link/unlink`, WAL flush, snapshot build, `tx.commit`.
* **Export**: pull-based JSON (e.g., `store.dump_metrics()`), no background threads on hot paths.

---

## Roadmap

* Freeze **query DSL** (overlay block included) and `explain(query)` with cardinalities.
* **Minimal WAL + snapshots** (done), compaction details and tooling.
* Optional disk-optimized indices for selected ttypes.
* Observability polish (metrics schema, plan traces).
* (Backlog) Typed edges if we ever need link attributes.

## Contributing

PRs and issues welcome once the repo opens up. Helpful contributions:

* Query DSL proposals and test cases
* Performance benchmarks
* Docs and real-world examples
* Storage backends and match functions

Please keep the project’s simplicity goal in mind.

## License

TBD (likely a permissive license such as MIT). Will be finalized before any public release.

## Appendix: API sketch (updated)

```python
class ZipTagStore:
    def __init__(self, path: str, *, flush_interval_ms: int = 200): ...

    # Tags (auto-commit single ops; use Tx for packages)
    def add_tag(self, ttype: str, tval: str) -> "Tag": ...
    def get_tag(self, ttype: str, tval: str) -> "Tag|None": ...
    def rem_tag(self, ttype: str, tag_or_ref: "Tag|int") -> None: ...

    # Untyped links
    def link(self, a: "Tag|int", b: "Tag|int") -> int: ...
    def unlink(self, a: "Tag|int", b: "Tag|int") -> int: ...

    # Tx lifecycle
    def begin_tx(self, parent_rev: "int|None" = None) -> "Tx": ...
    def current_rev(self) -> int: ...
    def fetch(self, query: str, *, rev: "int|None" = None,
              overlays: "list[Tx|str]|None" = None,
              limit: "int|None" = None, explain: bool = False) -> "list[Tag]": ...
    def sync(self) -> None: ...

class Tx:
    def add_tag(self, ttype: str, tval: str) -> "Tag": ...
    def get_tag(self, ttype: str, tval: str) -> "Tag|None": ...
    def rem_tag(self, ttype: str, tag_or_ref: "Tag|int") -> None: ...
    def link(self, a: "Tag|int", b: "Tag|int") -> int: ...
    def unlink(self, a: "Tag|int", b: "Tag|int") -> int: ...
    def declare_ttype(self, name: str) -> None: ...
    def drop_ttype(self, name: str) -> None: ...
    def commit(self) -> int: ...
    def abort(self) -> None: ...

class Tag:
    tref: int
    val: str   # ≤ 1,024 characters
    ttype: str
    links: list["Tag"]  # view-like
```
