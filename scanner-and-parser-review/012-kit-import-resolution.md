# Plan 012: Kit Import Resolution

## Context

ArkTS supports `@kit.X` symbolic imports that resolve at parse time to real file-path
imports via SDK JSON configuration files. The parser transforms these before type checking:

```
Input:   import { Button } from '@kit.ArkUI';
Output:  import { Button } from 'ohos3.2/arkui/component';
```

The reference implementation in ohos-typescript lives in:
- `ohApi.ts:1233-1694` — constants, types, JSON loading, node creation, processing
- `parser.ts:1835-1848` — integration into `parseSourceFileWorker`
- `types.ts:827` — `NodeFlags.KitImportFlags = 1 << 29`
- `types.ts:829` — `NodeFlags.NoOriginalText = 1 << 31`
- `types.ts:4221` — `SourceFile.markedKitImportRange`

This plan covers the core transformation pipeline. The white-list module attribute
supplementation (`extendComponentWhiteList`, `processApiNamedBindingsStatement`, etc.)
is a follow-up concern.

## Sub-tasks

### 1. NodeFlags additions (`internal/ast/nodeflags.go`)

Two new flags:

| Flag | Bit | Meaning |
|------|-----|---------|
| `NodeFlagsKitImportFlags` | 29 | Node was in a converted kit-import statement |
| `NodeFlagsNoOriginalText` | 31 | Node has no original text in source file |

Bit 29 is currently "reserved for upstream additions" comment. Bit 31 is free.

```go
NodeFlagsKitImportFlags NodeFlags = 1 << 29 // If node was in a converted kit-import statement
NodeFlagsNoOriginalText NodeFlags = 1 << 31 // If node has no original text in source file
```

**Regenerate:** `npx hereby generate:enums` → updates `_packages/native-preview/src/enums/nodeFlags.*.ts`

**Files changed:**
- `internal/ast/nodeflags.go` (+2, -1 comment)

---

### 2. SourceFile: MarkedKitImportRange (`internal/ast/ast.go`)

Add field after parser-set fields (~line 2456, after `CommonJSModuleIndicator`):

```go
MarkedKitImportRange []core.TextRange // Ranges of statements transformed by kit import
```

Stores ranges of original `@kit.X` statements replaced during transformation. Consumed by
`isInMarkedKitImport()` for downstream position queries.

**Files changed:**
- `internal/ast/ast.go` (+1)

---

### 3. CompilerOptions: NoTransformedKitInParser (`internal/core/compileroptions.go`)

Add near `EtsLoaderPath` (line 162):

```go
NoTransformedKitInParser bool `json:"noTransformedKitInParser,omitzero"`
```

Disables kit import transformation in parser when set.

**Files changed:**
- `internal/core/compileroptions.go` (+1)

---

### 4. SourceFileParseOptions: threading options (`internal/ast/parseoptions.go`)

Add to struct:

```go
EtsLoaderPath            string // SDK path for kit JSON config resolution  
NoTransformedKitInParser bool   // Skip kit import transformation
```

Pattern: identical to existing `EtsAnnotationsEnable` / `EtsCustomComponent`.

Wire in `internal/compiler/fileloader.go` (line ~370):

```go
EtsLoaderPath:            options.EtsLoaderPath,
NoTransformedKitInParser: options.NoTransformedKitInParser,
```

**Files changed:**
- `internal/ast/parseoptions.go` (+2)
- `internal/compiler/fileloader.go` (+2)

---

### 5. Core logic: `internal/parser/arkts_kit.go` (new file)

All kit transformation functions. ~250 lines. Organized into sections:

#### 5a. Constants, types, cache

```go
const kitPrefix = "@kit."
const defaultKeyword = "default"

type kitSymbolInfo struct {
    Source   string `json:"source"`
    Bindings string `json:"bindings"`
}

type kitJsonInfo struct {
    Symbols map[string]kitSymbolInfo `json:"symbols"`
}

var kitJsonCache sync.Map // map[string]*kitJsonInfo
```

**Design:** `sync.Map` for concurrent-safe caching. Parse can be called in parallel
goroutines during multi-threaded compilation.

#### 5b. SDK path resolution

```go
func getSdkPath(opts ast.SourceFileParseOptions) string
func isMixedCompilerSDKPath(etsLoaderPath string) bool
```

