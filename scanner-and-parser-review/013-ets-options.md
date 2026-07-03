# Issue #13 — EtsOptions struct, compiler options, CLI declarations, and tsconfig.json parsing

## Reference

- `_submodules/third_party_typescript/src/compiler/types.ts:6956-7013`
- `_submodules/third_party_typescript/src/compiler/commandLineParser.ts:1171-1524`

## What was ported

### 1. `CompilerOptions` Ets-fields (`types.ts:6956-6977`)

| Reference | Go (`internal/core/compileroptions.go:160-163`) |
|---|---|
| `ets?: EtsOptions` | `Ets *EtsOptions json:"ets,omitzero"` |
| `etsAnnotationsEnable?: boolean` | `EtsAnnotationsEnable core.Tristate json:"etsAnnotationsEnable,omitzero"` |
| `etsLoaderPath?: string` | `EtsLoaderPath string json:"etsLoaderPath,omitzero"` |
| `strictCheckerOnly?: boolean` | `StrictCheckerOnly core.Tristate json:"strictCheckerOnly,omitzero"` |

### 2. `EtsOptions` struct (`types.ts:6981-7013`)

**Go:** `internal/core/compileroptions.go:167-233`

| Reference | Go |
|---|---|
| `render: { method, decorator }` | `Render EtsRenderOptions` |
| `components: string[]` | `Components []string` |
| `libs: string[]` | `Libs []string` |
| `extend: { decorator, components: {name,type,instance}[] }` | `Extend EtsExtendOptions` |
| `styles: { decorator, component: {name,type,instance}, property }` | `Styles EtsStylesOptions` |
| `concurrent: { decorator }` | `Concurrent EtsConcurrentOptions` |
| `customComponent?: string` | `CustomComponent string` |
| `propertyDecorators: { name, needInitialization }[]` | `PropertyDecorators []EtsPropertyDecoratorOptions` |
| `emitDecorators: { name, emitParameters }[]` | `EmitDecorators []EtsEmitDecoratorOptions` |
| `syntaxComponents: { paramsUICallback, attrUICallback: {name,attributes}[] }` | `SyntaxComponents EtsSyntaxComponentsOptions` |

### 3. CLI option declarations (`commandLineParser.ts:1171-1178, 1516-1524`)

**Go:** `internal/tsoptions/declscompiler.go:976-1014`

| Option | Reference | Go |
|---|---|---|
| `ets` | object, description: Unknown_build_option_0 | Object, same flags and description |
| `etsAnnotationsEnable` | boolean, description: Enable_support_of_ETS_annotations | Boolean (Tristate), same description |
| `etsLoaderPath` | field only (not in commandLineParser) | CLI declaration — required by Go test infra, description: Unknown_build_option_0 |
| `strictCheckerOnly` | field only (not in commandLineParser) | CLI declaration — required by Go test infra, description: Unknown_build_option_0 |

### 4. Option parsing (`commandLineParser.ts` and `tsconfigparsing.go`)

**Go:** `internal/tsoptions/parsinghelpers.go:56-73, 247-254`

| Key | Handler |
|---|---|
| `ets` | `parseEtsOptions(value)` |
| `etsAnnotationsEnable` | `ParseTristate(value)` |
| `etsLoaderPath` | `ParseString(value)` |
| `strictCheckerOnly` | `ParseTristate(value)` |

## Implementation differences

### `parseEtsOptions` — direct field extraction

The reference tsc parses nested objects recursively via `convertJsonOptionOfObjectType`. Our codebase uses manual key-by-key parsing through `ParseCompilerOptions`. Fields are extracted directly from `*OrderedMap` using `m.Get()`, same pattern as `parseStringMap` and `parseProjectReference`. All parsing logic lives in `internal/tsoptions/arkts.go` — only the `case "ets"` switch branch in `parsinghelpers.go` calls into it.

### `etsLoaderPath` / `strictCheckerOnly` as CLI declarations

These are CompilerOptions fields in the reference, but not declared in `commandLineParser.ts` — TypeScript allows arbitrary properties via `[option: string]` index signature. Go's test infrastructure (`TestCompilerOptionsDeclaration`) requires every `CompilerOptions` field to have a corresponding CLI declaration entry, so they are declared here.

### Tristate for boolean options

`EtsAnnotationsEnable` and `StrictCheckerOnly` use `core.Tristate` (not `bool`) — consistent with all other boolean compiler options (`AllowJs`, `Strict`, etc.). Parsing uses `ParseTristate(value)` which handles `nil → TSUnknown`, `true → TSTrue`, `false → TSFalse`, matching the reference's tri-state boolean semantics.

### CLI descriptions

Descriptions for `ets`, `etsLoaderPath`, and `strictCheckerOnly` use `diagnostics.Unknown_build_option_0` — matching the reference (`commandLineParser.ts:1177`). `etsAnnotationsEnable` uses `diagnostics.Enable_support_of_ETS_annotations` as in the reference (`commandLineParser.ts:1522`).

## Tests

**Go:** `internal/tsoptions/arkts_test.go` (181 lines) — verifies EtsOptions struct parsing, default values, and syntax component configuration.

## Requirement Scenario

**Scenario 1: Configuring custom component base class**
An ArkTS developer sets `{ "ets": { "customComponent": "MyBaseComponent" } }` in `tsconfig.json`. All struct declarations without explicit `extends` automatically inherit from `MyBaseComponent`. The compiler reads this option via `EtsOptions.CustomComponent`.

**Scenario 2: Configuring decorator names**
`{ "ets": { "extend": { "decorator": ["Extend"] }, "styles": { "decorator": "Styles" }, "render": { "decorator": ["Builder", "LocalBuilder"] } } }` — the parser reads these configured names when detecting decorator-driven contexts. This allows ArkTS to support custom decorator naming conventions.

**Scenario 3: Enabling annotation parsing**
`{ "etsAnnotationsEnable": true }` — enables `@interface` annotation parsing in `.ets` files. When false (or absent), `@interface` is parsed as a regular (illegal) decorator.

## Target Users

- **ArkTS project maintainers** — configure ArkTS compilation behavior via tsconfig.json
- **Compiler developers** — access EtsOptions in parser, checker, and emitter code

## Restrictions & Constraints

- **Type mapping**: `EtsAnnotationsEnable` and `StrictCheckerOnly` use `Tristate` (not `bool`) for consistency with other boolean compiler options
- **JSON serialization**: `omitzero` ensures unset fields are omitted from serialized output
- **Nil-safety**: `Ets` is a pointer (`*EtsOptions`) — all consumers must nil-check before accessing sub-fields
- **CLI declarations**: `etsLoaderPath` and `strictCheckerOnly` are declared as CLI options even though the reference does not (required by Go test infrastructure)

## Acceptance Strategy

| Criterion | Verification |
|-----------|-------------|
| EtsOptions struct parses correctly from tsconfig | `internal/tsoptions/arkts_test.go` (181 lines) |
| CompilerOptions fields wired to CLI declarations | `TestCompilerOptionsDeclaration` passes |
| Default values and sub-structs populated correctly | Unit tests in `arkts_test.go` |
| Nil Ets pointer handled safely across codebase | Compilation and tests pass with Ets=nil |
| No regression in existing tsconfig parsing | `go test ./internal/tsoptions/...` — zero failures |
| AST and diagnostics match reference tsc | `arkts_cmp_test.go` verifies EtsOptions-dependent parser behavior matches reference |
