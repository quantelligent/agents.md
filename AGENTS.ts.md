# TypeScript Code Style Rules

Binding conventions for TypeScript codebases. Follow them for all new code. This list grows over time as more rules get added. This file is independent of the Python rules in AGENTS.py.md — do not assume a Python rule applies to TypeScript unless it's restated here.

## 1. Declaration "hats"

Every top-level declaration — class, `type`, `interface`, and standalone/module-level `function` — is preceded by a full-width, 120-character comment line using `=`. A method or constructor nested inside a class body gets the lighter `-` hat instead, marking it as a member rather than a top-level declaration.

Hat lines are exactly 120 characters total, including any leading indentation — shorten the fill-character count to compensate at deeper nesting levels rather than letting the line run past 120.

Top-level declaration hat (120 characters total, no indentation):
```
// =====================================================================================================================
```

Method/constructor hat (120 characters total; shown here at one level of 4-space indentation — recompute the fill count for whatever indentation actually applies):
```
    // -----------------------------------------------------------------------------------------------------------------
```

The descriptive comment for a declaration goes underneath its hat, between the hat line and the declaration it describes — never above the hat. The hat stays the topmost visual separator of the section; the comment belongs to the declaration beneath it.

Example:
```typescript
// =====================================================================================================================
export type MavenCoordinate = {
    groupId: string;
    artifactId: string;
    version: string;
};

// =====================================================================================================================
export class MavenRelease {

    // -----------------------------------------------------------------------------------------------------------------
    constructor(private readonly repository: MavenRepository) {}

    // -----------------------------------------------------------------------------------------------------------------
    async exists(): Promise<boolean> {
        return this.repository.exists(this.coordinate, MANIFEST_FILE_NAME);
    }
}

// =====================================================================================================================
function safeJoin(root: string, relativePath: string): string | null {
    ...
}
```

A function nested inside another function's body is exempt, and so is a one-line stub declaration (e.g. an abstract/`Protocol`-style member whose body is just `...`) — same carve-out as the Python function-hat rule. A `type` or `interface` that fits on one line (e.g. `export type MavenEnv = typeof MAVEN_ENVS[number];`) falls under that same one-line exemption and doesn't need a hat; a `type` or `interface` spanning multiple lines does, same as any other top-level declaration.

## 2. No `export default`

Every export is a named export. Never `export default`.

A default export carries no name of its own — whoever imports it picks a local name with nothing to check it against, so the same thing can end up called something different at every import site with no compiler warning. A named export forces the importer to use the real name, or to opt into a different one visibly (`import { realName as aliasName }`), rather than silently.

```typescript
// Not this:
export default createMavenReleaseRouter;
// ...elsewhere...
import mavenRouter from "./release-router";   // could be named anything, nothing enforces it

// This:
export function createMavenReleaseRouter() { ... }
// ...elsewhere...
import { createMavenReleaseRouter } from "./release-router";
```

## 3. No barrel files

Never create a file whose job is only to re-export things from other files (`export { X } from "./y"`), and never import from one. Every import names the file that actually declares the thing.

Reasoning: a barrel adds a hop between "I see a name used" and "I see where it's defined" — tracing anything through one means opening the barrel first, then the real file. That cost is paid on every read, for every future reader, in exchange for shorter import lines at write time. Import the real file directly instead:

```typescript
// Not this:
// router.ts (barrel)
export { createMavenReleaseRouter } from "./router/release-router";
// ...elsewhere...
import { createMavenReleaseRouter } from "./router";

// This:
import { createMavenReleaseRouter } from "./router/release-router";
```

This applies even when several things from the same real file are needed in one place — import them together from that file, not through an intermediary.

## 4. Single return per function/method

Every function and method has exactly one `return` statement, placed at the end. This includes Express route handlers — a chain of `if (!x) return res.status(400)...` guard clauses is exactly the kind of easy-to-miss short-circuiting this rule targets, not an exception to it.

A `throw` is error signaling, not a return, and is exempt — guard-clause throws at the top of a function to validate preconditions are fine.

A handler with many guard clauses almost always fixes itself the same way: accumulate the outcome into a variable across an if/else chain (guards included — `if (!x) { response = ...; } else { ... }` instead of `if (!x) return ...;`), and return that variable once at the end. Reach for this first. It's usually enough on its own, and each branch reads as "this condition produces this response," in place, rather than short-circuiting.

