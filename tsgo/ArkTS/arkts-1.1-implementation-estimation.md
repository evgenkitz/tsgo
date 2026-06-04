# ArkTS 1.1 Implementation Estimation for tsgo

Based on scope analysis from [[arkts-1.1-specification-analysis]].
**Cross-referenced with current tsgo codebase — items already supported are excluded.**

## Assumptions

- One senior engineer familiar with compiler internals
- Full-time dedication (8h/day, 5 days/week)
- Changes are incremental and testable at each phase
- Scope: Scanner, Parser, Binder only

---

## Items Already Supported in tsgo (Excluded from Estimate)

The following ArkTS 1.1 features were found to already exist in tsgo and are **removed** from this estimate:

| # | Feature | Where |
|---|---|---|
| 1 | String interpolation (template literals) | Scanner + Parser — `parseTemplateExpression()` |
| 2 | Labeled statements | Parser — `parseExpressionOrLabeledStatement()` + Binder — `bindLabeledStatement()` |
| 3 | Namespace declarations (`namespace N { }`) | Parser — `parseModuleDeclaration()` + Binder — `bindModuleDeclaration()` |
| 4 | Ambient declarations (`declare`) | Parser — `parseModifiersEx()` + Binder — `NodeFlagsAmbient` |
| 5 | Export type (`export type { ... }`) | Parser — `parseExportDeclaration()` + Binder |
| 6 | `in`/`out` type parameter variance | Parser — `parseTypeParameter()` via modifiers |
| 7 | Static initializer blocks (`static { }`) | Parser — `parseClassStaticBlockDeclaration()` |
| 8 | For-of loop with type annotation | Parser — `parseVariableDeclarationWorker()` |
| 9 | Typed catch clauses (`catch (e: T)`) | Parser — `parseCatchClause()` |
| 10 | `readonly` prefix on array/tuple types | Parser — `parseTypeOperatorOrHigher()` |
| 11 | `const enum` parsing & binding | Parser — modifier + `parseEnumDeclaration()`, Binder — `SymbolFlagsConstEnum` |
| 12 | `this` parameter in functions | Standard TypeScript feature |
| 13 | Primitive type resolution (number, string, etc.) | Checker — intrinsic types (checker.go L969-1009) |

---

## Phase 1 — Basic Recognition (Scanner)

| Task | Effort |
|---|---|
| 12 new keywords in `textToKeyword` map + Kind entries + regenerate | 1d |
| Character literal scanning (`c'...'`) | 2d |
| Float suffix scanning (`3.14f`) | 1d |
| `LanguageVariant` plumbing (ArkTS mode switch) | 1d |
| Classification helpers (`IsPrimitiveTypeKind`, etc.) | 0.5d |
| Unit tests | 2d |
| **Subtotal** | **7.5 days** |

> No change from original — all 12 keywords are genuinely new, char literals and float suffix are new.

---

## Phase 2 — Basic Parsing (Parser)

| Task | Effort |
|---|---|
| Annotation parsing (`@interface`, `@Anno(...)`) | 5d |
| Lambda expressions (reworked from arrow functions for type params, annotations) | 5d |
| Package declaration (`package P`) | 2d |
| `struct` keyword class handling | 1d |
| New modifiers (`final`, `native`, `internal`) — `override`/`abstract`/`async` already exist | 2d |
| Node definitions in `_scripts/ast.json` + regeneration (fewer nodes needed) | 2d |
| Parser dispatch updates (`parseStatement`, `parseExpression`) | 1.5d |
| Type-level parsing (`T!`, function type with receiver) — `readonly` already exists | 1.5d |
| Tests | 4d |
| **Subtotal** | **24 days** |

> Removed from original: import/export directive updates (already supported), `readonly` type prefix (already parsed).
> Reduced: AST node definitions (fewer new Kinds), dispatch updates (fewer new cases), tests.

---

## Phase 3 — Basic Binding (Binder)

| Task | Effort |
|---|---|
| Predefined symbols for 7 new primitive types (byte..char) | 2d |
| Scope rules — package scope, internal access (namespace scope already exists) | 3d |
| Annotation symbol binding (create symbols, bind fields to declarations) | 3d |
| `GetContainerFlags` updates for new Kinds | 1.5d |
| SymbolFlags / NodeFlags additions | 1d |
| Name resolution for package scope | 2d |
| Tests | 3d |
| **Subtotal** | **15.5 days** |

> Removed: numeric type hierarchy (checker), annotation field validation (checker), `@Retention` semantics (checker).
> Reduced: scope rules (only package + internal are new), tests.

---

## Phase 4 — Experimental Features (Parser + Binder)

| Task | Effort |
|---|---|
| Lambda with receiver + implicit `this` (parser + binder) | 5d |
| Trailing lambdas (parser + binder) | 5d |
| `launch` expression (`await` already exists in TS) | 3d |
| Function declarations with receiver | 3d |
| Function types with receiver | 2d |
| Array creation expressions (`new type[dims]`) | 3d |
| Operator methods (`$_get`, `$_set`, `$_iterator`, `$_invoke`, `$_instantiate`) — binder symbols only | 4d |
| Default interface methods (parser + binder) | 2d |
| Tests | 5d |
| **Subtotal** | **32 days** |

> Removed: FixedArray (SDK type, no compiler changes). Reduced: operator methods (call dispatch is checker).

---

## Summary

| Phase | Scope | Effort |
|---|---|---|
| 1 — Scanner | 12 keywords, char literals, float suffix, LanguageVariant | **7.5 days** |
| 2 — Parser | Annotations, lambdas, package, struct, modifiers, T! | **24 days** |
| 3 — Binder | 7 primitive type symbols, package scope, annotation binding | **15.5 days** |
| 4 — Experimental | Coroutines, receivers, trailing lambdas, operator methods | **32 days** |
| **Total** | | **79 days** |

### Comparison: Before vs After Cross-Reference

| Phase | Original | Corrected | Δ |
|---|---|---|---|
| 1 — Scanner | 7.5d | 7.5d | — |
| 2 — Parser | 27d | 24d | −3d |
| 3 — Binder | 24.5d | 15.5d | −9d |
| 4 — Experimental | 50.5d | 32d | −18.5d |
| **Total** | **109.5d** | **79d** | **−30.5d** |

---

## Bottom Line

| Team Size | Duration |
|---|---|
| **One engineer** | **3.5–4 months** |
| **Two engineers** | **2 months** |
| **Three engineers** | **1–1.5 months** |

---

## Scope Boundary

This estimate covers **Scanner, Parser, and Binder only**.

### Included
- Lexical analysis: new keywords, char literals, float suffix
- Syntax parsing: all ArkTS 1.1 grammar productions not already in tsgo, AST node construction
- Symbol binding: scope management, symbol table population, container flags

### Explicitly NOT Included
- **Checker** — type checking, type inference, assignability, boxing/unboxing semantics, implicit conversions, type hierarchy enforcement
- **Emitter** — code generation
- **Standard library** — `Promise<T>`, `BigInt`, `FixedArray`, boxed types, etc.
- **LSP / IDE support**
- **Formatter**
- **ArkTS↔TypeScript interop**

---

## Sources

- [[arkts-1.1-specification-analysis]] — Updated with cross-reference against tsgo codebase
- ArkTS 1.1 Specification (Release 1.1.0, 2025-03-04)
- tsgo codebase: `internal/scanner/scanner.go`, `internal/parser/parser.go`, `internal/binder/binder.go`
