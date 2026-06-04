# ArkUI Declarative Core Language — Formal Grammar Specification

Based on: ArkUI Declarative Core Language Specification (v2.18, June 28, 2023)
Editor: Guido Grassel, Huawei RnD Finland

This document defines the formal grammar of the ArkUI Declarative eDSL (embedded Domain-Specific Language). ArkUI is a superset of ArkTS/TypeScript — standard TS lexical and syntactic rules apply except where explicitly overridden below.

---

## 1. Lexical Grammar

### 1.1 Decorators (ArkUI-Specific)

```
Decorator ::=
    '@Component'
  | '@Entry' ['(' LocalStorageExpression ')']
  | '@State'
  | '@Prop'
  | '@Link'
  | '@Provide' ['(' StringLiteral ')']
  | '@Consume' ['(' StringLiteral ')']
  | '@ObjectLink'
  | '@Observed'          (* class decorator *)
  | '@Builder'
  | '@BuilderParam'
  | '@Extend' '(' BuiltInComponentName ')'
  | '@Styles'
  | '@AnimatableExtend' '(' BuiltInComponentName ')'
  | '@Watch' '(' FunctionName ')'
  | '@StorageLink' '(' StringLiteral ')'
  | '@StorageProp' '(' StringLiteral ')'
  | '@LocalStorageLink' '(' StringLiteral ')'
  | '@LocalStorageProp' '(' StringLiteral ')'
  | '@Entry'
```

Multiple decorators can apply to a single declaration; order is significant:
- `@Entry @Component struct ...` — valid
- `@Component @Entry struct ...` — valid
- A variable may carry `@State @Watch(...)` simultaneously

### 1.2 Reserved Identifiers in ArkUI Context

Member names must NOT start with `$` (reserved for `@Link` source references).
Member name `$$` is forbidden (reserved for `@Builder` parameter passing).

### 1.3 Soft Keywords (Context-Dependent)

The following identifiers have special meaning only inside `build()` and `@Builder` function bodies:

| Identifier | Context |
|---|---|
| `if` / `else` / `else if` | Conditional rendering (not TS `if` statement) |
| `ForEach` | Repeated content rendering |
| `$$` | Builder function parameter object |
| `this.$<var>` | `@Link` source reference syntax |

---

## 2. Compilation Unit Structure

```
CompilationUnit ::=
    { ImportDeclaration }
    { GlobalBuilderFunction | ExtendDeclaration | StylesDeclaration | TopLevelStatement }
    { ComponentDeclaration }
```

An ArkUI source file containing an `@Entry` component defines a page.

### 2.1 Imports

Standard ArkTS/TypeScript import syntax. The following are additionally relevant:

```
ImportDeclaration ::=
    'import' '{' ImportSpecifier {',' ImportSpecifier} '}' 'from' StringLiteral ';'
```

ArkUI decorators are imported implicitly through the framework — no explicit import required.

---

## 3. Component Declarations

### 3.1 Custom Component

```
ComponentDeclaration ::=
    '@Component'
    'struct' ComponentName
    '{'
        { ComponentMember }
    '}'

ComponentName ::=
    IdentifierStartWithUpperCase   (* must be >= 2 chars, unique in application *)
```

**Constraints:**
- No explicit constructor allowed (compiler generates one).
- All members are implicitly `private`. Explicit `private` is optional; any other access modifier is a syntax error.
- `static` members are forbidden.
- Member names must not start with `$`.

### 3.2 Entry Component

```
EntryComponentDeclaration ::=
    '@Entry' ['(' LocalStorageInstance ')']
    ComponentDeclaration
```

At most one `@Entry` component per source file.

### 3.3 Component Members

```
ComponentMember ::=
    DecoratedVariableDeclaration
  | RegularVariableDeclaration
  | BuildFunction
  | BuilderFunction
  | LifecycleFunction
  | OtherMemberFunction
```

### 3.4 Decorated Variable Declarations

```
DecoratedVariableDeclaration ::=
    {Decorator}
    Identifier ':' TypeAnnotation
    ['=' Expression]
    ';'

(* Decorators applicable to variables: *)
(* @State, @Prop, @Link, @Provide, @Consume, @ObjectLink *)
(* @StorageLink(key), @StorageProp(key) *)
(* @LocalStorageLink(key), @LocalStorageProp(key) *)
(* @BuilderParam *)
(* @Watch(callbackName) — must be combined with another state decorator *)
```

