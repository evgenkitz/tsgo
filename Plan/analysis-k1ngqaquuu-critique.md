# Critical Analysis: Non-Invasive ArkTS on TS-GO

> Analysis of `analysis-k1ngqaquuu-internals-en.md` against real tsgo source code.
> **Context:** The approach is a **fork** of `microsoft/typescript-go` with all new code in `internal/arkts/`.
> Goal: zero modifications to existing tsgo files → clean `git merge upstream/main`.

---

## What Actually Works (in the Fork Model)

The approach is a fork — `internal/arkts/` lives in the same Go module, so it **can** import `internal/compiler`, `internal/checker`, etc. The criticism that «there is no public API» is less relevant for a fork.

These specific APIs **do exist** and are usable:

| Document Claim | Actual API | Status |
|---|---|---|
| Wrapping VFS | `vfs.FS` interface (11 methods) | ✅ Exists, but 11 methods, not 4 |
| `ast.SourceFile`, `ast.Node` | `internal/ast` | ✅ Exists |
| `checker.Checker` queries | `c.GetTypeAtLocation()`, `c.GetSymbolAtLocation()`, etc. | ✅ Exists |
| `compiler.NewProgram()` | `compiler.NewProgram(opts ProgramOptions) *Program` | ✅ Exists |
| `printer.Emit` (not `EmitNode`) | `p.Emit(node, sourceFile) string` | ✅ Exists, different signature |
| `program.GetSourceFiles()` | `*Program.GetSourceFiles() []*ast.SourceFile` | ✅ Exists |
| `program.GetSemanticDiagnostics()` | `(ctx, sourceFile)` — requires context + file | ✅ Exists, more params |
| `program.GetTypeChecker()` | `(ctx) (*checker.Checker, func())` | ✅ Exists |
| Language service | `*project.Session.GetLanguageService(ctx, uri)` | ✅ Exists, but needs Session |

---

## 🔴 Critical Problems

### 1. No Stable API — Silent Breakage on Upmerge

This is the **central risk of the fork approach**. The document says:

> `git pull upstream/main # 0 conflicts. Always.`
> `go build ./cmd/arktsc # Still builds.`

**Merge conflicts** and **build breakage** are different things. Zero merge conflicts on existing files is achievable — all new code is in `internal/arkts/`. But `go build` can break silently because:

- All tsgo packages are `internal/` — **zero stability guarantees**
- Upstream can rename/remove/refactor any exported symbol in `internal/compiler`, `internal/checker`, `internal/ast`, `internal/vfs`, etc.
- No deprecation notices, no migration guides, no changelog for internal APIs
- `ProgramOptions` fields can change, `CompilerHost` methods can be added, `Checker` method signatures can shift

**Example:** If upstream adds a new method to `vfs.FS` interface (it already has 11), your `WrappingFS` breaks at compile time. If upstream changes `ProgramOptions.Host` from `CompilerHost` to a new interface, every call to `NewProgram` breaks.

The document counts «merge conflicts on files we don't touch» as zero risk. But a Go interface change in `internal/vfs/vfs.go` breaks `internal/arkts/vfs/wrapper.go` without touching it — and the fix is manual, not mergeable.

### 2. `program.GetLanguageService()` — Still Doesn't Exist

The LSP sections (§21, §22) are built on:

```go
tsGoLocs := program.GetLanguageService().FindAllReferences(name)
```

**Reality:** `*compiler.Program` has no `GetLanguageService()` method. None. Verified against `program.go` (all 70+ exported methods, lines 127–1988).

The language service lives in `*project.Session`:

```go
// project/session.go:983
func (s *Session) GetLanguageService(ctx context.Context, uri lsproto.DocumentUri) (*ls.LanguageService, error)
```

To get a `*project.Session`, you need the full LSP infrastructure: `lsp.Server`, `ServerOptions`, `requestQueue`, `outgoingQueue`, watchers, `InitializeParams`, etc. This is not a «thin mapping layer» — it's running a full LSP server.

**Impact:** The entire completion architecture (§22), Find All References (§21.3), and 12 LSP operations cannot work through a «forward mapping → TS-GO → reverse mapping» pattern without deep integration into the LSP server lifecycle.

