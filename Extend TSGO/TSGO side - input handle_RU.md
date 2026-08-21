# tsgo: схема обёртки внутреннего API оркестрации — в обход tsconfig, данные Monorepo напрямую в память

2026-08-20 10:22 · создано: fanglin 30015131 · последнее обновление: fanglin 30015131, 2026-08-20 10:22

> Содержимое сгенерировано ИИ, только для справки

> Цель: **не менять исходную логику оркестрации `internal/execute/build`**, сделать лишь тонкий слой обёртки; не читать tsconfig.json с диска, а напрямую передать описание monorepo-проектов, сконструированное в памяти, и переиспользовать весь существующий конвейер: ProjectGraph, топологическую сортировку, планирование воркеров, инкрементальность, build.
>
> Ключевое понимание: build-модуль `typescript-go` **жёстко зависит от tsconfig.SourceFile (распарсенного объекта конфигурации)**, исходный ProjectGraph строит узлы, читая tsconfig-файлы с диска. Наша задача: **сконструировать виртуальные tsconfig-узлы в памяти и скормить их существующей build-системе — заменить дисковый IO конструированием в памяти**.

> ⚠️ Замечание про уровень пакета: пакеты `internal/` в Go — внутренние, внешние проекты не могут их напрямую импортировать. Две формы использования:
>
> 1. Добавить слой обёртки внутри репозитория исходников tsgo (рекомендуется — есть прямой доступ к internal)
> 2. Для внешнего проекта: нужные API придётся поднять в экспорт `pkg/`, иначе ошибка компиляции. Примеры ниже основаны на **добавлении слоя обёртки внутри репозитория typescript-go**.

## Обзор ключевых типов исходного build (internal/execute/build)

```go
// builder.go
type Builder struct {
	graph         *ProjectGraph
	workerPool    *WorkerPool
	// ...
}

// project_graph.go
// Каждый узел графа представляет один composite-проект tsconfig
type ProjectNode struct {
	Config *tsconfig.SourceFile // 【Ключ】это распарсенный объект конфигурации tsconfig
	// Рёбра зависимостей: от каких других ProjectNode зависит узел
	Dependencies []*ProjectNode
	// ...
}

type ProjectGraph struct {
	Roots []*ProjectNode
	Nodes map[string]*ProjectNode // key: идентификатор пути файла конфигурации
}
```

Исходный поток: чтение tsconfig с диска → парсинг в `*tsconfig.SourceFile` → построение `ProjectNode` → сборка `ProjectGraph` → передача Builder'у.

Идея нашего слоя-обёртки:

> Больше никакого чтения json с диска: **конструируем в памяти набор виртуальных объектов `tsconfig.SourceFile`, вручную строим ProjectNode и собираем ProjectGraph**, напрямую отдаём исходному Builder'у — вся дальнейшая топология, воркеры, инкрементальность переиспользуют исходную логику, оркестрация не меняется вовсе.

> Примечание: инкрементальность `.tsbuildinfo` по-прежнему читает/пишет диск; для полностью in-memory инкрементальности пришлось бы заменить файловый IO в `incremental.go`. Здесь задача — просто обернуть, исходное инкрементальное поведение сохраняется.

---

# 1. Пользовательская входная структура данных (входной параметр внешнего API)

Определяем собственное простое описание Monorepo в памяти, **отвязанное от tsconfig**: бизнес-слой заполняет только эту структуру, никакие tsconfig-файлы писать не нужно.

Новый файл (пример): `internal/execute/custombuild/api.go`

