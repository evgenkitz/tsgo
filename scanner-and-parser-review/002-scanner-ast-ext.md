# Issue #2 — Scanner, AST, and file extensions for ArkTS (Phase 1)

## Context

Phase 1 lays the foundation for all subsequent ArkTS work: new AST nodes, file extensions,
script kind, compiler flags, and scanner keyword recognition. These five sub-tasks were
implemented as a single deliverable because every subsequent parser/binder task depends on
all of them.

## Sub-tasks

### 1. AST nodes (`_scripts/ast.json` → generated code)

Six new SyntaxKind values and four new node type definitions, all driven from the AST schema.

**New Kind values (in `kinds.elements`)**

| Kind | Insertion point | Section |
|------|----------------|---------|
| `StructKeyword` | After `ClassKeyword` | Reserved words |
| `LazyKeyword` | After `DeclareKeyword` | Contextual keywords |
| `StructDeclaration` | After `ClassDeclaration` | Element |
| `AnnotationDeclaration` | After `InterfaceDeclaration` | Element |
| `AnnotationPropertyDeclaration` | After `PropertyDeclaration` | TypeMember |
| `EtsComponentExpression` | After `SatisfiesExpression` | Expression |

**New node definitions (in `nodes.definitions`)**

| Node | Extends | Modeled after |
|------|---------|---------------|
| `StructDeclaration` | `DeclarationBase`, `StatementBase`, `ClassLikeBase` | `ClassDeclaration` |
| `AnnotationDeclaration` | `DeclarationBase`, `StatementBase`, `ExportableBase`, `ModifiersBase`, `LocalsContainerBase` | `InterfaceDeclaration` |
| `AnnotationPropertyDeclaration` | `NodeBase`, `AnnotationElementBase`, `CompositeBase` | `PropertyDeclaration` (no modifiers, no PostfixToken) |
| `EtsComponentExpression` | `PrimaryExpressionBase`, `DeclarationBase`, `CompositeBase` | — |

**New base type**

- `AnnotationElementBase` — `DeclarationBase` with `name: PropertyName`. No `ModifiersBase`
  (TS `AnnotationElement extends NamedDeclaration extends Declaration` — modifiers are absent
  at all levels of this chain). Hand-written methods in `ast.go`: `DeclarationData()`, `Name()`
  to resolve Go embedding ambiguity with `NodeDefault`.

**Type aliases updated**

- `ClassLikeDeclaration` — added `StructDeclaration` (NOT `AnnotationDeclaration`)
- `AnnotationElement` — new alias for `AnnotationPropertyDeclaration`
- `VariableOrPropertyDeclaration` — added `AnnotationPropertyDeclaration`
- `ImportPhaseModifierSyntaxKind` — added `LazyKeyword`

**Decorator extended**

- `Decorator.annotationDeclaration` — optional `*Node` field for annotation resolution

**Code generation pipeline**

```
ast.json
  → generate:ast → ast_generated.go, kind_generated.go, encoder/decoder, TS-AST
  → generate:enums → syntaxKind.enum.ts
  → go generate → kind_stringer_generated.go (stringer)
```

**Generated files (never edit directly)**

- `internal/ast/ast_generated.go` — structs, factories, visitors, clones
- `internal/ast/kind_generated.go` — Kind enum
- `internal/ast/kind_stringer_generated.go` — Kind stringer
- `internal/api/encoder/encoder_generated.go`
- `internal/api/encoder/decoder_generated.go`
- `_packages/native-preview/src/ast/*.generated.ts`
- `_packages/native-preview/src/enums/syntaxKind.*.ts`
- `_packages/native-preview/src/enums/symbolFlags.*.ts`
- `_packages/native-preview/src/enums/nodeFlags.*.ts`
- `_packages/native-preview/src/api/node/*.generated.ts`

**Hand-written files**

- `internal/ast/arkts.go` — `IsEtsOnlyKeyword` helper
- `internal/ast/ast.go` — `AnnotationElementBase` methods (Name, DeclarationData)

---

### 2. File extensions (`internal/tspath`)

ArkTS introduces two new extensions: `.ets` (implementation) and `.d.ets` (declaration).

**Decisions**

| Rule | Detail |
|------|--------|
| `.ets` is a TS implementation extension | Added to `SupportedTSImplementationExtensions` and `SupportedTSExtensionsFlat` |
| `.d.ets` is a declaration extension | Added to `SupportedDeclarationExtensions` |
| `ExtensionIsTs()` returns true for both | `.ets` and `.d.ets` are TypeScript variants |
| `GetDeclarationEmitExtensionForPath("file.ets")` returns `.d.ets` | Explicit case avoids default fallback `.d.ets.ts` |

**Files changed**

- `internal/tspath/extension.go` — constants + extension lists + helper functions
- `internal/tspath/extension_test.go` — 8 tests

---

### 3. ScriptKindETS (`internal/core`)

**Decisions**

