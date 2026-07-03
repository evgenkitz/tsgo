# Issue #5 — `struct` declaration parsing

## Context

ArkTS introduces `struct` — a top-level declaration similar to `class` but with a virtual
constructor and optional virtual `extends` heritage. This plan covers parsing `struct`
declarations in `.ets` files, producing `StructDeclaration` AST nodes (already available
from Phase 1 codegen).

A key feature of structs: the parser synthesizes a **virtual constructor** with two
parameters — `value?: { ...collected properties... }` and `##storage?: LocalStorage` —
based on the struct's `PropertyDeclaration` members. When no explicit `extends` clause
is present, a **virtual heritage clause** (`extends CustomComponent`) may also be injected
based on compiler options.

## Prerequisites

- **EtsFlags system** (Issue #3) — implemented ✓ (specifically `setStructContext`/`inStructContext`)
- **AST node** (`StructDeclaration`) — Phase 1 ✓
- **`ScriptKindETS`** and **`NodeFlagsEtsContext`** — Phase 1 ✓

## Affected files

| File | What |
|------|------|
| `internal/parser/arkts.go` | All new parsing functions (~170 lines) |
| `internal/parser/parser.go` | Routing edits (~35 lines in 8 functions) + 1 new field |

## Out of scope

- `extendEtsComponentDeclaration`, `stylesEtsComponentDeclaration`, `stateStylesRootNode`, `currentNodeName`, `fileStylesComponents`, `firstArgumentExpression`, `repeatEachRest` — these Parser fields are needed for later phases (Issue #6), not for basic struct parsing
- `NodeFlagsVirtual` / virtual node flag — virtual constructor and heritage clause can initially be finished with `p.finishNode()` without a special flag; the virtual flag is primarily consumed by the checker (separate task)
- Binder, checker, emitter — not in this phase
- Annotation declarations (Issue #4)
- ETS component expressions (Issue #7)
- Function/decorator modifications (Issue #6)

## Implementation steps

### Step 1: Parser struct field for struct member tracking

**In `parser.go`**, add to the `Parser` struct (near line 106, after `etsFlags`):

```go
structStylesComponents *collections.Set[string]
```

This set tracks `@Styles` component names collected during struct member parsing.
It is cleared after the struct body is parsed (in `parseStructDeclarationOrExpression`).

The set is automatically zeroed (nil) when `putParser` returns the struct to the pool.

### Step 2: `parseStructDeclaration()` — thin wrapper

**Reference:** ohos-typescript `parser.ts:8670-8672`

**In `arkts.go`**, add `parseStructDeclaration(pos int, jsdoc jsdocScannerInfo, modifiers *ast.ModifierList) *ast.Node`:

```go
return p.parseStructDeclarationOrExpression(pos, jsdoc, modifiers)
```

### Step 3: `parseStructDeclarationOrExpression()` — main logic

**Reference:** ohos-typescript `parser.ts:8701-8730`

**In `arkts.go`**, add `parseStructDeclarationOrExpression(pos int, jsdoc jsdocScannerInfo, modifiers *ast.ModifierList) *ast.Node`:

Follows the same save/restore pattern as `parseClassDeclarationOrExpression` (`parser.go:1767`):

1. `saveContextFlags := p.contextFlags` — save all context flags (not just await; Go has no `inAwaitContext()` method)
2. `p.parseExpected(KindStructKeyword)`
3. `p.setStructContext(true)`
4. `name := p.parseNameOfClassDeclarationOrExpression()` — reuses class name logic (checks `isBindingIdentifier() && !isImplementsClause()`)
5. `typeParameters := p.parseTypeParameters()` — reuses existing
6. `if modifiers != nil && core.Some(modifiers.Nodes, isExportModifier)` → `p.setContextFlags(NodeFlagsAwaitContext, true)`
7. `heritageClauses := p.parseHeritageClauses()` — reuses existing
8. `// TODO: virtual heritage clause injection when compilerOptions.ets.customComponent is available`
9. `members`: if `parseExpected(KindOpenBraceToken)` → `p.parseStructMembers(pos)` + `parseExpected(KindCloseBraceToken)`, else `createMissingList()`
10. `p.contextFlags = saveContextFlags` (restore all, same as class)
11. `p.structStylesComponents = nil` (clear)
12. `p.setStructContext(false)`
13. `result := p.factory.NewStructDeclaration(modifiers, name, typeParameters, heritageClauses, members)`
14. `p.finishNode(result, pos)` + `p.withJSDoc(result, jsdoc)`

### Step 4: `parseStructMembers()` — virtual constructor synthesis

**Reference:** ohos-typescript `parser.ts:8806-8849`

**In `arkts.go`**, add `parseStructMembers(pos int) *ast.NodeList`:

1. `structMembers := p.parseList(PCClassMembers, (*Parser).parseClassElement)` — reuses class member parsing
2. Collect `PropertyDeclaration` nodes from `structMembers`:
   - For each member with `kind == KindPropertyDeclaration`:
     - Create a `PropertySignatureDeclaration` (Go factory: `NewPropertySignatureDeclaration`) with `QuestionToken`, using the property's name, modifiers, and type; nil initializer
     - Add to `virtualParameterProperties` slice
3. Build virtual constructor parameters:
   - If there are property signatures:
     - Create a `TypeLiteralNode` from the property signatures
     - Create parameter `value?: { ...props... }` — `p.factory.NewIdentifier("value")` with `QuestionToken` (`p.factory.NewToken(KindQuestionToken)`) and the type literal
   - Always add parameter `##storage?: LocalStorage` — `p.factory.NewIdentifier("##storage")` with `QuestionToken` and `p.factory.NewTypeReferenceNode(p.factory.NewIdentifier("LocalStorage"), nil)`
4. Build virtual constructor:
   - Empty body block: `p.factory.NewBlock(p.parseEmptyNodeList(), false)`
   - `p.factory.NewConstructorDeclaration(nil /*modifiers*/, nil /*typeParameters*/, parameters, nil /*typeNode*/, nil /*fullSignature*/, emptyBody)` — note: **6 parameters**, not 3
5. Prepend constructor to members via `unshift` (in Go: `make([]*ast.Node, 0, len(structMembers.Nodes)+1)`, append constructor first, then copy original members; wrap with `p.newNodeList(core.NewTextRange(pos, p.nodePos()), virtualMembers)`)
6. Return the combined node list

**Virtual node creation:** For now, use `p.finishNode(node, pos)` (standard finish) for all
synthesized nodes. The `NodeFlagsVirtual` flag can be added later when the checker needs it.
The `NodeFlagsSynthesized` flag (bit 4) should NOT be used — it has a specific meaning for
the transformation phase.

### Step 5: `createVirtualHeritageClauses()` — when needed

**Reference:** ohos-typescript `parser.ts:8732-8742`

**In `arkts.go`**, add `createVirtualHeritageClauses(customComponent string) *ast.NodeList`:

Creates `[HeritageClause(ExtendsKeyword, [ExpressionWithTypeArguments(Identifier(customComponent))])]`.

All nodes finished with `p.finishNode()` (standard, no virtual flag for now).

This function is called when:
- No explicit `extends`/`implements` clause was parsed
- AND `customComponent` compiler option is set (not yet available — add a TODO/placeholder)

### Step 6: Three struct context helper functions

**Reference:** ohos-typescript `parser.ts:8095-8115`

**In `arkts.go`**, add:

1. **`isTokenInsideStructBuild(methodName *ast.Node) bool`:**
   - If `methodName.Kind == KindIdentifier && methodName.AsIdentifier().Name() == "build"` → return `true`
   - Note: reference tsc reads the build method name from `compilerOptions.ets.render.method`; simplified version hardcodes `"build"`

2. **`isTokenInsideStructBuilder(decorators *ast.ModifierList) bool`:**
   - Delegates to `isTokenInsideBuilder(decorators)` — this is an ohApi helper implemented in Issue #6
   - For now: create a stub that returns `false` (will be wired later)

3. **`isTokenInsideStructPageTransition(methodName *ast.Node) bool`:**
   - If `methodName.Kind == KindIdentifier && methodName.AsIdentifier().Name() == "pageTransition"` → return `true`
   - Simplified version hardcodes `"pageTransition"` (reference reads from compiler options)

These helpers are used during `parseClassElement` for class members inside struct bodies
(Issue #6), but must live in `arkts.go` for the struct parsing package.

### Step 7: Routing edits in `parser.go`

Each edit is minimal — a few lines that call out to `arkts.go` functions.

#### 7a. `parseStatement` (line 1071)

Add `KindStructKeyword` case after `KindClassKeyword`:
```go
case ast.KindStructKeyword:
    if p.inEtsContext() {
        return p.parseStructDeclaration(p.nodePos(), p.jsdocScannerInfo(), nil /*modifiers*/)
    }
```

#### 7b. `parseDeclarationWorker` (line 1159)

Add `KindStructKeyword` case after `KindClassKeyword`:
```go
case ast.KindStructKeyword:
    if p.inEtsContext() {
        return p.parseStructDeclaration(pos, jsdoc, modifiers)
    }
```

#### 7c. `isStartOfStatement` (line 6068)

Add `KindStructKeyword` to the case list (alongside `KindClassKeyword`):

```go
case ast.KindAtToken, ast.KindSemicolonToken, ast.KindOpenBraceToken, ast.KindVarKeyword, ast.KindLetKeyword,
    ast.KindUsingKeyword, ast.KindFunctionKeyword, ast.KindClassKeyword, ast.KindStructKeyword, ast.KindEnumKeyword,
    ...
```

Since the scanner only emits `KindStructKeyword` in ETS context (`ScriptKindETS`), and in `.ts`
files `struct` is scanned as an identifier, adding `KindStructKeyword` unconditionally is safe —
the case simply never fires outside ETS context.

#### 7d. `scanStartOfDeclaration` (line 6101)

Add `KindStructKeyword` case:
```go
case ast.KindStructKeyword:
    return p.inEtsContext()
```

#### 7e. `nextTokenCanFollowDefaultKeyword` (line 4000)

Add `KindStructKeyword` alongside `KindClassKeyword`:
```go
func (p *Parser) nextTokenCanFollowDefaultKeyword() bool {
    switch p.nextToken() {
    case ast.KindClassKeyword, ast.KindStructKeyword, ast.KindFunctionKeyword, ast.KindInterfaceKeyword, ast.KindAtToken:
        return true
    case ast.KindAbstractKeyword:
        return p.lookAhead((*Parser).nextTokenIsClassKeywordOnSameLine)
    case ast.KindAsyncKeyword:
        return p.lookAhead((*Parser).nextTokenIsFunctionKeywordOnSameLine)
    }
    return false
}
```

Since the scanner only emits `KindStructKeyword` in ETS context, no explicit `inEtsContext()`
gate is needed in this function.

#### 7f. `isStartOfExpressionStatement` (line 4524)

**Reference:** ohos-typescript `parser.ts:5242-5250`

Add `KindStructKeyword` guard so that `struct` is not treated as the start of an expression statement:

```go
func (p *Parser) isStartOfExpressionStatement() bool {
    return p.token != ast.KindOpenBraceToken &&
        p.token != ast.KindFunctionKeyword &&
        p.token != ast.KindClassKeyword &&
        (!p.inEtsContext() || p.token != ast.KindStructKeyword) &&  // ← added
        p.token != ast.KindAtToken &&
        p.isStartOfExpression()
}
```

#### 7g. `parseArrowFunctionExpressionBody` (line 4499)

**Reference:** ohos-typescript `parser.ts:5771-5777`

Add `KindStructKeyword` to the list of non-statement tokens checked before the expression-body recovery path:

```go
if p.token != ast.KindSemicolonToken &&
    p.token != ast.KindFunctionKeyword &&
    p.token != ast.KindClassKeyword &&
    (!p.inEtsContext() || p.token != ast.KindStructKeyword) &&  // ← added
    p.isStartOfStatement() &&
    !p.isStartOfExpressionStatement() {
```

#### 7h. `isStartOfLeftHandSideExpression` (line 6230)

**Reference:** ohos-typescript `parser.ts:5188-5192`

Add `KindStructKeyword` case to the switch. In ETS context, `struct` is a valid start of a left-hand-side expression (it produces "Expression expected" in `parsePrimaryExpression` — intentional error recovery matching reference tsc behaviour):

```go
case ast.KindStructKeyword:
    return p.inEtsContext()
```

## Verification

1. **Build:** `go build ./internal/parser/...` — compiles without errors
2. **Parser round-trip tests:**
   - `struct Foo { }` — minimal struct, no members
   - `struct Bar extends Base { prop: string; }` — with explicit extends, virtual constructor has one property
   - `struct Baz<T> extends Base<T> { @State count: number; build() { } }` — type parameters, heritage, decorated property, build method
   - `struct Empty { }` → virtual constructor with only `##storage?: LocalStorage` parameter
   - `struct` keyword in `.ts` file → parsed as identifier (VariableStatement), NOT StructDeclaration
3. **Virtual constructor verification:**
   - Parse a struct with properties → verify `ConstructorDeclaration` is first member
   - Verify `value?` parameter contains a type literal with matching property signatures
   - Verify `##storage?` parameter with type `LocalStorage`
4. **Existing tests:** `npx hereby test` — zero regressions in existing TS/TSX tests
5. **Lint:** `npx hereby lint`

## References

| Reference tsc file | Lines | Content |
|---|---|---|
| `parser.ts` | 8670–8672 | `parseStructDeclaration` wrapper |
| `parser.ts` | 8701–8730 | `parseStructDeclarationOrExpression` |
| `parser.ts` | 8732–8742 | `createVirtualHeritageClauses` |
| `parser.ts` | 8806–8849 | `parseStructMembers` |
| `parser.ts` | 8851–8853 | `finishVirtualNode` |
| `parser.ts` | 8095–8115 | Struct context helpers (build/builder/pageTransition) |
| `parser.ts` | 7461–7474 | `isDeclaration` — `StructKeyword` guard |
| `parser.ts` | 7578–7579 | `parseStatement` — `StructKeyword` guard |
| `parser.ts` | 7790–7810 | `parseDeclarationWorker` routing |
| `parser.ts` | 3018–3024 | `nextTokenCanFollowDefaultKeyword` |
| `parser.ts` | 8744–8753 | `parseNameOfClassDeclarationOrExpression` (reused) |
| `parser.ts` | 8674–8699 | `parseClassDeclarationOrExpression` (model for struct) |
| `parser.ts` | 5190 | `isStartOfLeftHandSideExpression` — `StructKeyword` guard |
| `parser.ts` | 5242–5250 | `isStartOfExpressionStatement` — `StructKeyword` guard |
| `parser.ts` | 5769–5777 | `parseArrowFunctionExpressionBody` — `StructKeyword` guard |

## Requirement Scenario

**Scenario 1: Basic struct declaration**
An ArkTS developer writes `struct MyComponent { message: string = "Hello"; }` in a `.ets` file. The parser produces a `StructDeclaration` node with a synthesized virtual constructor (`constructor(value?: { message: string; }, ##storage?: LocalStorage) {}`). If `EtsCustomComponent` is configured (e.g. `"CustomComponent"`), a virtual `extends CustomComponent` heritage clause is injected.

**Scenario 2: Struct with explicit extends**
`struct MyPage extends ParentComponent { title: string; }` — the explicit `extends ParentComponent` takes priority over the virtual heritage clause. No virtual heritage is injected.

**Scenario 3: Struct as identifier outside ETS context**
In a `.ts` file, `const struct = 1;` — `struct` is parsed as a regular identifier, not a keyword. Parser routing checks `inEtsContext()` before treating `struct` as a declaration keyword.

## Target Users

- **ArkTS component developers** — write `.ets` files with `struct` declarations for UI component definitions
- **Compiler developers** — maintain struct parsing logic and virtual constructor synthesis

## Acceptance Strategy

| Criterion | Verification |
|-----------|-------------|
| `struct` keyword parsed as declaration in .ets | `arkts_test.go`: struct parsing tests |
| Virtual constructor synthesized with correct params | Golden test files in `testdata/arkts/` |
| Explicit `extends` overrides virtual heritage | `TestStructExplicitExtendsTakesPriority` |
| `struct` is identifier in .ts files | `TestStructIsIdentifierNotKeywordInTs` |
| Golden file tests match expected output | All `testdata/arkts/struct_*.ets` files produce correct AST |
| AST and diagnostics match reference tsc | `arkts_cmp_test.go` compares against reference for struct test cases in `arkTSTest/testcase/` |