**Possible workaround:** Create a `*ls.LanguageService` directly with `ls.NewLanguageService()`, which requires implementing `ls.Host` (11 methods). But then `ls.Host.ReadFile` must return desugared content, `Host.Converters()` needs position mapping, and you're reimplementing what `project.Session` already does — hardly "500 lines."

### 3. Mini-Lexer: Viable Approach, Realistic Complexity ~500 Lines

**Correction from initial review:** The mini-lexer does NOT need to be a full TypeScript scanner. It only needs to find 9 specific patterns and transform them. Comparing it to `internal/scanner/` is wrong — that scanner must tokenize every byte of every possible TS construct. The mini-lexer only needs to track enough state to avoid false matches and to locate the specific patterns listed in the document.

The 5-state automaton (`normal, string, comment, template, regex`) is the right approach — it prevents matching keywords inside strings/comments/templates. The actual transforms are localized:

| Transform | What the lexer does | Complexity |
|---|---|---|
| `struct → class extends __S__` | Match `struct` keyword at declaration position | Low — `struct` is always a keyword in valid ArkTS; `const struct = 1` is already invalid |
| `import lazy → import` | Match `import` then `lazy` | Trivial — unambiguous token sequence |
| `@Deco function → function` | Match `@` + identifier + optional `(...)` + `function` | Low — balanced paren counting |
| Trailing block `() { } → (() => { })` | Match `Ident(...) {` without `if/for/while` before, no `;`/`=>` between | Medium — needs bracket stack |
| `@Styles` body `.x()` → `__s__.x()` | In `@Styles` context, insert prefix | Low — context from DesugarState |
| `@interface → declare function` | Match `@interface` keyword at declaration position | Low — unambiguous |
| Nominal brand injection | Insert private field at class body start | Trivial — position arithmetic |
| `@Builder` body | Trailing block DSL enabled by decorator context | Low — context from DesugarState |
| `.d.ets` path remapping | Extension swap in VFS, not lexer | Trivial |

**Realistic assessment:** ~500 lines (not 200, not 2000). The core state machine is compact; the bulk of code goes into balanced bracket tracking for trailing block detection and edge case handling.

**Valid concern that remains — trailing block edge cases:**

```typescript
Column()         // ASI inserts ; here in standard TS
{ const x = 1; } // ← separate block statement in TS, trailing block in ArkTS intent
```

In ArkTS, `{` on the next line after a component call is almost certainly a trailing block, not a block statement. The distinction relies on the fact that ArkTS UI code has a constrained grammar inside `build()`/`@Builder` — standalone block statements are prohibited there (enforced by the overlay `UISyntaxOnlyRule`). So treating newline-`{` as trailing block is correct for valid ArkTS, and for invalid code the TS-GO checker will catch the resulting type error.

**Valid concern — template literal depth:**

`` `a ${foo(`b ${c} d`)} e` `` requires counting `${`/`}` depth. A stack-based counter handles this correctly; the logic is straightforward but must be implemented.

**Verdict:** The mini-lexer approach is sound. The document underestimates lines (~200 → ~500), but the architecture is correct: targeted pattern matching with a state machine is exactly the right tool for this job, not a full parser.

### 4. `vfs.FS` — 11 Methods, Not 4

The document only mentions `FileExists, ReadFile, DirectoryExists, WalkDir`. The actual interface:

```go
// internal/vfs/vfs.go:12-50
type FS interface {
    UseCaseSensitiveFileNames() bool
    FileExists(path string) bool
    ReadFile(path string) (contents string, ok bool)
    WriteFile(path string, data string) error        // ← must map .ets.ts → .ets
    AppendFile(path string, data string) error        // ← must map .ets.ts → .ets
    Remove(path string) error                         // ← must map .ets.ts → .ets
    Chtimes(path string, aTime, mTime time.Time) error // ← must map
    DirectoryExists(path string) bool                 // ← must map
    GetAccessibleEntries(path string) Entries         // ← must list virtual .ts AND real .ets
    Stat(path string) FileInfo                        // ← must map
    WalkDir(root string, walkFn WalkDirFunc) error    // ← must map callback paths
    Realpath(path string) string                      // ← must reverse-map
}
```

Every method that takes or returns a path must handle the `.ets` ↔ `.ets.ts` mapping. `GetAccessibleEntries` for directory `/src/` must return BOTH virtual names (`foo.ets.ts`) AND filter/map to real names (`foo.ets`). `Stat` on `foo.ets.ts` must return the real file's info but with the virtual name.

