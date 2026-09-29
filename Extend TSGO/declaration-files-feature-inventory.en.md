# [ArkTS][Declaration Files] Feature Inventory for tsgo

> Source of truth: `declaration-files-handoff.en.md` (the English original). Every item below is traceable to a section of that document; the section is named per item. The repository rules in §7 are quoted from `AGENTS.md` and are not part of the handoff.
>
> Scope: the ten delivery items of §2 of the handoff, the prerequisites they cannot pass acceptance without, and the decisions that change the size of the work. §5 lists what is deliberately not in this inventory.
>
> Requirement IDs are stable: `F<n>-<m>` is the m-th requirement of feature n, `P<n>` is a prerequisite, `D<n>` is an open decision. Reference locations are given as "file + function" — never line numbers — matching the handoff.
>
> §8 maps every feature and prerequisite to the implementation that already exists in the OH fork (`D:\Work\tsgo\typescript-go_OH`, remote `k1ngqaquuu/typescript-go_OH`). §9 recalculates the backlog under the assumption that all of it — main, the orchestration week branch, the 47 branch, the profile-metrics branch and the changes of the open pull requests — lands in this repository. Read §9 for what is genuinely left to do.

---

## 1. Summary

| ID | Feature | tsgo landing site | Handoff | Blocked by | Existing work |
|---|---|---|---|---|---|
| F1 | Annotation declarations and annotation properties in declaration files | `transformers/declarations/{transform,util}.go` + new `arkts.go` | 2.1, rows 1–7 | P1 | #53 |
| F2 | Decorator retention per the `ets.emitDecorators` allowlist | `transformers/declarations/transform.go` + new `arkts.go` | 2.2, rows 8–10 | — | #53 |
| F3 | `@Sendable` kept verbatim, bypassing the allowlist | same as F2 | 2.3, row 11 | — | #53 |
| F4 | Visibility trimming and visibility checks for annotation references | `transformers/declarations/{transform,util,diagnostics}.go` + new `arkts.go` | 2.4, rows 7, 12–14 | P1 (annotation half) | #53 |
| F5 | `export` stripping inside ambient namespaces, keeping decorators and annotations | `transformers/declarations/transform.go` + new `arkts.go` | 2.5, row 15 | — | #53, partial |
| F6 | Output shape: triple-slash references, portable specifiers, `Resource` | `transformers/declarations/transform.go`; `checker/nodebuilderimpl.go` + new `arkts.go`; `modulespecifiers` (table entries only) | 2.6, rows 16–18 | P4 | #64, partial |
| F7 | Output set and publishing path by module form | new build-facing layer (D4) | 2.7 | P5, D4 | #56, different interface |
| F8 | Collecting and recording declaration outputs; the two write entry points | new build-facing layer (D4) + `oh-exports` channel | 2.8 | P5, P6, D4 | #56, different interface |
| F9 | Source-output extension by `useTsHar` and its place in the HAR | new build-facing layer (D4) | 2.9 | P5, D4 | #56 + #63 |
| F10 | Declaration diagnostics for library builds | build-facing gate (D4) | 2.10 | P5, D4 | — |

Verification of F1–F6 additionally needs P2 and P3; acceptance criterion 1 additionally needs P7 and is partly blocked by P8.

**Prerequisites**

| ID | What must exist first | Blocks | Existing work |
|---|---|---|---|
| P1 | The checker understands annotation declarations | F1, annotation half of F4 | #53 |
| P2 | The `ets` test configuration of the 12 declaration cases | verification of F1–F6 | partial |
| P3 | A golden mechanism for emit text | verification of every feature | partial |
| P4 | The specifier machinery understands `oh_modules` | portable-specifier part of F6 | #64 |
| P5 | Nine inputs agreed with the build system | F7–F10 | different shape, #56 |
| P6 | The `oh-exports` data channel | F8 | open |
| P7 | The 12 reference baselines wired into a runner | acceptance criterion 1 | not found |
| P8 | Struct declaration emit (ArkUI stream, out of scope here) | 10 of the 12 baselines in criterion 1 | #53 |

---

## 2. Prerequisites

### P1 — The checker understands annotation declarations

Today annotation declarations in tsgo are bound as Class, `isArkTSClassLike` in `internal/ast/arkts.go` covers only Struct, and the combination of `@interface` with `@Anno class` panics in the checker; `getDeclarationSpaces` in `internal/checker/checker.go` has no case for it and panics on its last line. Without both, F1 cannot pass acceptance. Whether these two count toward this requirement or toward their own piece of work is D8.

The same area owns the two checker-side diagnostics of §1.3: TS28033 (annotation property type may only be number / boolean / string / const enum / arrays of those) and TS28034 — both missing in tsgo today, both reported by `checkAnnotationPropertyDeclaration` in the reference.

### P2 — The `ets` test configuration of the 12 declaration cases

`tests/dets/tsconfig.json` carries `declaration: true` + `emitDeclarationOnly: true`, 19 `emitDecorators` entries (all `emitParameters: false`, including `Entry` / `Component` / `State` / `Styles` / `Builder` / `Observed`), 4 `propertyDecorators`, the `render` method and decorator, about 100 `components`, the full `extend.components` mapping, `styles`, `customComponent: "CustomComponent"`, `syntaxComponents`, `target: es2021` / `module: commonjs`. Without bringing this configuration over, F2 — the only item that can start immediately and has golden backing — cannot be verified.

### P3 — A golden mechanism for emit text

tsgo's comparison tests today compare AST, symbol tables and diagnostics, not output text; everything this requirement is accepted on is output text. This infrastructure cost belongs in the schedule explicitly.

### P4 — The specifier machinery understands `oh_modules`

The portable-specifier requirement of F6 (F6-2, F6-3) needs the generator that derives a specifier from a file path to know the `oh_modules` package directory and the `oh-package.json5` manifest — the four pieces `getAllModulePathsWorker`, `getNearestAncestorDirectoryWithPackageJson`, `getNodeModulePathParts`, `tryDirectoryWithPackageJson` in the reference's `moduleSpecifiers.ts`. The tsgo counterpart is `internal/modulespecifiers`. This is the same machinery as "make the compiler understand `oh_modules`" (same `packageManagerType` option, same `ohApi.ts` helpers); ownership is D5. **Neither side claiming it is the worst outcome**, because a wrong specifier produces zero diagnostics at build time.

