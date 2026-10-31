# Python Code Style Rules

Binding conventions for Python codebases. Follow them for all new code. This list grows over time as more rules get added.

## 1. Function/method "hats"

Every function and method definition is preceded by a full-width, 120-character comment line — `-` for functions/methods, `=` for classes.

Function/method hat (120 characters total):
```
# ----------------------------------------------------------------------------------------------------------------------
```

Class hat (120 characters total):
```
# ======================================================================================================================
```

Example:
```python
# ======================================================================================================================
@dataclass
class ServerConfig:
    repo_root: Path


# ----------------------------------------------------------------------------------------------------------------------
def config_dir_from_env() -> Path | None:
    ...
```

Pytest fixtures are functions and take the function hat like any other function. Two exemptions where a hat would be noise rather than structure: function definitions nested inside another function body, and one-line stub declarations (e.g. `Protocol` or abstract method bodies that are just `...`).

## 2. Single return per function/method

Every function/method has exactly one `return` statement, placed at the end. Don't early-return in the middle of a function body — restructure instead (accumulate into a result variable, use if/elif/else branches that all fall through to one final return, etc.).

Reasoning: an early return that short-circuits execution is easy to miss on a read-through. Missing it means reading to the end of the function while wondering about a case that was already handled higher up. One return at the end means nothing is silently short-circuited.

This applies to `return` statements specifically. Guard-clause `raise`s at the top of a function to validate preconditions are fine and don't count against this rule — exceptions are error signaling, not the kind of silent short-circuit this rule targets.

## 3. Type hints on every local variable

Every local variable assignment gets a type hint, not just function parameters and return types. This matters most when the variable's value comes from a function or method call, since the type isn't otherwise visible at the call site without navigating to the definition.

```python
# Not this:
result = some_module.some_function()

# This:
result: dict = some_module.some_function()
```

## 4. Import modules, not names, for project code

Never `from some_module import some_function` and then call `some_function()` bare. Import the containing module and call through it, so the call site makes clear where the function came from without checking the imports:

```python
# Not this:
from ..dependencies import require_auth
...
Depends(require_auth)

# This:
from .. import dependencies
...
Depends(dependencies.require_auth)
```

This applies to first-party code within the project. Standard library and third-party package imports (`from pathlib import Path`, `from fastapi import FastAPI`) follow normal ecosystem convention and are exempt.

## 5. Return-value order matches the function name's order

When a function/method name enumerates multiple values (via "and," or by listing them), the values it returns — e.g. a tuple — come back in that same order.

```python
# Not this:
def sha256_and_read(local_path: Path) -> tuple[bytes, str]:
    ...
    return data, digest

# This:
def sha256_and_read(local_path: Path) -> tuple[str, bytes]:
    ...
    return digest, data
```

## 6. No bare `dict` for known shapes

Never type a parameter or return value as bare `dict`, `dict[str, Any]`, or `list[dict]` when the dict actually has a fixed, known shape. Define a `pydantic.BaseModel` capturing the real keys and value types instead (not a `TypedDict` — see rule 14 for why).

```python
# Not this:
def begin_transaction(self, files: list[dict]) -> str:
    ...

# This:
class DeclaredFile(BaseModel):
    path: str
    sha256: str
    size: int

def begin_transaction(self, files: list[DeclaredFile]) -> str:
    ...
```

Positional/array-shaped data (e.g. something that serializes to/from a JSON array like `[size, sha256]` rather than a JSON object with named keys) doesn't fit `TypedDict` — use `NamedTuple` instead:

```python
# Not this: a JSON array [size, sha256] typed as bare dict values
def load_local_state(target_dir: Path) -> dict:
    ...

# This:
class SyncedFileState(NamedTuple):
    size: int
    sha256: str

def load_local_state(target_dir: Path) -> dict[str, SyncedFileState]:
    ...
```

## 7. Shared fields appear in the same order everywhere

