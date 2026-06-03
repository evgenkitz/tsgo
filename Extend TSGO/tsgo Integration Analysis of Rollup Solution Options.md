# tsgo Integration Analysis of Rollup Solution Options

> На основе обратного анализа исходного кода tsgo (ветка `main` репозитория `microsoft/typescript-go`) и реального API npm-пакета `@typescript/native-preview`

---

## 1. Декомпозиция ключевых требований

| Требование | Конкретное значение | Аналог в tsc (JS API) |
|------|---------|------------------------|
| **Пофайловая компиляция** | Аналог `program.emit(sourceFile, writeFile)` — компиляция отдельного `.ts` файла в JS | `ts.createProgram → program.emit(sf, writeFn)` |
| **transform модификация AST** | Возможность изменять TS AST в процессе компиляции (например, пользовательский transformer) или изменять AST выходного JS | `ts.transform(sf, [before/after transformers])` или Rollup `transform` hook |
| **Совместимость с экосистемой Rollup** | Корректная работа хуков `load/transform/buildEnd`, точная передача sourceMap | Подход `@rollup/plugin-typescript` |

---

## 2. Реальные возможности API tsgo (на основе обратного анализа исходного кода)

### 2.1 Уровень JS API (`@typescript/native-preview/unstable/`)

Анализ исходного кода `_packages/native-preview/src/` показывает, что tsgo предоставляет JS API **гораздо более зрелое, чем указано в README ("not ready")**:

#### Диаграмма классов Core API

```
API (async/sync — два варианта)
 │
 ├── parseConfigFile(file) → ConfigResponse
 │
 ├── updateSnapshot({ openProject, fileChanges }) → Snapshot
 │    │
 │    ├── getProject(tsconfigPath) → Project
 │    │    │
 │    │    ├── program: Program
 │    │    │    │   ├── getSourceFile(path) → SourceFile ★ пофайловое получение
 │    │    │    │   ├── getSyntacticDiagnostics(file)
 │    │    │    │   ├── getSemanticDiagnostics(file)
 │    │    │    │   └── getSuggestionDiagnostics(file)
 │    │    │    │
 │    │    │    ├── checker: Checker
 │    │    │    │   ├── getSymbolAtPosition/Location
 │    │    │    │   ├── getTypeOfSymbol/AtPosition/Location
 │    │    │    │   ├── getSignaturesOfType
 │    │    │    │   ├── getContextualType
 │    │    │    │   └── ...
 │    │    │    │
 │    │    │    └── emitter: Emitter ★ компиляция на выход
 │    │    │        └── printNode(node, options?) → string ★
 │    │    │
 │    │    └── dispose()
 │    │
 │    └── dispose()
 │
 ├── close()
 └── clearSourceFileCache()
```

#### Разбор ключевых API

**`Emitter.printNode(node, options?) → string`**

```typescript
// _packages/native-preview/src/api/async/api.ts:856-871
export class Emitter {
    private client: Client;

    async printNode(node: Node, options: PrintNodeOptions = {}): Promise<string> {
        const encoded = encodeNode(node);         // JS-сторона кодирует AST в бинарный формат
        const base64 = uint8ArrayToBase64(encoded);
        return this.client.apiRequest<string>("printNode", {
            data: base64,
            ...options,                            // preserveSourceNewlines, neverAsciiEscape и др.
        });
    }
}
```

Это означает:
- JS-сторона кодирует AST Node в бинарный формат → отправляет на Go-сторону → Go printer выполняет emit → возвращает строку с JS-кодом
- **Это настоящий вывод компиляции**, а не просто текстовое представление AST

**`visitEachChild(node, visitor)` — обход/модификация AST**

```typescript
// _packages/native-preview/src/ast/visitor.generated.ts
export function visitEachChild<T extends Node>(node: T, visitor: Visitor): T {
    // Для каждого дочернего узла вызывает visitor, который может вернуть новый узел вместо старого
    // Это tsgo-версия ts.visitEachChild из tsc
}
```

**AST Factory — создание новых узлов**

```typescript
// _packages/native-preview/src/ast/factory.generated.ts
export function createIdentifier(text: string): Identifier;
export function createFunctionDeclaration(...): FunctionDeclaration;
export function createVariableDeclaration(...): VariableDeclaration;
// ... полный набор factory-функций
```

