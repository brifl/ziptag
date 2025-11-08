# ZipTag

*A tiny, schema-less, graph-flavored data store for anything texty.*
Status: **experimental / pre-1.0**

## Why ZipTag?

Most app data is small, text-ish, and doesn’t need heavy infra. **ZipTag** gives you:

* **Schema-less** storage that still feels like a sparse, dynamic schema.
* **Simple graph**: untyped, bidirectional links between tags.
* **Make your own indices** by tagging; query from any angle.
* **Fast in-memory queries** with plan simplification and smart traversal.
* **Threaded joins** and two-way traversals.
* **Composable query variables** and pluggable match functions.
* Focus on **lightweight code** and **ever-improving efficiency**.

North star: keep the model simple, keep the ops small, keep the queries snappy.

## Caveats

* **Memory-first**: queries are optimized for RAM, not disk. If you keep gigabytes of data, expect to have spare gigabytes of memory.
* **Momentary inconsistency**: writes are write-through to memory, then flushed to disk asynchronously.
* **Tag limits**:

  * Tag types: lower-case (hyphens allowed), max 64 chars
  * Tag values: max 65,536 chars
* **Dirty reads**: queries allow concurrent updates to save memory and compute.
* **Security**: none built-in yet.

  * In memory: not encrypted
  * No access controls
  * Not encrypted at rest by default (opt-in planned)
* **Single node** only.
* **Performance is on you**: if your custom match functions are inefficient, so are your searches.
* **Searching giant blobs can be slow**; file references aren’t searchable unless you tag attributes on them.

## How it works

* **All state lives in memory** for simplicity and speed.
* **Disk persistence** happens via a **write-through event pump** (asynchronous).
* **Two primitives**:

  * **ttype** (tag type)
  * **tag** (typed value + untyped links)
* **Uniqueness**: a tag is uniquely identified by **(ttype, val)**. `tagid` is a compact int64 handle for fast references.
* **Links are untyped and bidirectional**. Think “adjacency” rather than typed edges.
* **Relationship semantics** are expressed by *tags* too. For a “relationship”, create a tag whose type encodes the relation and link through it. Example: person ↔ `rel: speaks` ↔ language.
* **Query optimizer** removes redundant steps, prefers small→large traversals, and runs **multi-threaded joins**.

## Data model

```
ttype
 └─ identifier : str   # lower-case, hyphens allowed, ≤ 64 chars

tag
 ├─ tagid : int64                      # stable per tag
 ├─ val   : str                        # ≤ 65,536 chars
 └─ ttype : ttype

links
 └─ bidirectional adjacency between tags (untyped)
```

### Tag object (runtime)

```python
tag.tagid  # int
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
from ziptag import ZipTagStore  # planned module name

store = ZipTagStore(path="./zdata")  # data dir; created if missing

# Create / fetch tags (idempotent by (ttype, val))
ada     = store.add_tag("person",   "ada lovelace")
python  = store.add_tag("language", "python")
speaks  = store.add_tag("rel",      "speaks")  # relation-as-tag convention

# Untyped, bidirectional links
store.link(ada.tagid,   speaks.tagid)
store.link(speaks.tagid, python.tagid)

# Languages Ada speaks
q = '| person == "ada lovelace" > rel == "speaks" > language'
langs = store.fetch(q)                        # -> list[Tag]
print([t.val for t in langs])                 # ['python']
```

> `add_tag(ttype, val)` is **get-or-create** by `(ttype, val)`.

## Core API (subject to change)

Untyped links simplify the API:

```text
add_tag(ttype, tval) -> Tag                 # idempotent by (ttype, tval)
get_tag(ttype, tval) -> Tag|None            # optional helper

rem_tag(ttype, tag_or_id) -> None           # removes the tag and its incident links
rem_tags(query) -> int                      # remove all results of a query; returns count

link(a, b) -> int                           # create untyped, bidirectional link; returns 1 if added, 0 if existed
link_many(pairs: list[tuple[a,b]]) -> int   # number of new links added

unlink(a, b) -> int                         # returns 1 if removed, 0 if missing
unlink_many(pairs) -> int

fetch(query, *, limit=None, explain=False) -> list[Tag]
explain(query) -> Plan                      # human-readable plan

sync() -> None                              # force flush to disk
```