When the same piece of data (e.g. a file's path/size/hash) shows up across multiple structures — a manifest entry, a `TypedDict`, a pydantic model, literal dict/JSON construction — the shared fields are declared and constructed in the same order in every one of them. Pick one canonical order per project for each recurring shape (for a file record, e.g. `path, sha256, size`) and use it everywhere that shape appears, rather than letting each definition drift independently.

## 8. One file per domain class; its utilities live in that same file

A domain class — a class that's a primary, named concept in the codebase (e.g. `StoreClient`) — gets its own file, named after the class in snake_case: `StoreClient` lives in `store_client.py`.

Utility functions and small supporting/utility classes that exist to support that domain class and aren't shared with anything else (exceptions like `StoreError`, pure functions like `load_local_state`) live in the same file as the domain class, not split out into a separate utilities module.

```
myapp/client/
  store_client.py   # StoreError, SyncedFileState, load_local_state(), ..., class StoreClient
  cli.py
```

**Exception: types shared across independently-deployable components go in a dedicated shared module, not duplicated in each.** If a project's server and client are meant to stay independently deployable, they don't import from each other — but the wire-format types they both need to agree on exactly (`DeclaredFile`, `TransactionManifest`, ...) live in a dedicated `protocol.py`/`types.py` module with no dependency on either side, that both sides import. This is not the same as sharing implementation — the shared module holds only the data shapes exchanged between the components, nothing about how either side handles them. Types that happen to look similar but are used by only one side stay put in the file that uses them; moving something to the shared module is warranted only when both sides actually need the same definition.

**File placement is independent of key shape.** Whether a class's primary key is surrogate (an independent concept) or composite with its parent's key (a dependent/weak-entity concept) is a data-model decision; which file it lives in is a readability decision, and neither implies the other. A dependent class that consumers reference directly — one with its own query methods, its own child tables, or meaningful behavior — gets its own file like any other domain class (e.g. a `Sleeve` keyed `(fund_id, mandate_id)` still lives in `sleeve.py`). Only small satellites manipulated exclusively through their parent (policy rows, settings records) stay in the parent's file.

## 9. Class body layout: four ordered sections

Leave exactly one empty line between the `class Foo:` line and the first member (field or method) of the class.

Within the class body, members fall into four sections, always in this order:
1. Class-level fields (attributes declared directly in the class body, not inside `__init__`), then `@staticmethod`/`@classmethod` methods
2. `__init__` (the constructor)
3. Public instance methods
4. Private (`_`-prefixed) instance methods

Every static/class-level method goes in section 1 regardless of whether its name is public or `_`-prefixed — the public/private split in Rule 9 only distinguishes instance methods (sections 3 vs. 4). See Rule 10 for how methods are ordered *within* each of these sections.

```python
class StoreClient:

    # ------------------------------------------------------------------------------------------------------------------
    @staticmethod
    def _raise_for_status(response: httpx.Response) -> None:
        ...

    # ------------------------------------------------------------------------------------------------------------------
    def __init__(self, base_url: str, token: str | None = None) -> None:
        ...

    # ------------------------------------------------------------------------------------------------------------------
    def close(self) -> None:
        ...
```

Data-model classes — pydantic models, `NamedTuple`s, dataclasses, exception classes — whose bodies are just field declarations (and possibly methods on a `NamedTuple`) may omit the blank line; the layout rules matter for behavior classes, where the class body has sections to organize.

## 10. Within a section, order methods by call sequence

Within the public-methods section (Rule 9 §3) and separately within the private-methods section (§4), order methods to match the call sequence rather than alphabetically or by when they were added — the same reasoning as TypeScript Rule 5 (entry point first, supporting functions in call order), applied inside each of Rule 9's sections.

Public methods aren't usually called by each other, so their relative order within §3 is whatever reads best for the class (e.g. grouped by lifecycle, or by how central each one is). The private section (§4) is where call-order grouping does the real work: each public method's private helpers appear together, in the order that public method calls them — the first helper it calls, then that helper's own private helpers (recursively), then the next helper it calls, and so on — followed by the next public method's helpers, in the same fashion. Section 1's static/class-level methods are ordered the same way among themselves, relative to whichever method(s) call them.

```python
# Not this: private helper before its single caller, and no call-order grouping
def _upload_file(self, ...): ...
def upload_files(self, ...): ...   # loops over _upload_file, nothing else

# This: §3 (public) has upload_files; §4 (private) has _upload_file right after it
def upload_files(self, ...): ...
...
def _upload_file(self, ...): ...
```

If a class has more than one public entry point, each one's private helpers are still grouped together and kept in call order within §4 — don't interleave one entry point's helpers with another's:

```python
class PriceDatabase(Database):
    # section 1: static/class-level, in call order relative to ingest_cobra below
    def _cobra_dbname(cobra_folder): ...
    def _remove_sqlite_family(path): ...
    def _connect_builder(path, cache_mib): ...
    def _insert_price_rows(connection, rows, batch_size): ...
    def _iter_all_price_rows(cobra_folder): ...
    def _cobra_lastupdate(cobra_folder): ...

    # section 3: public methods
    def apply_decoded_update(self, ...): ...
    def ingest_cobra(self, ...): ...

    # section 4: private instance methods, grouped by entry point, in call order
    def _check_schema_version(self, ...): ...      # apply_decoded_update's helpers
    def _apply_price_rows(self, ...): ...
    def _ensure_price_schema(self, ...): ...
    def _apply_native_schema_aux(self, ...): ...
    # ingest_cobra's only helpers are static (section 1), so it contributes nothing here
```

A helper shared by more than one entry point's call tree goes with whichever entry point calls it first in file order; it does not need to be duplicated or cross-referenced.

## 11. A step of a caller-driven lifecycle never performs another step on its own

When a class exposes a multi-step lifecycle as separate public methods — e.g. `begin_transaction` → `upload_files` → `commit` or `abort` — no single method in that lifecycle may silently perform a *different* step on the caller's behalf. Each exposed step does exactly its own step, nothing more, so the caller keeps full control over when — and whether — to advance to the next one.

```python
# Not this: upload_files() secretly owns the whole lifecycle
def upload_files(self, files: dict[str, Path]) -> CommitResponse:
    txn_id = self.begin_transaction(self.declare_files(files))
    try:
        for rel_path, path in files.items():
            self.upload_file(txn_id, rel_path, path.read_bytes())
        return self.commit(txn_id)          # caller never chose to commit
    except Exception:
        self.abort(txn_id)                  # caller never chose to abort
        raise

# This: upload_files() does only its own step; the caller drives the rest
def upload_files(self, txn_id: str, files: dict[str, Path]) -> list[DeclaredFile]:
    return [self.upload_file(txn_id, rel_path, path.read_bytes()) for rel_path, path in files.items()]
```

Why this matters: if `upload_files()` silently commits or aborts, a caller can no longer inspect intermediate state, decide *not* to commit, or add more files before committing — the four-independent-steps abstraction the class advertises becomes false for that one method.

This rule is about steps of a protocol the class explicitly exposes for the *caller* to drive — it is not a ban on one method calling another in general. A private implementation helper that isn't itself an independently-exposed lifecycle step (`download_file` called from inside `sync`) is fine, because `sync` is the sole sanctioned way to perform a sync, not one step of a multi-step protocol the caller is meant to orchestrate themselves.

If a one-call convenience wrapper that does perform the whole lifecycle is genuinely wanted, give it its own distinctly-named method rather than folding that behavior into a method whose name and signature already promise it performs only one step.

## 12. Group code into commented sections

Break a function/method body into logical sections, each doing one coherent piece of work, and separate each section from the next with a single blank line. Precede each section with a one-line comment explaining what that section is doing, and why when that isn't obvious from the code alone. Don't number the steps ("1. first we...", "2. then we...") — describe what's happening, and the reasoning behind it, in prose.

Prefer a single line per comment. Only go multi-line when the explanation genuinely doesn't fit on one line — that's the exception, not the default.

Comments start lowercase, even full sentences — this applies at the start of a comment, not just to continuations.

```python
# ensure that the txn_id is valid
txn_dir: Path = require_txn_dir(txn_id)

# only files in the original TransactionManifest can be uploaded
declared: dict[str, FileMeta] = load_declared(txn_dir)
if rel_path not in declared:
    raise HTTPException(HTTP_400_BAD_REQUEST, f"path was not declared when the transaction was opened: {rel_path}")
```

A short function doing one thing with nothing meaningfully separable doesn't need sections forced onto it — this applies once a body has more than one coherent piece of work in it, not to every function regardless of size.

## 13. Construct a model by calling it, not with a dict literal

When building a value whose type is a `pydantic.BaseModel`, invoke it as a constructor with keyword arguments rather than writing a `{"key": value}` literal. This is the standard way a `BaseModel` is built, and it's what makes rule 14's attribute access possible in the first place — a dict literal assigned to a model-typed variable only satisfies the type checker, it doesn't actually produce an instance of the model.

```python
# Not this:
declared[declared_file.path] = {
    "sha256": declared_file.sha256,
    "size": declared_file.size,
    "supersedes": declared_file.supersedes,
}

# This:
declared[declared_file.path] = DeclaredMeta(
    sha256=declared_file.sha256,
    size=declared_file.size,
    supersedes=declared_file.supersedes,
)
```

This applies everywhere a model is constructed, including inside comprehensions and with `**` used to conditionally include an optional field:

```python
DirectoryEntry(
    name=entry.name,
    type="directory" if entry.is_dir() else "file",
    **({"size": entry.stat().st_size} if entry.is_file() else {}),
)
```

This is specifically about constructing a *named* model. A plain `dict[K, V]` built via a literal or comprehension (e.g. `{f.path: SyncedFileState(...) for f in manifest.files}`) isn't an instance of any model and isn't affected by this rule.

## 14. Access a known shape's fields with `.attribute`, never `["key"]`

When a value's type is a `pydantic.BaseModel` (or any other class with real attributes — `NamedTuple`, `dataclass`), read its fields with dot notation, not dict-style subscripting.

```python
# Not this:
check_no_unsafe_collision(rel_path, meta["sha256"])
for old_path in meta["supersedes"]:
    ...

# This:
check_no_unsafe_collision(rel_path, meta.sha256)
for old_path in meta.supersedes:
    ...
```

This is the reason rule 6 calls for `BaseModel` rather than `TypedDict` for a known shape: a `TypedDict` is a plain `dict` at runtime and *only* supports `["key"]` access — there's no `.attribute` form available for it, no matter how it was constructed. Getting real attribute access means the type actually needs to be a `BaseModel` (or `NamedTuple`/`dataclass`), not a matter of writing the access differently.

This rule is about a value whose type is a known model — it doesn't apply to a plain `dict[K, V]` (e.g. `declared[rel_path]` where `declared: dict[str, DeclaredMeta]` — the outer subscript is an ordinary dict lookup by key; it's `.sha256` on the `DeclaredMeta` result that this rule is about).