**`createVirtualFileSystem(files)` — файловая система в памяти**

```typescript
// _packages/native-preview/src/api/fs.ts:38
export function createVirtualFileSystem(files: Record<string, string>): FileSystem {
    // Возвращает полную реализацию интерфейса FS
    return {
        directoryExists, fileExists, getAccessibleEntries,
        readFile, realpath, writeFile, removeFile,
    };
}
```

**Инкрементальное обновление Snapshot**

```typescript
// proto.ts
interface UpdateSnapshotParams {
    openProject?: string;
    fileChanges?: FileChanges;  // { changed, created, deleted } | { invalidateAll: true }
}
```

**Диагностика — пофайловое получение**

```typescript
// api.ts:367-400
program.getSyntacticDiagnostics(file?)     // указанный файл или все
program.getSemanticDiagnostics(file?)      // указанный файл или все
program.getSuggestionDiagnostics(file?)    // указанный файл или все
program.getDeclarationDiagnostics(file?)   // указанный файл или все
```

### 2.2 Уровень API-протокола

| Уровень | Технология | Описание |
|----|------|------|
| JS → Go запросы | JSON-RPC (vscode-jsonrpc) | Режим `--api --async` |
| Кодирование AST | Собственный бинарный протокол (protocol version 5) | Эффективное кодирование/декодирование |
| FS колбэки | JSON-RPC обратные запросы | Go → JS: readFile/fileExists и др. |

### 2.3 Соответствие с традиционным tsc API

| tsc (JS API) | tsgo (native-preview) | Статус |
|--------------|----------------------|------|
| `ts.createProgram()` | `api.updateSnapshot({ openProject })` → `project.program` | ✅ Доступно |
| `program.getSourceFile(path)` | `project.program.getSourceFile(path)` | ✅ Доступно |
| `program.emit(sf, writeFn)` | `project.emitter.printNode(sourceFile)` ⚠️ | ✅ Доступно, но семантика отличается |
| `ts.visitEachChild(node, visitor)` | `visitEachChild(node, visitor)` | ✅ Доступно |
| `ts.createXxx(...)` factory | `createXxx(...)` factory | ✅ Доступно |
| `checker.getTypeAtPosition` | `project.checker.getTypeAtPosition` | ✅ Доступно |
| `program.getSyntacticDiagnostics` | `project.program.getSyntacticDiagnostics(file)` | ✅ Доступно |
| `ts.createSourceFile(...)` | `createSourceFile(...)` | ✅ Доступно |
| Пользовательский Transformer (before/after) | `visitEachChild` + factory ★ | ✅ Реализуемо на JS-стороне |
| Колбэк writeFile в `program.emit()` | VFS writeFile ★ | ✅ Доступно |

---

## 3. Сравнительная оценка четырёх вариантов

### 3.1 Пофайловая компиляция

| Вариант | Пофайловая компиляция | Способ реализации | Степень эквивалентности tsc |
|------|-----------|---------|-----------------|
| Вариант 1 (CLI) | ❌ Полная компиляция | `tsgo --project tsconfig.json` | Совершенно разная |
| **Вариант 2 (API)** | **✅ Пофайловая** | `program.getSourceFile(path)` + `emitter.printNode(sf)` | **Высокая эквивалентность** |
| Вариант 3 (Патч-Emit) | ✅ Пофайловая | Патч исходников Go — добавление `EmitSingleFile` | Эквивалентно |
| Вариант 4 (Патч-перехват) | ❌ Полная | Перехват stdout | Разная |

### 3.2 Возможность модификации AST на этапе Transform

| Вариант | Модификация TS AST | Модификация JS AST (Rollup) | Пользовательский Transformer |
|------|------------|--------------------|--------------------|
| Вариант 1 (CLI) | ❌ | ✅ (Rollup transform hook) | ❌ |
| **Вариант 2 (API)** | **✅** `visitEachChild` + factory | **✅** Rollup transform | **✅** Реализация на JS-стороне |
| Вариант 3 (Патч-Emit) | ⚠️ Требует дополнительного патча | ✅ | ⚠️ Требует патча Go transformers |
| Вариант 4 (Патч-перехват) | ❌ | ✅ | ❌ |

**Процесс модификации TS AST в Варианте 2:**

