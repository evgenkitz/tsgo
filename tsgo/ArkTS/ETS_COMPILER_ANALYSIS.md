# Анализ изменений компилятора TypeScript для поддержки ArkTS + ArkUI

**Репозиторий:** ohos-typescript 4.9.5-r4 (форк Microsoft TypeScript 4.9.5)
**Назначение:** Добавление поддержки ETS (Extensible TypeScript) / ArkTS / ArkUI для платформы OpenHarmony/HarmonyOS

---

## Обзор архитектуры

Компилятор TypeScript имеет 6-этапный конвейер:

```
Исходный код (.ts/.ets)
  → Scanner (сканирование токенов)
  → Parser (построение AST)
  → Binder (связывание символов)
  → Checker (проверка типов)
  → Transformer (трансформация AST)
  → Emitter (генерация JS/.d.ts)
```

Модификации для ArkTS/ArkUI добавлены на **каждом** этапе конвейера. Ключевой файл — `src/compiler/ohApi.ts` (1915 строк), содержащий основную логику ETS.

---

## 1. Scanner (Лексический анализатор)

**Файл:** `src/compiler/scanner.ts`
**Характер изменений:** Минимальные, точечные

### Добавлено:

| Изменение | Строки | Описание |
|-----------|--------|----------|
| `setEtsContext(isEtsContext: boolean)` | 109, 1044, 1059-1061 | Новый метод интерфейса сканера для включения/выключения ETS-режима |
| `struct` как ключевое слово | 176 | Добавлено в `textToKeywordObj` → `SyntaxKind.StructKeyword` |
| `lazy` как ключевое слово | 185 | Добавлено в `textToKeywordObj` → `SyntaxKind.LazyKeyword` (для `import lazy X from "..."`) |
| `inEtsContext` флаг | 996 | Внутренняя переменная состояния, по умолчанию `false` |
| Условное распознавание `struct` | 1587-1589 | **Критическая логика:** `struct` распознаётся как ключевое слово только когда `inEtsContext === true`. В обычных `.ts` файлах `struct` — обычный идентификатор. |

**Gating:** Только `struct` имеет ETS-контекстное gating (`!inEtsContext`). Ключевое слово `lazy` распознаётся **безусловно** — синтаксис `import lazy X from "..."` не существует в стандартном TypeScript, поэтому коллизий не возникает.

### Затрагиваемые этапы конвейера:
Scanner — только этап сканирования. Влияет на все последующие этапы через токены.

---

## 2. Parser (Синтаксический анализатор)

**Файл:** `src/compiler/parser.ts`
**Характер изменений:** Очень существенные (~800+ строк ETS-специфичного кода)

### 2.1 Новые AST-узлы (определены в `types.ts`)

```typescript
SyntaxKind.StructKeyword = 140              // ключевое слово struct (ETS-добавление, с gating)
SyntaxKind.LazyKeyword = 213                 // ключевое слово lazy (ETS-добавление)
SyntaxKind.AnnotationPropertyDeclaration = 235  // свойство внутри @interface
SyntaxKind.EtsComponentExpression = 286     // Column() { ... }
SyntaxKind.StructDeclaration = 334          // struct MyComponent { ... }
SyntaxKind.AnnotationDeclaration = 335      // @interface MyAnnotation { ... }
```

### 2.2 Система ETS-флагов контекста (`EtsFlags`)

12-битовая маска для отслеживания контекста парсинга:

| Флаг | Бит | Назначение |
|------|-----|------------|
| `StructContext` | 1 | Внутри объявления struct |
| `EtsExtendComponentsContext` | 2 | Внутри @Extend декоратора |
| `EtsStylesComponentsContext` | 3 | Внутри @Styles декоратора |
| `EtsBuildContext` | 4 | Внутри метода build() |
| `EtsBuilderContext` | 5 | Внутри @Builder функции |
| `EtsStateStylesContext` | 6 | Внутри stateStyles |
| `EtsComponentsContext` | 7 | В контексте компонентов (build/builder) |
| `EtsNewExpressionContext` | 8 | Внутри new-выражения |
| `UICallbackContext` | 9 | Внутри UI callback (стрелочная функция в build/builder) |
| `SyntaxComponentContext` | 10 | Внутри ForEach/Repeat компонента |
| `SyntaxDataSourceContext` | 11 | Внутри первого аргумента ForEach |
| `NoEtsComponentContext` | 12 | Запрет создания ETS-компонентов |

### 2.3 Парсинг `struct`

**Основная функция:** `parseStructDeclaration()` (строка 8670)

```
1. parseExpected(StructKeyword)      // съедает ключевое слово struct
2. setStructContext(true)            // устанавливает контекст
3. parseIdentifier()                 // имя структуры
4. parseTypeParameters()             // дженерики
5. parseHeritageClauses()            // наследование
6. parseStructMembers()              // члены структуры (как члены класса)
7. factory.createStructDeclaration() // создание узла
```

**Виртуальное наследование (`createVirtualHeritageClauses`, строка 8732):**
Если struct не имеет явного наследования, при включённой опции `customComponent` создаётся виртуальный `extends CustomComponent`.

**Виртуальный конструктор (`parseStructMembers`, строка 8806):**
Для каждой struct автоматически создаётся виртуальный конструктор:
- Все PropertyDeclaration становятся опциональными параметрами конструктора
- Добавляется `##storage?: LocalStorage` для поддержки LocalStorage
- Конструктор вставляется в начало списка членов

### 2.4 Парсинг аннотаций (`@interface`)

**Основная функция:** `parseAnnotationDeclaration()` (строка 7706)

Аннотации парсятся когда `@` стоит перед `interface`:
```
@interface MyAnnotation { ... }
```

**Валидация членов** (`isAnnotationMemberStart`, строка 8256):
- Статические блоки запрещены
- Геттеры/сеттеры запрещены
- Конструкторы запрещены
- Индексные сигнатуры запрещены
- Вычисляемые имена свойств запрещены
- Разрешены: `имя: тип` или `имя: тип = значение`

### 2.5 Парсинг ETS-компонентов

**Основная функция:** `parseEtsComponentExpression()` (строка 6873)

```
Text("Hello") {           → EtsComponentExpression
  .fontSize(20)             expression: Text
  .onClick(() => { ... })   arguments: ["Hello"]
}                           body: Block { .fontSize(20), .onClick(...) }
```

**Интеграция в парсинг выражений:**
- В `parsePrimaryExpression()` (строка 6928): если текущий токен — имя компонента и мы в ETS-контексте, вызывается `parseEtsComponentExpression()`
- В `parseAssignmentExpressionOrHigher()` (строка 5342): если CallExpression с последующей `{`, преобразуется в `EtsComponentExpression`
- В `parseNewExpressionOrNewDotTarget()` (строка 7098): `new` предотвращает интерпретацию как компонент