## 15. No bare-declare-then-unpack; return a `NamedTuple` instead

Python forbids annotations inside tuple unpacking, so a function returning `tuple[A, B]` forces callers into bare declarations followed by an unannotated unpack — which is rule 3 compliance at the price of readability:

```python
# Not this:
def load_state() -> tuple[int, dict[str, SyncedFileState]]: ...

old_seq: int
synced: dict[str, SyncedFileState]
old_seq, synced = load_state()

# This:
class SyncedState(NamedTuple):
    seq: int
    files: dict[str, SyncedFileState]

def load_state() -> SyncedState: ...

state: SyncedState = load_state()
```

When you reach for the bare-declare-then-unpack pattern, treat it as a smell: the function should return a `NamedTuple` (rule 6 already calls for this on positional shapes), giving callers one annotated variable and attribute access (rule 14) instead.

## 16. No compound `A and B` condition leading a branch chain

When a branch chain's first condition is a conjunction (`if A and B:`), every later branch carries the rest of the truth table implicitly — the reader has to reconstruct which combinations of A and B each `elif`/`else` actually covers. Nest instead, so every branch carries exactly one condition:

```python
# Not this:
if rel_path in staged_added and rel_path not in workspace_files:
    del staged_added[rel_path]
elif rel_path not in workspace_files:
    raise StoreError(f"not tracked: {rel_path}")
else:
    stage_removal(rel_path)

# This:
if rel_path not in workspace_files:
    if rel_path in staged_added:
        del staged_added[rel_path]
    else:
        raise StoreError(f"not tracked: {rel_path}")
else:
    stage_removal(rel_path)
```

