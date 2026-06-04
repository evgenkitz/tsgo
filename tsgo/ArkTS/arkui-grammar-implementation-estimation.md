# ArkUI Declarative Core Language — Implementation Estimation for tsgo

Based on grammar analysis from [[arkui-grammar-specification]] and codebase study of tsgo scanner/parser/binder.

**Scope:** Scanner, Parser, Binder only. Checker, Emitter, LSP, and other components excluded.

---

## Assumptions

- One senior engineer familiar with tsgo compiler internals
- Full-time dedication (8h/day, 5 days/week)
- ArkTS 1.1 base ([[arkts-1.1-implementation-estimation]]) is already implemented:
  - `struct` keyword, `@Component`/`@State`/etc. as decorators (parser already parses `@` as `KindAtToken`)
  - `override`, `final`, `native`, `internal` modifiers
  - Lambda expressions, trailing lambdas
- Changes are incremental and testable at each phase

---

## Architecture Overview

ArkUI eDSL is an **embedded DSL** inside `build()` and `@Builder` function bodies. It requires a **new parsing mode** — analogous to how JSX is a different parsing mode within TypeScript. When the parser enters a `build()` or `@Builder` body, it switches to eDSL mode where:

- Component creation replaces expression statements
- `if`/`else` becomes rendering control (not standard TS if)
- `ForEach` becomes a special repeated-content construct
- Attribute chaining (`.width(100).onClick(...)`) replaces method calls
- Regular TS expressions, variable declarations, and function calls are forbidden

---

## Phase 1 — Scanner: Lexical Additions

### 1.1 `$$` Token Recognition

**Current state:** `$` is a valid identifier character in TypeScript. The sequence `$$` is scanned as a single identifier. The scanner does not distinguish `$$` from any other identifier.

**Required change:** Add `KindDollarDollarToken` (or `KindDoubleDollarToken`) as a new token kind in `_scripts/ast.json`. In the scanner, when `$` is encountered as the start of an identifier, check if the next character is also `$` and the character after that is NOT an identifier start — if so, emit `KindDollarDollarToken` instead of scanning a full identifier.

This is needed to disambiguate `$$` (the ArkUI two-way binding token) from `$$myVar` (a regular JS identifier starting with `$$`).

**Alternative (less invasive):** Keep `$$` as an identifier. The parser checks `token == KindIdentifier && tokenValue == "$$"` where needed. This avoids scanner changes entirely. Given that `$$` is only meaningful in eDSL context, parser-level checking is simpler and preferred.

**Decision:** Handle `$$` at parser level (no scanner changes). Add `IsDollarDollar()` helper.

### 1.2 `ForEach` as Contextual Keyword

`ForEach` is a regular identifier. In eDSL parsing mode, the parser recognizes it as the `ForEach` construct when followed by `(`. No scanner changes needed.

### 1.3 eDSL Mode Flag

Add `LanguageVariantArkUI` to `LanguageVariant` enum in `internal/core/languagevariant.go`. This flag controls:
- Scanner: no behavioral changes needed (all ArkUI tokens are standard TS tokens)
- Parser: switches to eDSL body parsing inside `build()`/`@Builder` functions
- Used for error messages ("X is not allowed in ArkUI eDSL context")

### 1.4 New Kind Constants

In `_scripts/ast.json`, add the following parse-tree node Kinds (NOT token kinds — these are post-parsing AST nodes):

| Kind                       | Purpose                                                 |
| -------------------------- | ------------------------------------------------------- |
| `ComponentBlock`           | Children of a container component (like Column body)    |
| `ComponentCreation`        | Component instantiation: `Comp(params) { body }.attr()` |
| `ComponentAttribute`       | Single attribute in chain: `.width(100)`                |
| `ComponentParameter`       | Named parameter: `paramName: value`                     |
| `ForEachExpression`        | `ForEach(arr, itemBuilder, idBuilder?)`                 |
| `BuilderFunction`          | `@Builder` decorated function                           |
| `BuilderParam`             | `@BuilderParam` variable declaration                    |
| `ExtendFunction`           | `@Extend(Component) function f(...)`                    |
| `StylesFunction`           | `@Styles function f()`                                  |
| `TwoWayBinding`            | `$$this.var` expression                                 |
| `LinkSource`               | `this.$var` or `$$.$var` expression                     |
| `AnimatableExtendFunction` | `@AnimatableExtend(Component) function f(...)`          |