### P5 — Nine inputs agreed with the build system

`moduleType` (`entry` / `feature` / `har` / `shared`), `byteCodeHar`, `useTsHar`, `singleFileEmit`, `emitDeclarations`, `declarationOutDir`, `harOutDir`, `bundledDeclare`, `declarationEntryFiles`. Without all nine the layer cannot compute destinations. Conflicts with the standard `outDir` / `declarationDir` shape are described in §1.6 of the handoff; who writes to disk is D4.

### P6 — The `oh-exports` data channel

The parsing logic exists on the tsgo side, but `internal/module/resolver.go` hardcodes nil, so the channel is not connected. Two consumers wait on the same slot: F8 needs it as the root file set, module resolution needs it to report "this file is not in the package's export list". Ownership is D6.

### P7 — The 12 reference baselines wired into a runner

Nothing runs them today: OH's own mocha never does (the entry `src/testRunner/unittests/tsc/etsTests.ts` is not imported by `tests.ts` and shells out to `lib/tsc.js`), and none of the 12 case names appear in `testdata/` or `internal/`. Their configuration also lists an external SDK file `interface/sdk-js/api/@internal/component/ets/index-full.d.ts`, which must be handled or trimmed into the fixture.

### P8 — Struct declaration emit

Out of scope here (ArkUI stream). It blocks 10 of the 12 baselines of acceptance criterion 1: only `functionWithDecorators.d.ets` and `functionAndClassWithDecorator.d.ets` are struct-free; the other 10 mention `struct` from 1 to 17 times. Criterion 1 must be split ("2 verified immediately, 10 together with struct") or the struct-independent differences extracted into this requirement's own cases. Warning to whoever takes struct: tsgo does not synthesize a virtual constructor when an explicit one exists, so the reference's "visit from index 1" must be replaced by a filter on `ast.NodeIsVirtual` — deleting the user's own constructor is the failure mode.

---

## 3. Features and requirements

### F1 — Annotation declarations and annotation properties in declaration files

Handoff §2.1 and §1.3; change points 1–7 of the §1.8 table.

**Where in tsgo.** `internal/transformers/declarations/transform.go` (dispatch table, `ensureType`), `internal/transformers/declarations/util.go` (`isDeclarationAndNotVisible`, `isEnclosingDeclaration`, `canProduceDiagnostics`); the ArkTS logic itself in a new `arkts.go` in that package. The four declaration files currently have zero ArkTS hits.

**Reference.** `transformers/declarations.ts`: `transformTopLevelDeclaration` (`AnnotationDeclaration` branch), `visitDeclarationSubtree` (`AnnotationPropertyDeclaration` branch), `HasInferredType`, `ensureType`, `isPreservedDeclarationStatement`, `ProcessedComponent` / `isProcessedComponent`, `isEnclosingDeclaration`, `isDeclarationAndNotVisible`.

**Requirements**

- **F1-1** With the ETS options on, an exported `@interface` declaration and its properties appear in the emitted `.d.ets`; a non-exported one does not.
- **F1-2** A property default value is carried into `.d.ets` verbatim: parentheses, operators and identifier references keep their source form (`a: number = +1`, `d: boolean = !1`, `c: Array<Array<number>> = [[1 + a * b / 3], [E.A + b], [a + E.A], []]`).
- **F1-3** A property with no explicit type gets its type inferred from the default value and written out.
- **F1-4** Emit proceeds and the default value is still written out when the default is not a constant expression; TS28034 is reported and does not block emit. No guard of our own is added on this path.
- **F1-5** An annotation declaration inside an ambient namespace is emitted the same way.
- **F1-6** The top-level statement allowlist and the member allowlist admit both node kinds and change together — adding one without the other only trades a printer panic for a transform-phase panic.
- **F1-7** Declaration emit serializes the original AST node; evaluation results (`getAnnotationPropertyEvaluatedInitializer`, `getAnnotationPropertyInferredType` on the `EmitResolver`) stay on the script-output side and are not consulted.
- **F1-8** With the ArkTS options absent, behaviour and output in `.ts` are unchanged.
- **F1-9** TS28033 and TS28034 are reported where the reference reports them (checker side, shared with P1).

**Done when** the 9 reference outputs of acceptance criterion 2 match byte for byte, and rows 1–7 of the §1.8 table are covered by the merge request's call-site list.

### F2 — Decorator retention per the `ets.emitDecorators` allowlist

Handoff §2.2 and §1.4; change points 8–10 of the §1.8 table.

**Where in tsgo.** `ensureModifiers` in `internal/transformers/declarations/transform.go` unconditionally does `core.Filter(mods.Nodes, ast.IsModifier)` today; the replacement logic goes into a new `arkts.go` in the same package. `EtsOptions.EmitDecorators` (`internal/core/arkts.go`) and the parsing (`ParseEtsOptions`, `parseEtsEmitDecoratorsOptions` in `internal/tsoptions/arkts.go`) already exist but have no consumer outside their own unit tests.

**Reference.** `getEffectiveDecorators` in `ohApi.ts`; entry points `ensureEtsDecorators` / `getReservedDecoratorsOfEtsFile` / `getReservedDecoratorsOfStructDeclaration`; `inEtsStylesContext` in `ohApi.ts`, consumed in `visitDeclarationSubtree` and `transformTopLevelDeclaration` in `declarations.ts`.

**Requirements**

