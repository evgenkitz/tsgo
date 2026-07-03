# Issue #17 — ArkUI loop component parsing (ForEach, LazyForEach, Repeat.each)

## Reference

- `_submodules/third_party_typescript/src/compiler/parser.ts:6752-6760` — attrUICallback detection in `parsePropertyAccessExpressionRest`
- `_submodules/third_party_typescript/src/compiler/parser.ts:6782-6786` — paramsUICallback detection in `parseCallExpressionRest`
- `_submodules/third_party_typescript/src/compiler/parser.ts:6957-6980` — `parseArgumentExpression` state management
- `_submodules/third_party_typescript/src/compiler/parser.ts:6808-6810` — `isValidVirtualTypeArgumentsContext`
- `_submodules/third_party_typescript/src/compiler/parser.ts:2127-2141` — `firstArgumentExpression` / `repeatEachRest` state
- `_submodules/third_party_typescript/src/compiler/utilities.ts:4274-4290` — `getRootComponent` (reference, not ported inline)

## What was ported

### 1. New EtsFlags (`internal/ast/arkts.go`)

| Flag | Bit | Purpose |
|------|-----|---------|
| `EtsFlagsFirstArgumentExpression` | 1 << 13 | Tracks first argument of syntax component (data source) |
| `EtsFlagsRepeatEachRest` | 1 << 14 | Inside `Repeat.each` rest arguments |

### 2. Parser state management (`internal/parser/arkts.go`)

| Go function | Reference |
|------------|-----------|
| `setFirstArgumentExpression(val)` | parser.ts:2129-2131 |
| `inFirstArgumentExpression()` | parser.ts:2133-2134 (getFirstArgumentExpression) |
| `setRepeatEachRest(val)` | parser.ts:2139-2141 |
| `inRepeatEachRest()` | — (reference uses local variable) |
| `isValidVirtualTypeArgumentsContext()` | parser.ts:6808-6810 |

### 3. Syntax component detection (`internal/parser/arkts.go`)

| Go function | Reference | What it does |
|------------|-----------|-------------|
| `detectSyntaxComponentCall(expression)` | parser.ts:6782-6786 + 6752-6760 | Detects ForEach/LazyForEach by identifier (paramsUICallback) or Repeat.each by property access (attrUICallback) |
| `getRootComponentName(expression)` | utilities.ts:4274-4290 | Walks property access chain to find root identifier and rightmost property name |
| `hasSyntaxComponentParam(name)` | — | Checks name against `EtsOptions.SyntaxComponents.ParamsUICallback` |
| `getSyntaxComponentAttr(rootName, propName)` | — | Checks rootName/propName against `EtsOptions.SyntaxComponents.AttrUICallback` |

### 4. Integration into parser (`internal/parser/parser.go`)

| Location | What |
|----------|------|
| `parseCallExpressionRest` | Before `parseArgumentList`: call `detectSyntaxComponentCall(inner)` to detect syntax component. After argument parsing: reset `SyntaxComponentContext` and `FirstArgumentExpression`. |
| `parseArgumentExpression` | State management for `SyntaxDataSourceContext` / `RepeatEachRest`: on first argument of syntax component, set `SyntaxDataSourceContext` (for data source argument). For `Repeat.each`, set `RepeatEachRest` instead. Reset after argument is parsed. |

### 5. EtsOptions configuration (`internal/core/arkts.go`)

Already existed from Issue #13:

| Field | Type | Purpose |
|-------|------|---------|
| `EtsSyntaxComponentsOptions.ParamsUICallback` | `[]string` | Component names for simple call detection (e.g. `["ForEach", "LazyForEach"]`) |
| `EtsSyntaxComponentsOptions.AttrUICallback` | `[]EtsSyntaxComponentsAttrUICallbackOptions` | Property-access-based detection (e.g. `{Name: "Repeat", Attributes: ["each"]}`) |

## Implementation differences

### getRootComponent — simplified

The reference `getRootComponent` (utilities.ts:4274) is a shared utility that classifies component type (`etsComponentType`, `callExpressionComponentType`, `otherType`) and returns both the root node and type. The Go implementation (`getRootComponentName`) is a parser-private method that only extracts the root identifier name and property name — sufficient for syntax component detection.

