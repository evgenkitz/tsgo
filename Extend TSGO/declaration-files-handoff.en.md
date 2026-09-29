# [ArkTS][Declaration Files]　ArkTS Declaration File Generation and Normalized Library Output Layout

> Requirement: support `.d.ets` declaration generation (including annotation declarations with their property default values preserved), and write library outputs to disk following the agreed layout.
>
> This document covers the background and the delivery scope of that requirement. Reference implementations are pinned at `ace_ets2bundle` @ `5b89da97cc91695b9623e1d76afb6e2a0859ec3d` and OH TSC (ohos-typescript, vendored here as `_submodules/third_party_typescript`) @ `f2a0cebe8c` (4.9.5-r4); upstream TypeScript used for comparison is `_submodules/TypeScript` @ `c3bd12d88`. If you change versions, every reference has to be re-located.
>
> Throughout, "the reference" and "the existing toolchain" mean the ArkTS compilation chain that ships with the SDK today, i.e. ets-loader (`ace_ets2bundle`) driving OH TSC. This requirement replaces both of them.
>
> All references are given as "file + function", never line numbers — line numbers drift between versions, function names do not. This document is self-contained and does not depend on any other document. Drafted 2026-09-22.

---

## 0. Terminology

These terms come up repeatedly below; here they are in plain words so you don't have to guess while reading.

| Term                             | Plain meaning                                                                                                                                                                                                                   |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Declaration file                 | The file that carries only types and no implementation. `.ts` produces `.d.ts`, `.ets` produces `.d.ets`. Consumers read only this when compiling, never the source                                                             |
| Declaration emit                 | The process by which the compiler cuts a source file down to a declaration file: drop function bodies, drop private members, write inferred types out explicitly                                                                |
| Annotation                       | ArkTS's `@interface Foo { ... }` declaration, plus its use elsewhere as `@Foo(...)`. It is a different mechanism from TS decorators; it only looks similar                                                                      |
| Annotation property              | The fields inside an annotation declaration body, e.g. `a` in `@interface Anno { a: number = 1 }`. They **must** appear in declaration files together with their default values                                                 |
| `illegalDecorators`              | An internal TS field name holding decorators attached to nodes that syntactically should not carry decorators but do. ArkTS annotations and `@Sendable` both land in this field                                                 |
| Decorator retention allowlist    | The `ets.emitDecorators` compiler option. Declaration files drop every decorator by default; only names on this allowlist are kept                                                                                              |
| Ambient namespace                | `declare namespace X { ... }` in a declaration file. Its members are implicitly exported, so `export` modifiers must be stripped                                                                                                |
| HAR / HSP                        | OpenHarmony's two library forms. Once published, the type surface a consumer gets is the set of `.d.ets` files it ships                                                                                                         |
| Source HAR / bytecode HAR        | Two packaging forms of the same HAR. A source HAR carries source plus declarations; a bytecode HAR carries a precompiled `modules.abc` plus declarations                                                                        |
| `useTsHar`                       | A switch for source HARs: when on, outputs stay `.ts` instead of being converted to `.js`                                                                                                                                       |
| `singleFileEmit`                 | A pipeline-mode switch passed down by the build system. It toggles between "emit per file" and "batch emit at `buildEnd`", **and it decides which write-to-disk entry point declaration outputs go through**                    |
| Writing to disk                  | Which directory the compiler writes outputs to and under what relative path. In this requirement it is not "just call `WriteFile`"; it is a set of branching rules                                                              |
| Output record                    | `harFilesRecord` in the reference: a table of "source file → paths of each output it produced". Declaration bundling, obfuscation, and the final write all read it                                                              |
| `etsFortgz`                      | Root directory for declaration outputs of bytecode HARs and shared HSPs, located at `<module build dir>/../etsFortgz`                                                                                                           |
| `oh-exports`                     | The field in the package manifest `oh-package.json5` that declares which files the package allows to be exported                                                                                                                |
| Declaration bundling (declmerge) | The logic that merges a library's entry `.d.ets` together with the declarations it depends on into a single file, which becomes the only external type surface. In the reference it is the three `declaration_merger*.ts` files |
| Companion file                   | A by-product of declaration bundling, `<entry name>-declarations.d.ts` / `.d.ets`, holding the half of the declarations that crosses extensions                                                                                 |

---

## 1. Background

### 1.1 Publishing a library takes two things, and neither holds in tsgo today

The acceptance baseline project `sample_in_harmonyos` has 12 modules: 8 are libraries (`common` plus 7 features) and 4 are entry HAPs. For a library to be usable by other modules it has to ship a type surface, and that type surface is `.d.ets`. Getting there requires two things at once: the compiler must cut `.ets` down to `.d.ets` correctly, and those `.d.ets` files, along with the other outputs, must be placed where consumers can find them. Doing only the first still leaves the library unpublishable, which is why both live in the same requirement.

Neither holds in tsgo today, but to very different degrees.

**For declaration emit, tsgo has the complete upstream implementation; what's missing is the ArkTS layer.** The four files under `internal/transformers/declarations/` (`transform.go`, `util.go`, `diagnostics.go`, `tracker.go`) are a complete port of upstream declaration emit, and the `.ets → .d.ets` extension mapping is already implemented (`GetDeclarationEmitExtensionForPath` and `GetPossibleOriginalInputExtensionForExtension` in `internal/tspath/extension.go`). But these four files have **zero** ArkTS-related hits: annotation declarations, annotation properties, the decorator retention allowlist, `@Sendable`, export stripping inside ambient namespaces — none of it is there. On the reference side the corresponding changes are concentrated in `transformers/declarations.ts` and the helper file `ohApi.ts`, with three more in `checker.ts` and `transformers/declarations/diagnostics.ts`. Every change point is listed in the table in 1.8, each tagged with the delivery item in this document that owns it.

**For writing to disk, tsgo doesn't even have the corresponding layer.** tsgo has standard options such as `outDir` / `declarationDir`, which express "shift the whole tree onto another directory using one base". The reference's rules don't have that shape: the output set varies with module form, the base directory is chosen from three options, files under `oh_modules` are excluded wholesale, and in some pipelines the same file has its path computed twice against two different bases and is then copied pair by pair. In the reference this entire layer lives in ets-loader, not in the compiler.

### 1.2 The reference splits this into three segments

```
   hvigor passes down projectConfig (moduleType / byteCodeHar / useTsHar / singleFileEmit / bundledDeclare / directories …)
        │
        ▼
   ets-loader (ace_ets2bundle)
     · processHarSourceFiles()      walks "resolved modules ∪ root files", deciding per file: copy as-is, or re-emit
     · generateSourceFilesInHar()   picks one of three publishing paths, computes the destination, registers it in harFilesRecord
     · writeDeclarationFiles() / mangleDeclarationFileName()   two mutually exclusive write entry points, selected by singleFileEmit
     · performDeclarationMerging()  conditionally triggered declaration bundling, builds a second Program of its own
        │
        │  program.emit(sourceFile, writeFile, /*emitOnlyDtsFiles*/ true, …)
        ▼
   OH TSC (ohos-typescript)
     · transformers/declarations.ts   annotations, decorator retention, visibility trimming, ambient namespace handling
     · ohApi.ts                        emitDecorators allowlist engine, @Styles context, @Sendable check
     · checker.ts                      type inference and constant evaluation for annotation properties (inputs to declaration emit)
```

