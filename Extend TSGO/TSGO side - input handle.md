tsgo 内部编排API 封装方案：绕过 tsconfig，直接传入内存 Monorepo 数据![md](data:image/svg+xml,%3csvg%20viewBox='0%200%2024%2024'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%20width='24.000000'%20height='24.000000'%20fill='none'%3e%3cdefs%3e%3cclipPath%20id='clipPath_1'%3e%3crect%20width='24.000000'%20height='22.000000'%20x='0.000000'%20y='1.000000'%20rx='2.000000'%20fill='rgb\(255,255,255\)'%20/%3e%3c/clipPath%3e%3c/defs%3e%3crect%20id='%E5%AE%B9%E5%99%A8%204212'%20width='24.000000'%20height='24.000000'%20x='0.000000'%20y='0.000000'%20fill='rgb\(255,255,255\)'%20fill-opacity='0'%20/%3e%3cg%20id='%E5%AE%B9%E5%99%A8%204208'%20clip-path='url\(%23clipPath_1\)'%20customFrame='url\(%23clipPath_1\)'%3e%3crect%20id='%E5%AE%B9%E5%99%A8%204208'%20width='24.000000'%20height='22.000000'%20x='0.000000'%20y='1.000000'%20rx='2.000000'%20fill='rgb\(255,255,255\)'%20/%3e%3crect%20id='%E5%AE%B9%E5%99%A8%204208'%20width='22.500000'%20height='20.500000'%20x='0.750000'%20y='1.750000'%20rx='1.250000'%20stroke='rgb\(104,104,104\)'%20stroke-width='1.5'%20/%3e%3cpath%20id=''%20d='M11.7139%208.0166L11.7139%2016.7998L10.0586%2016.7998L10.0586%2011.1367L7.82617%2014.459L6.91211%2014.459L4.6709%2011.1104L4.6709%2016.7998L3.02441%2016.7998L3.02441%208.0166L4.4043%208.0166L7.4043%2012.5635L10.3926%208.0166L11.7139%208.0166ZM16.4901%208.0166Q17.7733%208.0166%2018.8104%208.53223Q19.8505%209.04785%2020.4481%2010.0381Q21.0487%2011.0283%2021.0487%2012.3965Q21.0487%2013.7646%2020.4481%2014.7607Q19.8505%2015.7568%2018.8163%2016.2783Q17.785%2016.7998%2016.4901%2016.7998L13.5604%2016.7998L13.5604%208.0166L16.4901%208.0166ZM16.285%2015.3115Q17.2225%2015.3115%2017.8758%2014.9775Q18.5292%2014.6406%2018.8719%2013.9873Q19.2147%2013.3311%2019.2147%2012.3965Q19.2147%2011.4707%2018.8719%2010.8232Q18.5292%2010.1758%2017.8758%209.83594Q17.2225%209.49316%2016.285%209.49316L15.3006%209.49316L15.3006%2015.3115L16.285%2015.3115Z'%20fill='rgb\(104,104,104\)'%20fill-rule='nonzero'%20/%3e%3c/g%3e%3crect%20id='%E5%AE%B9%E5%99%A8%204208'%20width='22.500000'%20height='20.500000'%20x='0.750000'%20y='1.750000'%20rx='1.250000'%20stroke='rgb\(104,104,104\)'%20stroke-width='1.5'%20data-pixso-skip-parse='true'%20/%3e%3c/svg%3e)

2026-08-20 10:22 Create by fanglin 30015131 , last update by fanglin 30015131 at 2026-08-20 10:22

Content is questionable? Click me

### 内容由 AI 生成，仅供参考

> 目标：**不修改原有 `internal/execute/build` 编排逻辑**，只做薄薄一层包装；不读磁盘 tsconfig.json，直接传入内存构造的 monorepo 工程描述，复用现有的 ProjectGraph、拓扑排序、worker 调度、增量、build 整套流程。
> 
> 核心认知：`typescript‑go` 的 build 模块**强依赖 tsconfig.SourceFile（解析后的配置对象）**，原版 ProjectGraph 是从磁盘读取 tsconfig 文件构建节点。我们要做：**内存构造虚拟 tsconfig 节点，喂给原有 build 体系，磁盘 IO 替换成内存构造**。

