tsgo Monorepo 编排逻辑梳理![md](data:image/svg+xml,%3csvg%20viewBox='0%200%2024%2024'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%20width='24.000000'%20height='24.000000'%20fill='none'%3e%3cdefs%3e%3cclipPath%20id='clipPath_1'%3e%3crect%20width='24.000000'%20height='22.000000'%20x='0.000000'%20y='1.000000'%20rx='2.000000'%20fill='rgb\(255,255,255\)'%20/%3e%3c/clipPath%3e%3c/defs%3e%3crect%20id='%E5%AE%B9%E5%99%A8%204212'%20width='24.000000'%20height='24.000000'%20x='0.000000'%20y='0.000000'%20fill='rgb\(255,255,255\)'%20fill-opacity='0'%20/%3e%3cg%20id='%E5%AE%B9%E5%99%A8%204208'%20clip-path='url\(%23clipPath_1\)'%20customFrame='url\(%23clipPath_1\)'%3e%3crect%20id='%E5%AE%B9%E5%99%A8%204208'%20width='24.000000'%20height='22.000000'%20x='0.000000'%20y='1.000000'%20rx='2.000000'%20fill='rgb\(255,255,255\)'%20/%3e%3crect%20id='%E5%AE%B9%E5%99%A8%204208'%20width='22.500000'%20height='20.500000'%20x='0.750000'%20y='1.750000'%20rx='1.250000'%20stroke='rgb\(104,104,104\)'%20stroke-width='1.5'%20/%3e%3cpath%20id=''%20d='M11.7139%208.0166L11.7139%2016.7998L10.0586%2016.7998L10.0586%2011.1367L7.82617%2014.459L6.91211%2014.459L4.6709%2011.1104L4.6709%2016.7998L3.02441%2016.7998L3.02441%208.0166L4.4043%208.0166L7.4043%2012.5635L10.3926%208.0166L11.7139%208.0166ZM16.4901%208.0166Q17.7733%208.0166%2018.8104%208.53223Q19.8505%209.04785%2020.4481%2010.0381Q21.0487%2011.0283%2021.0487%2012.3965Q21.0487%2013.7646%2020.4481%2014.7607Q19.8505%2015.7568%2018.8163%2016.2783Q17.785%2016.7998%2016.4901%2016.7998L13.5604%2016.7998L13.5604%208.0166L16.4901%208.0166ZM16.285%2015.3115Q17.2225%2015.3115%2017.8758%2014.9775Q18.5292%2014.6406%2018.8719%2013.9873Q19.2147%2013.3311%2019.2147%2012.3965Q19.2147%2011.4707%2018.8719%2010.8232Q18.5292%2010.1758%2017.8758%209.83594Q17.2225%209.49316%2016.285%209.49316L15.3006%209.49316L15.3006%2015.3115L16.285%2015.3115Z'%20fill='rgb\(104,104,104\)'%20fill-rule='nonzero'%20/%3e%3c/g%3e%3crect%20id='%E5%AE%B9%E5%99%A8%204208'%20width='22.500000'%20height='20.500000'%20x='0.750000'%20y='1.750000'%20rx='1.250000'%20stroke='rgb\(104,104,104\)'%20stroke-width='1.5'%20data-pixso-skip-parse='true'%20/%3e%3c/svg%3e)

2026-08-20 09:59 Create by fanglin 30015131 , last update by fanglin 30015131 at 2026-08-20 10:04

Content is questionable? Click me

### 内容由AI生成，仅供参考