**Segment one is declaration emit inside the compiler.** This is a function-by-function port with a clear change surface and existing baselines to back it; it is the lowest-risk segment.

**Segment two is ets-loader's disk writing and bookkeeping.** tsgo has no corresponding layer, so we first have to decide where it hooks in and which inputs the build system passes down before talking about how to write it.

**Segment three is declaration bundling.** It is conditionally triggered and sized separately: `declaration_merger.ts` 1998 lines + `declaration_merger_utils.ts` 559 lines + `declaration_merger_sysapi.ts` 375 lines, about 2932 lines in total, with 117 fixtures. It is currently scheduled outside this requirement, but whether it actually triggers has been described incorrectly before; see 1.7.

These three segments must not be estimated together.

### 1.3 In declaration files, annotation rules are the exact opposite of ordinary properties

This is the most time-consuming part of the whole requirement, and the hardest to get right by intuition.

When an ordinary class property goes into a declaration file, its initializer is always dropped — that is exactly what `ensureNoInitializer` in `declarations.ts` does. Annotation properties are the opposite: the initializer **must** remain in the declaration file verbatim, because consumers rely on it at compile time to know "what this annotation property defaults to if not written". On top of that, properties without a type annotation must have their type inferred from the default value and filled in.

In the reference these two things are one line and one branch, respectively. Preserving the initializer is in `visitDeclarationSubtree` in `declarations.ts`, where the `AnnotationPropertyDeclaration` case passes `input.initializer` straight through to `factory.updateAnnotationPropertyDeclaration`; type inference is in `ensureType` in the same file, which on the `AnnotationPropertyDeclaration` branch first tries `createTypeOfDeclaration` and, failing that, falls back to `createTypeOfExpression(node.initializer, …)`.

Here is what an existing output looks like (test case `tests/cases/conformance/annotations/annotationDeclarationFieldInitializer1.ets`, expected output in the `.d.ets` section of `tests/baselines/reference/annotationDeclarationFieldInitializer1.js`):

```ets
// source
@interface Anno {
    a: number = +1
    d  = !1
    z = 1 == 1
}

// .d.ets
declare @interface Anno {
    a: number = +1;
    d: boolean = !1;
    z: boolean = 1 == 1;
}

// .js — empty; the annotation declaration is erased entirely from the script output
```

The picture is only complete with all three together: initializers kept verbatim, untyped `d` and `z` inferred as `boolean` from their defaults, and the `.js` side empty. The third is not part of this requirement (how annotations get erased from script outputs is a separate stream of work), but it explains why the declaration side must keep the initializers — the runtime cannot see annotation declarations, so default values can only travel through the declaration file.

**Expressions are carried over verbatim, not evaluated and then printed.** Look at the output of `annotationDeclarationFieldInitializer8`: `c: Array<Array<number>> = [[1 + a * b / 3], [E.A + b], [a + E.A], []];` — parentheses, operators, and identifier references all keep their source form. The checker does have a constant evaluator (`evaluateAnnotationPropertyConstantExpression` in `checker.ts`), but its result flows through `getAnnotationPropertyEvaluatedInitializer` on the `EmitResolver`, and its consumer is the script-output chain in `ohApi.ts`, **not** declaration emit. Declaration emit wants the original AST node.

**Emit proceeds even when evaluation fails.** In the `annotationDeclarationFieldInitializerError1` case all three properties report TS28034 (`Default value of annotation property can be a constant expression, got: '{0}'.`), yet the `.d.ets` still writes out `a: number = A;`, `b: number = B();`, `c: number = C;` verbatim. Diagnostics don't block emit; copy this behavior and don't add guards of your own.

Paired with this are two checker-side diagnostics, both reported in `checkAnnotationPropertyDeclaration` in `checker.ts`: TS28033 (annotation property type may only be number / boolean / string / const enum / arrays of those) and TS28034. There is also a special case: for a property of `const enum` type with no default, `addAnnotationPropertyEnumInitalizer` in the same file synthesizes an initializer of the form `new Number(0) as number` and stores it in `NodeLinks` — the reference's own comment calls it "a useless expression that is forbidden for users", used to carry the numeric type information of the enum member downstream. This, too, only goes through the script-output chain.

### 1.4 Decorator retention: tsgo is plainly wrong today, and you hit it without annotations or structs

Declaration files drop every decorator by default. ArkTS can't work that way — `@Component`, `@State`, `@Styles` and the like must stay in `.d.ets`, otherwise the type surface consumers get is incomplete. The reference uses an allowlist engine: `getEffectiveDecorators` in `ohApi.ts` reads `ets.emitDecorators`, keeps only names listed in the configuration, and for entries configured with `emitParameters: false` also reduces `@Consume("1")` to `@Consume`. Not configuring the option means dropping everything.

tsgo today unconditionally does `core.Filter(mods.Nodes, ast.IsModifier)` in `ensureModifiers` in `internal/transformers/declarations/transform.go`, filtering out every decorator. The option itself parses fine (`EtsOptions.EmitDecorators` in `internal/core/arkts.go`, `ParseEtsOptions` and `parseEtsEmitDecoratorsOptions` in `internal/tsoptions/arkts.go`), but apart from its own unit tests it has **no consumer at all**.

This has been verified: with `ets.emitDecorators = [Observed, State, Styles]` configured and input `@Observed export class Model { @State value: string = "a" }`, tsgo outputs `export declare class Model { value: string; }`, with both decorators lost.

**This piece deserves to be called out on its own because its trigger surface is far wider than the other items.** The annotation items have to wait for the checker to understand annotations, and the struct items have to wait for the ArkUI stream's struct transform; this one needs neither — an ordinary `class` or `function` written in an `.ets` file already hits it. The reference has 12 existing baselines backing it (see 5, Acceptance).

Two more items belong to the same group with related mechanics. One is `inEtsStylesContext` in `ohApi.ts`: methods and functions decorated with `@Styles` carry no type parameters in `.d.ets`; it is consumed in `visitDeclarationSubtree` and `transformTopLevelDeclaration` in `declarations.ts`. The other is `stripExportModifiersForEts` in `declarations.ts`: it strips `export` modifiers inside ambient namespaces while keeping ETS decorators and annotations. Upstream only has a variant that doesn't keep decorators, and so does tsgo. The reference baselines contain a clean example of what the two together produce (`functionAndClassWithDecorator.d.ets`):

```ets
declare namespace ns {
    class A {
        a: number;
    }
    function F(): void;
    @Styles
    function globalFancy();
    @Styles
    function globalFancy2();
}
```

All four members are written with `export` in the source; in the output the `export`s are gone while `@Styles` stays, and `globalFancy` doesn't even carry a return type.

### 1.5 Which files a library actually has to produce

