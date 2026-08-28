# tsgo Project Roadmap

> **Version**: v1.1  
> **Date**: 2026-07-02  
> **Status**: Planning Established  
> **Based on**: [[8-tsgo迁移方案设计/tsgo-migration-design]] v0.2 + [[日常/2026-07-02]] Project Requirements

---

## 1. Project Overview

| Item                   | Details                                                                                                                                                      |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Project Name**       | 7.0 TD Project                                                                                                                                               |
| **Duration**           | 2026.04.07 ~ 2027.04.30                                                                                                                                      |
| **Development Period** | 2026.07.02 → 2027.01.31 (7 months, 28 weeks)                                                                                                                 |
| **Total Duration**     | 4 phases · 28 weeks                                                                                                                                          |
| **Core Objective**     | Cover 10% ArkTS syntax checking, complete ArkUI syntax transformation, establish core compilation pipeline, support ArkTS project build to artifact packages |

### Key Milestones

| Milestone | Date | Description |
|-----------|------|-------------|
| **Charter** | 2026.04.07 | Project kickoff |
| **PDCP** | 2026.05.31 | Plan Decision Checkpoint |
| **TDCP** | 2027.01.31 | Technical Decision Checkpoint (delivery deadline) |
| **EDCP** | 2027.04.30 | Ecosystem Development Checkpoint (polish period) |

**Note**: All requirements must be delivered before TDCP; the period between TDCP and EDCP is reserved for ecosystem expansion and polish.

---

## 2. Roadmap Overview

> 📊 **Milestone Chart**: [milestone_timeline_en.html](milestone_timeline_en.html)

| Phase | Milestone | Time Window | Weeks | Core Objective |
|-------|-----------|-------------|-------|----------------|
| **Phase 1** | Core Spike & Foundation | 07.02 → 08.28 | 8 weeks | Architecture review + test infrastructure + basic syntax parsing |
| **Phase 2** | Single-Module Compilation | 08.28 → 10.23 | 8 weeks | Single HAP + HelloWorld debug package compilation |
| **Phase 3** | Complex Project Compilation | 10.23 → 12.18 | 8 weeks | Multi-HAP + multi-HSP complex project compilation |
| **Phase 4** | Performance Tuning & Delivery | 12.18 → 01.15 | 4 weeks | Performance optimization + TDCP delivery |

---

## 3. Task Inventory (11 Items)

### Syntax Parsing
| Task | Details |
|------|---------|
| **Syntax Specification** | Define ArkTS (including ArkUI) syntax specification; implement Schema-driven multi-platform syntax parser generation |

### Dependency Resolution
| Task | Details |
|------|---------|
| **Module Import** | Support HarmonyOS SDK initialization and global object injection; implement import resolution for @kit, @ohos system modules, Native (so packages), and third-party packages |

### Semantic Checking
| Task | Details | Coverage |
|------|---------|----------|
| **ArkTS High-Performance Syntax Check** | 9 technical spikes | 9 categories / 113 total rules |
| **ArkUI Decorator & Component Syntax Check** | 8 technical spikes | 8 categories / 135 total rules |
| **HarmonyOS API Check** | API annotation checking: @since, @deprecated, @syscap, etc. | — |

### Syntax Transformation
| Task | Details |
|------|---------|
| **ArkUI Syntax Transformation** | ArkUI component, component extension, decorator, and state management syntax transformation |
| **Import Statement Transformation** | Ohmurl, TypeOnly, variable spreading, and other import syntax transformation |

### Project Orchestration
| Task | Details |
|------|---------|
| **HarmonyOS Project Orchestration** | Cross-module SDK and dependency information exchange; build module dependency graph; orchestrate compilation tasks based on dependency graph |

### Compilation Optimization
| Task | Details |
|------|---------|
| **Incremental Compilation** | File-level incremental compilation: VFS-based file version and dependency tracking; Hash-based change detection to skip unchanged files |
| **Parallel Compilation** | File-level parallel compilation: Thread pool management for parallelized parsing and file checking |

### Infrastructure
| Task | Details |
|------|---------|
| **Unified Utilities** | Path conversion, file I/O, inter-process communication, etc. |

---

## 4. Phase Breakdown

### Phase 1: Core Spike & Foundation (8 weeks: 07.02 → 08.28)

**Phase Objectives**:
1. **Core Architecture Spike & Review** (within 4 weeks)
   - Process & thread pool scheduling — Liu Yuan
   - Rollup necessity assessment — Liu Yuan
   - Plugin ecosystem compatibility plan — Liu Yuan
   - etsbundle repo capability sink (API-level → pipeline) — Yuhan, Ye Nan: complete module audit by 07.09
   - ArkTS Linter + Checker unification — Yanchen
   - UI transformation & checking separation — Ye Nan

