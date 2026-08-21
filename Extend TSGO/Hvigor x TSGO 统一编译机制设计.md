
> **日期：2026-08-13** **状态：设计评审阶段**

---

## 一、现状分析

### 1.1 当前 ArkTS 编译架构

```
 每个 Module 独立编译：
 
   CompileArkTS(moduleA)                    CompileArkTS(moduleB)
     |                                        |
     +- initDefaultArkCompileConfig()         +- initDefaultArkCompileConfig()
     |  -> 生成 ProjectConfig A               |  -> 生成 ProjectConfig B
     |                                        |
     +- submitArkCompileWork(configA)         +- submitArkCompileWork(configB)
     |  -> workerPool.submit -> worker_thread |  -> workerPool.submit -> worker_thread
     |                                        |
     +- Worker 内:                            +- Worker 内:
        runArkPack(configA)                      runArkPack(configB)
          +- generateConfig()                     +- generateConfig()
          |  -> rollup plugins[]                  |  -> rollup plugins[]
          +- runHvigorCompile()                   +- runHvigorCompile()
          |  -> rollup.build()                    |  -> rollup.build()
          |    +- resolveId                       |    +- resolveId
          |    +- load (ets-loader)               |    +- load (ets-loader)
          |    +- transform (tsc/etsTransform)    |    +- transform (tsc/etsTransform)
          |    +- generateBundle (genAbc)         |    +- generateBundle (genAbc)
          |    +- writeBundle                     |    +- writeBundle
          +- 返回 CompileEvent[]                  +- 返回 CompileEvent[]
```

### 1.2 核心瓶颈

|**问题**|**根因**|**影响**|
|---|---|---|
|**N 次独立 tsc 实例**|**每个 worker 独立加载**`runArkPack`->`runHvigorCompile`-> rollup -> tsc|**类型解析重复、LanguageService 无法跨模块复用**|
|**模块间类型重复解析**|**各 worker 独立解析 HAR 依赖的**`.d.ts`|**同一份**`.d.ts`被 N 个 worker 重复解析|
|**worker 冷启动开销**|**每个 worker 需加载**`@ohos/hvigor-arkts-compose`+ SDK 插件|**daemon 模式下虽有 resident worker 缓解，但首次构建仍慢**|

### 1.3 关键代码路径

```
 CompileArkTS 任务执行链：
 
 AbstractArkCompile.doTaskAction()          abstract-ark-compile.ts:443
   +- initDefaultArkCompileConfig()         -> 生成单个模块的 ProjectConfig
   +- worker-id-helper.getWorkerIdWithModule()  -> 模块->worker 亲和性
   +- submitArkCompileWork()                run-ark.ts:35
        +- workerPool.submit(task, './job.js/run', { workInput: config })
             +- Worker 内:
                job.ts:28  run(config)
                  +- runArkPack(config, log, ...)    arkts-pack.ts:207
                       +- generateConfig()            -> rollup config + plugins
                       +- runHvigorCompile(config)    -> rollup.build()
```

### 1.4 现有可复用的基础设施

|**设施**|**位置**|**对 TSGO 的价值**|
|---|---|---|
|`TaskScheduleOptimizationService`|`task-schedule-optimization.ts:12`|**已遍历所有**`CompileArkTS`任务及依赖链，是模块清单的天然来源|
|`hvigorCore.getCompileDependArr()`|`hvigor-core.ts:99`|**已收集所有编译任务 path 的 Set**|
|`GlobalProjectDataService.compileProjectModel`|`global-project-data-service.ts:207`|**已有全模块映射表**`Record<sourceRoot, ModuleInfoType>`|
|`BuildModeManager.targetBuildOptionMap`|`build-mode-manager.ts:59`|**已有每个 module x target 合并后的**`BuildOpt`|
|`WorkerPool`+`pendingPromises`|`worker-pool-impl.ts`+`task-runner.ts:91`|**支持单 work 提交 + 多任务等待同一 promise**|
|`projectTaskDag`|`task-directed-acyclic-graph.ts:181`|**全局 DAG，可查询任务依赖关系**|

---

## 二、TSGO 交互机制设计

### 2.1 设计目标