The same applies to sequential guard raises — `if A and B: raise X` followed by `if A: raise Y` is the same truth table in disguise; nest both raises under one `if A:`. A compound condition in a *single* if with no chain is fine: there it's one logical test, not a decision tree the reader has to unfold. Nested chains also surface truth-table rows the compound form hides — a case that "can't happen" often turns out to be a case nobody handled.

## 17. No pass-through wrapper methods

A method whose only job is to call another method with an identical signature is not an abstraction — it's maintenance debt: every signature change must be made twice, and the wrapper invites the two names to drift apart. If the wrapped method should be public, make it public; if it shouldn't, call the private one directly. One of the pair must go.

```python
# Not this:
def _tracked_files(self) -> dict[str, FileState]: ...
def tracked_files(self) -> dict[str, FileState]:
    return self._tracked_files()

# This:
def tracked_files(self) -> dict[str, FileState]: ...
```

A method only earns a place on a class if it *adds* something — validation, error translation, coordination across collaborators. Forwarding unchanged is not adding something.

## 18. Public methods are porcelain; keep machinery private

A class's public methods should be the things you'd want a stranger to call: the operations the class exists for, plus read-only *reports* computed for consumers (e.g. `status()`). The data structures those operations and reports are built from — bookkeeping maps, raw state-file loads — are plumbing and stay private, even when a report doesn't literally duplicate them (a report may deliberately abstract over internal bookkeeping rather than expose it).

```python
# Not this: internals made public because one caller happened to want them
def staged_manifest(self) -> TransactionManifest: ...
def tracked_files(self) -> dict[str, FileState]: ...

# This: the report is the public read; the pieces behind it are private
def status(self) -> WorkspaceStatus: ...
def _staged_manifest(self) -> TransactionManifest: ...
def _tracked_files(self) -> dict[str, FileState]: ...
```

If a piece of internal state genuinely gains outside consumers, promote it then — but default to private. The report should be the only read most clients ever need.

## 19. Test the positive in an if/else

`if not A: foo; else: bar` is `if A: bar; else: foo` — but the second form is dramatically more readable, because the affirmative case comes first and the reader doesn't carry a negation across the branch. When an if has an else, phrase the test positively:

```python
# Not this:
if rel_path not in tracked:
    raise StoreError(f"not tracked: {rel_path}")
else:
    stage_removal(rel_path)

# This:
if rel_path in tracked:
    stage_removal(rel_path)
else:
    raise StoreError(f"not tracked: {rel_path}")
```

This is about if/else specifically: a guard clause with no else (`if not valid: raise`) is fine — there the negation *is* the exceptional case, and there's no branch for the reader to carry it across. A single simple (non-compound) check likewise doesn't have to use positive phrasing. `x is not None` also counts as positive phrasing (it tests "we have a value"), not a negative.

## 20. Cohesive stateful resources become a class

When a domain concept owns multiple related, mutable resources that are initialized together, shared by other entities, and torn down together, represent it as a class. Don't leave the resources as module-level globals accessed through module-level functions that merely pretend to be methods of an invisible object.