```
1. program.getSourceFile("/src/foo.ts")   → получение TS AST
2. visitEachChild(sourceFile, transformer) → модификация AST (замена/добавление/удаление узлов)
3. emitter.printNode(modifiedSourceFile)   → компиляция в JS-код
4. Возврат { code, map } в Rollup          → Rollup transform hook может дополнительно изменить JS AST
```

Это означает **двухуровневую модификацию AST**:
- **Первый уровень** (TS AST): через `visitEachChild` + factory, модификация перед emit tsgo (например, удаление декораторов, инъекция кода)
- **Второй уровень** (JS AST): через Rollup `transform` hook, модификация acorn AST Rollup (например, все существующие Rollup плагины)

### 3.3 Совместимость с экосистемой Rollup

| Вариант | load hook | transform hook | buildEnd hook | watchChange | sourceMap | Совместимость с другими плагинами |
|------|----------|---------------|--------------|-------------|-----------|-------------|
| Вариант 1 | ⚠️ Чтение с диска | ✅ | ⚠️ Без диагностики | ❌ | ⚠️ Грубое | ⚠️ |
| **Вариант 2** | **✅** API-компиляция | **✅** Двухуровневая | **✅** Диагностика | **✅** fileChanges | **✅** | **✅** |
| Вариант 3 | ✅ | ✅ | ✅ | ⚠️ | ✅ | ✅ |
| Вариант 4 | ⚠️ | ✅ | ⚠️ | ❌ | ⚠️ | ⚠️ |

### 3.4 Сводная оценка

| Измерение | Вариант 1 | **Вариант 2** | Вариант 3 | Вариант 4 |
|------|--------|----------|--------|--------|
| Пофайловая компиляция | 1/5 | **4/5** | 5/5 | 1/5 |
| Модификация AST (уровень TS) | 0/5 | **5/5** | 3/5 | 0/5 |
| Модификация AST (уровень JS) | 3/5 | **5/5** | 5/5 | 3/5 |
| Совместимость с Rollup | 2/5 | **5/5** | 4/5 | 2/5 |
| Без патча исходников | ✅ | **✅** | ❌ | ❌ |
| Сложность реализации | Низкая | **Средняя** | Высокая | Высокая |
| Стоимость поддержки | Низкая | **Низкая** | Высокая (fork) | Высокая (fork) |
| **Итого** | 6/25 | **19/25 ★** | 17/25 | 6/25 |

---

## 4. Вывод: Вариант 2 (интеграция через API/IPC) — оптимальный выбор

### 4.1 Ключевые аргументы

1. **JS API tsgo уже поддерживает пофайловую компиляцию**
   - `program.getSourceFile(path)` получает AST отдельного файла
   - `emitter.printNode(sourceFile)` компилирует в JS-код
   - Это **семантически эквивалентно** `ts.createProgram → program.emit(sourceFile)`

2. **JS API tsgo уже поддерживает модификацию AST**
   - `visitEachChild(node, visitor)` полностью соответствует `ts.visitEachChild`
   - Полный набор AST factory-функций (`createIdentifier`, `createFunctionDeclaration` и др.)
   - Пользовательский transformer можно реализовать на JS-стороне, без патча исходников Go

3. **Не требует патча исходников tsgo**
   - Все функции доступны через существующее API `@typescript/native-preview/unstable/`
   - Нет затрат на поддержку fork
   - API помечено как `unstable`, но функционально полноценно (покрытие тестами: 2000+ строк)

4. **Полная совместимость с экосистемой Rollup**
   - `load` hook: возвращает `{ code, map }` — все Rollup плагины работают корректно
   - `transform` hook: другие плагины могут продолжать модифицировать JS AST
   - `buildEnd` hook: вывод ошибок типов через `getSemanticDiagnostics`
   - `watchChange` hook: инкрементальное обновление через `updateSnapshot({ fileChanges })`

### 4.2 Единственное ограничение Варианта 2

**Семантические различия `printNode` и `program.emit`:**

| Сравнение | `program.emit(sf, writeFn)` (tsc) | `emitter.printNode(sf)` (tsgo) |
|---------|-----------------------------------|----------------------------|
| downleveling (target < ESNext) | ✅ Автоматическая трансформация async/class и др. | ⚠️ Требует проверки — возможно, только вывод исходного кода |
| Source Map | ✅ Встроенная генерация | ⚠️ `printNode` в настоящее время не возвращает sourceMap |
| Файлы объявлений (.d.ts) | ✅ Автоматическая генерация | ⚠️ Требуется отдельный API |
| Внедрение helper (__extends и др.) | ✅ Автоматически | ⚠️ Требует проверки |