> ⚠️包层级注意：`internal/` 包是 Go internal，外部项目无法直接 import。两种使用形态：
> 
> 1. 在 tsgo 源码库内部新增一层封装（推荐，可以直接访问 internal）
> 2. 如果是外部项目：需要把要暴露的 API 提升到 `pkg/` 导出，否则编译报错。下面示例基于**在 typescript‑go 仓库内新增封装层**。

## 原有 build 核心类型回顾（internal/execute/build）

```go
// builder.go
type Builder struct {
	graph         *ProjectGraph
	workerPool    *WorkerPool
	// ...
}

// project_graph.go
// 图中每一个节点，代表一个 composite tsconfig 工程
type ProjectNode struct {
	Config *tsconfig.SourceFile // 【关键】这个就是解析后的tsconfig配置对象
	// 依赖边：依赖哪些其他 ProjectNode
	Dependencies []*ProjectNode
	// ...
}

type ProjectGraph struct {
	Roots []*ProjectNode
	Nodes map[string]*ProjectNode // key: config 文件路径标识
}
```

原版流程：磁盘读 tsconfig 文件 → 解析得到 `*tsconfig.SourceFile` → 构建 `ProjectNode` → 组装 `ProjectGraph` → 交给 Builder。

我们的包装层思路：

> 不再读磁盘 json，**内存构造一批虚拟 `tsconfig.SourceFile` 对象，手动构造 ProjectNode、组装 ProjectGraph**，直接喂给原有 Builder，后面拓扑、worker、增量全部复用原有逻辑，编排逻辑零改动。

> 注意：`.tsbuildinfo` 增量依然会读写磁盘；如果你想要全内存增量，还需要替换 `incremental.go` 的文件 IO，这里需求是简单包一层，保留原有增量行为。

---

# 1. 自定义输入数据结构（对外暴露的API入参）

自己定义一套简单的 Monorepo 内存描述，**和 tsconfig 解耦**，业务层只填这个结构体，不需要写任何 tsconfig 文件。

新建文件示例：`internal/execute/custombuild/api.go`

```go
package custombuild

import (
	"github.com/microsoft/typescript-go/internal/tsconfig"
	"github.com/microsoft/typescript-go/internal/execute/build"
)

// 对外：用户传入的 monorepo 工程描述，不需要任何磁盘tsconfig
type MonorepoProject struct {
	// 虚拟项目标识，用作ProjectGraph的key，可以不是真实磁盘路径
	ProjectID string

	// 本项目源文件列表（内存源文件，或者磁盘文件路径）
	SourceFiles []string

	// 编译输出目录
	OutDir string

	// 依赖的其他 ProjectID，对应上面 MonorepoProject.ProjectID
	DependsOn []string

	// 编译选项，等价tsconfig compilerOptions
	CompilerOptions tsconfig.CompilerOptions
}

// 对外顶层入参
type CustomMonorepoInput struct {
	Projects []*MonorepoProject

	// build 执行参数，等价命令行参数
	BuildOptions struct {
		Builders int // --builders
		Checkers int // --checkers
	}
}

// 对外API：入口函数，直接传入内存monorepo定义，复用原有build编排
func BuildMonorepo(input *CustomMonorepoInput) (*build.BuildResult, error) {
	// 1. 将自定义内存模型，转换为build模块需要的 *build.ProjectGraph
	graph, err := buildGraphFromInput(input)
	if err != nil {
		return nil, err
	}

	// 2. 复用原有Builder配置，完全复用原来的编排逻辑
	builderOpts := build.BuilderOptions{
		Builders: input.BuildOptions.Builders,
		Checkers: input.BuildOptions.Checkers,
		// 其他原有配置直接照搬tsgo CLI的默认值
		ContinueOnError: false,
	}

	b := build.NewBuilder(graph, &builderOpts)

	// 3. 执行整套build流程：拓扑排序、worker池、增量、编译、收集诊断
	result, err := b.Run()
	return result, err
}
```