```go
package custombuild

import (
	"github.com/microsoft/typescript-go/internal/tsconfig"
	"github.com/microsoft/typescript-go/internal/execute/build"
)

// Внешний интерфейс: описание monorepo-проекта, передаваемое пользователем,
// никакого tsconfig на диске не требуется
type MonorepoProject struct {
	// Идентификатор виртуального проекта, используется как key в ProjectGraph,
	// может не быть реальным путём на диске
	ProjectID string

	// Список исходных файлов проекта (файлы в памяти либо пути к файлам на диске)
	SourceFiles []string

	// Каталог вывода компиляции
	OutDir string

	// ProjectID других проектов-зависимостей; соответствует MonorepoProject.ProjectID выше
	DependsOn []string

	// Опции компиляции, эквивалент compilerOptions из tsconfig
	CompilerOptions tsconfig.CompilerOptions
}

// Внешний входной параметр верхнего уровня
type CustomMonorepoInput struct {
	Projects []*MonorepoProject

	// Параметры выполнения build, эквивалент аргументов командной строки
	BuildOptions struct {
		Builders int // --builders
		Checkers int // --checkers
	}
}

// Внешний API: функция входа — напрямую принимает in-memory определение monorepo,
// переиспользует исходную оркестрацию build
func BuildMonorepo(input *CustomMonorepoInput) (*build.BuildResult, error) {
	// 1. Преобразуем пользовательскую in-memory модель в *build.ProjectGraph,
	//    нужный build-модулю
	graph, err := buildGraphFromInput(input)
	if err != nil {
		return nil, err
	}

	// 2. Переиспользуем конфигурацию исходного Builder, полностью переиспользуем
	//    исходную логику оркестрации
	builderOpts := build.BuilderOptions{
		Builders: input.BuildOptions.Builders,
		Checkers: input.BuildOptions.Checkers,
		// Прочие исходные настройки переносим как есть из значений CLI tsgo по умолчанию
		ContinueOnError: false,
	}

	b := build.NewBuilder(graph, &builderOpts)

	// 3. Выполняем весь build-конвейер: топологическая сортировка, пул воркеров,
	//    инкрементальность, компиляция, сбор диагностики
	result, err := b.Run()
	return result, err
}
```

## 2. Ключевая функция адаптации: построение `*build.ProjectGraph` из пользовательской in-memory структуры

> Это и есть тонкий клеевой слой: **вручную конструируем виртуальный tsconfig.SourceFile, строим ProjectNode, заводим рёбра зависимостей — логику чтения tsconfig с диска не вызываем**.
>
> > Исходная `project_graph.go#BuildGraphFromConfigFile()` читает диск; мы не идём через эту функцию, а сами конструируем весь граф в памяти.

```go
// buildGraphFromInput превращает пользовательский in-memory monorepo в ProjectGraph,
// нужный build-модулю
func buildGraphFromInput(input *CustomMonorepoInput) (*build.ProjectGraph, error) {
	// 1. Сначала создаём все ProjectNode, каждый узел привязываем к виртуальному
	//    in-memory tsconfig.SourceFile
	idToNode := make(map[string]*build.ProjectNode)

	for _, proj := range input.Projects {
		// 👉 In-memory конструирование виртуального tsconfig.SourceFile,
		//    json-файла на диске нет
		virtualTsConfig := &tsconfig.SourceFile{
			// Path: здесь в качестве key пути конфигурации берём наш ProjectID,
			// путь может быть виртуальным, не обязан реально существовать
			Path: proj.ProjectID,
			CompilerOptions: proj.CompilerOptions,
			Composite: true, // обязательно composite:true — build-модуль жёстко требует composite-проекты
			OutDir: proj.OutDir,
			Files:  proj.SourceFiles,
			// References оставляем пустым: мы вручную строим ProjectNode.Dependencies,
			// поле references из tsconfig не используем
			References: nil,
		}

		node := &build.ProjectNode{
			Config: virtualTsConfig,
			// Dependencies заполним позже
			Dependencies: make([]*build.ProjectNode, 0),
		}
		idToNode[proj.ProjectID] = node
	}

	// 2. Заполняем рёбра зависимостей: по DependsOn строим ProjectNode.Dependencies
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

	// 3. Собираем ProjectGraph
	graph := &build.ProjectGraph{
		Nodes: make(map[string]*build.ProjectNode),
	}
	for id, n := range idToNode {
		graph.Nodes[id] = n
	}

	// 4. Проверяем циклические зависимости (переиспользуем внутреннюю валидацию build!
	//    свою проверку циклов не пишем)
	// Внимание: внутри build есть ValidateGraph для проверки циклов
	if err := graph.ValidateGraph(); err != nil {
		return nil, err
	}

	// roots: верхнеуровневые узлы, на которые никто не ссылается
	// (эквивалент узлов из references корневого tsconfig)
	graph.Roots = collectRootNodes(graph)

	return graph, nil
}

func collectRootNodes(g *build.ProjectGraph) []*build.ProjectNode {
	// Простая реализация: узлы без входящих рёбер считаются корневыми;
	// можно также позволить верхнему слою передавать явный список ID корней
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

## 3. Пример вызова с верхнего уровня (бизнес-слой использует этот новый API)

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

	// res — это build.BuildResult: переиспользуем исходную структуру результата,
	// читаем диагностику, успех/неудачу
	for _, diag := range res.Diagnostics {
		// handle diagnostics
	}
}
```