The mapping is not a simple 1:1 transform — it requires a bidirectional path registry maintained in sync with the filesystem.

---

## 🟡 Serious Problems

### 5. Segment Mapping × 9 Transforms = Combinatorial Debugging Hell

Volar does **1** transform (template → virtual JSX). ArkTS does **9**:

```
struct→class, @interface→declare, import lazy removal, function decorator removal,
trailing block→arrow, @Styles body, nominal brand injection, @Builder body,
.d.ets→.d.ts mapping
```

Each transform adds segments. After N transforms, the segment list is at least 2N+1 entries per modified region. A diagnostic at virtual offset V must walk the segment list to find the original .ets offset. If **any** segment has an off-by-one error, the diagnostic appears at the wrong location.

Debugging a «wrong line» bug means tracing through 9 transform passes for that specific byte range. The document provides no tooling for this.

### 6. Trailing Block Detection: Solvable with Bracket Stack

The bracket type stack (section 13.2) is not a «recursive descent parser in disguise» — it's a bracket-depth tracker with keyword discrimination. The distinction matters:

- The lexer does NOT need to understand TypeScript expression grammar
- It only needs to: (a) distinguish `Ident(` from `if/for/while/switch(`, and (b) track balanced brackets

The cases listed are all resolvable without a full parser:

| Case | How the lexer handles it |
|---|---|
| `Ident(...)` vs `if (...)` | One-token lookahead — check if token before `(` is a keyword |
| `else { }` | `else` after `if`-block close — extends keyword-block, not a new trailing block |
| `ForEach(arr, (item) => { })` | Arrow `=>` inside — bracket stack sees `{` and `=>` before it → arrow body, not trailing block |
| `Column() { }.width(100)` | `}` at depth-1 after trailing block → pop trailing state. `.width` after `}` — chain continuation |
| `Column().width(100) { }` | Block AFTER chain → the lexer detects this as invalid; overlay `UISyntaxOnlyRule` reports it |

The bracket type stack adds a type tag per depth level (`trailing` vs `keyword`). When `}` closes depth N, it checks the tag to determine what to emit (`)` for trailing, nothing for keyword). This is a well-scoped algorithm, not a parser.

**Remaining concern:** `Column()\n{ }` with newline between `)` and `{`. In standard TS, ASI inserts `;` making `{ }` a block statement. In ArkTS, standalone block statements are forbidden in UI context (enforced by overlay), so treating it as trailing block is the correct behavior for all valid ArkTS code. For invalid code, the TS-GO checker catches the resulting type error.

### 7. Function Decorator Removal — Balanced Paren Scanning

```typescript
@Sendable(scope)
@AnotherDeco(arg1, arg2)
function f() { }
```

The operation: detect `@` in `normal` state → read identifier → if `(` follows, scan balanced parens → repeat for chained decorators → verify `function` follows → remove span.

This is straightforward balanced-delimiter scanning. The complexity is in tracking `(`, `)`, `{`, `}`, `[`, `]` depth, which the lexer already does for trailing blocks. Decorator arguments like `@Deco({x: 1, y: foo(a, b)})` are handled correctly by counting `{`/`}` and `(`/`)` depth — the removal span ends when all brackets return to the pre-decorator level.

**Corner case that IS valid concern:** `@` can appear in JSDoc `/** @param */`, in arrow `x => @dec f()`, etc. But the lexer's `normal` state + lookahead for `function/class/field` after the decorator resolves this — JSDoc is in `comment` state, arrows are identified by `=>` before `@`.

**Critical mapping gap — decorator arguments are invisible to TS-GO:**

Consider a decorator with non-trivial arguments:

```typescript
@Sendable(computeScope(someVar, anotherVar))
function f() { }
```

The desugar removes the entire decorator line. In the virtual `.ts`, `computeScope`, `someVar`, and `anotherVar` **do not exist**. This has cascading consequences for ALL position-dependent LSP operations:

