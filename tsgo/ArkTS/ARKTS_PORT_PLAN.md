# План портирования ArkTS в typescript-go (Scanner / Parser / Binder)

> Исходный код: `D:\Work\our\third_party_typescript` (ohos-typescript, форк TS 4.9.5)
> Целевой код: `D:\Work\our\ms\typescript-go` (Microsoft/typescript-go, Go-реализация TS)
> Объём: только scanner, parser, binder + типы AST. Без checker/emitter/linter.
> Команда: 3 разработчика

---

## 0. Разведка целевого репозитория

**`D:\Work\our\ms\typescript-go`** — нативный порт TypeScript на Go ("Corsa" / TypeScript 7).

### Ключевые файлы

| Модуль | Путь | Размер |
|--------|------|--------|
| Сканер | `internal/scanner/scanner.go` | ~2,882 строки |
| Парсер | `internal/parser/parser.go` | ~6,822 строки |
| Биндер | `internal/binder/binder.go` | ~2,805 строк |
| AST-узлы | `internal/ast/ast.go` | ~3,002 строки (рукописный) |
| AST-узлы | `internal/ast/ast_generated.go` | ~9,884 строки (кодогенерируемый) |
| Flow-граф | `internal/ast/flow.go` | Flow-узлы |
| Символы | `internal/ast/symbol.go` | Symbol/SymbolTable |
| Флаги | `internal/ast/nodeflags.go`, `symbolflags.go`, `tokenflags.go` | Flag-перечисления |
| Коды узлов | `internal/ast/kind_generated.go` | Kind enum (кодогенерируемый) |
| Тесты | `testdata/tests/cases/compiler/` | 257 TS-тестов |
| Базилины | `testdata/baselines/reference/` | 49,155 baseline-файлов |
| Тест-раннер | `internal/testrunner/` | Харнесс для компиляторных тестов |
| Кодоген AST | `_scripts/ast.json`, `_scripts/generate-go-ast.ts` | Схема AST → Go-код |

### Отличия от ohos-typescript

| Аспект | ohos-typescript | typescript-go |
|--------|----------------|---------------|
| Язык | TypeScript | Go |
| Версия TS | 4.9.5 | 7.x (Corsa) |
| AST | Интерфейсы TS | Структуры Go + кодогенерация |
| Фабрики | `factory/nodeFactory.ts` | `NodeFactory` с arena-аллокацией |
| Символы | `Symbol` с битовыми флагами | `ast.Symbol` с битовыми флагами |
| Flow | `FlowNode` union type | `ast.FlowNode` |
| Парсер | Hand-written recursive descent | Hand-written recursive descent + reparser |
| Структура | Монолит: `src/compiler/` | Разделение: `internal/parser/`, `internal/scanner/`, etc. |

---

## 1. Что именно портировать

Сводка из `ARKTS_SCANNER_PARSER_BINDER.md` (1023 строки):

### 1.1. Сканер (~15 строк логики)

- [ ] `struct` → `StructKeyword`, `lazy` → `LazyKeyword` (в `textToKeyword`)
- [ ] `inEtsContext: bool` + метод `SetEtsContext(bool)`
- [ ] Условная отмена: `if keyword == StructKeyword && !inEtsContext → Identifier`

### 1.2. AST / Типы (~200 строк)

**Новые Kind (SyntaxKind):**
- [ ] `KindStructKeyword` — ключевое слово struct
- [ ] `KindLazyKeyword` — ключевое слово lazy
- [ ] `KindStructDeclaration` — объявление struct
- [ ] `KindAnnotationDeclaration` — объявление @interface
- [ ] `KindAnnotationPropertyDeclaration` — свойство аннотации
- [ ] `KindEtsComponentExpression` — выражение UI-компонента

**Новые NodeFlags:**
- [ ] `NodeFlagsEtsContext` (1 << 30) — файл в режиме ArkTS
- [ ] `NodeFlagsKitImport` (1 << 29) — узлы kit-импортов

**Новые SymbolFlags:**
- [ ] `SymbolFlagsAnnotation` (1 << 28)

