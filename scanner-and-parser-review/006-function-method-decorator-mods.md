# Issue #6 — Function/method decorator modifications and auto-readonly injection

## Reference

- `_submodules/third_party_typescript/src/compiler/parser.ts:7998-8061` — `parseFunctionDeclaration` modifications
- `_submodules/third_party_typescript/src/compiler/parser.ts:8133-8165` — `parseMethodDeclaration` modifications
- `_submodules/third_party_typescript/src/compiler/parser.ts:7730-7761` — `parseDeclaration` decorator context detection
- `_submodules/third_party_typescript/src/compiler/parser.ts:8471-8501` — `hasParamAndNoOnceDecorator`, `hasEnvDecorator`
- `_submodules/third_party_typescript/src/compiler/parser.ts:8556-8558` — auto-readonly injection in `parseClassElement`
- `_submodules/third_party_typescript/src/compiler/parser.ts:8095-8115` — `isTokenInsideStruct*` helpers
- `_submodules/third_party_typescript/src/compiler/parser.ts:7097-7098` — `EtsNewExpressionContext`
- `_submodules/third_party_typescript/src/compiler/parser.ts:5766-5774` — struct ASI guard
- `_submodules/third_party_typescript/src/compiler/parser.ts:4145-4148` — `parseEtsTypeParameters`
- `_submodules/third_party_typescript/src/compiler/parser.ts:2882-2886` — `parseEtsIdentifier` (virtual type param name)
- `_submodules/third_party_typescript/src/compiler/ohApi.ts:303-505` — decorator classification helpers

## What was ported

### 1. Decorator helpers (`ohApi.ts:303-505`)

**Go:** `internal/parser/arkts_decorators.go`

| Reference | Go |
|---|---|
| `hasEtsExtendDecoratorNames(decorators, options)` → checks `options.ets.extend.decorator` | `(*Parser).hasEtsExtendDecoratorNames(decorators)` → reads `EtsOptions.Extend.Decorator`, default `["Extend"]` |
| `hasEtsStylesDecoratorNames(decorators, options)` → checks `options.ets.styles.decorator` | `(*Parser).hasEtsStylesDecoratorNames(decorators)` → reads `EtsOptions.Styles.Decorator`, default `"Styles"` |
| `hasEtsBuilderDecoratorNames(decorators, options)` → checks `options.ets.render.decorator` | `(*Parser).hasEtsBuilderDecoratorNames(decorators)` → reads `EtsOptions.Render.Decorator`, default `["Builder", "LocalBuilder"]` |
| `isTokenInsideBuilder(decorators, options)` → `options.ets.render.decorator ?? ["Builder", "LocalBuilder"]` | `(*Parser).isTokenInsideBuilder(decorators)` → delegates to `hasEtsBuilderDecoratorNames` |
| `getEtsExtendDecoratorsComponentNames(decorators, options)` → `options.ets.extend.decorator ?? "Extend"` | `(*Parser).getEtsExtendDecoratorsComponentNames(decorators)` → reads `EtsOptions.Extend.Decorator`, default `["Extend"]` |
| `getEtsStylesDecoratorComponentNames(decorators, options)` → `options.ets.styles.decorator ?? "Styles"` | `(*Parser).getEtsStylesDecoratorComponentNames(decorators)` → reads `EtsOptions.Styles.Decorator`, default `"Styles"` |
| `hasParamAndNoOnceDecorator(decorators)` — hardcoded `"Param"`/`"Once"` | `(*Parser).hasParamAndNoOnceDecorator(decorators)` — same |
| `hasEnvDecorator(decorators)` — hardcoded `"Env"`/`"CustomEnv"` | `(*Parser).hasEnvDecorator(decorators)` — same |
| `applyEtsDecoratorContext` (inline in `parseDeclaration`) | `(*Parser).applyEtsDecoratorContext(decorators)` — standalone method |
| `injectReadonlyModifier` (inline in `parseModifiers`) | `(*Parser).injectReadonlyModifier(modifiers)` — standalone method |
| `createSyntheticTypeReference` (inline) | `(*Parser).createSyntheticTypeReference(typeName)` — shared helper |
| `parseEtsTypeParameters` → calls `parseEtsIdentifier` with `stylesEtsComponentDeclaration.type` | `(*Parser).parseEtsTypeParameters(pos)` → uses `stylesEtsComponentDecl.typeName` |
| `extractDecorators` (no equivalent — decorators are inline in modifiers) | `extractDecorators(modifiers)` — Go decorators are part of modifier list, needs extraction |