[https://gitcode.com/GitHub_Trending/ty/typescript-go/tree/main/internal/execute/build](https://gitcode.com/GitHub_Trending/ty/typescript-go/tree/main/internal/execute/build)

> > 源码目录：`internal/execute/build`，对应 tsgo 的 `--build/-b`（Project References 项目引用）构建模式，等价于原版 `tsc‑‑build` 的 Go 重写版本，专门处理 monorepo 多子工程编排、依赖拓扑、增量、并行调度。  
> > tsgo monorepo 编排**完全基于 tsconfig 的 `references`**，不读取 package.json workspace，和 pnpm/turborepo 编排体系分离。

## 目录文件职责（internal/execute/build）

|文件|核心职责|
|---|---|
|`builder.go`|构建器主入口；解析项目图、拓扑排序、任务调度、增量状态判断|
|`project_graph.go`|构建项目依赖图：扫描所有 tsconfig references，构建有向图，检测循环依赖|
|`incremental.go`|`.tsbuildinfo` 增量状态读取/写入；文件指纹比对，判断子项目是否需要重编译|
|`worker.go`|Go goroutine 工作池；`--builders=N` 参数控制并发构建子项目数量|
|`diagnostics.go`|聚合多个子项目编译报错、错误传播|

## 整体编排总流程

### 关键概念区分 tsgo 的两组并发参数

1. `--builders N`：**monorepo 项目级并行**，控制同时跑多少个不同子 tsconfig 工程，对应 `internal/execute/build` worker 池，编排层；
2. `--checkers N`：**单个项目内部文件级并行**，一个子项目内部，多个源文件并行做类型检查，属于 compiler/checker 层，不属于 monorepo 编排逻辑。

> 组合效果：`--builders 4 --checkers 4`，最多同时 4 个子项目，每个子项目内部最多4个文件并行检查，理论最大并发16个checker任务。

## 1. 构建项目依赖图 ProjectGraph（project_graph.go）

### 输入

根 tsconfig 的 `references[]`，每一项 `{path: "../packages/xxx"}`，要求子项目必须开启 `composite:true`（和原版tsc约束一致）。

### 处理步骤

1. 递归解析每个引用路径，读取子项目 tsconfig；子项目内部又可以继续写 references，递归展开整张图；
2. 建立节点：每个节点代表一个 composite tsconfig 工程；
3. 建立边：A references B → A **依赖于 B**；B 必须先构建完成，A 才可以构建；
4. **循环依赖检测**：图中发现环直接抛出诊断，终止构建；monorepo 跨子包循环引用不允许；
5. 输出 DAG（有向无环图）。

> ⚠️注意：**不读取 package.json dependencies/workspace**。依赖关系只来自 tsconfig references，不是 npm/pnpm workspace。即便 package.json 写了 workspace:*，tsgo‑build 也感知不到，必须手动写 tsconfig references 声明依赖。

## 2. 拓扑排序确定构建顺序 builder.go

拿到 DAG 之后做拓扑排序：

- 入度为0的节点（没有依赖其他工程）优先；
- 某个节点的**全部依赖项目构建完成后，才允许调度该节点进入worker池**；
- 同一层级（互相无依赖）的多个项目可以并行执行，由 `--builders` 限制最大并发。

示例：monorepo 依赖关系

```
app → utils
app → ui
ui → utils
```

拓扑执行顺序：

1. 先构建 utils（无依赖）
2. utils完成后，并行构建 ui、app（ui依赖utils；app依赖utils/ui）

## 3. 增量构建判断 incremental.go

每个 composite 项目输出目录会生成 `.tsbuildinfo`，存储：

- 所有源文件、输入依赖项目的文件指纹 hash；
- 上一次编译选项快照。

对于图中每一个项目：

1. 如果不存在 `.tsbuildinfo` → 需要构建；
2. 如果 tsconfig 编译选项变更 → 需要构建；
3. 如果本项目源文件发生变更 → 需要构建；
4. 如果**该项目依赖的任意子项目发生过重新构建** → 当前项目必须重新构建（传递失效）；
5. 全部指纹匹配 → 直接跳过编译。

> 重点：传递失效。B被A引用，B重新构建，那么A即使自己文件没改动，也必须重新编译，保证类型引用一致性。这是 monorepo 编排的核心逻辑。

## 4. Worker池调度 worker.go

- 参数 `--builders` 设置 worker 数量，默认值由CPU核数推导；
- Go goroutine 实现工作池；任务队列存放待构建的项目节点；
- 约束：一个项目的所有前置依赖没有全部完成，不会调度该项目；
- 每个worker拿到一个项目任务，调用编译器执行该子项目完整编译（类型检查 + emit）；
- 子项目编译报错：会记录诊断；根据 continueOnError 配置，可选择终止整体构建或者继续构建其他不相关项目。

> 对比原版 Node‑tsc‑‑build：原版是单线程串行，tsgo利用goroutine实现真正项目粒度并行，这是tsgo monorepo最大性能提升点。

## 5. 诊断聚合与退出 diagnostics.go

- 每个子项目编译产生的 error/warning 全部收集；
- 全部任务结束后统一打印所有诊断，标注每个错误来自哪个子项目路径；
- 存在错误，进程返回非0退出码，CI场景可以正常识别失败。

## 6. 和传统工具对比（monorepo编排维度）

|工具|依赖来源|调度粒度|并行能力|增量来源|
|---|---|---|---|---|
|tsgo‑b|tsconfig references|tsconfig工程|项目级并行（--builders）+.单项目文件并行（--checkers）|.tsbuildinfo 文件指纹传递失效|
|原版tsc‑b|tsconfig references|tsconfig工程|单线程串行|.tsbuildinfo|
|turborepo/nx|package.json dependsOn|npm package|进程级并行|自定义cache hash|
|pnpm‑r run build|package.json dependencies|npm package|进程级，拓扑|无原生ts增量|

> 重要差异：
> 
> 1. tsgo 编排层只认 tsconfig references，**不读 package.json**；
> 2. tsgo 是编译器内建编排，不需要外部工具；可以做到编译粒度增量；turborepo是外部进程调度，只能缓存整个进程输出。

## 7. 典型monorepo使用示例

目录结构

```
monorepo/
├─ tsconfig.json（root solution，references指向packages/*）
└─ packages/
   ├─ utils/tsconfig.json composite:true
   ├─ ui/tsconfig.json composite:true references:["../utils"]
   └─ app/tsconfig.json composite:true references:["../utils","../ui"]
```

执行命令：

```bash
# monorepo全量构建，项目并行4，每个项目内部并行4个checker
tsgo -b --builders 4 --checkers 4
```

## 8. 限制与坑点

1. 必须每个子包设置 `composite:true`，否则 references 不生效；
2. 跨子包循环引用会直接报错；
3. references 必须手写，不会自动读取 package.json workspace 依赖；
4. `.tsbuildinfo` 是增量核心，删除该文件会强制全量重编；
5. watch模式下内部build编排逻辑复用同一套ProjectGraph，增量监听文件变更触发局部重构建。

## 补充：调用链路总栈

```
cmd/tsgo/main.go CLI入口
    → internal/execute.Execute()
        → internal/execute/build.Builder（--build模式）
            → BuildGraph() 构建project_graph
            → TopologicalSort 拓扑排序
            → worker pool调度
                → 单项目编译：internal/compiler 编译器执行
            → 聚合diagnostics输出
```