## 2. 核心适配函数：把自定义内存结构构建出 `*build.ProjectGraph`

> 这就是薄薄的胶水层：**手动构造虚拟 tsconfig.SourceFile，构造 ProjectNode，建立依赖边，不调用磁盘读取tsconfig的逻辑**。
> 
> > 原版 `project_graph.go#BuildGraphFromConfigFile()` 会读磁盘；我们不走这个函数，自己内存构造整张图。

```go
// buildGraphFromInput 将自定义内存monorepo转为build模块需要的ProjectGraph
func buildGraphFromInput(input *CustomMonorepoInput) (*build.ProjectGraph, error) {
	// 1. 先创建全部 ProjectNode，每个节点绑定一个内存虚拟 tsconfig.SourceFile
	idToNode := make(map[string]*build.ProjectNode)

	for _, proj := range input.Projects {
		// 👉 内存构造虚拟 tsconfig.SourceFile，没有磁盘json文件
		virtualTsConfig := &tsconfig.SourceFile{
			// Path：这里用我们的ProjectID充当配置路径key，可以是虚拟路径，不需要真实存在
			Path: proj.ProjectID,
			CompilerOptions: proj.CompilerOptions,
			Composite: true, // 必须composite:true，build模块强制要求composite项目
			OutDir: proj.OutDir,
			Files:  proj.SourceFiles,
			// References 这里留空，我们手动构建 ProjectNode.Dependencies，不走tsconfig references字段
			References: nil,
		}

		node := &build.ProjectNode{
			Config: virtualTsConfig,
			// Dependencies 后续回填
			Dependencies: make([]*build.ProjectNode, 0),
		}
		idToNode[proj.ProjectID] = node
	}

	// 2. 回填依赖边：根据 DependsOn 建立 ProjectNode.Dependencies
	for _, proj := range input.Projects {
		thisNode := idToNode[proj.ProjectID]
		for _, depID := range proj.DependsOn {
			depNode, ok := idToNode[depID]
			if !ok {
				return nil, fmt.Errorf("project %s depends on unknown project %s", proj.ProjectID, depID)
			}
			thisNode.Dependencies = append(thisNode.Dependencies, depNode)
		}
	}

	// 3. 组装ProjectGraph
	graph := &build.ProjectGraph{
		Nodes: make(map[string]*build.ProjectNode),
	}
	for id, n := range idToNode {
		graph.Nodes[id] = n
	}

	// 4. 检测循环依赖（复用原有build内部校验逻辑！不要自己写环检测）
	// 注意：build 内部有 ValidateGraph 做环检测
	if err := graph.ValidateGraph(); err != nil {
		return nil, err
	}

	// roots：没有被别人引用的顶层节点（等价原来根tsconfig references的节点）
	graph.Roots = collectRootNodes(graph)

	return graph, nil
}

func collectRootNodes(g *build.ProjectGraph) []*build.ProjectNode {
	// 简单实现：没有入边的节点作为root；也可以允许上层传入指定root ID列表
	hasInEdge := make(map[*build.ProjectNode]bool)
	for _, node := range g.Nodes {
		for _, dep := range node.Dependencies {
			hasInEdge[dep] = true
		}
	}
	var roots []*build.ProjectNode
	for _, n := range g.Nodes {
		if !hasInEdge[n] {
			roots = append(roots, n)
		}
	}
	return roots
}
```

## 3. 上层调用示例（业务层使用这个新API）

```go
func demoUsage() {
	input := &custombuild.CustomMonorepoInput{
		Projects: []*custombuild.MonorepoProject{
			{
				ProjectID: "packages/utils",
				SourceFiles: []string{"./packages/utils/index.ts"},
				OutDir: "./packages/utils/dist",
				DependsOn: []string{},
				CompilerOptions: tsconfig.CompilerOptions{
					Strict: true,
					Emit: true,
				},
			},
			{
				ProjectID: "packages/app",
				SourceFiles: []string{"./packages/app/main.ts"},
				OutDir: "./packages/app/dist",
				DependsOn: []string{"packages/utils"},
				CompilerOptions: tsconfig.CompilerOptions{Strict:true},
			},
		},
	}
	input.BuildOptions.Builders = 4
	input.BuildOptions.Checkers = 2

	res, err := custombuild.BuildMonorepo(input)
	if err != nil {
		panic(err)
	}

	// res 就是 build.BuildResult，复用原有结果结构，读取诊断、成功失败
	for _, diag := range res.Diagnostics {
		// handle diagnostics
	}
}
```