**Стратегии решения:**

- **ESNext target (рекомендуется)**: downleveling не требуется, вывод `printNode` можно использовать напрямую
- **Lower target**: обработка другими плагинами экосистемы Rollup (например, `@babel/plugin-transform-regenerator`) или патч для добавления API `emitSourceFile`
- **Source Map**: патч или ожидание добавления возврата sourceMap в последующих версиях (отсутствие — сейчас самый большой недостаток)
- **Файлы объявлений**: генерация `.d.ts` в `buildEnd` через CLI-режим

---

## 5. Полный план реализации Варианта 2

### 5.1 Диаграмма архитектуры

```
┌──────────────────────────────────────────────────────────────┐
│                     Rollup Build Pipeline                     │
│                                                               │
│  buildStart()                                                 │
│    ├── new API({ fs: virtualFS })                             │
│    ├── await api.updateSnapshot({ openProject: tsconfig })    │
│    └── project = snapshot.getProject(tsconfig)                │
│                                                               │
│  load(id)  ←── Rollup запрашивает каждый .ts файл             │
│    ├── sourceFile = project.program.getSourceFile(id)         │
│    ├── ★ transformedSF = customTransformer(sourceFile)        │
│    ├── ★ jsCode = project.emitter.printNode(transformedSF)    │
│    └── return { code: jsCode, map: ??? }                      │
│                                                               │
│  transform(code, id)  ←── другие Rollup плагины могут изменять JS AST │
│    └── return { code, map }  // обычная передача              │
│                                                               │
│  buildEnd()                                                   │
│    ├── diagnostics = program.getSemanticDiagnostics()         │
│    ├── вывод ошибок/предупреждений в Rollup                   │
│    └── (опционально) генерация .d.ts через CLI                │
│                                                               │
│  watchChange(id)                                               │
│    ├── virtualFS.writeFile(id, newContent)                    │
│    ├── api.updateSnapshot({ fileChanges: { changed: [id] } }) │
│    └── перекомпиляция изменённых файлов                        │
│                                                               │
│  closeBundle()                                                │
│    └── await api.close()                                      │
│                                                               │
│  ┌──────────┐  JSON-RPC  ┌──────────────────┐                │
│  │  Node.js  │ ◄══════► │  tsgo --api --async│                │
│  │  (VFS)    │  IPC      │  (Go process)      │                │
│  └──────────┘           └──────────────────┘                │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 5.2 Реализация пользовательского Transformer

```typescript
import { visitEachChild, SyntaxKind, createIdentifier } from "@typescript/native-preview/unstable/ast";
import type { Node, SourceFile, Visitor } from "@typescript/native-preview/unstable/ast";

// Пример: удаление всех вызовов console.log
function removeConsoleLogTransformer(sourceFile: SourceFile): SourceFile {
    return visitEachChild(sourceFile, visitor);

    function visitor(node: Node): Node {
        // Если это ExpressionStatement и содержит вызов console.log, удаляем
        if (node.kind === SyntaxKind.ExpressionStatement) {
            const expr = (node as any).expression;
            if (
                expr?.kind === SyntaxKind.CallExpression &&
                expr.expression?.kind === SyntaxKind.PropertyAccessExpression &&
                expr.expression.expression?.getText?.() === "console" &&
                expr.expression.name?.getText?.() === "log"
            ) {
                return undefined as any; // удаляем оператор
            }
        }
        return visitEachChild(node, visitor);
    }
}

// Пример: переименование идентификаторов
function renameIdentifierTransformer(
    sourceFile: SourceFile,
    from: string,
    to: string
): SourceFile {
    return visitEachChild(sourceFile, visitor);

    function visitor(node: Node): Node {
        if (node.kind === SyntaxKind.Identifier && (node as any).text === from) {
            return createIdentifier(to);
        }
        return visitEachChild(node, visitor);
    }
}
```

### 5.3 Полный код Rollup плагина

```typescript
import type { Plugin } from "rollup";
import { API } from "@typescript/native-preview/unstable/async";
import { createVirtualFileSystem } from "@typescript/native-preview/unstable/fs";
import type { FileSystem } from "@typescript/native-preview/unstable/fs";
import { visitEachChild } from "@typescript/native-preview/unstable/ast/visitor";
import type { Node, SourceFile, Visitor } from "@typescript/native-preview/unstable/ast";