### Phase 1 Effort

| Task | Effort |
|---|---|
| `LanguageVariantArkUI` constant + plumbing | 0.5d |
| New Kinds in `_scripts/ast.json` + regenerate | 1d |
| New AST node type definitions in `ast.json` | 1d |
| Unit tests for scanner mode switching | 1d |
| **Subtotal** | **3.5 days** |

---

## Phase 2 — Parser: eDSL Mode

### 2.1 eDSL Parsing Context

Add `NodeFlagsArkUIeDSLContext` (or reuse a parsing context flag) that activates when the parser enters `build()` or `@Builder` function body.

**Entry points:**
- `parseFunctionDeclaration()` / method parsing — after seeing a function named `build` in a `@Component struct`, switch to eDSL mode for the body
- `@Builder` function parsing — always eDSL mode for the body
- `ForEach` item build function — eDSL mode
- `if`/`else` branch build functions — eDSL mode

**Implementation:** When parsing function body of `build()` or `@Builder`, set `contextFlags |= NodeFlagsArkUIeDSLContext` before parsing the body block.

### 2.2 Component Creation Parsing

This is the core eDSL construct. Syntax:
```
ComponentName({param: value, ...}) { childComponents } .attr1(args) .attr2(args) ...
```

**Parsing algorithm (in eDSL mode):**
1. Parse identifier (component name) — must start with uppercase
2. Optionally parse parameter list: `parseNamedParameterList()` — `({ name: expr, ... })` — similar to object literal but with named parameter constraints
3. Optionally parse component body: `{ eDSL statements }` — a block containing child component creations
4. Parse attribute chain: while `.` token, parse method call

**New parse functions needed:**
- `parseComponentCreation()` — top-level dispatch
- `parseComponentName()` — validate uppercase start
- `parseNamedParameterList()` — `({ name: expr, ... })`
- `parseComponentBody()` — `{ eDSL body }` 
- `parseComponentAttributeChain()` — `.method(args).method(args)...`
- `parseComponentAttribute()` — single `.method(args)`

### 2.3 eDSL if/else Parsing

In eDSL mode, `if`/`else` is rendering control, not a TS if statement. The syntax is similar but:
- Each branch body is an eDSL build function (≥1 component required)
- Empty branch is a syntax error
- Branches are "transparent" for parent-child relationships
- State is NOT preserved when branches swap

**Implementation:** Reuse existing `parseIfStatement()` infrastructure but with eDSL-specific validation:
- After parsing `if (condition)`, switch to eDSL body mode for the branch
- Validate each branch has ≥1 component
- Add eDSL-specific error messages

Could either:
a) Reuse `IfStatement` AST node with a flag indicating "eDSL rendering if"
b) Create new `RenderingIf` AST node

Option (a) is simpler and preferred, as the binder treats them similarly.

### 2.4 ForEach Parsing

```
ForEach(arrayExpr, (item, index?) => { body }, (item, index?) => idExpr)
```

**Parsing algorithm:**
1. Consume `ForEach` identifier (validated as contextual keyword in eDSL mode)
2. Consume `(`
3. Parse array expression (standard TS expression, must be `Array<T>`)
4. Consume `,`
5. Parse item build function — `(item, index?) => { eDSL body }`
6. Optionally consume `,` and parse id function — `(item, index?) => stringExpr`
7. Consume `)`

**New parse functions:**
- `parseForEachExpression()` — top-level ForEach parsing
- `parseForEachItemBuilder()` — the item build lambda
- `parseForEachIdFunction()` — the id generator lambda

**New AST node:** `ForEachExpression` with fields: `ArrayExpression`, `ItemBuilder` (arrow function), `IdFunction` (arrow function, optional)

### 2.5 `$` Prefix — Link Source Parsing

Syntax: `this.$varName` or `$$.$varName`

In the parser, `this.$varName` is a regular `PropertyAccessExpression`. After parsing the dot and identifier, check if the identifier starts with `$` (and is not `$$`). If so, create a `LinkSource` node wrapping the actual variable reference.