1. **统一编译**：hvigor 收集所有待编译模块的配置，一次性传给 TSGO
2. **开关控制**：通过 `coreParameter.properties` 开关启用，默认关闭，保留原路径
3. **DAG 不破坏**：不改变现有任务依赖结构，下游任务无感知
4. **结果分发**：TSGO 返回的结果按模块分发回各自的 `CompileArkTS` 任务
5. **增量兼容**：保留模块级增量跳过能力

### 2.2 核心设计：Shared Barrier Pattern（共享屏障 + 批量提交）

```
 TSGO 模式下的执行流：
 
   CompileArkTS(A)                CompileArkTS(B)                CompileArkTS(C)
     |                              |                              |
     +- initDefaultArkCompileConfig +- initDefaultArkCompileConfig +- initDefaultArkCompileConfig
     |  -> ProjectConfig A          |  -> ProjectConfig B          |  -> ProjectConfig C
     |                              |                              |
     +- coordinator.register(A)     +- coordinator.register(B)     +- coordinator.register(C)
          |                              |                              |
          |  +--------------------------+------------------------------+
          |  |
          |  v  当 registeredCount === expectedCount 时：
          |  coordinator.submitBatchWork()
          |  -> workerPool.submit(1个work, workInput: { configs: [A,B,C] })
          |       |
          |       +- Worker 内: tsgoRunner.run(configs)
          |            -> TSGO 统一编译所有模块
          |            -> 返回 Map<moduleName, ModuleCompileResult>
          |
          v  所有 CompileArkTS 任务通过 pendingPromises 机制等待同一个 batchPromise：
          (不在 doTaskAction 中 await，而是 pendingPromises.add(processingPromise))
          task-runner 检测到 hasPendingPromises → handleTaskPendingPromises
          → Promise.all(pendingPromises).then(onTaskFinished) → 解锁下游
```

**设计要点**：

- **无新任务**：不需要在 DAG 中插入新的 `BatchCompileArkTS` 任务，避免 DAG 重构
- **CompileArkTS 变为"配置收集器 + 结果消费者"**：不再独立 submit worker，而是注册配置 + 等待批量结果
- **最后一个注册者触发提交**：当注册数达到 `expectedCount` 时，由最后一个执行的 `CompileArkTS` 任务触发批量提交
- **共享 Promise 屏障（通过 pendingPromises）**：所有 `CompileArkTS` 任务将结果处理 promise 注册到 `this.pendingPromises`，TSGO 完成后统一解锁。**不能在 `doTaskAction` 中 `await` batchPromise**——否则会阻塞 task-runner 的递归调度，导致后续 `CompileArkTS` 任务无法被 pop，造成死锁

> **关键约束：doTaskAction 不能 await 长时间未 resolve 的 promise**
> 
> **task-runner 的执行链是** `runTaskFromQueue → await executeOneTask → await doTaskAction`，** **如果 `doTaskAction` 中 `await` 了 `batchPromise`（未 resolve），整个 task-runner 会挂起，** **后续 `CompileArkTS` 任务永远不会被调度 → `expectedCount` 永远达不到 → **死锁**。
> 
> **原机制中** `workerPool.submit` 是同步提交：往 `task.pendingPromises` 塞一个 promise 但不 await，** **`doTaskAction` 立即返回，task-runner 继续调度下一个任务。** **TSGO 模式必须复用同样的模式：`this.pendingPromises.add(processingPromise)` 而非 `await`。

### 2.2.1 task-runner 执行机制与死锁规避

**task-runner 的串行递归调度链**（`task-runner.ts:30-89`）：

```
 runTaskFromQueue()
   ├─ popRunnableTask()                    // 从就绪队列 pop 一个任务
   ├─ await executeOneTask(task)           // ← AWAIT：等待 doTaskAction 完成
   │    └─ await coreTask.execute()
   │         └─ await this.doTaskAction()
   │
   ├─ hasPendingPromises = task.pendingPromises.get().length > 0
   ├─ if (hasPendingPromises)
   │    handleTaskPendingPromises(task)    // 注册异步链，不阻塞
   └─ await runTaskFromQueue()             // ← 递归调度下一个任务
```

**原机制为什么不会卡住**：

`workerPool.submit` (`worker-pool-impl.ts:270-290`) 是**同步提交**：

```
 // workerPool.submit 内部：
 const promise = new Promise((resolve, reject) => {
   tcb.on(WORK_DONE, () => resolve(true));   // worker 完成时才 resolve
 });
 coreTask.pendingPromises.add(promise);       // ← 塞入 pendingPromises，不 await
 return tcb;                                  // ← 同步返回
```