```python
# Not this: a set of globals masquerading as "the server"
_repos_root: Path | None = None
_metadata_store: MetadataStore | None = None
_object_store: ObjectStore | None = None
_repositories: dict[str, Repository] = {}

def configure(config: ServerConfig) -> None: ...
def get_repository(name: str) -> Repository: ...
def create_repository(name: str) -> Repository: ...
def close() -> None: ...

# This: the concept is explicit, state is encapsulated, and Repository can receive its Server as a collaborator
class Server:
    def __init__(self, config: ServerConfig) -> None:
        self.metadata_store: MetadataStore = SqliteMetadataStore(config.db_path)
        self.object_store: ObjectStore = ObjectStore(config.object_store_root)
        self._repos: dict[str, Repository] = {}

    def get_repository(self, name: str) -> Repository: ...
    def create_repository(self, name: str) -> Repository: ...
    def close(self) -> None: ...
```

Signals that a concept wants to be a class:

1. **Multiple pieces of mutable state always travel together** and must be initialized in a specific order.
2. **A lifecycle** (open/configure, use, close/recover) applies to the whole group atomically.
3. **Shared resources** are consumed by other domain objects that need a stable place to obtain them.
4. **You're about to swap implementations** (SQLite ↔ Postgres, local disk ↔ S3) and want the boundary behind a stable object rather than scattered global references.
5. **Tests repeatedly reconfigure the same bundle** of state, which forces awkward "reset the globals" teardown.

Pure behavior with no state can stay as module functions. A single global constant or configuration flag can stay as module data. But once the concept becomes "the thing that holds and manages these resources," make the thing real.

## 21. Constructor parameters and instance assignments share order

When a constructor takes parameters that become instance variables, assign those variables in the same order the parameters appear. Derived fields are grouped with the parameter they are derived from.

```python
# Not this: param 4 (obj_store) assigned before param 3 (store)
def __init__(self, name: str, repo_dir: Path, store: MetadataStore, obj_store: ObjectStore) -> None:
    self.name = name
    self.staging_root = repo_dir / "staging"
    self.manifest_root = repo_dir / "manifests"
    self.latest_pointer = self.manifest_root / "_latest"
    self.object_store = obj_store
    self.metadata_store = store

# This: assignments follow parameter order; derived fields stay with their source parameter
def __init__(self, name: str, repo_dir: Path, store: MetadataStore, obj_store: ObjectStore) -> None:
    self.name = name
    self.staging_root = repo_dir / "staging"
    self.manifest_root = repo_dir / "manifests"
    self.latest_pointer = self.manifest_root / "_latest"
    self.metadata_store = store
    self.object_store = obj_store
```

The goal is to make the parameter list, the constructor body, and the object's shape easy to scan together. When dependencies force a different order, keep direct parameter-to-attribute assignments in parameter order and place derived fields with their source.

## 22. Pass the domain object, not its decomposed fields

When a method operates on a domain concept that already has a named class (e.g. `Transaction`), pass the object itself rather than pulling its fields apart and sending them as separate parameters. The caller already has the object; decomposing it at the API boundary adds noise, invites parameter-order bugs, and makes later additions to the concept painful.

```python
# Not this:
self.metadata_store.create_transaction(txn.txn_uuid, txn.added, txn.removed, txn.updated)

# This:
self.metadata_store.store_transaction(txn)
```

The same applies when updating state derived from the object:

```python
# Not this:
self.repo.metadata_store.save_transaction_uploaded(self.txn_uuid, self.uploaded)

# This:
self.repo.metadata_store.save_transaction_uploaded(self)
```

**Exception:** operations that are purely about identity — e.g. `delete_transaction(txn_uuid)` or `get_transaction(txn_uuid)` — legitimately take only the identifier because the object does not yet exist (or no longer exists) on the called side. Once the callee needs the object's state, pass the object.

## 23. A nested collection with a fixed meaning gets a named type

A flat `list[X]` or `dict[K, V]` is a fine general-purpose container. But once the *structure itself* carries domain meaning — a filter represented as a list of AND-groups that OR together, for example — the bare nested type (`list[list[str]]`, `dict[str, list[str]]`) is a smell: every call site has to re-derive what each level of nesting means, nothing stops the levels from being transposed or a level being silently flattened, and the type signature alone can't tell a reader what the value *is*.

Wrap the structure in a named type whose name states the concept — a `NamedTuple`, or a `BaseModel` per rule 6 — and put the operations over the structure (matching, formatting, legacy-format upgrades) on the type as methods, so the meaning lives in exactly one place:

```python
# Not this: the OR-of-AND-groups meaning re-derived at every call site
def _load_query(self) -> list[list[str]]: ...
def _path_matches_query(rel_path: str, query: list[list[str]]) -> bool: ...

# This: the concept has a name and owns its operations
class SubsetQuery(NamedTuple):
    groups: list[list[str]]  # each group is ANDed; groups are OR'd — empty means "match everything"

    def matches(self, rel_path: str) -> bool: ...

def _load_query(self) -> SubsetQuery: ...
```

This rule is about the type that crosses function/method boundaries, not about storage formats: the on-disk JSON for a `SubsetQuery` may stay a plain nested list if that's the convenient serialization, as long as the value is wrapped in the named type at the load/save boundary and travels as that type everywhere else. If an inner level itself earns a name (e.g. a `QueryGroup` with its own behavior), wrap that too.