- **F2-1** Only decorator names listed in `ets.emitDecorators` survive into `.d.ets`; a decorator not on the list is dropped; when the option is not configured everything is dropped.
- **F2-2** For an entry configured `emitParameters: false`, a decorator with arguments is reduced to the bare name (`@Consume("1")` → `@Consume`); a decorator without arguments stays as-is.
- **F2-3** Functions and methods decorated with `@Styles` carry no type parameters in declaration files.
- **F2-4** The engine works on ordinary classes and functions written in an `.ets` file — it needs no annotations, no structs and no binder changes, and it is verified on that basis.
- **F2-5** The engine is shared with the ArkUI stream: it is owned here, the semantics of UI decorators are not.
- **F2-6** With `ets.emitDecorators` absent, output in `.ts` is unchanged from today.

**Done when** the reference baselines that are struct-free (`functionWithDecorators.d.ets`, `functionAndClassWithDecorator.d.ets`) match byte for byte, and the reproduced case `ets.emitDecorators = [Observed, State, Styles]` on `@Observed export class Model { @State value: string = "a" }` keeps both decorators.

### F3 — `@Sendable` kept as-is in declaration files

Handoff §2.3; change point 11 of the §1.8 table.

**Where in tsgo.** Same files as F2.

**Reference.** `isSendableFunctionOrType` in `ohApi.ts`, consumed in `transformTopLevelDeclaration` in `declarations.ts` (TypeAlias and FunctionDeclaration branches): the decorators are assigned straight through without allowlist filtering.

**Requirements**

- **F3-1** A `@Sendable`-decorated type alias and a `@Sendable`-decorated function declaration carry their decorator into `.d.ets` verbatim, bypassing allowlist filtering.
- **F3-2** The bypass is taken only when the node is a function declaration or a type alias, the file is `.ets`, `illegalDecorators` holds exactly one entry, and that entry is exactly `Sendable`. Any other combination falls back to allowlist filtering.

### F4 — Declaration visibility trimming and visibility checks for annotation references

Handoff §2.4; change points 7, 12–14 of the §1.8 table.

**Where in tsgo.** `internal/transformers/declarations/transform.go` (primitives exist), `internal/transformers/declarations/util.go` (`isDeclarationAndNotVisible`, `isEnclosingDeclaration`), `internal/transformers/declarations/diagnostics.ts` counterpart (`canProduceDiagnostics` and friends) — ArkTS wiring in a new `arkts.go`.

**Reference.** `checkAnnotationVisibilityByDecorator` in `declarations.ts` (18 call sites); `getAnnotations` / `getAnnotationsFromIllegalDecorators` in `utilitiesPublic.ts` (20 call sites in `declarations.ts`, of which 2 are in the struct branch); `DeclarationDiagnosticProducing`, `canProduceDiagnostics`, `createGetSymbolAccessibilityDiagnosticForNode` in `transformers/declarations/diagnostics.ts`.

**Requirements**

- **F4-1** Unexported annotation declarations do not appear in `.d.ets`.
- **F4-2** Every annotation reference in `.d.ets` — both the callee of `@Anno({...})` and the initializers of object-literal members — gets the same entity-name visibility check as a type reference; an unexported name produces a visibility diagnostic.
- **F4-3** Applied annotations are picked out and re-attached on every declaration emit branch (20 call sites in the reference; the 2 in the struct branch are out of scope with P8).
- **F4-4** Annotation properties participate in the anchor computation for declaration diagnostics.
- **F4-5** With the ArkTS options absent, `.ts` behaviour is unchanged.

**Done when** the four scenarios of §2.4 hold against the reference, and no ArkTS-off regression appears in the non-ETS gate of §7.

### F5 — `export` stripping inside ambient namespaces, keeping decorators and annotations

Handoff §2.5; change point 15 of the §1.8 table.

**Where in tsgo.** `internal/transformers/declarations/transform.go` holds only the variant that does not keep decorators; the ETS variant goes into a new `arkts.go`.

**Reference.** `stripExportModifiersForEts` in `declarations.ts` — `ensureEtsDecorators + getAnnotations` for `ClassDeclaration`, `getAnnotationsFromIllegalDecorators` for `VariableStatement`; called from `transformTopLevelDeclaration`.

**Requirements**

- **F5-1** Inside an ambient namespace, classes and functions come out of `.d.ets` with `export` removed and their decorators and annotations kept.
- **F5-2** Variable statements keep their annotations.
- **F5-3** `export import` and `export default` are not stripped (explicit exemption in the reference).
- **F5-4** The result matches the `functionAndClassWithDecorator.d.ets` shape — including `@Styles function globalFancy();` with no return type.

### F6 — Three output-shape fixes for declaration files

Handoff §2.6; change points 16–18 of the §1.8 table.

**Where in tsgo.** Triple-slash references: `mapReferencesIntoArray` counterpart in `internal/transformers/declarations/transform.go`. Portable specifiers and `Resource`: `symbolToTypeNode` in `internal/checker/nodebuilderimpl.go` (which hardcodes `/node_modules/` in two places and has no ohpm branch) — ArkTS branch in a new `internal/checker/arkts.go`; the underlying generator in `internal/modulespecifiers`, which per §7 takes no `arkts.go` and accepts table entries and case arms only.

**Reference.** `mapReferencesIntoArray` in `declarations.ts` with `isOhpm` / `isOHModulesReference` in `ohApi.ts`; `symbolToTypeNode` in `checker.ts` with `isOhpmAndOhModules` in `ohApi.ts`; the `Resource` special case from commit `a0f18856d2`.

**Requirements**

- **F6-1** Triple-slash references in `.d.ts` contain no `oh_modules` paths.
- **F6-2** A type reference crossing into another ohpm package is written as a bare package name — never as a relative or absolute path diving into `oh_modules`.
- **F6-3** Specifiers stay portable when the referenced file comes in through a symlink.
- **F6-4** The SDK `Resource` type is emitted as a bare type reference instead of `import("…").Resource` when the module specifier ends with `/ets/api/global/resource` or `/ets/dynamic/api/global/resource` and the last item of the symbol chain is `Resource`.
- **F6-5** F6-2 and F6-3 are verified in output comparison, not only by unit tests: a wrong specifier reports no diagnostic at build time, only in the consumer project.

### F7 — Output set and publishing path by module form

Handoff §2.7, §1.5, §1.6.