**EtsFlags (13 флагов):**
- [ ] `EtsFlagsNone`, `StructContext`, `EtsExtendComponentsContext`, `EtsStylesComponentsContext`, `EtsBuildContext`, `EtsBuilderContext`, `EtsStateStylesContext`, `EtsComponentsContext`, `EtsNewExpressionContext`, `UICallbackContext`, `SyntaxComponentContext`, `SyntaxDataSourceContext`, `NoEtsComponentContext`

**Новые AST-интерфейсы в Go (структуры + кодогенерация):**
- [ ] `StructDeclaration` — extends `ClassLikeBase`, `Declaration`
- [ ] `AnnotationDeclaration` — extends `Declaration`
- [ ] `AnnotationPropertyDeclaration` — extends `AnnotationElement`
- [ ] `EtsComponentExpression` — extends `Expression`, `Declaration`
- [ ] `AnnotationElement` — базовый тип для элементов аннотации

**CompilerOptions:**
- [ ] `ets: EtsOptions` (структура конфигурации)
- [ ] `etsLoaderPath: string`
- [ ] `etsAnnotationsEnable: bool`
- [ ] `strictCheckerOnly: bool`
- [ ] `tsImportSendableEnable: bool`

**ScriptKind / Extension:**
- [ ] `ScriptKindETS` (значение 8)
- [ ] `ExtensionEts = ".ets"`, `ExtensionDets = ".d.ets"`

### 1.3. Парсер (~600 строк нового кода + 36 модификаций)

**Система EtsFlags:**
- [ ] 12 `Set*Context(bool)` + 14 `In*Context() bool`
- [ ] Вход в ETS-режим в `initializeState` / `ParseSourceFile`
- [ ] Очистка в `clearState`

**Новые функции (~19 шт.):**
- [ ] `parseStructDeclaration` + `parseStructDeclarationOrExpression`
- [ ] `parseStructMembers` + виртуальный конструктор
- [ ] `createVirtualHeritageClauses`
- [ ] `finishVirtualNode`
- [ ] `parseAnnotationDeclaration` + `parseAnnotationPropertyDeclaration` + `parseAnnotationElement` + `parseAnnotationMembers`
- [ ] `isAnnotationMemberStart`
- [ ] `parseEtsComponentExpression` + `isCurrentTokenAnEtsComponentExpression` + `makeEtsComponentExpression`
- [ ] `parseEtsIdentifier`, `parseEtsType`, `parseEtsTypeParameters`, `parseEtsTypeArguments`
- [ ] `isTokenInsideStructBuild`, `isTokenInsideStructBuilder`, `isTokenInsideStructPageTransition`
- [ ] `tryParseConstructorDeclaration`, `parseConstructorName`
- [ ] `hasParamAndNoOnceDecorator`, `hasEnvDecorator`
- [ ] `isValidExtendOrStylesContext`, `isValidVirtualTypeArgumentsContext`

**Модификации (~36 мест):**
- [ ] `doInDecoratorContext` — детект @Extend/@Styles
- [ ] `parseStatement` — `StructKeyword` → `parseStructDeclaration`
- [ ] `parseDeclaration` — @Extend/@Styles/@Builder контекст
- [ ] `parseDeclarationWorker` — struct + @interface роутинг
- [ ] `parseFunctionDeclaration` — ETS-контекст, виртуальные типы возврата
- [ ] `parseMethodDeclaration` — build/builder/styles контекст
- [ ] `parseClassElement` — auto-readonly для @Param/@Env
- [ ] `parseModifiers` — инжекция виртуального readonly
- [ ] `parseCallExpressionRest` — виртуальные type arguments для компонентов
- [ ] `parsePrimaryExpression` — вызов `parseEtsComponentExpression`
- [ ] `parseAssignmentExpressionOrHigher` — CallExpression+`{` → EtsComponentExpression
- [ ] `parseNewExpressionOrNewDotTarget` — EtsNewExpressionContext
- [ ] `parseArgumentExpression` — SyntaxDataSourceContext
- [ ] `parseArrowFunctionExpressionBody` — UICallback + struct ASI
- [ ] `parseLeftHandSideExpressionOrHigher` / `parseMemberExpressionOrHigher` — виртуальные Extend/Styles идентификаторы
- [ ] `parseExpected` — обход stateStyles
- [ ] `parseIdentifier` — виртуальный stateStyles идентификатор
- [ ] `tryParseDecorator` — пропуск @interface
- [ ] `canFollowModifier` — @interface после модификатора
- [ ] `isStartOf*` — ~6 функций: StructKeyword валидный старт
- [ ] `isDeclaration` — struct + @interface
- [ ] `nextTokenCanFollowDefaultKeyword` — `export default struct`
- [ ] `setLanguageVersionByFilePath` — ETS-версия
- [ ] 3 `forEachChild`-функции для новых узлов