| Operation | Problem |
|---|---|
| **Go To Definition** on `computeScope` | Forward mapping `.ets` → virtual `.ts` returns `ok=false` — position is inside removed span. TS-GO cannot resolve it. |
| **Hover** on `computeScope` | Same — no virtual `.ts` position, no type info from TS-GO. |
| **Find All References** on `computeScope` | TS-GO doesn't see this usage. Document proposes `DesugarState.DecoratorUsages` to manually merge removed locations — but constructing a `Location` for a *removed* span requires reverse-mapping from nothing. |
| **Rename** on `computeScope` | TS-GO renames all occurrences it can see → the decorator argument is NOT among them. The rename misses this occurrence silently. |
| **Signature Help** inside decorator args | `@Sendable(comp█)` — forward mapping fails, no signature help. |
| **Code Actions** | Any fix that touches the decorator argument can't be mapped through TS-GO. |

The segment mapping works for **inserted** text (synthesized segments → `MapToETS` returns `ok=false` → diagnostic is dropped). It works for **verbatim** text (position preserved modulo offset). But it has **no mechanism for removed text** — the virtual `.ts` has no position corresponding to the removed span. Forward mapping (`.ets` → `.ts`) for a position inside removed text has no target.

The document acknowledges this for Find All References (§21.3) via `DesugarState.DecoratorUsages`:

```go
for _, usage := range state.DecoratorUsages {
    if usage.Decorator == name {
        allLocs = append(allLocs, ...)
    }
}
```

But `DecoratorUsages` stores decorator *names*, not references inside decorator *arguments*. `computeScope` and `someVar` are references within the argument expression — they need their own tracking entries. And for operations other than Find All References (hover, go-to-def, rename, signature help), the document provides **no mechanism at all**.

**What this means in practice:** If a developer hovers over `computeScope` in `@Sendable(computeScope(...))`, the LSP has two choices:
1. Return nothing (forward mapping fails)
2. Implement an independent service that resolves `computeScope` without TS-GO

Option 2 requires the overlay/lowering layer to have its own symbol resolution for expressions that only exist in `.ets`. For complex expressions, this replicates binder/checker functionality — exactly what the approach claims to avoid.

### 8. TS2339 Cascade Suppression Can Suppress Real Errors

Section 15.6: `.cardStyle()` on `Column` → TS2339 (method belongs to struct, not Column). Overlay filter suppresses this.

But how to distinguish from a **genuine** TS2339 where the method truly doesn't exist?

```typescript
struct MyComponent {
  @Styles cardStyle() { .width(100) }
  build() {
    Column()
      .cardStyle()    // ← should be OK (on struct via @Styles)
      .nonexistent()  // ← should be ERROR (method doesn't exist at all)
  }
}
```

Both produce TS2339 from the checker. To filter correctly, the overlay must:
1. Identify current UI context (inside `build()` of a struct)
2. Resolve the owning struct
3. Check if `cardStyle` exists on it as `@Styles`
4. Check if `nonexistent` exists on it at all

This is non-trivial type-level logic, not a «~5-line filter.»

### 9. Nominal Typing via Brand Field — Known Hole

The document admits (section 17.2): sibling classes with same brand are silently compatible.

```typescript
class A { private __arkts_nominal?: never; }
class B extends A {}
class C extends A {}
const c: C = new B();  // TS-GO silent — both inherit A's brand
```

The mitigation is `NominalSubclassRule` — an overlay rule that must check **all pairs** of sibling classes. For a class hierarchy of depth D with branching factor B, this is O(B² × D) checks. For a real ArkUI app with 100+ component classes, this is thousands of pairwise comparisons.

---

## 🟢 Moderate Problems

### 10. Kit Module Resolution — No Extension Point

`@kit.ArkUI` → JSON config lookup → real SDK path. This is custom module resolution, but TS-GO's module resolution is in `internal/module/`. There is **no hook** for custom resolution prefixes. The Wrapping VFS only intercepts file operations, not the resolution algorithm.

**Workaround:** Pre-resolve `@kit/*` paths and inject them as path mappings in `tsconfig.json`. This works but is fragile — it depends on `tsconfig.json` being generated/patched correctly.

### 11. Error Recovery Chain of 5 Steps

```
.ets source
  → mini-lexer (step 1) — if bug here
    → desugared .ts (step 2) — garbage in
      → TS-GO parser (step 3) — parse error
        → TS-GO diagnostic (step 4) — virtual .ts position
          → Segment mapping (step 5) — .ets position
            → USER SEES GARBAGE
```