**Type constraints per decorator:**

| Decorator | Allowed Types `T` |
|---|---|
| `@State` | `T` |
| `@Prop` | `T` (must match sync source type) |
| `@Link` | `T` (must match sync source type) |
| `@Provide` | `T` |
| `@Consume` | `T` (must match provide type + key/name) |
| `@ObjectLink` | Class type decorated with `@Observed` |
| `@StorageLink(k)` | `T` (persisted in AppStorage) |
| `@StorageProp(k)` | `T` (persisted in AppStorage, one-way) |
| `@LocalStorageLink(k)` | `T` (persisted in LocalStorage) |
| `@LocalStorageProp(k)` | `T` (persisted in LocalStorage, one-way) |
| `@BuilderParam` | `BuilderType<C>` where `C extends Object` |

Where `T` = `class | SubscribableAbstract | number | boolean | string | enum | Date | Map<K,V> | Set<V>` and `Array<X>` where `X ∈ T`.

### 3.5 Regular Variable Declaration

```
RegularVariableDeclaration ::=
    Identifier ':' TypeAnnotation
    ['=' Expression]
    ';'
```

Non-decorated variables are NOT observed by the framework. Changes do NOT trigger UI updates.

---

## 4. Build Function

### 4.1 Definition

```
BuildFunction ::=
    'build' '(' ')' '{'
        BuildBodyStatement*
    '}'
```

- Takes no parameters.
- Returns `void` (implicitly).
- Exactly one in every `@Component`.

### 4.2 Build Body (the eDSL)

```
BuildBodyStatement ::=
    ComponentCreation
  | IfStatement
  | ForEachStatement
  | BuilderCall
  | Comment

BuildBody ::=
    BuildBodyStatement*
```

### 4.2.1 Top-Level Constraint

The `build()` body MUST contain exactly ONE top-level component creation. `if` and `ForEach` are NOT allowed at the top level of `build()`.

**Valid:**
```typescript
build() {
    Column() { ... }
}
```

**Invalid:**
```typescript
build() {
    Text("a")
    Text("b")    // ERROR: second top-level component
}
build() {
    if (cond) { ... }  // ERROR: if at top level
}
```

### 4.2.2 One Component Per Line

At most one component creation statement per source line. Multiple components on one line separated by `;` is a syntax error.

### 4.2.3 Forbidden Constructs Inside Build

The following are NOT allowed inside `build()` and `@Builder` function bodies:
- Variable declarations (`let`, `const`, `var`)
- Block scope creation (`{ ... }` standalone)
- Regular function calls (except those explicitly permitted)
- `console.log` and similar side-effecting built-ins
- `switch` statements (use `if`/`else`)
- Ternary expressions as statements (`cond ? CompA() : CompB()`)
- `new` keyword for component creation
- Any mutation of application state (e.g. `this.count++` inside expressions)

### 4.2.4 Permitted Function Calls Inside Build

Only these calls are allowed:
1. `@Builder` function calls (component-level: `this.builderName(...)`, global: `GlobalBuilderName(...)`)
2. `@BuilderParam` invocation: `this.builderParamVar(...)` or `$$.builderParamVar(...)`
3. Attribute function chain calls (`.width(...)`, `.height(...)`, etc.)
4. Event handler attachment (`.onClick(...)`, `.onChange(...)`, etc.)
5. `@Extend` attribute function calls
6. `@Styles` function calls
7. Sub-component creation: `ChildComponent({...})`
8. TS pure functions used as parameter value sources: `Text(this.calcValue())`

---

## 5. Component Creation

### 5.1 Syntax

```
ComponentCreation ::=
    ComponentName [ParameterList]
    '{'
        [BuildBodyStatement*]
    '}'
    {AttributeChain}
```

The short form without body is allowed for leaf components:
```
Text("hello")
    .fontSize(20)
    .fontColor(Color.Red)
```

### 5.2 Parameter Passing to Child Component

```
ParameterList ::=
    '('
        [NamedParameter {',' NamedParameter}]
    ')'

NamedParameter ::=
    Identifier ':' Expression
```

Parameters initialize child component variables. Only variables defined in the child component are allowed. Each decorator defines whether initialization from parent is required, optional, or prohibited.

