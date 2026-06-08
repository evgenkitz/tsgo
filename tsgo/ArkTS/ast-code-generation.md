# AST Code Generation from `ast.json`

## Contents

1. [Overview: Schema-Driven Code Generation](#chapter-1-overview-schema-driven-code-generation)
2. [The `ast.json` Format and Generation Pipeline](#chapter-2-the-astjson-format-and-generation-pipeline)
   - 2.1 [Schema Format](#21-schema-format)
   - 2.2 [What Gets Generated](#22-what-gets-generated)
   - 2.3 [What Must Be Written by Hand](#23-what-must-be-written-by-hand)

---

## Chapter 1. Overview: Schema-Driven Code Generation

### What It Is

The TypeScript 7 (tsgo) compiler uses a **schema-driven code generation** system. A single JSON file — `_scripts/ast.json` (~5,500 lines) — defines every AST node type, every syntax kind, and every type relationship. Three TypeScript-based code generators read this file and produce ~15,000 lines of boilerplate Go and TypeScript code.

### Why Schema-Driven?

| Pros | Cons |
|------|------|
| **Single source of truth.** Change one JSON entry → all boilerplate regenerated consistently. No drift between Go and TS representations. | **Indirection.** Contributors must learn the schema format before they can add a node. |
| **Compile-time safety.** The schema is validated on load — missing `extends` references, invalid type aliases, and broken marker chains are caught before any code is emitted. | **Generator complexity.** The three generators + shared `schema.ts` are ~4,000 lines of meta-code. Bugs in the generator are harder to debug than bugs in generated code. |
| **Eliminates hundreds of copy-paste errors.** Per-node structs, factories (`New*`/`Update*`), visitors (`ForEachChild`/`VisitEachChild`), cloners, type guards (`Is*`), casters (`As*`), and binary encoders are all mechanized. | **Escape hatches needed.** Some nodes have runtime-dependent child ordering (`handWrittenVisitor`) or entirely custom implementations (`handWritten`). The schema must accommodate these exceptions. |
| **Arena allocation.** Nodes marked `arena: true` get pooled allocation via `core.Arena[T]` — the generator emits the arena field, the `New()` call, and the factory wiring automatically. | **Generated code is verbose.** `ast_generated.go` alone is ~10,000 lines. The generated visitor/clone/factory methods for every node make the file large. |
| **Cross-language sync.** The same schema produces Go structs for the native compiler AND TypeScript interfaces for the npm client library (`native-preview`). | **Two-step pipeline for TS enums.** `SyntaxKind` goes `json → Go → TS` (via `generate:enums` parsing Go source), adding an extra step. |

### The Three Commands

| Command | What it runs | Output |
|---------|-------------|--------|
| `npx hereby generate:ast` | `node --experimental-strip-types _scripts/generate.ts` | Go: `ast_generated.go`, `kind_generated.go`, `encoder/decoder_generated.go`<br>TS: `ast.generated.ts`, `factory.generated.ts`, `is.generated.ts`, `visitor.generated.ts`, `protocol.generated.ts`, `encoder.generated.ts`, `node.generated.ts` |
| `npx hereby generate:enums` | Enum parser in `Herebyfile.mjs` (regex-parses Go source) | TS: `syntaxKind.enum.ts`, `syntaxKind.ts`, `nodeFlags.*.ts`, `modifierFlags.*.ts`, etc. (~22 enum pairs) |
| `npx hereby generate` | `go generate -v ./...` | `kind_stringer_generated.go` + all other `stringer`/`moq`/`bundled` outputs (see [section 2.2.1](#221-go-generated-files)) |

### Real-World Scale

Experimental addition of `struct` support (a new declaration type similar to `class` without specific `struct` logic) touched **56 files**: 13 generated files plus 43 hand-written files across 11 modules. The generated files changed automatically from the `ast.json` edit; the hand-written files required manual `switch` case additions throughout the compiler.

---

## Chapter 2. The `ast.json` Format and Generation Pipeline

### 2.1 Schema Format

`_scripts/ast.json` has four top-level sections. The formal schema is in `_scripts/ast.schema.json`.

```
ast.json
├── kinds
│   ├── elements[]     → Kind enum (iota constants)
│   ├── markers[]      → Named aliases for kind values
│   └── aliases{}      → Kind union types (TokenSyntaxKind, BinaryOperator, …)
├── bases{}            → Mixin structs (StatementBase, ExpressionBase, …)
└── nodes
    ├── definitions{}  → Concrete node types (IfStatement, Identifier, …)
    ├── aliases{}      → Node union types (Expression, Statement, …)
    └── listAliases{}  → NodeList specializations (StatementList, …)
```

#### 2.1.1 `kinds.elements` — The SyntaxKind Enum

An ordered array. Each entry is either a string (kind name) or an object with `name` and/or `comment`.

```jsonc
"elements": [
    "Unknown",
    "EndOfFile",
    { "comment": "Punctuation" },
    "OpenBraceToken",
    "CloseBraceToken",
    { "comment": "Reserved words" },
    "BreakKeyword",
    // ...
    { "name": "DeferKeyword", "comment": "LastKeyword" },
    { "comment": "Parse tree nodes" },
    "SourceFile"
]
```

**Generates** (`kind_generated.go`):
```go
type Kind int16

const (
    KindUnknown Kind = iota
    KindEndOfFile
    // Punctuation
    KindOpenBraceToken
    KindCloseBraceToken
    // Reserved words
    KindBreakKeyword
    // ...
)
```

#### 2.1.2 `kinds.markers` — Range Sentinel Values

Named aliases pointing to existing kind values, used to define ranges.

```jsonc
"markers": [
    { "name": "FirstToken",    "value": "Unknown" },
    { "name": "LastToken",     "value": "LastKeyword" },
    { "name": "FirstStatement","value": "VariableStatement" },
    { "name": "LastStatement", "value": "DebuggerStatement" }
]
```

**Generates**: constants equal to their targets: `KindFirstToken = KindUnknown`, etc.

#### 2.1.3 `kinds.aliases` — Kind Union Types

Two forms:

**Range** — uses markers to define a contiguous block:
```jsonc
"TokenSyntaxKind":  { "range": ["FirstToken", "LastToken"] },
"KeywordSyntaxKind": { "range": ["FirstKeyword", "LastKeyword"] }
```

**Enumerated** — explicit list, can reference sub-aliases:
```jsonc
"ModifierSyntaxKind": ["AbstractKeyword", "AccessorKeyword", /*...*/ "StaticKeyword"],
"BinaryOperator":     ["AssignmentOperatorOrHigher", "CommaToken"]
```

**Generates**: Go type alias + guard function:
```go
type TokenSyntaxKind = Kind // KindUnknown | KindEndOfFile | ... | KindDeferKeyword

func IsTokenKind(kind Kind) bool {
    return kind >= KindUnknown && kind <= KindDeferKeyword
}
```

#### 2.1.4 `bases` — Mixin Structs

Each base maps to a Go struct with embedded fields. Multiple inheritance is supported.

```jsonc
"StatementBase": {
    "brand": "_statementBrand",   // TS nominal typing brand
    "extends": ["NodeBase", "FlowNodeBase"]
},
"FunctionLikeBase": {
    "extends": ["DeclarationBase", "LocalsContainerBase"],
    "fields": {
        "TypeParameters": { "type": "TypeParameterDeclaration", "list": "NodeList", "optional": true },
        "Parameters":     { "type": "ParameterDeclaration", "list": "NodeList" },
        "Type":           { "type": "TypeNode", "optional": true }
    }
},
"DeclarationBase": {
    "fields": {
        "Symbol": { "type": "*Symbol", "goOnly": true }  // Go-only: not in TS, not serialized
    }
}
```

**Field properties:**

| Property | Effect |
|----------|--------|
| `type` | Go type, or `[string, array]` for union |
| `list` | `"NodeList"` / `"ModifierList"` / `"raw"` (for `[]*Node`) |
| `optional` | Marks field as optional |
| `private` | Unexported in Go (lowercase) |
| `goOnly` | Go-only — implies both `noTS` and `noFactory` |
| `noGo` | Excluded from Go structs entirely |
| `noTS` | Excluded from TS output and binary encoding |
| `noFactory` | Excluded from factory params, visitor, and clone |
| `visit` | Visitor method override name (e.g. `"embeddedStatement"`) |

The `goOnly` pattern is used extensively for compiler-internal state (symbol tables, flow nodes, facts) that should never appear in the serialized AST or TS interfaces.

#### 2.1.5 `nodes.definitions` — Concrete AST Nodes

Each key is a node name. Every node must declare `extends` (which bases it embeds). Members define the node's own fields.

```jsonc
"IfStatement": {
    "generateSubtreeFacts": true,
    "extends": ["StatementBase", "CompositeBase"],
    "members": [
        { "name": "Expression",    "type": "Expression" },
        { "name": "ThenStatement", "type": "Statement", "visit": "embeddedStatement" },
        { "name": "ElseStatement", "type": "Statement", "optional": true, "visit": "embeddedStatement" }
    ],
    "arena": true
},
"Token": {
    "extends": ["NodeBase"],
    "typeParameters": [
        { "name": "TKind", "constraint": "TokenSyntaxKind", "default": "TokenSyntaxKind" }
    ],
    "members": [
        { "name": "Kind", "type": "TKind", "inherited": true }
    ],
    "arena": true,
    "instantiationAliases": {
        "DotToken": "DotToken",
        "AbstractKeyword": "AbstractKeyword",
        "BinaryOperatorToken": "BinaryOperator"
    }
}
```

**Node properties:**

| Property | Effect |
|----------|--------|
| `extends` | **Required.** Base names to embed. First is primary. |
| `kind` | SyntaxKind constant override. String = single kind; array = multiple kinds sharing one struct. |
| `members` | Ordered fields — defines factory param order, visitor traversal order, and binary encoding order. |
| `arena` | Use pooled allocation (`core.Arena[T]`). |
| `handWritten` | Skip all generation — struct, factory, visitor, clone are hand-written. Only `Is*()`/`As*()` generated. |
| `handWrittenVisitor` | `ForEachChild`/`VisitEachChild` are hand-written (for nodes with runtime-dependent child order, e.g. `BinaryExpression`). |
| `generateSubtreeFacts` | Generate `computeSubtreeFacts()` method. |
| `typeParameters` | Generic type params (e.g., `Token<TKind>`). |
| `instantiationAliases` | Named instantiations (e.g., `DotToken = Token<SyntaxKind.DotToken>`). |

**Member properties** (same as base fields, plus):

| Property | Effect |
|----------|--------|
| `inherited` | This field comes from a base; only override specific attributes (like `visit` or `optional`). |
| `bitmask` | Bitmask to AND with the value in factory functions. |

**How `noFactory`/`noTS`/`goOnly`/`noGo` control where fields appear:**

```
                     ┌─────────────────────────────────────────────┐
                     │           WHERE THE FIELD APPEARS           │
                     ├──────────┬──────────┬────────┬────────┬─────┤
                     │ Go struct│ Factory  │Visitor │ Clone  │ TS/ │
                     │          │ params   │        │        │ Enc │
┌────────────────────┼──────────┼──────────┼────────┼────────┼─────┤
│ (default)          │    ✓     │    ✓    │   ✓    │   ✓    │ ✓  │
│ private            │    ✓*    │    ✗    │   ✗    │   ✓†   │ ✗  │
│ noFactory          │    ✓     │    ✗    │   ✗    │   ✗    │ ✓  │
│ noTS               │    ✓     │    ✓    │   ✓    │   ✓    │ ✗  │
│ goOnly (noTS+nofac)│    ✓     │    ✗    │   ✗    │   ✗    │ ✗  │
│ noGo               │    ✗     │    ✗    │   ✗    │   ✗    │ ✓  │
└────────────────────┴──────────┴──────────┴────────┴─────────┴────┘
* unexported (lowercase)   † accessible via struct method
```

#### 2.1.6 `nodes.aliases` — Node Union Types

Two forms:

**Base aliases** — all nodes extending a given base:
```jsonc
"Expression":  { "base": "ExpressionBase" },
"Statement":   { "base": "StatementBase" }
```

**Enumerated aliases** — explicit list:
```jsonc
"ConciseBody": ["Block", "Expression"],
"ModuleName":  ["Identifier", "StringLiteral"]
```

**Generates**: Go type alias to `*Node` with a doc comment.

#### 2.1.7 `nodes.listAliases` — NodeList Specializations

```jsonc
"StatementList":    "Statement",
"ParameterList":    "ParameterDeclaration",
"ArgumentList":     "Expression"
```

**Generates**: `type StatementList = NodeList // NodeList[*Statement]`

#### 2.1.8 The `inherited` Pattern

Members marked `inherited: true` refer to a field defined in a base. The node can override specific attributes without redefining the entire field. This is how the same base field gets different visitor behavior in different contexts:

```jsonc
// Base:
"IterationStatementBase": {
    "fields": { "Statement": { "type": "Statement" } }
}
// Nodes override visitor:
"WhileStatement": {
    "members": [
        { "name": "Statement", "type": "Statement", "visit": "iterationBody", "inherited": true }
    ]
}
```

---

### 2.2 What Gets Generated

The following diagram shows the complete generation pipeline with all commands and output files:

```
                             ┌─────────────────┐
                             |_scripts/ast.json|
                             └─────────────────┘
                                     │
                                     ▼
                           npx hereby generate:ast
                                     │
               ┌─────────────────────┼─────────────────────────┐
               │                     │                         │
               ▼                     ▼                         ▼
 ┌──────────────────────┐   ┌──────────────────────┐   ┌───────────────────────┐
 │ Go AST generator     │   │ TS AST generator     │   │ Encoder/Decoder       |
 │ (generate-go-ast.ts) │   │ (generate-ts-ast.ts) │   │ (generate-encoder.ts) │
 └─────────┬────────────┘   └─────┬────────────────┘   └──────┬────────────────┘
           │                      │                           │
           ▼                      ▼                           ▼
┌──────────────────────┐ ┌─────────────────┐ ┌──────────────────────────┐
│  internal/ast/       │ │ _packages/      │ │ internal/api/encoder/    │
│  ast_generated.go    │ │ native-preview/ │ │  encoder_generated.go    │
│  kind_generated.go   │ │  src/ast/       │ │  decoder_generated.go    │
│                      │ │  *.generated.ts │ │                          │
│  NodeFactory,        │ │                 │ │ _packages/               │
│  node structs,       │ │ _packages/      │ │ native-preview/          │
│  New*/Update*,       │ │ native-preview/ │ │  src/api/node/           │
│  ForEachChild,       │ │  src/api/node/  │ │  *.generated.ts          │
│  VisitEachChild,     │ │  *.generated.ts │ │                          │
│  Clone, Is*, As*,    │ │                 │ │                          │
│  kind guards,        │ └─────────────────┘ └──────────────────────────┘
│  type aliases        │
└──────────┬───────────┘
           │
           ├──────────────────────────────────────────┐
           │                                          │
           ▼                                          ▼
   npx hereby generate                     npx hereby generate:enums
   (go generate ./...)                     (Herebyfile.mjs: parse Go → TS enums)
           │                                          │
           ▼                                          ▼
┌──────────────────────────────┐    ┌──────────────────────────────────────┐
│ kind_stringer_generated.go   │    │ _packages/native-preview/src/enums/  │
│ + 10 other stringer files    │    │                                      │
│ + 3 moq mock files           │    │ syntaxKind.enum.ts + syntaxKind.ts   │
│   (vfsmock, projecttestutil) │    │ nodeFlags.enum.ts  + nodeFlags.ts    │
│ + bundled libs               │    │ modifierFlags.enum.ts + .ts          │
│ + diagnostics generated      │    │ tokenFlags, symbolFlags, typeFlags,  │
└──────────────────────────────┘    │ objectFlags, signatureKind,          │
                                    │ elementFlags, typePredicateKind,     │
                                    │ diagnosticCategory, nodeBuilderFlags,│
                                    │ outerExpressionKinds                 │
                                    │ (13 enum pairs total)                │
                                    └──────────────────────────────────────┘
```

#### 2.2.1 Go Generated Files

<details>
<summary><strong><code>npx hereby generate:ast</code></strong> producing</summary>

| File | Lines | Description |
|------|-------|-------------|
| `internal/ast/ast_generated.go` | ~10,000 | `NodeFactory` struct with arena fields; base struct definitions; node struct definitions; `New*()` / `Update*()` factory methods; `ForEachChild()` / `VisitEachChild()` / `Clone()` methods; `computeSubtreeFacts()` methods; `Name()` accessors; `Is*()` type guards; `As*()` cast methods; kind alias guards (`IsTokenKind`, `IsKeywordKind`, etc.); NodeList / Node / union type aliases |
| `internal/ast/kind_generated.go` | ~466 | `Kind` enum (iota constants), marker constants, kind union type aliases (`type TokenSyntaxKind = Kind`) |
| `internal/api/encoder/encoder_generated.go` | ~713 | Binary serialization for every node type |
| `internal/api/encoder/decoder_generated.go` | auto | Binary deserialization for every node type |
</details>


<details>
<summary><strong><code>npx hereby generate</code></strong> (<code>go generate -v ./...</code>) produces additional <code>stringer</code>/<code>moq</code>/bundled outputs</summary>

| Tool | Output File | Purpose |
|------|-------------|---------|
| `stringer` | `internal/ast/kind_stringer_generated.go` | `Kind.String()` |
| `stringer` | `internal/core/languagevariant_stringer_generated.go` | `LanguageVariant.String()` |
| `stringer` | `internal/core/modulekind_stringer_generated.go` | `ModuleKind.String()` |
| `stringer` | `internal/core/scripttarget_stringer_generated.go` | `ScriptTarget.String()` |
| `stringer` | `internal/core/scriptkind_stringer_generated.go` | `ScriptKind.String()` |
| `stringer` | `internal/core/tristate_stringer_generated.go` | `Tristate.String()` |
| `stringer` | `internal/checker/stringer_generated.go` | `SignatureKind.String()` |
| `stringer` | `internal/diagnostics/stringer_generated.go` | `Category.String()` |
| `stringer` | `internal/ls/autoimport/export_stringer_generated.go` | `ExportSyntax.String()` |
| `stringer` | `internal/vfs/vfsmatch/stringer_generated.go` | `Usage.String()` |
| `stringer` | `internal/project/project_stringer_generated.go` | `Kind.String()` |
| `moq` | `internal/vfs/vfsmock/mock_generated.go` | Mock for `vfs.FS` interface |
| `moq` | `internal/testutil/projecttestutil/clientmock_generated.go` | Mock for `project.Client` interface |
| `moq` | `internal/testutil/projecttestutil/npmexecutormock_generated.go` | Mock for `project/ata.NpmExecutor` interface |
| `go run` | `internal/diagnostics/diagnostics_generated.go` | Diagnostic message constants |
| `go run` | `internal/diagnostics/loc_generated.go` | Localization keys |
| `go run` | `internal/bundled/` | Bundled lib files (lib.es\*.d.ts) |
</details>

#### 2.2.2 TypeScript Generated Files

<details>
<summary><strong><code>npx hereby generate:ast</code></strong> producing</summary>

| File | Description |
|------|-------------|
| `_packages/native-preview/src/ast/ast.generated.ts` | TS interfaces for each AST node |
| `_packages/native-preview/src/ast/factory.generated.ts` | TS factory functions (`createIdentifier`, `createIfStatement`, ...) |
| `_packages/native-preview/src/ast/is.generated.ts` | TS type guard functions (`isIdentifier`, `isIfStatement`, ...) |
| `_packages/native-preview/src/ast/visitor.generated.ts` | TS visitor/transformation functions |
| `_packages/native-preview/src/api/node/protocol.generated.ts` | Binary protocol type definitions |
| `_packages/native-preview/src/api/node/encoder.generated.ts` | TS binary encoder |
| `_packages/native-preview/src/api/node/node.generated.ts` | TS node data structures |
</details>

<details>
<summary><strong><code>npx hereby generate:enums</code></strong> produces (by parsing Go source in <code>Herebyfile.mjs</code>)</summary>

| File | Description |
|------|-------------|
| `_packages/native-preview/src/enums/syntaxKind.enum.ts` | TS `SyntaxKind` enum (type-level) |
| `_packages/native-preview/src/enums/syntaxKind.ts` | TS `SyntaxKind` runtime (IIFE) |
| `_packages/native-preview/src/enums/nodeFlags.enum.ts` + `.ts` | `NodeFlags` enum pair |
| `_packages/native-preview/src/enums/modifierFlags.enum.ts` + `.ts` | `ModifierFlags` enum pair |
| `_packages/native-preview/src/enums/tokenFlags.enum.ts` + `.ts` | `TokenFlags` enum pair |
| `_packages/native-preview/src/enums/symbolFlags.enum.ts` + `.ts` | `SymbolFlags` enum pair |
| `_packages/native-preview/src/enums/typeFlags.enum.ts` + `.ts` | `TypeFlags` enum pair |
| `_packages/native-preview/src/enums/objectFlags.enum.ts` + `.ts` | `ObjectFlags` enum pair |
| `_packages/native-preview/src/enums/signatureKind.enum.ts` + `.ts` | `SignatureKind` enum pair |
| `_packages/native-preview/src/enums/elementFlags.enum.ts` + `.ts` | `ElementFlags` enum pair |
| `_packages/native-preview/src/enums/typePredicateKind.enum.ts` + `.ts` | `TypePredicateKind` enum pair |
| `_packages/native-preview/src/enums/diagnosticCategory.enum.ts` + `.ts` | `DiagnosticCategory` enum pair |
| `_packages/native-preview/src/enums/nodeBuilderFlags.enum.ts` + `.ts` | `NodeBuilderFlags` enum pair |
| `_packages/native-preview/src/enums/outerExpressionKinds.enum.ts` + `.ts` | `OuterExpressionKinds` enum pair |
</details>

#### 2.2.3 The `nodeData` Polymorphic Dispatch Pattern

All generated node structs participate in a polymorphic dispatch system through the `nodeData` interface:

```
┌──────────────────────────────────────────┐
│              *Node                       │
│  Kind, Flags, Loc, id, Parent            │
│  data → nodeData (polymorphic dispatch)  │
└────────────┬─────────────────────────────┘
             │
    ┌────────┴────────────────────────────┐
    │         nodeData interface          │
    │  AsNode(), ForEachChild(),          │
    │  VisitEachChild(), Clone(),         │
    │  Name(), Modifiers(), ...           │
    └────────┬────────────────────────────┘
             │
    ┌────────┴───────────────┐
    │                        │
    ▼                        ▼
┌───────────┐          ┌───────────────┐
│NodeDefault│          │  IfStatement  │
│ (embedded │          │  (generated)  │
│  in every │          │               │
│  struct)  │          │ StatementBase │
│ returns   │          │ CompositeBase │
│ nil/false │          │ Expression    │
│ for all   │          │ ThenStatement │
│ methods   │          │ ElseStatement │
└───────────┘          └───────────────┘
```

Every generated struct embeds `NodeDefault` (which embeds `Node`). `NodeDefault` returns nil/zero for all interface methods — individual nodes override only what they actually have. `IfStatement` overrides `ForEachChild`/`VisitEachChild` (it has children); `Identifier` only overrides `Clone` (it has data fields but no children).

#### 2.2.4 How `generate:enums` Works

The `generate:enums` task is defined directly in `Herebyfile.mjs` and does NOT read `ast.json`. Instead, it regex-parses Go source files:

```
internal/ast/kind_generated.go   --parse--→  syntaxKind.enum.ts + syntaxKind.ts
internal/ast/nodeflags.go        --parse--→  nodeFlags.enum.ts + nodeFlags.ts
internal/ast/modifierflags.go    --parse--→  modifierFlags.enum.ts + modifierFlags.ts
...
```

Each Go `const` block is parsed into members, then two TS files are written:
- **`.enum.ts`** — TypeScript `enum` declaration (for type-level usage)
- **`.ts`** — IIFE-based runtime object (for value-level usage, mirrors the enum)

This means the TS enums are always consistent with Go without a separate schema.

#### 2.2.5 Generated Method Details

For each node in `ast.json` → `nodes.definitions`, the `generate-go-ast.ts` generator produces:

**`New<Name>()` factory on `NodeFactory`:**
```go
func (f *NodeFactory) NewIfStatement(expression *Expression, thenStatement *Statement, elseStatement *Statement) *Node {
    data := f.ifStatementArena.New()  // pooled allocation if arena: true
    data.Expression = expression
    data.ThenStatement = thenStatement
    data.ElseStatement = elseStatement
    return f.newNode(KindIfStatement, data)
}
```

**`Update<Name>()` factory** — copy-on-write: creates a new node only if a parameter differs:
```go
func (f *NodeFactory) UpdateIfStatement(node *IfStatement, expression *Expression, ...) *Node {
    if expression != node.Expression || thenStatement != node.ThenStatement || ... {
        return updateNode(f.NewIfStatement(...), node.AsNode(), f.hooks)
    }
    return node.AsNode()
}
```

**`ForEachChild()`** — depth-first traversal:
```go
func (node *IfStatement) ForEachChild(v Visitor) bool {
    return visit(v, node.Expression) || visit(v, node.ThenStatement) || visit(v, node.ElseStatement)
}
```

**`VisitEachChild()`** — structural copy with transformed children:
```go
func (node *IfStatement) VisitEachChild(v *NodeVisitor) *Node {
    return v.Factory.UpdateIfStatement(node,
        v.visitNode(node.Expression),
        v.visitEmbeddedStatement(node.ThenStatement),  // from "visit": "embeddedStatement"
        v.visitEmbeddedStatement(node.ElseStatement),
    )
}
```

**`Clone()`**:
```go
func (node *IfStatement) Clone(f NodeFactoryCoercible) *Node {
    return cloneNode(f.AsNodeFactory().NewIfStatement(
        node.Expression, node.ThenStatement, node.ElseStatement,
    ), node.AsNode(), f.AsNodeFactory().hooks)
}
```

**`computeSubtreeFacts()`** (if `generateSubtreeFacts: true`):
```go
func (node *IfStatement) computeSubtreeFacts() SubtreeFacts {
    return propagateSubtreeFacts(node.Expression) |
        propagateSubtreeFacts(node.ThenStatement) |
        propagateSubtreeFacts(node.ElseStatement)
}
```

**`Is<Name>()` type guard:**
```go
func IsIfStatement(node *Node) bool { return node.Kind == KindIfStatement }
```

**`As<Name>()` cast:**
```go
func (n *Node) AsIfStatement() *IfStatement { return n.data.(*IfStatement) }
```

---

### 2.3 What Must Be Written by Hand

The generated code covers mechanical boilerplate, but the **semantic behavior** of each node must be written by hand. This section describes the general algorithm: which modules need changes depending on the **category** of node being added, and what kind of changes are needed in each module.

#### 2.3.1 Decision Table: What Needs Changes by Node Category

Not every new node requires changes in every module. The scope depends on the node's role:

| Node category | Scanner | Parser | AST | Binder | Checker | Printer | Transformers | Formatter | LS | Diag | Tests |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| New keyword/token | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| New declaration | ? | ✓ | ✓ | ✓ | ✓ | ✓ | ? | ✓ | ✓ | ? | ✓ |
| New expression | ✗ | ✓ | ✓ | ? | ✓ | ✓ | ? | ✗ | ? | ✗ | ✓ |
| New statement | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ | ? | ✓ | ✓ | ✗ | ✓ |
| New type node | ✗ | ✓ | ✓ | ✗ | ✓ | ✓ | ✗ | ✗ | ? | ✗ | ✓ |
| New JSDoc node | ✗ | ✓* | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ | ✓ |
| Internal/synthetic | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |

```
✓ = always needed   ? = depends on semantics   ✗ = usually not needed   ✓* = in JSDoc parser, not main parser
```

**Example.** The `struct` declaration (category: *new declaration*) required changes in all 11 hand-written modules — **43 files** across scanner, parser, AST utils, binder, checker, printer, transformers, formatter, LS, diagnostics, and tests.

#### 2.3.2 General Algorithm

```
                        Edit ast.json
                             │
                       Run generators
                             │
              ┌──────────────┴────────────────┐
              │   For each compiler module:   │
              │   Does this module have a     │
              │   switch/case on the analogous│
              │   node that my new node       │
              │   behaves like?               │
              └──────────────┬────────────────┘
                             │
                     ┌───────┴───────┐
                     │               │
                     ▼               ▼
                    YES             NO
                Add new kind     May still need
                to the same      a new handler:
                switch cases     write module-
                                 specific logic
                     │               │
                     └───────┬───────┘
                             │
                             ▼
                     Write tests
                     (fourslash + compiler baselines)
```

**Step-by-step:**

1. **Identify the analogous node.** Find an existing node that behaves most like your new one (e.g., for `struct` → `class`). This is your reference for where to add cases.
2. **For each module**, search for `switch node.Kind` (or `switch kind`) that handles the analogous node. Add your new kind to the same case.
3. **For modules that need unique behavior**, write new functions (parse function, check function, emit function). The generated `New*()` factory handles node construction; your code handles semantics.
4. **Add diagnostic messages** if the node has new error conditions.
5. **Add tests.** At minimum: a compiler baseline test (`.ts` + baselines) and, if the node affects editor behavior, fourslash tests.

#### 2.3.3 Module-by-Module Guide

Below, each module is described in general terms — what it does, what kind of changes are typically needed, and what questions to ask when adding a node.

---

**Scanner** (`internal/scanner/`)

*Role:* Lexical analysis — converts source text into tokens with `Kind` values.

*Changes needed when:*
- Adding a **new keyword** → add entry to `textToKeyword` map
- Adding a **new token type** → update `GetErrorRangeForNode` and token-classification switches

*Questions to ask:* Does my node start with a new keyword? If not, skip this module.

```go
// To add a new keyword "example", add to textToKeyword:
var textToKeyword = map[string]ast.Kind{
    "example": ast.KindExampleKeyword,  // new keyword
}
```

---

**Parser** (`internal/parser/`)

*Role:* Consumes tokens and builds AST nodes using generated `New*()` factories.

*Changes needed when:*
- Adding any **non-synthetic node** → write a `parse<Name>()` function that calls the generated factory
- Add the new kind to dispatch switches: `parseStatement()`, `parseDeclarationWorker()`, `isStartOfStatement()`, `scanStartOfDeclaration()` (or expression equivalents: `parsePrimaryExpression()`, `parsePrefixUnaryExpression()`, etc.)
- If the node needs **incremental re-parsing** → update `reparser.go`

*Questions to ask:* What token triggers parsing of my node? Is it a statement, declaration, expression, or type?

---

**AST Utilities** (`internal/ast/`)

*Role:* Kind classification helpers (`IsDeclarationKind`, `IsStatementKind`, etc.) and core infrastructure.

*Changes needed when:*
- **`utilities.go`** — add new kind to relevant classification lists (e.g., `IsDeclaration`, `IsStatement`, `IsExpression`, `IsTypeNode`). These lists are used by the binder, checker, and LS for dispatch.
- **`ast.go`** — only if the new node requires new `nodeData` interface methods or new polymorphic accessors on `*Node`. Most nodes reuse existing base accessors; struct didn't need any changes here.

*Questions to ask:* Which classification group does my node belong to? Does it produce a declaration, a statement, an expression, or a type?

---

**Binder** (`internal/binder/`)

*Role:* Name resolution — walks the AST, creates `Symbol` objects, builds symbol tables, resolves references.

*Changes needed when:*
- Adding a **declaration** or **statement that introduces a scope** → add kind cases to `bind()`, `declareSymbolAndAddToSymbolTable()`, `GetContainerFlags()`, and related switches
- **`nameresolver.go`**, **`referenceresolver.go`** — if the node introduces new names or references that need resolution

*Questions to ask:* Does my node declare a name? Does it introduce a new scope? Does it reference names from other scopes?

If the node behaves identically to an existing one (e.g., a new declaration that scopes like a class), reusing the same bind function is typical — just add the new kind to the same switch case:

---

**Checker** (`internal/checker/`) — the largest consumer

*Role:* Type checking, type inference, flow analysis, grammar validation.

*Changes needed when:*
- Adding a **declaration** → write a `check<Name>()` function with type-checking logic. Add cases to `checkSourceElementWorker()` (or the expression/type equivalent).
- **Declaration-space classification** → add to `getDeclarationSpaces()`, `getGlobalTypeDeclaration()`
- **Type parameter handling** → add to `getOuterTypeParameters()`, `appendLocalTypeParametersOfClassOrInterfaceOrTypeAlias()`
- **Unused identifier tracking** → add to `registerForUnusedIdentifiersCheck()`, `checkUnusedIdentifiers()`
- **Decorator handling** (if applicable) → add to `checkDecorator()`, `getDecoratorArgumentCount()`, `getESDecoratorCallSignature()`, etc.
- **`grammarchecks.go`** → if the node has grammar-level constraints (e.g., "cannot be used with `using` declarations")
- **`flow.go`** → only if the node introduces new control flow (break/continue/return/throw/try). Struct didn't need this.
- **`nodebuilder.go`** → if the node participates in type resolution that needs node building
- **`emitresolver.go`** → if the node affects emit-time type resolution
- **`services.go`**, **`symbolaccessibility.go`**, **`utilities.go`** → add to any switch that handles the analogous node

*Questions to ask:* What type does my node produce? Does it have type parameters? Can it have decorators? Does it introduce control flow? Is it a declaration space? Does it participate in overload resolution?

*When this module can be skipped:* If the node reuses an existing declaration shape (same scoping rules, same type inference pattern, no new control flow), the checker may only need switch-case additions and no new checker functions.

---

**Printer / Emitter** (`internal/printer/`)

*Role:* Converts AST back to text (JS emit, declaration emit, source maps).

*Changes needed when:*
- Adding any **non-internal node** that appears in output → write an `emit<Name>()` function
- Add to dispatch switch: `emitStatement()`, `emitExpression()`, `emitDeclaration()`, or equivalent
- Trailing comma handling → `shouldAllowTrailingComma()`, `hasTrailingComma()`
- Name generation → `generateNames()`, `generateNameIfNeeded()`

*Questions to ask:* Does my node appear in emitted JavaScript? In `.d.ts` declarations? Does it have type parameters that should be emitted (or erased)?

---

**Transformers** (`internal/transformers/`)

*Role:* AST-to-AST transformations for downlevel emit and type erasure.

*Changes needed when:*
- **Type erasure** (`tstransforms/typeeraser.go`) — add case to visit method to strip type annotations (erase type parameters, type references)
- **ES transforms** (`estransforms/`) — if the node needs downlevel transformation for older JS targets (e.g., class fields, decorators, async/await)
- **Declaration transforms** (`declarations/`) — if the node appears in `.d.ts` output and needs declaration-specific transformation

*Questions to ask:* Does my node contain type annotations that should be erased for JS emit? Does it need downlevel transformation for older ES targets?

*When this module can be skipped:* If the node's runtime semantics are identical to an existing node (same type erasure, same ES transform behavior), you may only need to add a case to `typeeraser.go`. Nodes with unique runtime semantics (new ES features, new decorator shapes, new declaration patterns) need custom transform logic.

---

**Formatter** (`internal/format/`)

*Role:* Code formatting (indentation, line breaks, spacing).

*Changes needed when:*
- Adding a **node that appears in formatted output** → add to indentation rules (`getListByRange()`, `NodeWillIndentChild()`), formatting rule context (`rulecontext.go`)

*Questions to ask:* How should my node be indented? Same as the analogous node, or differently?

---

**Language Service / LSP** (`internal/ls/`)

*Role:* Editor features — semantic tokens, completions, go-to-definition, find references, folding, etc.

*Changes needed when:*
- Adding a **declaration, statement, or expression** → add to `switch node.Kind` in most LS feature files
- **`semantictokens.go`** — map the new kind to a token type for syntax highlighting
- **`completions.go`** — if the node offers completions (e.g., members inside a struct/class body)
- **`symbols.go`** — if the node should appear in document/workspace symbol searches
- **`folding.go`** — if the node's body can be folded in the editor
- **`callhierarchy.go`**, **`findallreferences.go`**, **`documenthighlights.go`** — if the node participates in call/reference chains
- **`codelens.go`**, **`inlay_hints.go`** — if the node should show code lens or inlay hints
- **`codeactions_fixmissingtypeannotation.go`** — if the node can trigger fix-it code actions

*Questions to ask:* Should this node be highlighted differently? Can you fold its body? Does it appear in symbol search? Do its members get completions?

*When this module can be skipped:* If the node introduces a completely new kind of scope or body that needs unique completion providers, new LS logic is required. Nodes that share body semantics with an existing node (e.g., same member completions, same folding rules) only need switch-case additions.

---

**Diagnostics** (`internal/diagnostics/`)

*Role:* Error and warning message definitions.

*Changes needed when:*
- Adding a **node with new error conditions** → add message definitions to `extraDiagnosticMessages.json`
- Run `npx hereby generate` to regenerate `diagnostics_generated.go` from the updated JSON

*Questions to ask:* Can this node be used incorrectly? What errors should the checker report?

---

**Tests** (`internal/fourslash/tests/`, `testdata/tests/cases/compiler/`)

*Role:* Integration tests for LSP features and baseline tests for compiler output.

*Changes needed when:*
- **Compiler baseline tests** — create a `.ts` source file under `testdata/tests/cases/compiler/`. Run tests to generate baseline files (`.js`, `.symbols`, `.types`, etc.). Accept baselines with `npx hereby baseline-accept`.
- **Fourslash tests** — create test files under `internal/fourslash/tests/` for LSP features (completions, quick info, go-to-definition, semantic tokens, etc.)
- **Test utilities** — update `internal/fourslash/tests/util/util.go` if the test framework needs to know about the new kind

#### 2.3.4 What People Often Forget

Based on real PRs, these are commonly missed:

| What | Where | Symptom if missed |
|------|-------|-------------------|
| Kind classification lists | `internal/ast/utilities.go` | Node not recognized as a declaration/statement; binder and checker skip it |
| Keyword→Kind mapping | `internal/scanner/scanner.go` | Parser never sees the keyword; node can't be parsed |
| `isStartOfStatement()` / `scanStartOfDeclaration()` | `internal/parser/parser.go` | Parser doesn't know when to enter the new production |
| `GetErrorRangeForNode` | `internal/scanner/scanner.go` | Error squiggles cover the wrong range |
| Trailing comma rules | `internal/printer/printer.go` | Formatter crashes or emits wrong commas |
| Formatting indent rules | `internal/format/indent.go` | Formatter crashes on the new node |
| Symbol search kind switch | `internal/ls/symbols.go` | Node invisible in "Go to Symbol" |
| Folding kind switch | `internal/ls/folding.go` | Node body can't be collapsed in editor |
| Semantic token mapping | `internal/ls/semantictokens.go` | Node has wrong/no syntax highlighting |
| Declaration space | `internal/checker/checker.go` (`getDeclarationSpaces`) | Wrong collision detection; import/export errors |
| Transformer type erasure | `internal/transformers/tstransforms/typeeraser.go` | Type annotations leak into JS output |
| Test utility kind switch | `internal/fourslash/tests/util/util.go` | Fourslash tests can't find the new node |
| Enum generation | `Herebyfile.mjs` (`generate:enums`) | SyntaxKind not available in TS client |
| Marker range updates | `_scripts/ast.json` (`kinds.markers`) | `IsTokenKind`-style guards miss the new kind |
| Scoping fallthrough | `internal/scanner/scanner.go` (`GetErrorRangeForNode` → `fallthrough`) | Error ranges for parent nodes break when child is the new kind |