### 1.4. Биндер (~90 строк)

- [ ] `StructDeclaration` как `IsContainer` + `bindClassLikeDeclaration`
- [ ] `AnnotationDeclaration` как `IsContainer` + `bindAnnotationDeclaration`
- [ ] `AnnotationPropertyDeclaration` как `ControlFlowContainer` (если есть initializer)
- [ ] `bindPropertyWorker`: пропуск `?` optionality для AnnotationProperty; optional по initializer

### 1.5. AST-кодогенерация (~100 строк схемы)

- [ ] Добавить новые Kind в `_scripts/ast.json`
- [ ] Добавить новые структуры узлов в схему
- [ ] Сгенерировать `ast_generated.go` и `kind_generated.go`

---

## 2. Стратегия тестирования

Поскольку checker и emitter не меняются (их будет делать другая команда), baseline-тесты на полный цикл компиляции не подходят. Вместо этого:

### 2.1. Парсер-раундтрип (основной метод)

```
.ets исходник → ParseSourceFile(ETS) → AST → Print AST → ParseSourceFile(ETS) → AST'
Сравнить AST == AST'
```

**Что покрывает:**
- Scanner: корректная токенизация `.ets` файлов
- Parser: корректная структура AST
- Все новые конструкции: struct, @interface, EtsComponentExpression

**Что НЕ покрывает:** правильность контекстов (EtsFlags), виртуальные узлы.

### 2.2. Сканнер-тесты (последовательность токенов)

```
.ets исходник → ScanTokens → [TokenKind, TokenKind, ...]
Сравнить с ожидаемой последовательностью
```

**Что покрывает:**
- `struct` в ETS-контексте → StructKeyword
- `struct` в TS-контексте → Identifier
- `lazy` → LazyKeyword
- Все остальные токены

### 2.3. Сравнительные AST-тесты (ohos-typescript как оракул)

```
.ets исходник → [ohos-typescript: ParseSourceFile] → AST_json
.ets исходник → [typescript-go: ParseSourceFile] → AST_json
Сравнить AST_json == AST_json
```

**Преимущества:** эталонная реализация как оракул, ловит любые расхождения в структуре AST.

**Недостатки:** требует запуска ohos-typescript (Node.js) из Go-тестов.

### 2.4. Юнит-тесты на конкретные конструкции

Для каждой новой фичи:
```go
func TestStructDeclaration(t *testing.T) {
    source := `@Component struct Test { @State count: number = 0; build() {} }`
    file := ParseSourceFile(source, ScriptKindETS)
    // Проверить: есть StructDeclaration, есть виртуальный конструктор, 
    // есть @State свойство, есть build метод
}
```

### 2.5. Контекст-тесты (EtsFlags)

Специфические тесты на правильность установки/сброса контекстов:
```go
func TestBuilderContext(t *testing.T) {
    // Внутри @Builder функции → EtsBuilderContext = true
    // Вне @Builder → EtsBuilderContext = false
    // EtsComponentExpression разрешён только внутри build/builder
}
```

### 2.6. Негативные тесты (ошибки парсинга)

```go
func TestStructOutsideETS(t *testing.T) {
    source := `struct Foo {}`  // TS-контекст, struct = Identifier
    file := ParseSourceFile(source, ScriptKindTS)
    // Должен распарситься как VariableStatement (struct — идентификатор)
}
```

---

## 3. Разделение работы на троих

### Архитектура зависимостей

```
Фаза 1: Фундамент (все вместе, 3-5 дней)
├── AST-типы + Kind + флаги + кодогенерация
├── EtsOptions (конфигурация)
└── Scanner (struct + lazy + EtsContext)

Фаза 2: Параллельная разработка (10-15 дней)
├── Dev A: Парсер — Декларации
├── Dev B: Парсер — Выражения + Контекст
└── Dev C: Биндер + Тестовая инфраструктура

Фаза 3: Интеграция + Тесты (5-10 дней)
├── Интеграция треков A и B
├── Раундтрип-тесты
├── Сравнительные тесты
└── Баг-фиксы
```