**Where in tsgo.** A new build-facing layer; tsgo has none today, and where it hooks in is D4. It is not built out of `outDir` / `declarationDir`.

**Reference.** The form branch in `generateModuleAbc` in `generate_module_abc.ts`; path computation in `generateSourceFilesInHar` and `genTemporaryPath` in `utils.ts`.

**Requirements**

- **F7-1** Output set by module form: source HAR → source outputs (`.js`, or `.ts` when `useTsHar` is on) + `.d.ets` / `.d.ts` + sourcemap, no abc; bytecode HAR → `modules.abc` + declarations + sourcemap; shared HSP → same as bytecode HAR; HAP → abc + sourcemap and no declarations.
- **F7-2** Relative base and output root by form: shared HSP → project root / `etsFortgz`; bytecode HAR → module root / `etsFortgz`; source HAR → module root / cache directory.
- **F7-3** When `buildInHar` is in effect, the module-name prefix is erased from the relative path and no `TEMPORARY` level is inserted.
- **F7-4** Files under the package directory (`oh_modules`) are excluded wholesale by default and included when `bundledDeclare` is on.
- **F7-5** The `compileHar && !byteCodeHar` shape, where two bases (cache and build) are live in the same run and files are copied pair by pair, is either reproduced or explicitly assigned to the build system with the rules written down on that side (D4).
- **F7-6** Where `bundledDeclare` is on and declaration bundling is not implemented, the build stops with a diagnostic instead of silently producing per-file declarations.

### F8 — Collecting and recording declaration outputs; the two write entry points

Handoff §2.8, §1.6.

**Where in tsgo.** The same new layer (D4), plus the `oh-exports` channel in `internal/module/resolver.go` (P6).

**Reference.** `processHarSourceFiles` in `ets_checker.ts` (collection), `harFilesRecord` / `GeneratedFileInHar` in `utils.ts` (record table), `writeDeclarationFiles` in `ark_utils.ts` (called from `beforeBuildEnd` in `rollup-plugin-gen-abc.ts`) and `mangleDeclarationFileName` in `ark_utils.ts` (called from `handlePostObfuscationTasks` in `ob_config_resolver.ts`).

**Requirements**

- **F8-1** The traversal set is "resolved modules ∪ root files"; the root file set comes from `oh-exports`, falling back to the manifest `main` with its extension changed to `.d.<original extension>`.
- **F8-2** A file that already is `.d.ets` / `.d.ts` is read whole and copied as-is.
- **F8-3** A `.ets` / `.ts` is emitted on its own (`emitOnlyDtsFiles`) and written as `.d` + the original extension.
- **F8-4** Declarations are produced even when there are errors (`forceDtsEmit`), and annotation back-references are filled in without running type checking (`skipDiagnostics`). Both arguments are load-bearing; neither is dropped in a rewrite.
- **F8-5** A failed single-file emit is skipped silently and does not interrupt the run (reference behaviour, copied as-is).
- **F8-6** Both write entry points exist and are selected by `singleFileEmit`. The declaration path is not cut together with "the obfuscation stuff": `mangleDeclarationFileName` must run with obfuscation off, and it is one of the two entry points.
- **F8-7** An output record table exists with the reference's six fields and a defined lifecycle: when entries are registered, which entry point writes them, how they are invalidated under incremental builds. It is a seam, not an internal detail — declaration bundling writes merge results back into it and obfuscation locates outputs through it.
- **F8-8** With `bundledDeclare` on, registration means immediate write.
- **F8-9** Two filters are reproduced: a path containing the package directory name (`oh_modules`) is skipped, and `.ets` / `.ts` under `/oh_modules/` are skipped; both stop applying when `bundledDeclare` is on.
- **F8-10** Exceptions raised inside the per-file loop are swallowed without logging (reference behaviour; do not turn it into an error).

### F9 — Source outputs: extension by `useTsHar`, destination in the HAR

Handoff §2.9, §1.5.

**Where in tsgo.** The same new layer (D4). How the source text is produced is not part of this feature — only the extension and the destination.

**Reference.** `setIncrementalFileInHar` in `rollup-plugin-ets-typescript.ts` (extension and the cache→build mapping); `writeObfuscatedSourceCode` in `ark_utils.ts` (registration into `sourceCachePath` in the live rollup pipeline).

**Requirements**

- **F9-1** With `useTsHar` off, a source HAR emits `.js`; with it on, `.ts`.
- **F9-2** In both cases the destination is registered into `sourceCachePath` of the record table.
- **F9-3** A change in `useTsHar` invalidates the whole cache on a source HAR pipeline — the extension is read back by the bundle side, so a wrong value breaks more than the output itself.

### F10 — Declaration diagnostics for library builds

Handoff §2.10.

**Where in tsgo.** The build-facing gate (D4).

**Reference.** `printDeclarationDiagnostics` in `ets_checker.ts`, called from `processBuildHap` on the `compileHar || compileShared` branch.

**Requirements**

- **F10-1** Building a HAP reports no declaration diagnostics.
- **F10-2** Building a HAR / HSP computes and reports them, skipping `.d.ets` / `.d.ts`, `.js`, and files under `oh_modules`.
- **F10-3** Only the gate — when to compute and for which files — belongs here; which diagnostic is dropped or downgraded belongs to the diagnostics-alignment stream.

---

## 4. Open decisions that change the scope