1. `ScriptKindETS = 8` — after `ScriptKindDeferred = 7`, matching ohos-typescript.
2. `GetScriptKindFromFileName` maps `.ets`/`.d.ets` → `ScriptKindETS`.
3. `scriptkind_stringer_generated.go` regenerated via `go generate`.

**Files changed**

- `internal/core/scriptkind.go` — ScriptKindETS constant
- `internal/core/core.go` — GetScriptKindFromFileName mapping
- `internal/core/scriptkind_stringer_generated.go` — regenerated stringer
- `internal/core/scriptkind_test.go` — 4 tests for .ets/.d.ets + value check

---

### 4. Compiler flags (`internal/ast`)

**NodeFlagsEtsContext**

- `NodeFlagsEtsContext = 1 << 30` — marks a source file or node as being in ArkTS context.
  Used by `Parser.inEtsContext()`.

**SymbolFlagsAnnotation**

- `SymbolFlagsAnnotation = 1 << 31` — marks a symbol as an annotation declaration (`@interface`).

**Files changed**

- `internal/ast/nodeflags.go` — `NodeFlagsEtsContext`
- `internal/ast/symbolflags.go` — `SymbolFlagsAnnotation`
- `_packages/native-preview/src/enums/nodeFlags.*.ts` — regenerated
- `_packages/native-preview/src/enums/symbolFlags.*.ts` — regenerated

---

### 5. Scanner keywords (`internal/scanner`)

ArkTS adds two new keywords: `struct` and `lazy`. Both are keywords only in ETS context;
in regular `.ts` files they remain ordinary identifiers.

**Design: separate `textToKeywordEts` map in `scanner/arkts.go`**

ArkTS keywords live in a dedicated map, NOT in the shared `textToKeyword`. This:

- Keeps `textToKeyword` unchanged — upstream functions (`GetIdentifierToken`, `StringToToken`,
  `GetViableKeywordSuggestions`) remain diff-free
- Avoids adding ETS-context parameters to shared function signatures
- No filtering helpers needed in shared files

**Scanner thin seams in `scanner/arkts.go`**

```go
var textToKeywordEts = map[string]ast.Kind{
    "struct": ast.KindStructKeyword,
    "lazy":   ast.KindLazyKeyword,
}

func (s *Scanner) SetEtsContext(ets bool) { s.inEtsContext = ets }

// Called from Scan(), ScanJsxIdentifier(), ScanJSDocToken()
func (s *Scanner) resolveKeywordToken(str string) ast.Kind {
    if s.inEtsContext {
        if kind, ok := textToKeywordEts[str]; ok {
            return kind
        }
    }
    return GetIdentifierToken(str)
}

// Called by IdentifierToKeywordKind — AST-level keyword resolution
func resolveIdentifierKeywordKind(node *ast.Identifier) ast.Kind {
    if kind, ok := textToKeywordEts[node.Text]; ok {
        sf := ast.GetSourceFileOfNode(node.AsNode())
        if sf != nil && sf.ScriptKind == core.ScriptKindETS {
            return kind
        }
        return ast.KindUnknown
    }
    return textToKeyword[node.Text]
}
```

**Leak paths closed**

| Path | Function | Fix |
|------|----------|-----|
| Spelling suggestions | `GetViableKeywordSuggestions()` | Not affected — `textToKeyword` doesn't contain ETS keywords |
| LSP keyword completions | `allKeywordCompletions()` | Filter via `ast.IsEtsOnlyKeyword(i)` |

**Shared files — minimal changes**

| File | Change | Lines |
|------|--------|-------|
| `scanner/scanner.go` | `inEtsContext` field, `resolveKeywordToken` calls (×5), `GetScannerForSourceFile` seam | 7 |
| `scanner/utilities.go` | `IdentifierToKeywordKind` delegates to `resolveIdentifierKeywordKind` | 1 |

**Files changed**

- `internal/scanner/arkts.go` — `textToKeywordEts`, `SetEtsContext`, `resolveKeywordToken`, `resolveIdentifierKeywordKind`
- `internal/scanner/arkts_test.go` — 8 tests
- `internal/scanner/scanner.go` — 3 thin seams
- `internal/scanner/utilities.go` — 1-line delegation
- `internal/binder/binder.go` — uses `IdentifierToKeywordKind` (already correct via delegation)
- `internal/ls/completions.go` — `allKeywordCompletions()` filter

**Tests**

- `TestStructKeywordInEtsContext` — `struct` → `StructKeyword` in ETS
- `TestStructIsIdentifierOutsideEtsContext` — `struct` → `Identifier` outside ETS
- `TestLazyKeywordContextDependent` — `lazy` → `LazyKeyword` in ETS, `Identifier` outside
- `TestOtherKeywordsUnaffected` — `class` always keyword
- `TestSetEtsContext` — flag toggles correctly
- `TestIdentifierToKeywordKind_EtsContext` — ETS-aware AST keyword resolution
- `TestGetScannerForSourceFile_SetsEtsContext` — context from source file

