# Инвентарь правил ArkTS-линтера

Версии: **1.0** (базовая) и **1.1** (расширенная — Sendable, Shared Module, TS Interop)

Кодовая база: `src/linter/ArkTSLinter_1_0/` и `src/linter/ArkTSLinter_1_1/`, общие утилиты: `src/linter/Common/`

---

## Архитектура

```
LinterRunner.runArkTSLinter(builderProgram)
  └─ TSCCompiledProgram — запуск TSC-диагностик в strict и non-strict режимах,
  │   разница (strict-only) → правило arkts-strict-typing
  └─ TypeScriptLinter.lint() — обход AST .ets файла
       ├─ incrementOnlyTokens: Map<SyntaxKind, FaultID> — мгновенная регистрация
       └─ handlersMap: Map<SyntaxKind, handlerFunction> — контекстные проверки
            └─ incrementCounters(node, faultId) → ProblemInfo[]
                 └─ translateDiag() → Diagnostic (ошибка компилятора)
```

- **v1.0:** код диагностики = `-1` для всех правил, severity через `faultAttrs.warning: boolean`
- **v1.1:** код = номер cookBook-правила (1–184), severity через `ProblemSeverity` enum (ERROR=2, WARNING=1)

---

## Категории правил

### 1. Система типов (запрет неподдерживаемых типов TS)

| Правило | Описание | Версии |
|---|---|---|
| `arkts-no-any-unknown` | Запрет `any` и `unknown` — используйте явные типы | Обе |
| `arkts-no-symbol` | Запрет `Symbol()` API | Обе |
| `arkts-no-typing-with-this` | Запрет `this` в типовых позициях | Обе |
| `arkts-no-type-query` | Запрет `typeof` в типовых позициях (разрешён только в выражениях) | Обе |
| `arkts-no-is` | Запрет оператора-предиката `is` | Обе |
| `arkts-no-conditional-types` | Запрет conditional types (`T extends U ? X : Y`) | Обе |
| `arkts-no-mapped-types` | Запрет mapped types (`{ [K in T]: ... }`) | Обе |
| `arkts-no-intersection-types` | Запрет intersection types (`A & B`) — используйте наследование | Обе |
| `arkts-no-aliases-by-index` | Запрет indexed access types (`T[K]`) | Обе |
| `arkts-no-indexed-signatures` | Запрет index signatures (`[key: string]: T`) | Обе |
| `arkts-no-obj-literals-as-types` | Запрет объектных литералов как деклараций типов | Обе |
| `arkts-no-ctor-signatures-type` | Запрет конструктор-сигнатур в типах (используйте класс) | Обе |
| `arkts-no-ctor-signatures-iface` | Запрет конструктор-сигнатур в интерфейсах | Обе |
| `arkts-no-ctor-signatures-funcs` | Запрет конструктор-функциональных типов | Обе |
| `arkts-no-call-signatures` | Запрет call-сигнатур в типах (используйте класс) | Обе |
| `arkts-no-utility-types` | Ограничение стандартных utility-типов (`Partial`, `Pick`, `Omit`, etc.) | Обе |
| `arkts-no-structural-typing` | Запрет структурной типизации — типы обязаны иметь номинальную идентичность | Обе |
| `arkts-no-inferred-generic-params` | Generic-вызовы обязаны иметь явные type-аргументы когда компилятор вывел бы `unknown` | Обе |
| `arkts-limited-esobj` | Ограничение использования типа `ESObject` | Обе |

### 2. Sendable (только v1.1)

