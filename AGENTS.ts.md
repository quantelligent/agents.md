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