| ID | Decision | Effect on this inventory |
|---|---|---|
| D1 | Whether `bundledDeclare` is on in the baseline project | Declaration bundling (about 2932 lines, 117 fixtures) enters or stays out. Until the value is measured, F7-6 keeps the fail-loud guard. The decisive value is `arkOptions.bundle.bundledDeclare` in `build-profile.json5` |
| D2 | The producer form of the 8 HARs | Which output sets of §1.5 and which of the three paths of §1.6 are exercised on the baseline; not readable backwards from `byteCodeHarInfo` (consumer-side). Measure `loader.json`, `filesInfo.txt` presence, and where declarations land |
| D3 | The values of `useTsHar` and `singleFileEmit` on the baseline | Which branch of F9 and which write entry point of F8 the baseline takes. Both branches are implemented either way; only scheduling depends on the answer |
| D4 | Who writes declaration outputs to disk | Until settled, F7, F8 and F9 have no inputs: either tsgo writes to the final directories (the nine inputs of §1.6 assume this) or tsgo writes to cache and the build system copies into `loader_out` |
| D5 | Who owns the specifier / `oh_modules` machinery | P4 and the portable-specifier part of F6. Best placed with whoever brings `oh_modules` module resolution into tsgo |
| D6 | Who takes the `oh-exports` slot | P6 and F8-1; module resolution has a second use for the same slot |
| D7 | Whether the `@Retention` / `SourceRetention` family is taken | `isRetentionAnnotationDeclaration`, `checkSourceRetentionAnnotation`, `hasSourceRetentionPolicy` in `checker.ts`; half lands in declaration emit, half in SDK API availability checking |
| D8 | Whether the two checker prerequisites count toward this requirement | P1: if not, F1 waits on a separate piece of work with its own schedule |
| D9 | Who builds the golden mechanism and when | P3; without it every feature below is unverifiable and review stalls |

---

## 5. Out of scope — do not add these to the plan

- **Struct declaration emit** — ArkUI stream (`StructDeclaration` branch of `transformTopLevelDeclaration`). Affects acceptance through P8.
- **Annotation erasure and evaluation in `.js` / abc** — the script-output side. The dividing line: anything going through `getAnnotationPropertyEvaluatedInitializer` / `getAnnotationPropertyInferredType` on the `EmitResolver`, including `transformers/ts.ts`, `classFields.ts`, `legacyDecorators.ts`.
- **Declaration bundling (declmerge)** — until D1 says otherwise.
- **Two obfuscation items** — the declaration file name obfuscation interface (`tryMangleFileName`) and the consumer obfuscation configuration on HAR publication. Obfuscation is off within acceptance scope. `mangleDeclarationFileName` is not one of them: it must run with obfuscation off.
- **Resolution-time visibility decisions for `oh-exports`** — the `oh_modules` stream.
- **HAR packaging, signing and installation after outputs are written** — hvigor.

---

## 6. Acceptance mapping

| Handoff criterion | Covered by | Needs |
|---|---|---|
| 1 — 12 declaration baselines byte for byte | F2, F3, F5, F6; F1 and F4 for their parts | P2, P3, P7; 10 of 12 additionally P8 |
| 2 — 9 annotation outputs byte for byte | F1 | P1, P2, P3 |
| 3 — output set and layout per module form; consumer compiles with no resolution diagnostics | F7, F8, F9, F10 | P5, D4 |

One knock-on effect belongs in the schedule: once `ensureModifiers` starts keeping decorators, baselines involving `.ets` under `testdata/baselines/reference/` drift in bulk, and per repository convention the drift is explained category by category before it is accepted — not accepted with a bare `baseline-accept`.

---

## 7. Cross-cutting requirements (repository rules)

These come from `AGENTS.md`, not from the handoff, and apply to every feature above.

- **ArkTS behaviour is off by default.** With the ArkTS options absent, observable behaviour matches upstream. Where that is impossible, the merge request description says "non-ETS path also affected" and points at the corresponding un-gated location in the reference.
- **All new ArkTS logic lives in `arkts*.go`** in the package it extends — including a new `internal/transformers/declarations/arkts.go` and `internal/checker/arkts.go`. An upstream file gets a call-out only: one line, one `case`, one early return, with the decision itself in `arkts*.go`.
- **`internal/tspath` and `internal/modulespecifiers` take no `arkts.go`** — the registered exception. Changes there are table entries, `case` arms, and early returns returning constants, and every added entry needs a same-package guard test.
- **No signature, parameter or return-type change** to an existing exported function or type.
- **The non-ETS regression gate** applies to any change touching `internal/tspath/extension.go`, `internal/module/resolver.go`, `internal/module/util.go`, `internal/modulespecifiers/util.go`, or emit/transformers: run the tests, accept baselines, and judge every non-extension-name diff — a non-empty result means upstream semantics were changed and must be restored first.
- **Porting source is the OpenHarmony reference**, never `_submodules/TypeScript`, and the reference pin is not advanced by feature work.
- **Test obligations:** every feature is covered by `TestArktsCmpAst` / `TestArktsCmpSymTables` or the reference suites; a Go unit test only for what the AST comparison cannot observe; and every behaviour that differs from plain TypeScript gets a negative `.ts` file under `testdata/arkts/` proving the plain-TypeScript behaviour is unchanged.
- **Coverage pipeline** runs after any change to `testdata/arkts/`, `docs/coverage_matrix/`, or `_scripts/coverage_matrix/`.

---

## 8. Existing work in the OH fork

Source: `D:\Work\tsgo\typescript-go_OH`, remote `origin` = `k1ngqaquuu/typescript-go_OH`. Checked across branches, including branches whose pull request is closed or superseded. A reference below is evidence that an implementation exists **at that ref** — not that it is merged into `main` or complete. Branch and pull-request state moves; re-check the ref before planning.

A "not found" row is absence of evidence, not proof of absence: those rows come from searches over file names, symbols and commit messages, so a fuller read of a branch, or the 40-feature inventory in `docs/plans/047-arkts-path-resolution-features.md`, may still list the item as done.