**Parsing changes:**
- In `parsePropertyAccessExpression()` or `parseMemberExpressionOrHigher()`, after parsing `.Identifier`, check for `$` prefix → create `LinkSource` node
- For `$$.$varName` inside builder functions: `$$` is the parameter name, `.` is property access. After `.`, check if property starts with `$` → `LinkSource`.

### 2.6 `$$` Two-Way Binding Parsing

Two contexts:

**A) Builder function parameter declaration:**
```
@Builder function f($$ : { label: string })
```
`$$` is the parameter name. This is parsed as a regular parameter. Validate that `$$` is used as the parameter name.

**B) Component attribute two-way binding:**
```
TextInput({ text: $$this.value })
```
In component parameter lists, `$$` prefix on a value expression indicates two-way binding. Parse `$$` then parse the expression — create `TwoWayBinding` wrapper node.

**C) Builder function call with `$$`:**
```
this.myBuilder($$ : { label: val })
```
Similar to parameter declaration — `$$` as object key in named parameter passing. Parser-level validation.

### 2.7 Attribute Chain Parsing

```
Component()
    .width(100)
    .height(200)
    .backgroundColor(Color.Red)
    .onClick(() => { this.count++ })
```

After parsing a component creation, enter attribute chain parsing loop:
```
while (token == KindDotToken) {
    parseComponentAttribute()
}
```

Each attribute is a method call: `.methodName(args)`. Event handlers are attributes where `methodName` is an event name (onClick, onChange, etc.) and the argument is an arrow function.

### 2.8 eDSL Constraint Validation

The parser must enforce ArkUI eDSL constraints and emit diagnostics:

| Constraint | Check |
|---|---|
| Exactly one top-level component in `build()` | Count components at top level; error if != 1 |
| No `if`/`ForEach` at `build()` top level | Check parent context is container, not root |
| At most one component per line | Check line numbers of consecutive component nodes |
| No variable declarations in eDSL body | Block `let`, `const`, `var` in eDSL context |
| No `new` for component creation | Error if `new` used before component name |
| No `switch` in eDSL | Error on `switch` keyword in eDSL context |
| No ternary as statement in eDSL | Error on `? :` at statement level |
| No `console.log` etc. in eDSL | Error on forbidden call expressions |
| No state mutation in eDSL | Error on `++`, `--`, assignment to state var |
| Component member access is private | Error on `public` / `protected` |
| No `static` component members | Error on `static` keyword in @Component |
| No constructor in @Component | Error on `constructor` in @Component struct |
| `@BuilderParam` type must be `BuilderType<C>` | Validate type annotation |
| Builder function naming (lowercase for component, uppercase for global) | Check first letter case |
| Component name uppercase, ≥2 chars | Validate in `parseComponentName()` |

### Phase 2 Effort

| Task | Effort |
|---|---|
| eDSL parsing context (`NodeFlagsArkUIeDSLContext` + mode switching) | 2d |
| AST node definitions in `_scripts/ast.json` (all new types) + regenerate | 3d |
| `parseComponentCreation()` — component instantiation with params + body | 5d |
| `parseNamedParameterList()` — named parameter passing | 2d |
| `parseComponentBody()` — children block | 1d |
| `parseComponentAttributeChain()` — attribute method chaining | 2d |
| eDSL `if`/`else` — rendering control variant | 3d |
| `parseForEachExpression()` — ForEach parsing | 4d |
| `$` prefix / LinkSource parsing | 1.5d |
| `$$` two-way binding parsing (3 contexts) | 2d |
| `@Builder` function parsing (component-level + global) | 3d |
| `@BuilderParam` variable parsing + `BuilderType<C>` validation | 2d |
| `@Extend` / `@AnimatableExtend` function parsing (attribute-only body) | 2.5d |
| `@Styles` function parsing (attribute-only body) | 1d |
| eDSL constraint validation (all diagnostics) | 4d |
| Parser error recovery in eDSL mode | 3d |
| `parseDeclaration()` dispatch for `@Component struct` | 1d |
| Unit tests (parsing positive + negative cases) | 8d |
| **Subtotal** | **50–57 days** |

---

## Phase 3 — Binder: Symbol Binding for ArkUI Constructs