```typescript
// Not this: six early returns for six different conditions in one function
async function handleReleaseGet(req: Request, res: Response) {
    const target = parseRelativeMavenPath(...);
    if (!target) return invalidResponse(res);
    if (target.kind === "metadata") { ...; return sendXmlOrChecksum(...); }
    const buffer = await ...;
    if (!buffer) return res.status(404).json({ error: "not_found" });
    ...
    return sendBufferOrChecksum(...);
}

// This: same branches, same order, but each one assigns instead of returning
async function handleReleaseGet(req: Request, res: Response): Promise<Response> {
    const target = parseRelativeMavenPath(...);
    let response: Response;
    if (!target) {
        response = invalidResponse(res);
    }
    else if (target.kind === "metadata") {
        ...
        response = sendXmlOrChecksum(...);
    }
    else {
        const buffer = await ...;
        response = buffer ? sendBufferOrChecksum(...) : res.status(404).json({ error: "not_found" });
    }
    return response;
}
```

Don't defer the response into a `let send: () => Response` closure assigned per branch and invoked once at the end — that adds a layer with no payoff, since `res.json()`/`res.send()`/`res.end()` already execute synchronously and return `Response` directly. Assign the actual return value to the variable, not a thunk that produces it later.

Only split a handler into a separate resolve-then-respond pair of functions when there's a real second reason beyond single-return to want that boundary — e.g. the resolved value has more than one consumer, or the resolution logic is genuinely reused elsewhere. Don't do it just because the accumulate-into-a-variable version has many branches: inventing an intermediate result type that exactly one function produces and exactly one other function immediately re-switches on is a tiny DSL with a single producer and a single consumer — it adds a layer of indirection (jump to the other function, re-derive which case you're in) without decoupling anything, since the two functions are 1:1 coupled to that type either way.

## 5. Entry point first, supporting functions in call order

When a file has one function that's the reason the file exists — the one other files import and call — that entry point comes first in the file, not last. Every supporting function that exists only to be called (directly or transitively) by the entry point follows it, ordered to match the call sequence: a helper appears before the next helper it hands off to, and before functions that only that later helper needs.

Reasoning: a reader opens the file because they want to understand the entry point. Reading top-to-bottom should trace the call graph in the same order execution follows it, instead of making the reader piece the flow together from the bottom up or jump around to find where each helper fits.

```typescript
// Not this: helpers first, entry point buried at the bottom
async function resolveReleaseGet(req: Request): Promise<ReleaseGetResult> { ... }
async function respondToReleaseGet(req: Request, res: Response, result: ReleaseGetResult): Promise<Response> { ... }
export async function handleReleaseGet(req: Request, res: Response): Promise<Response> {
    const result = await resolveReleaseGet(req);
    return respondToReleaseGet(req, res, result);
}

// This: entry point first, then its helpers in the order they're called
export async function handleReleaseGet(req: Request, res: Response): Promise<Response> {
    const result = await resolveReleaseGet(req);
    return respondToReleaseGet(req, res, result);
}
async function resolveReleaseGet(req: Request): Promise<ReleaseGetResult> { ... }
async function respondToReleaseGet(req: Request, res: Response, result: ReleaseGetResult): Promise<Response> { ... }
```

Type declarations used by the entry point's signature or a helper's may still precede all functions, grouped with the imports — this rule is about function order, not where types sit. TypeScript's function hoisting means this is purely a readability convention, not something the compiler enforces or cares about either way.

A file with several independent, equally-important exports (e.g. a grab-bag of unrelated utility functions, or several route handlers mounted side by side with no call relationship between them) doesn't have a single "entry point" and is exempt — this rule applies to files built around one orchestrating function and its private helpers.

## 6. Ternaries only for trivial value picks

The ternary operator (`cond ? a : b`) is for picking between two small, side-effect-free values — literals, simple identifiers, a single short expression on each side. Once a branch is a function call, a multi-line object/array literal, or anything else substantial, use `if`/`else` instead, even though that means declaring the target as `let` and assigning it in each branch.

Reasoning: a ternary reads as "this expression evaluates to one of two values." That works when both branches are simple enough to hold in your head as you read the condition. Once a branch is a function call or a multi-line literal, what's actually happening is a control-flow decision, not a value pick — and `if`/`else` states that plainly, top to bottom, instead of asking the reader to parse a condition and two branches as a single expression.

```typescript
// Not this: the ternary is dispatching to two different code paths, not picking a value
result = baseFileName === "maven-metadata.xml"
    ? parseReleaseMetadataPath(envName, segments, checksum)
    : parseReleaseArtifactPath(envName, segments, baseFileName, checksum);

// This:
if (baseFileName === "maven-metadata.xml") {
    result = parseReleaseMetadataPath(envName, segments, checksum);
}
else {
    result = parseReleaseArtifactPath(envName, segments, baseFileName, checksum);
}

// A trivial value pick is still fine as a ternary:
extension: baseFileName === expectedPom ? "pom" : "jar",
```

## 7. Prefer namespace imports over named imports

Import a module as a namespace (`import * as x from "./y"`) and access its exports through that namespace (`x.someExport`), rather than destructuring named imports (`import { someExport } from "./y"`) and calling it bare. This applies to first-party modules and third-party packages alike.

```typescript
// Not this:
import { Router } from "express";
import { createMavenReleaseRouter } from "./router/api/release-router";
...
const router = Router();
router.use("/maven", createMavenReleaseRouter());

// This:
import * as express from "express";
import * as releaseRouter from "./router/api/release-router";
...
const router = express.Router();
router.use("/maven", releaseRouter.createMavenReleaseRouter());
```

Reasoning: the namespace prefix at the call site shows where a name came from without checking the imports, the same motivation as Python Rule 4 ("import modules, not names"). It also sidesteps collisions between similarly-shaped exports from sibling modules — this codebase has many `createMavenXRouter`-style factory functions across separate router files; named imports of several side by side either collide or need ad hoc renaming at the import line, while a namespace import keeps each one's own prefix automatically. Type-only exports work the same way — `import * as t from "./types"` still lets you write `t.MavenEnv` in a type position.

**Exception: test-framework DSL imports.** `describe`/`it`/`expect`/`beforeEach`/etc. from `vitest` (and similarly-shaped test runners) are named-imported everywhere in the JS ecosystem as a near-universal convention; writing `vitest.describe(...)` fights that convention for no benefit. Leave test-runner imports as named imports.

## 8. Type annotations on every local variable

Every local variable declaration gets an explicit type annotation, not just function parameters and return types. This
matters most when a variable's value comes from a function or method call, since the type is not otherwise visible at the
call site without navigating to the definition.

~~~typescript
// Not this:
const result = someModule.someFunction();

// This:
const result: SomeResult = someModule.someFunction();
~~~

Annotate a destructuring declaration on the complete pattern or, preferably, bind the named result to one annotated
variable. Reassignments do not repeat the annotation already present on the declaration. Loop bindings and callback
parameters whose types are fixed by their typed container or signature are exempt.

## 9. Return-value order matches the function name's order

When a function or method name enumerates multiple values, through "and" or by listing them, tuple values come back in
that same order.

~~~typescript
// Not this:
function sha256AndRead(localPath: string): [Buffer, string] {
    // ...
    return [data, digest];
}

// This:
function sha256AndRead(localPath: string): [string, Buffer] {
    // ...
    return [digest, data];
}
~~~

## 10. No vague object types for known shapes

Never type a parameter or return value as `object`, `any`, `Record<string, any>`, `Record<string, unknown>`, or an
array of those when the value has a fixed, known shape. Define a named `type` or `interface` capturing the real
properties and their value types. Use a class instead when the shape owns behavior, runtime validation, or mutable
lifecycle state.

~~~typescript
// Not this:
function beginTransaction(files: Array<Record<string, any>>): string {
    // ...
    return transactionId;
}

// This:
type DeclaredFile = {
    path: string;
    sha256: string;
    size: number;
};

function beginTransaction(files: DeclaredFile[]): string {
    // ...
    return transactionId;
}
~~~

When a wire format is position-based, such as `[size, sha256]`, parse it at the boundary into a named object before
passing it through application code. A labeled tuple is acceptable only when the positional representation itself is the
domain contract and callers do not need to reinterpret its positions.

## 11. Shared fields appear in the same order everywhere

When the same piece of data, such as a file's path, size, and hash, appears across multiple structures — a manifest entry,
a type, an interface, a class, or literal JSON construction — declare and construct the shared fields in the same order
everywhere. Pick one canonical order per project for each recurring shape and do not let definitions drift independently.

## 12. One file per domain class; its utilities live in that same file

A domain class — a class that is a primary, named concept in the codebase, such as `StoreClient` — gets its own file.
Use the project's kebab-case convention: `StoreClient` lives in `store-client.ts`.

Utility functions and small supporting classes that exist only to support that domain class and are not shared with
anything else live in the same file as the domain class, not in a separate utilities module.

~~~text
myapp/client/
  store-client.ts   # StoreError, local helpers, and class StoreClient
  cli.ts
~~~

Exception: types shared across independently deployable components go in a dedicated shared module, not duplicated in
each component. If a server and client are meant to stay independently deployable, the wire-format types they must agree
on live in a dependency-light `protocol.ts` or `types.ts` module that both import. That module holds only exchanged
data shapes, not either side's implementation.

Types that happen to look similar but are used by only one side stay with that side. Move a type into the shared module
only when both components genuinely use the same contract.

## 13. Class body layout: blank line after the declaration; static members before instance members

Leave exactly one empty line between the `class Foo {` line and the first member.

Within the class body, members appear in this order:

1. Static fields
2. Static methods
3. Instance fields
4. Constructor
5. Other instance methods, while preserving the helper-before-caller rule below

~~~typescript
export class StoreClient {

    private readonly baseUrl: string;
    private readonly token: string | null;

    // -----------------------------------------------------------------------------------------------------------------
    constructor(baseUrl: string, token: string | null = null) {
        this.baseUrl = baseUrl;
        this.token = token;
    }

    // -----------------------------------------------------------------------------------------------------------------
    private raiseForStatus(response: Response): void {
        // ...
    }

    // -----------------------------------------------------------------------------------------------------------------
    async close(): Promise<void> {
        // ...
    }
}
~~~

Data-only classes whose bodies are just field declarations may omit the blank line. The layout rules matter for behavior
classes, where the body has sections to organize.

## 14. A helper goes next to its near-exclusive caller

When a method exists mainly to be called by one other method in the same class, place its definition immediately before
that caller. Do not let unrelated methods sit between them.

If a caller uses several such helpers in sequence, order the helper definitions to match the call sequence, ending with
the caller itself.

~~~typescript
private async uploadFile(...): Promise<DeclaredFile> {
    // ...
}

async uploadFiles(...): Promise<DeclaredFile[]> {
    // calls uploadFile in a loop
}
~~~

## 15. A step of a caller-driven lifecycle never performs another step on its own

When a class exposes a multi-step lifecycle as separate public methods — for example,
`beginTransaction → uploadFiles → commit or abort` — no lifecycle method may silently perform a different step on the
caller's behalf. Each exposed step performs only its own step, so the caller controls when and whether to advance.

~~~typescript
// Not this: uploadFiles secretly owns the whole lifecycle.
async uploadFiles(files: Map<string, Buffer>): Promise<CommitResponse> {
    const transactionId: string = await this.beginTransaction(this.declareFiles(files));
    // uploads files, then silently commits or aborts
}

// This: uploadFiles performs only the upload step.
async uploadFiles(transactionId: string, files: Map<string, Buffer>): Promise<DeclaredFile[]> {
    const uploaded: DeclaredFile[] = [];
    // upload files without committing or aborting
    return uploaded;
}
~~~

This rule is about steps of a protocol explicitly exposed for the caller to drive. It is not a ban on one method calling
another. A private helper used by the one sanctioned public operation is fine.

If a one-call convenience operation that performs the whole lifecycle is genuinely wanted, give it a distinct name
rather than hiding that behavior inside a method whose name promises only one step.

## 16. Code comments

Comments explain what each section is doing, and why when that is not obvious from the code alone. Do not number the steps
with comments such as "1. first" and "2. then"; describe the behavior and reasoning in prose.

Prefer a single line per comment. Use multiple lines only when the explanation genuinely does not fit on one line.

Comments start lowercase, even full sentences. This applies to the start of a comment, not just continuation lines. The
structural hats from rule 1 are not prose comments.

~~~typescript
// ensure that the transaction id is valid
const transactionDirectory: string = requireTransactionDirectory(transactionId);

// only files in the original manifest can be uploaded
const declared: Map<string, DeclaredFile> = loadDeclared(transactionDirectory);
if (!declared.has(relativePath)) {
    throw new StoreError("path was not declared when the transaction was opened: " + relativePath);
}
~~~

## 17. Construct a class by calling its constructor, not with an object literal cast

When building a value whose type is a class, invoke it with `new` rather than creating an object literal and forcing the
type with `as`. Construction is where a class establishes invariants and runtime behavior.

~~~typescript
// Not this:
const declaredFile: DeclaredFileModel = {
    path,
    sha256,
    size,
} as DeclaredFileModel;

// This:
const declaredFile: DeclaredFileModel = new DeclaredFileModel(path, sha256, size);
~~~

This applies everywhere a class instance is constructed, including inside array operations.

A plain DTO represented by a TypeScript `type` or `interface` has no runtime constructor, so a correctly typed object
literal is appropriate for that case. Do not introduce a class solely to avoid an object literal.

## 18. Access a known shape's fields with dot notation, never string subscripting

When a value has a known class, type, or interface, read its properties with dot notation rather than
`value["property"]`.

~~~typescript
// Not this:
checkNoUnsafeCollision(relativePath, metadata["sha256"]);

// This:
checkNoUnsafeCollision(relativePath, metadata.sha256);
~~~

This does not apply to a genuine dynamic map or dictionary whose keys are runtime data. Looking up
`declared[relativePath]` or `declared.get(relativePath)` is ordinary keyed lookup; use dot notation on the known value
returned from that lookup.

## 19. No bare-declare-then-unpack; return a named result instead

Avoid APIs that return opaque tuples and force callers to declare variables separately before unpacking them. Return a
named result object so callers use one annotated variable and property access.

~~~typescript
// Not this:
function loadState(): [number, Map<string, SyncedFileState>] {
    // ...
}

let oldSequence: number;
let synced: Map<string, SyncedFileState>;
[oldSequence, synced] = loadState();

// This:
type SyncedState = {
    sequence: number;
    files: Map<string, SyncedFileState>;
};

function loadState(): SyncedState {
    // ...
}

const state: SyncedState = loadState();
~~~

When reaching for bare declaration followed by unpacking, treat it as a smell: the function should normally return a
named object.

## 20. No compound A-and-B condition leading a branch chain

When a branch chain's first condition is a conjunction, every later branch carries the rest of the truth table
implicitly. Nest instead, so every branch carries one condition.

~~~typescript
// Not this:
if (stagedAdded.has(relativePath) && !workspaceFiles.has(relativePath)) {
    stagedAdded.delete(relativePath);
} else if (!workspaceFiles.has(relativePath)) {
    throw new StoreError("not tracked: " + relativePath);
} else {
    stageRemoval(relativePath);
}

// This:
if (!workspaceFiles.has(relativePath)) {
    if (stagedAdded.has(relativePath)) {
        stagedAdded.delete(relativePath);
    } else {
        throw new StoreError("not tracked: " + relativePath);
    }
} else {
    stageRemoval(relativePath);
}
~~~

The same applies to sequential guard throws: `if (A && B) throw X` followed by `if (A) throw Y` is the same hidden
truth table. Nest both under one `if (A)`.

A compound condition in a single `if` with no chain is fine. There it is one logical test rather than a decision tree.

## 21. No pass-through wrapper methods

A method whose only job is to call another method with an identical signature is not an abstraction. It creates duplicate
maintenance and allows names and signatures to drift. If the wrapped method should be public, make it public; otherwise
call the private method directly. One of the pair must go.

~~~typescript
// Not this:
private trackedFiles(): Map<string, FileState> {
    // ...
}

public getTrackedFiles(): Map<string, FileState> {
    const files: Map<string, FileState> = this.trackedFiles();
    return files;
}

// This:
public trackedFiles(): Map<string, FileState> {
    // ...
}
~~~

A method earns its place when it adds validation, error translation, or coordination across collaborators. Forwarding
unchanged does not add anything.

## 22. Public methods are porcelain; keep machinery private

A class's public methods should be the operations the class exists for, plus read-only reports intended for consumers.
The bookkeeping structures and raw state loaders behind those operations are plumbing and stay private.

~~~typescript
// Not this: internals exposed because one caller wants their pieces.
public stagedManifest(): TransactionManifest {
    // ...
}

public trackedFiles(): Map<string, FileState> {
    // ...
}

// This: expose the report and keep its machinery private.
public status(): WorkspaceStatus {
    // ...
}

private stagedManifest(): TransactionManifest {
    // ...
}

private trackedFiles(): Map<string, FileState> {
    // ...
}
~~~

If internal state genuinely gains outside consumers, promote it then. Default to private. The public report should be the
only read most clients need.

## 23. Test the positive in an if/else

When an `if` has an `else`, phrase the test positively. `if (!A) foo; else bar` is `if (A) bar; else foo`, but the
second form is clearer because the affirmative case comes first and the reader does not carry a negation across the
branch.

~~~typescript
// Not this:
if (!tracked.has(relativePath)) {
    throw new StoreError("not tracked: " + relativePath);
} else {
    stageRemoval(relativePath);
}

// This:
if (tracked.has(relativePath)) {
    stageRemoval(relativePath);
} else {
    throw new StoreError("not tracked: " + relativePath);
}
~~~

This rule is about `if/else`. A guard throw with no `else`, such as `if (!valid) throw ...`, is fine because the
negation is the exceptional case. A single simple check likewise does not have to use positive phrasing. Tests such as
`value !== null` count as positive phrasing because they test that a value exists.

## 24. Cohesive stateful resources become a class

When a domain concept owns multiple related mutable resources that are initialized together, shared by other entities,
and torn down together, represent it as a class. Do not leave those resources as module-level globals accessed through
functions that pretend to be methods of an invisible object.

~~~typescript
// Not this: module globals masquerading as the server.
let repositoriesRoot: string | null = null;
let metadataStore: MetadataStore | null = null;
let objectStore: ObjectStore | null = null;
const repositories: Map<string, Repository> = new Map();

// This: the stateful concept is explicit.
export class Server {

    readonly metadataStore: MetadataStore;
    readonly objectStore: ObjectStore;
    private readonly repositories: Map<string, Repository>;

    // -----------------------------------------------------------------------------------------------------------------
    constructor(config: ServerConfig) {
        this.metadataStore = new SqlMetadataStore(config.databasePath);
        this.objectStore = new ObjectStore(config.objectStoreRoot);
        this.repositories = new Map();
    }

    // public operations and lifecycle methods
}
~~~

Signals that a concept wants to be a class:

1. Multiple pieces of mutable state travel together and require ordered initialization.
2. A lifecycle such as open, configure, use, close, or recover applies to the group.
3. Other domain objects consume shared resources and need a stable collaborator.
4. Implementations may be swapped behind a stable boundary.
5. Tests repeatedly reconfigure the same state bundle and require awkward global teardown.

Pure behavior with no state can stay as module functions. A single constant or configuration flag can stay as module
data. Once the concept becomes the thing that owns and manages related resources, make that concept explicit.

## 25. Constructor parameters and instance assignments share order

When constructor parameters become instance fields, assign them in the same order in which the parameters appear.
Derived fields stay grouped with the parameter from which they are derived.

~~~typescript
// Not this:
constructor(name: string, repositoryDirectory: string, store: MetadataStore, objectStore: ObjectStore) {
    this.name = name;
    this.repositoryDirectory = repositoryDirectory;
    this.objectStore = objectStore;
    this.metadataStore = store;
}

// This:
constructor(name: string, repositoryDirectory: string, store: MetadataStore, objectStore: ObjectStore) {
    this.name = name;
    this.repositoryDirectory = repositoryDirectory;
    this.metadataStore = store;
    this.objectStore = objectStore;
}
~~~

When dependencies force a different initialization order, keep direct parameter-to-field assignments in parameter order
and place derived fields with their source.

## 26. Pass the domain object, not its decomposed fields

When a method operates on a domain concept that already has a named class, pass the object itself rather than extracting
its fields and passing them as separate parameters. The caller already has the object; decomposing it at the API boundary
adds noise, permits parameter-order mistakes, and makes future additions harder.

~~~typescript
// Not this:
this.metadataStore.storeTransaction(
    transaction.id,
    transaction.added,
    transaction.removed,
    transaction.updated
);

// This:
this.metadataStore.storeTransaction(transaction);
~~~

Operations purely about identity, such as `deleteTransaction(transactionId)` or `getTransaction(transactionId)`, may
take only the identifier because the object does not yet exist or no longer exists. Once the callee needs the object's
state, pass the object.

## 27. A nested collection with a fixed meaning gets a named type

A flat array or map is a normal general-purpose container. Once the structure itself carries domain meaning, a bare nested
type such as `string[][]` or `Map<string, string[]>` becomes a smell. Every caller would otherwise have to reconstruct
what each layer means.

Wrap the structure in a named type or class whose name states the concept. If the concept owns matching, formatting, or
legacy-upgrade behavior, use a class and put those operations on it.

A direct alias of the bare nested collection, such as `type SubsetQuery = string[][]`, is not sufficient. Use a named
object shape that explains the levels or a class that owns the structure.

~~~typescript
// Not this: callers re-derive the OR-of-AND-groups meaning.
function loadQuery(): string[][] {
    // ...
}

// This:
export class SubsetQuery {

    // -----------------------------------------------------------------------------------------------------------------
    constructor(readonly groups: string[][]) {}

    // -----------------------------------------------------------------------------------------------------------------
    matches(relativePath: string): boolean {
        let result: boolean = false;
        // each group is ANDed; groups are ORed; empty means match everything
        return result;
    }
}
~~~

This rule applies to values crossing function or method boundaries, not necessarily to storage formats. On-disk JSON may
remain a nested array if convenient, as long as the boundary converts it to the named type used by the application.

## 28. Return the domain type a method's name promises; parse wire formats at the boundary

A method's return value is the concept its name says it produces, represented by a named type or class. Do not return a
transport representation that every caller must reinterpret.

XML nodes, raw JSON objects, database rows, and byte streams are parsed into domain values at the layer that receives
them and do not leak upward past that boundary. If every caller converts a method's return value before using it, the
method returned the wrong thing.

A cast such as `JSON.parse(raw) as SomeType` performs no runtime validation. Validate untrusted data at the boundary
with the project's schema validator or explicit property checks before treating it as a domain value.

~~~typescript
// Not this: an XML node escapes and the caller must reinterpret it.
const result: XmlResult = await client.login();
const summary: EntitlementSummary = entitlementSummary(result.root);

// This: login returns what login means.
const result: LoginResult = await client.login();
if (result.summary.login === "1") {
    // ...
}
~~~

## 29. Feature ownership and separate tests

Place application code with the feature that owns it, following
`docs/development/project-structure.md`. Shared technical mechanisms belong in
`src/infrastructure/`; server configuration and dependency wiring belong in `src/app/`.

All tests, test fixtures and test helpers belong under the project-root `tests/` directory.
Never colocate them with application code in `src/` or deployable assets in `public/`.
Server tests mirror feature paths under `tests/server/`; browser tests use `tests/browser/`;
shared test helpers use `tests/support/`; standalone SQL tests use `tests/database/`.

## 30. No inline import types

Never use an inline import type such as `import("fs").Dirent` in an annotation. Every type a
file needs comes in through an explicit import at the top of the file, so the imports section
is the complete statement of the file's dependencies.

When the value import and the type live in different entry points of the same package, such as
`fs/promises` for the functions and `fs` for the types, add a type-only import next to the
value import. A type-only import is erased at compile time and adds no runtime cost.

~~~typescript
// Not this:
async entries(): Promise<import("fs").Dirent[]> {
    // ...
}

// This:
import fs from "fs/promises";
import type {Dirent} from "fs";

async entries(): Promise<Dirent[]> {
    // ...
}
~~~

## 31. Every branch gets braces

Every `if`, `else if`, and `else` body is wrapped in `{ }`, even when the body is a single
statement. Braceless one-line bodies such as `if (!valid) throw new Error("bad");` are not
allowed: they are easy to misread, and adding a second statement to the branch later silently
changes what the condition covers.

~~~typescript
// Not this:
if (!directory) throw new Error("invalid_path");
return new DirectoryStore(directory);

// This:
if (!directory) {
    throw new Error("invalid_path");
}
return new DirectoryStore(directory);
~~~

## 32. Zod object schemas are one field per line

Every zod object schema literal — `z.object({ ... })`, `z.looseObject({ ... })`, `z.strictObject({ ... })`, and the
shape literal passed to `.extend({ ... })` — puts each field on its own line, with the closing brace on a line of
its own. Fields packed onto one line are hard to scan and produce noisy diffs when a field is added or changed.

~~~typescript
// Not this:
const options = z.object({ kind: z.enum(["release", "snapshot"]).default("release"), githubRepo: z.string().url().optional() });

// This:
const options = z.object({
    kind: z.enum(["release", "snapshot"]).default("release"),
    githubCommit: z.string().regex(/^[a-fA-F0-9]{7,40}$/).optional(),
    githubRepo: z.string().url().optional()
});
~~~