| Правило | Описание |
|---|---|
| `arkts-sendable-class-inheritance` | `@Sendable` класс может наследовать/реализовывать только sendable-типы; не-sendable класс не может наследовать sendable |
| `arkts-sendable-prop-types` | Свойства `@Sendable` классов/интерфейсов должны иметь sendable-типы данных |
| `arkts-sendable-definite-assignment` | Оператор `!` (definite assignment) запрещён в `@Sendable` классах |
| `arkts-sendable-generic-types` | Типовые аргументы generic `@Sendable` типов сами должны быть sendable |
| `arkts-sendable-imported-variables` | Только импортированные/верхнеуровневые переменные могут захватываться методами `@Sendable` класса |
| `arkts-sendable-class-decorator` | Только `@Sendable` декоратор разрешён на `@Sendable` классах (без ArkUI-декораторов) |
| `arkts-sendable-obj-init` | Объекты `@Sendable` типа нельзя инициализировать объектными/массивными литералами |
| `arkts-sendable-computed-prop-name` | Вычисляемые имена свойств запрещены в `@Sendable` классах/интерфейсах |
| `arkts-sendable-as-expr` | Запрет каста не-sendable данных к sendable-типу через `as` |
| `arkts-sendable-explicit-field-type` | Поля `@Sendable` класса должны иметь явные аннотации типов |
| `arkts-sendable-function-imported-variables` | `@Sendable` функции могут захватывать только импортированные/верхнеуровневые переменные |
| `arkts-sendable-function-decorator` | Только `@Sendable` декоратор на `@Sendable` функциях |
| `arkts-sendable-function-overload-decorator` | Все перегрузки `@Sendable` функции должны иметь `@Sendable` декоратор |
| `arkts-sendable-function-property` | Доступ к свойствам `@Sendable` функциональных типов ограничен |
| `arkts-sendable-function-as-expr` | Запрет каста не-sendable функции к sendable type alias |
| `arkts-sendable-typealias-decorator` | Только `@Sendable` декоратор на sendable type alias |
| `arkts-sendable-typeAlias-declaration` | Только функциональные типы могут объявлять `@Sendable` type alias |
| `arkts-sendable-function-assignment` | Только sendable функции/type-alias объекты могут присваиваться sendable type alias |
| `arkts-sendable-decorator-limited` | `@Sendable` декоратор только на `class`, `function`, `typeAlias` |
| `arkts-sendable-beta-compatible` | Sendable функции/typealias недоступны до API12 beta3 |
| `arkts-no-ts-sendable-type-inheritance` | В `.ts` файлах sendable-типы не могут использоваться в `extends`/`implements` |
| `arkts-no-dts-sendable-type-export` | В SDK `.d.ts` файлах sendable классы/интерфейсы нельзя экспортировать |

### 3. Shared Module (только v1.1, `use shared`)

| Правило | Описание |
|---|---|
| `arkts-no-side-effects-imports` | Side-effect импорты запрещены в shared-модулях |
| `arkts-shared-module-exports` | Только sendable-сущности можно экспортировать из shared-модулей |
| `arkts-shared-module-no-wildcard-export` | `export * from` запрещён в shared-модулях |

### 4. TS Interop (только v1.1)

| Правило | Описание |
|---|---|
| `arkts-no-ts-import-ets` | Только sendable классы/интерфейсы разрешены при импорте `.ets` в `.ts` |
| `arkts-no-ts-re-export-ets` | Реэкспорт из `.ets` в `.ts` не поддерживается |
| `arkts-no-namespace-import-in-ts-import-ets` | Namespace-импорт из `.ets` в `.ts` запрещён |
| `artkts-no-side-effect-import-in-ts-import-ets` | Side-effect импорт из `.ets` в `.ts` запрещён |

### 5. Декораторы

| Правило | Описание | Версии |
|---|---|---|
| `arkts-no-decorators-except-arkui` | Разрешены только ArkUI-декораторы (whitelist в `Common/ArkUIDecoratorBlackList.ts`) | Только 1.0 |

Белый список ArkUI-декораторов (`ARKUI_DECORATOR_LIST`): `Entry`, `Component`, `Reusable`, `CustomDialog`, `Consume`, `Link`, `LocalStorageLink`, `LocalStorageProp`, `ObjectLink`, `Prop`, `Provide`, `State`, `StorageLink`, `StorageProp`, `Builder`, `LocalBuilder`, `BuilderParam`, `Observed`, `Require`, `Sendable`, `Track`, `ComponentV2`, `ObservedV2`, `Trace`, `Local`, `Param`, `Once`, `Event`, `Monitor`, `Provider`, `Consumer`, `Computed`, `Type`, `Env`.