### 5.3 `$` Prefix — `@Link` Source Reference

To pass a `@Link` source from parent to child, use the `$` prefix:

```
LinkSourceExpression ::=
    'this' '.' '$' Identifier
```

Example:
```typescript
ChildComponent({ linkVar: this.$parentStateVar })
```

The `$` creates a reference to the state variable (not its value). This is syntactic sugar compiled by `ace-ets2bundle`.

---

## 6. Attribute Chain

### 6.1 Syntax

```
AttributeChain ::=
    { '.' AttributeFunctionCall }

AttributeFunctionCall ::=
    FunctionName '(' [Expression {',' Expression}] ')'
```

Attribute functions are chainable methods that configure component properties, styling, and event handlers. They always follow the component creation on the same or subsequent lines.

### 6.2 Event Handler Attachment

```
EventHandlerAttachment ::=
    '.' EventName '(' ArrowFunctionBody ')'

EventName ::=
    'onClick' | 'onChange' | 'onAppear' | 'onDisAppear'
  | 'onTouch' | 'onKeyEvent' | 'onDrag' | 'onDrop'
  | (* ... other built-in events *)
```

Event handlers mutate `@Component` state. State changes trigger asynchronous UI re-render.

### 6.3 Attribute Function by Reference (for `@Builder` parameters)

When passing a state variable to an attribute function, the framework records a dependency:

```
SomeComponent()
    .someAttribute(this.stateVar)
```

A "get" on `this.stateVar` is registered — when the variable changes, the component re-renders.

---

## 7. Builder Functions (`@Builder`)

### 7.1 Component-Level Builder

```
ComponentBuilderFunction ::=
    '@Builder'
    IdentifierStartWithLowerCase '(' '$$' ':' ParameterType ')'   (* by-reference *)
  | '@Builder'
    IdentifierStartWithLowerCase '(' [Parameter {',' Parameter}] ')'   (* by-value, deprecated *)
    '{'
        BuildBodyStatement*
    '}'
```

- Name must start with **lowercase** letter, >= 2 chars.
- Called via `this.builderName(...)`.
- `this` refers to the owning component instance.
- Body follows same rules as `build()`.
- By-reference (using `$$`) is the **required** style; by-value is deprecated.

### 7.2 Global Builder Function

```
GlobalBuilderFunction ::=
    '@Builder'
    'function' IdentifierStartWithUpperCase '(' '$$' ':' ParameterType ')'
    '{'
        BuildBodyStatement*
    '}'
```

- Name must start with **uppercase** letter, >= 2 chars.
- Name must be unique across the application.
- Called via `GlobalBuilderName(...)` (without `this`).
- `this` is NOT available inside the function body.
- Cannot use `bind(this)`.

### 7.3 Builder Parameter Passing — `$$` Mechanism

```
BuilderByReferenceParam ::=
    '$$' ':' TypeAnnotation

(* Inside builder body, access parameters via $$.paramName *)
BuilderParamAccess ::=
    '$$' '.' Identifier

(* For passing @Link sources through a builder: *)
BuilderLinkPassing ::=
    '$$' '.' '$' Identifier
```

Example:
```typescript
@Builder function MyBuilder($$ : { label: string, linkSource: string }) {
    Text($$.label)
    CreatedByBuilder({ link: $$.$linkSource })
}
```

### 7.4 Builder Function Call

```
BuilderCall ::=
    'this' '.' BuilderFunctionName '(' [NamedParameterList] ')'     (* component-level *)
  | GlobalBuilderFunctionName '(' [NamedParameterList] ')'            (* global *)
```

Short-hand for parameterless builder:
```
@Builder myBuilder() { ... }     (* desugars to @Builder myBuilder($$ : {}) *)
this.myBuilder()                 (* valid call *)
```

---

## 8. BuilderParam (`@BuilderParam`)

### 8.1 Declaration

```
BuilderParamDeclaration ::=
    '@BuilderParam'
    Identifier ':' 'BuilderType' '<' TypeAnnotation '>'
    ['=' BuilderFunctionReference]
    ';'
```

`BuilderType<C>` is defined by the framework as:
```typescript
type BuilderType<C extends Object> = ($$ : C) => void
```

### 8.2 Initialization