### 2. EtsOptions accessor methods

**Go:** `internal/parser/arkts_decorators.go`

All decorator/method name lookups go through accessor methods that read from `p.opts.EtsOptions` with defaults matching the reference:

| Go accessor | Reads from | Default |
|---|---|---|
| `(*Parser).getEtsExtendDecorators()` | `EtsOptions.Extend.Decorator` | `["Extend"]` |
| `(*Parser).getEtsStylesDecorator()` | `EtsOptions.Styles.Decorator` | `"Styles"` |
| `(*Parser).getEtsRenderDecorators()` | `EtsOptions.Render.Decorator` | `["Builder", "LocalBuilder"]` |
| `(*Parser).getEtsRenderMethods()` | `EtsOptions.Render.Method` | `["build", "pageTransition"]` |
| `(*Parser).hasEtsRenderMethod(name)` | `EtsOptions.Render.Method` | checks membership |

`EtsOptions` is passed to the parser via `ast.SourceFileParseOptions.EtsOptions`, set from `compiler.CompilerOptions.Ets` in `fileloader.go`.

### 3. parseDeclaration — decorator context detection (`parser.ts:7730-7761`)

**Go:** Moved into `parseFunctionDeclaration` / `parseMethodDeclaration` (not `parseDeclaration`) to prevent context leakage for non-function declarations.

Reference logic:
```
if (FunctionKeyword || ExportKeyword):
  @Extend → set EtsExtendComponentsContext + store component decl
  @Styles → set EtsStylesComponentsContext + store component decl
  else    → set EtsComponentsContext(isTokenInsideBuilder())
```

Go implementation in `(*Parser).applyEtsDecoratorContext(decorators)` — called from both `parseFunctionDeclaration` and `parseMethodDeclaration`.

### 4. parseFunctionDeclaration modifications (`parser.ts:7998-8061`)

| Reference | Go |
|---|---|
| Save `originalUICallbackContext` | `originalUICallbackContext = p.inUICallbackContext()` |
| `setEtsBuilderContext(hasEtsBuilderDecoratorNames(...))` | same |
| `setUICallbackContext(inBuilderContext())` | same |
| Record in `fileStylesComponents.set(name, FunctionDeclaration)` | `p.fileStylesComponents[name.Text()] = ast.KindFunctionDeclaration` |
| `parseEtsTypeParameters` in @Styles context | same |
| Virtual return type for @Extend/@Styles | same — `createSyntheticTypeReference` |
| **Cleanup:** `setEtsBuilderContext(false)`, `extendEtsComponentDeclaration = undefined`, `setEtsExtendComponentsContext(false)`, `stylesEtsComponentDeclaration = undefined`, `setEtsStylesComponentsContext(false)`, `setEtsComponentsContext(inBuildContext())`, `setUICallbackContext(originalUICallbackContext)` | same |

### 5. parseMethodDeclaration modifications (`parser.ts:8133-8165`)

| Reference | Go |
|---|---|
| Save `originalUICallbackContext` | same |
| `applyEtsDecoratorContext` (same as parseFunctionDeclaration) | same |
| `setEtsComponentsContext` for struct build/@Builder/pageTransition methods | `p.isTokenInsideStructBuild(name) \|\| p.isTokenInsideStructBuilder(modifiers) \|\| p.isTokenInsideStructPageTransition(name)` |
| `structStylesComponents` tracking for @Styles methods in struct context | `p.structStylesComponents.Add(componentName)` (lazy init) |
| Virtual type parameters + return type | same as parseFunctionDeclaration |
| Cleanup — hard reset | same |

### 6. isTokenInsideStruct* helpers (`parser.ts:8095-8115`)