### 2.6 Парсинг ETS-декораторов

**В `parseDeclaration()` (строка 7731):**
- `@Extend(ComponentName)` → устанавливает `EtsExtendComponentsContext` и находит компонент в `compilerOptions.ets.extend.components`
- `@Styles` → устанавливает `EtsStylesComponentsContext`
- `@Builder` → устанавливает `EtsBuilderContext` и `UICallbackContext`

**В `parseFunctionDeclaration()` (строка 8012):**
- `@Styles`-функции регистрируются в `fileStylesComponents` Map
- `@Builder`-функции получают `EtsBuilderContext`
- Для @Extend/@Styles функций без явного возвращаемого типа создаётся виртуальный тип возврата

**В `parseMethodDeclaration()` (строка 8117):**
- Метод `build()` (или сконфигурированное имя) → `EtsBuildContext`
- `@Builder` метод → `EtsBuilderContext`
- `@Styles` метод в struct → регистрируется в `structStylesComponents`

### 2.7 Обработка `@kit.*` импортов

**В `parseSourceFileWorker()` (строка 1835-1841):**
`processKit()` вызывается на этапе парсинга для трансформации `import ... from "@kit.ModuleName"` в реальные пути модулей, используя JSON-конфигурацию из SDK (`ets/build-tools/ets-loader/kit_configs/`).

### 2.8 Виртуальные идентификаторы для Extend/Styles/stateStyles

В `parseLeftHandSideExpressionOrHigher()` (строка 6237-6248):
- В контексте `@Extend` создаётся виртуальный идентификатор `ИмяКомпонентаInstance`
- В контексте `@Styles` создаётся виртуальный идентификатор для экземпляра компонента
- В контексте `stateStyles` создаётся `${rootNode}Instance`

### Затрагиваемые этапы конвейера:
Parser напрямую, и косвенно все последующие этапы через новые AST-узлы.

---

## 3. Binder (Связывание символов)

**Файл:** `src/compiler/binder.ts`
**Характер изменений:** Минимальные — ETS-конструкции связываются аналогично стандартным TypeScript-конструкциям.

### Добавлено:

| Изменение | Строки | Описание |
|-----------|--------|----------|
| `StructDeclaration` как контейнер | 2141 | Возвращает `ContainerFlags.IsContainer` — struct создаёт новую область видимости |
| `StructDeclaration` в `bindClassLikeDeclaration` | 3565, 2991 | Связывается с `SymbolFlags.Class` — идентично классу |
| `AnnotationDeclaration` как контейнер | 2142 | Возвращает `ContainerFlags.IsContainer` |
| `bindAnnotationDeclaration()` | 3600-3615 | Новая функция: связывает с `SymbolFlags.Class \| SymbolFlags.Annotation`. Создаёт prototype-символ как у класса |
| `AnnotationPropertyDeclaration` | 3057-3064 | Пропускает `?` optionality (аннотации не поддерживают `?`); свойства со значением по умолчанию получают `SymbolFlags.Optional` |
| `SymbolFlags.Annotation = 1 << 28` | types.ts:5539 | Новый флаг символа для аннотаций |

### Затрагиваемые этапы конвейера:
Binder напрямую, Checker и Emitter косвенно (через символы и флаги).

---

## 4. Checker (Проверка типов)

**Файл:** `src/compiler/checker.ts`
**Характер изменений:** Очень существенные — наиболее объёмный этап с точки зрения ETS-логики

### 4.1 Проверка StructDeclaration

- **`checkStructDeclaration()`** (строка 43211): проверяется идентично классу (`checkClassLikeDeclaration`) + `checkStructName()`
- **`checkStructName()`** (строка 43222): имя struct не должно совпадать с зарезервированными именами системных компонентов из `compilerOptions.ets.components`
- **Совместимость вызова:** struct можно вызывать как функцию (без `new`) — `resolveCallExpression` преобразует вызов struct в вызов конструктора (строка 33232)

### 4.2 Проверка EtsComponentExpression

- **`SkipEtsComponentBody`** (`CheckMode`, бит 7): специальный режим проверки, который пропускает тело компонента при разрешении сигнатур (для производительности)
- **Контекстная типизация** (строка 29143): в ETS-файлах аргументы импорта получают контекстный тип `string`
- **Проверка размещения UI-компонентов** (строка 34069): системные UI-компоненты можно использовать только внутри `build()`, `pageTransition()` или `@Builder`-функций. При нарушении: _"UI component {0} cannot be used in this place"_
- **Рекурсивный обход тела компонента** (`traverseEtsComponentStatements`, строка 37814): обходит if-выражения в теле компонента

### 4.3 Проверка аннотаций

**Полный цикл проверки аннотаций:**

1. **`checkAnnotationDeclaration()`** (строка 43236):
   - Запрет аннотаций в HAR, компилируемых в JS
   - Декораторы не валидны на аннотациях
   - Проверка коллизий имён
   - Проверка свойств аннотации

2. **`checkGrammarAnnotationDeclaration()`** (строка 43269):
   - Аннотации разрешены только на верхнем уровне

3. **`resolveAnnotation()`** (строка 33649):
   - `@Anno` → сигнатура по умолчанию
   - `@Anno()` → сигнатура по умолчанию
   - `@Anno({...})` → проверка каждого свойства как константного выражения
   - Проверка, что все поля без значений по умолчанию предоставлены

4. **`checkAnnotation()`** (строка 40252):
   - Аннотации разрешены только на классах и методах (не на конструкторах, геттерах/сеттерах)
   - Запрет на абстрактных классах/методах
   - Проверка дубликатов
   - Поддержка SourceRetention аннотаций на StructDeclaration

5. **Валидация типов свойств** (`isAllowedAnnotationPropertyType`, строка 38341):
   - Только: `number`, `boolean`, `string`, const enum типы, или массивы вышеперечисленного

### 4.4 ETS-декораторы в проверке типов

- **`isEtsFunctionDecorators()`** (строка 32538, 32570, 33622): определяет, что @Builder/@Extend/@Styles — валидные ETS-декораторы, даже если они нарушают стандартные правила TS-декораторов
- **@Builder в `checkGrammarDecorators`** (строка 47811): в ETS-файлах @Builder на не-конструкторе не вызывает ошибок
- **@Extend возвращаемый тип** (строка 35641): запрет явного указания возвращаемого типа на @Extend-функциях
- **@Styles возвращаемый тип** (строка 35701): запрет явного указания возвращаемого типа на @Styles-методах

### 4.5 Декораторы свойств (State Management)

- **Проверка инициализации** (строка 31290-31318): свойства с `@Link`, `@Prop`, `@ObjectLink`, `@Consume` (из `compilerOptions.ets.propertyDecorators`) не требуют инициализации
- **@Require и позиция объявления** (строка 31317): проверка через `checkStructPropertyPosition()`, что @Require-свойство объявлено до использования