| Function | Logic |
|----------|-------|
| `getSdkPath` | If mixed: `ResolvePath(etsLoaderPath, "../../../../..")` (5 levels up). Otherwise: `ResolvePath(etsLoaderPath, "../../../..")` (4 levels up). Returns empty if `EtsLoaderPath` is empty. |
| `isMixedCompilerSDKPath` | `NormalizePath(etsLoaderPath)` ends with `"dynamic/build-tools/ets-loader"` |

#### 5c. JSON config loading

```go
func getKitJsonObject(name string, sdkPath string, opts ast.SourceFileParseOptions) *kitJsonInfo
func cleanKitJsonCache()
```

- Looks up in `kitJsonCache` first (cached lookups & negative caching).
- On miss: reads `{sdkPath}/openharmony/ets/{dynamic/}build-tools/ets-loader/kit_configs/{name}.json`
- Falls back to `{sdkPath}/hms/ets/{dynamic/}build-tools/ets-loader/kit_configs/{name}.json`
- Mixed compiler (1.2+) inserts `dynamic/` segment in path.
- Uses `os.ReadFile` directly (SDK configs are on the real filesystem, not project VFS).
- Uses `internal/json` for Unmarshal.
- `cleanKitJsonCache` replaces with fresh `sync.Map{}`.

#### 5d. Statement filtering

```go
func excludeStatementForKitImport(statement *ast.Node) bool
```

Returns `true` (exclude from transformation) when:
- Not an `ImportDeclaration`, or has no `ImportClause`
- Is a namespace import (`import * as X from ...`)
- Module specifier is not a `StringLiteral`
- Has decorators (indicates parse error)
- Module specifier text doesn't start with `@kit.`
- Has modifiers or attributes (assert clause)

#### 5e. Virtual/synthetic nodes: reference tsc vs tsgo

Reference tsc has `node.virtual = true` — a dedicated boolean field. tsgo has **no such
field**. Instead, synthetic nodes are determined by position:

```go
// internal/ast/utilities.go:76-87
func NodeIsSynthesized(node *Node) bool {
    return PositionIsSynthesized(node.Loc.Pos()) || PositionIsSynthesized(node.Loc.End())
}
func PositionIsSynthesized(pos int) bool {
    return pos < 0  // negative position = synthetic node
}
```

`NodeFlagsReparsed` (bit 3, "synthesized during parsing") is a separate marker checked
by functions like `FindLastVisibleNode` to skip reparsed nodes — it does NOT make
`NodeIsSynthesized` return true.

**Implication for kit import nodes:** Kit import nodes must have **positive positions**
(copied from original statements) so error messages point to the right source
location. They get `NodeFlagsKitImportFlags` as the sole distinguishing marker.
`NodeFlagsReparsed` is NOT set — otherwise `FindLastVisibleNode` would skip them.

This matches the reference tsc pattern in `utilities.ts:1893`:
```ts
(node.virtual && !(node.flags & NodeFlags.KitImportFlags))
// → kit import nodes (virtual + KitImportFlags) are treated as "not truly virtual"
```
In tsgo, the equivalent is simply: kit nodes have positive positions (not
synthesized) + `NodeFlagsKitImportFlags` for downstream identification.

#### 5f. Node creation

```go
func createImportDeclarationForKit(
    factory *ast.NodeFactory,
    isType bool,
    name *ast.Node,
    symbol *kitSymbolInfo,
    oldStatement *ast.ImportDeclaration,
    importSpecifier *ast.Node,
) *ast.Node
```

Markings on new nodes:

| Nodes | Flags |
|-------|-------|
| All new nodes (ImportDeclaration, ImportClause, NamedImports, ImportSpecifier, StringLiteral, propertyName Identifier) | `NodeFlagsKitImportFlags` |
| `StringLiteral` (module specifier) | `NodeFlagsKitImportFlags \| NodeFlagsNoOriginalText` |
| `Identifier` (property name, if created) | `NodeFlagsKitImportFlags \| NodeFlagsNoOriginalText` |

`NodeFlagsNoOriginalText` on the module specifier and property name prevents the emitter
from reading the source file text at those positions (which would yield `@kit.ArkUI` or
an unrelated identifier instead of the resolved path / binding name).