### Детальный план по разработчикам

#### Фаза 1: Фундамент (3-5 дней, вместе)

**Все вместе (синхронно):**
1. Согласовать naming convention для ArkTS-дополнений в Go (префиксы, имена констант)
2. Обновить `_scripts/ast.json` — схему AST с новыми узлами
3. Сгенерировать `ast_generated.go` и `kind_generated.go`
4. Добавить новые Kind, NodeFlags, SymbolFlags, TokenFlags, ScriptKind, Extension
5. Реализовать `EtsFlags` + EtsOptions структуры
6. Реализовать сканер (`struct`, `lazy`, `SetEtsContext`)

**Результат:** Можно отсканировать `.ets` файл, `struct` и `lazy` распознаются как ключевые слова.

**Критические решения:**
- Как интегрировать ETS-режим в `ParseSourceFile` (новый параметр? расширение файла? ScriptKind?)
- Именование: `EtsFlags` или `ArkTSFlags`? `StructDeclaration` или `EtsStructDeclaration`?

#### Фаза 2: Параллельная разработка (10-15 дней)

##### Dev A: Парсер — Декларации

**Файлы:** `internal/parser/parser.go`

**Задачи:**
1. Система EtsFlags (12 Set/14 In функций) — **2 дня**
2. `parseStructDeclaration` + `parseStructDeclarationOrExpression` + `parseStructMembers` + виртуальный конструктор — **3-4 дня**
3. `createVirtualHeritageClauses` + `finishVirtualNode` — **1 день**
4. `parseAnnotationDeclaration` + `parseAnnotationPropertyDeclaration` + `parseAnnotationElement` + `parseAnnotationMembers` + `isAnnotationMemberStart` — **3-4 дня**
5. Struct/Builder хелперы (`isTokenInsideStructBuild`, `isTokenInsideStructPageTransition`, `tryParseConstructorDeclaration`, `parseConstructorName`) — **1 день**
6. `doInDecoratorContext` — детект @Extend/@Styles — **1 день**
7. `parseDeclaration` / `parseDeclarationWorker` — struct + @interface роутинг — **1 день**
8. `parseFunctionDeclaration` — Builder/Styles/Extend контекст — **2 дня**
9. `parseMethodDeclaration` — build/builder/styles контекст + auto-readonly — **2 дня**
10. `parseClassElement` / `parseModifiers` — @Param/@Env auto-readonly — **1 день**
11. Токен-уровневые модификации (`isStartOf*`, `isDeclaration`, `nextTokenCanFollowDefaultKeyword`, etc.) — **1 день**

**Итого: 18-21 день**

##### Dev B: Парсер — Выражения + Контекст

**Файлы:** `internal/parser/parser.go`

**Задачи:**
1. `parseEtsComponentExpression` + `isCurrentTokenAnEtsComponentExpression` + `makeEtsComponentExpression` — **2-3 дня**
2. `isValidExtendOrStylesContext` + `isValidVirtualTypeArgumentsContext` — **1 день**
3. `parseEtsIdentifier` + `parseEtsType` + `parseEtsTypeParameters` + `parseEtsTypeArguments` — **2 дня**
4. `parseCallExpressionRest` — виртуальные type arguments (самый сложный участок) — **5-7 дней**
5. `parseLeftHandSideExpressionOrHigher` / `parseMemberExpressionOrHigher` — виртуальные Extend/Styles идентификаторы — **2 дня**
6. `parseArgumentExpression` — SyntaxDataSourceContext — **2 дня**
7. `parseArrowFunctionExpressionBody` / `tryParseParenthesizedArrowFunctionExpression` — UICallback — **2 дня**
8. `parseExpected` / `parseIdentifier` — stateStyles обходы — **1 день**
9. `parseNewExpressionOrNewDotTarget` — EtsNewExpressionContext — **0.5 дня**
10. `tryParseDecorator` — пропуск @interface — **0.5 дня**
11. 3 `forEachChild`-функции — **1 день**
12. `setLanguageVersionByFilePath` — **0.5 дня**