| Reference | Go |
|---|---|
| `isTokenInsideStructBuild(name)` → `ets.render.method.find("build") ?? "build"` | `(*Parser).isTokenInsideStructBuild(name)` → `core.Find(getEtsRenderMethods(), "build")` with default `"build"` |
| `isTokenInsideStructBuilder(decorators)` → `isTokenInsideBuilder(decorators, options)` | `(*Parser).isTokenInsideStructBuilder(modifiers)` → `p.isTokenInsideBuilder(extractDecorators(modifiers))` |
| `isTokenInsideStructPageTransition(name)` → `ets.render.method.find("pageTransition") ?? "pageTransition"` | `(*Parser).isTokenInsideStructPageTransition(name)` → same pattern as `isTokenInsideStructBuild` |

### 7. Auto-readonly injection (`parser.ts:8471-8501, 8556-8558`)

| Reference | Go |
|---|---|
| `hasParamAndNoOnceDecorator(decorators)` — hardcoded `"Param"`/`"Once"` | `(*Parser).hasParamAndNoOnceDecorator(decorators)` — same |
| `hasEnvDecorator(decorators)` — hardcoded `"Env"`/`"CustomEnv"` | `(*Parser).hasEnvDecorator(decorators)` — same |
| `shouldAddReadonly = inStructContext() && (hasParamAndNoOnce \|\| hasEnv)` | same |
| Inject virtual `readonly` in `parseModifiers` via `shouldAddReadonly` flag | `(*Parser).injectReadonlyModifier(modifiers)` prepends virtual `readonly` token |

### 8. Integration glue

| Reference | Go |
|---|---|
| `setEtsNewExpressionContext(inEtsComponentsContext())` in `parseNewExpressionOrNewDotTarget` (`parser.ts:7097`) | `p.setEtsNewExpressionContext(p.inEtsComponentsContext())` with save/restore |
| `(!inEtsContext() \|\| token !== StructKeyword)` in `parseArrowFunctionExpressionBody` (`parser.ts:5766`) | `!p.inEtsContext() \|\| p.token != ast.KindStructKeyword` |
| `getLanguageVariant` — implied `LanguageVariantStandard` for ETS | `case core.ScriptKindETS: return core.LanguageVariantStandard` |

### 9. AST/checker utilities

| Reference | Go (`internal/ast/utilities.go`, `internal/checker/checker.go`) |
|---|---|
| `CanHaveDecorators` extended for `StructDeclaration`, `FunctionDeclaration` | same |
| `NodeCanBeDecorated` extended for `StructDeclaration` | same |
| `getDiagnosticHeadMessageForDecoratorResolution` extended | same |

### 10. Parser fields added

| Field | Type | Purpose |
|-------|------|---------|
| `extendEtsComponentDecl` | `*etsComponentDecl` | Component info from `@Extend` decorator |
| `stylesEtsComponentDecl` | `*etsComponentDecl` | Component info from `@Styles` decorator |
| `structStylesComponents` | `*collections.Set[string]` | @Styles component names in struct scope (cleared per file) |
| `fileStylesComponents` | `map[string]ast.Kind` | @Styles function names in file scope (cleared per file) |

## Implementation differences

### EtsOptions wiring

The reference passes `CompilerOptions` directly to every helper function. In Go, the parser has no access to the full `CompilerOptions` — options arrive via `SourceFileParseOptions`. Added `EtsOptions *core.EtsOptions` to `SourceFileParseOptions`, set from `CompilerOptions.Ets` in `fileloader.go`. All decorator/method name lookups go through accessor methods (`getEtsExtendDecorators`, `getEtsStylesDecorator`, `getEtsRenderDecorators`, `getEtsRenderMethods`) that read from `p.opts.EtsOptions` with defaults matching the reference.

### Decorator extraction

In the reference, decorators and modifiers are separate node arrays. In Go/tsgo, decorators are part of the modifier list (`ModifierList`). Added `extractDecorators(modifiers)` to filter decorator nodes from the combined list.

### Context detection moved to per-declaration functions