> В v1.1 `UnsupportedDecorators` удалён — проверка декораторов перешла в sendable-правила.

### 6. Функции и управление потоком

| Правило | Описание | Версии |
|---|---|---|
| `arkts-no-func-expressions` | Используйте стрелочные функции вместо function-выражений | Обе |
| `arkts-no-generic-lambdas` | Используйте generic-функции вместо generic стрелочных | Обе |
| `arkts-no-generators` | Generator-функции (`function*`) и `yield` не поддерживаются | Обе |
| `arkts-no-nested-funcs` | Вложенные/локальные function-декларации не поддерживаются | Обе |
| `arkts-no-standalone-this` | `this` внутри standalone-функций не поддерживается | Обе |
| `arkts-no-implicit-return-types` | Вывод возвращаемого типа ограничен — часто требуются явные типы | Обе |
| `arkts-no-func-props` | Объявление свойств на функциях не поддерживается | Обе |
| `arkts-no-func-apply-bind-call` | `Function.apply`/`bind`/`call` не поддерживаются | Только 1.0 |
| `arkts-no-func-apply-call` | `Function.apply`/`call` не поддерживаются | Только 1.1 |
| `arkts-no-func-bind` | `Function.bind` не поддерживается | Только 1.1 |

### 7. Объекты и массивы

| Правило | Описание | Версии |
|---|---|---|
| `arkts-no-untyped-obj-literals` | Объектные литералы должны иметь контекстный класс/интерфейс | Обе |
| `arkts-no-noninferrable-arr-literals` | Элементы массивных литералов должны быть выводимых типов | Обе |
| `arkts-no-ambiguity-obj-literal` | Тип объектного литерала неоднозначен между несколькими возможными типами | Только 1.1 |
| `arkts-identifiers-as-prop-names` | Строковые/числовые литералы не могут быть именами свойств (используйте идентификаторы) | Обе |
| `arkts-computed-prop-names` | Вычисляемые имена свойств не поддерживаются (1.1: разрешён `[Symbol.iterator]` в коллекциях) | Обе |
| `arkts-no-props-by-index` | Индексный доступ (`obj["prop"]`) запрещён для полей классов/интерфейсов | Обе |
| `arkts-no-spread` | Spread только для массивов и array-производных типов | Обе |
| `arkts-no-undefined-prop-access` | Доступ к неопределённым полям | Только 1.0 |
| `arkts-no-method-reassignment` | Переназначение методов объекта не поддерживается (замена `no-undefined-prop-access` в 1.1) | Только 1.1 |

### 8. Переменные и декларации

| Правило | Описание | Версии |
|---|---|---|
| `arkts-no-var` | Используйте `let`/`const` вместо `var` | Обе |
| `arkts-no-destruct-decls` | Деструктурирующие декларации не поддерживаются | Обе |
| `arkts-no-destruct-assignment` | Деструктурирующее присваивание не поддерживается | Обе |
| `arkts-no-destruct-params` | Деструктурирующие параметры не поддерживаются | Обе |
| `arkts-no-ctor-prop-decls` | Параметр-свойства в конструкторе не поддерживаются | Обе |
| `arkts-no-definite-assignment` | Оператор `!` (definite assignment) не поддерживается (WARNING) | Обе |
| `arkts-no-private-identifiers` | Приватные `#` идентификаторы не поддерживаются | Обе |
| `arkts-no-as-const` | `as const` утверждения не поддерживаются | Обе |
| `arkts-unique-names` | Уникальные имена обязательны для типов и пространств имён | Обе |
| `arkts-no-class-literals` | Class-выражения (анонимные классы) не поддерживаются | Обе |

### 9. Операторы и выражения