### 4.6 Системные компоненты

- **`isSystemEtsComponent()`** (строка 34315): определяет, является ли идентификатор системным UI-компонентом:
  1. Проверка имени в `compilerOptions.ets.components`
  2. Проверка, что символ происходит из SDK `ets-loader/declarations/`
- **Проверка размещения** (строка 34069): системные компоненты только в build/pageTransition/@Builder

### 4.7 Sendable-проверки

- **`isSendableFunctionOrType()`** (строка 679): подавление ошибок валидации декораторов для @Sendable функций/типов
- **`allowImportSendable()`** (строка 5138): контроль импорта Sendable в .d.ts внутри SDK или при включённом `tsImportSendableEnable`
- **Импорт ArkTS из TS** (строка 4952): при `needDoArkTsLinter` импорт `.ets`/`.d.ets` из `.ts` файлов запрещён или предупреждается

### 4.8 Разрешение свойств в ETS-компонентах

- **`getEtsComponentExpressionPropertyOfType()`** (строка 34415): при доступе к свойствам внутри компонента ищет `@Extend` и `@Styles` декораторы
- **Резолвинг через локальные символы** (строка 31094): проверка локальных символов и членов struct на наличие @Styles-декорированных свойств

### Затрагиваемые этапы конвейера:
Checker напрямую.

---

## 5. Transformer (Трансформация AST)

**Файлы:** `src/compiler/ohApi.ts` (основные трансформеры), `src/compiler/transformer.ts` (стандартный конвейер)

**Характер изменений:** Стандартный `transformer.ts` НЕ содержит ETS-кода. ETS-трансформеры определены в `ohApi.ts` и применяются как **customTransformers** внешними инструментами (главным образом `developtools_ace_ets2bundle`).

### 5.1 Annotation Transformer (`getAnnotationTransformer`, строка 516)

Двухпроходная трансформация:

**Проход 1 — Объявления аннотаций:**
- Переименование: `@interface Anno` → `@interface __$$ETS_ANNOTATION$$__Anno`
- Добавление явных аннотаций типов к свойствам
- Вычисление и встраивание константных инициализаторов (`a = 10 + 5` → `a: number = 15`)

**Проход 2 — Использование аннотаций:**
- Переименование: `@Anno(...)` → `@__$$ETS_ANNOTATION$$__Anno(...)`
- Добавление значений по умолчанию в объектные литералы аннотаций
- Удаление SourceRetention аннотаций (только для compile-time)
- Удаление аннотаций с не-классов/не-методов
- Переименование импортов аннотаций

### 5.2 Type Export/Import & Const Enum Transformer (`getTypeExportImportAndConstEnumTransformer`, строка 512)

- Разделение смешанных импортов/экспортов на type-only и value
- Замена const enum обращений на их вычисленные константные значения
- Замена const enum member инициализаторов на вычисленные значения

### 5.3 Kit Import Transformer (`processKit`, строка 1588)

Вызывается на этапе **парсинга**, но по сути является трансформацией AST:
- Чтение JSON-конфигурации kit из SDK (`ets/build-tools/ets-loader/kit_configs/`)
- Замена `import { X } from "@kit.ModuleName"` на реальные пути модулей
- Автоматическое добавление Attribute-импортов для whitelist-компонентов (ArcList, ArcSwiper, MovingPhotoView и др.)
- Поддержка `isLazy` флага для отложенных импортов
- Кэширование JSON-конфигурации в `kitJsonCache`

### Затрагиваемые этапы конвейера:
Transformer напрямую. Применяется между Checker и Emitter (или вместо Emitter внешними инструментами).

---

## 6. Emitter (Генерация кода)

**Файлы:** `src/compiler/emitter.ts`, `src/compiler/transformers/declarations.ts`
**Характер изменений:** Средние — точечные изменения для вывода ETS-синтаксиса

### 6.1 Генерация JavaScript

**StructDeclaration:**
- Маршрутизация: `SyntaxKind.StructDeclaration` → `emitClassOrStructDeclaration` (строка 2048)
- Вывод `struct` вместо `class`: `writeKeyword("struct")` (строка 3872)
- В остальном идентично выводу класса: модификаторы, type parameters, heritage clauses, члены

**AnnotationDeclaration:**
- Маршрутизация: `SyntaxKind.AnnotationDeclaration` → `emitAnnotationDeclaration` (строка 2050)
- Вывод: `@interface Имя { члены }` (строка 3914-3936)

**AnnotationPropertyDeclaration:**
- Вывод: `имя: тип = инициализатор;` (строка 2662-2667)

**EtsComponentExpression:**
- **Критический пропуск** (строка 2350-2355): `findAncestor` проверяет, находится ли узел внутри `EtsComponentExpression`, и **полностью пропускает** его генерацию. Тела компонентов обрабатываются внешним инструментом `ets2bundle`.

**ETS-декораторы на функциях:**
- В ETS-файлах `illegalDecorators` (включая @Builder, @Extend, @Styles) выводятся перед ключевым словом `function` (строка 3737-3740)

### 6.2 Генерация деклараций (`.d.ets`)

**Файл:** `src/compiler/transformers/declarations.ts`

- **@Styles методы** (строка 1052, 1409): Type parameters **подавляются** в декларациях для @Styles-методов
- **StructDeclaration** (строка 1571-1589): сохраняются ETS-декораторы (`ensureEtsDecorators`) и аннотации
- **AnnotationDeclaration** (строка 1591-1607): сохраняются мета-аннотации
- **ClassDeclaration/MethodDeclaration** (строка 1693-1711): сохраняются зарезервированные декораторы и аннотации
- **@Sendable функции** (строка 1378-1415): сохраняют `illegalDecorators`
- **Стриппинг export для ETS** (`stripExportModifiersForEts`, строка 1304): сохранение декораторов при удалении export

### Затрагиваемые этапы конвейера:
Emitter напрямую (финальный этап).

---

## 7. Другие затронутые компоненты

### 7.1 Система типов (`types.ts`)

- **`EtsOptions`** интерфейс (строка 6981-7013): конфигурация компонентов, декораторов, render-методов
- **`EtsFlags`** enum (строка 848-862): 12 флагов контекста парсинга
- **`NodeFlags.EtsContext`** (строка 828): флаг ETS-контекста на узлах AST
- **`NodeFlags.KitImportFlags`** (строка 827): флаг узлов, созданных из kit-импортов
- **`SymbolFlags.Annotation`** (строка 5539): флаг символа аннотации
- **`ObjectFlags.Annotation`** (строка 6035): флаг типа аннотации
- **`ScriptKind.ETS = 8`** (строка 7101): новый вид скрипта
- **`Extension.Ets = ".ets"`**, **`Extension.Dets = ".d.ets"`** (строка 7496-7497)
- **`CheckMode.SkipEtsComponentBody`** (строка 1268): флаг пропуска тела компонента