`doTaskAction` 立即返回 → `executeOneTask` 立即返回 → `runTaskFromQueue` 检测到 `hasPendingPromises=true` → 调用 `handleTaskPendingPromises`（注册 `Promise.all().then(onTaskFinished)` 异步链）→ 继续递归 `runTaskFromQueue` 调度下一个任务。

**TSGO 方案必须复用此模式**：

```
 // 正确写法：
 this.pendingPromises.add(processingPromise);  // add，不 await
 // doTaskAction 立即返回
 
 // 错误写法（会导致死锁）：
 const myResult = await myResultPromise;  // batchPromise 未 resolve → task-runner 挂起 → 死锁
```

**修正后的完整执行流**：

```
 runTaskFromQueue()
   ├─ pop CompileArkTS(A)
   │   doTaskAction: registerConfig(A) → pendingPromises.add(promiseA) → 返回
   ├─ hasPendingPromises=true → handleTaskPendingPromises(A)
   ├─ runTaskFromQueue()
   │   ├─ pop CompileArkTS(B)
   │   │   doTaskAction: registerConfig(B) → pendingPromises.add(promiseB) → 返回
   │   ├─ handleTaskPendingPromises(B)
   │   ├─ runTaskFromQueue()
   │   │   ├─ pop CompileArkTS(C)  ← 最后一个
   │   │   │   registerConfig(C): count===expected → submitBatchWork() → 触发 TSGO
   │   │   │   pendingPromises.add(promiseC) → 返回
   │   │   ├─ handleTaskPendingPromises(C)
   │   │   └─ runTaskFromQueue() ← 无更多任务，返回
   │   │
   │   │  TSGO 编译中，所有 pendingPromises 未 resolve ...
   │   │
   │   └─ TSGO 完成 → batchPromise resolve
   │       ├─ promiseA resolve → Promise.all([promiseA]).then(onTaskFinished(A)) → 解锁 A 下游
   │       ├─ promiseB resolve → Promise.all([promiseB]).then(onTaskFinished(B)) → 解锁 B 下游
   │       └─ promiseC resolve → Promise.all([promiseC]).then(onTaskFinished(C)) → 解锁 C 下游
```

**DAG 依赖验证**：

**此方案成立的前提是所有模块的** `CompileArkTS` 任务之间**没有 DAG 依赖**，能同时出现在 `runnableTaskQueue` 中。

**从** `arkts-task-initializer.ts:78-85`：

```
 declareDepends = (): string[] => {
   return [GenerateLoaderJson.name, CompileResource.name]
     .concat(getBuildProfileDependence(this.targetService));
 }
```

`getBuildProfileDependence` (`arkts-task-initializer.ts:220-231`) 的跨模块依赖是 `CREATE_HAR_BUILD_PROFILE`（生成 `BuildProfile.ets`），**不是** HAR 的 `CompileArkTS`。

**因此 HAP 的** `CompileArkTS` 不依赖 HAR 的 `CompileArkTS`，所有模块的 `CompileArkTS` 任务可以同时就绪。

### 2.3 详细组件设计

#### 2.3.1 TsgoBatchCompileCoordinator（核心协调器）