The reference does decorator context detection in `parseDeclaration` (before dispatching to specific parsers). This approach can leak context for non-function declarations (class, variable). Moved the detection into `parseFunctionDeclaration` and `parseMethodDeclaration` directly.

### EtsNewExpressionContext save/restore scope

The reference sets `EtsNewExpressionContext` at function entry without save/restore. Added save/restore to prevent leakage to nested expressions.

### Cleanup: hard reset vs save/restore

The reference hard-resets all decorator contexts at the end of `parseFunctionDeclaration`/`parseMethodDeclaration` (e.g., `setEtsBuilderContext(false)`, `extendEtsComponentDeclaration = undefined`). Matches the reference exactly — no save/restore for decorator contexts, only `originalUICallbackContext` is restored.

## Tests

**Go:** `internal/parser/arkts_decorators_test.go` — unit tests for all decorator helpers, function/method parsing, auto-readonly injection, @Sendable support, StructDeclaration decorator container

## Deferred to future issues

| Item | Issue |
|------|-------|
| Printer support for ArkTS nodes (StructDeclaration, AnnotationDeclaration, EtsComponentExpression) | Printer phase |
| Round-trip tests (parse → print → re-parse) | Blocked by printer |
| Port arkTSTest/testcase/ test data | Separate test issue |
| `fileStylesComponents` consumption by checker/emitter | Binder/checker phase |
| `structStylesComponents` consumption by checker/emitter | Binder/checker phase |

## Requirement Scenario

**Scenario 1: @Builder function**
An ArkTS developer writes `@Builder function myBuilder() { Text("Hello"); }`. The parser detects the `@Builder` decorator, sets `EtsBuilderContext`, and the function body is parsed in builder context — allowing ArkUI expression syntax inside.

**Scenario 2: @Extend/@Styles with virtual return type**
`@Extend(Button) function customButton() { .width(100).height(50) }` — the parser extracts `Button` from the decorator argument, synthesizes a virtual return type `Button`, and sets `EtsExtendComponentsContext`. The leading `.width()` chain works because the parser knows it's inside an extend context.

**Scenario 3: Auto-readonly injection**
Inside a struct: `@Component struct Foo { @Param title: string = ""; }` — the parser detects `@Param` (without `@Once`) in struct context, and prepends a virtual `readonly` modifier to the property. The resulting AST has `readonly @Param title: string`.

**Scenario 4: @Sendable on function and type alias**
`@Sendable function myFunc() {}` and `@Sendable type MyType = () => void` — the parser recognizes these as valid decorator targets in `.ets` files (via `NodeCanBeDecorated` extension), but rejects them in `.ts` files.

## Target Users

- **ArkTS application developers** — use `@Builder`, `@Extend`, `@Styles`, `@Param`, `@Env`, `@Sendable` decorators
- **ArkTS compiler developers** — maintain decorator context detection and auto-readonly injection

## Restrictions & Constraints

- **Decorator names**: currently hardcoded (`"Builder"`, `"Extend"`, `"Styles"`, `"Param"`, `"Env"`, `"Once"`, `"Sendable"`). Full EtsOptions-based configuration deferred to binder/checker phase.
- **Context cleanup**: decorator contexts are hard-reset (not saved/restored) at the end of `parseFunctionDeclaration`/`parseMethodDeclaration`, matching the reference.
- **Decorator extraction**: Go stores decorators in `ModifierList` alongside keyword modifiers (unlike reference where they are separate arrays). `extractDecorators()` filters decorator nodes from the combined list.
- **EtsOptions wiring**: parser receives `EtsOptions` via `SourceFileParseOptions`, not via `CompilerOptions` directly (parser has no access to full compiler options).

## Acceptance Strategy

| Criterion | Verification |
|-----------|-------------|
| Decorator helpers, function/method parsing, auto-readonly, @Sendable | Unit tests in `arkts_decorators_test.go` |
| Existing TS decorator behavior unaffected | All existing decorator tests pass unchanged |
| AST and diagnostics match reference tsc | `arkts_cmp_test.go` compares against reference for decorator test cases (`arkts-sendable-decorator-limited`, etc.) |