Position assignment: each new node gets the **positive** text range of the corresponding
node in the original statement:

| New node | Position source |
|----------|----------------|
| `ImportDeclaration` | `oldStatement.Pos()..oldStatement.End()` |
| `ImportClause` | `importSpecifier.Pos()..importSpecifier.End()` |
| `NamedImports` | `importSpecifier.Pos()..importSpecifier.End()` |
| `ImportSpecifier` | `importSpecifier.Pos()..importSpecifier.End()` |
| `StringLiteral` (module specifier) | `oldModuleSpecifier.Pos()..oldModuleSpecifier.End()` |
| `Identifier` (property name) | `name.Pos()..name.End()` |

Factory method details (from `ast_generated.go`):
- `NewImportClause(phaseModifier ImportPhaseModifierSyntaxKind, name *IdentifierNode, namedBindings *NamedImportBindings) *Node`
- `NewImportSpecifier(isTypeOnly bool, propertyName *ModuleExportName, name *IdentifierNode) *Node`
- `NewNamedImports(elements *ImportSpecifierList) *Node`
- `NewImportDeclaration(modifiers *ModifierList, importClause *ImportClauseNode, moduleSpecifier *Expression, attributes *ImportAttributesNode) *Node`
- `NewStringLiteral(text string, tokenFlags TokenFlags) *Node` — tokenFlags = 0 for synthesized nodes

