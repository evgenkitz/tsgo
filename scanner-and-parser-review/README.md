# Scanner & Parser Plans — Requirements Review

This folder contains copies of all scanner/parser plans for review against the SR decomposition dimensions. 8 plans, each corresponding 1:1 to a gitcode issue.

## Plan Inventory

| # | Plan | Phase | Status |
|---|------|-------|--------|
| #2 | Scanner, AST, and file extensions | Phase 1 | ✅ Closed |
| #3 | EtsFlags context-flag system | Phase 1/2 | ✅ Closed |
| #4 | @interface annotation declarations | Phase 1/2 | ✅ Closed |
| #5 | struct declarations | Phase 2 | Open |
| #6 | Decorator modifications + auto-readonly | Phase 2 | Open |
| #7 | EtsComponentExpression, virtual type arguments, UICallback | Phase 2 | Open |
| #12 | Kit import resolution (KitImportFlags + processKit) | Phase 2 | Open |
| #13 | EtsOptions compiler configuration | Phase 2 | Open |
| #17 | ArkUI loop components (ForEach/LazyForEach/Repeat.each) | Phase 2 | Open |

Note: Plan #7 consolidates issues #7, #8, and #9 into a single implementation plan.

## Requirements Coverage Matrix

| Dimension | #2 | #3 | #4 | #5 | #6 | #7 | #12 | #13 | #17 |
|-----------|----|----|----|----|----|----|----|----|----|
| 1. Demand value | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 2. Requirement scenario | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 3. Target users | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 4. Restrictions & constraints | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 5. External dependency | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 6. Performance indicators | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ⚠️ |
| 7. Acceptance Strategy | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

### Legend
- ✅ Present and adequate
- ⚠️ Present but not quantifiable (parser/scanner level — no runtime perf targets applicable)
- ❌ Missing

### Acceptance Strategy Note
All plans include AST comparison testing via `arkts_cmp_test.go` — verifying that our Go parser output matches the reference tsc for both AST shape and diagnostics. Test cases are either hand-written (in `testdata/arkts/`) or sourced from the reference's `arkTSTest/testcase/` directory.