- Can be initialized locally with a `@Builder` function reference.
- Can be initialized from parent via `ChildComponent({ builderParam: this.someBuilder })`.
- Default value is `undefined` — framework skips execution when undefined.
- Assigning a non-`@Builder` function is a syntax error.

### 8.3 Invocation

```
BuilderParamCall ::=
    'this' '.' BuilderParamName '(' [NamedParameterList] ')'    (* from build() *)
  | '$$' '.' BuilderParamName '(' [NamedParameterList] ')'       (* from @Builder *)
```

The type of the `@BuilderParam` and the assigned builder function must match exactly.

---

## 9. Extend Declarations (`@Extend`)

### 9.1 Syntax

```
ExtendDeclaration ::=
    '@Extend' '(' BuiltInComponentName ')'
    'function' FunctionName '(' [Parameter {',' Parameter}] ')'
    '{'
        { AttributeFunctionCall }
    '}'
```

### 9.2 Constraints

- Applies only to built-in (framework) components, NOT custom components.
- Function name should be descriptive (globally visible to all instances of the extended component).
- Function body may ONLY contain attribute function calls (chained via `.`).
- `this` is NOT available.
- `if`/`ForEach`/other statements are NOT allowed.
- Must be defined before first use.

### 9.3 Usage

The extended function is used like any pre-defined attribute:
```typescript
Text("hello")
    .fancy(Color.Red)
```

### 9.4 AnimatableExtend

```
AnimatableExtendDeclaration ::=
    '@AnimatableExtend' '(' BuiltInComponentName ')'
    'function' FunctionName '(' [Parameter {',' Parameter}] ')'
    '{'
        { AttributeFunctionCall }
    '}'
```

Special variant of `@Extend` that supports animation interpolation via `IAnimatableArithmetic`.

---

## 10. Styles Declarations (`@Styles`)

```
StylesDeclaration ::=
    '@Styles'
    'function' FunctionName '(' ')'
    '{'
        { AttributeFunctionCall }
    '}'
```

Defines a reusable set of attribute functions. Details still being specified (marked TBD in core spec v2.18).

---

## 11. Rendering Control Syntax

### 11.1 Conditional Rendering (`if` / `else`)

```
IfStatement ::=
    'if' '(' Expression ')'
    '{'
        BuildBodyStatement*
    '}'
    {'else' 'if' '(' Expression ')' '{' BuildBodyStatement* '}'}
    ['else' '{' BuildBodyStatement* '}']

Expression ::=
    BooleanExpression   (* must use at least one state variable *)
```

**Constraints:**
- `if`/`else`/`else if` is a **rendering control** construct, NOT the standard TypeScript `if` statement.
- Available only inside container components, NOT at `build()` top level.
- Each branch body is a build function — follows all build function rules.
- Each branch must create **one or more** components. Empty branch is a syntax error.
- `if` is "transparent" regarding parent-child relationships of components.
- When the branch changes, all sub-components of the old branch are **deleted** (state not preserved).

**Update semantics:**
1. State variable used in condition changes.
2. Conditions re-evaluated.
3. If branch changed: delete old branch components, build new branch components.

### 11.2 Repeated Content (`ForEach`)

```
ForEachStatement ::=
    'ForEach' '('
        ArrayExpression ','
        ItemBuildFunction ','
        [ItemIdFunction]
    ')'

ArrayExpression ::=
    Expression   (* must evaluate to Array<T>, can be function returning array *)

ItemBuildFunction ::=
    '(' Identifier [',' 'index'? ']'? Identifier? ')' '=>' '{'
        BuildBodyStatement*
    '}'

(* Note: the index parameter syntax is informal — actual syntax: *)
(* (item, index?) => { ... } *)
(* Type annotations on parameters are omitted. *)

ItemIdFunction ::=
    '(' Identifier [',' Identifier]? ')' '=>' Expression   (* must return string *)
```

**Constraints:**
- First parameter: `Array<T>`. Empty array is allowed (renders nothing).
- Second parameter (item build function): follows build function rules. Must create ≥1 components.
- Third parameter (id function, optional but **strongly recommended**): returns a **unique, persistent** string id for each array item.
- `ForEach` is NOT allowed at `build()` top level.
- `ForEach` is "transparent" regarding parent-child relationships.
- When the array changes: only newly added items trigger build; retained items are moved (not rebuilt); deleted items have their components removed.
- Functions modifying array in-place (`.sort()`, `.splice()`, `.reverse()`) must NOT be used directly as the array parameter (they mutate state).

