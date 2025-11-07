# ZipTag

*A tiny, schema-less, graph-flavored data store for “anything texty.”*
Status: **experimental / pre-1.0**

---

## Why ZipTag?

Most of the data many apps juggle is small, text-ish, and doesn’t need heavyweight infra. **ZipTag** gives you:

* **Schema-less** storage that still *feels* like a sparse, dynamic schema.
* **The simplest graph** you’ve ever used: tags + typed links.
* **Make your own indices** by tagging; query from any angle.
* **Fast in-memory queries** with plan simplification and smart traversal.
* **Threaded joins** and two-way traversals.
* **Composable query variables** and pluggable match functions.
* Focus on **lightweight code** and **ever-improving efficiency**.

> ZipTag’s north star: *keep the model simple, keep the ops small, keep the queries snappy*.

---

## Caveats (important!)

* **Memory-first**: queries are optimized for RAM, not disk. If you keep gigabytes of data, expect to have spare gigabytes of memory.
* **Momentary inconsistency**: writes are **write-through to memory**, then flushed to disk asynchronously.
* **Tag limits**:

  * Tag types: lower-case (hyphens allowed), max **64 chars**
  * Tag values: max **65,536 chars**
* **Dirty reads**: queries allow concurrent updates to save memory/compute.
* **Security**: none built-in yet.

  * In-memory: not encrypted
  * No access controls
  * Not encrypted at rest by default (opt-in planned)
* **Not distributed** (single node).
* **Performance is on you**: if your custom match functions are inefficient, so are your searches.
* **Searching giant blobs can be slow**; file references aren’t searchable unless you tag attributes on them.

---

## How it works (in a nutshell)

* **All state lives in memory** for simplicity and speed.
* **Disk persistence** happens via a **write-through event pump** (asynchronous).
* **Two primitives**:

  * **ttype** (tag type)
  * **tag** (typed value + typed links)
* **Every tag has one type.** Tags are immutable (you can delete and re-add).
* **Links are typed** (`links[ttype] -> list[tag]`), giving you a simple graph.
* **Query optimizer** removes redundant steps (even partial overlaps), picks smaller → larger traversals, and runs **multi-threaded joins**.

---

## Data model

```
ttype
 └─ identifier : str   # lower-case, hyphens allowed, ≤ 64 chars

tag
 ├─ tagid : int64
 ├─ val   : str       # ≤ 65,536 chars
 ├─ ttype : ttype     # denormalized for locality
 └─ links : dict[ttype, list[tag]]  # typed edges (two-way traversal supported)
```

---

## Install

ZipTag isn’t on PyPI (yet). Use it as a local dependency:

```bash
# From this repo root:
python -m venv .venv && source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -U pip
pip install -e .
```

> Prefer a dedicated virtual environment. Python 3.9+ recommended.

---

## Quick start

```python
from ziptag import ZipTagStore  # planned module name

store = ZipTagStore(path="./zdata")  # data dir; created if missing

# Create / fetch tags (idempotent by value within a type)
ada    = store.add_tag("person", "ada lovelace")
python = store.add_tag("language", "python")
paper  = store.add_tag("doc", "notes on computing machinery")

# Typed links (edges). “speaks”, “authored”, “mentions” are just tag types.
store.link("speaks",  [ada.tagid, python.tagid])
store.link("authored", [ada.tagid, paper.tagid])
store.link("mentions", [paper.tagid, python.tagid])

# Simple fetch by query (DSL is evolving; see “Queries”)
q = '| person > startswith("ada") > speaks > language'
langs = store.fetch(q)  # -> list[tag]
print([t.val for t in langs])  # ['python']
```

> `add_tag(ttype, value)` is **get-or-create** within `(ttype, value)`.

---

## Core API (subject to change)

```text
add_tag(ttype, tval) -> tag            # returns existing if (ttype, tval) already present
rem_tag(ttype, tag)                    # remove by object
rem_tag(ttype, tagid)                  # remove by id
rem_tags(query)                        # remove all results of a query
fetch(query) -> list[tag]              # execute a query DSL string

link(ttype, tag)                       # link by object
link(ttype, list[tag])                 
link(ttype, tagid)                     # link by id
link(ttype, list[tagid])               
link(query)                            # link results of a query (advanced)

unlink(ttype, tag)
unlink(ttype, list[tag])
unlink(ttype, tagid)
unlink(ttype, list[tagid])
unlink(query)
```

### Tag object (runtime)

```python
tag.tagid     # int
tag.val       # str
tag.ttype     # str
tag.links     # dict[str, list[tag]]  (may be lazy / view-like)
```

---