2. **Compatibility Assurance — Test Infrastructure** (4 weeks)
   - Test framework design — Yuhan (create an SR tracking framework + one SR tracking case)
   - E2E project test cases — Wenpeng (set up private repo, build end-to-end test cases)
   - UT syntax Spec test cases (300 cases) — Ye Nan
   - Linter check test cases (300+ cases) — Yanchen

3. **TS-GO Foundation**
   - Basic syntax parsing
   - ArkTS syntax tree parsing
   - Related unit test case development

**Deliverables**:
- Architecture review report
- Test case suite
- Basic syntax parser + UT

---

### Phase 2: Single-Module Compilation (8 weeks: 08.28 → 10.23)

**Phase Objectives**:
1. **Single HAP Compilation**
   - Single-module ArkTS project compilation
   - HelloWorld sample project validation

2. **Debug Package Generation**
   - HAP artifact generation
   - Debug symbol support

3. **Basic Dependency Resolution**
   - @kit system module import
   - Third-party package import resolution

**Deliverables**:
- Single HAP compilation capability
- HelloWorld debug package
- Basic dependency resolution module

---

### Phase 3: Complex Project Compilation (8 weeks: 10.23 → 12.18)

**Phase Objectives**:
1. **Multi-HAP Compilation**
   - Multi-module project compilation
   - Inter-module dependency resolution

2. **Multi-HSP Compilation**
   - Shared package compilation
   - Cross-module dependency graph construction

3. **Checked HAP Artifact Generation**
   - Syntax checking integration
   - Artifact consistency validation

**Deliverables**:
- Multi-HAP + multi-HSP compilation capability
- Module dependency graph
- Check results + HAP artifacts

---

### Phase 4: Performance Tuning & Delivery (4 weeks: 12.18 → 01.15)

**Phase Objectives**:
1. **Performance Tuning**
   - Full compilation performance optimization
   - Incremental compilation performance optimization
   - Memory usage optimization

2. **TDCP Delivery Preparation**
   - Documentation completion
   - Test reports
   - Delivery review

**Deliverables**:
- Performance optimization report
- Complete test report
- TDCP delivery package

---

## 5. Phase Gates

| Gate | Date | Entry Criteria | Exit Criteria | Rollback Trigger |
|------|------|----------------|---------------|-----------------|
| **Gate P1→P2** | 08.28 | Architecture review passed | Basic syntax parsing complete + UT coverage | Review not passed |
| **Gate P2→P3** | 10.23 | Single HAP compilation succeeds | HelloWorld debug package generated | Compilation failure or artifact inconsistency |
| **Gate P3→P4** | 12.18 | Multi-module compilation succeeds | Complex project HAP artifact generated | Dependency resolution failure or performance regression |
| **Gate P4→TDCP** | 01.15 | Performance targets met | Delivery review passed | Performance targets missed or blocking bugs |

---

## 6. Success Metrics (KPI)

| Metric                       | Baseline (Current) | Target (TDCP) | Improvement | Measurement            |
| ---------------------------- | ------------------ | ------------- | ----------- | ---------------------- |
| Full Compilation Time        | ~16 min            | ≤ 5 min       | **3.2x**    | CI pipeline statistics |
| Incremental Compilation Time | ~6 min             | ≤ 2 min       | **3x**      | IDE response time      |
| Peak Memory                  | 20-30 GB           | ≤ 10 GB       | **50-67%↓** | System monitoring      |
| ArkTS Syntax Coverage        | 0%                 | 10%           | —           | Syntax check tests     |
| Artifact Consistency         | —                  | 100%          | —           | Binary diff            |

---

## 7. Resource Allocation

| Phase | Roles | Headcount | Effort (person-months) |
|-------|-------|-----------|----------------------|
| **Phase 1** | Architect + Developers | 3 | 6 |
| **Phase 2** | Developers + QA | 3 | 6 |
| **Phase 3** | Developers + QA | 4 | 8 |
| **Phase 4** | All hands | 4 | 4 |
| **Total** | | | **~24 person-months** |

---

## 8. Risks & Mitigations

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|-----------|
| ArkTS syntax adaptation complexity exceeds expectations | High | High | Complete Gap analysis in Phase 1; batch adaptation by priority |
| Multi-module dependency resolution difficulties | Medium | High | Reference existing Hvigor dependency graph implementation; progressive replacement |
| Performance tuning results below expectations | Medium | Medium | Buffer time reserved in Phase 4; adjust targets if necessary |
| Insufficient test case coverage | Medium | Medium | Invest in test infrastructure in Phase 1; continuous supplementation |

---

## Related Documents

- [[8-tsgo迁移方案设计/tsgo-migration-design]] — Complete migration design
- [[日常/2026-07-02]] — Project requirements document
- [[方案分析/4-ArkTS鸿蒙适配]] — ArkTS adaptation analysis
- [[方案分析/6-tsgo编译管道]] — Compilation pipeline deep dive
