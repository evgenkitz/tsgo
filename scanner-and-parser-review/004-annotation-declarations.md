# Issue #4 — @interface annotation declarations

## Context

ArkTS introduces `@interface` — an annotation declaration syntax (`@interface Name { ... }`)
used to define custom decorator/annotation types. This plan covers parsing `@interface`
declarations in `.ets` files, producing `AnnotationDeclaration` and
`AnnotationPropertyDeclaration` AST nodes (already available from Phase 1 codegen).

## Prerequisites

- **EtsFlags system** (Issue #3) — implemented ✓
- **AST nodes** (`AnnotationDeclaration`, `AnnotationPropertyDeclaration`) — Phase 1 ✓
- **`ScriptKindETS`** and **`NodeFlagsEtsContext`** — Phase 1 ✓
- Parser routing: `parseStatement` already sends `KindAtToken` to `parseDeclaration()` ✓

## Affected files

| File | What |
|------|------|
| `internal/parser/arkts.go` | All new parsing functions (~120 lines) |
| `internal/parser/parser.go` | Routing edits (~20 lines in 4 functions) |

## Out of scope

- `etsAnnotationsEnable` compiler option — simplified initial port always allows annotations in `.ets` files; the option can be added later
- `isArkguardInputSourceFile` — not needed in Go port
- Binder, checker, emitter — not in this phase
- Struct declarations (Issue #5)
- ETS component expressions (Issue #7)
- Function/decorator modifications (Issue #6)

## Implementation steps

### Step 1: Guard decorator parsing against `@interface` consumption

**Reference:** ohos-typescript `parser.ts:8416-8426`

The critical problem: when `parseModifiersEx` encounters `@interface`, it unconditionally
treats `@` as a decorator start, consuming `@` and then trying to parse `interface` as a
decorator expression — which fails.

In the reference tsc, `tryParseDecorator()` checks: if `@` is followed by `interface` in
annotation context → return `undefined` (not a decorator). We need the same guard in Go.

**In `arkts.go`**, add `tryParseDecorator() *ast.Node`:
1. If `inAllowAnnotationContext()` && current token is `KindAtToken` && `lookAhead(nextTokenIsInterfaceKeyword)`:
   - Return `nil` (not a decorator)
2. Otherwise: parse and return the decorator (wrap existing `parseDecorator` logic)

**In `parser.go`**, modify `parseModifiersEx` (line 3883-3886):
Replace the inline `p.parseDecorator()` call with `p.tryParseDecorator()`. If it returns
`nil`, break out of the decorator loop (the `@` will be consumed by `parseAnnotationDeclaration` instead).

**In `arkts.go`**, add `nextTokenIsInterfaceKeyword(p *Parser) bool`:
```
return p.nextToken() == ast.KindInterfaceKeyword
```

### Step 2: `inAllowAnnotationContext()` helper

**Reference:** ohos-typescript `parser.ts:2323-2325`

**In `arkts.go`**, add:
```go
func (p *Parser) inAllowAnnotationContext() bool {
    return p.inEtsContext()
}
```

Simplified from the reference tsc (which also checks `etsAnnotationsEnable` option).
The option can be added in a follow-up task.

### Step 3: `parseAnnotationDeclaration()`

**Reference:** ohos-typescript `parser.ts:7706-7729`

**In `arkts.go`**, add `parseAnnotationDeclaration(pos int, jsdoc jsdocScannerInfo, modifiers *ast.ModifierList) *ast.Node`:

1. Record `atTokenPos` from scanner
2. `parseExpected(KindAtToken)`
3. Record `interfaceTokenPos`
4. `parseExpected(KindInterfaceKeyword)`
5. Validate: if `interfaceTokenPos - atTokenPos > 1` → error `In_annotation_declaration_any_symbols_between_and_interface_are_forbidden` (diagnostic may need to be added, or use a placeholder)
6. `name := p.createIdentifier(p.isBindingIdentifier())`
7. `members`: if `parseExpected(KindOpenBraceToken)` → `parseAnnotationMembers()` + `parseExpected(KindCloseBraceToken)`, else `createMissingList()`
8. `result := p.factory.NewAnnotationDeclaration(modifiers, name, members)`
9. `p.finishNode(result, pos)` + `p.withJSDoc(result, jsdoc)`

### Step 4: `parseAnnotationPropertyDeclaration()`

**Reference:** ohos-typescript `parser.ts:8188-8201`

**In `arkts.go`**, add `parseAnnotationPropertyDeclaration(pos int, jsdoc jsdocScannerInfo, name *ast.Node) *ast.Node`:

1. `typeNode := p.parseTypeAnnotation()`
2. `initializer`: inline the `doOutsideOfContext` logic (Go doesn't have this helper yet):
   ```go
   saveContextFlags := p.contextFlags
   p.contextFlags &^= ast.NodeFlagsYieldContext | ast.NodeFlagsAwaitContext | ast.NodeFlagsDisallowInContext
   initializer := p.parseInitializer()
   p.contextFlags = saveContextFlags
   ```
3. `p.parseSemicolonAfterPropertyName(name, typeNode, initializer)` — reuses existing (signature: `func (p *Parser) parseSemicolonAfterPropertyName(name *ast.Node, typeNode *ast.TypeNode, initializer *ast.Expression)`)
4. `result := p.factory.NewAnnotationPropertyDeclaration(name, typeNode, initializer)`
5. `p.finishNode(result, pos)` + `p.withJSDoc(result, jsdoc)`

Note: the reference tsc signature is `parseAnnotationPropertyDeclaration(pos, hasJSDoc, name)` — no decorators or modifiers.
Annotation properties are simpler than regular property declarations.

### Step 5: `parseAnnotationElement()` — member dispatcher

**Reference:** ohos-typescript `parser.ts:8621-8660`

**In `arkts.go`**, add `parseAnnotationElement() *ast.Node`:

Switch on current token:
- `KindSemicolonToken` → error `Unexpected_keyword_or_identifier`
- `KindStaticKeyword` + lookahead `nextTokenIsOpenBrace` → `createMissingNode(AnnotationPropertyDeclaration, ...)`
- `KindGetKeyword` (contextual) → `createMissingNode(AnnotationPropertyDeclaration, ...)`
- `KindSetKeyword` (contextual) → `createMissingNode(AnnotationPropertyDeclaration, ...)`
- `KindConstructorKeyword`, `KindStringLiteral` → `createMissingNode(AnnotationPropertyDeclaration, ...)`
- `isIndexSignature()` → `createMissingNode(AnnotationPropertyDeclaration, ...)`
- If `tokenIsIdentifierOrKeyword()` → `parsePropertyName()` then `parseAnnotationPropertyDeclaration(pos, hasJSDoc, name)`
- Default → `panic` (should never reach here because `isAnnotationMemberStart` prevents it)

### Step 6: `parseAnnotationMembers()` and `isAnnotationMemberStart()`

**Reference:** ohos-typescript `parser.ts:8802-8804, 8256-8306`

**In `arkts.go`**, add `parseAnnotationMembers() *ast.NodeList`:
```go
return p.parseList(PCAnnotationMembers, (*Parser).parseAnnotationElement)
```

**In `arkts.go`**, add `isAnnotationMemberStart() bool`:

1. If `p.token == KindAtToken` → return `false`
2. If `ast.IsModifierKind(p.token)` → return `false`
3. If `p.token == KindAsteriskToken` → return `false`
4. If `p.isLiteralPropertyName()`:
   - Save `idToken := p.token`, call `p.nextToken()`
5. If `p.token == KindOpenBracketToken` → return `false` (no index signatures in annotations)
6. If `idToken` was saved and `ast.IsKeyword(idToken)` → return `false`
7. If `idToken` was saved:
   - `KindColonToken` or `KindEqualsToken` → return `true`
   - Otherwise → return `p.canParseSemicolon()`
8. Return `false`

Note: `canParseSemicolon()` already exists in the Go parser (`parser.go:6043`).

### Step 7: Wire `isAnnotationMemberStart` into list termination

**In `arkts.go`**, add `isAnnotationMemberStart` as a list-element-start check.
The `parseList` mechanism needs to know when an annotation member starts vs. when
the list should terminate. In the reference tsc, `isListElement` already calls
`isAnnotationMemberStart` for `ParsingContext.AnnotationMembers`.

**In `parser.go`**, add a case in `isListElement` for `PCAnnotationMembers`:
```go
case PCAnnotationMembers:
    return p.isAnnotationMemberStart()
```

### Step 8: Routing edits in `parser.go`

#### 8a. `parseDeclarationWorker` (line 1159)

Add `KindAtToken` case before the fallthrough/default:
```go
case ast.KindAtToken:
    if p.inAllowAnnotationContext() &&
        lookAhead((*Parser).nextTokenIsInterfaceKeyword) {
        return p.parseAnnotationDeclaration(pos, jsdoc, modifiers)
    }
    return p.parseDeclarationDefault(pos, modifiers)
```

Note: need to import `lookAhead` pattern — this is a method call. Use `p.lookAhead(...)`.

#### 8b. `scanStartOfDeclaration` (line 6101)

Add `KindAtToken` case in the switch:
```go
case ast.KindAtToken:
    return p.inAllowAnnotationContext() &&
        p.nextToken() == ast.KindInterfaceKeyword
```

#### 8c. `canFollowModifier` (line 4045)

Add `KindAtToken` condition so that modifiers (like `export`) can precede `@interface`:
```go
func (p *Parser) canFollowModifier() bool {
    return p.token == ast.KindOpenBracketToken ||
        p.token == ast.KindOpenBraceToken ||
        p.token == ast.KindAsteriskToken ||
        p.token == ast.KindDotDotDotToken ||
        (p.token == ast.KindAtToken && p.inAllowAnnotationContext() &&
            p.lookAhead((*Parser).nextTokenIsInterfaceKeyword)) ||
        p.isLiteralPropertyName()
}
```

### Step 9: Wire `PCAnnotationMembers` ParsingContext

**In `parser.go`**, add `PCAnnotationMembers` to the `ParsingContext` iota enum (after `PCHeritageClauses`).

No explicit initialization is needed — `parsingContexts` is set per-`parseList` call.

### Step 10: Add error diagnostic (if needed)

Check if `In_annotation_declaration_any_symbols_between_and_interface_are_forbidden` diagnostic
exists in `internal/diagnostics`. If not, add it via the diagnostic codegen workflow.

## References

| Reference tsc file | Lines | Content |
|---|---|---|
| `parser.ts` | 2323–2325 | `inAllowAnnotationContext` |
| `parser.ts` | 7461–7474 | `isDeclaration` guards |
| `parser.ts` | 7706–7729 | `parseAnnotationDeclaration` |
| `parser.ts` | 7790–7810 | `parseDeclarationWorker` routing |
| `parser.ts` | 8188–8201 | `parseAnnotationPropertyDeclaration` |
| `parser.ts` | 8256–8306 | `isAnnotationMemberStart` |
| `parser.ts` | 8416–8426 | `tryParseDecorator` guard |
| `parser.ts` | 8621–8660 | `parseAnnotationElement` |
| `parser.ts` | 8802–8804 | `parseAnnotationMembers` |
| `parser.ts` | 3009–3016 | `canFollowModifier` |

## Result

### Implemented

**New functions in `internal/parser/arkts.go`:**
- `nextTokenIsInterfaceKeyword()` — lookAhead callback: `nextToken() == KindInterfaceKeyword`
- `inAllowAnnotationContext()` — ETS context + `EtsAnnotationsEnable` compiler option check
- `isAtAnnotationDeclaration()` — shared guard: `inAllowAnnotationContext() && token==@ && lookahead(interface)`
- `isAtTokenNotClassMember()` — guard for `scanClassMemberStart`: prevents `@interface` inside class body from being treated as a decorator class member
- `tryParseDecorator()` — prevents `@interface` from being consumed as a decorator; returns `nil` when annotation context is detected
- `parseAnnotationDeclaration()` — `@interface Name { members }`
- `parseAnnotationPropertyDeclaration()` — `name: Type = initializer`
- `parseAnnotationElement()` — single-member dispatcher (identifier → success; else → recover)
- `parseAnnotationMembers()` — `parseList(PCAnnotationMembers, ...)`
- `isAnnotationMemberStart()` — list gate: only plain identifiers (not literals)
- `createMissingAnnotationProperty()` — error recovery helper

**Edits in `internal/parser/parser.go`:**
- `PCAnnotationMembers` in `ParsingContext` enum
- `isListTerminator`: `PCAnnotationMembers` → CloseBraceToken
- `parsingContextErrors`: `PCAnnotationMembers` → diagnostic 28018
- `isListElement`: `PCAnnotationMembers` → `lookAhead(isAnnotationMemberStart)`
- `parseDeclarationWorker`: `KindAtToken` → annotation routing
- `parseModifiersEx`: `parseDecorator()` → `tryParseDecorator()` with nil guard
- `scanStartOfDeclaration`: `KindAtToken` case
- `canFollowModifier`: `@interface` condition
- `parseDecoratedExpression`: nil-modifiers guard — when `tryParseDecorator` returns nil for `@interface`, produce `parseIdentifierWithDiagnostic(Expression_expected, nil)` instead of `MissingDeclaration`
- `scanClassMemberStart`: `isAtTokenNotClassMember()` call at top — prevents `@interface` inside class body from starting class member parsing

**Compiler option `etsAnnotationsEnable`:**
- `internal/core/compileroptions.go` — `EtsAnnotationsEnable bool` field added to `CompilerOptions` struct
- `internal/ast/parseoptions.go` — `EtsAnnotationsEnable bool` field added to `SourceFileParseOptions` struct
- `internal/compiler/fileloader.go` — passed from compiler options into parse options
- `internal/tsoptions/declscompiler.go` — CLI option registration (`--etsAnnotationsEnable`, boolean, affects semantic/emit/buildInfo/sourceFile)
- `internal/tsoptions/parsinghelpers.go` — `etsAnnotationsEnable` case added to `parseCompilerOptions`

**Diagnostics:**
- Code 28037 — `"In annotation declaration any symbols between '@' and 'interface' are forbidden."`
- Code 28018 — `"Unexpected token. An annotation property was expected."`
- Code 28038 — `"Enable support of ETS annotations."` (category: Message, used as `etsAnnotationsEnable` option description)

**Tests (`internal/parser/arkts_test.go`):** 4 unit tests (all pass):
- `TestParseAnnotationDeclaration_NotInEtsContext` — `.ts` context ignores `@interface`
- `TestParseAnnotationDeclaration_EtsAnnotationsDisabled` — `.ets` with `EtsAnnotationsEnable=false` ignores `@interface`
- `TestParseAnnotationElement_NumericLiteralPropertyName` — numeric literal names produce 28018 (no panic)
- `TestParseAnnotationElement_StringLiteralPropertyName` — string literal names produce 28018 (no panic)

**Testdata (`testdata/arkts/`):** 15 `.ets` files for golden/baseline tests:
- Positive cases: `annotation_empty`, `annotation_exported`, `annotation_exported_declare.d`, `annotation_with_typed_prop`, `annotation_with_multi_prop`, `annotation_missing_body`, `annotation_missing_name`, `annotation_property_without_type`
- Error-recovery cases (28018): `annotation_diag_28018_constructor`, `annotation_diag_28018_get_modifier`, `annotation_diag_28018_multi`, `annotation_diag_28018_semicolon`, `annotation_diag_28018_static_block`
- Contiguity-error cases (28037): `annotation_diag_28037_comment`, `annotation_diag_28037_space`

### Deviations from reference tsc

- `inAllowAnnotationContext()` checks both `inEtsContext()` and `EtsAnnotationsEnable` compiler option.
  The plan originally deferred the option check, but it was implemented — without it, annotations
  would parse even when explicitly disabled.
- `isAnnotationMemberStart` uses `tokenIsIdentifierOrKeyword` instead of
  `isLiteralPropertyName` — literal property names (string/numeric/bigint) are not
  valid annotation property names; the reference tsc has a bug where numeric literals
  would hit `Debug.fail`
- `parseAnnotationElement` uses a recovery safety net (`nextToken() + missing node`)
  instead of `panic` for unexpected tokens — the reference tsc uses `Debug.fail`
- No top-level check in parser (deferred to checker — `checkGrammarAnnotationDeclaration`)

### Bugs found and fixed during implementation

- P0: Missing `PCAnnotationMembers` in `isListTerminator` → panic on `}` in annotation body
- P0: Missing `PCAnnotationMembers` in `parsingContextErrors` → panic on unexpected tokens
- P1: Semicolon in annotation body → panic (missing return in error path)
- P1: Numeric/bigint literal property names → panic (`tokenIsIdentifierOrKeyword` gate)
- Infinite loop: `isAnnotationMemberStart` called via `lookAhead` doesn't consume token →
  `parseAnnotationElement` must advance past invalid tokens

### Cleanup

- Extracted `isAtAnnotationDeclaration()` to deduplicate repeated guard condition
- Extracted `createMissingAnnotationProperty()` to deduplicate 5 identical error-recovery blocks
- Used `doInContext` instead of manual `contextFlags` save/restore
- Moved `jsdocScannerInfo()` call inside success branch (not computed on error paths)

## Requirement Scenario

**Scenario 1: Defining a custom decorator type**
An ArkTS developer writes `@interface CustomProp { label: string = ""; }` in a `.ets` file. The parser recognizes `@interface` as an annotation declaration (not a decorator), produces `AnnotationDeclaration` with an `AnnotationPropertyDeclaration` child for `label`. The `@` is NOT consumed by the decorator parser.

**Scenario 2: Decorator vs annotation disambiguation**
The parser encounters `@interface` in a `.ets` file. The guard in `tryParseDecorator` prevents `@` from being consumed as a decorator start — it returns nil, and `parseAnnotationDeclaration` consumes both `@` and `interface`.

**Scenario 3: .ts file ignores annotations**
A developer writes `@interface Foo { }` in a `.ts` file. `inAllowAnnotationContext()` returns false (not in ETS context), so the `@` is parsed as a regular decorator — producing a grammar error, which is correct.

## Target Users

- **ArkTS application developers** — define custom decorator types via `@interface` for use in component declarations
- **ArkTS compiler developers** — maintain the parser's decorator/annotation disambiguation logic

## Acceptance Strategy

| Criterion | Verification |
|-----------|-------------|
| `@interface` in .ets produces AnnotationDeclaration AST | Integration tests in `arkts_test.go` |
| `@interface` does NOT consume `@` as decorator | Unit test: `tryParseDecorator` returns nil for `@interface` |
| `.ts` files reject `@interface` as grammar error | `TestParseAnnotationDeclaration_NotInEtsContext` |
| Existing decorator parsing unaffected | All existing decorator tests pass unchanged |
| AST and diagnostics match reference tsc | `arkts_cmp_test.go` compares against reference for annotation test cases in `arkTSTest/testcase/` |

| Criterion | Verification |
|-----------|-------------|
| `@interface` in .ets produces AnnotationDeclaration AST | Integration tests in `arkts_test.go` |
| `@interface` does NOT consume `@` as decorator | Unit test verifying `tryParseDecorator` returns nil |
| `.ts` files reject `@interface` as grammar error | `TestParseAnnotationDeclaration_NotInEtsContext` |
| Existing decorator parsing unaffected | All existing decorator tests pass unchanged |
| Valid annotation syntax parses without errors | `@interface Foo { x: string; }` clean parse |