| Правило | Описание | Версии |
|---|---|---|
| `arkts-no-with` | `with` не поддерживается | Обе |
| `arkts-limited-throw` | `throw` принимает только Error-производные типы | Обе |
| `arkts-no-for-in` | `for..in` не поддерживается | Обе |
| `arkts-no-in` | Оператор `in` не поддерживается | Обе |
| `arkts-no-delete` | Оператор `delete` не поддерживается | Обе |
| `arkts-no-comma-outside-loops` | Оператор «запятая» разрешён только в `for`-циклах | Обе |
| `arkts-no-polymorphic-unops` | Унарные `+`, `-`, `~` работают только с числами | Обе |
| `arkts-instanceof-ref-types` | `instanceof` только для ссылочных типов, не примитивов | Обе |
| `arkts-no-regexp-literals` | RegExp-литералы не поддерживаются | Обе |
| `arkts-no-jsx` | JSX не поддерживается | Обе |

### 10. Возможности TS, отсутствующие в ArkTS

| Правило | Описание | Версии |
|---|---|---|
| `arkts-no-decl-merging` | Declaration merging интерфейсов не поддерживается | Обе |
| `arkts-no-enum-merging` | Слияние enum не поддерживается | Обе |
| `arkts-extends-only-class` | Интерфейсы не могут расширять классы | Обе |
| `arkts-no-extend-same-prop` | Интерфейс не может расширять интерфейсы с одинаковым свойством разных типов | Обе |
| `arkts-implements-only-iface` | Только интерфейсы в `implements`, не классы | Обе |
| `arkts-no-multiple-static-blocks` | Только один статический блок на класс | Обе |
| `arkts-no-enum-mixed-types` | Члены enum должны иметь compile-time инициализаторы одного типа | Обе |
| `arkts-as-casts` | Только синтаксис `as T` для кастов; также покрывает `number as Number` | Обе |
| `arkts-no-prototype-assignment` | Присваивание prototype не поддерживается | Обе |
| `arkts-no-new-target` | `new.target` не поддерживается | Обе |
| `arkts-no-globalthis` | `globalThis` не поддерживается | Обе |

### 11. Модули и импорты

| Правило | Описание | Версии |
|---|---|---|
| `arkts-no-ns-as-obj` | Пространства имён нельзя использовать как объекты | Обе |
| `arkts-no-classes-as-obj` | Классы нельзя использовать как объекты (1.1: WARNING) | Обе |
| `arkts-no-ns-statements` | Не-декларационные statement'ы в namespace не поддерживаются | Обе |
| `arkts-no-import-default-as` | Default-импорты не поддерживаются | Только 1.0 |
| `arkts-no-export-assignment` | `export =` не поддерживается | Обе |
| `arkts-no-require` | `require` и `import =` не поддерживаются | Обе |
| `arkts-no-misplaced-imports` | Импорты должны быть в начале файла | Обе |
| `arkts-no-ambient-decls` | Ambient-декларации модулей не поддерживаются | Обе |
| `arkts-no-module-wildcards` | Wildcard в именах модулей не поддерживаются | Обе |
| `arkts-no-umd` | UMD-определения модулей не поддерживаются | Обе |
| `arkts-no-import-assertions` | Import-утверждения не поддерживаются | Обе |
| `arkts-limited-stdlib` | Использование API стандартной библиотеки ограничено | Обе |

### 12. Строгая типизация и подавление ошибок

| Правило | Описание | Версии |
|---|---|---|
| `arkts-strict-typing` | Принудительная строгая проверка типов (TSC strict-mode диагностики) | Обе |
| `arkts-strict-typing-required` | `@ts-ignore`, `@ts-nocheck`, `@ts-expect-error` запрещены | Обе |

### 13. TaskPool (только v1.1)

| Правило | Описание |
|---|---|
| `arkts-taskpool-concurrent-function-args` | Функции, передаваемые в `taskpool`, должны быть `@Concurrent`-декорированными обычными функциями |

---

## Различия между версиями

### Добавлено в v1.1

| Группа | Количество | Правила |
|---|---|---|
| Sendable | 23 | `arkts-sendable-*` |
| Shared Module | 4 | `arkts-shared-*`, `arkts-no-side-effects-imports` |
| TS Interop | 4 | `arkts-no-ts-*`, `arkts-no-namespace-*`, `artkts-no-side-effect-*` |
| TaskPool | 1 | `arkts-taskpool-concurrent-function-args` |
| Объекты | 1 | `arkts-no-ambiguity-obj-literal` |
| Функции | 2 | `arkts-no-func-apply-call`, `arkts-no-func-bind` (разделение `arkts-no-func-apply-bind-call`) |
| Заменено | 1 | `arkts-no-method-reassignment` вместо `arkts-no-undefined-prop-access` |

