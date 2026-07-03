# Issue #7 — EtsComponentExpression parsing and virtual type arguments

## Reference

- `_submodules/third_party_typescript/src/compiler/parser.ts:5351-5354, 6865-6880, 6928-6929` — EtsComponentExpression parsing
- `_submodules/third_party_typescript/src/compiler/parser.ts:6720-6806` — virtual type arguments in parseCallExpressionRest
- `_submodules/third_party_typescript/src/compiler/parser.ts:6230-6250` — virtual identifiers for @Extend/@Styles
- `_submodules/third_party_typescript/src/compiler/parser.ts:5420-5440, 5720-5734` — UICallback context management
- `_submodules/third_party_typescript/src/compiler/parser.ts:8095-8102, 8130-8173` — struct build() context
- `_submodules/third_party_typescript/src/compiler/utilities.ts:4274-4301` — expression-walking helpers

## Context

Enable the Go parser (tsgo) to recognize ArkTS UI component expression syntax `ComponentName(args) { body }` and inject virtual type arguments on component property access chains (`Button("hi").fontSize(14)`). Ported from reference ohos-typescript, this is a core ArkTS language feature required for OpenHarmony/HarmonyOS application compilation.

## Dependencies

- Branch `arkts/6-phase-2-parser-modifications` — provides EtsOptions struct, struct parsing, decorator context detection, auto-readonly injection
- Phase 1: EtsComponentExpression AST node (generated), EtsFlags enum on Parser, EtsOptions configuration

## Out of scope

- Checker/binder/emitter modifications
- Function/method declaration hooks
- Decorator argument parsing for @Extend/@Styles metadata

## What was ported

### Step 1: EtsComponentExpression parsing

Recognize `ComponentName(args)` and `ComponentName(args) { body }` as EtsComponentExpression AST nodes instead of regular CallExpression.

| Function | Reference | Purpose |
|----------|-----------|---------|
| `isCurrentTokenAnEtsComponentExpression()` | parser.ts:6865-6868 | Gate: checks token text against `compilerOptions.ets.components` |
| `parseEtsComponentExpression()` | parser.ts:6873-6880 | Parses name + arguments + optional body |
| `makeEtsComponentExpression()` | parser.ts:5351-5354 | Converts CallExpression + `{` to ECE |

**Files:** `internal/parser/arkts.go`, `internal/parser/parser.go`

### Step 2: Virtual type arguments in parseCallExpressionRest

Inject synthesized `TypeReference("ButtonAttribute")` as virtual type argument on property access calls.

| Function | Reference | Purpose |
|----------|-----------|---------|
| `tryInjectEtsVirtualTypeArgs()` | parser.ts:6720-6780 | Main injection logic |
| `isValidVirtualTypeArgumentsContext()` | parser.ts:6808-6809 | Context gate |
| `tryMakeEtsComponentExpression()` | — | CallExpression → ECE conversion |

**Files:** `internal/parser/arkts.go`, `internal/parser/parser.go`

### Step 3: Expression-walking and type factory helpers

| Function | Reference | Purpose |
|----------|-----------|---------|
| `getRootComponent()` | utilities.ts:4274-4290 | Walk chain to find component root |
| `getVirtualEtsComponent()` | utilities.ts:4293-4301 | Find virtual PropertyAccess nodes |
| `parseEtsType()` / `parseEtsTypeArguments()` | parser.ts:5140-5155 | Create synthesized type nodes |
| `markVirtualNode()` / `isVirtualNode()` | — | Virtual node tracking |

**Files:** `internal/parser/arkts.go`

### Step 4: Virtual identifiers for @Extend/@Styles

Synthesize implicit `ButtonInstance` identifier for `.fontSize(14)` inside `@Extend(Button)`.

| Function | Reference | Purpose |
|----------|-----------|---------|
| `tryParseEtsVirtualIdentifier()` | parser.ts:6237-6244 | Creates virtual identifier before `.` |
| `isValidExtendOrStylesContext()` | — | Context gate for virtual IDs |
| `lookupEtsComponentInstance()` | — | Resolves instance name from EtsOptions |

**Files:** `internal/parser/arkts.go`, `internal/parser/arkts_decorators.go`, `internal/parser/parser.go`

### Step 5: UICallback context management

Enable `makeEtsComponentExpression` path for arrow function callbacks inside `ForEach`.

| Function | Reference | Purpose |
|----------|-----------|---------|
| `enterEtsArrowContext()` / `exitEtsArrowContext()` | parser.ts:5420-5440, 5720-5734 | Save/restore callback context flags |

**Files:** `internal/parser/arkts.go`, `internal/parser/parser.go`

### Step 6: Remaining expression-level modifications

| Function | Reference | Purpose |
|----------|-----------|---------|
| `enterEtsArgumentContext()` / `exitEtsArgumentContext()` | parser.ts:6957-6979 | Syntax component argument flags |
| `isEtsStateStylesBypass()` / `tryParseEtsStateStylesIdentifier()` | parser.ts:2512-2522, 2850-2857 | StateStyles recovery |

**Files:** `internal/parser/arkts.go`, `internal/parser/parser.go`

### Step 7: Struct build() method context

Activate `EtsBuildContext` so virtual type arguments are injected inside struct `build()`.