## 24. Return the domain type a method's name promises; parse wire formats at the boundary

A method's return value is the concept its name says it produces, in a named type — never a transport representation the caller must interpret. XML trees, raw JSON dicts, and byte streams are parsed into a `BaseModel`/`NamedTuple` at the layer that receives them, and never leak upward past that layer. If a caller converts a method's return value before it can use it, the method returned the wrong thing.

```python
# Not this: the XML tree escapes, and the caller re-parses it into an untyped dict
result = client.login()                     # XmlResult(root, url)
summary = entitlement_summary(result.root)  # dict[str, Any]

# This: login() returns what login means
result = client.login()                     # LoginResult(summary, url)
if result.summary.login == "1": ...
```

## 25. No keyword-only parameter marker (`*`)

Never use a bare `*` in a parameter list to force later parameters to be keyword-only. Every parameter stays callable either positionally or by keyword.

```python
# Not this:
def sync(self, *, full: bool) -> None: ...
sync(full=True)

# This:
def sync(self, full: bool) -> None: ...
sync(True)
```

Reasoning: `*` is unexplained syntax to anyone not already fluent in Python, and it forces a specific calling convention on every caller for a benefit (preventing an ambiguous positional call) that isn't worth that cost here. Existing call sites that already pass the argument by keyword (`sync(full=True)`) keep working unchanged after `*` is removed — keyword-only is a restriction on top of normal calling, not a different one.

## 26. The calls that matter are the steak, not the garnish

A block of code usually exists to make one or two calls that are the actual point of it — the rest is scaffolding around them. That scaffolding must never out-compete the calls it's serving: if bookkeeping (a running count, an accumulated list, a second pass to add something up) sits at the same visual weight as the calls that matter, a reader scanning the function can't tell the steak from the garnish. The fix isn't merely stylistic — if the value that bookkeeping produces isn't read by anything, delete the bookkeeping, don't just tidy it, so the calls that matter are the only thing left to look at.

```python
# Not this: the actual work -- two method calls -- is buried under bookkeeping for a count nobody reads
files_archived: int = sum(self._harvest_decoded(source, dbname) for dbname in dbnames)
files_discarded: int = sum(self._discard_downloaded(source, dbname) for dbname in dbnames)
logger.info("harvested %d update files, discarded %d raw downloads", files_archived, files_discarded)

# This: nothing competes with the two calls for attention
for dbname in dbnames:
    self._harvest_decoded(source, dbname)
    self._discard_downloaded(source, dbname)
```

The two-loops-into-one collapse above is a symptom of applying this, not the rule itself: each loop existed only to isolate its own now-unused `sum()`, so once the sums are gone the loops have no reason left to be apart. Don't over-apply the symptom — if a second pass genuinely needs the first pass's complete result before it can start (e.g. building an index in pass one that pass two looks values up in), the passes stay separate on purpose, and that separation is itself doing real work, not obscuring it. The actual test is always: read the block and ask which lines are the reason it exists; everything else is justified only as long as something downstream reads what it produces.

## 27. No chaining a lookup-or-create with a mutation

`d.setdefault(k, []).append(v)` performs two operations — find-or-create the list, then mutate it — behind one line, so a reader has to unpack the chain to see either step. Split them: bind the lookup-or-create to a local variable, then mutate it on its own line.

```python
# Not this: lookup-or-create and mutation fused into one line
dbname_2_targets.setdefault(target.dbname, []).append(target)

# This: the two operations are two visible lines
targets_for_db: list[UpdateFileMetadata] = dbname_2_targets.setdefault(target.dbname, [])
targets_for_db.append(target)
```

This is one instance of a broader principle: favor explicit, unclever code over terse code, even when the terse form is idiomatic Python. A reader should be able to scan a line and know it does exactly one thing. See also rule 28.

## 28. Comprehensions are for reshaping, not constructing, and never filtered

A comprehension is for reshaping values already in hand — a trivial value pick or transform (a bare attribute/name, a simple expression, string formatting: `{name.lower() for name in dbs}`, `[p.stem.lower() for p in paths]`) applied to every item, unconditionally. Once the body is a constructor call, a method call that does real work, or anything with a side effect, it's doing construction, not reshaping — use an explicit loop instead, so each construction gets its own line and the accumulation is visible.

The same goes for a comprehension's trailing `if` clause: don't use one, ever, not even for a single trivial condition. A filter clause reads right-to-left against the rest of the comprehension, gets worse fast once there's more than one condition (`if x not in excluded and not x.startswith(...)` is already hard to parse), and hides the skip the same way rule 29's `continue` does. An explicit loop with a guarding `if` states the same filter top-to-bottom instead.