### Удалено в v1.1

- `RegexLiteral` (`arkts-no-regexp-literals` — проверяется иначе)
- `ImportFromPath` / `DefaultImport` (покрываются новыми shared/ts-interop правилами)
- `LambdaWithTypeParameters` (cookBook-тег 49 пуст)
- `UnsupportedDecorators` (заменён sendable-правилами декораторов)
- `FunctionApplyBindCall` (разделён на `FunctionApplyCall` + `FunctionBind`)
- `NoUndefinedPropAccess` (заменён на `MethodReassignment`)

### Поведенческие изменения

- **Коды ошибок:** v1.0 использует `-1` для всех правил; v1.1 — уникальный номер `cookBookRef` (1–184)
- **Severity:** v1.0 — булевский `warning`; v1.1 — `ProblemSeverity` enum (ERROR=2, WARNING=1)
- **LibraryTypeCallDiagnosticChecker:** v1.0 — instance-based; v1.1 — singleton с новыми типами ошибок (`POSSIBLY_UNDEFINED`, downgrade для OHModules)
- **InteropTypescriptLinter** (v1.1): новый линтер для `.ts` файлов с sendable import/export проверками
- **handleComputedPropertyName** (v1.1): разрешает `[Symbol.iterator]` в ArkTS-коллекциях
- **handleObjectLiteralExpression** (v1.1): добавляет проверки `SendableObjectInitialization` и `ObjectLiteralAmbiguity`

---

## Файловая структура

| Файл | Назначение |
|---|---|
| `Problems.ts` | Enum `FaultID`, `FaultAttributes`, маппинг на cookBook |
| `TypeScriptLinter.ts` | Основной класс линтера с handler-методами |
| `InteropTypescriptLinter.ts` (1.1) | Линтер `.ts` файлов (TS→ETS interop) |
| `TypeScriptLinterConfig.ts` | `nodeDesc[]`, `terminalTokens`, `incrementOnlyTokens` |
| `LinterRunner.ts` | Входная точка `runArkTSLinter()` |
| `CookBookMsg.ts` | Человекочитаемые названия правил (`cookBookTag[]`) |
| `Utils.ts` | Утилиты, хелперы типов, API allowlist |
| `Common.ts` | Интерфейсы `AutofixInfo`, `LintOptions`, `ProblemSeverity` |
| `TSDiagnostics.ts` | `TSCCompiledProgram` — strict/non-strict diff диагностики |
| `DiagnosticChecker.ts` | Интерфейс `DiagnosticChecker` |
| `LibraryTypeCallDiagnosticChecker.ts` | Фильтрация ошибок null/undefined/unknown-типов |
| `Autofixer.ts` | Автофиксы (часть отключена как «unsafe») |
| `Common/ArkUIDecoratorBlackList.ts` | Белый список ArkUI-декораторов |
| `Common/ArkTSTimePrinter.ts` | Утилита замера времени |
| `linter.ts` | Модульная точка входа |

---

## Сводная статистика

| Категория | 1.0 | 1.1 (новые) | Всего в 1.1 |
|---|---|---|---|
| Система типов | 19 | — | 19 |
| Sendable | — | 23 | 23 |
| Shared Module | — | 4 | 4 |
| TS Interop | — | 4 | 4 |
| Декораторы | 1 | — | 0 (поглощено sendable) |
| Функции | 9 | 2 (разделение) | 10 |
| Объекты/массивы | 8 | 2 | 9 |
| Переменные | 10 | — | 10 |
| Операторы | 10 | — | 10 |
| Возможности TS | 11 | — | 11 |
| Модули/импорты | 13 | — | 11 (2 удалено) |
| Строгая типизация | 2 | — | 2 |
| TaskPool | — | 1 | 1 |
| **Всего** | **83** | **36** | **114** |