```
 // 位置: hvigor-ohos-plugin/src/tasks/worker/tsgo-batch-coordinator.ts
 
 export class TsgoBatchCompileCoordinator {
   private static _instance: TsgoBatchCompileCoordinator;
 
   // 预期注册数（来自 TaskScheduleOptimizationService 或 DAG 遍历）
   private _expectedCount: number = 0;
   // 已注册的模块配置
   private _configs: ProjectConfig[] = [];
   // 已注册的模块名（用于结果分发）
   private _moduleNames: string[] = [];
   // 批量编译的共享 Promise
   private _batchPromise: Promise<Map<string, ModuleCompileResult>> | null = null;
   private _batchResolve!: (result: Map<string, ModuleCompileResult>) => void;
   private _batchReject!: (error: Error) => void;
   // 是否已提交（防止重复提交）
   private _submitted: boolean = false;
 
   static getInstance(): TsgoBatchCompileCoordinator { ... }
 
   /** 初始化：从 DAG 中统计 CompileArkTS 任务数量 */
   init(expectedCount: number): void {
     this._expectedCount = expectedCount;
     this._configs = [];
     this._moduleNames = [];
     this._submitted = false;
     this._batchPromise = new Promise((resolve, reject) => {
       this._batchResolve = resolve;
       this._batchReject = reject;
     });
   }
 
   /** 注册模块配置，返回该模块的编译结果 Promise */
   registerConfig(config: ProjectConfig): Promise<ModuleCompileResult> {
     this._configs.push(config);
     this._moduleNames.push(config.moduleName);
 
     // 最后一个注册者触发批量提交
     if (this._configs.length === this._expectedCount && !this._submitted) {
       this._submitted = true;
       this.submitBatchWork();
     }
 
     // 返回该模块的结果 Promise
     return this._batchPromise.then(batchResult => batchResult.get(config.moduleName)!);
   }
 
   /** 提交批量编译 work 到 worker pool */
   private async submitBatchWork(): Promise<void> {
     const workerPool = WorkerPoolFactory.getDefaultWorkerPool();
     const workInput = {
       configs: this._configs,
       // 全局信息（所有模块共享）
       projectArkOption: this._configs[0]?.projectArkOption,
       sdkPath: this._configs[0]?.sdkPath,
       // ...其他全局配置
     };
 
     // 提交到固定 worker（锁定 workerId=0，避免 TSGO warmup 成本）
     workerPool.submit(null, './tsgo-job.js/run', {
       workInput,
       callback: (result: BatchCompileResult | Error) => {
         if (result instanceof Error) {
           this._batchReject(result);
         } else {
           this._batchResolve(result.moduleResults);
         }
       },
       targetWorkers: [0],  // 固定 worker
       useReturnVal: true,
       memorySensitive: true,
       preludeDeps: ['@ohos/hvigor', '@ohos/hvigor-arkts-compose'],
     });
   }
 
   getBatchPromise(): Promise<Map<string, ModuleCompileResult>> {
     return this._batchPromise!;
   }
 
   reset(): void { /* 清理状态，供下次构建使用 */ }
 }
```

#### 2.3.2 CompileArkTS 任务改造（开关控制）

```
 // abstract-ark-compile.ts -- doTaskAction 改造
 
 protected async doTaskAction(): Promise<void> {
   this.validateModuleJsonAbstract();
   const config: ProjectConfig = await this.initDefaultArkCompileConfig();
 
   // ===== TSGO 开关分支 =====
   if (coreParameter.properties.ohosTsgoCompileEnabled) {
     await this.doTsgoBatchCompile(config);
     return;  // 不走原路径
   }
 
   // ===== 原路径（保持不变）=====
   // ... 原有 submitArkCompileWork 逻辑 ...
 }
 
 /** TSGO 批量编译路径 */
 private async doTsgoBatchCompile(config: ProjectConfig): Promise<void> {
   const coordinator = TsgoBatchCompileCoordinator.getInstance();
 
   // 1. 注册配置，获取该模块的结果 Promise（不 await！）
   const myResultPromise = coordinator.registerConfig(config);
 
   // 2. 将结果处理注册为 pending promise（与 workerPool.submit 的机制一致）
   //    不能 await myResultPromise —— 否则会阻塞 task-runner 递归调度，导致死锁
   const processingPromise = myResultPromise.then(result => {
     this.handleCompileEvents(result.compileEvents, this.durationEvent, false);
     return this.writeBuildMode(config);
   });
   this.pendingPromises.add(processingPromise);
 
   // 3. doTaskAction 立即返回，task-runner 继续调度下一个 CompileArkTS 任务
   //    task-runner 检测到 hasPendingPromises=true → handleTaskPendingPromises
   //    → Promise.all(pendingPromises).then(onTaskFinished) → 解锁下游
 
   // 4. Widget 编译（仍可走批量或独立处理）
   if (this.needSubmitArkTsWidget) {
     // widget 配置也注册到 coordinator
   }
 }
```

#### 2.3.3 TSGO Worker 入口脚本