**Item build function with index:**
```
(item: T, index: number) => { ... }
```
- `index` — optional second parameter, type `number`.
- Using `index` degrades update performance significantly.
- If `index` is used in the build function, it should also appear in the id function.

**Nested ForEach:**
Nesting `ForEach` inside `ForEach` in the same component is allowed but discouraged. Prefer splitting into sub-components.

---

## 12. Two-Way Binding Syntax (`$$`)

### 12.1 Built-in Component Two-Way Sync

```
TwoWayBinding ::=
    Identifier ':' '$$' Expression
```

Used with built-in components that support two-way value synchronization (e.g., text input fields):

```typescript
TextInput({ text: $$this.inputValue })
```

### 12.2 Builder Function `$$` (Parameter Object)

Inside `@Builder` function declarations, `$$` is the parameter name for by-reference passing:

```typescript
@Builder function MyBuilder($$ : { label: string, value: number }) {
    Text($$.label)
    // $$.label — access parameter value
}
```

### 12.3 `$` Prefix — Link Source Syntax

```
LinkSource ::=
    'this' '.' '$' Identifier
```

Creates a reference to a state variable for `@Link` initialization:
```typescript
ChildComponent({ linkVar: this.$parentStateVar })
```

---

## 13. Lifecycle Callbacks

### 13.1 Syntax

```
LifecycleFunction ::=
    'aboutToAppear' '(' ')' ':' 'void' '{' FunctionBody '}'
  | 'aboutToDisappear' '(' ')' ':' 'void' '{' FunctionBody '}'
```

### 13.2 Semantics

- `aboutToAppear()`: Called before the component's first `build()` execution.
- `aboutToDisappear()`: Called before the component is deleted (due to `if` branch change, `ForEach` update, or page navigation).

---

## 14. Observed Class Decorator (`@Observed`)

### 14.1 Syntax

```
ObservedClassDeclaration ::=
    '@Observed'
    'class' ClassName '{'
        { ClassMember }
    '}'
```

`@Observed` decorates a class to enable deep observation of property changes. Required for classes used with `@ObjectLink`.

### 14.2 `@ObjectLink` Variable

```
ObjectLinkDeclaration ::=
    '@ObjectLink'
    Identifier ':' ObservedClassName
    ';'
```

- The source in the parent must be an `@Observed` class instance from an `@State`/`@Provide`/`@Link` array item or object property.
- The class of `@ObjectLink` must match the class of the sync source.

---

## 15. State Storage Decorators

### 15.1 AppStorage Decorators

```
StorageLinkDeclaration ::=
    '@StorageLink' '(' StringLiteral ')'
    Identifier ':' TypeAnnotation
    ['=' Expression]
    ';'

StoragePropDeclaration ::=
    '@StorageProp' '(' StringLiteral ')'
    Identifier ':' TypeAnnotation
    ['=' Expression]
    ';'
```

- `@StorageLink(key)`: Two-way sync with `AppStorage`.
- `@StorageProp(key)`: One-way sync from `AppStorage` (value flows AppStorage → component).

### 15.2 LocalStorage Decorators

```
LocalStorageLinkDeclaration ::=
    '@LocalStorageLink' '(' StringLiteral ')'
    Identifier ':' TypeAnnotation
    ['=' Expression]
    ';'

LocalStoragePropDeclaration ::=
    '@LocalStorageProp' '(' StringLiteral ')'
    Identifier ':' TypeAnnotation
    ['=' Expression]
    ';'
```

- `@LocalStorageLink(key)`: Two-way sync with `LocalStorage`.
- `@LocalStorageProp(key)`: One-way sync from `LocalStorage`.

### 15.3 `@Entry` with LocalStorage

```
EntryWithLocalStorage ::=
    '@Entry' '(' LocalStorageInstance ')'
    ComponentDeclaration
```

Passes a `LocalStorage` instance to the entry component and its subtree.

---

## 16. `@Watch` Decorator

### 16.1 Syntax

```
WatchDecorator ::=
    '@Watch' '(' Identifier ')'
```

Must be combined with another state decorator:

```
@State @Watch('onCountChange') count : number = 0;
```

The referenced function is called whenever the decorated variable changes.

---

## 17. `$$` Two-Way Value Synchronization with Built-in Components