## Queries

The query language is **still being designed**, but the intended shape is:

* **Pipes**: a query starts with `|` (root `"."` implied unless overridden).
* **Single input/output types** per query for clarity and optimization.
* **Traversal steps** alternate between types and filters:
  `| ttype > tag-filter > ttype > tag-filter > ttype`
* **Named variables** can splice reusable sub-paths.
* **Custom match functions** plug into filters.
* **DNF** (`(A and B) or (C and (D or E))`) supported for expanding/contracting sets.

### Built-in functions (planned)

```
==, !=, startswith, regex(expr),
>=, >, <=, <,
>=#, >#, <=#, <#      # numeric comparisons
all, any,
top(f(n)),            # take top N by function f
match(f(n)),          # keep items where f returns truthy
match_first(f(n)),    # keep first match per group/key
exclude(f(n))         # drop items where f returns truthy
```

### Tiny examples

```text
# People who speak Python
| person > all > speaks > language == "python"

# Documents that mention any language starting with "p"
| doc > mentions > language > startswith("p")

# Variables
let py = (| language == "python")
| person > speaks > py

# Numeric (by-value) compare
| build > version >=# 3
```

---

## Performance notes

* Traversals run **small → big** when joining sets.
* The planner prunes redundant filters/steps when possible.
* **Multi-threaded** joins split large intersections across workers.
* **Write path**: memory first, then async durability flush.
* If you search **huge text blobs**, consider pre-tagging attributes (e.g., `word:...`, `hash:...`) to avoid scanning the whole blob.

---

## Durability & consistency

* **Write-through memory**, **async disk**.
* Reads may observe in-flight writes (**momentary inconsistency**).
* Crash during flush may lose the very latest writes; background replays minimize drift.

---

## Security

ZipTag currently has **no** built-in auth/ZK/ACL.

* Memory unencrypted.
* Disk not encrypted by default (opt-in planned).
* Do **not** expose the store process to untrusted users.

---

## When to use ZipTag

* You want a small, embeddable data store for **text-centric** data.
* You like **typed tags** + **typed edges** more than rigid tables.
* You need **fast, ad-hoc** in-memory traversals and simple persistence.
* You’re comfortable with **eventual consistency** and **single-node** operation.

When **not** to use it:

* You need **strong consistency**, **encryption**, **distribution**, or **row-level ACLs** today.
* Your data is mostly **huge blobs** you expect to full-text index out of the box.

---

## Example patterns

### “Make your own index” by tagging

```python
# Index a file by attributes you care about
f = store.add_tag("file", "/var/log/app.log")
store.link("has-ext", store.add_tag("ext", "log").tagid)
store.link("in-dir", store.add_tag("dir", "/var/log").tagid)

# Find all .log files in /var/log
logs = store.fetch('| file > has-ext > ext == "log" > in-dir > dir == "/var/log" > file')
```

### Two-way traversals

Design your edge types so you can traverse in either direction by including both link types (e.g., `parent-of` and `child-of`), or implement symmetric semantics in the planner.

---

## Roadmap (high level)

* Finalize the **query DSL** + variable syntax.
* Configurable **encryption at rest**.
* Pluggable **on-disk formats** & **compaction**.
* Optional **disk-optimized** indices for selected types.
* Better **observability**: metrics + traceable plans.
* Fancier **link semantics** (weights, timestamps).

---

## Contributing

PRs and issues welcome once the repo opens up. Helpful contributions include:

* Query DSL proposals & test cases
* Performance benchmarks
* Docs/tutorials and real-world examples
* Storage backends and matching functions

Before proposing features, please keep the project’s **simplicity** goal in mind.

---

## License

TBD (likely a permissive license such as MIT). Will be finalized before any public release.

---

## Appendix: API sketch (expanded)

> Names/params may evolve; treat this as a working spec.

```python
class ZipTagStore:
    def __init__(self, path: str, *, workers: int = 0, flush_interval_ms: int = 200):
        ...

    # Tags
    def add_tag(self, ttype: str, tval: str) -> "Tag": ...
    def rem_tag(self, ttype: str, tag_or_id: "Tag|int") -> None: ...
    def rem_tags(self, query: str) -> int: ...  # returns count

    # Links (typed)
    def link(self, ttype: str, to: "Tag|int|list[Tag]|list[int]") -> int: ...
    def unlink(self, ttype: str, to: "Tag|int|list[Tag]|list[int]") -> int: ...

    # Query
    def fetch(self, query: str) -> list["Tag"]: ...

class Tag:
    tagid: int
    val: str
    ttype: str
    links: dict[str, list["Tag"]]
```

---
