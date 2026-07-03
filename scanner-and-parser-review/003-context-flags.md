# Issue #3 — EtsFlags Context-Flag System

## Context

The EtsFlags context-flag system is a shared prerequisite for all three Phase 2 parser
tasks (struct/@interface declarations, ETS component expressions, function/decorator
modifications). It provides thin boolean tracking methods that let the parser answer
"am I currently inside an ArkTS struct body?" or "am I inside a @Builder function?"
without threading state through every parse function manually.

The system has been fully implemented. This document describes what was done.

## What was implemented

### 1. EtsFlags type (`internal/ast/arkts.go`)

A `uint32` flags enum with 13 values mirroring the reference `tsc`:

| Constant | Bit | Meaning |
|---|---|---|
| `EtsFlagsNone` | 0 | No ETS context |
| `EtsFlagsStructContext` | 1 | Inside struct declaration body |
| `EtsFlagsEtsExtendComponentsContext` | 2 | Inside @Extend component |
| `EtsFlagsEtsStylesComponentsContext` | 3 | Inside @Styles component |
| `EtsFlagsEtsBuildContext` | 4 | Inside struct `build()` method |
| `EtsFlagsEtsBuilderContext` | 5 | Inside @Builder function/method |
| `EtsFlagsEtsStateStylesContext` | 6 | Inside stateStyles component |
| `EtsFlagsEtsComponentsContext` | 7 | Inside ETS component expression |
| `EtsFlagsEtsNewExpressionContext` | 8 | Inside `new` expression (suppresses EtsComponentExpression) |
| `EtsFlagsUICallbackContext` | 9 | Inside UI callback arrow function |
| `EtsFlagsSyntaxComponentContext` | 10 | Inside ForEach / LazyForEach / Repeat.each |
| `EtsFlagsSyntaxDataSourceContext` | 11 | Inside data-source argument of ForEach |
| `EtsFlagsNoEtsComponentContext` | 12 | Context where ETS component expression is suppressed |

**Reference:** `ohos-typescript types.ts:848-862`

### 2. Parser field (`internal/parser/parser.go:104`)

```go
etsFlags ast.EtsFlags
```

The field is added to the `Parser` struct. It is automatically zeroed when
`putParser` returns the struct to the pool — no explicit `clearState` needed.

### 3. Flag helpers (`internal/parser/arkts.go`)

Two internal helpers are shared by all set/in methods:

```go
func (p *Parser) setEtsFlag(val bool, flag ast.EtsFlags)
func (p *Parser) inEtsFlagsContext(flags ast.EtsFlags) bool
```

`setEtsFlag` sets the bit if `val` is `true`, clears it if `false`.
`inEtsFlagsContext` returns `true` when all given bits are set.

### 4. 12 set*Context methods

Each is a one-liner calling `setEtsFlag` with the corresponding constant:

| Method | Flag |
|---|---|
| `setStructContext` | `EtsFlagsStructContext` |
| `setEtsComponentsContext` | `EtsFlagsEtsComponentsContext` |
| `setEtsNewExpressionContext` | `EtsFlagsEtsNewExpressionContext` |
| `setEtsExtendComponentsContext` | `EtsFlagsEtsExtendComponentsContext` |
| `setEtsStylesComponentsContext` | `EtsFlagsEtsStylesComponentsContext` |
| `setEtsBuildContext` | `EtsFlagsEtsBuildContext` |
| `setEtsBuilderContext` | `EtsFlagsEtsBuilderContext` |
| `setEtsStateStylesContext` | `EtsFlagsEtsStateStylesContext` |
| `setUICallbackContext` | `EtsFlagsUICallbackContext` |
| `setSyntaxComponentContext` | `EtsFlagsSyntaxComponentContext` |
| `setSyntaxDataSourceContext` | `EtsFlagsSyntaxDataSourceContext` |
| `setNoEtsComponentContext` | `EtsFlagsNoEtsComponentContext` |

### 5. 14 in*Context methods

Thirteen check individual EtsFlags, each gated on `inEtsContext()` first.
The fourteenth is `inEtsContext()` itself — it checks the source-file-level
`NodeFlagsEtsContext` (set during `initializeState`), not EtsFlags.

| Method | Logic |
|---|---|
| `inEtsContext` | `contextFlags & NodeFlagsEtsContext != 0` |
| `inStructContext` | `inEtsContext && EtsFlagsStructContext` |
| `inEtsComponentsContext` | `inEtsContext && EtsFlagsEtsComponentsContext` |
| `inEtsNewExpressionContext` | `inEtsContext && EtsFlagsEtsNewExpressionContext` |
| `inEtsExtendComponentsContext` | `inEtsContext && EtsFlagsEtsExtendComponentsContext` |
| `inEtsStylesComponentsContext` | `inEtsContext && EtsFlagsEtsStylesComponentsContext` |
| `inBuildContext` | `inEtsContext && inStructContext && EtsFlagsEtsBuildContext` |
| `inBuilderContext` | `inEtsContext && EtsFlagsEtsBuilderContext` |
| `inEtsStateStylesContext` | `inEtsContext && (inBuildContext \|\| inBuilderContext \|\| inEtsExtendComponentsContext \|\| inEtsStylesComponentsContext) && EtsFlagsEtsStateStylesContext` |
| `inUICallbackContext` | `inEtsContext && (inBuildContext \|\| inBuilderContext) && EtsFlagsUICallbackContext` |
| `inSyntaxComponentContext` | `inEtsContext && (inBuildContext \|\| inBuilderContext) && EtsFlagsSyntaxComponentContext` |
| `inSyntaxDataSourceContext` | `inEtsContext && (inBuildContext \|\| inBuilderContext) && EtsFlagsSyntaxDataSourceContext` |
| `inNoEtsComponentContext` | `inEtsContext && (inBuildContext \|\| inBuilderContext) && EtsFlagsNoEtsComponentContext` |