```
 // 位置: hvigor-ohos-plugin/src/tasks/worker/tsgo-job.ts
 
 export const run = async (
   input: { configs: ProjectConfig[] },
   cacheStoreManager?: CacheStoreManager
 ): Promise<BatchCompileResult | Error> => {
   const { configs } = input;
 
   try {
     // 构建 TSGO 统一编译参数
     const tsgoOptions: TsgoCompileOptions = {
       modules: configs.map(config => ({
         moduleName: config.moduleName,
         modulePath: config.modulePath,
         sourceRoot: config.projectPath,
         entryObj: getCompileInputObject(config),
         outputDir: config.aceModuleBuild,
         buildMode: config.buildMode,
         arkOptions: config.arkOptions,
         tscConfig: config.arkOptions?.tscConfig,
         externalApiPaths: config.externalApiPaths,
         entryModules: config.entryModules,
         // ... 其他 per-module 配置
       })),
       // 全局配置
       sdkPath: configs[0].sdkPath,
       compileProjectModel: GlobalProjectDataService.getInstance().getCompileProjectModel(),
       cachePath: configs[0].cachePath,
       // ... 其他全局配置
     };
 
     // 调用 TSGO 统一编译
     const tsgoResult = await runTsgoCompile(tsgoOptions);
 
     // 转换为 per-module 结果 Map
     const moduleResults = new Map<string, ModuleCompileResult>();
     for (const moduleResult of tsgoResult.moduleResults) {
       moduleResults.set(moduleResult.moduleName, moduleResult);
     }
 
     return { moduleResults, isSuccess: true };
   } catch (error) {
     return error as Error;
   }
 };
```

#### 2.3.4 TSGO 侧适配接口契约

```
 // hvigor -> TSGO 的数据契约
 
 interface TsgoCompileOptions {
   /** 所有待编译模块的配置 */
   modules: TsgoModuleConfig[];
   /** SDK 路径 */
   sdkPath: string;
   /** 全局模块映射表（sourceRoot -> ModuleInfo） */
   compileProjectModel: ArkCompileProjectModelType;
   /** 缓存路径 */
   cachePath: string;
   /** 构建模式 */
   buildMode: string;
 }
 
 interface TsgoModuleConfig {
   moduleName: string;
   modulePath: string;
   sourceRoot: string;
   /** 编译入口（rollup input 格式或 TSGO 自定义格式） */
   entryObj: Record<string, string>;
   /** 输出目录 */
   outputDir: string;
   /** ArkTS 编译选项（对应 BuildOpt.arkOptions） */
   arkOptions: ArkOptions;
   /** TSC 配置 */
   tscConfig?: TscConfig;
   /** 外部 API 路径（依赖的 HAR 的 .d.ts） */
   externalApiPaths: string[];
   /** 入口模块列表 */
   entryModules: string[];
   /** 混淆选项 */
   obfuscationOptions?: ObfuscationOptions;
   /** 模块路径信息 */
   aceModuleRoot: string;
   aceModuleBuild: string;
   /** 源码映射路径 */
   sourceMapPath?: string;
   /** 是否 widget 编译 */
   widgetCompile?: string;
 }
 
 // TSGO -> hvigor 的返回契约
 
 interface BatchCompileResult {
   moduleResults: Map<string, ModuleCompileResult>;
   isSuccess: boolean;
   error?: Error;
 }
 
 interface ModuleCompileResult {
   moduleName: string;
   compileEvents: CompileEvent[];
   compileLogEvents: LogEvent[];
   isSuccess: boolean;
   error?: Error;
 }
```

---

## 三、初始化时机与模块计数

```
 构建生命周期：
 
   init()                          -- Project/Module 模型创建
   configuration()                 -- hvigorfile 评估、插件绑定、任务注册
     +- nodesEvaluated hook        -- 所有模块配置就绪
   buildTaskGraph()                -- DAG 构建
     +- taskGraphResolved hook     -- DAG 就绪，可以统计 CompileArkTS 任务数
       +- [TSGO 初始化点]
           TsgoBatchCompileCoordinator.getInstance().init(
             countCompileArkTsTasks(projectTaskDag)
           )
   execute()                       -- 任务执行
     +- CompileArkTS(A/B/C...)     -- 注册配置 + 等待批量结果
```

**模块计数实现**：

```
 // 在 taskGraphResolved hook 中注册
 function initTsgoCoordinator(): void {
   if (!coreParameter.properties.ohosTsgoCompileEnabled) return;
 
   const compileArkTsTasks = hvigorCore.getCompileDependArr();
   let count = 0;
   for (const taskPath of compileArkTsTasks) {
     if (taskPath.includes('CompileArkTS')) {
       count++;
     }
   }
   TsgoBatchCompileCoordinator.getInstance().init(count);
 }
```

---

## 四、增量编译适配

