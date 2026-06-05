# ArkTS 1.1 Implementation Estimation for tsgo

Based on scope analysis from [[arkts-1.1-specification-analysis]].
**Cross-referenced with current tsgo codebase — items already supported are excluded.**

## Assumptions

- One senior engineer familiar with compiler internals
- Full-time dedication (8h/day, 5 days/week)
- Changes are incremental and testable at each phase
- Scope: Scanner, Parser, Binder only

---


## Phase 1 — Core Types & Lexical Extensions

New primitive types and their literals. Foundation everything else builds on.

| Task | Component | Effort |
|---|---|---|
| 12 new keywords in `textToKeyword` map + Kind entries + regenerate `ast.json` | Scanner | 1d |
| Character literal scanning (`c'...'`) | Scanner | 2d |
| Float suffix scanning (`3.14f`) | Scanner | 1d |
| `LanguageVariant` plumbing (ArkTS mode switch) | Scanner | 1d |
| Classification helpers (`IsPrimitiveTypeKind`, etc.) | Scanner | 0.5d |
| Predefined symbols for 7 new primitive types (byte..char) | Binder | 2d |
| `GetContainerFlags` / SymbolFlags / NodeFlags for new types | Binder | 1.5d |
| Tests | | 3d |
| **Subtotal** | | **12 days** |

---

## Phase 2 — Declarations, Modifiers & Scopes

Annotations, package, struct, access modifiers — structural building blocks.

| Task | Component | Effort |
|---|---|---|
| Annotation declaration parsing (`@interface`) | Parser | 3d |
| Annotation usage parsing (`@Anno(...)`) | Parser | 2d |
| Annotation symbol binding (symbols, fields → declarations) | Binder | 3d |
| Package declaration parsing (`package P`) | Parser | 2d |
| Package scope + internal access binding | Binder | 3d |
| `struct` keyword class handling | Parser | 1d |
| New modifier parsing (`final`, `native`, `internal`) | Parser | 2d |
| `GetContainerFlags` updates for new Kinds | Binder | 1.5d |
| Name resolution for package scope | Binder | 2d |
| Default interface methods (body in interface) | Parser | 2d |
| Tests | | 3d |
| **Subtotal** | | **24.5 days** |

---

## Phase 3 — Lambdas, Coroutines & Expressions

Features that extend the expression/statement grammar: lambdas, receivers, trailing lambdas, coroutines, array creation, T!.

| Task | Component | Effort |
|---|---|---|
| Lambda expressions (type params + annotations on lambdas) | Parser | 5d |
| Lambda with receiver (`(this: T, ...) => body`) | Parser | 3d |
| Lambda with receiver — binder scope + implicit `this` | Binder | 2d |
| Trailing lambdas | Parser | 5d |
| Trailing lambda binding | Binder | 1.5d |
| `launch` expression | Parser | 3d |
| `launch` expression binding (scope + `Promise<T>` result) | Binder | 1.5d |
| Array creation expressions (`new type[dims]`) | Parser | 3d |
| `T!` non-nullish type parameter | Parser | 1.5d |
| Operator methods (`$_get`, `$_set`, `$_iterator`, `$_invoke`, `$_instantiate`) | Binder | 4d |
| Node definitions in `_scripts/ast.json` + regeneration | Parser | 2d |
| Tests | | 5d |
| **Subtotal** | | **37.5 days** |

---

## Summary

| Phase | Scope | Effort |
|---|---|---|
| 1 — Core Types & Lexical | 12 keywords, char literals, float suffix, primitive type symbols | **12 days** |
| 2 — Declarations & Scopes | Annotations, package, struct, modifiers, default interface methods | **24.5 days** |
| 3 — Lambdas & Expressions | Lambdas, receivers, trailing lambdas, launch, array creation, T!, operator methods | **37.5 days** |
| **Total** | | **74 days** |

---

## Bottom Line

| Team Size | Duration |
|---|---|
| **One engineer** | **3.5 months** |
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