`a` and `b` may be `Tag` or `int` `tagid`. All operations are idempotent.

## Queries

The query language is designed for clarity and optimization with untyped links.

* A query starts with `|`.
* `>` means “follow links” (untyped). You then **filter by ttype and value**.
* Steps alternate between traversals and filters. You can filter by `ttype` and then by value functions.
* Single input and output ttypes per query keep things fast and predictable.
* Named variables can splice reusable subpaths.
* Functions plug into filters.
* DNF is supported: `(A and B) or (C and (D or E))`.

### Grammar quick-ref

```
query        := '|' step { '>' step } ;
step         := type_filter [ value_filter ] | any | varref ;
type_filter  := IDENT               # ttype name, e.g., person, language, rel
any          := '*'                 # do not constrain type at this step
value_filter := compare | funcall | group ;
compare      := '==' STRING
             | '!=' STRING ;
group        := '(' disjunction ')' ;
disjunction  := conjunction { 'or' conjunction } ;
conjunction  := predicate { 'and' predicate } ;
predicate    := compare | funcall ;
funcall      := NAME '(' args? ')' ;
args         := expr { ',' expr } ;
expr         := STRING | NUMBER | NAME | funcall ;
varref       := NAME                # a previously bound variable
```

Variables:

```
let py = (| language == "python")
| person > rel == "speaks" > py
```

### Built-in functions

```
startswith(s)
regex(expr)
num(s)                              # parse numeric from string
>=, >, <=, <                        # applied to num(val) or function results
all                                 # keep all current tags (noop filter)
any                                 # synonym for '*'
top(n, by=func?)                    # take top N, optional key function
match(func)                         # keep items where func(tag) returns truthy
match_first(keyfunc)                # keep first match per key
exclude(func)                       # drop items where func(tag) is truthy
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

### Determinism and ordering

Results are sets. Order is unspecified unless you apply `top`, `sort(by=...)` if provided, or another ordering function. Tie-breakers default to `tagid` ascending.

### Explain plans

`explain(query)` returns the simplified plan with chosen join order, cardinality estimates, and memoized sub-traversals.

## Performance notes

* Traversals run **small → big** when joining sets.
* The planner prunes redundant filters and subpaths where possible.
* **Multi-threaded** joins split large intersections across workers.
* **Memoization**: sub-traversals may be cached by `(input_tagid_set_digest, subquery_digest)` to skip recomputation.
* **Built-in indices**:

  * per-ttype map `(val -> tagid)` for get-or-create
  * adjacency sets for tag links
* `startswith` and `regex` are scan-based today. Add your own match functions as needed.
* Write path: memory first, then async durability flush.

If you search very large text blobs, pre-tag attributes such as `word:...` or `hash:...`.

## Durability and consistency

* **Atomicity**:

  * Single `add_tag`, `rem_tag`, `link`, `unlink` are transactional.
  * Bulk operations are best-effort and can be partially applied if the process dies mid-flight.
* **Visibility**: writes return after updating the in-memory graph.
* **Durability**: an async message queues the change to the file system; a crash can lose the latest window until next flush.
* **Controls**:

  * `flush_interval_ms` in `ZipTagStore(...)`
  * `sync()` to force flush

Suggested path forward:

* WAL with append-only records, plus periodic compact **snapshots**.
* On startup: replay WAL after the latest snapshot.
* Keep a monotonic `next_tagid` and persist it to avoid reuse after crash.

### Guarantees table

| Operation     | Atomic | Visible when | Durable when |
| ------------- | ------ | ------------ | ------------ |
| add_tag       | yes    | on return    | after flush  |
| rem_tag       | yes    | on return    | after flush  |
| link          | yes    | on return    | after flush  |
| unlink        | yes    | on return    | after flush  |
| bulk variants | no     | stepwise     | after flush  |

## Identity, uniqueness, and deletes

* **Uniqueness**: `(ttype, val)` is the identity. `add_tag` is idempotent.
* **tagid**: monotonic int64 assigned at creation, used as a lightweight handle.
* **Deletes**:

  * `rem_tag` removes the tag and all incident links.
  * After deletes and unlinks, ZipTag checks for **orphaned ttypes** and prunes empty ttype buckets in in-memory maps.
  * A single unlinked tag is valid and kept.
* **Duplicates and inconsistencies**:

  * Duplicate links are ignored and logged.
  * Other inconsistencies are logged for a cleanup script.
  * Invalid queries raise an exception with a reason and a suggested fix.

## Security

ZipTag has no built-in auth or encryption. Use OS file permissions. Do not expose the process to untrusted users. Encryption at rest is planned as an opt-in.

## When to use ZipTag

* You want a small, embeddable data store for text-centric data.
* You like **typed tags** plus **untyped links** more than rigid tables.
* You need fast, ad-hoc in-memory traversals and simple persistence.
* You are comfortable with eventual consistency and single-node operation.

When not to use it:

* You need strong consistency, encryption, distribution, or row-level ACLs today.
* Your data is mostly huge blobs you expect to full-text index out of the box.

## Example patterns

### Relation-as-tag

```python
ada     = store.add_tag("person",   "ada lovelace")
python  = store.add_tag("language", "python")
speaks  = store.add_tag("rel",      "speaks")

