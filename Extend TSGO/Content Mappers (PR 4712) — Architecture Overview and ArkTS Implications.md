# Content Mappers (Upstream PR 4712) — Architecture Overview and ArkTS Implications

Upstream `microsoft/typescript-go` PR [#4712](https://github.com/microsoft/typescript-go/pull/4712) ("Content mappers", by andrewbranch, merged 2026-08-19T21:13Z via merge queue as commit `01b9e721f3d7f8037d700daff94f5808c1afb97e`, 105 commits, **364 files, +21,267 / −1,282**, branch deleted after merge) implements the content-mapper design discussed in the external-integration API design track (microsoft/TypeScript#63800). This document summarizes the mechanism for architects and assesses what it means for the tsgo-arkts port.

---

## 1. What Content Mappers Are

Content mappers are upstream's sanctioned answer to "TypeScript cannot load files it doesn't understand" (`.vue`, `.svelte`, `.mdx`, …). Since the Go server cannot dynamically link third-party code (the old tsc plugin API is gone), integration happens through **external child processes**: a project declares an npm package as a *content mapper* for certain file extensions; at program construction tsgo spawns that package, sends it the file text, and receives back **virtual TypeScript text** plus a **span map** that relates virtual positions to original positions. The virtual text is parsed, bound, checked, and served by the language service like any normal `.ts` file; positions in diagnostics and LSP results are mapped back to the original document through the span map.

Diff scale: **364 files, +21,267 / −1,282, 105 commits**. All major subsystems are touched: compiler (`fileloader.go` +294, `program.go` +173), a new `internal/spanmap` package (+778), a new `internal/contentmapper` package (host + IPC implementation ≈ +2,000), LSP (`server.go` +560, `lsconv/converters.go` +307, and every LS feature file), tsconfig parsing (+207 in `tsconfigparsing.go` alone), build/incremental/watch, and the API encoder (protocol version 5 → 7).

---

## 2. Architecture

### 2.1 Configuration surface

**tsconfig** — top-level array, alongside `files`/`include`/`exclude`, inherited through `extends` (`internal/tsoptions/tsconfigparsing.go:31,1169–1205`):

```jsonc
{
  "contentMappers": [
    { "package": "@vitejs/vue-mapper", "extensions": [".vue"], "options": { "…": "opaque to tsgo" } }
  ]
}
```

`Definition` = `{ package, extensions, options }` (`internal/contentmapper/contentmapper.go`). The mapper package is located by ordinary node module resolution from the config directory and **never executed during resolution**; its `package.json` provides the manifest (`internal/packagejson/packagejson.go:117–143`, `internal/tsoptions/contentmappers.go`):

```jsonc
// package.json of the mapper package
{
  "name": "@vitejs/vue-mapper",
  "version": "1.2.3",
  "typescript": {
    "contentMapper": {
      "exec": ["node", "bin/mapper.js"],   // non-empty array required
      "compilerOptions": ["jsx", "strict"], // subset of compiler options the transform depends on
      "dynamicConfig": false                 // mapper returns extra watched files / config identity
    }
  }
}
```

`name@version` forms the mapper **identity**; processes are consolidated by identity, so all projects sharing a mapper version share one process.

### 2.2 Process and protocol model

Per identity, tsgo lazily spawns the mapper (argv from `exec`, cwd = package directory) and talks **JSON-RPC over STDIO**, reusing `internal/ipc`. The content-mapper protocol has its **own version (1)** — separate from the API protocol (7). Four methods (`internal/contentmapper/hostimpl.go:31–42`):

- **`initialize`** — request: `{ protocolVersion: 1, positionEncodings: ["utf-8","utf-16"] }` (locale is also passed); response selects `positionEncoding` and a **`diagnosticSource`**: a prefix for every mapper-authored diagnostic code. The prefix must be non-empty, must not be `typescript`/`tsc`, and must not equal the mapped extension (so mapper diagnostics and tsgo diagnostics never collide, `hostimpl.go:1096–1125`).
- **`openProject` / `closeProject`** — project-scoped lifecycle; with `dynamicConfig: true` the mapper may return a `configIdentity` (changes invalidate cached transforms) and `watchedFiles` (fed into watch mode).
- **`transform`** — request `{ fileName, content }`; response:

```go
type Result struct {
    Text                 string                          // virtual TypeScript source
    VirtualExtension     string                          // how Text is parsed
    Diagnostics          []*ast.Diagnostic               // syntax errors in ORIGINAL positions
    Mappings             *spanmap.SpanMap                // virtual ↔ original
    DiagnosticDirectives []ast.MappedDiagnosticDirective // Ignore / Expect policies
    Supplemental         []MappedResult                  // extra unnamed virtual outputs
}
```

`VirtualExtension` is restricted to `.js/.jsx/.mjs/.cjs/.ts/.tsx/.mts/.cts/.json` (`contentmapper.go`). Failure handling is explicit: transform errors become diagnostics; a mapper that fails 5 times is disabled for the rest of the run with a program-level diagnostic (`fileloader.go:32–34`); timings (spawn/initialize/openProject/closeProject/transform) are instrumented for `--stats`.

### 2.3 Span maps (`internal/spanmap`, +778)

Unlike source maps (point correspondences; "no origin" implicit), a span map records **explicit half-open segments** for the parts of the virtual text that correspond to the original. Any virtual position **not covered by a segment is synthesized** — content with no original counterpart. An empty map therefore means "fully synthesized output". All positions are absolute offsets (`core.TextPos`), matching the compiler's `TextRange` model.

- **Kind**: `Verbatim` (length-preserving, interior maps 1:1), `Atom` (maps a whole span; lengths may differ, interior clamps to endpoints — for renamed identifiers/short expressions), `Alias` (atom geometry + asserts both texts name the same logical entity; diagnostics may substitute the original name).
- **Feature**: a 20-bit mask per segment selecting which LS operations may use it — hover, signature help, completion, definition, type definition, implementation, references, document highlights, rename, call hierarchy, code actions, formatting, inlay hints, semantic tokens, folding, selection ranges, linked editing, auto-insert, document symbols, code lens. **Diagnostics deliberately bypass the mask** (they may not opt out) and text edits require exact `Verbatim` geometry.
- **Fidelity** of a query result: `Exact` (within one verbatim segment — safe for edits written back to the original), `Atom`, `Approximate` (crossed boundaries), `None` (synthesized gap). Bidirectional queries (`VirtualToOriginal*` and `OriginalToVirtual*`) plus `Validate()`.

### 2.4 Program integration

`fileloader.go` folds the mapper extensions into supported extensions and into the **module resolver** (a `.vue` import resolves like a module); on load, files with mapped extensions are transformed before parsing, with parse-cache keys that preserve the mapper context (`ContentMapperParseOptions`, `fileloader.go:89–98`). Mapper-authored syntax diagnostics and the program's own diagnostics are aggregated; **diagnostic directives** (`Ignore`/`Expect` per virtual range, with an `unusedCode` for unsatisfied expectations) let a mapper suppress or require specific tsgo diagnostics inside synthesized regions — e.g. Vue's `expect-error` in template expressions.

### 2.5 Emit

Deliberate asymmetry (`internal/compiler/emitter.go:477–481`): **content-mapped files are not emitted to JS** — runtime output is owned by the mapper or the build tool. **Declaration emit is supported** (`App.d.svelte.ts`-style canonical names); declaration *maps* are disabled because mapped positions would point into in-memory text (`emitter.go:227–230`, with a comment reserving span-map double-mapping as future work). **Supplemental outputs** (e.g. per-module generated declarations) get compiler-assigned numbered `.d.ts` names, with collision detection.

### 2.6 Incremental builds and watch

`TransformIdentity` — `xxh3(identity ‖ mapper options ‖ declared compiler options)` (`contentmapper.go`) — is folded into cache keys and `.tsbuildinfo`, so a mapper-version or relevant-option change invalidates exactly the affected transforms. Build mode propagates mapper failures into build exit status; `dynamicConfig`/`watchedFiles` integrate with `--watch` and `--build --watch`; process lifetimes are tied to project sessions.

### 2.7 Security

Content mappers are **arbitrary code execution by design** (they run from the project's `node_modules`), so they are double-gated: the compiler option `RunExternalCode` (`internal/execute/tsc/compile.go:86–91`) — without it `NewContentMapperHost` returns nil and **no mapped files load at all** — and, on the editor side, VS Code passes `runExternalCode` only in trusted workspaces (`js/ts.contentMappers.enabled`, experimental, default true).

### 2.8 LSP and API surface

The server registers **per-feature dynamic capabilities** for mapped documents (did-open/change/close, diagnostic, hover, signature-help, definition, type-definition, implementation, references, document-highlight, completion, rename, semantic-tokens, document-symbol, folding-range, selection-range, inlay-hint, code-lens, code-action, formatting, range/on-type-formatting, linked-editing — `lsp/server.go:324–347`); every LS feature maps its results through the span map. The extension API adds `registerContentMappers` (`_extension/src/contentMapperContributions.ts`), including contributions for **inferred projects**. API protocol goes 5 → 7; `SourceFile` gains `originalText`, `spanMap`, `contentMapper`, `virtualFileName`, `diagnosticDirectives`, `supplementalSourceFileNames`, `canonicalSourceFileName`.

---

## 3. Assessment for the tsgo-arkts Port

### 3.1 Content mappers are NOT a replacement for the in-compiler ArkTS pipeline

Four structural reasons, plus one documented conformance reason:

1. **No semantics across the boundary.** `transform` receives only `{fileName, content}` and returns text. The ArkTS UI transform (ViewPU generation in `internal/transformers/arktsuitransforms/`) consumes bind/check information — decorator semantics, imported symbols, struct constraints, `$$` bindings — that exists only inside the program. A mapper cannot reproduce it.
2. **No checker parity.** Mapper diagnostics are positioned syntax errors; ArkTS type rules (no-`any`, struct rules, Sendable constraints — the 92-rule linter unified in the checker) require type information. A mapper could only emit them by embedding the entire old ArkTS checker — the very fork the port retires.
3. **Emit is excluded upstream on purpose.** Mapped files do not emit JS. The hvigor integration is built on tsgo's own emit (`.ets` → JS + sourcemaps → es2abc bookkeeping in `internal/buildapi/`); a mapper path would hand emit back to an external tool, i.e. regress to the ets2bundle model.
4. **PDCP conformance conflict.** PDCP Core principle ② mandates **Transform consolidation inside the tsgo process** ("перенос AST-преобразований ArkUI со стороны Node.js внутрь процесса tsgo для устранения узкого места межпроцессного взаимодействия" — see `PDCP_Core_ArkTS_RU.md`). An external content mapper doing ArkTS transforms is exactly the "AST over IPC" bottleneck that principle exists to eliminate. Re-architecting the build path around content mappers would **violate the project's own conformance document**.

### 3.2 The merge is mandatory and must be a dedicated task

The fork's base predates both `internal/ipc` (restructured in this PR) and `internal/spanmap` (new). The API encoder protocol goes 5 → 7 (fork is at 5, `internal/api/encoder/encoder.go:65`), and `SourceFile`'s wire shape grows — the build-driving daemon (`internal/buildapi/session.go` implements `api.Handler` in-process) and the hvigor-side client (`hvigor-poc/tsgo-plugin.ts`) must be re-verified against protocol 7 after the pull. Concrete conflict surface where the PR meets ArkTS edits:

| Upstream (PR 4712) | Fork (ArkTS) | Collision |
|---|---|---|
| `fileloader.go` +294 | parse options `EtsAnnotationsEnable/EtsCustomComponent/EtsOptions/EtsLoaderPath/NoTransformedKitInParser` at `fileloader.go:372–379` | PR adds `ContentMapperParseOptions` to the same option struct and same file |
| `emitter.go` +24 | UI transformer chain for `ScriptKindETS` (`emitter.go:140–152`) and `StripStructTransformer` in decl emit (`emitter.go:253–256`) | same file, adjacent regions |
| `ast.go` +137, `module/resolver.go` +23, `tsoptions/*` +207 | ArkTS hooks in `ast/`, `module/`, `core/arkts.go` (EtsOptions), `tsconfigparsing` | textual/structural conflicts expected |
| `lsp/server.go` +560, `lsconv/converters.go` +307 | LS currently untouched in the fork (deferred) | low direct risk, high future relevance |

Recommended: a dedicated rebase task (one commit, no ArkTS work mixed in), followed by re-validation of the buildapi daemon and the hvigor client against protocol 7, before any Phase work resumes on top.

### 3.3 Mechanisms worth adopting into the fork after the merge

These are in-process compiler mechanisms — consistent with PDCP ② — that directly replace current fork hacks and shrink the diff against upstream:

- **Span maps for synthesized UI-transform regions.** Generated nodes today get `core.UndefinedTextRange()` (`internal/transformers/arktsuitransforms/component.go:1008`, `arkts_normalizedurl.go:66,74,104`). Span maps make "this region is synthesized" a first-class, queryable property, with per-feature participation — and give declaration diagnostics/positions a principled home (upstream itself reserves span-map double-mapping for declaration maps as future work, `emitter.go:227–230`).
- **`virtualFileName` / `canonicalSourceFileName` / supplemental outputs** instead of the bespoke virtual-name rename layer in `internal/buildapi/arktsmanifest.go:26–28,92–96,159–166` (which currently renames emitted `.ets` JS to `.ts` names on disk to imitate the reference toolchain's filesInfo naming). Upstream now formalizes canonical-vs-virtual file identity; the es2abc bookkeeping can reuse it.
- **Diagnostic directives** instead of ad-hoc suppression around generated declarations — cf. the lookup guard at `internal/binder/nameresolver.go:173–179` ("Generated declarations (e.g. ArkTS UI transform output) have no symbol").
- **`contentMapperExtensions` seam for `.ets` inclusion** — upstream now has an official mechanism for "extra extensions in the program"; where possible, the fork's `.ets` inclusion in `fileloader.go` should route through the same seam rather than parallel patches.

### 3.4 Optional research experiment (explicitly NOT a build-path plan)

A cheap experiment to inform long-term fork-minimization: package the already-ported Go ArkTS logic as a separate binary behind a thin mapper, and measure how many ArkTS-specific checks survive the mapper-diagnostics model. If most do, a *hybrid* becomes thinkable — mapper for syntax/editor, fork reduced to checker hooks only. If not (expected), the result still documents precisely why in-compiler integration is required. This experiment does not touch the build pipeline and does not violate PDCP ②; it is research, not a plan.

### 3.5 Editor-side upside

If DevEco's language service ever migrates onto tsgo, `.ets` files can be exposed through a content-mapper contribution: the per-feature registration matrix (§2.8) then provides hover/completion/rename/references/etc. in original coordinates for free. That removes the deferred LS-port work from the fork's backlog — a concrete reason to keep LS untouched in the fork for now and to monitor upstream's extension API (`registerContentMappers`) evolution.

### 3.6 Conformance summary

| Conformance anchor | Status |
|---|---|
| PDCP ① atomic capabilities (parse/check/lint-only) | unaffected by PR 4712 |
| PDCP ② Transform consolidation inside tsgo | aligned: reject external-mapper build path (§3.1); adopt in-process mechanisms (§3.3) |
| PDCP ③ unified 92-rule checker | unaffected; reinforced (§3.1.2) |
| Roadmap tasks (orchestration, incremental, parallel) | PR's build/watch/tsbuildinfo integration is adjacent upstream machinery; not a substitute for fork tasks |
| KPI "artifact consistency 100% binary diff" | post-rebase re-baselining required (protocol 7, new SourceFile fields) |

---

## 4. Recommended Actions

1. **Dedicated rebase task** on upstream main (post-4712), no ArkTS work mixed in; re-verify buildapi daemon + hvigor client against API protocol 7; re-run byte-parity goldens.
2. **Adoption backlog** (post-rebase, in order): span maps for synthesized UI-transform regions; virtual/canonical file names + supplementals in `arktsmanifest.go`; diagnostic directives for generated-declaration suppression; `.ets` inclusion through the `contentMapperExtensions` seam.
3. **Do not** pursue content mappers as the ArkTS build path — PDCP ② violation plus loss of checker parity and emit (§3.1).
4. **Optional:** run the §3.4 mapper experiment; monitor `registerContentMappers` for the DevEco LSP scenario.
5. The in-repo plan (`docs/plans/035-upstream-4712-content-mappers.md`) is to be written once the working repository is free; this document is its reference.

## 5. References

- PR: <https://github.com/microsoft/typescript-go/pull/4712> (merged `01b9e72`, 2026-08-19)
- Design track: <https://github.com/microsoft/TypeScript/issues/63800>
- `PDCP_Core_ArkTS_RU.md`, `tsgo_project_roadmap_en.md` (same vault folder)
- Upstream code: `internal/spanmap/spanmap.go`, `internal/contentmapper/{contentmapper,host,hostimpl,transform}.go`, `internal/compiler/{fileloader,emitter,program}.go`, `internal/lsp/server.go`, `internal/api/encoder/encoder.go`
- Fork code: `internal/compiler/{fileloader,emitter}.go`, `internal/transformers/arktsuitransforms/*`, `internal/buildapi/{session,arktsmanifest}.go`, `internal/binder/nameresolver.go`, `internal/api/encoder/encoder.go`