| Function | Reference | Purpose |
|----------|-----------|---------|
| `isTokenInsideStructBuild()` | parser.ts:8095-8102 | Detect build method |
| `setEtsBuildContext()` save/restore | parser.ts:8130-8173 | Context management |

**Files:** `internal/parser/parser.go`

### Step 8: Test harness configuration

Infrastructure for feeding `.ets` test files through both parsers with ArkTS compiler options.

| Function | Purpose |
|----------|---------|
| `parseEtsComponentsDirective` | Extract `// @etsComponents:` directive from test source |
| `initEtsFromOptions()` | Wire EtsOptions from SourceFileParseOptions into parser state |

**Files:** `internal/testrunner/arkts_cmp_test.go`, `internal/parser/arkts.go`

## Implementation differences

- All ArkTS logic in `arkts*.go` files; `parser.go` changes are minimal one-line callouts
- Virtual type arguments consolidated into `tryInjectEtsVirtualTypeArgs()` instead of inline 74-line block
- Virtual identifiers use `finishVirtualNode` pattern matching reference tsc
- UICallback context management uses save/restore pattern to prevent leakage

## Tests

**Go:** `internal/parser/arkts_ets_component_test.go` — unit tests for EtsComponentExpression parsing, virtual type arguments, virtual identifiers, UICallback context
**Go:** `internal/testrunner/arkts_cmp_test.go` — AST comparison vs reference tsc (34/34 files passing, 0 known gaps)

**Test data:** `testdata/arkts/ets_components_*.ets`, `testdata/arkts/struct_build_components.ets`, `testdata/arkts/struct_with_component.ets`, `testdata/arkts/struct_with_entry_component.ets`

## Deferred to future issues

| Item | Issue |
|------|-------|
| Checker-level validation of component arguments | Binder/checker phase |
| Emitter support for EtsComponentExpression | Emitter phase |
| Full StateStyles integration | Issues #8, #9 |

## References

| Reference tsc file | Lines | Content |
|---|---|---|
| `parser.ts` | 5351-5354 | `makeEtsComponentExpression` |
| `parser.ts` | 6865-6880 | `parseEtsComponentExpression` + `isCurrentTokenAnEtsComponentExpression` |
| `parser.ts` | 6928-6929 | Hook in `parsePrimaryExpression` |
| `parser.ts` | 6720-6806 | Virtual type arguments in `parseCallExpressionRest` |
| `parser.ts` | 6230-6250 | Virtual identifiers for @Extend/@Styles |
| `parser.ts` | 5420-5440, 5720-5734 | UICallback context management |
| `parser.ts` | 8095-8102, 8130-8173 | Struct build() context |
| `utilities.ts` | 4274-4301 | Expression-walking helpers |

## Requirement Scenario

**Scenario 1: Component instantiation**
An ArkTS developer writes `Button("hello")` inside a `@Builder` function. The parser recognizes `Button` as a known component name and produces an `EtsComponentExpression` node with the argument list.

**Scenario 2: Component with nested child body**
`Column() { Text("child") }` — the parser detects the `{` following the call arguments and parses a function body containing child component expressions, forming a tree of nested `EtsComponentExpression` nodes.

**Scenario 3: Property access with virtual type arguments**
`Button("hi").fontSize(14).fontColor(Color.Red)` — inside a `@Builder` function, the parser injects a synthesized `TypeReference("ButtonAttribute")` as a virtual type argument on each chained property access call.

**Scenario 4: Decorator-based virtual identifiers**
`@Extend(Button) function myExtend() { .fontSize(14) }` — the parser synthesizes a virtual `ButtonInstance` identifier for the leading `.` so the property access chain can be parsed correctly.

**Scenario 5: CallExpression-to-ECE conversion**
`@Builder function myBuilder() { UnknownFunc("arg") { Text("child") } }` — when a call expression is followed by `{` inside a UI callback context, the parser converts it to an `EtsComponentExpression` with a trailing body.

**Scenario 6: UICallback context in arrow functions**
Inside `ForEach("data").itemBuilder((item) => { ... })`, the parser enters `UICallbackContext` so that `{ body }` following a call expression inside the arrow function produces an `EtsComponentExpression`.

## Target Users

- **ArkTS application developers** — write `.ets` files using ArkUI declarative UI syntax
- **ArkTS compiler developers** — maintain EtsComponentExpression parsing and virtual type argument injection

## Restrictions & Constraints

- All ArkTS logic must live in dedicated `arkts*.go` files; modifications to `parser.go` are minimal one-line callouts
- Must match reference ohos-typescript parser.ts behavior exactly for AST structure
- Must maintain backward compatibility: non-ETS TypeScript parsing unchanged
- Not in scope: checker/binder/emitter modifications

## Acceptance Strategy

| Criterion | Verification |
|-----------|-------------|
| Component expressions parse correctly | `testdata/arkts/ets_components_*.ets` files |
| Virtual type arguments injected on property chains | Parser unit tests for chained calls |
| Virtual identifiers work in @Extend/@Styles context | Unit tests for decorator-based patterns |
| UICallback context enables ECE in arrow functions | Parser unit tests for ForEach callbacks |
| AST matches reference tsc | `go test -run='^TestArktsCmp$' ./internal/testrunner/` — 34/34 pass |
| No regressions in existing parser tests | `go test ./internal/parser/ -count=1` |