The first question for the disk-writing half is not "where" but "what". In the reference this is decided by module form, branching in `generateModuleAbc` in `generate_module_abc.ts` (rollup's `buildEnd`): when `compileHar && !byteCodeHar` holds, it only runs `SourceMapGenerator.buildModuleSourceMapInfo` and then **`return`s**, skipping the entire `generateAbc`; every other form goes through full abc generation.

| Module form | Output set |
|---|---|
| Source HAR (`compileHar && !byteCodeHar`) | Source outputs (`.js`, or `.ts` when `useTsHar` is on) + `.d.ets` / `.d.ts` + sourcemap, **no abc** |
| Bytecode HAR (`compileHar && byteCodeHar`) | `modules.abc` + `.d.ets` / `.d.ts` + sourcemap |
| Shared HSP (`compileShared`) | Same as bytecode HAR, but the relative base is the project root |
| HAP | abc + sourcemap, **no declarations** (`processHarSourceFiles` is only called under `compileHar \|\| compileShared`) |

**`useTsHar` is the first fork in this table: it decides whether a source HAR outputs `.js` or `.ts`.** The extension is decided in `setIncrementalFileInHar` in `fast_build/ets_ui/rollup-plugin-ets-typescript.ts` (`useTsHar ? '.ts' : '.js'`), and the same function registers the cache→build mapping for both declarations and source outputs into `allFilesInHar`. Beyond that, two other places on the bundle side read it: `getOrCreateLanguageService` in `ets_checker.ts` uses whether it changed as a cache invalidation condition under `compileHar && !byteCodeHar` (`useTsHarDiff`; a change discards the whole cache); `process_ui_syntax.ts` emits a warning when it meets a `@Sendable` class in a JS HAR (`compileHar && !useTsHar`).

**How the text of source outputs is produced is not part of this requirement; their extension and destination are.** In other words, how the `.js` / `.ts` text comes about is not discussed here, but "whether this module should produce `.js` or `.ts`, and which directory it goes into" is.

### 1.6 Where these files go: three publishing paths, one record table, two write entry points

**The collection entry is `processHarSourceFiles` in `ets_checker.ts`**, called by `processBuildHap` in the same file under `compileHar || compileShared`. It walks "the set of resolved modules ∪ the set of root files" and does one of two things per file:

- if it already is a `.d.ets` / `.d.ts`, **read the whole file and copy it as-is**;
- if it is a `.ets` / `.ts`, emit it once on its own with `program.emit(sourceFile, writeFile, undefined, /*emitOnlyDtsFiles*/ true, undefined, true, true)`, capture the text, and write it out as `.d` + the original extension.

**The last two of those seven arguments deserve separate mention.** OH extended the `Program.emit` signature to `(sourceFile, writeFileCallback, cancellationToken, emitOnlyDtsFiles, transformers, forceDtsEmit, skipDiagnostics)`; the final `skipDiagnostics` was added by OH, upstream stops at `forceDtsEmit`. `forceDtsEmit: true` makes `emitWorker` in `program.ts` skip `handleNoEmitOptions` — **declarations are produced even when there are errors**; `skipDiagnostics: true` lets emit fill in annotation back-references without running type checking. Copying this call without understanding these two arguments will miss the "produce even on error" behavior.

Two filters must be copied: when compiling a HAR, anything whose path contains the package directory name (`oh_modules`) is skipped entirely; `.ets` / `.ts` under `/oh_modules/` are skipped as well. Both filters stop applying when `bundledDeclare` is on — with that switch on, declarations under `oh_modules` must be carried out too. One more thing matters even more: **the whole loop body is wrapped in `try { … } catch (err) {}`, so exceptions are swallowed without logging**. This is the reference's existing shape; copy it as-is and don't turn it into an error.

**What actually determines the path is `generateSourceFilesInHar` in `utils.ts`**, which combines three things into the destination path — the relative base, the output root, and a switch called `buildInHar` — and the three publishing paths are three combinations of them:

| Module form | Relative base | Output root | `buildInHar` |
|---|---|---|---|
| Shared HSP (`compileShared`) | Project root `projectRootPath` | `<module build dir>/../etsFortgz` | true |
| Bytecode HAR (`byteCodeHar`) | Module root `moduleRootPath` | `<module build dir>/../etsFortgz` | false |
| Source HAR | Module root `moduleRootPath` | `cachePath` | false |

`buildInHar` is more than a boolean: in `genTemporaryPath` in `utils.ts` it controls two things at once — whether the relative path keeps the module-name prefix, and whether a `TEMPORARY` directory level is inserted into the output path. When `compileHar && buildInHar` both hold, the module-name prefix is erased to an empty string (`moduleA/src/main/ets/test.js` → `src/main/ets/test.js`), which is the relative form inside a HAR package.

**Writing to disk also means registering an output record.** `harFilesRecord` in `utils.ts` is a table of "source file path → `GeneratedFileInHar`", with six fields: `sourcePath`, `sourceCachePath`, `obfuscatedSourceCachePath`, `originalDeclarationCachePath`, `originalDeclarationContent`, `obfuscatedDeclarationCachePath`. The two kinds of outputs are registered from two different places:

- **Declarations** are registered by `generateSourceFilesInHar`, and **by default only registered, not written** — `originalDeclarationContent` stays in memory; only when `bundledDeclare` is on is it written once at registration time.
- **Source outputs** are registered by `writeObfuscatedSourceCode` in `ark_utils.ts`: when `moduleInfo.originSourceFilePath` is non-empty it records `buildFilePath` into `sourceCachePath`, then writes directly with `writeFileSyncCaseAware`. The live rollup pipeline takes this path, not `generateSourceFilesInHar` — the latter's two `.js` call sites in `process_har_writejs.ts` and `result_process.ts` date from the webpack era and are no longer taken.

**Declaration outputs have two mutually exclusive write entry points, selected by `singleFileEmit`. This is the easiest thing to miss.**

- `singleFileEmit` on: the `beforeBuildEnd` handler in `rollup-plugin-gen-abc.ts` (`order: 'pre'`) calls `writeDeclarationFiles` in `ark_utils.ts`, which walks `harFilesRecord` writing `originalDeclarationContent` to `originalDeclarationCachePath`, then `return`s, skipping all later branches in the same handler.
- `singleFileEmit` off: goes through `handlePostObfuscationTasks` in `ob_config_resolver.ts` → `mangleDeclarationFileName` in `ark_utils.ts`, gated by `compileToolIsRollUp() && compileMode === ESMODULE`, **independent of whether obfuscation is on**; it likewise walks `harFilesRecord`, calling `writeObfuscatedSourceCode` on each entry, and with obfuscation off that function runs all the way to the final `writeFileSyncCaseAware` and writes the file out.

Both entry points must be covered, otherwise declarations stop being written as soon as the pipeline mode changes. A note on a distortion in the reference itself: the function comment of `writeDeclarationFiles` claims "Writes declaration files in debug mode", which doesn't match the actual gate (`singleFileEmit`); don't port it based on the comment.

**This layer needs nine inputs agreed with the build system.** This requirement depends on all of them; missing any one, the layer cannot compute destinations:

| Input | Meaning | Reference counterpart |
|---|---|---|
| `moduleType` | `entry` / `feature` / `har` / `shared`; decides the output set and which path is taken | Replaces the two booleans `compileHar` + `compileShared` |
| `byteCodeHar` | Whether this module is compiled as a bytecode HAR | `projectConfig.byteCodeHar` |
| `useTsHar` | Whether source HAR outputs stay `.ts` or become `.js` | `projectConfig.useTsHar` |
| `singleFileEmit` | Pipeline mode; also decides which write entry point declarations use | `projectConfig.singleFileEmit` |
| `emitDeclarations` | Whether to produce declarations | Derived from `compileHar` / `compileShared` |
| `declarationOutDir` | Root directory for declaration outputs | `declaredFilesPath`, or `<module build dir>/../etsFortgz` |
| `harOutDir` | The HAR's loader_out directory | `aceModuleBuild` / `buildPath` |
| `bundledDeclare` | Declaration bundling switch | `projectArkOption.bundle.bundledDeclare` |
| `declarationEntryFiles` | Entry set for declaration bundling | Computed from `oh-exports` or the manifest `main` |

**`harOutDir` cannot be merged into the standard `outDir`.** In the `compileHar && !byteCodeHar` pipeline, the two bases cachePath and buildPath are **both live in the same run**: the same file has its path computed once against cache and once against build (the second call additionally passes `buildInHar`, so the relative-path rules differ), and the files are then copied pair by pair — the `incrementalFileInHar` table in `handleFinishModules` in `compile_info.ts` is exactly the cache→build mapping. Either follow this shape, or explicitly decide that "the cache→loader_out copy belongs to the build system" and write the rules into the build-system side; see 4.

### 1.7 The trigger condition for declaration bundling has been stated wrongly twice

This trigger condition has previously been stated wrongly twice: once as "the project pulls in a bytecode HAR or a shared HSP", and once as `bundledDeclare && (byteCodeHar || compileShared)`. **Both are wrong.** The only hard gate is in `serviceChecker` in `ets_checker.ts`, and it looks at a single switch, `projectArkOption.bundle.bundledDeclare`; the subsequent `byteCodeHar || compileShared` only decides whether the merge's input directory is `declaredFilesPath` or `cachePath/<moduleName>`, and the value of the `isByteCodeHar` argument — it **does not decide whether it runs**; that else branch exists precisely for source HARs / HAPs.

The correct statement is: **for any module whose build we take over (HAP / HSP / source HAR / bytecode HAR), if `arkOptions.bundle.bundledDeclare` is on, declaration bundling runs.** It has nothing to do with being a bytecode HAR — a bytecode HAR with the switch off doesn't run it, a source HAR with the switch on does, and the latter additionally overwrites the entry `.d.ets`. The deciding factor is location, not a switch: declaration bundling runs to completion inside the etsChecker plugin's `buildStart` handler (`rollup-plugin-ets-checker.ts` → `serviceChecker` in `ets_checker.ts`), while es2abc's input lists are all produced at `buildEnd`; the two are not in the same phase.

The scheduling consequence is direct: declaration bundling is currently outside this requirement, and whether it comes in **depends on whether the baseline project has `bundledDeclare` on**, not on the HAR packaging form. Nobody has measured this value yet; it is listed under decisions needed before work starts.

**The entry call chain is recorded here too, so nobody has to look it up again and a spike doesn't have to start from scratch.** The entry is `performDeclarationMerging` in `ets_checker.ts`: it first takes `oh-exports` from `oh-package.json5` at the module root, falling back to the `main` field with its extension changed to `.d.<original extension>` when empty, computes the entry set, and then calls `DeclarationMerger.mergeDeclarationFiles` in `declaration_merger.ts`. The latter's main chain is `mergeEntry` → `collectExports` → `collectTypeDependencies` → `resolveNames` → `emit`; it builds a second Program of its own (`noEmit: true` + an empty `host.writeFile`), and module resolution uses `createDeclarationModuleResolver` in `declaration_merger_utils.ts` (a resolver of its own, not reusing the main Program's); output self-validation is in `validateOutput` in the same file; naming of the cross-extension companion file is in `DeclarationMerger.getCompanionFileName` (of the form `<entry name>-declarations.d.ts` / `.d.ets`); the system API branch lives separately in `declaration_merger_sysapi.ts`. The merge result is written back into `originalDeclarationContent` of `harFilesRecord` — in other words, it plugs into the record table from 1.6 and does not write to disk itself.

### 1.8 Overview of the reference's declaration emit change points

The 19 rows below are every ArkTS change the reference makes to **the declaration emit segment** relative to upstream TypeScript (the disk-writing half is in 1.5 and 1.6, declaration bundling in 1.7; neither is in this table). The "Owner" column points to a delivery item in §2 of this document; those marked "out of scope" are covered in 3. Use this table as a checklist when porting declaration emit.

| # | Change point | Reference location | Owner |
|---:|---|---|---|
| 1 | `.d.ets` emit for `@interface` (does not call `ensureEtsDecorators`, visits all members) | `transformTopLevelDeclaration` in `declarations.ts`, `AnnotationDeclaration` branch | 2.1 |
| 2 | Annotation properties pass `initializer` through (in contrast to ordinary properties going through `ensureNoInitializer`) | `visitDeclarationSubtree` in `declarations.ts`, `AnnotationPropertyDeclaration` branch | 2.1 |
| 3 | The `HasInferredType` type alias and `ensureType`'s kind list include annotation properties | `HasInferredType` and `ensureType` in `declarations.ts` | 2.1 |
| 4 | Top-level statement allowlist includes annotation declarations | `isPreservedDeclarationStatement` in `declarations.ts` | 2.1 |
| 5 | Member allowlist includes annotation properties | `ProcessedComponent` and `isProcessedComponent` in `declarations.ts` | 2.1 |
| 6 | Serialization base for inferred types and TS4xxx anchors include annotation declarations | `isEnclosingDeclaration` in `declarations.ts` | 2.1 |
| 7 | Visibility trimming includes annotation declarations (without it, unexported annotations leak into `.d.ets`) | `isDeclarationAndNotVisible` in `declarations.ts` | 2.1 / 2.4 |
| 8 | `ets.emitDecorators` allowlist engine, including the `emitParameters: false` reduction | `getEffectiveDecorators` in `ohApi.ts` | 2.2 |
| 9 | The three entry points of the allowlist engine | `ensureEtsDecorators` / `getReservedDecoratorsOfEtsFile` / `getReservedDecoratorsOfStructDeclaration` in `ohApi.ts` | 2.2 |
| 10 | Methods and functions decorated with `@Styles` carry no type parameters | `inEtsStylesContext` in `ohApi.ts`; consumed in `visitDeclarationSubtree` and `transformTopLevelDeclaration` in `declarations.ts` | 2.2 |
| 11 | `@Sendable`'s `illegalDecorators` kept as-is, bypassing allowlist filtering | `isSendableFunctionOrType` in `ohApi.ts`; consumed in `transformTopLevelDeclaration` in `declarations.ts` (TypeAlias and FunctionDeclaration branches) | 2.3 |
| 12 | Visibility check for annotation references, with its 18 call sites | `checkAnnotationVisibilityByDecorator` in `declarations.ts` | 2.4 |
| 13 | Picking out applied annotations and re-attaching them, 20 call sites in total in `declarations.ts` (including 2 in the struct branch and 2 in `stripExportModifiersForEts`) | `getAnnotations` and `getAnnotationsFromIllegalDecorators` in `utilitiesPublic.ts` | 2.4 |
| 14 | Node types for declaration diagnostics include annotation properties (the **only** ArkTS changes in that file) | `DeclarationDiagnosticProducing`, `canProduceDiagnostics`, `createGetSymbolAccessibilityDiagnosticForNode` in `transformers/declarations/diagnostics.ts` | 2.4 |
| 15 | Strip export inside ambient namespaces while keeping decorators and annotations | `stripExportModifiersForEts` in `declarations.ts`; called from `transformTopLevelDeclaration` in the same file | 2.5 |
| 16 | `.d.ts` triple-slash references exclude `oh_modules` | `mapReferencesIntoArray` in `declarations.ts`; decided via `isOhpm` and `isOHModulesReference` in `ohApi.ts` | 2.6 |
| 17 | nodeBuilder's portable-specifier guard extended to `oh_modules` | `symbolToTypeNode` in `checker.ts`, using `isOhpmAndOhModules` in `ohApi.ts` | 2.6 |
| 18 | Special case for the `Resource` type in nodeBuilder (emit a bare type reference) | `symbolToTypeNode` in `checker.ts`, right next to the previous item; from commit `a0f18856d2` | 2.6 |
| 19 | `.d.ets` emit for struct (members skip the virtual constructor, heritage dropped entirely), plus the matching virtual TypeReference bail | `transformTopLevelDeclaration` in `declarations.ts` (`StructDeclaration` branch) and `ensureType` | Out of scope |

tsgo has **none** of these 19 rows today: the four files under `internal/transformers/declarations/` have zero ArkTS hits, and the same-named `symbolToTypeNode` in `internal/checker/nodebuilderimpl.go` lacks the branches for rows 17 and 18. The only thing that comes close is the **option** in row 8 — `ets.emitDecorators` parses correctly, but has zero consumers.

---

## 2. What we need to deliver

### 2.1 Support annotation declarations and annotation properties in declaration files

Support `@interface` declarations and their properties appearing in `.d.ets`, with property default values preserved verbatim and types inferred from the default for properties without an explicit type annotation.

**What this item is about.** The example in 1.3 has to work; this corresponds to rows 1–7 of the table in 1.8: both kinds must enter declaration emit's dispatch table; annotation properties must take the "pass the initializer through" path rather than "clear the initializer"; `ensureType` must recognize both kinds and fall back to computing from the initializer when the declared type is unavailable; the top-level statement and member allowlists must let them through; visibility trimming and the serialization base must include them.

On the tsgo side this lands in `internal/transformers/declarations/transform.go` (the counterparts of the dispatch table and `ensureType`) and `internal/transformers/declarations/util.go` (`isDeclarationAndNotVisible`, `isEnclosingDeclaration`, `canProduceDiagnostics`).

**There are two traps to avoid together.** First, the allowlist and dispatch table must change together: adding annotation declarations to `isPreservedDeclarationStatement` without the corresponding case in `transformTopLevelDeclaration` just trades the printer's panic for a panic in the transform phase. Second, this item is blocked by the checker in tsgo: annotation declarations are currently bound as Class in tsgo, `isArkTSClassLike` in `internal/ast/arkts.go` only covers Struct, and the combination of `@interface` plus `@Anno class` panics in the checker; `getDeclarationSpaces` in `internal/checker/checker.go` has no corresponding case and panics on its last line. These two are prerequisites; without resolving them this item can't pass acceptance.

Scenarios:
- Exported annotation declarations go into `.d.ets`; unexported ones don't
- Annotation properties with defaults keep them verbatim (including parentheses and operator forms)
- Annotation properties without a type get it inferred from the default
- Emit proceeds normally when the default is not a constant expression (TS28034 reported)
- Annotation declarations appearing inside an ambient namespace

### 2.2 Support retaining decorators per the `ets.emitDecorators` allowlist

Support deciding which decorators are retained in declaration files per the `ets.emitDecorators` configuration, reducing calls with arguments to bare names for entries configured with `emitParameters: false`; also support erasing virtual type parameters in `@Styles` contexts.

**What this item is about.** See 1.4; this corresponds to rows 8–10 of the table in 1.8. It is the only part of this requirement that "is already a bug today and depends on no prerequisites": no annotations, no structs, no binder changes needed — ordinary classes and functions in `.ets` already hit it, and 12 existing baselines back it.

On the tsgo side, the change is in `ensureModifiers` in `internal/transformers/declarations/transform.go` — today it unconditionally filters out every decorator.

**Note that this mechanism is shared with the ArkUI stream.** This requirement owns the engine, but its consumers today are all UI decorators (`@Component` / `@State` / `@Styles` / `@Observed` and friends). Verify it on the basis of "this path works"; don't take over the semantics of the UI decorators along with it.

Scenarios:
- Allowlisted decorators without arguments are kept as-is
- Allowlisted decorators with `emitParameters: false` and arguments are reduced to bare names
- Decorators not on the allowlist are dropped
- Everything is dropped when `ets.emitDecorators` is not configured
- Functions and methods decorated with `@Styles` carry no type parameters in declaration files

### 2.3 Support keeping `@Sendable` as-is in declaration files

Support carrying the decorators on `@Sendable`-decorated type aliases and function declarations into `.d.ets` verbatim, bypassing allowlist filtering.

**What this item is about.** The reference opens a dedicated path for `@Sendable`: on the TypeAlias and FunctionDeclaration branches of `transformTopLevelDeclaration` in `declarations.ts`, once `isSendableFunctionOrType` in `ohApi.ts` matches, it directly does `clean.illegalDecorators = input.illegalDecorators`, **without** going through `getAnnotationsFromIllegalDecorators` filtering. The check itself is strict: the node must be a function declaration or type alias, must be in an `.ets` file, must have **exactly one** entry in `illegalDecorators`, and that name must be exactly `Sendable`. One extra decorator and this path is not taken.

`@Sendable` has zero hits in the baseline project's sources, but `@Sendable` is within this team's scope, and its path into declaration files runs in parallel with the annotations in 2.1; without it, the type surface consumers get would be missing the marker.

Scenarios:
- A `@Sendable`-decorated `type` alias goes into the declaration file with the decorator kept
- Same for a `@Sendable`-decorated `function` declaration
- When a declaration carries more than one decorator, this path is not taken and it falls back to allowlist filtering

### 2.4 Support declaration visibility trimming and visibility checks for annotation references

Support keeping unexported annotation declarations out of `.d.ets`, and running an entity-name visibility check on every annotation reference appearing in `.d.ets`.

**What this item is about.** A declaration file must not reference a name consumers can't see, otherwise the consumer gets "cannot find this type" at compile time. Upstream already has a complete `checkEntityNameVisibility` mechanism for type references; the reference extends it to annotations: `checkAnnotationVisibilityByDecorator` in `declarations.ts` runs the check on both the callee of `@Anno({...})` and the initializers of object-literal members, with 18 call sites. The trimming half is adding the annotation-declaration case to `isDeclarationAndNotVisible`.

On the tsgo side the primitives are already in `internal/transformers/declarations/transform.go`; what's missing is the ArkTS wiring. `isDeclarationAndNotVisible` and `isEnclosingDeclaration` in `internal/transformers/declarations/util.go` are the two corresponding functions. The three spots in `transformers/declarations/diagnostics.ts` (the only ArkTS changes in that file) are also needed, bringing annotation properties into the anchor computation for declaration diagnostics.

Scenarios:
- Unexported annotation declarations do not appear in `.d.ets`
- `.d.ets` references an unexported annotation → visibility diagnostic reported
- An annotation argument object literal references an unexported name → same
- Applied annotations on every declaration emit branch are picked out and re-attached (the reference has 20 calls to `getAnnotations` / `getAnnotationsFromIllegalDecorators` in `declarations.ts`; the 2 in the struct branch are out of scope)

### 2.5 Support stripping export inside ambient namespaces while keeping decorators and annotations

Support removing members' `export` modifiers inside ambient namespaces in declaration files, while keeping ETS decorators and annotations.

**What this item is about.** Members of an ambient namespace are implicitly exported, so upstream strips `export`. The reference must re-attach decorators while stripping, and for that it has a separate variant, `stripExportModifiersForEts` in `declarations.ts`: for `ClassDeclaration` it uses `ensureEtsDecorators + getAnnotations`, and for `VariableStatement` it uses `getAnnotationsFromIllegalDecorators`. tsgo today only has the variant that doesn't keep decorators, in `internal/transformers/declarations/transform.go`.

For the output shape, see the `functionAndClassWithDecorator.d.ets` example at the end of 1.4.

Scenarios:
- Classes inside an ambient namespace have export stripped and decorators and annotations kept
- Same for functions inside an ambient namespace
- Variable statements inside an ambient namespace keep their annotations
- `export import` and `export default` are not stripped (explicitly exempted in the reference)

### 2.6 Support three output-shape fixes for declaration files

Support excluding entries under `oh_modules` when generating triple-slash references in `.d.ts`, writing cross-ohpm-package type references as portable specifiers, and emitting a bare type reference instead of `import("…").Resource` for the SDK `Resource` type.

**What this item is about.** None of these three is about "trimming source"; they change the shape of the output, so they are grouped together, corresponding to rows 16–18 of the table in 1.8.

On triple-slash references: declaration files automatically carry a set of `/// <reference path="…" />`. Upstream already drops those under `node_modules` (because paths are unreliable once packages are installed), and the reference adds an equivalent for `oh_modules` in `mapReferencesIntoArray` in `declarations.ts`.

**The portable specifier item matters most.** When `symbolToTypeNode` in `checker.ts` serializes a type pointing into another file, it first generates a specifier and then checks it: if it contains `/node_modules/`, it is "a relative path diving into a package directory", which is not portable, so it regenerates one in a different mode as a bare package name. The reference extends this guard to ohpm (via `isOhpmAndOhModules` in `ohApi.ts`). The same-named `symbolToTypeNode` in tsgo's `internal/checker/nodebuilderimpl.go` hardcodes `/node_modules/` in two places, with no ohpm branch.

Not doing it hits the library's external type surface: cross-package `import("…")` in `.d.ets` gets written as a relative or even absolute path diving into `oh_modules`; the consumer project's directory layout differs from the publisher's, so that path points to nothing — **and compiling the publishing module reports not a single diagnostic**.

The guard itself is a single line, but for it to take effect, the underlying generator that "derives a specifier back from a file path" must also understand `oh_modules`. On the reference side that part is in `moduleSpecifiers.ts`, with four substantive pieces: `getAllModulePathsWorker` decides by package manager type "whether this path is inside a package directory"; `getNearestAncestorDirectoryWithPackageJson` picks the manifest filename by type and walks up to find the package root; `getNodeModulePathParts` switches the separator fragment used to split a path into "package root / package name / path within package" to `/oh_modules/`; `tryDirectoryWithPackageJson` reads the package-root manifest. The tsgo counterpart is `internal/modulespecifiers`. This machinery is the same thing as "making the compiler understand the `oh_modules` package directory and the `oh-package.json5` manifest" (same `packageManagerType` option, same `ohApi.ts` helpers), so ownership needs to be settled once; see 4. Either way, this item is responsible for wiring up the guard and verifying it in output comparison.

The `Resource` item is new: when the module specifier ends with `/ets/api/global/resource` or `/ets/dynamic/api/global/resource` and the last item of the symbol chain is named `Resource`, emit a bare `Resource` type reference instead of `import("…").Resource`. It sits right next to the previous item in the same function.

Scenarios:
- No `oh_modules` paths appear in `.d.ts` triple-slash references
- Cross-ohpm-package type references are written as bare package names in `.d.ets`, not relative paths
- Specifiers stay portable when the referenced file comes in through a symlink
- Declaration outputs referencing the SDK `Resource` type contain a bare `Resource`

### 2.7 Support deciding the output set and publishing path by module form

Support determining "which files to produce" by module form, choosing the relative base and output root among the three paths of shared HSP / bytecode HAR / source HAR, and replicating the package-directory exclusion and the `buildInHar` path rules.

**What this item is about.** See the output set table in 1.5 and the path table in 1.6. On the reference side it lands in three functions: the form branch in `generateModuleAbc` in `generate_module_abc.ts` (source HARs `return` early here, skipping all abc generation), and path computation in `generateSourceFilesInHar` and `genTemporaryPath` in `utils.ts`. The work is to add this layer to tsgo: it reads the module form and directories passed down by the build system and decides each output's destination; files under `oh_modules` are excluded wholesale by default, and included instead when `bundledDeclare` is on.

**Don't try to build this layer out of `outDir` / `declarationDir`.** Those two options express a whole-tree shift from a single base, while in the reference the base is one of three depending on form, and on the `compileHar && !byteCodeHar` line two bases are live at once. Forcing it doesn't produce errors; it produces outputs in the wrong place — consumers fail to resolve them and the compile step gives no signal at all.

Scenarios:
- Source HARs produce no abc, only source outputs, declarations, and sourcemaps
- Shared HSP: lands in `etsFortgz` with the project root as base
- Bytecode HAR: lands in `etsFortgz` with the module root as base
- Source HAR: lands in the cache directory with the module root as base
- Files under the package directory (`oh_modules`) are excluded, and included when `bundledDeclare` is on
- When `buildInHar` is in effect, the module-name prefix is erased and no `TEMPORARY` level is inserted

### 2.8 Support collecting and recording declaration outputs, and the two write entry points

Support carrying existing `.d.ets` / `.d.ts` out whole, re-emitting `.ets` / `.ts` as declarations one file at a time, registering them in the output record table, and writing them to disk through one of the two entry points depending on pipeline mode.

**What this item is about.** On the reference side it is four functions: collection in `processHarSourceFiles` in `ets_checker.ts`, the record table as `harFilesRecord` and `GeneratedFileInHar` in `utils.ts`, and writing via `writeDeclarationFiles` in `ark_utils.ts` (when `singleFileEmit` is on, called from `beforeBuildEnd` in `rollup-plugin-gen-abc.ts`) and `mangleDeclarationFileName` (when `singleFileEmit` is off, called from `handlePostObfuscationTasks` in `ob_config_resolver.ts`, independent of the obfuscation switch). Details in 1.6.

The record table is not an internal implementation detail — declaration bundling relies on it to write back merge results, obfuscation relies on it to locate declaration outputs, and both write entry points walk it. The tsgo side needs an equivalent data structure and its lifecycle (when entries are registered, which entry point writes them, how they are invalidated under incremental builds).

"Which files get carried out" is itself a rule: the traversal set is "resolved modules ∪ root files", i.e. export reachability plus the transitive import closure, and the root file set for a library build is decided by the package manifest's `oh-exports`. The `oh-exports` decision logic is already written on the tsgo side, but `internal/module/resolver.go` currently hardcodes nil, so the data channel isn't connected — this requirement needs it as the root file set, and module resolution needs it to report "this file is not in the package's export list"; both uses are waiting on the same slot, and it is easy for neither side to claim it.

Scenarios:
- Existing `.d.ets` / `.d.ts` copied whole
- `.ets` / `.ts` emitted one file at a time into `.d.ets` / `.d.ts`, with `forceDtsEmit` (produce even on error)
- A failed single-file emit is silently skipped without interrupting the whole run (reference behavior, copy it)
- Declarations are written in both `singleFileEmit` on and off modes
- With `bundledDeclare` on, registering means writing immediately
- The root file set is decided by `oh-exports`, falling back to the manifest `main`

### 2.9 Support source outputs choosing their extension by `useTsHar` and going into the HAR

Support source HAR source outputs staying `.ts` or becoming `.js` according to `useTsHar`, and registering them in the output record table.

**What this item is about.** See the `useTsHar` paragraph in 1.5. How the **text** of source outputs is produced is not part of this requirement; this item is responsible for two things: the extension is decided by `useTsHar` (in the reference, in `setIncrementalFileInHar` in `rollup-plugin-ets-typescript.ts`); and the destination is registered into `sourceCachePath` of `harFilesRecord` — in the live rollup pipeline this is done by `writeObfuscatedSourceCode` in `ark_utils.ts`, not `generateSourceFilesInHar`.

**Watch out for a reverse dependency in this item.** Two places on the bundle side read this state back: `getOrCreateLanguageService` in `ets_checker.ts` treats a change in `useTsHar` as a condition for invalidating the whole cache under `compileHar && !byteCodeHar`; `process_ui_syntax.ts` warns about `@Sendable` classes in a JS HAR. Get the extension wrong and more than the output itself breaks.

Scenarios:
- `useTsHar` off: source HAR produces `.js`
- `useTsHar` on: source HAR produces `.ts`
- In both cases the destination is registered into `sourceCachePath` of the output record table
- On a source HAR, a change in `useTsHar` invalidates the whole cache

### 2.10 Support declaration diagnostics for library builds

Support computing and reporting declaration diagnostics only when building libraries, skipping declaration files themselves, `.js` files, and files under `oh_modules`.

**What this item is about.** Declaration diagnostics (the TS4xxx batch, "inferred type references a name that isn't visible") only make sense when declarations are actually being produced. The reference hangs them on the `compileHar || compileShared` branch: `processBuildHap` in `ets_checker.ts` calls `printDeclarationDiagnostics` in the same file, which explicitly skips those three kinds of files.

This item has a seam with the "diagnostic behavior alignment" stream of work: whether a given diagnostic should be dropped or downgraded belongs to that stream; this item only owns the gate of "when to compute and for which files".

Scenarios:
- No declaration diagnostics when building a HAP
- Computed when building a HAR / HSP, skipping `.d.ets` / `.d.ts` / `.js` and files under `oh_modules`

---

## 3. Out of scope

Six things look like declaration-file work on the surface, but all already have owners; don't re-decide them. "Others" here doesn't mean other documents; it means the several streams of work proceeding in parallel: the ArkUI stream (struct and UI decorator semantics), the script-output side (intermediate `.js` / abc outputs and annotation erasure), the stream making the compiler understand `oh_modules` (package directories, manifests, visibility diagnostics), the diagnostic behavior alignment stream, and obfuscation and packaging. Each item below says which stream it belongs to.

**`.d.ets` emit for struct belongs to the ArkUI stream.** The reference location is the `StructDeclaration` branch of `transformTopLevelDeclaration` in `declarations.ts` (members skip the virtual constructor, heritage dropped entirely). This directly affects acceptance of this requirement; see 5. One more warning: tsgo main doesn't synthesize a virtual constructor when an explicit one already exists, so copying "visit from index 1" would delete the user's own constructor; it must filter by `ast.NodeIsVirtual` instead — leave this to whoever takes over struct, and don't do it in passing in this requirement.

**Erasure and evaluation of annotations in `.js` and abc intermediate outputs is a separate stream of work (the script-output side).** The dividing line is clear: whatever goes through `getAnnotationPropertyEvaluatedInitializer` / `getAnnotationPropertyInferredType` on the `EmitResolver` belongs there; declaration emit reads the original AST nodes, and the two sides don't share a path. The same goes for the annotation erasure in `transformers/ts.ts`, `classFields.ts`, and `legacyDecorators.ts`.

**Declaration bundling (declmerge) is currently outside this requirement.** But its trigger condition has been stated wrongly twice before (see 1.7); the real gate is the single `bundledDeclare` switch. Nobody has measured this value on the baseline project; if it turns out to be on, this whole block (about 2932 lines, 117 fixtures) comes back into this requirement.

**Two obfuscation-related things are not done:** the declaration file name obfuscation interface (`tryMangleFileName` in `ark_utils.ts`) and generating the consumer obfuscation configuration when publishing a HAR. Obfuscation is off within acceptance scope. Note that "once obfuscation is turned on" this requirement grows again, and tsgo would need to provide the obfuscator with the full set of program files and a per-file dependency map — nobody owns that today. Another reminder: although `mangleDeclarationFileName` has "mangle" in its name, it must also run with obfuscation off (it is one of the two write entry points in 1.6); don't cut it along with "the obfuscation stuff".

**Resolution-time visibility decisions for `oh-exports` belong to the stream making the compiler understand `oh_modules`.** This requirement only uses `oh-exports` as input for "which root files to carry out", and is not responsible for the diagnostic chain of "a resolved file not in the export list is an error and is kept out of the module graph".

**HAR packaging, signing, and installation after outputs are written belong to hvigor.** The exit point of this requirement is "every file that should be produced has been produced and written in the right place"; archiving is not included.

---

## 4. Decisions needed before work starts

**Whether `bundledDeclare` is actually on in the baseline project.** This decides whether declaration bundling (about 2932 lines, 117 fixtures) enters this requirement's scope, and is the single largest uncertainty in this requirement. The deciding value is the actual setting of `arkOptions.bundle.bundledDeclare` in `build-profile.json5`; measure it when the baseline project snapshot is locked. Until the answer comes back, the implementation keeps a fail-loud guard: on reading `bundledDeclare === true`, exit with a diagnostic rather than silently treating it as off — silently treating it as off means the library builds as usual and declarations are produced as usual, but they are per-file declarations rather than a single merged type surface, and that only surfaces in the consumer project.

**The producer form of the 8 HARs.** This decides which of the output sets in 1.5 and which of the three paths in 1.6 are actually exercised on the baseline, and, if declaration bundling triggers, whether its input directory is `declaredFilesPath` or `cachePath/<moduleName>`. Two fields look alike but are not the same thing, and are easily mixed up when drawing conclusions:

- `buildOption.arkOptions.byteCodeHar` is the **producer-side** switch, saying "which form this module itself is compiled in". With project-level `useNormalizedOHMUrl: true`, leaving it unset means bytecode HAR; a source HAR requires explicitly writing `false`.
- `byteCodeHarInfo` is the **consumer-side** table, saying "which of the packages I depend on were installed as `.har` archives". The criterion is the form of the dependency, not "whether this HAR is one we compiled ourselves" — the decisive evidence is that `getHarData` in `parseUserIntents.ts` derives the package root from `abcPath` via `split('ets')[0]` and then appends `src/main/resources/...`, a path derivation that only holds for the shape of an unpacked `.har` archive, not for a local module's `build/` directory. The baseline project's dependencies are all `file:../../<mod>` source directories, so this table is `{}` — but that **cannot** be read backwards to tell which form those 8 modules are compiled in.

This requirement needs the former, and nobody has measured its actual value. The decisive measurement: after a clean build, check `loader.json` and whether `filesInfo.txt` exists under each module's cachePath, and whether declaration outputs land in `etsFortgz` or the cache directory. This does not block starting on 2.1–2.6.

**The values of the `useTsHar` and `singleFileEmit` switches on the baseline.** The former decides whether source HARs produce `.js` or `.ts` (the whole decision in 2.9); the latter decides which write entry point declarations use (the two branches in 2.8). Both are passed down by the build system and neither has been measured; measure them together with the item above when locking the snapshot. Both branches must be implemented either way — but scheduling needs to know which one the baseline actually takes.

**Who writes declaration outputs to disk.** Two options: tsgo writes directly to the final directories, or tsgo writes only to cache and the build system copies into loader_out. The nine inputs in 1.6 are currently listed for the former (`harOutDir` and `declarationOutDir` given separately). Until this is settled, 2.7 has no inputs.

**Who owns the specifier-generation machinery.** The second item of 2.6 depends on `internal/modulespecifiers` understanding `oh_modules`, and that machinery (the four functions in `moduleSpecifiers.ts`) is the same thing as "making the compiler understand the `oh_modules` package directory and the `oh-package.json5` manifest" — same `packageManagerType` option, same `ohApi.ts` helpers. Whoever brings `oh_modules` module resolution into tsgo is best placed to do this part too; this requirement is responsible for wiring up the guard in `symbolToTypeNode` and verifying it in output comparison. **Neither side claiming it is the worst outcome**, because when it's wrong the compile step reports zero diagnostics.

**Who takes the `oh-exports` slot.** The decision logic is ready on the tsgo side, but `internal/module/resolver.go` hardcodes nil and the data channel isn't connected. It has two uses: this requirement needs it as the root file set (2.8), and module resolution needs it to report "this file is not in the package's export list". Both uses are waiting on the same slot, and it is easy for neither side to do it.

**Whether to do the `@Retention` / `SourceRetention` family.** The reference has a set of checks (`isRetentionAnnotationDeclaration`, `checkSourceRetentionAnnotation`, `hasSourceRetentionPolicy` in `checker.ts`) that decide whether a given annotation must be erased from `.js` and `.d.ets`. Half of it lands in declaration emit (this requirement) and half in the SDK API availability checking logic; ownership hasn't been settled.

**Annotation property type inference depends on the checker; how to schedule the prerequisites.** "Infer the type from the default" in 2.1 ultimately lands on `createTypeOfExpression`, and tsgo's checker currently panics on annotations (see the two spots at the end of 2.1). What needs deciding is whether those two prerequisites count toward this requirement or toward "making the checker understand annotation declarations" — the latter being an independent piece of work in its own right.

**The golden mechanism for emit-type acceptance.** tsgo's comparison tests today only compare AST, symbol tables, and diagnostics, not output text, while everything this requirement is accepted on is output text. This infrastructure cost must be written into the schedule explicitly, or review will get stuck here.

---

## 5. Acceptance

1. The reference's 12 declaration baselines (`_submodules/third_party_typescript/tests/dets/`, sources in `cases/`, expected outputs in `baselines/reference/*.d.ets`) pass byte for byte.
2. Annotation declarations match the reference's 9 existing outputs (`tests/cases/conformance/annotations/annotationDeclarationFieldInitializer1..8` and `annotationDeclarationFieldInitializerError1`; expected outputs are the `.d.ets` sections in `tests/baselines/reference/*.js`).
3. The output set and directory layout for each module form match the existing toolchain: compile the same module once with the existing toolchain and once with tsgo, and compare the sets of file paths under both output directories; then feed the produced library into a consumer project and compile it once, with no resolution diagnostics.

Four external preconditions need to be stated plainly.

**For criterion 1 to run, that `ets` configuration has to be brought over first.** The `tests/dets/tsconfig.json` for those 12 cases carries: `declaration: true` + `emitDeclarationOnly: true`, 19 `emitDecorators` entries (all `emitParameters: false`, including `Entry` / `Component` / `State` / `Styles` / `Builder` / `Observed`, etc.), 4 `propertyDecorators`, `render` method and decorator, about 100 `components`, the full `extend.components` mapping, `styles`, `customComponent: "CustomComponent"`, `syntaxComponents`, plus `target: es2021` / `module: commonjs`. Without bringing this configuration over, 2.2 — "the only delivery item that can start immediately and has golden backing" — cannot be verified.

**10 of the 12 baselines in criterion 1 contain struct, so acceptance will be blocked by the ArkUI stream.** Counting file by file, only `functionWithDecorators.d.ets` and `functionAndClassWithDecorator.d.ets` are entirely free of struct; the other 10 contain `struct` anywhere from 1 to 17 times (`statusManagementOfPageLevelVariables.d.ets` has the most). In other words, only those 2 in criterion 1 are decided by this requirement alone; the other 10 must wait for struct declaration emit to land. There are two ways to handle this; pick one before starting: split criterion 1 into "2 verified immediately, 10 verified together with struct", or extract the struct-independent differences from those 10 into this requirement's own test cases. Don't write a blanket "12 baselines pass" and then discover during integration that it can't be verified.

**These 12 baselines are not in any runner today.** OH's own mocha never runs them (the entry `src/testRunner/unittests/tsc/etsTests.ts` is not imported by `tests.ts`, and it shells out to `lib/tsc.js`), and nothing on the tsgo side consumes them either — none of the 12 case names can be found in this repo's `testdata/` or `internal/`. Wiring them in has to be built from scratch, and their configuration explicitly lists an external SDK file `interface/sdk-js/api/@internal/component/ets/index-full.d.ts` (present on this machine); that dependency must be handled or trimmed into the fixture as well.

**The 9 expected outputs for criterion 2 are high quality and can be used directly.** They are OH TSC's own conformance cases, with the `.d.ets` sections inside the `.js` baseline files, covering unary / binary / comparison / logical operators, identifier references, `const enum` members, array and nested array literals, and the error form of "default is not a constant expression". The accompanying diagnostics are TS28033 and TS28034, both of which are missing in tsgo today.

One last item is not part of this requirement but will be triggered by it: once `ensureModifiers` starts keeping decorators, baselines involving `.ets` under `testdata/baselines/reference/` will drift in bulk. Per repo convention, this drift must be explained category by category before being accepted; don't just run `baseline-accept`.