```
 增量场景处理：
 
 1. 模块级增量跳过（保留）：
    - IncrementalTask 的 declareInputs/declareOutputs 不变
    - UP-TO-DATE 的模块不执行 doTaskAction -> 不注册配置
    - 需要调整 expectedCount：只统计实际会执行的 CompileArkTS 任务数
 
 2. TSGO 内部增量（TSGO 侧适配）：
    - TSGO 接收全量模块列表，但内部可基于文件 hash 跳过未变更模块
    - TSGO 的 LanguageService 天然支持增量编译（跨模块缓存复用）
 
 3. expectedCount 动态调整：
    - 在 taskGraphResolved 后、execute 前，遍历 DAG 中 CompileArkTS 任务的 isUpToDate 标志
    - 只统计 !isUpToDate 的任务数作为 expectedCount
```

---

## 五、错误处理策略

```
 错误传播机制：
 
   TSGO 编译失败
     |
     +- 单模块失败：
     |  coordinator._batchReject(error)
     |  -> 所有 await batchPromise 的 CompileArkTS 任务 reject
     |  -> task-runner 的 onTaskFailed 触发
     |  -> 构建终止
     |
     +- 全局失败（如 TSGO 进程崩溃）：
        worker-action 的 WORK_ERROR 事件
        -> TCB callback 触发 reject
        -> 同上传播
```

**优化**：TSGO 应尽可能返回 per-module 的错误，而非整体失败。即使模块 A 编译失败，模块 B 的结果仍可用（取决于业务策略）。

---

## 六、Spinner / 日志适配

```
 Spinner 行为（TSGO 模式）：
 
   方案A（推荐）：统一 Spinner
     - 第一个 CompileArkTS 启动 "TSGO Batch Compiling..." spinner
     - TSGO 内部进度回调更新 spinner 状态
     - 批量完成后 stop spinner
 
   方案B：保留 per-module spinner
     - 每个 CompileArkTS 启动自己的 spinner
     - 但在 await batchPromise 期间 spinner 处于 "waiting" 状态
     - 批量完成后依次 stop
```

---

## 七、改造影响面与兼容性

### 7.1 改动清单

|**文件**|**改动类型**|**说明**|
|---|---|---|
|`abstract-ark-compile.ts:443`|**修改**doTaskAction|**新增 TSGO 分支，开关控制**|
|`tsgo-batch-coordinator.ts`|**新增**|**核心协调器单例**|
|`tsgo-job.ts`|**新增**|**Worker 入口脚本**|
|`run-ark.ts`|**不改**|**保留原 submitArkCompileWork**|
|`job.ts`|**不改**|**保留原 run 函数**|
|`worker-id-helper.ts`|**不改**|**TSGO 模式不使用，但不删除**|
|`global-core-parameters.ts:124`|**修改**|**新增**`ohosTsgoCompileEnabled`属性|
|`task-schedule-optimization.ts`|**不改**|**复用现有 compileDependArr**|
|`arkts-pack.ts`|**不改**|**保留原 runArkPack**|
|**DAG / TaskControlCenter**|**不改**|**无 DAG 结构变更**|

### 7.2 兼容性保证（遵循 AGENTS.md 规范）

1. **不删除已有逻辑**：`submitArkCompileWork`、`runArkPack`、`worker-id-helper` 全部保留
2. **不修改默认行为**：`ohosTsgoCompileEnabled` 默认 `false`，走原路径
3. **开关控制**：通过 `coreParameter.properties.ohosTsgoCompileEnabled` 控制
4. **保留旧接口**：`AbstractArkCompile.doTaskAction` 原路径完整保留

### 7.3 TSGO 侧需要适配的内容

|**适配项**|**说明**|
|---|---|
|**接收批量模块配置**|**接收**`TsgoModuleConfig[]`而非单个`ProjectConfig`|
|**统一 LanguageService**|**单个 TSGO 进程内为所有模块共享 LanguageService（跨模块类型解析复用）**|
|**per-module 产物输出**|**按模块分别输出 ABC 字节码、sourceMap 到各自**`aceModuleBuild`|
|**per-module 结果返回**|**返回**`Map<moduleName, ModuleCompileResult>`|
|**进度回调**|**可选：通过**`parentPort.postMessage`回报编译进度|
|**缓存适配**|**接收 hvigor 的**`cacheStoreManager`，复用 daemon 模式的持久化缓存|