**Итого: 18-23 дня**

##### Dev C: Биндер + Тестовая инфраструктура + Подготовка `.ets` тестов

**Задачи (Биндер):**
1. `StructDeclaration` как `IsContainer` + `bindClassLikeDeclaration` — **1 день**
2. `AnnotationDeclaration` как `IsContainer` + `bindAnnotationDeclaration` — **1 день**
3. `AnnotationPropertyDeclaration` как `ControlFlowContainer` — **0.5 дня**
4. `bindPropertyWorker` модификации — **0.5 дня**

**Задачи (Тестовая инфраструктура):**
5. Создать тестовый runner для `.ets` файлов — **3 дня**
6. Написать раундтрип-тесты для парсера — **3 дня**
7. Подготовить тестовые `.ets` файлы из `tests/arkTSTest/testcase/` и `tests/dets/cases/` — **2 дня**
8. Сканнер-тесты (последовательности токенов) — **2 дня**

**Задачи (помощь Dev A/B):**
9. Помогать Dev A или B с менее критичными функциями по мере завершения биндера — **5-7 дней**

**Итого: 18-21 день**

#### Фаза 3: Интеграция и тестирование (5-10 дней, вместе)

1. **Слияние треков A и B** — 2-3 дня
   - Разрешение конфликтов в общих функциях (`initializeState`, `clearState`, `parseSourceFileWorker`)
   - Проверка, что struct внутри EtsComponentExpression корректно парсится
2. **Раундтрип-тесты на 50+ `.ets` файлов** — 2-3 дня
3. **Сравнительные тесты (ohos-typescript как оракул)** — 2-3 дня
4. **Баг-фиксы и краевые случаи** — 3-5 дней

### Итоговая временная оценка

| Фаза | Календарных недель | Суммарно человеко-дней |
|------|-------------------|----------------------|
| Фаза 1: Фундамент | 1 | 15 |
| Фаза 2: Параллельно | 3-4 | 54-65 |
| Фаза 3: Интеграция | 1-2 | 21-30 |
| **Итого** | **5-7 недель** | **90-110** |

---

## 4. Стратегия тестирования — детально

### 4.1. Что можем тестировать БЕЗ checker/emitter