### 6. initializeState wiring (`internal/parser/parser.go:312`)

When the source file's `scriptKind` is `ScriptKindETS`, the parser sets
`contextFlags = NodeFlagsEtsContext`. This is the single source of truth
that `inEtsContext()` queries.

### 7. Context hierarchy

The `in*Context` methods encode a structural hierarchy. Some flags are
standalone (Struct, Builder, Extend, Styles), while others are nested
inside the builder/build context:

```
EtsContext
├── StructContext
│   └── EtsBuildContext        (build() method body)
│       ├── UICallbackContext
│       ├── SyntaxComponentContext
│       ├── SyntaxDataSourceContext
│       └── NoEtsComponentContext
├── EtsBuilderContext          (@Builder function/method)
│   ├── UICallbackContext
│   ├── SyntaxComponentContext
│   ├── SyntaxDataSourceContext
│   └── NoEtsComponentContext
├── EtsExtendComponentsContext (@Extend)
│   └── EtsStateStylesContext
├── EtsStylesComponentsContext (@Styles)
│   └── EtsStateStylesContext
├── EtsComponentsContext       (component expression — e.g. Column() { ... })
├── EtsNewExpressionContext    (new expression — suppresses component)
└── EtsStateStylesContext      (stateStyles inside any of the four parent contexts)
```

## Files changed

| File | Lines added | What |
|---|---|---|
| `internal/ast/arkts.go` | +21 | `EtsFlags` type and 13 constants |
| `internal/parser/arkts.go` | +137 | 2 helpers, 12 setters, 14 getters |
| `internal/parser/parser.go` | +5 | `etsFlags` field + `initializeState` wiring |

## References

| Reference tsc file | Lines | Content |
|---|---|---|
| `types.ts` | 848–862 | `EtsFlags` enum definition |
| `parser.ts` | 1794 | `scanner.setEtsContext` in `initializeState` |
| `parser.ts` | 2054–2061 | `setEtsFlag` helper |
| `parser.ts` | 2079–2125 | 12 `set*Context` functions |
| `parser.ts` | 2243–2321 | `inEtsFlagsContext` + 14 `in*Context` functions |

## Requirement Scenario

**Scenario 1: Struct body tracking**
The parser enters a struct declaration body (`struct Foo { ... }`). `setStructContext(true)` is called. Subsequent parse functions query `inStructContext()` to decide behavior — e.g., auto-readonly injection for `@Param` properties, or ETS component expression recognition inside `build()`.

**Scenario 2: Builder function tracking**
A function is decorated with `@Builder`: `@Builder function myBuilder() { ... }`. `setEtsBuilderContext(true)` is called before parsing the function body. Inside the body, `inBuilderContext()` gates ArkUI-specific expression parsing (ForEach, component instantiation, UICallback arrow functions).

**Scenario 3: Context nesting**
Inside a struct's `build()` method, the parser enters multiple nested contexts simultaneously: `StructContext` + `EtsBuildContext`. When it encounters a `ForEach` call, it further enters `SyntaxComponentContext`. Each level can be independently queried — `inBuildContext()` remains true even while `inSyntaxComponentContext()` is active.

## Target Users

- **ArkTS compiler developers** — use the context system to implement ArkTS-specific parser logic without manual state threading
- **ArkTS compiler developers** — may query parser state via exposed context methods during type-checking

## Restrictions & Constraints

- **Pool safety**: `etsFlags` must be zeroed between parses. The `Parser` struct is pooled — `putParser` zeroes it implicitly via `clearState`. No explicit reset needed.
- **Flag lifetime**: Context flags are per-parse-session, not per-file. They do not persist across `ParseSourceFile` calls.
- **Thread safety**: `Parser` is not thread-safe by design (single-threaded batch compilation). No synchronization needed.
- **Flag bit limit**: `uint32` allows up to 32 flags. Current usage is 15 flags (bits 1–14), leaving ample room for future extensions.

## Acceptance Strategy

| Criterion | Verification |
|-----------|-------------|
| All 12 setters / 14 getters compile and are callable | `go build ./internal/parser/...` |
| Context isolation: ETS flags never affect .ts parsing | All existing TS tests pass unchanged (`go test ./...`) |
| Hierarchy correctness: nested flags properly gate sub-contexts | Unit tests exercising flag combination scenarios |
| No memory leaks: pool-returned parser has zeroed flags | Static analysis: `putParser` calls `clearState` which zeroes the struct |