---

## Files changed (total)

### New files

- `internal/ast/arkts.go` — `IsEtsOnlyKeyword`
- `internal/scanner/arkts.go` — `textToKeywordEts`, `SetEtsContext`, `resolveKeywordToken`, `resolveIdentifierKeywordKind`
- `internal/scanner/arkts_test.go` — 8 tests
- `internal/core/scriptkind_test.go` — extension-to-ScriptKind tests
- `docs/plans/001-scanner-ast-ext.md` — this file

### Modified files

- `_scripts/ast.json` — schema (Kind + node definitions)
- `internal/ast/ast.go` — `AnnotationElementBase` methods
- `internal/ast/nodeflags.go` — `NodeFlagsEtsContext`
- `internal/ast/symbolflags.go` — `SymbolFlagsAnnotation`
- `internal/tspath/extension.go` — `.ets`/`.d.ets` extensions
- `internal/core/scriptkind.go` — `ScriptKindETS`
- `internal/core/core.go` — `GetScriptKindFromFileName` mapping
- `internal/scanner/scanner.go` — 3 thin seams (field, call sites, GetScannerForSourceFile)
- `internal/scanner/utilities.go` — 1-line delegation
- `internal/ls/completions.go` — LSP completion filter
- `internal/parser/parser.go` — `scanner.SetEtsContext` in `initializeState`
- `CLAUDE.md` — ArkTS code convention

### Generated files (committed with schema change)

- `internal/ast/ast_generated.go`
- `internal/ast/kind_generated.go`
- `internal/ast/kind_stringer_generated.go`
- `internal/core/scriptkind_stringer_generated.go`
- `internal/api/encoder/encoder_generated.go`
- `internal/api/encoder/decoder_generated.go`
- `_packages/native-preview/src/ast/*.generated.ts`
- `_packages/native-preview/src/enums/syntaxKind.*.ts`
- `_packages/native-preview/src/enums/symbolFlags.*.ts`
- `_packages/native-preview/src/enums/nodeFlags.*.ts`
- `_packages/native-preview/src/api/node/*.generated.ts`

## References

- ohos-typescript: `src/compiler/types.ts` (SyntaxKind, EtsFlags, ScriptKind, NodeFlags, SymbolFlags, Extension)
- ohos-typescript: `src/compiler/scanner.ts:1583-1590` (keyword lookup)
- ohos-typescript: `src/compiler/utilities.ts` (supportedTSExtensions)

## Requirement Scenario

**Scenario 1: Parsing an .ets source file**
A developer writes `struct MyComponent { ... }` in a `.ets` file. The scanner must recognize `struct` as a keyword (not an identifier) when `inEtsContext` is true. The parser receives `KindStructKeyword` and can route to struct-parsing logic.

**Scenario 2: .ts file backward compatibility**
A developer writes `const struct = 1;` in a `.ts` file. The scanner must treat `struct` as a regular identifier — ETS keywords must NOT leak into standard TypeScript compilation.

**Scenario 3: Code generation pipeline**
A developer adds a new ArkTS AST node type to `_scripts/ast.json`. Running `npx hereby generate:ast` regenerates all Go and TypeScript code. The generated code compiles without manual edits.

## Target Users

- **ArkTS compiler developers** — engineers porting ArkTS features from the reference tsc to Go
- **ArkTS application developers** — indirectly, as end users of the `.ets` file format and ArkTS syntax

## Restrictions & Constraints

- **Generated files**: `ast_generated.go`, `kind_generated.go`, `encoder_generated.go`, and `decoder_generated.go` must never be edited directly — always edit `_scripts/ast.json` and regenerate.
- **Keyword isolation**: ETS keywords (`struct`, `lazy`) must NOT affect `.ts`/`.js` parsing. The `textToKeywordEts` map is separate from `textToKeyword` to prevent cross-contamination.
- **Backward compatibility**: all existing TypeScript tests must pass unchanged. Zero tolerance for regressions in non-ETS code paths.
- **AST schema stability**: new SyntaxKind values and node types must be placed at specific insertion points in `ast.json` to preserve binary compatibility of the generated encoder/decoder.

## Acceptance Strategy

| Criterion | Verification |
|-----------|-------------|
| `.ets` files produce correct tokens | Scanner tests: `struct`/`lazy` → keywords in ETS context |
| `.ts` files unaffected | Scanner tests: `struct`/`lazy` → identifiers outside ETS context |
| Code generation produces valid Go | `npx hereby build` compiles without errors |
| All existing tests pass | `go test ./...` — zero regressions |
| LSP keyword completions exclude ETS keywords | `allKeywordCompletions()` filter via `IsEtsOnlyKeyword` |
| AST and diagnostics match reference tsc | `arkts_cmp_test.go` compares scanner output against reference for .ets test files |