### 17.1 Syntax

```
TwoWaySyncAttribute ::=
    AttributeName ':' '$$' Expression

(* Only valid for built-in components that support two-way binding *)
```

### 17.2 Constraints

- `$$` is only valid with specific built-in component attributes (e.g., `TextInput.text`).
- The expression must be a state variable reference.
- Not valid with custom components.

---

## 18. SubscribableAbstract

### 18.1 Definition

```
SubscribableAbstractClass ::=
    'class' ClassName 'extends' 'SubscribableAbstract' '{'
        { ClassMember }
    '}'
```

Allows application-defined classes to participate in ArkUI state observation. Properties of a `SubscribableAbstract` subclass are observed individually.

---

## 19. Initialization and Update Rules

### 19.1 Variable Initialization Order

For each component instance:
1. Local default values (document order)
2. Constructor parameters from parent (if supplied)
3. First `build()` execution (records state variable → UI component dependencies)

### 19.2 Initialization by Decorator

| Decorator | Local Init | Init from Parent | Notes |
|---|---|---|---|
| `@State` | Required | Prohibited | Component owns the state |
| `@Prop` | Optional | Optional | One-way sync from parent |
| `@Link` | Prohibited | Required (via `$`) | Two-way sync with parent |
| `@Provide` | Required | Prohibited | Like `@State` but available to descendants |
| `@Consume` | Prohibited | From ancestor `@Provide` | Matched by name/alias + type |
| `@ObjectLink` | Prohibited | Required | Must be `@Observed` class instance |
| `@StorageLink(k)` | Optional | N/A | Synced with AppStorage key |
| `@StorageProp(k)` | Optional | N/A | Synced from AppStorage key |
| `@LocalStorageLink(k)` | Optional | N/A | Synced with LocalStorage key |
| `@LocalStorageProp(k)` | Optional | N/A | Synced from LocalStorage key |
| `@BuilderParam` | Optional | Optional | Builder function reference |
| Regular (non-decorated) | Optional | Optional | Not reactive |

---

## 20. Summary: Complete Grammar (EBNF)