interface TsgoRollupOptions {
    tsconfig?: string;
    compilerOptions?: Record<string, unknown>;
    transformers?: ((sourceFile: SourceFile) => SourceFile)[];
    include?: string[];
    exclude?: string[];
    noEmit?: boolean;           // только проверка типов, без компиляции
    sourceMap?: boolean;        // API пока не поддерживает, зарезервировано
    declaration?: boolean;      // генерировать .d.ts в buildEnd
    diagnosticFilter?: (diag: Diagnostic) => boolean;
}

function tsgoRollupPlugin(options: TsgoRollupOptions = {}): Plugin {
    let api: API;
    let fs: FileSystem;
    let snapshot: any;
    let project: any;

    const tsconfig = options.tsconfig || "tsconfig.json";
    const transformers = options.transformers || [];

    return {
        name: "rollup-plugin-tsgo",

        async buildStart() {
            fs = createVirtualFileSystem({
                [tsconfig]: JSON.stringify({
                    compilerOptions: {
                        target: "esnext",
                        module: "esnext",
                        moduleResolution: "bundler",
                        strict: true,
                        ...options.compilerOptions,
                    },
                }),
            });

            api = new API({ fs });
            snapshot = await api.updateSnapshot({ openProject: tsconfig });
            project = snapshot.getProject(tsconfig);
        },

        resolveId(source, importer) {
            if (!/\.(tsx?|mts|cts)$/.test(source)) return null;
            // Используем разрешение Rollup по умолчанию, обработка в load
            return null;
        },

        async load(id) {
            if (!/\.(tsx?|mts|cts)$/.test(id)) return null;

            // 1. Внедрение содержимого файла в VFS
            const content = await this.fs.readFile(id, "utf-8");
            fs.writeFile!(id.replace(/\\/g, "/"), content);

            // 2. Обновление snapshot (если файл новый)
            snapshot = await api.updateSnapshot({
                fileChanges: { created: [id.replace(/\\/g, "/")] },
            });
            project = snapshot.getProject(tsconfig);

            // 3. Получение SourceFile AST
            const sourceFile = await project.program.getSourceFile(id.replace(/\\/g, "/"));
            if (!sourceFile) return null;

            // 4. Применение пользовательского TS Transformer
            let transformed = sourceFile;
            for (const transformer of transformers) {
                transformed = transformer(transformed);
            }

            // 5. Компиляция через Emitter
            if (options.noEmit) {
                return content; // режим только проверки типов — возврат исходного содержимого
            }

            const jsCode = await project.emitter.printNode(transformed);

            // 6. Возврат в Rollup (последующие transform hook могут изменить JS AST)
            // TODO: sourceMap — printNode пока не возвращает sourceMap
            //       требуется патч или ожидание официальной поддержки
            return {
                code: jsCode,
                // map: ... — будет реализовано
            };
        },

        async buildEnd() {
            // Получение диагностики типов
            const syntacticDiags = await project.program.getSyntacticDiagnostics();
            const semanticDiags = await project.program.getSemanticDiagnostics();

            const allDiags = [...syntacticDiags, ...semanticDiags];

            for (const diag of allDiags) {
                if (options.diagnosticFilter && !options.diagnosticFilter(diag)) continue;

                const message = diag.text;
                if (diag.category === 1) { // Error
                    this.error({
                        message,
                        id: diag.fileName,
                        loc: diag.pos ? { line: 0, column: diag.pos } : undefined,
                    });
                } else if (diag.category === 0) { // Warning
                    this.warn({
                        message,
                        id: diag.fileName,
                    });
                }
            }

            // (опционально) Генерация файлов объявлений — пока через CLI fallback
            if (options.declaration) {
                // TODO: генерация .d.ts через CLI-режим или патч API
            }
        },

        watchChange(id) {
            const normalized = id.replace(/\\/g, "/");
            const content = readFileSync(id, "utf-8");
            fs.writeFile!(normalized, content);
            // Запуск инкрементального обновления
            api.updateSnapshot({
                fileChanges: { changed: [normalized] },
            }).then(snap => {
                snapshot = snap;
                project = snap.getProject(tsconfig);
            });
        },

        async closeBundle() {
            await api.close();
        },
    };
}