Note: `isLazy` propagation is omitted since `ImportClause.IsLazy` does not yet exist in
the Go codebase (it's an ArkTS `lazy` keyword feature). A follow-up task will add it.

#### 5g. Statement processing

```go
func processKitStatement(factory *ast.NodeFactory, statement *ast.Node, jsonInfo *kitJsonInfo) []*ast.Node
```

- For default import (`import X from '@kit.Y'`): looks up `"default"` key in kit JSON, creates one `ImportDeclaration`.
- For named imports (`import { A, B } from '@kit.Y'`): iterates each `ImportSpecifier`, looks up by import name (or property name if aliased), creates one `ImportDeclaration` per symbol.
- Returns `nil` on any failure (original statement kept as-is).
- Skips white-list, ETS-context, and attribute supplementation checks (follow-up task).

#### 5h. Main orchestrator

```go
func processKit(
    factory *ast.NodeFactory,
    statements []*ast.Node,
    sdkPath string,
    markedRanges *[]core.TextRange,
    opts ast.SourceFileParseOptions,
) []*ast.Node
```

Iterates statements, applies `excludeStatementForKitImport`, loads JSON for matching
statements, calls `processKitStatement`, records ranges of transformed statements.

**Decision:** Skip `inEtsContext` check — in the reference, it's always true when parsing
`.ets` files. The ArkTS rule "imports before non-imports" (skipRestStatements) is also
omitted for now — it relies on `inEtsContext`. Both can be added when the ETS context
gating is needed.

#### 5i. Range queries

```go
func IsInMarkedKitImport(sourceFile *ast.SourceFile, pos, end int) bool
```

Exported — consumed by other packages for diagnostic position correction. Checks if
`[pos, end)` falls within any marked kit import range.

---

### 6. Parser integration (`internal/parser/parser.go`)

**Parser struct** — add field (after `etsFlags`, line ~106):

```go
markedKitImportRanges []core.TextRange // accumulated during processKit
```

**in `parseSourceFileWorker`** — after statement parsing, before `finishNode`:

```go
// --- Kit import transformation ---
sdkPath := getSdkPath(p.opts)
skipKit := p.opts.NoTransformedKitInParser || sdkPath == "" || p.hasParseError
if !skipKit {
    var markedRanges []core.TextRange
    statements = processKit(&p.factory, statements, sdkPath, &markedRanges, p.opts)
    p.markedKitImportRanges = markedRanges
}
```

Skip conditions: `NoTransformedKitInParser`, no SDK path, or parse errors — matching
the reference tsc. The `languageVersionCallBack` is omitted (not needed in this port).

**in `finishSourceFile`** — after existing assignments (~line 493):

```go
result.MarkedKitImportRange = slices.Clone(p.markedKitImportRanges)
```

`p.markedKitImportRanges` auto-clears when the pooled parser is reused (zeroed in `putParser`).
`slices.Clone` prevents the pooled slice from being shared across files.

**Files changed:**
- `internal/parser/parser.go` — +1 field, +7 lines in `parseSourceFileWorker`, +1 line in `finishSourceFile`

---

## Requirement Scenario

**Scenario 1: Kit import resolution**
An ArkTS developer writes `import { Button } from '@kit.ArkUI'` in a `.ets` file. When `EtsLoaderPath` points to a valid SDK, the parser reads `{sdkPath}/openharmony/ets/build-tools/ets-loader/kit_configs/ArkUI.json`, looks up the `Button` symbol, and transforms the import to `import { Button } from 'ohos3.2/arkui/component'` — all at parse time.

**Scenario 2: No kit transformation**
When `NoTransformedKitInParser` is set, `EtsLoaderPath` is empty, or the source file has parse errors — kit processing is skipped. The import statement remains as-written.

**Scenario 3: Multi-threaded compilation**
Multiple `.ets` files are parsed in parallel. The `sync.Map` kit JSON cache ensures concurrent-safe access to loaded kit configurations without data races.

## Target Users

- **ArkTS application developers** — use `@kit.X` shorthand imports; SDK providers configure kit JSON mapping files
- **ArkTS compiler developers** — maintain kit import transformation and parser integration

## Acceptance Strategy

| Criterion | Verification |
|-----------|-------------|
| Kit import correctly transformed to file-path import | Unit tests for `createImportDeclarationForKit` with default and named bindings |
| Transform skipped when disabled or no SDK path | Unit tests for skip conditions |
| Thread-safe concurrent kit access | `sync.Map` cache + existing multi-threaded compilation tests pass |
| Node flags correctly set on transformed nodes | Unit tests verify `NodeFlagsKitImportFlags` and `NodeFlagsNoOriginalText` |
| Existing TS/ETS parsing unaffected | `go test ./...` — zero regressions |
| AST matches reference tsc after kit transform | `arkts_cmp_test.go` verifies transformed imports produce identical AST to reference |

---

## Files changed (total)

### New files

- `internal/parser/arkts_kit.go` — constants, types, cache, SDK path resolution, JSON loading, filtering, node creation, processing, range queries (~250 lines)
- `internal/parser/arkts_kit_test.go` — unit tests

### Modified files

| File | Change | Lines |
|------|--------|-------|
| `internal/ast/nodeflags.go` | Add `NodeFlagsKitImportFlags` (bit 29) and `NodeFlagsNoOriginalText` (bit 31) | +2, -1 |
| `internal/ast/ast.go` | Add `MarkedKitImportRange` field to `SourceFile` | +1 |
| `internal/core/compileroptions.go` | Add `NoTransformedKitInParser` field | +1 |
| `internal/ast/parseoptions.go` | Add `EtsLoaderPath` and `NoTransformedKitInParser` fields | +2 |
| `internal/compiler/fileloader.go` | Wire `EtsLoaderPath` and `NoTransformedKitInParser` to parse options | +2 |
| `internal/parser/parser.go` | Add `markedKitImportRanges` field, kit processing block, assignment in `finishSourceFile` | +10 |

### Generated files

- `_packages/native-preview/src/enums/nodeFlags.*.ts` — includes `KitImportFlags` and `NoOriginalText`

---

## References

| Reference tsc file | Lines | Content |
|---|---|---|
| `ohApi.ts` | 1233-1255 | Constants, KitJsonInfo, kitJsonCache |
| `ohApi.ts` | 1258-1306 | getSdkPath, isMixedCompilerSDKPath, getKitJsonObject, cleanKitJsonCache |
| `ohApi.ts` | 1308-1352 | setVirtualNodeAndKitImportFlags, createImportDeclarationForKit, markKitImport, isInMarkedKitImport |
| `ohApi.ts` | 1363-1373 | excludeStatementForKitImport |
| `ohApi.ts` | 1480-1635 | processKitStatementSuccess, processKit |
| `parser.ts` | 1835-1848 | Parser integration |
| `types.ts` | 827, 829 | NodeFlags definitions |
| `types.ts` | 4221 | SourceFile.markedKitImportRange |
| `utilities.ts` | 1892-1893 | KitImportFlags in span-of-error |
| `Issue #2` | — | AST schema + generated file pattern |
| `Issue #3` | — | Parser field addition + wiring pattern |