```python
# Not this: a comprehension used to construct one UpdateStep per db
steps: list[UpdateStep] = [
    UpdateStep(self.workspace, dbname, files, keep_intermediate_files)
    for dbname, files in sorted(dbname_2_targets.items())
]

# This: explicit loop, construction and accumulation as separate visible steps
steps: list[UpdateStep] = []
for dbname, files in sorted(dbname_2_targets.items()):
    step: UpdateStep = UpdateStep(self.workspace, dbname, files, keep_intermediate_files)
    steps.append(step)

# Not this: a compound filter condition folded into the comprehension
content_tables: list[str] = [row[0] for row in rows if row[0] not in excluded and not row[0].startswith("sqlite_")]

# This: the filter is a visible guard in an explicit loop
content_tables: list[str] = []
for row in rows:
    table: str = row[0]
    if table not in excluded and not table.startswith("sqlite_"):
        content_tables.append(table)
```

## 29. No `continue` in loops

A `continue` is a loop-scoped early exit: it skips the rest of the loop body for that iteration, the same way an early `return` skips the rest of a function (rule 2). A reader scanning the loop body has to notice the `continue` and carry it forward — for every line below it, "this doesn't run when the guard above fired." Restructure instead: wrap the rest of the loop body in a positive `if` (rule 19) so the skip is a visible branch, not a silent jump.

```python
# Not this: continue hides which iterations are skipped and forces the reader to track it
for update_file_meta in manifest.files:
    if update_file_meta.dbname not in requested_dbnames:
        continue
    if update_file_meta.dbname in maintained_dbnames:
        ...
    else:
        ...

# This: the whole per-iteration body is one visible if, no silent skip
for update_file_meta in manifest.files:
    if update_file_meta.dbname in requested_dbnames:
        if update_file_meta.dbname in maintained_dbnames:
            ...
        else:
            ...
```

When the `continue` is only filtering which elements get processed at all — nothing else in the loop depends on the guard — prefer filtering the collection before the loop instead of nesting, so the loop body never needs a guard:

```python
# Also acceptable: filter before the loop instead of guarding inside it
for target in (t for t in targets if t.dbname in requested_dbnames):
    ...
```

## 30. No `break` to short-circuit a loop when a condition can express it

`break` exits a loop from partway through its body — the same abrupt-exit problem `continue` has for a single iteration (rule 29), except it hides the loop's *whole* stopping condition instead of just one iteration's skip condition. A reader can no longer trust the loop header (`while True`) to say when the loop ends; they have to read the whole body looking for the `break`. When the real stopping condition is a plain value, express it in the loop's own header instead of writing `while True:` and breaking out of it.

The common case is a read-until-empty loop:

```python
# Not this: an unconditional loop that only stops via break
while True:
    chunk: bytes = reader.read(1024 * 1024)
    if not chunk:
        break
    ...

# This: a priming read up front lets the loop's own condition end it
chunk: bytes = reader.read(1024 * 1024)
while chunk:
    ...
    chunk = reader.read(1024 * 1024)
```

The walrus form (`while chunk := reader.read(1024 * 1024):`) says the same thing in one line, but the assignment can't carry a type hint — rule 3 requires typed locals, so prefer the priming-read form above.

This isn't a ban on every `break`. A `break` inside a loop whose stopping point genuinely depends on something only discovered mid-body — searching a collection and stopping once a match is found, where the found-ness can't be hoisted into the loop header — is fine. The rule targets the specific `while True` + unconditional-`break` pattern, where the loop's real stopping condition is a value already available and could be written directly in the loop header.

## 31. Environment variables belong to the CLI layer

Reading an environment variable (`os.environ.get(...)`), naming its variable name as a constant, and resolving it against a `--flag` are all the province of the CLI entry point (`cli.py` or equivalent), not of domain/library code. A domain class takes an already-resolved value in its constructor and never touches `os.environ` or holds an env-var-name constant itself — it works identically whether it's driven from a CLI flag, a test, or another program entirely.

```python
# Not this: the domain class reaches into the environment itself, via a shared config module
class SourceRoot:
    @classmethod
    def from_env_or_flag(cls, cli_root: Path | None) -> SourceRoot:
        root: Path = config.resolve_root(cli_root, config.SOURCE_ROOT_ENV_VAR, "--source-root", "source")
        return cls(root)

# This: cli.py owns the env var name and the flag/environment precedence; SourceRoot just takes a Path
SOURCE_ROOT_ENV_VAR = "NORGATE_ROOT"

def _resolve_root(cli_root: Path | None, env_var: str, flag: str, purpose: str) -> Path:
    ...

def cmd_bootstrap(args: argparse.Namespace) -> int:
    source: SourceRoot = SourceRoot(_resolve_root(args.source_root, SOURCE_ROOT_ENV_VAR, "--source-root", "source"))
    ...
```

This keeps a whole category of environment-coupling out of modules that get imported and reused elsewhere (tests, other entry points, a future second CLI): only one file needs to change if an env var is renamed, added, or replaced by a config file. If a project has no separate `config.py`, this usually means such a module simply shouldn't exist — the env var names and their resolution logic live directly in `cli.py`.

## 32. Logger messages use f-strings, not `%s`/`%d` token replacement