# 关键约束 & 注意点（非常重要）

1. **完全不改动 `internal/execute/build` 内部任何代码**，所有胶水逻辑全部写在新增的 `custombuild` 包。原有 ProjectGraph、Builder、WorkerPool、Incremental 全部原样跑。
2. `ProjectNode.Config` 必须设置 `Composite:true`，原版 build 模块对 composite 项目有强假设，`.tsbuildinfo`、引用解析都依赖这个标记。
3. 我们**没有使用 tsconfig 的 `references` 数组**，而是直接手动填充 `ProjectNode.Dependencies`。

> 👉 原版 `project_graph.go` 在从磁盘tsconfig构建图的时候，会解析 `references` 然后填充 `.Dependencies`；我们直接跳过解析json，直接赋值这个字段。**build.Builder 在运行时只读取 ProjectNode.Dependencies，不再读取 tsconfig.Config.References**，源码确认：Builder 拓扑排序、调度全部使用 `node.Dependencies`。

4. `tsconfig.SourceFile.Path`：这里用虚拟 `ProjectID` 作为key，`ProjectGraph.Nodes` map 的 key 和这个 Path 对应。

> ⚠️但是：**增量 `.tsbuildinfo` 文件默认以 config.Path 作为基础去生成文件路径**。如果你 Path 是虚拟不存在的路径，会导致 `.tsbuildinfo` 输出位置异常。  
> 两种解决：
> 
> - 方案A：给每个虚拟项目传入真实磁盘路径作为 Path（即使没有tsconfig.json文件），`.tsbuildinfo` 就会输出到该目录；推荐。
> - 方案B：如果要完全内存增量，需要修改 `incremental.go` 的读取写入点，传入自定义内存存储，这就超出“简单包一层”范围。

5. 源文件：`Config.Files` 填磁盘文件路径；如果要完全内存源文件（不读磁盘），还需要替换底层 fs 抽象（`sys` 层），属于编译器底层改造，不是编排层的改动。当前封装只解决**monorepo编排层绕过tsconfig json磁盘文件**。
6. internal包访问限制：

> 如果你的代码不在 typescript‑go 仓库内部，不能 import `internal/*`。解决方案：把我们写的 `custombuild` API 挪到 `pkg/` 目录，并且把 build 模块需要的类型做导出包装。

## 原有执行链路 vs 封装后链路

### 原版链路

```
CLI → 读取磁盘tsconfig.json → BuildGraphFromConfigFile(磁盘) → ProjectGraph → Builder → WorkerPool → 编译
```

### 封装后链路（我们新增的薄包装）

```
【业务层传入内存MonorepoInput】
    → custombuild胶水层：内存构造虚拟tsconfig.SourceFile + 手动组装ProjectGraph
        → 直接喂给原有 build.Builder
            → 拓扑排序、worker调度、增量、编译【完全复用原有逻辑，零修改】
```

## 可扩展点（不改动build内部）

1. 如果需要指定根项目，新增字段 `RootProjectIDs []string`，在胶水层手动设置 `graph.Roots`，不再自动收集root。
2. 如果需要关闭磁盘增量：在调用 Builder 之前，可以将每个 node 的增量信息强制禁用；不需要修改 build 源码。
3. 诊断、退出逻辑完全复用 `build.BuildResult`，不需要自己实现错误聚合。

如果你需要，我可以：

1. 给出对应关键源码片段定位（typescript‑go 仓库中哪几行确认 Builder 只依赖 ProjectNode.Dependencies）；
2. 给出最小可编译的完整项目文件；
3. 给出规避 `.tsbuildinfo` 虚拟路径问题的适配代码。