### 7.2 Разрешение модулей (`moduleNameResolver.ts`)

- Приоритет расширений для ETS-файлов: `.ets` > `.ts` > `.tsx` > `.d.ets` > `.d.ts`
- Для обычных файлов: `.ts` > `.tsx` > `.d.ts` > `.ets` > `.d.ets` (ETS-расширения в конце)
- Поддержка `oh_modules` наравне с `node_modules`

### 7.3 Утилиты (`utilities.ts`, `utilitiesPublic.ts`)

- **`isInETSFile()`** (строка 2991): дубликат `isInEtsFile` из ohApi
- **`supportedTSExtensions`** включает `.ets` и `.d.ets`
- **`isIllegalDecorator`** включает проверки для ArkTS-декораторов и @Sendable

### 7.4 Program (`program.ts`)

- **Linter-режим** (строка 2508): при `strictCheckerOnly` и ETS-файле используется `getLinterTypeChecker()`
- **Загрузка ETS-библиотек** (строка 1579): `options.ets.libs` загружаются в программу
- **Блокировка импорта `.d.ts` → `.ets`** (строка 2655-2688)

### 7.5 ArkTS Linter (`src/linter/`)

Статический анализатор для ArkTS, работающий поверх TypeScript:

- **ArkTSLinter_1_0/** и **ArkTSLinter_1_1/** — версионированные реализации
- **LinterRunner** — основной раннер с поддержкой инкрементального линтинга
- Проверяет строгие правила типизации ArkTS (запрет `any`/`unknown`, запрет структурной типизации и др.)
- Интегрируется с `strictCheckerOnly` в Program для ETS-файлов

### 7.6 Error Codes

- TSC ошибки маппятся в формат OpenHarmony (например, `10505114`)
- UI ошибки: `28000`–`28007`, `28015`
- Linter ошибки: `28016`–`28017`
- Определены через `ErrorInfo` класс и `getErrorCode()` в `ohApi.ts`

---

## 8. Сводная матрица: "Что добавлено на каком этапе"

| Синтаксическая конструкция | Scanner | Parser | Binder | Checker | Transformer | Emitter |
|---------------------------|---------|--------|--------|---------|-------------|---------|
| `struct` keyword | ✓ token (ETS-добавление, с gating) | ✓ AST node | ✓ symbols | ✓ type check | — | ✓ emit |
| `lazy` keyword | ✓ token (ETS-добавление) | — импорт | — | — | — | — |
| `@interface` annotations | — | ✓ AST node | ✓ symbols | ✓ full validation | ✓ rename+defaults | ✓ emit |
| EtsComponentExpression | — | ✓ AST node | — | ✓ type/context check | — | ✓ skip body |
| @Builder decorator | — | ✓ context flags | — | ✓ grammar relax | — | ✓ emit via decorators |
| @Styles decorator | — | ✓ context flags | — | ✓ return type check | — | ✓ emit + strip TPs in decl |
| @Extend decorator | — | ✓ context flags | — | ✓ return type check | — | ✓ emit |
| @State / @Prop / @Link / @Watch / @Consume / @Provide | — | — | — | ✓ init check | — | ✓ emit via EtsOptions |
| @Require | — | — | — | ✓ position check | — | ✓ emit |
| @Sendable | — | — | — | ✓ suppress errors | — | ✓ retain in decl |
| @kit.* imports | — | ✓ parse-time transform | — | — | ✓ AST rewrite | — |
| oh_modules | — | — | — | ✓ module resolution | — | — |
| stateStyles | — | ✓ context flags | — | ✓ property access | — | — |
| Виртуальные конструкторы struct | — | ✓ auto-generate | — | — | — | — |
| Виртуальные идентификаторы Extend/Styles | — | ✓ create virtual | — | — | — | — |

---

## 9. Ключевые выводы

1. **Parser — самый модифицированный этап.** Наибольший объём ETS-кода добавлен именно в парсер: новые AST-узлы, система контекстных флагов, виртуальные конструкторы, kit-трансформация на этапе парсинга.

2. **Checker — самый сложный этап для ETS-валидации.** Содержит всю логику проверки типов для аннотаций, правил размещения компонентов, валидации декораторов и sendable-правил.

3. **Стандартный Transformer не модифицирован.** ETS-трансформации вынесены в `ohApi.ts` и применяются как `customTransformers` внешним инструментом `developtools_ace_ets2bundle`, а не встроены в стандартный конвейер компилятора.

4. **Emitter минимален.** Тела ETS-компонентов пропускаются при стандартной генерации JS — их обработка делегирована внешнему инструменту. Добавлен только вывод `struct` и `@interface` как ключевых слов.

5. **Сканер минимален.** Только условное распознавание `struct` как ключевого слова в зависимости от ETS-контекста.

6. **Binder минимален.** ETS-конструкции повторно используют существующую инфраструктуру связывания (struct как class).

7. **Обратная совместимость — приоритет.** Все ETS-функции активируются только при `ScriptKind.ETS` или `isInEtsFile(node)`. В стандартных `.ts` файлах поведение компилятора полностью сохранено.

8. **Двойной модульный менеджмент.** Код поддерживает сосуществование `node_modules` (npm) и `oh_modules` (ohpm) с разными форматами пакетов (`package.json` vs `oh-package.json5`).

---

## 10. Оценка трудозатрат на портирование (Scanner → Parser → Binder включительно)

### 10.1 Что входит в оценку

Портирование всей ETS/ArkTS-логики до этапа Binder включительно, **включая** структуры данных, которые понадобятся Checker'у в будущем (флаги, свойства символов, интерфейсы), но **без** реализации самого Checker, Transformer и Emitter.

### 10.2 Подсчёт строк кода по компонентам

| Компонент | Файл | Строк ETS-кода | Новых функций | Модификаций существующих |
|-----------|------|----------------|---------------|--------------------------|
| Типы и интерфейсы | `types.ts` | ~185 | — | ~15 (enum, interface, union types) |
| Сканнер | `scanner.ts` | ~15 | 1 (`setEtsContext`) | 2 (`getIdentifierToken`, keyword map) |
| Парсер | `parser.ts` | ~575 | 35 | 36 |
| Биндер | `binder.ts` | ~90 | 1 (`bindAnnotationDeclaration`) | 7 |
| ETS-хелперы (подмножество) | `ohApi.ts` | ~970 | ~45 | — |
| Фабрики узлов | `factory/nodeFactory.ts` | ~100 | 8 | 2 (`createDecorator`/`updateDecorator`) |
| Разрешение модулей | `moduleNameResolver.ts` | ~30 | — | 5 |
| Утилиты | `utilities.ts`, `utilitiesPublic.ts` | ~30 | 2 | 3 |
| Диагностики | `diagnosticMessages.json` | ~25 | — | — |
| Опции компилятора | `commandLineParser.ts` | ~15 | — | 2 |
| **ИТОГО** | | **~2,035** | **92** | **72** |

### 10.3 Детализация по компонентам

#### Типы и интерфейсы (`types.ts`) — ~185 строк

Что добавляется:
- 8 новых `SyntaxKind` значений + обновление union-типов
- `EtsFlags` enum (14 значений)
- `EtsComponentExpression`, `StructDeclaration`, `AnnotationDeclaration`, `AnnotationPropertyDeclaration`, `AnnotationElement` интерфейсы
- `EtsOptions` интерфейс (33 строки конфигурации)
- 20+ новых полей в `CompilerOptions`
- `ScriptKind.ETS = 8`
- `Extension.Ets` / `Extension.Dets`
- `SymbolFlags.Annotation`, `ObjectFlags.Annotation`
- `NodeFlags.EtsContext`, `NodeFlags.KitImportFlags`
- `Decorator.annotationDeclaration?` поле
- `Annotation = Decorator` type alias
- `SourceFile.markedKitImportRange`
- 7 кэш-полей в `Flow` типе для аннотаций
- ~12 сигнатур фабричных методов в `NodeFactory`

#### Сканнер (`scanner.ts`) — ~15 строк

Новый код:
- `setEtsContext(isEtsContext: boolean)` — метод интерфейса + реализация
- `inEtsContext` — внутренняя переменная состояния
- 2 новых ключевых слова в `textToKeywordObj`:
  - `struct` → `StructKeyword` — **с ETS-контекстным gating** (только когда `inEtsContext === true`)
  - `lazy` → `LazyKeyword` — безусловно (для `import lazy X from "..."`)

#### ohApi.ts (подмножество) — ~970 строк из 1915

Нужные функции (сгруппированы по назначению):

| Группа | Функции | Строк | Потребитель |
|--------|---------|-------|-------------|
| Детекция ETS-файла | `isInEtsFile`, `isInEtsFileWithOriginal` | 12 | Parser, Checker |
| ETS-декораторы | `isEtsFunctionDecorators`, `isArkTsDecorator`, все `hasEts*DecoratorNames`, `isTokenInsideBuilder` | 89 | Parser, Checker |
| ohpm/модули | `isOhpm`, `isOHModules`, `getModuleByPMType`, `getPackageJsonByPMType`, `pathContainsOHModules`, `choosePathContainsModules` и др. | 60 | ModuleResolution, Checker |
| Component-хелперы | `getEtsComponentExpressionInner*`, `getRootEtsComponent*`, `isInStateStylesObject` | 54 | Checker (данные) |
| Extend/Styles имена | `getEtsExtendDecoratorComponentNames`, `getEtsStylesDecoratorComponentNames` | 42 | Parser, Checker |
| Sendable/Struct | `isSendableFunctionOrType`, `checkStructPropertyPosition` | 19 | Checker |
| **processKit** + все хелперы | `processKit`, `getSdkPath`, `getKitJsonObject`, `createImportDeclarationForKit`, `markKitImport`, `processKitStatementSuccess`, white-листы и др. | ~440 | **Parser** |
| Прочее | `annotationMagicNamePrefix`, `getMaxFlowDepth`, импорты | ~254 | Все этапы |

**Самая объёмная часть — `processKit` (~440 строк):**
- Чтение JSON-конфигурации kit из SDK
- Трансформация `import { X } from "@kit.ModuleName"` в реальные пути модулей
- 10 white-листов (extend-компоненты, API-модули, kit-модули и др.)
- Генерация виртуальных import-specifier'ов
- Кэширование kit JSON
- Поддержка `isLazy`, `isType` флагов

**Не включаются** (для Transformer/Emitter):
- `getAnnotationTransformer` / `transformAnnotation` (~256 строк)
- `getTypeExportImportAndConstEnumTransformer` (~300 строк)
- `createObfTextSingleLineWriter` (~114 строк)
- `convertTsAstToJsAst` и AST-конвертеры (~339 строк)
- `ErrorInfo` / `getErrorCode` / `getErrorCodeArea` (~82 строки)

#### Парсер (`parser.ts`) — ~575 строк

**35 новых функций:**

| Группа | Функции | Строк |
|--------|---------|-------|
| Система EtsFlags | `setEtsFlag` + 12 `set*Context` | 44 |
| Система EtsFlags | `inEtsFlagsContext` + 14 `in*Context` запросов | 45 |
| `forEachChild` | 3 новые: `forEachChildInAnnotationPropertyDeclaration`, `forEachChildInEtsComponentExpression`, `forEachChildInAnnotationDeclaration` | 17 |
| Struct | `parseStructDeclaration`, `parseStructDeclarationOrExpression`, `parseStructMembers`, `createVirtualHeritageClauses`, `finishVirtualNode` | 90 |
| Struct/Builder | `isTokenInsideStructBuild`, `isTokenInsideStructBuilder`, `isTokenInsideStructPageTransition`, `tryParseConstructorDeclaration`, `parseConstructorName` | 47 |
| Аннотации | `parseAnnotationDeclaration`, `parseAnnotationPropertyDeclaration`, `parseAnnotationElement`, `parseAnnotationMembers`, `isAnnotationMemberStart` | 132 |
| Декораторы | `hasParamAndNoOnceDecorator`, `hasEnvDecorator` | 30 |
| ETS-компоненты | `parseEtsComponentExpression`, `isCurrentTokenAnEtsComponentExpression`, `makeEtsComponentExpression`, `isValidExtendOrStylesContext`, `isValidVirtualTypeArgumentsContext` | 26 |
| Типы | `parseEtsTypeParameters`, `parseEtsTypeArguments`, `parseEtsType`, `parseEtsTypeReferenceWorker` | 41 |
| Прочее | `firstArgumentExpression` getter/setter, `repeatEachRest` getter/setter, `setLanguageVersionByFilePath` | 27 |
| **Итого новых** | | **~499** |

**36 модификаций существующих функций:**

| Функция | Что добавлено | Строк |
|---------|---------------|-------|
| `initializeState` | `ScriptKind.ETS` case, `Extension.Ets`, `scanner.setEtsContext` | 5 |
| `clearState` | сброс ETS-переменных | 6 |
| `parseSourceFileWorker` | вызов `processKit`, `markedKitImportRange` | 8 |
| `doInDecoratorContext` | детекция Extend/Styles декораторов | 10 |
| `parseStatement` | `StructKeyword` → `parseStructDeclaration` | 5 |
| `parseDeclaration` | Extend/Styles/Builder контекст | 22 |
| `parseDeclarationWorker` | `StructKeyword` + `@interface` | 9 |
| `parseFunctionDeclaration` | ETS-контекст, виртуальные типы возврата | 29 |
| `parseMethodDeclaration` | build/builder/styles контекст | 37 |
| `parseClassElement` | `shouldAddReadonly` для struct | 10 |
| `parseModifiers` | автодобавление `readonly` | 22 |
| `parseCallExpressionRest` | виртуальные type arguments для компонентов | 98 |
| `parseAssignmentExpressionOrHigher` | детекция ETS-компонента | 3 |
| `parsePrimaryExpression` | вызов `parseEtsComponentExpression` | 3 |
| `parseNewExpressionOrNewDotTarget` | `setEtsNewExpressionContext` | 2 |
| `parseArgumentExpression` | SyntaxComponent/SyntaxDataSource контекст | 22 |
| `parseArrowFunctionExpressionBody` | UICallback/SyntaxComponent контекст | 11 |
| `tryParseParenthesizedArrowFunctionExpression` | UICallback контекст | 12 |
| `parseLeftHandSideExpressionOrHigher` | виртуальные идентификаторы Extend/Styles/stateStyles | 13 |
| `parseMemberExpressionOrHigher` | DotToken для Extend/Styles/stateStyles | 13 |
| Остальные (~16 функций) | токен-уровневые проверки (`isStartOf*`, `isListElement`, `isListTerminator`, `isReusableClassMember`, etc.) | 22 |
| **Итого модификаций** | | **~362** |

> **Примечание:** сумма новых (499) и модифицированных (362) = 861 строк, но в итоговой оценке указано ~575 из-за пересечений (некоторые модификации учтены внутри новых функций) и того, что часть «модификаций» — это однострочные добавления в switch/case.

#### Биндер (`binder.ts`) — ~90 строк

Новая функция:
- `bindAnnotationDeclaration()` (16 строк) — связывание с `SymbolFlags.Class | SymbolFlags.Annotation`, создание prototype-символа

Модификации:
- `getContainerFlags` — `StructDeclaration`, `AnnotationDeclaration`, `AnnotationPropertyDeclaration`
- `declareSymbolAndAddToSymbolTable` — `StructDeclaration`, `AnnotationDeclaration`
- `bindWorker` — `StructDeclaration` (в strict mode + `bindClassLikeDeclaration`), `AnnotationDeclaration` (→ `bindAnnotationDeclaration`)
- `bindClassLikeDeclaration` — приём `StructDeclaration`
- `bindPropertyWorker` — пропуск `?` optionality для `AnnotationPropertyDeclaration`, default-value optionality

#### Фабрика (`factory/nodeFactory.ts`) — ~100 строк

Новые функции: `createStructDeclaration`, `updateStructDeclaration`, `createAnnotationDeclaration`, `updateAnnotationDeclaration`, `createAnnotationPropertyDeclaration`, `updateAnnotationPropertyDeclaration`, `createEtsComponentExpression`, `updateEtsComponentExpression`

Модификации: `createDecorator`, `updateDecorator` — параметр `annotationDeclaration`

### 10.4 Оценка трудозатрат в человеко-днях

| Компонент | Строк | Сложность | Дни | Обоснование |
|-----------|-------|-----------|-----|-------------|
| **Изучение кодовой базы** | — | Высокая | 5-10 | TS-компилятор сложен; нужно понять архитектуру парсера, биндера, системы типов |
| **types.ts** | ~185 | Средняя | 2-3 | Механическое добавление enum'ов/интерфейсов, но нужно обновить все union-типы |
| **ohApi.ts (подмножество)** | ~970 | Средняя | 8-12 | Много мелких функций; самая сложная часть — `processKit` с white-листами и JSON-конфигурацией |
| **Scanner** | ~15 | Низкая | 0.5-1 | Минимум кода, но критично для обратной совместимости |
| **Parser** | ~575 | **Очень высокая** | 18-28 | 35 новых функций + 36 модификаций. Система EtsFlags пронизывает весь парсер. Виртуальные узлы (конструкторы, идентификаторы, type arguments). Самая рискованная часть. |
| **Binder** | ~90 | Низкая | 2-3 | В основном маршрутизация новых узлов |
| **Factory** | ~100 | Низкая | 1-2 | Шаблонные factory-функции |
| **Module Resolution** | ~30 | Низкая | 1-2 | Изменение приоритета расширений |
| **Utilities** | ~30 | Низкая | 1 | Пара функций + реэкспорты |
| **Диагностики + опции** | ~40 | Низкая | 1-2 | Новые сообщения об ошибках, CLI-опции |
| **Интеграция, тесты, отладка** | — | Высокая | 15-25 | Сборка всей цепочки, написание тестов, отлов регрессий |
| **ИТОГО** | **~2,035** | | **55-89 дней** | |

**Центральная оценка: 70 человеко-дней (14 недель / ~3.5 месяца) для одного разработчика.**

### 10.5 Распределение по рискам

| Уровень риска | Компоненты | Причина |
|---------------|------------|---------|
| 🔴 **Высокий** | Parser, processKit | Глубокая интеграция с существующим кодом; ошибки ломают весь компилятор |
| 🟡 **Средний** | types.ts, ohApi.ts (декораторы/хелперы) | Большой объём кода, но относительно изолированные функции |
| 🟢 **Низкий** | Scanner, Binder, Factory, ModuleResolution | Минимум кода, в основном маршрутизация и шаблоны |

### 10.6 Что конкретно нужно добавить в Parser/Scanner/Binder «про запас» для Checker

Checker'у потребуются от предыдущих этапов:

1. **От Scanner:** `struct` как `SyntaxKind.StructKeyword` (токен); `inEtsContext` флаг
2. **От Parser:**
   - Все новые AST-узлы (`StructDeclaration`, `EtsComponentExpression`, `AnnotationDeclaration`, `AnnotationPropertyDeclaration`)
   - 12 `EtsFlags` на каждом узле (Checker проверяет `inBuildContext()`, `inBuilderContext()` для размещения UI-компонентов)
   - `NodeFlags.EtsContext` на SourceFile (Checker проверяет `isInEtsFile()`)
   - `NodeFlags.KitImportFlags` на kit-импортах
   - `Decorator.annotationDeclaration` ссылка на узел (Checker использует для resolveAnnotation)
   - `SourceFile.markedKitImportRange` (Checker использует для проверок импорта)
   - Виртуальные узлы: конструкторы struct, Extend/Styles идентификаторы, type arguments компонентов
3. **От Binder:**
   - `SymbolFlags.Annotation` на символах аннотаций
   - `SymbolFlags.Class` на символах struct
   - `ObjectFlags.Annotation` на типах аннотаций
   - Prototype-символы для аннотаций (нужны для resolveAnnotation в Checker)
   - `ContainerFlags.IsContainer` для struct и аннотаций
4. **От ohApi.ts:**
   - Все `hasEts*DecoratorNames` / `isEtsFunctionDecorators` / `isArkTsDecorator` (Checker вызывает их)
   - `getEtsComponentExpressionInnerCallExpressionNode` и др. (Checker для property access)
   - `getEtsExtendDecoratorComponentNames` / `getEtsStylesDecoratorComponentNames` (Checker)
   - `isSendableFunctionOrType` (Checker для grammar checks)
   - `checkStructPropertyPosition` (Checker для @Require валидации)
   - `isOhpm`, `isOHModules`, `pathContainsOHModules` и др. (Checker для module resolution)
   - `getMaxFlowDepth` (Checker для flow analysis)

### 10.7 Факторы, влияющие на оценку

**Увеличивают срок:**
- Разработчик не знаком с внутренностями TS-компилятора: +15-20 дней на изучение
- Целевая версия TypeScript сильно отличается от 4.9.5: парсер мог быть реструктурирован
- Отсутствие тестовой инфраструктуры: нужно писать тесты с нуля
- Требуется полная обратная совместимость (`.ts` файлы не должны затрагиваться)

**Уменьшают срок:**
- Разработчик уже работал с TS-компилятором: -10 дней
- Можно копировать код из ohos-typescript (лицензия Apache 2.0): -30% времени
- Некоторые функции можно пока сделать заглушками (раз аннотации не проверяются чекером, `@interface` парсить всё равно нужно, но упрощённо)
- Не нужно портировать Linter (`src/linter/`) — это отдельный компонент

### 10.8 Оценка для команды из 3 разработчиков

#### Граф зависимостей между компонентами

```
types.ts ─────────────┬──────────────────┬─────────────────────┐
(интерфейсы, enum'ы)  │                  │                     │
                      ▼                  ▼                     ▼
                  Scanner           ohApi.ts               Factory
                  (15 строк)     (декораторы,            (100 строк)
                      │           хелперы, ohpm)             │
                      │               │                      │
                      │               ├──────────┐           │
                      │               │          │           │
                      ▼               ▼          ▼           ▼
              ┌──────────────────────────────────────────────────┐
              │                PARSER (575 строк)                │
              │                                                  │
              │  ┌──────────────────┐  ┌───────────────────────┐ │
              │  │ Трек A:          │  │ Трек B:               │ │
              │  │ Декларации       │  │ Выражения + Контекст  │ │
              │  │ • struct         │  │ • EtsFlags система    │ │
              │  │ • @interface     │  │ • EtsComponentExpr    │ │
              │  │ • parseDecl      │  │ • parseCallExprRest   │ │
              │  │ • parseFuncDecl  │  │ • parseArrowFunc      │ │
              │  │ • parseMethodDecl│  │ • parseArgumentExpr   │ │
              │  │ • parseModifiers │  │ • virtual identifiers │ │
              │  └────────┬─────────┘  └──────────┬────────────┘ │
              │           │                       │              │
              └───────────┼───────────────────────┼──────────────┘
                          │                       │
                          └───────────┬───────────┘
                                      │
                          ┌───────────┴───────────┐
                          │                       │
                          ▼                       ▼
                       Binder            ModuleResolution
                      (90 строк)           + Utilities
                          │                 (60 строк)
                          │                       │
                          └──────────┬───── ──────┘
                                     │
                                     ▼
                            Интеграция + Тесты
```

**Критический путь:** types.ts → ohApi.ts (декораторы) → Parser (EtsFlags → оба трека) → Binder → Интеграция

**Независимые компоненты (можно делать параллельно с чем угодно):**
- `processKit` (~440 строк) — полностью автономен, вызывается одной строкой в `parseSourceFileWorker`
- Module Resolution (~30 строк)
- Utilities (~30 строк)
- Диагностики + CLI-опции (~40 строк)
- Factory (~100 строк) — после стабилизации types.ts

#### План работ по фазам

##### Фаза 1: Фундамент (Недели 1-2)

На этом этапе критически важна координация между всеми разработчиками, т.к. принимаются архитектурные решения.

| Разработчик | Задача | Дни | Строк |
|-------------|--------|-----|-------|
| **Dev A** | types.ts — SyntaxKind, NodeFlags, интерфейсы узлов, EtsFlags, EtsOptions, CompilerOptions | 5-6 | ~185 |
| **Dev B** | ohApi.ts (подмножество) — декораторы (`hasEts*DecoratorNames`, `isArkTsDecorator`, `isTokenInsideBuilder`), component-хелперы, ohpm-модули | 5-6 | ~370 |
| **Dev C** | Scanner + Factory + ohApi.ts (processKit — чтение JSON, хелперы трансформации, white-листы) | 6-7 | ~515 |
| **Все вместе** | Синхронизация: утверждение сигнатур, ревью дизайна EtsFlags, согласование контрактов между компонентами | 1-2 | — |

**Результат фазы 1:** types.ts стабилен, все хелперы ohApi.ts написаны, Scanner и Factory готовы.

##### Фаза 2: Парсер — параллельные треки (Недели 3-6)

Парсер разбивается на три трека. **Ключевое условие:** EtsFlags-система (`set*Context` / `in*Context`, ~89 строк) делается **совместно в начале фазы** (2-3 дня), т.к. на неё завязаны оба трека. EtsFlags — это «контракт» между треками A и B.

| Разработчик | Трек | Задача | Дни | Строк |
|-------------|------|--------|-----|-------|
| **Все вместе** | EtsFlags | `setEtsFlag` + 12 `set*Context` + 14 `in*Context` + `firstArgumentExpression`/`repeatEachRest` getter/setter | 2-3 | ~107 |
| **Dev A** | **A: Декларации** | | | |
| | | `parseStructDeclaration` + `parseStructDeclarationOrExpression` + `parseStructMembers` + `createVirtualHeritageClauses` + `finishVirtualNode` | 3-4 | 90 |
| | | `parseAnnotationDeclaration` + `parseAnnotationPropertyDeclaration` + `parseAnnotationElement` + `parseAnnotationMembers` + `isAnnotationMemberStart` | 4-5 | 132 |
| | | Модификации: `parseStatement`, `parseDeclaration`, `parseDeclarationWorker`, `isDeclaration`, `isStartOfStatement` (struct + @interface routing) | 2-3 | 50 |
| | | Модификации: `parseFunctionDeclaration`, `parseMethodDeclaration` (Builder/Styles/Extend контекст) | 3-4 | 66 |
| | | Модификации: `parseClassElement`, `parseModifiers` (readonly-инъекция) + `hasParamAndNoOnceDecorator`, `hasEnvDecorator` | 2-3 | 62 |
| | | Struct/Builder хелперы (`isTokenInsideStructBuild`, etc.) + `parseConstructorName`, `tryParseConstructorDeclaration` | 1-2 | 47 |
| | | `forEachChild`-добавления (3 новые функции + модификации) | 1 | 20 |
| | | `initializeState`, `clearState`, `doInDecoratorContext`, `parseSourceFileWorker` модификации | 2 | 29 |
| | | **Итого Dev A** | **18-24** | **496** |
| **Dev B** | **B: Выражения + Контекст** | | | |
| | | `parseEtsComponentExpression` + `isCurrentTokenAnEtsComponentExpression` + `makeEtsComponentExpression` + `isValidExtendOrStylesContext` + `isValidVirtualTypeArgumentsContext` | 2-3 | 26 |
| | | `parseEtsTypeParameters` + `parseEtsTypeArguments` + `parseEtsType` + `parseEtsTypeReferenceWorker` | 2 | 41 |
| | | Модификации: `parsePrimaryExpression`, `parseAssignmentExpressionOrHigher`, `parseNewExpressionOrNewDotTarget` (вызов component expression) | 1-2 | 8 |
| | | Модификации: `parseCallExpressionRest` (виртуальные type arguments, syntaxComponents — **самый сложный участок**) | 5-7 | 98 |
| | | Модификации: `parseLeftHandSideExpressionOrHigher`, `parseMemberExpressionOrHigher` (виртуальные Extend/Styles идентификаторы) | 2 | 26 |
| | | Модификации: `parseArgumentExpression` (SyntaxComponent/SyntaxDataSource) | 2-3 | 22 |
| | | Модификации: `parseArrowFunctionExpressionBody`, `tryParseParenthesizedArrowFunctionExpression` (UICallback) | 2 | 23 |
| | | Модификации: `parseExpected` + `createIdentifier` (stateStyles bypass) | 0.5-1 | 12 |
| | | Токен-уровневые модификации (`isStartOfLeftHandSideExpression`, `nextTokenCanFollowDefaultKeyword`, `isStartOfExpressionStatement`, etc.) | 1 | 22 |
| | | **Итого Dev B** | **17-22** | **278** |
| **Dev C** | **C: Инфраструктура + Kit** | | | |
| | | `processKit` — завершение: `processKitStatementSuccess`, `createImportDeclarationForKit`, `supplementNamedBindings`, `preProcessSpecifiedImportDeclaration`, `processExtendComponentMap` | 8-10 | 300 |
| | | Интеграция `processKit` в `parseSourceFileWorker` | 0.5 | 8 |
| | | Binder: `bindAnnotationDeclaration` + все модификации | 3-4 | 90 |
| | | Module Resolution: расширения `.ets`/`.d.ets`, `oh_modules` | 1-2 | 30 |
| | | Utilities: `isInETSFile`, `supportedTSExtensions`, `isIllegalDecorator` | 1 | 30 |
| | | Diagnostics: новые сообщения в `diagnosticMessages.json` | 0.5-1 | 25 |
| | | CLI-опции: регистрация в `commandLineParser.ts` | 0.5-1 | 15 |
| | | Помощь Dev A/B по окончании своих задач | 3-5 | — |
| | | **Итого Dev C** | **18-24** | **498** |

**Ключевые точки синхронизации в фазе 2:**
1. Начало фазы: совместная реализация EtsFlags-системы
2. Ежедневно: короткая синхронизация по `initializeState` / `clearState` / `parseSourceFileWorker` (общие функции)
3. Середина фазы: интеграция треков A+B — проверка, что struct-декларации корректно работают внутри EtsComponentExpression
4. Конец фазы: подключение `processKit` (Dev C) к `parseSourceFileWorker` (результат Dev A)

##### Фаза 3: Интеграция и тестирование (Недели 7-9)

| Разработчик | Задача | Дни |
|-------------|--------|-----|
| **Dev A** | Интеграционное тестирование: struct + аннотации + декораторы (полный цикл «парсинг → биндинг» для `.ets` файлов) | 8-12 |
| **Dev B** | Интеграционное тестирование: ETS-компоненты + выражения + контекст (вложенные компоненты, ForEach, stateStyles) | 8-12 |
| **Dev C** | Интеграционное тестирование: `@kit.*` импорты + `oh_modules` + обратная совместимость (`.ts` файлы не затронуты) | 8-12 |
| **Все вместе** | Регрессионное тестирование, баг-фиксы, ревью | 5-8 |

#### Итоговая оценка

| Показатель | 1 разработчик | 3 разработчика |
|------------|---------------|-----------------|
| Календарных недель | **14** (3.5 мес.) | **7-9** (≈2 мес.) |
| Рабочих дней (календарных) | 55-89 | 32-42 |
| Человеко-дней (суммарно) | 70 | **105** (3 × 35) |
| Эффективность распараллеливания | — | ~67% (35д × 3 / 105д) |
| Пиковая загрузка | 100% | 80-100% (неравномерно) |

#### Когда распараллеливание даёт выигрыш, а когда нет

**Выигрыш (≈2.5x ускорение):**
- Фаза 1: types.ts + ohApi + Scanner + Factory — разные файлы, минимум конфликтов
- Фаза 2 (середина): треки A, B, C работают в разных участках парсера (декларации vs выражения vs processKit)
- Фаза 3: тесты пишутся независимо по разным фичам

**Потери от распараллеливания (≈1.3x ускорение):**
- Начало фазы 2: EtsFlags-система делается совместно (не распараллеливается)
- Конец фазы 2: `initializeState`/`clearState`/`parseSourceFileWorker` требуют координации
- Конец фазы 3: баг-фиксы часто последовательны (одна проблема — один фикс)
- Коммуникационные издержки: ежедневные синхронизации, ревью кода

#### Риски при параллельной работе

| Риск | Вероятность | Влияние | Митигация |
|------|------------|---------|-----------|
| Конфликты слияния в общих функциях парсера | Средняя | Задержка 1-2 дня | Чёткое разделение треков (декларации vs выражения); частые коммиты |
| Расхождение в интерпретации EtsFlags | Средняя | Переписывание 2-3 дня | Совместная реализация EtsFlags-системы всеми разработчиками |
| Dev B не может начать до стабилизации EtsFlags | Высокая | Простой 1-3 дня | Dev B начинает с выражения-специфичных новых функций; EtsFlags — приоритет №1 |
| processKit задерживает Dev C дольше ожидаемого | Средняя | Сдвиг интеграции на 2-3 дня | processKit — изолированный модуль; можно интегрировать позже заглушкой |
| Обратная совместимость ломается в процессе | Средняя | Доп. 3-5 дней тестов | Регулярный прогон стандартных TS-тестов (не регрессия) |