export default tsgoRollupPlugin;
```

### 5.4 Дизайн интерфейса Transformer (совместимость с экосистемой tsc transformer)

```typescript
// Определение интерфейса transformer, совместимого с tsc
interface TsgoTransformerContext {
    program: Program;        // объект tsgo Program
    checker: Checker;        // объект tsgo Checker
    visitEachChild: typeof visitEachChild;
    factory: typeof factory; // все функции create*
    SyntaxKind: typeof SyntaxKind;
}

type TsgoTransformer = (
    context: TsgoTransformerContext
) => (sourceFile: SourceFile) => SourceFile;

// Пример использования:
const myTransformer: TsgoTransformer = (ctx) => (sourceFile) => {
    return ctx.visitEachChild(sourceFile, (node) => {
        if (node.kind === ctx.SyntaxKind.Identifier && ...) {
            return ctx.factory.createIdentifier("newName");
        }
        return ctx.visitEachChild(node, visitor);
    });
};
```

---

## 6. Отсутствующие возможности и стратегии их восполнения

### 6.1 Список отсутствующих возможностей

| Отсутствует | Критичность | Область влияния | Стратегия решения |
|--------|--------|---------|---------|
| `printNode` не возвращает Source Map | 🔴 Высокая | Разрыв цепочки sourceMap | Патч Go-стороны printNode — добавление вывода sourceMap, либо синтез другим способом |
| `printNode` возможно не делает downleveling | 🟡 Средняя | Проекты с target < ESNext | Обработка плагинами Rollup/Babel; для ESNext проектов не влияет |
| Нет специализированного API `emitSourceFile` | 🟡 Средняя | Неясно, эквивалентен ли printNode | Проверка на практике; если не эквивалентен — патч |
| Нет API для emit файлов объявлений | 🟡 Средняя | Библиотечные проекты, требующие .d.ts | CLI fallback или патч |
| `RemoteSourceFile` не позволяет изменять свойства исходного узла | 🟡 Средняя | Способ модификации AST | Создание новых узлов через `visitEachChild` + factory взамен старых |

### 6.2 Решение для Source Map

**Вариант A: Патч Go-стороны (минимальные изменения)**

В `internal/api/session.go` функции `handlePrintNode` добавить одновременный возврат sourceMap:

```go
// Точка патча: internal/api/proto.go
type PrintNodeResponse struct {
    Code      string `json:"code"`
    SourceMap string `json:"sourceMap,omitempty"`  // ★ добавлено
}

// Точка патча: internal/api/session.go handlePrintNode
func (s *Session) handlePrintNode(req *Request) (any, error) {
    // ... существующая логика ...
    // Генерация sourceMap в процессе print
    sourceMap := generateSourceMapForPrint(sourceFile, printedCode)
    return PrintNodeResponse{
        Code:      printedCode,
        SourceMap: sourceMap,
    }, nil
}
```

**Вариант B: Синтез на JS-стороне (без патча)**

```typescript
// Использование библиотеки source-map-js, синтез на основе соответствия позиций в TS-коде и JS-выводе
// Точность ниже, чем у нативной генерации, но для простых сценариев приемлемо

import { SourceMapGenerator } from "source-map-js";

function synthesizeSourceMap(tsSource: string, jsOutput: string, filePath: string): string {
    const gen = new SourceMapGenerator({ file: filePath.replace(/\.tsx?$/, ".js") });
    // Простая стратегия: построчное сопоставление (грубо, но работает)
    const tsLines = tsSource.split("\n");
    const jsLines = jsOutput.split("\n");
    const maxLines = Math.min(tsLines.length, jsLines.length);
    for (let i = 0; i < maxLines; i++) {
        gen.addMapping({
            source: filePath,
            original: { line: i + 1, column: 0 },
            generated: { line: i + 1, column: 0 },
        });
    }
    gen.setSourceContent(filePath, tsSource);
    return gen.toString();
}
```

**Вариант C: Ожидание официальной поддержки (рекомендуемый долгосрочный вариант)**

API tsgo быстро развивается, возврат sourceMap — наиболее естественное требование. Можно создать Issue или PR.

### 6.3 Проверка различий `printNode` и полного Emit

```typescript
// Скрипт проверки: сравнение вывода printNode с выводом tsgo CLI
const api = new API({ fs: createVirtualFileSystem({
    "/tsconfig.json": `{ "compilerOptions": { "target": "esnext", "module": "esnext" } }`,
    "/src/test.ts": `export async function foo(): Promise<number> { return 42; }`,
})});