### Context reset — done after argument list

The reference resets `SyntaxComponentContext` in `parsePropertyAccessExpressionRest` loop exit (parser.ts:6801-6803). In Go, the reset is done after `parseArgumentList()` returns in `parseCallExpressionRest` — equivalent timing, different location.

### Local variables → flags

The reference uses TypeScript local variables (`firstArgumentExpression: boolean = false`, `repeatEachRest: boolean = false`). Go uses `EtsFlags` bitfield flags on the Parser struct — consistent with the existing ArkTS flag pattern.

## Tests

**Go:** `internal/parser/arkts_loop_test.go`

| Test | What it covers |
|------|---------------|
| `TestParseForEachCall` | `ForEach(items, item => ..., key => ...)` with paramsUICallback detection |
| `TestParseRepeatEachCall` | `Repeat.each(data, item => ...)` with attrUICallback detection |
| `TestParseOrdinaryCall_NoFalsePositive` | Ordinary function call does NOT trigger detection |
| `TestParseSyntaxComponent_OutsideBuilderContext` | Syntax component outside @Builder context is not detected |

## Deferred to future issues

| Item | Issue |
|------|-------|
| Full `getRootComponent` port (with type classification) | Needed for StateStyles (#8, #9) |
| Checker-level validation of syntax component arguments | Binder/checker phase |
| `SyntaxDataSourceContext` consumption by checker | Binder/checker phase |
| ForEach/LazyForEach/Repeat.each argument type checking | Checker phase |

## Requirement Scenario

**Scenario 1: ForEach list rendering**
An ArkTS developer writes:
```typescript
@Builder function build() {
  ForEach(items, (item: Item) => { Text(item.name) }, (item: Item) => item.id)
}
```
The parser detects `ForEach` as a `paramsUICallback` entry, sets `SyntaxComponentContext`. The first argument (`items`) is parsed with `SyntaxDataSourceContext` — marking it as the data source. The second argument (item builder) and third argument (key generator) are parsed normally.

**Scenario 2: Repeat.each list rendering**
```typescript
@Builder function build() {
  Repeat.each(data, (item: Item) => { Text(item) })
}
```
The parser detects `Repeat.each` as an `attrUICallback` entry (root=`Repeat`, prop=`each`), sets `SyntaxComponentContext` + `RepeatEachRest`. The first argument triggers `SyntaxDataSourceContext` (like ForEach), but with `RepeatEachRest` enabled.

**Scenario 3: Ordinary call — no detection**
```typescript
@Builder function build() {
  ordinaryFunction(x, y)
}
```
`ordinaryFunction` is not in `ParamsUICallback` or `AttrUICallback`. No context flags are set. Arguments are parsed as regular expressions.

## Target Users

- **ArkUI application developers** — use `ForEach`, `LazyForEach`, `Repeat.each` for list rendering in UI component definitions
- **Compiler developers** — maintain the parser's context-sensitive argument parsing for ArkUI loop constructs

## Restrictions & Constraints

- **Detection gated by `isValidVirtualTypeArgumentsContext()`**: only activates inside `build()`, `@Builder`, `@Styles`, `@Extend` contexts
- **EtsOptions dependency**: detection requires `EtsOptions.SyntaxComponents.ParamsUICallback` and `AttrUICallback` to be configured. Without configuration, no detection occurs (graceful degradation).
- **O(n*m) complexity**: component name matching is O(number of params * number of components), acceptable for small configured lists (typically <5 components)
- **getRootComponentName simplification**: only extracts identifier names (not full type classification). Full port needed for StateStyles (#8, #9)

## Acceptance Strategy

| Criterion | Verification |
|-----------|-------------|
| ForEach call detected in @Builder context | `TestParseForEachCall` in `arkts_loop_test.go` |
| Repeat.each call detected in @Builder context | `TestParseRepeatEachCall` |
| Ordinary calls not falsely detected | `TestParseOrdinaryCall_NoFalsePositive` |
| Detection disabled outside @Builder context | `TestParseSyntaxComponent_OutsideBuilderContext` |
| No regression in existing parser tests | `go test ./internal/parser/...` |
| AST comparison tests unchanged | `go test ./internal/testrunner/... -run TestArktsRefTscCmp` |