| Item | Ref | What is there | What is missing |
|---|---|---|---|
| F1, F2, F3, F4 | `arkts/39-annotation-parity`, PR #53 (open, → `main`; supersedes the closed PR #52) | `internal/transformers/declarations/arkts.go`: annotation and `ets.emitDecorators` decorators re-attached ahead of the computed modifiers, the `@Sendable` bypass (`arktsIsSendableFunctionOrType`, `arktsSendableDecorators`), the `@Styles` type-parameter seat (the reference's `inEtsStylesContext`), and visibility validation following `checkAnnotationVisibilityByDecorator`; tests `arkts_annotation_shape_test.go`, `arkts_annotation_visibility_test.go`, `TestArkTSDeclarationEmitMatchesReferenceBaselines` | the pull request's own open list: expando functions bypass `ensureModifiers`, so `@Sendable`, annotations and `ets.emitDecorators` are dropped from `.d.ets` (`declarations/transform.go:2487`). "Not ported" in its difference table: the synthetic annotation call signature (TS2322 ×2 missing), TS28041 (needs host configuration), `isArkguardInputSourceFile`. And one **open** item: `.ets` falls back to ES decorators when `experimentalDecorators` is absent — confirm what ace_ets2bundle passes before relying on decorator output |
| F5 | same branch, partly | the `@Styles` seat, and `TestArkTSNestedAnnotationDeclarationLosesItsExportKeyword` shows the ambient-namespace export case for annotations | a **recorded defect**, carried in the pull request's own open list: `stripExportModifiers` re-attaches annotations with no `isInEtsFile` gate and no per-kind restriction, so ArkTS logic reaches `.ts` (`declarations/transform.go:1576`). That contradicts the off-by-default rule in §7 and is a bug to fix, not a gap to close |
| P1 | same branch | `internal/checker/arkts.go`, `arkts_annotation_args.go`, `arkts_annotation_info.go`; `internal/binder/arkts.go` `bindAnnotationDeclaration` | none of it is on `main` or on the week-work branch: there `isArkTSClassLike` still names only Struct and `getDeclarationSpaces` still panics on its last line |
| P8 (struct `.d.ets`) | same branch | `arkts_struct_test.go`, including `TestArkTSStructDeclarationEmit` — the "10 of 12 baselines" blocker is addressed there | — |
| F6 | Triple-slash exclusion: `arkts/39-annotation-parity` (PR #53). Portable specifier: `arkts/47-path-resolution-parity`, PR #64 (open, → `arkts/tsgo-orchestration-week-work`), on its **remote** tip `c66e35a018` — the local branch lags one commit behind | triple-slash: `arktsIsOmittedOHModulesReference` in `internal/transformers/declarations/arkts.go:466`, wired at `transform.go:453`. Portable specifier: the guard in `symbolToTypeNode` now reads `strings.Contains(specifier, "/node_modules/") \|\| core.IsOhpmAndOhModules(options.PackageManagerType, specifier)` (`nodebuilderimpl.go:685`), with the helper in `internal/core/arkts.go:201-205` citing `ohApi.ts:244-246`. Plus the ohpm overlay (store `oh_modules`, manifest `oh-package.json5`, path part `/oh_modules/`), `internal/modulespecifiers/arkts.go`, `internal/outputpaths/arkts.go` | only the bare-`Resource` case is still absent: `ets/api/global/resource` has no occurrence in any branch or commit of the repository. Do not derive resolution requirements here: the area has its own inventory — 40 features with a standard-versus-ArkTS classification and a tsgo status each — in `docs/plans/047-arkts-path-resolution-features.md` and `047-arkts-path-resolution-plan.md` on branch `arkts/47-path-resolution-tests` |
| P4 | same PR #64 | the reference's four pieces exist in Go form (`getModuleByPMType`, `getModulePathPart`, the manifest filename, the store folder) | — |
| F7, F8 | PR #56 (merged into `arkts/tsgo-orchestration-week-work`), plan `docs/plans/035-tsgo-monorepo-v1-contract.md`; `arkts/35-tsgo-profile-metrics` adds per-package phases, the worker timeline and the standard compiler options to the result contract | one hvigor config JSON in, one result JSON out; a tsgo program per emitting package; outputs under `<outputDir>/<module path from projectRoot>` with `rootDir = projectRoot` (the loader_out layout); `.d.ets` when `generateDeclarations`; declaration-reference targets, mirrors and waits; package-level incremental state under `projectRoot/.tsgo_monorepo` | the handoff's nine inputs do not exist in this design, and the §1.6 mapping has to be re-derived rather than ported: `singleFileEmit`, `harOutDir`, `declarationOutDir` and `bundledDeclare` have **no occurrence anywhere in that repository's history**. es2abc, abc layout, manifests, `byteCodeHar`, obfuscation and ArkUI lowering for emit are declared out of scope on that branch |
| F9 | PR #56 + PR #63 (merged; plan `docs/plans/043-add-ts-files-emitting.md`) | `harmonyOptions.compileMode == "esmodule"` writes TypeScript verbatim (annotations kept, no JS lowering) as `.ts` + `.map`, otherwise JS — the `useTsHar` decision, reached through the hvigor contract instead of the option | — |
| F10 | not found | — | no declaration-diagnostics gate in the driver |
| P6 | open | `ResolvedModule.IsNotOhExport` exists (PR #21, merged); `internal/module/resolver.go` records in a comment that the `oh-exports` map source (hvigor's `depName2OhExports`) is not wired yet | the data channel itself |
| P2 | partial | `internal/tsoptions/declscompiler.go` exists and PR #53 touches it (+19 lines) | not verified in this pass whether the `tests/dets` configuration is complete there |
| P3 | mostly met | the emit-text harness `testdata/ts-emit/` with its expected-failures files (PR #63); `TestArkTSDeclarationEmitMatchesReferenceBaselines` on PR #53; and an emit-diff gate `_scripts/coverage_matrix/emit_diff.py` that compares reference tsc against tsgo emit, with the allowlist `testdata/arkts/expected-failures-emit.txt` | the gate covers script and TS emit; comparing **declaration** output is the part that is still new |
| P7 | partial | the 12 `dets/cases` are picked up by the AST and symbol-table comparison (`internal/testrunner/arkts_cmp_ast_test.go:64`, `arkts_cmp_sym_tables_test.go:41`), and a byte-for-byte `.d.ets` comparison exists — `TestArkTSDeclarationEmitMatchesReferenceBaselines`, which reads the `.d.ets` block out of the reference's own baselines | that byte comparison runs over the `conformance/annotations` suite, not over the 12 `dets` baselines; pointing it at them is the remaining work |

**Why the handoff's nine inputs never appear in the driver.** The mode refuses them by name, in both layers (the config's `globalCompilerOptions` and the command line): `outDir`, `rootDir`, `declarationDir`, `declaration`, `noEmit`, `emitDeclarationOnly`, `incremental`, `tsBuildInfoFile` and `composite` are all refused, because the per-package contract — `outputMode`, `generateDeclarations`, the `incremental` section — owns those decisions and the result document replaces the statistics table. So §1.6 of the handoff cannot be ported as it stands: the same decisions are made there, but by the config, and the port has to restate them in the driver's vocabulary.

**One prerequisite the handoff names only in passing is a hard blocker there.** `struct`, `@interface` and a decorator on a function declaration panic the shared emit/check pipeline: `internal/transformers/estransforms/classfields.go` descends into `KindStructDeclaration` where its sibling has the ArkTS passthrough guard, `printer.go` has no arm for `KindStructDeclaration`, `KindAnnotationDeclaration` or `AnnotationPropertyDeclaration`, and the checker's `getDiagnosticHeadMessageForDecoratorResolution` carries neither the `FunctionDeclaration` nor the `StructDeclaration` case the reference has. They were recorded as **one unit of work** — a guard added at the first site alone only relocates the panic to the next — and the ring pull requests (#66, #67) target exactly that path. Declaration emit cannot be accepted over a package whose emit panics, so this belongs beside P1 in the schedule.

**Relationship to this repository.** `third_party_typescript_go` (the SIG repository, where the work is ported issue by issue) has none of the above: `internal/transformers/declarations/` has zero ArkTS hits, `isArkTSClassLike` in `internal/ast/arkts.go` covers only Struct, `getDeclarationSpaces` in `internal/checker/checker.go` still panics on its last line, and `ets.emitDecorators` still has no consumer. So F1–F5 and P1, P8 are a port of an existing Go implementation rather than a port from the reference, and F7–F9 are a port of a design that has already been agreed with hvigor — which changes D4 from an open question into a decision that is already made in the OH fork ("tsgo writes directly into the directories the config names") and only needs to be ratified here.

**Adjacent work in flight there** (script-output side and packaging, out of scope for this inventory but they share the same files): PR #65 (hvigor ArkTS build-configuration contract), PR #66 (ArkTS transform ring and keep-TS emit path, ring seats left empty on purpose), PR #67 (port of the complete transform functionality from OH TSC, excluding ArkUI), PR #62, #60 and #59 (module identity, `ohmUrl` record names, es2abc input manifests), PR #58 (`ets_loader_plan` skeleton), PR #45 (ArkTS linter).

---

## 9. After the planned merges: what actually remains

Assumed baseline: this repository receives everything from OH `main`, then `arkts/tsgo-orchestration-week-work`, `arkts/47-*` and `arkts/35-tsgo-profile-metrics`, and then the changes of the OH pull requests that are still open (#53, #64, #65, #66, #67, #62, #60, #59, #58, #49, #45). Under that baseline every row of §8 moves from "exists over there" to "exists here", and the backlog is what is below.

| Item | What remains after the merges | Grounds |
|---|---|---|
| F1, F2, F3, F4 | Nothing to port, but three carried items arrive with it: expando functions bypass `ensureModifiers`, so `@Sendable`, annotations and `ets.emitDecorators` are dropped from `.d.ets` (`declarations/transform.go:2487`); `.ets` falls back to ES decorators when `experimentalDecorators` is absent, which the pull request itself marks as its most serious open item; and three branches stay "not ported" — the synthetic annotation call signature (TS2322 ×2 missing), TS28041, `isArkguardInputSourceFile`. `TestArkTSDeclarationEmitMatchesReferenceBaselines` is the entry point for verification, and F1-1…F4-5 stay the acceptance criteria | `internal/transformers/declarations/arkts.go` and its four test files; the pull request's own open and "not ported" lists |
| F5 | Fix the defect that arrives with the merge: `stripExportModifiers` re-attaches annotations with no `isInEtsFile` gate and no per-kind restriction, so ArkTS logic reaches `.ts` (`declarations/transform.go:1576`) — that breaks the off-by-default rule of §7. Then verify the full shape: classes, functions and variable statements keep decorators and annotations, `export import` and `export default` stay untouched | the pull request's own open list; `TestArkTSNestedAnnotationDeclarationLosesItsExportKeyword` covers only the annotation case |
| F6 | Two of the three shapes arrive with the merge: the triple-slash `oh_modules` exclusion (PR #53, `arktsIsOmittedOHModulesReference`) and the ohpm-aware guard in `symbolToTypeNode` (remote tip `c66e35a018` of #47, `core.IsOhpmAndOhModules`). **Only one item is left:** emit a bare `Resource` instead of `import("…").Resource` when the specifier ends with `/ets/api/global/resource` or `/ets/dynamic/api/global/resource` and the last item of the symbol chain is `Resource` | `ets/api/global/resource` has no occurrence in any branch or commit of the OH repository |
| F7, F8 | Nothing to port from the handoff: the design was replaced by the hvigor config contract, and the nine inputs are refused by name there. The driver's own acceptance vehicle exists and is green — the four-package demo project `D:\Work\tsgo\test_projects\tsgo-project` (cold, warm, propagation). What remains is acceptance over the real corpora, which the driver's plan records as "not met on this branch, and not claimed" — the corpus sources stop at constructs the emit pipeline does not lower yet — and the resolution inventory adds that the acceptance baseline project `sample_in_harmonyos` has never had its dependencies installed (no `oh_modules`), so its scenarios are not reproducible out of the box | `docs/plans/035-tsgo-monorepo-v1-contract.md` §2.5, §3, §4; `docs/plans/047-arkts-path-resolution-features.md` §7 |
| F9 | Nothing to port (merged PR #56 + #63) | `compileMode: "esmodule"` in the driver |
| F10 | **Open.** No declaration-diagnostics gate exists in any branch or pull request of the OH fork | searches for `printDeclarationDiagnostics` / `DeclarationDiagnostics` over all refs return only upstream commits |
| P1 | Covered (checker and binder annotation awareness on the annotation-parity branch), plus the emit-panic unit — `struct`, `@interface`, a decorator on a function declaration — which the ring pull requests #66 and #67 target. Treat that unit as part of the same schedule slot: declaration emit cannot be accepted over a package whose emit panics | `internal/checker/arkts*.go`, `internal/binder/arkts.go`; the panics are recorded in `classfields.go`, `printer.go`, `getDiagnosticHeadMessageForDecoratorResolution` |
| P2 | Verify whether the `tests/dets` compiler configuration is complete in the merged tree. The option table the declaration runs read is the fork file `internal/tsoptions/declscompiler.go`; PR #53 adds `packageManagerType` and `etsEmitIntermediate` to it | the `declscompiler.go` diff on `arkts/39-annotation-parity` |
| P3 | Mostly met by the emit-diff gate. The new part is comparing **declaration** output, which that gate does not cover | `_scripts/coverage_matrix/emit_diff.py`, allowlist `testdata/arkts/expected-failures-emit.txt` |
| P4 | Nothing to port (lands with `arkts/47-*`) | `internal/modulespecifiers/arkts.go`, the ohpm overlay in `internal/core/arkts.go` |
| P6 | **Open.** The `oh-exports` map source (hvigor's `depName2OhExports`) is not wired; `ResolvedModule.IsNotOhExport` already exists as the consumer-side flag | comment in `internal/module/resolver.go` |
| P7 | **Partly there.** The 12 `dets/cases` are already parsed and compared by AST and symbol tables in the OH fork, and a byte-for-byte `.d.ets` comparison exists (`TestArkTSDeclarationEmitMatchesReferenceBaselines`, which reads the `.d.ets` block out of the reference's own baselines). What remains: aim that byte comparison at the 12, and handle the external SDK file their configuration names | `arkts_cmp_ast_test.go:64`, `arkts_cmp_sym_tables_test.go:41`, `declarations/arkts_test.go` on PR #53 |
| P8 | Nothing to port (struct declaration emit is on the annotation-parity branch) | `arkts_struct_test.go` |
| D1 | Partly answered: declaration bundling is still not ported, and the ring carries the explicit trace `Not ported: findLocalDependencyFromBundledDeclare (utils.ts:1404-1407), the projectArkOption.bundle.bundledDeclare fallback… no tsgo counterpart yet`. Whether the baseline project has the switch on is still unmeasured | that "not ported" comment, which also satisfies one of the three traces the repository rules require |
| D2 | Still unmeasured — which form the 8 HARs are built in. The corpus runs are where it will surface | §4 of the handoff |
| D3 | Obsolete as stated, and the replacement is **not the same switch** — see the note below the table. `singleFileEmit` has no counterpart at all; the keep-`.ts` decision is carried by `harmonyOptions.compileMode == "esmodule"`, one value for the whole build, not by a per-module `useTsHar` | `harOutDir`, `singleFileEmit`, `declarationOutDir` have no occurrence in the OH history; `internal/execute/monorepo/compile.go:1251-1260,1331` |
| D4 | Decided by the merged work: tsgo writes into the directories the config names, and removal is hvigor's `clean` | `docs/plans/035-tsgo-monorepo-v1-contract.md` §2.5 |
| D5 | Decided: the specifier and package-directory machinery is in tsgo | `arkts/47-*` |
| D6 | Open, and it is P6 | comment in `internal/module/resolver.go` |
| D7 | Covered on the annotation-parity branch: the retention policy is read from the SDK's own `@arkts.lang.d.ets` and the `source` value gates the erasure | `internal/checker/arkts_annotations.go` |
| D8 | Obsolete: the two checker prerequisites are implemented | annotation-parity branch |
| D9 | Mostly resolved for script and TS emit; declaration comparison is the remainder, and it is P3 | `_scripts/coverage_matrix/emit_diff.py` |

**`useTsHar` against `compileMode` — related, but not the same switch.** `useTsHar` is ets-loader's per-module switch for source HARs: with it on, the source outputs stay `.ts` instead of being turned into `.js` (the reference decides the extension in `setIncrementalFileInHar`, `useTsHar ? '.ts' : '.js'`), and the module's *packaging form* is what makes the question meaningful. `compileMode` is hvigor's ArkTS compilation mode for the build — `esmodule` or `jsbundle` — and in the merged contract it is **one value for the whole run** (in `tsgo-monorepo-config.json` it sits in the root `harmonyOptions`; the per-package `harmonyOptions` carries only `moduleType`, `moduleName` and `sourceRoots`). The driver turns `esmodule` into the internal `EmitTs` flag (`internal/execute/monorepo/compile.go:1331`), and that flag makes the emitted `.ts` a verbatim print of the parsed tree — annotations kept, no script transformer runs — with the extension switched by `internal/outputpaths/arkts.go`; an unrecognised value warns and falls back to JavaScript. So the two overlap on the observable — does the output stay `.ts` — but the conditions differ in scope: `useTsHar` is per module and only for a source HAR, while `compileMode` is build-wide, so under the driver's rule an entry HAP with `compileMode: esmodule` also writes `.ts`, whereas the reference's `useTsHar` question never arises for a HAP. The OH TSC side also has a *per-file* `getShouldEmitJs`, which the ring pull request models as `core.EtsEmitMode` (`KeepTS` / `JS`) with the single source of truth in `outputpaths.ArkTSEmitMode`. **That is the measurable form of D3**: for each module form in the demo project, compare the extension the reference actually writes into `loader_out` against what the driver writes.

Two consequences for planning. First, the requirements F1-1…F6-5 and F9-1 are not to be re-derived: they are the acceptance criteria of code that will be sitting in this repository, and the task after the merge is to run them against the reference, not to rewrite them. Second, the genuinely open items are few and named: **F10**; the bare-`Resource` shape (**F6**); the F5 defect and the verification that follows it; the expando and `experimentalDecorators` items carried with F1–F4; **P6**; the last leg of **P7**; **P2**'s completeness check; the declaration half of **P3**; and the measurement decisions **D1**, **D2** and the new form of **D3** — everything else arrives with the merge.