store.link(ada, speaks)
store.link(speaks, python)

# Find all .log files in /var/log using attribute tags
f = store.add_tag("file", "/var/log/app.log")
ext = store.add_tag("ext", "log")
dir = store.add_tag("dir", "/var/log")
has_ext  = store.add_tag("rel", "has-ext")
in_dir   = store.add_tag("rel", "in-dir")
store.link(f, has_ext); store.link(has_ext, ext)
store.link(f, in_dir);  store.link(in_dir, dir)

logs = store.fetch('| file > rel == "has-ext" > ext == "log" > rel == "in-dir" > dir == "/var/log" > file')
```

### Two-way traversals

All links are bidirectional. You can traverse either direction and constrain by ttype at each step.

## Roadmap

* Freeze the **query DSL** and add `explain(query)`.
* Configurable **encryption at rest**.
* **WAL + snapshots**, compaction.
* Optional disk-optimized indices for selected ttypes.
* Better **observability**: counters, latency histograms, traceable plans.
* Enrich relation patterns: light weight attributes on links via dedicated tags.

## Contributing

PRs and issues welcome once the repo opens up. Helpful contributions:

* Query DSL proposals and test cases
* Performance benchmarks
* Docs and real-world examples
* Storage backends and match functions

Please keep the project’s simplicity goal in mind.

## License

TBD (likely a permissive license such as MIT). Will be finalized before any public release.

## Appendix: API sketch (expanded)

```python
class ZipTagStore:
    def __init__(self, path: str, *, workers: int = 0, flush_interval_ms: int = 200):
        ...

    # Tags
    def add_tag(self, ttype: str, tval: str) -> "Tag": ...
    def get_tag(self, ttype: str, tval: str) -> "Tag|None": ...
    def rem_tag(self, ttype: str, tag_or_id: "Tag|int") -> None: ...
    def rem_tags(self, query: str) -> int: ...

    # Untyped links
    def link(self, a: "Tag|int", b: "Tag|int") -> int: ...
    def link_many(self, pairs: "list[tuple[Tag|int, Tag|int]]") -> int: ...
    def unlink(self, a: "Tag|int", b: "Tag|int") -> int: ...
    def unlink_many(self, pairs: "list[tuple[Tag|int, Tag|int]]") -> int: ...

    # Query
    def fetch(self, query: str, *, limit: "int|None" = None, explain: bool = False) -> "list[Tag]": ...
    def explain(self, query: str) -> "str": ...

    # Durability
    def sync(self) -> None: ...

class Tag:
    tagid: int
    val: str
    ttype: str
    links: list["Tag"]  # view-like
```