---

## 八、性能收益预估

```
 场景：10 个模块的工程（5 HAR + 1 HAP + 1 HSP + 3 依赖库）
 
 当前模式（N 次独立编译）：
   +- Worker 冷启动 x 6（HAR 先编译，HAP/HSP 后编译）
   +- tsc LanguageService 初始化 x 6
   +- SDK .d.ts 解析 x 6（每个 worker 独立解析同一份 SDK 声明）
   +- rollup 插件链初始化 x 6
   +- 实际并行度：~2-3（受 DAG 依赖限制）
 
 TSGO 模式（1 次统一编译）：
   +- Worker 冷启动 x 1
   +- TSGO LanguageService 初始化 x 1（跨模块共享）
   +- SDK .d.ts 解析 x 1（全局缓存）
   +- TSGO 内部并行（Go runtime 天然多核）
   +- 无 DAG 依赖等待（所有模块统一编译）
 
 预估收益：
   +- 首次构建：提速 40-60%（主要来自 tsc 初始化和类型解析去重）
   +- 增量构建：提速 20-30%（TSGO 增量编译 + 跨模块缓存）
   +- daemon 模式：提速 30-50%（TSGO 进程常驻 + LanguageService 复用）
```

---

## 九、实施路径建议

```
 Phase 1: 基础设施搭建
   +- 实现 TsgoBatchCompileCoordinator
   +- 实现 tsgo-job.ts worker 入口
   +- 新增 ohosTsgoCompileEnabled 开关
   +- 改造 AbstractArkCompile.doTaskAction 添加 TSGO 分支
 
 Phase 2: TSGO 适配
   +- 定义 TsgoCompileOptions / TsgoModuleConfig 接口
   +- TSGO 侧实现批量模块读取和统一编译
   +- TSGO 侧实现 per-module 产物输出
   +- TSGO 侧实现结果 Map 返回
 
 Phase 3: 集成验证
   +- 增量编译适配（expectedCount 动态调整）
   +- Widget 编译适配
   +- Spinner / 日志适配
   +- 错误处理验证
   +- 端到端测试
 
 Phase 4: 性能调优
   +- TSGO worker 常驻 + daemon 复用
   +- TSGO 内部并行度调优
   +- 缓存策略优化
   +- 性能对比基准测试
```

---

## 十、关键文件引用

|**关注点**|**文件路径**|**行号**|
|---|---|---|
|**CompileArkTS 任务执行入口**|`hvigor-ohos-plugin/src/tasks/abstract-ark-compile.ts`|**443**|
|**单模块 worker 提交**|`hvigor-ohos-plugin/src/tasks/worker/run-ark.ts`|**35**|
|**Worker 内编译执行**|`hvigor-ohos-plugin/src/tasks/worker/job.ts`|**28**|
|**runArkPack 入口**|`hvigor-arkts-compose/src/arkts-pack.ts`|**207**|
|**Rollup 插件链构建**|`hvigor-arkts-compose/src/arkts-pack.ts`|**470**|
|**编译任务注册**|`hvigor-ohos-plugin/src/tasks/manager/arkts-task-initializer.ts`|**33**|
|**编译依赖链收集**|`hvigor-ohos-plugin/src/plugin/hooks/task-schedule-optimization.ts`|**44**|
|**全局模块映射构建**|`hvigor-ohos-plugin/src/tasks/service/global-project-data-service.ts`|**207**|
|**buildOption 合并管理**|`hvigor-ohos-plugin/src/project/build-option/build-mode-manager.ts`|**58**|
|**ProjectConfig 定义**|`hvigor-arkts-compose/src/project-config/project-config.ts`|**24**|
|**模块->worker 亲和性**|`hvigor-ohos-plugin/src/tasks/worker/worker-id-helper.ts`|**6**|
|**Worker 池工厂**|`hvigor/src/base/internal/pool/worker-pool/worker-pool-factory.ts`|**26**|
|**全局参数**|`hvigor/src/base/internal/data/global-core-parameters.ts`|**124**|
|**任务串行调度**|`hvigor/src/base/internal/task/core/task-runner.ts`|**30**|
|**pendingPromises 机制**|`hvigor/src/base/internal/task/core/task-runner.ts`|**91**|

---

**文档版本**: 1.0** ****最后更新**: 2026-08-13****** 维护团队**: Hvigor 开发团队