| Тип теста | Что проверяем | Инструмент |
|-----------|--------------|------------|
| **Сканнер-тест** | Правильная токенизация `.ets` | Самодельный test runner: `ScanFile(path) → []Token` |
| **Парсер-раундтрип** | AST → Print → AST' идентичны | `ParseFile → Print → ParseFile → DeepEqual(AST, AST')` |
| **AST-структура** | Конкретные узлы на месте | Юнит-тесты: `ParseSourceFile → assert IsStructDeclaration(node)` |
| **Ошибки парсинга** | Корректные диагностики | `ParseSourceFile → assert diagnostics.Len() > 0` |
| **Биндер-тест** | Символы созданы правильно | `Bind → assert symbol.Flags & SymbolFlagsClass` |
| **Flow-граф** | Flow-узлы созданы для struct | `Bind → assert len(file.FlowNode) > 0` |
| **Сравнительный AST** | AST совпадает с ohos-typescript | `diff <(ohos-parse) <(tsgo-parse)` |

### 4.2. Источники тестовых данных

1. `D:\Work\our\third_party_typescript\tests\arkTSTest\testcase\` — 110+ директорий с `-ok.ets` и `-error.ets` файлами
2. `D:\Work\our\third_party_typescript\tests\dets\cases\` — `.ets` файлы с декораторами
3. `D:\Work\our\third_party_typescript\tests\cases\conformance\parser\ets\annotations\` — тесты аннотаций
4. Собственноручно написанные `.ets` тесты для краевых случаев

### 4.3. Что НЕ сможем протестировать

| Ограничение | Причина | Когда решится |
|------------|--------|--------------|
| Правильность типов struct | Нужен checker | Когда команда сделает checker |
| `@throws` проверки | Нужен checker | Когда команда сделает checker |
| `apiAvailable` проверки | Нужен checker + host callbacks | Когда команда сделает checker |
| Sendable-правила | Нужен checker + линтер | Когда команда сделает checker/линтер |
| Сообщения об ошибках (точный текст) | Нужен checker | Когда команда сделает checker |
| Kit-трансформация | Нужен processKit (ohApi.ts) | Отдельная задача (зависит от SDK JSON) |
| `oh_modules` резолвинг | Нужен module resolver | Отдельная задача |

---

## 5. Риски и митигация

| Риск | Вероятность | Влияние | Митигация |
|------|------------|---------|-----------|
| Разное внутреннее устройство парсеров (TS vs Go) | Высокая | Задержка 3-5 дней | Dev A и Dev B начинают с изучения Go-парсера; Dev C делает тестовую инфраструктуру |
| Конфликты в общей зоне парсера (EtsFlags, init/clear) | Средняя | Переписывание 2-3 дня | Совместная реализация EtsFlags-системы в Фазе 1 |
| Кодогенерация AST требует переписывания схемы | Средняя | Задержка 2-3 дня | Выделить 1 день в Фазе 1 на изучение `_scripts/ast.json` |
| Отсутствие эталонного оракула для AST | Низкая | Ручная верификация | ohos-typescript как оракул (запуск через Node.js из Go-тестов) |
| Dev B зависит от EtsFlags (делает Dev A) | Высокая | Простой 2-3 дня | EtsFlags реализуются совместно в Фазе 1 |
| processKit слишком сложен и не нужен сейчас | Средняя | Лишняя работа | Отложить processKit; kit-импорты парсить как обычные импорты |

---

## 6. Что можно отложить (не делать сейчас)

| Задача | Причина отсрочки | Кто сделает |
|--------|-----------------|-------------|
| `processKit` (~440 строк) | Нужен SDK JSON, сложная логика, не нужна без checker | Отдельная задача позже |
| `oh_modules` резолвинг | Нужен module resolver, не нужен без checker | Команда checker |
| `markedKitImportRange` | Зависит от processKit | Вместе с processKit |
| `SourceFile.endFlowNode` / `returnFlowNode` для struct | Flow-анализ — зона checker | Команда checker |
| `CheckMode.SkipEtsComponentBody` | Используется ТОЛЬКО в checker | Команда checker |
| ETS-хелперы из `ohApi.ts` (кроме тех, что нужны парсеру) | 1915 строк кода, большинство для checker/emitter | По мере необходимости |

### Минимальный набор `ohApi.ts`-функций для парсера:

Только те, что вызываются напрямую из парсера:
- [ ] `hasEtsExtendDecoratorNames`
- [ ] `hasEtsStylesDecoratorNames`
- [ ] `hasEtsBuildDecoratorNames`
- [ ] `hasEtsBuilderDecoratorNames`
- [ ] `getEtsExtendDecoratorsComponentNames`
- [ ] `getEtsStylesDecoratorComponentNames`
- [ ] `isTokenInsideBuilder`
- [ ] `isInEtsFile`

Всё остальное из ohApi.ts — на будущее.

---

## 7. Порядок выполнения (граф зависимостей)

```
                      ┌──────────────────┐
                      │ AST schema update │
                      │ (ast.json + gen)  │
                      └────────┬─────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
   ┌─────────────┐     ┌─────────────┐     ┌──────────────┐
   │   Scanner   │     │  EtsFlags   │     │  EtsOptions   │
   │ (3 строки)  │     │ (enum +     │     │  (структуры)  │
   │             │     │  14 funcs)  │     │               │
   └──────┬──────┘     └──────┬──────┘     └──────┬────────┘
          │                   │                   │
          └───────────────────┼───────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│  ДЕКЛАРАЦИИ (A)  │ │  ВЫРАЖЕНИЯ (B)   │ │  БИНДЕР + ТЕСТЫ  │
│  struct           │ │  EtsComponent    │ │  (C)             │
│  @interface       │ │  Virtual types   │ │  bindStruct      │
│  parseFunction    │ │  @Extend/Styles  │ │  bindAnnot       │
│  parseMethod      │ │  stateStyles     │ │  Test infra      │
│  auto-readonly    │ │  UICallback      │ │                  │
└────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              │
                     ┌────────▼────────┐
                     │   ИНТЕГРАЦИЯ    │
                     │   + ТЕСТЫ       │
                     └─────────────────┘
```