### 3.1 `@Component struct` Binding

When binding a `@Component struct`:
- Create class symbol (same as `class`, using existing `bindClassLikeDeclaration`)
- Set `SymbolFlags` — possibly a new flag `SymbolFlagsComponent` to distinguish from regular classes
- The `struct` keyword already gets class treatment from ArkTS 1.1
- Bind all member variables (decorated and regular) within the component's symbol table
- Import ArkUI framework decorators into scope implicitly (or validate they're imported)

### 3.2 Decorated Variable Binding

For each state-decorated variable in a component:

```
@State count : number = 0;
@Prop label : string;
@Link linkedVar : SomeType;
@Provide('key') shared : string;
@Consume('key') consumed : string;
@ObjectLink objRef : ObservedClass;
@StorageLink('key') storedVal : string;
@StorageProp('key') storedProp : string;
@LocalStorageLink('key') localVal : string;
@LocalStorageProp('key') localProp : string;
@BuilderParam builderRef : BuilderType<{...}>;
```

**Binding tasks:**
- Create `PropertyDeclaration` symbol for each variable
- Set appropriate `SymbolFlags` based on decorator type (new flags needed: `SymbolFlagsState`, `SymbolFlagsProp`, `SymbolFlagsLink`, `SymbolFlagsProvide`, `SymbolFlagsConsume`, `SymbolFlagsObjectLink`, `SymbolFlagsStorageLink`, `SymbolFlagsStorageProp`, `SymbolFlagsLocalStorageLink`, `SymbolFlagsLocalStorageProp`, `SymbolFlagsBuilderParam`)
- Validate initialization rules per decorator (see table in grammar spec §19.2):
  - `@State`: must have local init, cannot init from parent
  - `@Link`: cannot local init, MUST init from parent via `$`
  - `@Prop`: optional local init, optional parent init
  - `@Provide`: must have local init
  - `@Consume`: cannot local init
  - `@ObjectLink`: cannot local init, MUST init from parent
  - etc.
- Validate type constraints per decorator (`@ObjectLink` must be `@Observed` class, etc.)

### 3.3 `@Observed` Class Binding

- Create class symbol with `SymbolFlagsObserved` flag
- Register the class as observable (so the checker can validate `@ObjectLink` usage)

### 3.4 `build()` Function Binding

- `build()` is a regular method — existing method binding handles it
- Set `SymbolFlagsBuildFunction` flag to identify it
- Validate: no parameters, returns void

### 3.5 `@Builder` Function Binding

**Component-level builder:**
- Create method symbol within the component's symbol table
- Set `SymbolFlagsBuilder` flag
- Validate: lowercase name (by convention, warning level)
- The `$$` parameter syntax — bind as regular parameter named `$$`

**Global builder:**
- Create function symbol in module scope
- Set `SymbolFlagsBuilder | SymbolFlagsGlobalBuilder` flags
- Validate: uppercase name, unique in application
- `this` is not available inside (validate)

### 3.6 `@BuilderParam` Variable Binding

- Create property symbol with `SymbolFlagsBuilderParam`
- Validate type is `BuilderType<C>` (or the desugared form `($$ : C) => void`)
- Track function reference assignments for later use in checker

### 3.7 `@Extend` / `@AnimatableExtend` Function Binding

- Create function symbol in module/global scope
- Associate with the extended built-in component (metadata)
- Validate: body contains only attribute function calls

### 3.8 `@Styles` Function Binding

- Create function symbol in module/global scope
- Validate: body contains only attribute function calls

### 3.9 `@Watch` Callback Binding

- `@Watch('callbackName')` — validate that `callbackName` refers to an existing method in the same component
- Bind the association between the decorated variable and the callback function

### 3.10 eDSL Construct Binding

**Component Creation:**
- Resolve component name to its `@Component struct` symbol
- Bind parameter initialization — match parameter names to component variable declarations
- Validate parameter-variable type compatibility

**ForEach:**
- Bind the array expression — must resolve to `Array<T>`
- Bind the item builder function — parameter types inferred from `T`
- Bind the id function — must return `string`

**if/else in eDSL:**
- Similar to existing if statement binding — no new symbols needed
- The "transparent" parent-child relationship is a checker concern (out of scope)

**`$` prefix / LinkSource:**
- `this.$varName` — resolve `varName` in the component's scope, validate it's a valid `@Link` source (must be `@State`, `@Provide`, `@Link`, etc.)
- `$$.$varName` — resolve `varName` in the builder parameter scope

**`$$` Two-Way Binding:**
- Resolve the expression after `$$` as a state variable reference
- Validate the built-in component attribute supports two-way binding

### 3.11 New SymbolFlags

| Flag | Purpose |
|---|---|
| `SymbolFlagsComponent` | `@Component struct` symbol |
| `SymbolFlagsState` | `@State` decorated variable |
| `SymbolFlagsProp` | `@Prop` decorated variable |
| `SymbolFlagsLink` | `@Link` decorated variable |
| `SymbolFlagsProvide` | `@Provide` decorated variable |
| `SymbolFlagsConsume` | `@Consume` decorated variable |
| `SymbolFlagsObjectLink` | `@ObjectLink` decorated variable |
| `SymbolFlagsObserved` | `@Observed` decorated class |
| `SymbolFlagsStorageLink` | `@StorageLink` decorated variable |
| `SymbolFlagsStorageProp` | `@StorageProp` decorated variable |
| `SymbolFlagsLocalStorageLink` | `@LocalStorageLink` decorated variable |
| `SymbolFlagsLocalStorageProp` | `@LocalStorageProp` decorated variable |
| `SymbolFlagsBuilder` | `@Builder` function |
| `SymbolFlagsGlobalBuilder` | Global `@Builder` function |
| `SymbolFlagsBuilderParam` | `@BuilderParam` variable |
| `SymbolFlagsBuildFunction` | `build()` method |
| `SymbolFlagsExtend` | `@Extend` function |
| `SymbolFlagsAnimatableExtend` | `@AnimatableExtend` function |
| `SymbolFlagsStyles` | `@Styles` function |
| `SymbolFlagsEntry` | `@Entry` decorated component |

### Phase 3 Effort

| Task | Effort |
|---|---|
| New `SymbolFlags` definitions | 1d |
| `@Component struct` binding (class-like + component flag) | 2d |
| Decorated variable binding (10 decorators × rules) | 5d |
| Variable initialization rule validation (local init, parent init, prohibited combos) | 3d |
| `@Observed` class binding | 1d |
| `build()` function binding | 1d |
| `@Builder` function binding (component-level + global, param validation) | 3d |
| `@BuilderParam` variable binding + type validation | 2d |
| `@Extend` / `@AnimatableExtend` function binding | 2d |
| `@Styles` function binding | 1d |
| `@Watch` callback association binding | 2d |
| Component creation binding (name resolution, param matching) | 3d |
| `ForEach` expression binding (array type, item builder, id function) | 3d |
| `$` prefix / LinkSource binding (source validation) | 2d |
| `$$` two-way binding (builder params, component attributes) | 2d |
| `@Entry` component binding + LocalStorage association | 2d |
| Implicit ArkUI decorator scope setup | 2d |
| `ContainerFlags` / `LocalsContainer` updates for new node types | 2d |
| Integration & regression tests | 8d |
| **Subtotal** | **47–56 days** |

---

## Phase 4 — eDSL-Specific Diagnostics & Validation (Parser + Binder)

Additional validation rules that span both parser and binder:

| Validation | Phase |
|---|---|
| Component name must be uppercase, ≥2 chars, unique in app | Parser + Binder |
| Builder naming: lowercase (component) vs uppercase (global) | Parser |
| `@BuilderParam` type must be `BuilderType<C>` | Parser + Binder |
| `@ObjectLink` type must be `@Observed` class | Binder |
| `@Link` source must be valid state variable | Binder |
| `@Provide`/`@Consume` key + type matching | Binder |
| `@StorageLink`/`@LocalStorageLink` key must exist in storage | Binder+Checker |
| Member names must not start with `$` | Parser |
| Member name `$$` forbidden | Parser |
| No constructor in `@Component` | Parser |
| `build()` must exist in `@Component` | Parser |
| `build()` takes no params, returns void | Parser |
| No `static` members in `@Component` | Parser |
| Access modifiers other than `private` are errors | Parser |
| `@Entry` at most one per file | Parser |
| eDSL: exactly one top-level component | Parser |
| eDSL: no `if`/`ForEach` at `build()` top level | Parser |
| eDSL: at most one component per line | Parser |
| eDSL: no variable declarations | Parser |
| eDSL: no `switch` | Parser |
| eDSL: no ternary as statement | Parser |
| eDSL: no state mutation | Parser |
| `ForEach`: item id function must return `string` | Binder |
| `ForEach`: array param must be `Array<T>` | Binder |
| `@Extend` body: only attribute calls allowed | Parser |
| `@Styles` body: only attribute calls allowed | Parser |
| `@Builder` param with `$$` uses by-reference semantics | Parser + Binder |
| Two-way binding `$$` only on supported attributes | Binder+Checker |

### Phase 4 Effort

| Task | Effort |
|---|---|
| Parser-level diagnostics (syntax constraints, naming rules) | 5d |
| Binder-level diagnostics (type constraints, init rules) | 5d |
| Integration tests (full component parsing + binding) | 5d |
| **Subtotal** | **15 days** |

---

## Summary

| Phase | Scope | Effort |
|---|---|---|
| 1 — Scanner | `LanguageVariantArkUI`, new AST Kinds, mode plumbing | **3.5 days** |
| 2 — Parser | eDSL mode, component creation, ForEach, if/else, `$$`, `$`, @Builder, @Extend, @Styles, constraints | **50–57 days** |
| 3 — Binder | Symbol flags, decorator binding, init rules, component resolution, ForEach/if binding, $$ binding | **47–56 days** |
| 4 — Diagnostics | Cross-cutting validation, integration tests | **15 days** |
| **Total** | | **115.5–131.5 days** |

---

## Bottom Line

| Team Size | Duration (Scanner + Parser + Binder only) |
|---|---|
| **One engineer** | **5.5–6.5 months** |
| **Two engineers** | **3–3.5 months** |
| **Three engineers** | **2–2.5 months** |

---

## Combined Estimate: ArkTS 1.1 + ArkUI eDSL

> ArkTS 1.1 numbers updated per [[arkts-1.1-implementation-estimation]] cross-reference with tsgo codebase (Jun 2026). 30.5 days removed: 20.5d already-supported features + 10d checker/SDK-level items.

| Component | ArkTS 1.1 (corrected) | ArkUI eDSL | Combined |
|---|---|---|---|
| Scanner | 7.5d | 3.5d | **11d** |
| Parser | 56d (24 + 32 experimental) | 57d | **113d** |
| Binder | 47.5d (15.5 + 32 shared) | 56d | **103.5d** |
| Shared diagnostics | — | 15d | **15d** |
| **Total** | **79d** | **131.5d** | **242.5d** |

| Team Size | Duration (Combined) |
|---|---|
| **One engineer** | **11–12 months** |
| **Two engineers** | **6 months** |
| **Three engineers** | **3.5–4 months** |

> Note: These are Scanner + Parser + Binder only. Checker, Emitter, and LSP are additional major efforts not included here.

---

## Out of Scope

The following are explicitly excluded from this estimate:

- **Checker (Type Checker):** Full semantic analysis including:
  - State variable dependency tracking (which components depend on which state)
  - UI update lambda generation from state → component mappings
  - `@Observed` class deep observation semantics
  - `SubscribableAbstract` observation protocol
  - `BuilderType<C>` type compatibility checking
  - `@Provide`/`@Consume` ancestor-descendant type matching
  - Storage key existence and type compatibility
  - `AnimatableData` / `IAnimatableArithmetic` type checking
  - Two-way binding `$$` attribute compatibility
  - eDSL parent-child component type validation (e.g., `ListItem` only in `List`)

- **Emitter / Code Generation:** `ace-ets2bundle` compilation:
  - Desugaring component creation to framework constructor calls
  - `$$` desugaring to ES6 Proxy-based by-reference passing
  - `@Builder`/`@BuilderParam` desugaring
  - State observation hooks injection
  - Update lambda generation from dependency mapping

- **LSP / IDE Support:**
  - eDSL-aware completions (component names, attribute functions, event handlers)
  - Go-to-definition for component names → component declaration
  - Hover info for decorators and state variables
  - Diagnostics in eDSL context
  - Syntax highlighting for eDSL constructs

- **Transformers:**
  - `esdecorator` transformer updates for ArkUI decorators
  - `declarations` transformer for component registration
  - `classfields` transformer for observable property wrapping

- **Runtime / Standard Library:**
  - ArkUI framework API type definitions
  - Built-in component type definitions (Text, Column, Row, Button, etc.)
  - `LocalStorage` / `AppStorage` / `PersistentStorage` / `Environment` API types
  - `BuilderType<C>` type definition
  - `SubscribableAbstract` class definition
  - `IAnimatableArithmetic` interface definition

- **ArkTS↔ArkUI Interop:**
  - Mixing regular TypeScript with eDSL
  - Migration tooling from legacy ArkUI patterns

---

## Risk Factors

| Risk | Impact | Mitigation |
|---|---|---|
| eDSL parsing mode complexity — interaction with existing TS parsing | High — could add weeks | Prototype eDSL mode in isolation first; use separate parse functions that don't touch existing TS paths |
| Ambiguity between eDSL component creation and regular TS call expressions | Medium | Clear mode switch at `build()`/`@Builder` entry; parse eDSL body with separate dispatch |
| `$$` syntax ambiguity with regular `$` identifiers | Low | Parser-level disambiguation; scanner unchanged |
| `ForEach` ambiguity with user-defined `ForEach` function | Medium | Only recognized as eDSL construct inside `build()`/`@Builder` body |
| `@Component struct` semantic differences from regular `class` — binder assumptions may break | Medium | Add `SymbolFlagsComponent` to gate component-specific behavior; reuse class infrastructure where possible |
| Decorator semantics (10+ decorators with complex init rules) | Medium | Implement incrementally; start with `@State`/`@Prop`/`@Link`, add others in follow-up phases |
| Existing ArkTS 1.1 changes may conflict or need rework | Medium | Implement ArkUI on top of completed ArkTS 1.1 base; coordinate struct/class handling |

---

## Incremental Implementation Strategy

Rather than implementing all at once, a phased approach within the estimate:

### Increment 1: Minimal Recognizer (≈30% of total)
- `LanguageVariantArkUI`, `@Component struct` recognition, `build()` with empty body
- Simple component creation: `Text("hello")` — no children, no attribute chains
- Basic binder: `@Component` symbol, `build()` symbol
- **Effort:** ~35 days

### Increment 2: Component Tree + Attributes (≈25% of total)
- Component children (container components with bodies)
- Attribute chains (`.width()`, `.fontSize()`, etc.)
- Event handler attachment (`.onClick(() => {...})`)
- Parameter passing to child components
- **Effort:** ~30 days

### Increment 3: State Decorators + Binding (≈25% of total)
- `@State`, `@Prop`, `@Link` binding with init rules
- `$` prefix for Link source
- `@Provide`/`@Consume`
- `@ObjectLink`/`@Observed`
- `@Watch`
- **Effort:** ~30 days  

### Increment 4: Rendering Control + Storage + Builder (≈20% of total)
- eDSL `if`/`else`
- `ForEach`
- `@Builder` / `@BuilderParam` / `$$`
- `@Extend` / `@Styles`
- Storage decorators
- Constraint diagnostics
- **Effort:** ~36.5 days

---

## Sources

- [[arkui-grammar-specification]] — Formal grammar of ArkUI Declarative Core Language
- [[arkui-core-specification]] — ArkUI Core Spec v2.18 (June 28, 2023)
- [[arkts-1.1-specification-analysis]] — ArkTS 1.1 syntax analysis
- [[arkts-1.1-implementation-estimation]] — ArkTS 1.1 estimation baseline
- `internal/scanner/scanner.go` — tsgo scanner implementation
- `internal/parser/parser.go` — tsgo parser implementation
- `internal/binder/binder.go` — tsgo binder implementation
- `internal/core/languagevariant.go` — Language variant mode switching
- `internal/ast/kind_generated.go` — Token and AST node Kinds
- `internal/ast/ast_generated.go` — AST node type definitions
- `_scripts/ast.json` — AST schema definition