# Ключевые ограничения и замечания (очень важно)

1. **Полностью не трогаем ни одной строки внутри `internal/execute/build`**, вся клеевая логика живёт в новом пакете `custombuild`. Исходные ProjectGraph, Builder, WorkerPool, Incremental работают как есть.
2. `ProjectNode.Config` обязан иметь `Composite:true` — исходный build-модуль делает жёсткие предположения о composite-проектах, от этого флага зависят `.tsbuildinfo` и разбор ссылок.
3. Мы **не используем массив `references` из tsconfig**, а вручную заполняем `ProjectNode.Dependencies`.

> 👉 Исходный `project_graph.go` при построении графа из дискового tsconfig разбирает `references` и заполняет `.Dependencies`; мы пропускаем разбор json и присваиваем это поле напрямую. **build.Builder во время выполнения читает только ProjectNode.Dependencies и больше не читает tsconfig.Config.References** — подтверждено по исходникам: топологическая сортировка и планирование Builder'а используют `node.Dependencies`.

4. `tsconfig.SourceFile.Path`: здесь виртуальный `ProjectID` используется как key, key в map `ProjectGraph.Nodes` соответствует этому Path.

> ⚠️ Но: **инкрементальный `.tsbuildinfo` по умолчанию строит путь файла на основе config.Path**. Если Path — несуществующий виртуальный путь, позиция вывода `.tsbuildinfo` будет аномальной.
> Два решения:
>
> - Вариант A: передавать каждому виртуальному проекту реальный путь на диске как Path (даже если файла tsconfig.json нет), тогда `.tsbuildinfo` пишется в этот каталог; рекомендуется.
> - Вариант B: для полностью in-memory инкрементальности нужно менять точки чтения/записи в `incremental.go` и передавать собственное in-memory хранилище — это уже выходит за рамки «просто обернуть».

5. Исходные файлы: в `Config.Files` кладём пути к файлам на диске; для полностью in-memory исходников (без чтения диска) пришлось бы заменить низкоуровневую абстракцию fs (слой `sys`) — это переделка недр компилятора, а не уровня оркестрации. Текущая обёртка решает только **обход дисковых tsconfig json на уровне оркестрации monorepo**.
6. Ограничение доступа к internal-пакетам:

> Если ваш код вне репозитория typescript-go, импортировать `internal/*` нельзя. Решение: перенести написанный нами API `custombuild` в каталог `pkg/` и сделать экспортные обёртки над типами, нужными build-модулю.

## Исходная цепочка выполнения vs цепочка после обёртки

### Исходная цепочка

```
CLI → чтение tsconfig.json с диска → BuildGraphFromConfigFile(диск) → ProjectGraph → Builder → WorkerPool → компиляция
```

### Цепочка после обёртки (наша новая тонкая обёртка)

```
[бизнес-слой передаёт in-memory MonorepoInput]
    → клеевой слой custombuild: in-memory виртуальный tsconfig.SourceFile + ручная сборка ProjectGraph
        → напрямую отдаём исходному build.Builder
            → топологическая сортировка, планирование воркеров, инкрементальность, компиляция [полное переиспользование исходной логики, ноль изменений]
```

## Точки расширения (без изменений внутри build)

1. Если нужно указать корневые проекты — добавить поле `RootProjectIDs []string` и в клеевом слое вручную выставить `graph.Roots` вместо автоматического сбора корней.
2. Если нужно отключить дисковую инкрементальность — перед вызовом Builder можно принудительно отключить инкрементальную информацию каждого узла; исходники build менять не нужно.
3. Диагностика и логика выхода полностью переиспользуют `build.BuildResult`, свой агрегатор ошибок реализовывать не нужно.

Если нужно, я могу:

1. Дать точную привязку к фрагментам исходников (в каких строках репозитория typescript-go подтверждается, что Builder зависит только от ProjectNode.Dependencies);
2. Дать минимальный полный компилируемый набор файлов проекта;
3. Дать код адаптации, обходящий проблему виртуального пути `.tsbuildinfo`.
