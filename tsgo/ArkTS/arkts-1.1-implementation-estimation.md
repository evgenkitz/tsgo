# ArkTS 1.1 Implementation Estimation for tsgo

Based on scope analysis from [[arkts-1.1-specification-analysis]].

## Assumptions

- One senior engineer familiar with compiler internals
- Full-time dedication (8h/day, 5 days/week)
- Changes are incremental and testable at each phase

---

## Phase 1 — Basic Recognition (Scanner)

| Task | Effort |
|---|---|
| ~40 new keywords in `textToKeyword` map + Kind constants | 1d |
| Character literal scanning (`c'...'`) | 2d |
| Float suffix scanning (`3.14f`) | 1d |
| `LanguageVariant` plumbing (ArkTS mode switch) | 1d |
| Regenerate/update `kind_generated.go` ranges | 1d |
| Classification helpers (`IsPrimitiveTypeKind`, etc.) | 0.5d |
| Unit tests | 2d |
| **Subtotal** | **7–9 days** |

---

## Phase 2 — Basic Parsing (Parser)

| Task | Effort |
|---|---|
| Annotation parsing (`@interface`, `@Anno(...)`) | 5d |
| Lambda expressions (reworked from arrow functions) | 5d |
| Package declaration (`package P`) | 2d |
| `struct` keyword class handling | 1d |
| New modifiers (`final`, `override`, `native`, `internal`) | 2d |
| Node definitions in `_scripts/ast.json` + regeneration | 3d |
| Parser dispatch updates (`parseStatement`, `parseExpression`, etc.) | 2d |
| Type-level parsing (`readonly` prefix, `T!`, function type with receiver) | 2d |
| Import/export directive updates (`import type`, `export default`, `export type`) | 2d |
| Tests | 5d |
| **Subtotal** | **25–29 days** |

---

## Phase 3 — Basic Binding (Binder)

| Task | Effort |
|---|---|
| Predefined symbols for 12+ primitive/boxed types | 3d |
| Numeric type hierarchy (`double > float > long > int > short > byte`) | 3d |
| Scope rules (package, namespace, internal access) | 5d |
| Annotation binding and validation | 5d |
| `GetContainerFlags` updates for new Kinds | 2d |
| SymbolFlags / NodeFlags additions | 1d |
| Name resolution for new scope types | 3d |
| Tests | 5d |
| **Subtotal** | **24–27 days** |

---

## Phase 4 — Experimental Features (Parser + Binder)

| Task | Effort |
|---|---|
| Lambda with receiver + implicit `this` | 5d |
| Trailing lambdas | 5d |
| `launch` / `await` expressions (coroutines) | 5d |
| Function declarations with receiver | 3d |
| Function types with receiver | 2d |
| Array creation expressions (`new type[dims]`) | 3d |
| FixedArray type | 2d |
| Initializer blocks (`static { ... }`) | 2d |
| Operator methods (`$_get`, `$_set`, `$_iterator`, `$_invoke`, `$_instantiate`) | 5d |
| Namespace declarations | 3d |
| Ambient declarations (`declare ...`) | 5d |
| Labeled statements, typed catch, default interface methods, for-of type annotation | 5d |
| Tests | 8d |
| **Subtotal** | **48–53 days** |

---

## Phase 5 — Full Semantics (Binder + Checker)

| Task | Effort |
|---|---|
| Boxing/unboxing semantics (primitive ↔ boxed) | 5d |
| Implicit conversions (widening, narrowing, constant narrowing) | 10d |
| Default values for all primitive types | 2d |
| NonNullish type parameter (`T!`) semantic checks | 3d |
| `const enum` skip semantics | 1d |
| `readonly` array/tuple semantics | 2d |
| Truthiness/extended conditional expressions | 3d |
| Integration & regression tests | 10d |
| **Subtotal** | **31–36 days** |

---

## Summary

| Phase | Scope | Effort |
|---|---|---|
| 1 — Scanner | Keywords, literals, LanguageVariant | **7–9 days** |
| 2 — Parser | Annotations, lambdas, package, struct, modifiers | **25–29 days** |
| 3 — Binder | Primitive types, scopes, annotation binding | **24–27 days** |
| 4 — Experimental | Coroutines, receivers, trailing lambdas, operator methods | **48–53 days** |
| 5 — Full Semantics | Boxing, conversions, type hierarchy | **31–36 days** |
| **Total** | | **135–154 days** |

---

## Bottom Line

| Team Size | Duration |
|---|---|
| **One engineer** | **6–7 months** |
| **Two engineers** | **3–4 months** |
| **Three engineers** | **2–3 months** |

---

## Out of Scope (not included in this estimate)

- Checker (type checker) full integration — Phases 1-3 cover binder only, the checker is an additional major component
- Emitter / code generation
- Standard library implementation (`Promise<T>`, `BigInt`, `FixedArray`, etc.)
- LSP / IDE support
- Formatter
- ArkTS↔TypeScript interop
- JS/TS→ArkTS migration tooling

The checker alone could add another 3–5 months of effort given the complexity of ArkTS's type system
(boxing/unboxing, value types, numeric hierarchy, union normalization differences).

---

## Sources

- [[arkts-1.1-specification-analysis]]
- ArkTS 1.1 Specification (Release 1.1.0, 2025-03-04)