`logger.info(...)`/`.warning(...)`/`.debug(...)`/`.error(...)` calls build their message with an f-string, not stdlib logging's lazy `%s`/`%d` placeholder-plus-args form. This is a deliberate readability choice for this codebase over the stdlib idiom (which exists to skip formatting work when the log level is disabled) — every call site reads as one self-contained string instead of a template the reader has to match up against a trailing argument list positionally.

```python
# Not this:
logger.info("bootstrapping %s", self.spec.dbname)
logger.info("%s: wrote %d table snapshots", self.spec.dbname, len(tables))

# This:
logger.info(f"bootstrapping {self.spec.dbname}")
logger.info(f"{self.spec.dbname}: wrote {len(tables)} table snapshots")
```

## 33. A plain string standing in for a richer concept is named after what it holds, not the concept

A bare `str` (or other primitive) is not the thing it identifies. `table: str` reads as if it holds a table — a richer concept with columns, rows, a schema — but it only ever holds that table's *name*. Name it for what it actually is: `table_name`. This codebase already gets this right for databases (`dbname: str`, never `database: str`); apply the same test everywhere a primitive stands in for a name/identifier of something richer: `table_name`, `column_name`, `dbname`. The suffix isn't decoration — it's what stops a reader from expecting attributes or methods the richer concept would have and the bare string doesn't have. Once a variable holds the richer object itself (a `ColumnSpec`, a `DatabaseSpec`), the bare noun (`column`, `spec`) is correct again — this rule is specifically about primitives standing in for something else.

```python
# Not this: table reads as if it holds a table, not just its name
def _write_sqlite_table_to_parquet(self, source_path: Path, table: str) -> None:
    ...

# This: the type is right there in the name
def _write_sqlite_table_to_parquet(self, source_path: Path, table_name: str) -> None:
    ...
```

## 34. Never construct an object and call a method on it in the same expression

`SomeClass(args).method()` hides a meaningful call behind what reads, at a glance, as just a constructor invocation — the eye catches the class name and the argument list and skips right past the `.method()` tagged on the end. Bind the constructed instance to a name first, then call the method as its own statement, even when the instance is never referenced again afterward.

```python
# Not this: the actual work -- running the step -- is easy to miss at the end of a long constructor call
bootstrap_step.BootstrapStep(spec, self.connection, self.source, self.output, self.archive).run()

# This: construction and the call that matters are two visible lines
step: bootstrap_step.BootstrapStep = bootstrap_step.BootstrapStep(
    spec, self.connection, self.source, self.output, self.archive
)
step.run()
```

This targets our own domain classes, not stdlib chaining idioms (`path.expanduser().resolve()`, `name.removesuffix(...).split(...)`) — those aren't hiding a constructor's follow-up action, they're a value-transformation pipeline where each step is a plain function-like call on the value in hand, not a meaningful, separately-named action worth its own line.

## 35. No docstrings or comment blocks at the top of a module, class, function, or method

A file, class, function, or method never opens with a docstring or a block of comments describing what it is. The name says what it is; rule 12's one-line section comments, placed in the body at the point they apply, say what each piece does and why. Anything that would have gone in a header block either belongs next to the code it explains, as a section comment, or doesn't belong at all.

```python
# Not this:
class PriceUpdater:
    """Incremental sync of one price database's parquet tree from its
    archived update files. Never takes a SourceRoot: nothing here reads the
    main sqlite database. Delegates cursor persistence to PriceCursorStore
    and compaction to PriceTierCompactor."""

# This:
class PriceUpdater:
```

Reasoning: header blocks grow into essays that restate the code, drift out of date as the code changes, and push the code itself below the fold. The hats (rule 1) already mark where each declaration begins, and a reader who wants to know what a class does reads its public methods.

## 36. A raise nested inside a construction branch becomes the `else`, not a mid-block guard

Rule 19's guard-clause carve-out — `if not valid: raise` is fine — is for a raise at the top of a function, before any other work has started; there's no branch for the reader to carry the negation across because nothing follows it but the rest of the function. Once a `raise` sits inside a block that already exists to build and return a value, that carve-out no longer applies: the block's later code is written on the assumption that the checked-for case didn't happen, so a mid-block guard clause makes the reader hold that assumption across the raise instead of seeing it stated as a branch. Express both outcomes as one `if <premise>: <happy path> else: raise` — the same shape rule 19's own example uses — so the two outcomes are two visible branches of one decision.

```python
# Not this: a guard clause partway through a block that's already building the result
if result is None:
    if name not in metadata_store.list_repos():
        raise RepositoryNotFoundError(f"unknown repository: {name}")
    name_2_repo[name] = Repository(name, repo_metadata_store(name), object_store)
    result = name_2_repo[name]

# This: the raise is the else branch of the block's own premise
if result is None:
    if name in metadata_store.list_repos():
        name_2_repo[name] = Repository(name, repo_metadata_store(name), object_store)
        result = name_2_repo[name]
    else:
        raise RepositoryNotFoundError(f"unknown repository: {name}")
```