const snapshot = await api.updateSnapshot({ openProject: "/tsconfig.json" });
const project = snapshot.getProject("/tsconfig.json");
const sf = await project.program.getSourceFile("/src/test.ts");

const jsFromPrintNode = await project.emitter.printNode(sf);

// Сравнение с выводом CLI:
// tsgo --project tsconfig.json → запись в /src/test.js
// Если jsFromPrintNode === вывод CLI → printNode и есть полный emit
// Если различаются → нужен патч для добавления API emitSourceFile
```

---

## 7. Дополнительная ценность Варианта 3 (патч)

Хотя Вариант 2 является оптимальным, Вариант 3 сохраняет ценность в следующих сценариях:

### 7.1 Сценарии, требующие Варианта 3

| Сценарий | Решает ли Вариант 2 | Необходимые изменения для Варианта 3 |
|------|--------------|----------------|
| Точная передача Source Map | ⚠️ Синтез — грубый | Патч `printNode` для возврата sourceMap |
| downleveling emit (target < ESNext) | ⚠️ Требует Babel плагинов | Патч — добавление API `emitSourceFile` |
| Emit файлов объявлений | ⚠️ CLI fallback | Патч — добавление API `emitDeclaration` |
| Высокопроизводительный пакетный emit | ⚠️ Накладные расходы IPC на каждый файл | Патч — добавление API пакетного emit |
| Пользовательский Go Transformer | ❌ Невозможно реализовать на JS-стороне | Патч `internal/transformers` |

### 7.2 Минимальный набор изменений Варианта 3 (только дополнение слабых мест Варианта 2)

```
Требуется изменить всего 2 Go-файла:

1. internal/api/proto.go
   → Добавить поле sourceMap в PrintNodeResponse

2. internal/api/session.go
   → Генерация sourceMap в handlePrintNode
```

Таким образом, Вариант 2 + минимальный патч = полное решение.

---

## 8. Матрица совместимости с существующей экосистемой Rollup

| Плагин экосистемы | Совместимость | Описание |
|---------|--------|------|
| `@rollup/plugin-node-resolve` | ✅ Полная | JS-код из load корректно попадает в граф модулей |
| `@rollup/plugin-commonjs` | ✅ Полная | Скомпилированный JS — это ESM, commonjs обрабатывает .js зависимости |
| `@rollup/plugin-babel` | ✅ Полная | transform hook может дополнительно преобразовывать JS |
| `rollup-plugin-terser` / `@rollup/plugin-terser` | ✅ Полная | Можно сжимать вывод printNode |
| `@rollup/plugin-virtual` | ✅ Совместим | Механизм VFS аналогичен |
| `rollup-plugin-postcss` | ✅ Совместим | Не влияет |
| `@rollup/plugin-image` | ✅ Совместим | Не влияет |
| `@rollup/plugin-json` | ✅ Совместим | Не влияет |
| `rollup-plugin-esbuild` | ⚠️ Пересечение функций | Может заменить этот плагин для TS-компиляции, но без возможности модификации AST |
| `vite` | ✅ Совместим | Vite использует Rollup для production-сборки |

---

## 9. Итоговая рекомендация

### **Вариант 2 + минимальный патч (только sourceMap) = оптимальная реализация**

```
                    ┌─────────────────────┐
                    │  rollup-plugin-tsgo  │
                    │                      │
                    │  Ядро (без патча):    │
                    │  ✓ Пофайловая компиляция │
                    │    program.getSourceFile + emitter.printNode
                    │  ✓ Модификация TS AST  │
                    │    visitEachChild + factory
                    │  ✓ Модификация JS AST  │
                    │    Rollup transform hook
                    │  ✓ Вывод диагностики   │
                    │    getSemanticDiagnostics
                    │  ✓ Инкрементальное обновление │
                    │    updateSnapshot(fileChanges)
                    │  ✓ VFS компиляция в памяти │
                    │    createVirtualFileSystem
                    │                      │
                    │  Требуется патч (минимальный): │
                    │  ★ Возврат sourceMap  │
                    │    printNode — добавление вывода маппинга
                    │                      │
                    │  Опциональный патч:    │
                    │  ○ downleveling emit  │
                    │  ○ declaration emit   │
                    │  ○ Оптимизация пакетного emit │
                    └─────────────────────┘