A bug at step 1 propagates through 4 more steps. The user sees a diagnostic that:
- May point to the wrong line
- May have an incomprehensible message (parser error on desugared code)
- Cannot be traced back to the original construct without deep knowledge of the desugar pipeline

### 12. Source File Identity — Two Competing Views

TS-GO thinks it's compiling `foo.ets.ts`. The user thinks they're editing `foo.ets`. Every diagnostic, every hover, every completion must translate between these identities. The `SourceFile.FileName()` in the AST says `foo.ets.ts`, but LSP URIs from the editor say `file:///src/foo.ets`.

The `WrappingFS` must maintain a bidirectional registry:
- `"foo.ets"` → `"foo.ets.ts"` (for ReadFile)
- `"foo.ets.ts"` → `"foo.ets"` (for diagnostics, LSP responses)
- `"foo.ets.ts"` → original mtime of `"foo.ets"` (for cache invalidation)

This is implementable but error-prone at scale (10K+ files).

### 13. Incremental Compilation with 10K Files

The document claims 5-10% overhead. For cold start with 10K files:
1. Read each `.ets` → desugar → cache virtual `.ts` (50MB memory per doc)
2. TS-GO parses virtual `.ts` (10K parse operations)
3. TS-GO binds and checks (full program)
4. Overlay pass walks each AST (10K walks)
5. ArkUI lowering transforms UI methods

Any change to one `.ets`:
- Invalidate cache entry
- Re-desugar
- TS-GO re-parses one file (incremental)
- TS-GO re-checks affected files
- Re-run overlay on affected files
- Re-lower affected methods

The overhead percentage depends heavily on the ratio of ArkTS-specific work to TS-GO work. For UI-heavy files (lots of trailing blocks, decorators, build methods), the ArkTS overhead could dominate.

---

## Not Risks (Correctly Identified)

| Claim | Verdict |
|---|---|
| Merge conflicts on upstream files | ✅ Zero — no modifications to existing files |
| Performance is not a differentiator | ✅ Agreed — overhead is manageable |
| Type information is available | ✅ Through `checker.Checker` on checked program |
| 160+ overlay rules are mechanically portable | ⚠️ Porting is mechanical, but **validating** correctness requires full test suite |

---

## What the Document Gets Right

1. **Wrapping VFS is the right seam** — `vfs.FS` is the correct abstraction for intercepting file reads and presenting desugared content. The fork model makes this viable.

2. **AST-level overlay pass works** — reading AST nodes and checking `Kind`, `Flags`, decorators is straightforward and doesn't require modifying TS-GO.

3. **Checker queries work** — `c.GetTypeAtLocation(node)` and `c.GetSymbolAtLocation(node)` provide type information needed for type-dependent overlay rules.

4. **ArkUI lowering via AST transforms works** — generating standard TS-GO AST nodes from typed AST and printing them with the standard printer is the correct approach for codegen.

5. **Post-emit text processing is viable** — module specifier normalization and lazy import restoration are text-level operations on already-emitted JS, which is lower risk than pre-parse desugaring.

---

## Summary: What Would Break First

If this were built and run against real ArkTS codebases:

1. **Week 1:** Mini-lexer bugs on edge cases (unicode, nested templates, regex ambiguity, `struct` as identifier)
2. **Month 1:** Upstream tsgo changes a method signature in `internal/checker` or `internal/compiler` → build breaks, manual fix required
3. **Month 3:** Subtle segment mapping bugs → «wrong line» diagnostics → user trust erodes
4. **Month 6:** LSP completions not working for complex cases because the mapping layer can't correctly feed desugared positions to `LanguageService`
5. **Ongoing:** Each upstream merge requires auditing all used internal APIs for breakage

The approach is **not impossible** — it's a valid architectural strategy for a fork. But the document understates the complexity of:
- The mini-lexer (~200 lines → more realistically ~500 for correctness with edge cases)
- The segment mapping (trivial for 1 transform, combinatorial debugging for 9)
- The LSP integration (not «500 lines» when you need `project.Session`)
- The upstream API stability risk (zero merge conflicts ≠ zero breakage)
- The `vfs.FS` wrapping (all 11 methods, not 4, each with bidirectional path mapping)