```
(* ===== Top Level ===== *)

CompilationUnit ::=
    { ImportDeclaration }
    { GlobalBuilderFunction | ExtendDeclaration | AnimatableExtendDeclaration | StylesDeclaration }
    { EntryComponentDeclaration | ComponentDeclaration }

(* ===== Component ===== *)

ComponentDeclaration ::=
    '@Component' 'struct' ComponentName '{' { ComponentMember } '}'

EntryComponentDeclaration ::=
    '@Entry' ['(' Expression ')'] ComponentDeclaration

ComponentMember ::=
    DecoratedVariable
  | BuildFunction
  | BuilderFunction
  | LifecycleFunction
  | RegularMemberFunction
  | RegularVariable

(* ===== Variables ===== *)

DecoratedVariable ::=
    {Decorator} Identifier ':' TypeAnnotation ['=' Expression] ';'

Decorator ::=
    '@State'
  | '@Prop'
  | '@Link'
  | '@Provide' ['(' StringLiteral ')']
  | '@Consume' ['(' StringLiteral ')']
  | '@ObjectLink'
  | '@StorageLink' '(' StringLiteral ')'
  | '@StorageProp' '(' StringLiteral ')'
  | '@LocalStorageLink' '(' StringLiteral ')'
  | '@LocalStorageProp' '(' StringLiteral ')'
  | '@BuilderParam'
  | '@Watch' '(' Identifier ')'

(* ===== Build Function ===== *)

BuildFunction ::=
    'build' '(' ')' '{' BuildBody '}'

BuildBody ::=
    ComponentCreation

(* Inside container components, the build body may also include: *)
ContainerBody ::=
    { ComponentCreation | IfStatement | ForEachStatement | BuilderCall | Comment }

ComponentCreation ::=
    ComponentName [ParameterList]
    ['{' ContainerBody '}']
    { '.' AttributeFunctionCall }

ParameterList ::=
    '(' [NamedParameter {',' NamedParameter}] ')'

NamedParameter ::=
    Identifier ':' Expression

LinkSourceExpression ::=
    'this' '.' '$' Identifier

(* ===== Attribute Chain ===== *)

AttributeFunctionCall ::=
    FunctionName '(' [Expression {',' Expression}] ')'

EventHandlerAttachment ::=
    '.' EventName '(' '(' [ParameterList] ')' '=>' '{' FunctionBody '}' ')'

(* ===== Builder Functions ===== *)

BuilderFunction ::=
    '@Builder' BuilderName '(' '$$' ':' TypeAnnotation ')'
    '{' ContainerBody '}'

GlobalBuilderFunction ::=
    '@Builder' 'function' BuilderName '(' '$$' ':' TypeAnnotation ')'
    '{' ContainerBody '}'

BuilderCall ::=
    'this' '.' BuilderName '(' [NamedParameterList] ')'     (* component-level *)
  | GlobalBuilderName '(' [NamedParameterList] ')'            (* global *)

BuilderParamType ::=
    'BuilderType' '<' TypeAnnotation '>'

(* ===== Rendering Control ===== *)

IfStatement ::=
    'if' '(' Expression ')'
    '{' ContainerBody '}'
    { 'else' 'if' '(' Expression ')' '{' ContainerBody '}' }
    [ 'else' '{' ContainerBody '}' ]

ForEachStatement ::=
    'ForEach' '('
        Expression ','
        '(' Identifier [',' Identifier]? ')' '=>' '{' ContainerBody '}'
        [',' '(' Identifier [',' Identifier]? ')' '=>' Expression]
    ')'

(* ===== Extend & Styles ===== *)

ExtendDeclaration ::=
    '@Extend' '(' BuiltInComponentName ')'
    'function' Identifier '(' [Parameter {',' Parameter}] ')'
    '{' { '.' AttributeFunctionCall } '}'

AnimatableExtendDeclaration ::=
    '@AnimatableExtend' '(' BuiltInComponentName ')'
    'function' Identifier '(' [Parameter {',' Parameter}] ')'
    '{' { '.' AttributeFunctionCall } '}'

StylesDeclaration ::=
    '@Styles'
    'function' Identifier '(' ')'
    '{' { '.' AttributeFunctionCall } '}'

(* ===== Lifecycle ===== *)

LifecycleFunction ::=
    'aboutToAppear' '(' ')' ':' 'void' '{' FunctionBody '}'
  | 'aboutToDisappear' '(' ')' ':' 'void' '{' FunctionBody '}'

(* ===== Observed Classes ===== *)

ObservedClassDeclaration ::=
    '@Observed' 'class' ClassName '{' { ClassMember } '}'

(* ===== Two-Way Binding ===== *)

TwoWayBinding ::=
    Identifier ':' '$$' Expression

BuilderParamAccess ::=
    '$$' '.' Identifier

BuilderLinkPassing ::=
    '$$' '.' '$' Identifier
```

---

## 21. Type System Constraints (ArkUI-Specific)

### 21.1 Union Types

API 10: permissive type checking.
API 11+: strict type checking. Union types in decorator variable type annotations: explicit handling rules apply.

### 21.2 `undefined` and `null`

- `@State`, `@Link`, `@Prop`, `@Provide`, `@Consume` support `undefined` and `null` (API 10+).
- `@ObjectLink` does NOT permit `undefined`/`null`.
- `@Builder` parameter values must not be `undefined`/`null`.

### 21.3 Map and Set

`Map<K,V>` and `Set<V>` are supported types for `@State`, `@Link`, `@Prop`, `@Provide`, `@Consume` (API 10+). Mutations via `.set()`, `.delete()`, `.add()`, `.clear()` are observed.

### 21.4 Date

`Date` type is supported for `@State`, `@Link`, `@Prop`, `@Provide`, `@Consume`, `@ObjectLink` (API 9+). Storage-related decorators support since API 10.

---

## References

- ArkUI Core Spec: https://gitee.com/arkui-finland/arkui-edsl-core-spec
- General UI Spec: `general-ui-spec.md`
- State Management Intro: `intro-state-mgmt.md`
- Component State Management: `manage-state-component.md`
- UI State Storages: `ui-state-storages.md`
- Other State Management: `other-state-mgmt.md`
- Rendering Control: `rendering-control-syntax.md`
- SwiftUI Comparison: `swiftui-feature-comparison.md`

---

*Generated from ArkUI Declarative Core Language Specification v2.18 (June 28, 2023).*
*Grammar notation: Extended Backus-Naur Form (EBNF) with explanatory prose.*