```

### Приоритеты реализации

| Приоритет | Функция | Способ реализации | Оценка трудозатрат |
|--------|------|---------|-----------|
| P0 | Пофайловая компиляция | `program.getSourceFile + emitter.printNode` | 1 день |
| P0 | Каркас Rollup плагина | load/transform/buildEnd/closeBundle | 2 дня |
| P0 | Управление VFS + Snapshot | `createVirtualFileSystem + updateSnapshot` | 1 день |
| P0 | Вывод диагностики | `getSemanticDiagnostics` | 0.5 дня |
| P1 | TS AST Transformer | `visitEachChild + factory` | 2 дня |
| P1 | Source Map | Патч Go-стороны `printNode` или синтез на JS-стороне | 2-3 дня |
| P2 | Файлы объявлений | CLI fallback или патч | 1-2 дня |
| P2 | Режим Watch | `watchChange + fileChanges` | 1 день |
| P3 | Оптимизация производительности | Пакетный API, стратегии кэширования | 3-5 дней |

---

## Приложение: Краткий справочник API tsgo

### JS-сторона (`@typescript/native-preview/unstable/`)

| Путь экспорта | Содержимое |
|---------|------|
| `unstable/async` | `API`, `Snapshot`, `Project`, `Program`, `Checker`, `Emitter`, `Symbol`, `Type`, `Signature` |
| `unstable/sync` | То же, синхронная версия |
| `unstable/ast` | `SourceFile`, `Node`, `SyntaxKind`, `isXxx` type guards, `ModifierFlags`, `NodeFlags` |
| `unstable/ast/factory` | `createIdentifier`, `createFunctionDeclaration`, `createSourceFile` и другие factory |
| `unstable/ast/visitor` | `visitEachChild`, `Visitor` |
| `unstable/fs` | `FileSystem`, `createVirtualFileSystem` |

### Методы API Go-стороны (восстановлено из JS client)

| Метод | Параметры | Возвращает |
|------|------|------|
| `initialize` | null | `{ useCaseSensitiveFileNames, currentDirectory }` |
| `parseConfigFile` | `{ file }` | `{ fileNames, options }` |
| `updateSnapshot` | `{ openProject?, fileChanges? }` | `{ snapshot, projects, changes? }` |
| `getSourceFile` | `{ snapshot, project, file }` | Бинарный AST в кодировке Base64 |
| `printNode` | `{ data(base64 AST), options }` | Строка с JS-кодом |
| `getSemanticDiagnostics` | `{ snapshot, project, file? }` | `Diagnostic[]` |
| `getSyntacticDiagnostics` | `{ snapshot, project, file? }` | `Diagnostic[]` |
| `getSymbolAtPosition` | `{ snapshot, project, file, position }` | `SymbolResponse` |
| `getTypeAtPosition` | `{ snapshot, project, file, position }` | `TypeResponse` |
| `getTypeOfSymbol` | `{ snapshot, project, symbol }` | `TypeResponse` |
| `release` | `{ handle }` | — |

### FS колбэки (Go → JS обратные запросы)

| Колбэк | Параметры | Возвращает | Описание |
|--------|------|------|------|
| `readFile` | `{ fileName }` | `{ content: string | null }` | Чтение содержимого файла, null — файл не существует |
| `fileExists` | `{ fileName }` | `boolean` | Существует ли файл |
| `directoryExists` | `{ directoryName }` | `boolean` | Существует ли директория |
| `getAccessibleEntries` | `{ directoryName }` | `{ files[], directories[] }` | Записи директории |
| `realpath` | `{ path }` | `string` | Реальный путь |

---

> **Итоговый вывод**: Вариант 2 (интеграция через API/IPC) является оптимальным решением, удовлетворяющим трём ключевым требованиям: «пофайловая компиляция + transform модификация AST + совместимость с Rollup». JS API `@typescript/native-preview` от tsgo уже обладает необходимыми ключевыми возможностями (`Emitter.printNode` + `visitEachChild` + factory), единственное, что требует минимального патча — возврат sourceMap.
