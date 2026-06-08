# Инвентарь изменений ETS/OpenHarmony в форке TypeScript 4.9.5-r4

## 1. Новые AST-узлы (SyntaxKind + интерфейсы)

### Новые значения SyntaxKind (`types.ts`)

| SyntaxKind                      | Строка | Назначение                                                  |
| ------------------------------- | ------ | ----------------------------------------------------------- |
| `StructKeyword`                 | 140    | Ключевое слово `struct` (только в `.ets` файлах)            |
| `LazyKeyword`                   | 213    | Ключевое слово `lazy` для ленивых импортов ArkTS            |
| `AnnotationPropertyDeclaration` | 235    | Свойство внутри `@interface`-блока                          |
| `EtsComponentExpression`        | 286    | Выражение декларативного UI-компонента (`Column() { ... }`) |
| `StructDeclaration`             | 334    | Объявление `struct`                                         |
| `AnnotationDeclaration`         | 335    | Объявление `@interface` (аннотации)                         |

### Новые AST-интерфейсы (`types.ts`)

| Интерфейс | Строка | Описание |
|---|---|---|
| `StructDeclaration` | 3356 | `struct Name { ... }` — облегчённая классоподобная конструкция ETS/ArkTS |
| `AnnotationDeclaration` | 3363 | `@interface Name { properties }` — определение аннотации |
| `AnnotationElement` | 3386 | Базовый интерфейс для элементов аннотации |
| `AnnotationPropertyDeclaration` | 1718 | Свойство внутри аннотации (`name: type = value`) |
| `EtsComponentExpression` | 2563 | Выражение UI-компонента: `ComponentName(args) { body }` |

### Модифицированные интерфейсы

| Интерфейс | Изменение |
|---|---|
| `Decorator` (стр. 1598) | Добавлено поле `annotationDeclaration?: AnnotationDeclaration` |
| `ClassLikeDeclarationBase.kind` (стр. 3342) | Расширен: `StructDeclaration \| AnnotationDeclaration` |

### Новые type-алиасы

| Алиас | Строка | Описание |
|---|---|---|
| `Annotation` | 1606 | `Decorator` с непустым `annotationDeclaration` |

---

## 2. Новые токены лексера (`scanner.ts`)

### Ключевые слова, добавленные форком

| Токен | Строка | Поведение |
|---|---|---|
| `struct` → `StructKeyword` | 176, 1587 | Вне `.ets` файлов парсится как `Identifier` |
| `lazy` → `LazyKeyword` | 185 | Контекстное ключевое слово ArkTS для `import lazy` |

### Состояние сканера

| Элемент | Строка | Описание |
|---|---|---|
| `setEtsContext(isEtsContext)` | 1059-1060 | Включает/выключает ETS-режим сканера |
| `var inEtsContext: boolean` | 996 | Внутренний флаг ETS-контекста |

### Специальная логика

- Строка 1587: если `struct` отсканирован, но `!inEtsContext` → возвращается `Identifier` вместо `StructKeyword`

---

## 3. Изменения в парсере (`parser.ts`)

### 3.1. Система ETS-контекстов

**Состояние (`etsFlags: EtsFlags`, стр. 1574):**

| Флаг | Бит | Назначение |
|---|---|---|
| `StructContext` | 1<<1 | Парсинг внутри struct |
| `EtsExtendComponentsContext` | 1<<2 | Парсинг @Extend компонента |
| `EtsStylesComponentsContext` | 1<<3 | Парсинг @Styles компонента |
| `EtsBuildContext` | 1<<4 | Парсинг внутри build() метода |
| `EtsBuilderContext` | 1<<5 | Парсинг @Builder метода/функции |
| `EtsStateStylesContext` | 1<<6 | Парсинг stateStyles |
| `EtsComponentsContext` | 1<<7 | Парсинг внутри ETS компонента |
| `EtsNewExpressionContext` | 1<<8 | Парсинг внутри new выражения |
| `UICallbackContext` | 1<<9 | Парсинг UI стрелочной функции |
| `SyntaxComponentContext` | 1<<10 | Парсинг ForEach/LazyForEach/Repeat |
| `SyntaxDataSourceContext` | 1<<11 | Парсинг первого аргумента ForEach |
| `NoEtsComponentContext` | 1<<12 | ETS-компоненты запрещены |

**Context-функции (12 set + 12 query):**
- `setStructContext`, `setEtsComponentsContext`, `setEtsNewExpressionContext`, `setEtsExtendComponentsContext`, `setEtsStylesComponentsContext`, `setEtsBuildContext`, `setEtsBuilderContext`, `setEtsStateStylesContext`, `setUICallbackContext`, `setSyntaxComponentContext`, `setSyntaxDataSourceContext`, `setNoEtsComponentContext`
- `inEtsContext`, `inStructContext`, `inEtsComponentsContext`, `inEtsNewExpressionContext`, `inEtsExtendComponentsContext`, `inEtsStylesComponentsContext`, `inBuildContext`, `inBuilderContext`, `inEtsStateStylesContext`, `inUICallbackContext`, `inSyntaxComponentContext`, `inSyntaxDataSourceContext`, `inNoEtsComponentContext`, `inAllowAnnotationContext`

### 3.2. Новые функции парсинга

| Функция | Строка | Назначение |
|---|---|---|
| `parseStructDeclaration` | 8670 | Входная точка парсинга `struct` |
| `parseStructDeclarationOrExpression` | 8701 | Ядро парсинга struct: keyword, имя, type params, heritage, тело |
| `parseStructMembers` | 8806 | Парсинг тела struct + инжекция виртуального конструктора |
| `parseAnnotationDeclaration` | 7706 | Парсинг `@interface Name { ... }` |
| `parseAnnotationElement` | 8621 | Парсинг элемента аннотации |
| `parseAnnotationPropertyDeclaration` | 8188 | Парсинг свойства аннотации |
| `parseAnnotationMembers` | 8802 | Обёртка для списка членов аннотации |
| `parseEtsComponentExpression` | 6873 | Парсинг `ComponentName(args) { body }` |
| `makeEtsComponentExpression` | 5351 | Конвертация `CallExpression` → `EtsComponentExpression` |
| `isCurrentTokenAnEtsComponentExpression` | 6865 | Проверка: является ли текущий токен именем ETS-компонента |
| `parseEtsIdentifier` | 2882 | Создание виртуального идентификатора для Styles компонента |
| `parseEtsTypeParameters` | 4145 | Создание виртуального type parameter |
| `parseEtsTypeArguments` | 4149 | Создание виртуального type argument |
| `parseEtsType` | 5140 | Парсинг ETS type reference |
| `isValidVirtualTypeArgumentsContext` | 6808 | Проверка контекста для виртуальных type arguments |
| `isValidExtendOrStylesContext` | 5237 | Проверка контекста @Extend/@Styles |
| `isTokenInsideStructBuild` | 8095 | Имя метода == `build`? |
| `isTokenInsideStructPageTransition` | 8108 | Имя метода == `pageTransition`? |
| `createVirtualHeritageClauses` | 8732 | Синтетический `extends CustomComponent` |
| `finishVirtualNode` | 8851 | `finishNode` с `virtual: true` |
| `hasParamAndNoOnceDecorator` | 8471 | `@Param` без `@Once` → auto-readonly |
| `hasEnvDecorator` | 8488 | `@Env` / `@CustomEnv` → auto-readonly |

### 3.3. Модифицированные функции парсинга

| Функция | Изменение |
|---|---|
| `initializeState` (1777-1794) | Установка `EtsContext` для `.ets` файлов, вызов `scanner.setEtsContext()` |
| `parseSourceFileWorker` (1835-1848) | Вызов `processKit()` для трансформации kit-импортов |
| `clearState` (1794-1821) | Очистка ETS-состояния |
| `doInDecoratorContext` (2210-2219) | Детект `@Extend`/`@Styles` и установка контекстов |
| `parseDeclaration` (7739-7760) | Обработка `@Extend`/`@Styles`/`@Builder` декораторов |
| `parseDeclarationWorker` (7800-7808) | Роутинг `struct` → `parseStructDeclaration`, `@interface` → `parseAnnotationDeclaration` |
| `parseStatement` (7639-7642) | Обработка `StructKeyword` как начала statement |
| `parseFunctionDeclaration` (8020-8059) | Установка `EtsBuilderContext`, виртуальные type params, виртуальные return types |
| `parseMethodDeclaration` (8129-8185) | Установка `EtsBuildContext`, виртуальные возвращаемые типы для @Styles |
| `parseClassElement` (8557-8560) | Авто-добавление `readonly` для @Param/@Env свойств |
| `parseModifiers` (8517-8529) | Инжекция виртуального `readonly` модификатора |
| `parseArrowFunctionExpressionBody` (5774) | Исключение `struct` из предотвращения ASI |
| `parseCallExpressionRest` (6744-6779) | Обработка `stateStyles`, syntaxComponents, виртуальных type arguments |
| `parseArgumentExpression` (6957-6980) | Отслеживание первого аргумента syntax-компонентов |
| `parseNewExpressionOrNewDotTarget` (7098, 7117) | Сохранение/восстановление `EtsNewExpressionContext` |
| `tryParseDecorator` (8418-8420) | Пропуск `@interface` в контексте аннотаций |

### 3.4. Kit-импорты

Механизм компилятора, преобразующий символические имена `@kit.X` в реальные пути к файлам деклараций SDK на этапе парсинга, до начала проверки типов.

**Синтаксис:**
```typescript
import { Button, Text } from '@kit.ArkUI';
import { image } from '@kit.ImageKit';
```

**Реализация:**
- `KIT_PREFIX = '@kit.'` (ohApi.ts:1235)
- `processKit()` (ohApi.ts:1588) — вызывается из `parseSourceFileWorker` (parser.ts:1837)
- `getKitJsonObject()` (ohApi.ts:1265) — загружает конфигурацию kit'а из `kit_configs/<kitName>.json` в SDK
- `createImportDeclarationForKit()` (ohApi.ts:1320) — создаёт новый import с реальным путём
- Трансформированные узлы помечаются `NodeFlags.KitImportFlags` и `virtual = true`
- Результаты сохраняются в `sourceFile.markedKitImportRange`
- Кэширование: `kitJsonCache` (ohApi.ts:1255), `cleanKitJsonCache()` (ohApi.ts:1304)

**White-list модулей:**
- API-модули для дополнения атрибутов: `@ohos.arkui.*`, `@hms.hds.*` (ohApi.ts:1571)
- Kit-модули: `@kit.ArkUI`, `@kit.MediaLibraryKit`, `@kit.UIDesignKit` (ohApi.ts:1582)

**Пропуск трансформации:** если `noTransformedKitInParser`, отсутствует `sdkPath`, есть ошибки парсинга, или language version callback отклоняет

**Пути SDK:**
- `getSdkPath()` (ohApi.ts:1258) — извлекает путь SDK из `compilerOptions.etsLoaderPath`
- Для mixed compiler SDK (1.2): путь `openharmony/ets/dynamic/build-tools/ets-loader/kit_configs`

### 3.5. Виртуальные узлы struct

- `parseStructMembers` инжектирует виртуальный конструктор с двумя параметрами:
  1. `value?: { prop1?: Type1; prop2?: Type2; ... }` — анонимный объектный тип из свойств
  2. `##storage?: LocalStorage` — специальный storage-параметр

---

## 4. Изменения в чекере (`checker.ts`, ~49683 строк, ~800-1000 строк ETS-изменений)

### 4.1. Система типов — новые флаги и типы

| Добавление | Значение | Назначение |
|---|---|---|
| `SymbolFlags.Annotation` | 1<<28 | Маркировка символов аннотаций |
| `ObjectFlags.Annotation` | 1<<27 | Маркировка object types аннотаций |
| `NodeFlags.EtsContext` | 1<<30 | Маркировка source file в ETS-контексте |
| `NodeFlags.KitImportFlags` | 1<<29 | Маркировка узлов kit-импортов |

**Тип `ESObject`:**
- Специальный тип ArkTS — алиас для `any`, разрешённый в строгом режиме ArkTS (добавлен PR !106, 2023)
- Используется для совместимости с JS API, где тип объекта неизвестен
- Ограничения использования проверяются линтером (`arkts-limited-esobj`)

### 4.2. Проверка новых деклараций

| Функция | Строка | Назначение |
|---|---|---|
| `checkStructDeclaration` | 43211 | Проверка struct (имя, не-зарезервированное, делегирование `checkClassLikeDeclaration`) |
| `checkStructName` | 43222 | Валидация имени struct против `ets.components` |
| `checkAnnotationDeclaration` | 43236 | Проверка `@interface`: top-level, без декораторов, конфликты, дубликаты |
| `checkGrammarAnnotationDeclaration` | 43269 | Принудительное top-level размещение |
| `checkAnnotationPropertyDeclaration` | 38556 | Проверка свойства аннотации (тип + инициализатор, compile-time константы) |

### 4.3. Система аннотаций (~30 функций)

**Разрешение типов аннотаций:**
- `resolveAnnotation` (33649) — разрешение `@Anno`, `@Anno()`, `@Anno({...})` как вызовов
- `annotationHasDefaultValue` (33632) — проверка дефолтных значений
- `setAnnotationDefaultSignature` (33637) — принудительное указание всех полей
- `annotationDefaultSignature` (2055) — синглтон-сигнатура `(undefined, emptyArray, voidType)` для fast path
- Специальная call-сигнатура `(prototype): void` для `SymbolFlags.Annotation` (13131-13144)

**Валидация типов свойств:**
- `isAllowedAnnotationPropertyType` (38341) — только number, boolean, string, const enum, массивы
- `isAllowedAnnotationPropertyEnumType` (38307) — const enum без смешивания number/string

**Compile-time вычисление констант:**
- `evaluateAnnotationPropertyConstantExpression` (38414) — вычисление prefix/binary/literal/identifier/array/conditional
- `annotationEvaluatedValueToExpr` (38358) — конвертация значения → Expression AST
- `addAnnotationPropertyEnumInitalizer` (38583) — синтез инициализатора для const enum

**Retention-аннотации:**
- `isRetentionAnnotationDeclaration` (40237) — проверка `@Retention` из `@arkts.lang.d.ets`
- `isRetentionPolicyEnumDeclaration` (40242) — проверка `RetentionPolicy` enum
- `hasSourceRetentionPolicy` (47122) — разбор `@Retention({policy: RetentionPolicy.SOURCE})`
- `isSourceRetentionAnnotation` (47187) — является ли аннотация source-retention
- `checkSourceRetentionAnnotation` (40343) — валидация размещения source-retention аннотаций

**Импорт/экспорт аннотаций:**
- `isReferredToAnnotation` (47046) — проверка ссылки на аннотацию в import/export
- `isReferredToSourceRetentionAnnotationOrRetentionAnnotation` (47088)
- `isReferredToRetentionPolicy` (47109)

**Связывание аннотаций с узлами:**
- `setAnnotationsOfNode` (40426) — прикрепление `annotationDeclaration` к декораторам
- `getAnnotationDeclaration` (40393) — поиск декларации аннотации по декоратору
- `checkAnnotations` (40223) — старт проверки аннотаций узла
- `checkAnnotation` (40252) — проверка области действия аннотации

### 4.4. Система `apiAvailable` — проверка доступности API по версиям SDK

Механизм, проверяющий что используемые API доступны в целевой версии SDK. API, помеченные аннотацией `@Available(since: "12")`, вызывают предупреждение при использовании на более старых версиях.

- `checkApiAvailableVersion()` (checker.ts:47221) — основная функция проверки при каждом вызове и new-выражении
- `apiAvailableGetTypeOfNode()` (checker.ts:47216) — получение типа узла для проверки (добавлена поддержка `getTypeOfNode` в PR !858, май 2026)
- `isApiAvailableVersionSpecifications` (checker.ts:1451) — host-callback для валидации версий API
- Интегрировано с аннотациями через `@Available(since: "...")` и `@Retention({policy: "source"})`
- Диагностика: `This_API_has_been_Special_Markings_exercise_caution_when_using_this_API` (код 28007/28046)
- Поддержка диапазонов версий: `@Available(since: "12")`, `@Available(since: "12", until: "15")`

### 4.5. `@SuppressWarnings` — подавление предупреждений

Source-retention аннотация, подавляющая определённые предупреждения компилятора на аннотированных объявлениях:

```arkts
@SuppressWarnings("unused")
function testFunc(): void {
  let unusedVar = 42;
}
```

- Относится к категории SourceRetention-аннотаций (checker.ts:47197-47199)
- Валидация содержимого через host callback `isSourceRetentionAnnotationContentValid` (checker.ts:40381-40382)
- Удаляется при emit'е (ohApi.ts:673)
- Добавлен PR !810 (2025)

### 4.6. Sendable — система статической проверки потокобезопасности

Система статической типизации для конкурентного программирования. Гарантирует, что данные, передаваемые между потоками (через TaskPool), безопасны для разделяемого доступа.

**Декоратор `@Sendable`** (ohApi.ts:1211):
- `isSendableFunctionOrType()` — проверка `@Sendable` на функциях и type alias
- Разрешён на `FunctionDeclaration`, `TypeAliasDeclaration`, `ClassDeclaration` (utilities.ts:2721)

**Контроль импорта:**
- `allowImportSendable()` (checker.ts:5138) — `.ts` файлы могут импортировать `.ets` только если включён `tsImportSendableEnable` или файл в SDK
- `disableSendableCheckRules?: string[]` — гранулярное отключение правил
- `tsImportSendableEnable?: boolean` — глобальное разрешение импорта Sendable из TS

**Ключевые концепции:**
- `@Sendable` декоратор: помечает класс, функцию или type alias как безопасный для передачи между потоками
- `ISendable` интерфейс: маркерный интерфейс для sendable-классов
- TaskPool: проверка аргументов — должны быть `@Concurrent`-декорированными

**Ограничения Sendable:**
1. Поля должны иметь явную аннотацию типа
2. Запрещены вычисляемые имена свойств
3. Наследование: только `@Sendable` класс → `ISendable`
4. Объекты `collections.Array` должны быть sendable
5. Импортированные переменные в замыканиях проверяются

### 4.7. EtsComponentExpression — проверка декларативного UI

| Функция | Строка | Назначение |
|---|---|---|
| `traverseEtsComponentStatements` | 37835 | Обход statement'ов в теле ETS-компонента |
| `checkIfEtsComponent` | 37814 | Рекурсивная проверка if/else в теле компонента |
| `checkIfChildComponent` | 37828 | Рекурсия в дочерние компоненты |
| `isSystemEtsComponent` | 34315 | Проверка: является ли идентификатор системным UI-компонентом из SDK |
| `getEtsComponentExpressionPropertyOfType` | 34415 | Разрешение свойств ETS-компонента через @Extend/@Styles |

### 4.8. Разрешение вызовов struct

- `isCalledStructDeclaration` — разрешение вызова struct без `new` (33232-33239)
- Если тип — struct с construct-сигнатурами, разрешается вызов без `new`

### 4.9. Модульное разрешение ETS

| Функция | Строка | Назначение |
|---|---|---|
| `resolveExternalModule` | 4910 | Ветвление для `.so`, `oh_modules`, запрет импорта `.ets` из `.ts` |
| `allowImportSendable` | 5138 | Разрешение импорта Sendable из ETS SDK в TS |

**Обработка `.so` файлов:**
- Опция `tsImportSoCheck` (commandLineParser.ts) — включает проверку типов для `.so` импортов
- При `!tsImportSoCheck`: предупреждение о ненайденном `.so` модуле подавляется, за исключением ETS-файлов с `needDoArkTsLinter`
- Проверка типов `.so` файлов: PR !844 (`feature/enable-so-file-type-check`)
- Игнорирование ошибок для `.so` модулей: PR !69, !78

### 4.10. ETS-декораторы (@Builder, @Styles, @Extend, @AnimatableExtend, @Concurrent)

- `checkGrammarDecorators` (47790-47828) — специальная логика для ETS-декораторов
- Подавление ошибок для `@Builder`, `@Styles`, `@Sendable`

**`@AnimatableExtend`:**
- Декоратор ArkUI для анимируемых расширений компонентов
- Добавлен PR !88 (2023)
- Настраивается через `compilerOptions.ets.extend.decorator`

### 4.11. Проверка инициализации свойств

- `checkPropertyAccessExpressionOrQualifiedName` (31285-31319) — поддержка `ets.propertyDecorators` (например, `@Link`, `@Prop`, `@ObjectLink`, `@Consume`)
- `checkStructPropertyPosition` — проверка позиции `@Require` свойств

### 4.12. Возвращаемые типы

- `checkAllCodePathsInNonVoidFunctionReturnOrThrow` (35615-35706) — проверка возвращаемых типов `@Extend`/`@Styles`

### 4.13. JSDoc @throws (OpenHarmony)

| Функция | Назначение |
|---|---|
| `shouldCheckThrows` (34124) | Определение необходимости проверки @throws |
| `hasThrowsTag` (34143) | Проверка наличия @throws |
| `isThrowsHandled` (34272) | Проверка try/catch/.catch() |
| `parseSinceTag` (34179) | Парсинг `[since]` версий |
| `hasAsyncErrorCallbacks` (34110) | Проверка AsyncCallback/ErrorCallback |
| `checkThrowableFunction` (34090) | Предупреждение о необработанных @throws |

### 4.14. Расширяемая система проверки JSDoc-тегов (@form)

Форк добавляет plugin-подобную инфраструктуру для пользовательской валидации JSDoc-тегов (используется `@form`-фреймворком):

- `JsDocNodeCheckConfig` / `JsDocNodeCheckConfigItem` (types.ts:7545-7560) — конфигурация проверок с callback'ами:
  - `checkValidCallback` — валидация отдельного JSDoc-тега
  - `checkJsDocSuppressorValidCallback` — проверка необходимости валидации
  - `checkConditionValidCallback` — условная валидация
- `FileCheckModuleInfo` (types.ts:7538) — информация о модуле для проверки
- `getJsDocNodeCheckedConfig` / `getJsDocNodeConditionCheckedResult` / `getFileCheckedModuleInfo` — коллбэки на Program/EmitResolver/CustomTransformers
- Интеграция в `checker.ts` (стр. 34475-34489) — вызов коллбэков при обработке JSDoc

### 4.15. Linter-режим и строгие проверки

- `isTypeCheckerForLinter` параметр (1393) — форсирует strict-режимы:
  - `strictNullChecks`, `strictFunctionTypes`, `strictPropertyInitialization`, `noImplicitReturns` (1456-1465)
- `strictCheckerOnly` — строгая проверка типов только для `.ets` файлов, отключает `getJsDocNodeCheckedConfig` (1463)
- `disableStrictCheckPaths?: string[]` — пути, исключённые из строгой проверки (добавлено в !853, апрель 2026)
- `enableStrictCheckOHModule?: boolean` — строгая проверка `oh_modules` (!733, !739)
- `skipOhModulesLint?: boolean` — пропуск линтинга oh_modules
- При `needDoArkTsLinter`: форсируется `skipLibCheck = false` (1452)
- Линтер форсирует strict-режимы даже если они выключены в конфигурации

---

## 5. Изменения в эмиттере (`emitter.ts`, ~6680 строк)

### 5.1. Новые функции эмита

| Функция | Строка | Назначение |
|---|---|---|
| `emitAnnotationPropertyDeclaration` | 2662 | Эмит свойства аннотации (имя, тип, инициализатор) |
| `emitAnnotationDeclaration` | 3914 | Эмит `@interface Name { ... }` |
| `emitClassOrStructDeclaration` | 3863 | Унифицированный эмит class/struct |
| `emitClassOrStructDeclarationOrExpression` | 3867 | Эмит `struct` вместо `class` для StructDeclaration |
| `setAnnotationsOfNode` (walker) | 1006 | Рекурсивная установка аннотаций при отсутствии checker'а |

### 5.2. Модифицированные функции эмита

| Функция | Изменение |
|---|---|
| `emitFunctionDeclarationOrExpression` (3738-3740) | Эмит `illegalDecorators` для функций в ETS-файлах |
| `emitImportClause` (4046-4049) | Эмит ключевого слова `lazy` для ArkTS-импортов |
| `emitTypeParameters` (4950) | Добавлен `StructDeclaration` в union-тип parent |
| `pipelineEmitWithHintWorker` | Добавлены case для `StructDeclaration`, `AnnotationDeclaration`, `AnnotationPropertyDeclaration` |
| `generateMemberNames` (5739) | Добавлен `AnnotationPropertyDeclaration` |

### 5.3. Пропуск эмита ETS-компонентов

- Строка 2350-2355: поиск предка `EtsComponentExpression` → пропуск эмита (компоненты обрабатываются ArkTS runtime)

### 5.4. `illegalDecorators`

Паттерн эмита невалидных декораторов ETS в 10+ местах:
- `emitPropertySignature` (2644), `emitMethodSignature` (2671), `emitVariableStatement` (3456), `emitFunctionDeclarationOrExpression` (3739), `emitInterfaceDeclaration` (3901), `emitTypeAliasDeclaration` (3939), `emitEnumDeclaration` (3953), `emitModuleDeclaration` (3966)

### 5.5. HAR-комментарии

| Механизм | Назначение |
|---|---|
| `writeTsHarComments` (4587-4590) | Добавление `// @keepTs` и `// @ts-nocheck` для HAR-пакетов |
| `reservedComments` / `universalReservedComments` (1678-1681) | Выборочное сохранение комментариев |
| `needToKeepComments` (6061-6077) | Проверка необходимости сохранения комментария по имени узла |

### 5.6. Stub-резолвер

`notImplementedResolver` (1207-1215) — 9 ETS-стабов для работы без checker'а:
- `getAnnotationObjectLiteralEvaluatedProps`, `getAnnotationPropertyEvaluatedInitializer`, `getAnnotationPropertyInferredType`, `isReferredToAnnotation`, `isSourceRetentionAnnotation`, `isSourceRetentionAnnotationDeclaration`, `isReferredToSourceRetentionAnnotationOrRetentionAnnotation`, `isReferredToRetentionPolicy`, `setAnnotationsOfNode`

---

## 6. Новые утилиты / хелперы

### 6.1. `ohApi.ts` — главный ETS-хаб (1915 строк)

**Детекция контекста:**
- `isInEtsFile`, `isInEtsFileWithOriginal` — проверка `.ets` файла
- `isInETSFile` (в utilities.ts) — дублирующая проверка
- `isInBuildOrPageTransitionContext` — проверка build/pageTransition контекста

**Декораторы:**
- `getReservedDecoratorsOfEtsFile`, `getReservedDecoratorsOfStructDeclaration`
- `ensureEtsDecorators`, `concatenateDecoratorsAndModifiers`, `getEffectiveDecorators`
- `isArkTsDecorator`, `isEtsFunctionDecorators`, `isTokenInsideBuilder`
- `hasEtsExtendDecoratorNames`, `hasEtsStylesDecoratorNames`, `hasEtsBuildDecoratorNames`
- `hasEtsBuilderDecoratorNames`, `hasEtsConcurrentDecoratorNames`
- `inEtsStylesContext`

**Компоненты:**
- `getEtsComponentExpressionInnerCallExpressionNode`
- `getRootEtsComponentInnerCallExpressionNode`
- `getEtsComponentExpressionInnerExpressionStatementNode`
- `getEtsExtendDecoratorsComponentNames`, `getEtsStylesDecoratorComponentNames`
- `filterEtsExtendDecoratorComponentNamesByOptions`

**Модульная система (oh_modules):**
- `isOhpm`, `isOHModules`, `isOhpmAndOhModules`
- `getModulePathPartByPMType`, `getModuleByPMType`, `getPackageJsonByPMType`
- `isOHModulesDirectory`, `isTargetModulesDerectory`, `pathContainsOHModules`
- `choosePathContainsModules`, `isOHModulesAtTypesDirectory`, `isOHModulesReference`

**`oh-exports` — контроль экспорта пакетов:**
- Поле в `oh-package.json5`, ограничивающее какие модули пакета можно импортировать извне
- `ResolvedModule.isNotOhExport?: boolean` (types.ts:7439-7440)
- Если модуль разрешён, но не указан в `oh-exports` → `isNotOhExport = true`
- В program.ts (3822): модули с `isNotOhExport` исключаются из программы
- В checker.ts (4944): попытка импорта → ошибка `Cannot_find_module_0_This_module_is_not_exported`
- Добавлен PR !804 (2026-03)

**Трансформеры:**
- `getTypeExportImportAndConstEnumTransformer` — `type` keyword + const enum inline
- `getAnnotationTransformer`, `transformAnnotation` — магический префикс `__$$ETS_ANNOTATION$$__`
- `transformTypeExportImportAndConstEnumInTypeScript`

**Kit-импорты:**
- `processKit` — трансформация `@kit.*` импортов
- `isInMarkedKitImport`, `cleanKitJsonCache`

**Sendable:**
- `isSendableFunctionOrType`

**Структуры:**
- `checkStructPropertyPosition`
- `REQUIRE_DECORATOR`

**Обработка ошибок:**
- `ErrorInfo` класс, `ErrorCodeArea` enum, `getErrorCode`, `getErrorCodeArea`

**Конвертация AST:**
- `convertTsAstToJsAst` + 20+ вспомогательных функций для конвертации TS AST → JS AST

**Прочее:**
- `getSdkPath`, `isMixedCompilerSDKPath`, `getMaxFlowDepth`
- `hasTsNoCheckOrTsIgnoreFlag`, `createObfTextSingleLineWriter`
- `annotationMagicNamePrefix` (`__$$ETS_ANNOTATION$$__`)
- `THROWS_TAG`, `THROWS_CATCH`, `THROWS_ASYNC_CALLBACK`, `THROWS_ERROR_CALLBACK`

### 6.2. `utilities.ts`

| Функция | Строка | Назначение |
|---|---|---|
| `getEtsLibs` | 1012 | Получение ETS-библиотек из `options.ets.libs` |
| `getContainingStruct` | 2415 | Поиск ближайшего `StructDeclaration` предка |
| `getRootEtsComponent` | 4263 | Поиск корневого `EtsComponentExpression` |
| `getRootComponent` | 4274 | То же + учёт syntax-компонентов |
| `getVirtualEtsComponent` | 4293 | Поиск виртуального ETS-компонента |
| `isCalledStructDeclaration` | 9086 | Проверка на StructDeclaration в массиве деклараций |
| `isInBuildOrPageTransitionContext` | 2996 | Проверка контекста build/pageTransition |
| `commonPackageFolders` | 7865 | Включает `"oh_modules"` |
| `supportedTSExtensions` | 8254 | Включает `.ets`, `.d.ets` |
| `extensionIsTS` | 8406 | Возвращает true для `.ets`, `.d.ets` |

### 6.3. `utilitiesPublic.ts`

| Функция | Назначение |
|---|---|
| `getAnnotations(node)` | Получение аннотаций из modifiers |
| `getAnnotationsFromIllegalDecorators(node)` | Получение аннотаций из illegalDecorators |
| `isAnnotationElement(node)` | Проверка `AnnotationPropertyDeclaration` |
| `isStruct(node)` | Проверка `StructDeclaration` |
| `isOnlyAnnotationsAreExportedOrImported(s, resolver)` | Проверка на файл только с аннотациями |
| `canHaveIllegalDecorators` | Расширено: StructDeclaration, AnnotationDeclaration, Constructor |

### 6.4. `factory/nodeTests.ts`

| Функция | Строка | Назначение |
|---|---|---|
| `isDecoratorOrAnnotation` | 408 | Базовый тест для Decorator (общий для декораторов и аннотаций) |
| `isDecorator` | 412 | Декоратор БЕЗ annotationDeclaration |
| `isAnnotation` | 416 | Декоратор С annotationDeclaration |
| `isAnnotationPropertyDeclaration` | 430 | Проверка типа узла |
| `isEtsComponentExpression` | 624 | Проверка типа узла |
| `isStructDeclaration` | 820 | Проверка типа узла |
| `isAnnotationDeclaration` | 824 | Проверка типа узла |

### 6.5. `factory/nodeFactory.ts`

| Функция | Назначение |
|---|---|
| `createEtsComponentExpression` / `updateEtsComponentExpression` | Создание EtsComponentExpression |
| `createStructDeclaration` / `updateStructDeclaration` | Создание StructDeclaration |
| `createAnnotationDeclaration` / `updateAnnotationDeclaration` | Создание AnnotationDeclaration |
| `createAnnotationPropertyDeclaration` / `updateAnnotationPropertyDeclaration` | Создание AnnotationPropertyDeclaration |
| `createDecorator` / `updateDecorator` | Добавлен параметр `annotationDeclaration` |

### 6.6. `visitorPublic.ts`

Добавлены visitor-ы для всех новых типов узлов:
- `SyntaxKind.Decorator` (564), `AnnotationPropertyDeclaration` (588), `EtsComponentExpression` (906), `StructDeclaration` (1207), `AnnotationDeclaration` (1216)

### 6.7. `binder.ts`

| Изменение                            | Строка | Назначение                                              |
| ------------------------------------ | ------ | ------------------------------------------------------- |
| `case StructDeclaration`             | 2141   | Создание контейнера                                     |
| `case AnnotationDeclaration`         | 2142   | Создание контейнера                                     |
| `case AnnotationPropertyDeclaration` | 2189   | Control flow контейнер                                  |
| `case AnnotationPropertyDeclaration` | 2923   | Биндинг через `bindPropertyWorker`                      |
| `case StructDeclaration`             | 2991   | Биндинг struct                                          |
| `case AnnotationDeclaration`         | 2996   | Биндинг аннотации                                       |
| `bindAnnotationDeclaration`          | 3600   | Биндинг с `SymbolFlags.Class \| SymbolFlags.Annotation` |

### 6.8. `moduleNameResolver.ts`

- Расширение списка расширений: `.ets`, `.d.ets`
- Поддержка `oh_modules` директорий
- Разрешение `oh-package.json5` вместо `package.json` с использованием библиотеки `json5` для парсинга JSON5-формата
- Приоритет `.d.ets` над `.d.ts` при наличии `compilerOptions.ets`

### 6.9. `commandLineParser.ts` — ETS-опции компилятора

| Опция | Тип | Назначение |
|---|---|---|
| `ets` | `EtsOptions` | Главная ETS-конфигурация (компоненты, декораторы, render, etc.) |
| `packageManagerType` | `string` | Тип пакетного менеджера (`"ohpm"` для OpenHarmony) |
| `emitNodeModulesFiles` | `boolean` | Эмит node_modules/oh_modules файлов |
| `etsAnnotationsEnable` | `boolean` | Включение аннотаций |
| `etsLoaderPath` | `string` | Путь к ETS-загрузчику (SDK) |
| `tsImportSendableEnable` | `boolean` | Разрешение импорта Sendable из TS в ETS |
| `tsImportSoCheck` | `boolean` | Проверка типов для `.so` импортов |
| `maxFlowDepth` | `number` | Максимальная глубина flow-анализа (2000–65535) |
| `skipOhModulesLint` | `boolean` | Пропуск линтинга oh_modules |
| `enableStrictCheckOHModule` | `boolean` | Строгая проверка oh_modules |
| `disableStrictCheckPaths` | `string[]` | Пути, исключённые из строгой проверки |
| `disableSendableCheckRules` | `string[]` | Sendable-правила для отключения |
| `strictCheckerOnly` | `boolean` | Строгая проверка только для `.ets` файлов |
| `mixCompile` | `boolean` | Режим смешанной компиляции |
| `isCompileJsHar` | `boolean` | Вывод HAR как JS файла |
| `moduleRootPath` | `string` | Корневой путь модуля |
| `compatibleSdkVersion` | `number` | Совместимая версия SDK |
| `compileSdkVersion` | `number` | Целевая версия SDK |
| `noTransformedKitInParser` | `boolean` | Пропуск kit-трансформации в парсере |
| `skipPathsInKeyForCompilationSettings` | `boolean` | Пропуск путей в ключе компиляции |
| `skipBaseUrlInKeyForCompilationSettings` | `boolean` | Пропуск baseUrl в ключе компиляции |
| `compatibleSdkVersionStage` | `string` | Стадия совместимой версии SDK |

### 6.10. `program.ts`

| Изменение | Назначение |
|---|---|
| `linterTypeChecker` | Отдельный TypeChecker для ArkTS-линтера |
| `getEtsLibSFromProgram` | Получение ETS-библиотек |
| `getLinterTypeChecker` | Ленивое создание линтер-чекера |
| `filterDiagnostics` | Фильтрация диагностик для `oh_modules` и kit-деклараций |
| `isNotOhExport` | Проверка экспорта из oh_modules |
| `needDoArkTsLinter` | Гард для ArkTS-линтера |

### 6.11. `builder.ts`

| Изменение | Назначение |
|---|---|
| `arktsLinterDiagnosticsPerFile` | Кэш диагностик линтера |
| `arkTSVersion` | Кэш версии ArkTS |
| `isForLinter` | Флаг режима линтера |
| `createBuilderProgram` с `isForLinter` | Создание builder-программы в режиме линтера |
| Build-info сериализация | Сохранение/восстановление диагностик линтера |

### 6.12. `diagnosticMessages.json`

**42 новых диагностических сообщения (коды 28000-28046):**
- 28000-28007, 28015: UI-компоненты и struct
- 28008-28014: oh_modules разрешение
- 28016-28017: ArkTS импорт
- 28018-28045: Аннотации
- 28046: Специальные отметки API

### 6.13. Инфраструктура профилирования

**`performanceDotting.ts`** (новый файл, ~200 строк) — инфраструктура замера производительности компилятора:
- `PerformanceDotting.start`/`stop` — иерархический замер времени операций (start/stop с поддержкой вложенности до 5 уровней)
- Режимы: `DEFAULT`, `VERBOSE`, `TRACE` (через `AnalyzeMode` enum)
- `getEventData()` — агрегация собранных метрик
- Используется в `checker.ts`, `binder.ts`, `parser.ts`, `emitter.ts`, `program.ts`

**`memorydotting/memoryDotting.ts`** (новый файл) — инфраструктура профилирования потребления памяти:
- Добавлен коммитом `d0e4d9fa6c Add memoryDotting`
- Используется для мониторинга аллокаций памяти в режиме сборки

### 6.14. Реорганизация `_namespaces/`

Коммит `183292ce68 Convert codebase from namespace into module` разделил монолитный namespace TypeScript на модульные файлы:
- `_namespaces/ts.ts` — основные типы и утилиты
- `_namespaces/ts.moduleSpecifiers.ts` — спецификаторы модулей
- `_namespaces/ts.performance.ts` — performance API

Структурное изменение, облегчающее поддержку и навигацию по коду.

### 6.15. `parser.ts` — arkguard-режим

- `createSourceFile` принимает опциональный параметр `isArkguardInput?: boolean` (стр. 1378)
- `isArkguardInputSourceFile` — глобальный флаг парсера (стр. 1377)
- Расширяет `inAllowAnnotationContext()`: аннотации разрешены не только в `.ets` файлах, но и при входе из arkguard (стр. 2324)

### 6.16. `@Concurrent` декоратор

Декоратор ArkTS для пометки функций, выполняемых в параллельном режиме:
- `hasEtsConcurrentDecoratorNames` в `ohApi.ts` (стр. 368) — проверка наличия `@Concurrent`
- Поддержка в `EtsOptions.concurrent.decorator`
- Используется TaskPool: аргументы функций, передаваемых в `taskpool`, должны быть `@Concurrent`-декорированными
- Добавлен коммитом `6987d740bd Support concurrent decorator`

### 6.17. ArkGuard и обфускация

- **ArkGuard** — инструмент обфускации кода ArkTS
- `--enable-arkguard` — флаг для включения arkguard-режима (PR !362)
- `createObfTextSingleLineWriter()` (ohApi.ts:1095) — однострочный writer для обфускации (убирает пробелы/переносы строк)
- **useTsHar**: флаги для обфускации в контексте HAR-пакетов (!657)
- **Obfuscation whitelist**: поддержка wildcard'ов в белом списке обфускации (!359)
- **Compact sourcemap**: исправление sourcemap при обфускации (!490)

### 6.18. Изолированные декларации (isolate declarations)

- Поддержка изолированных деклараций для ускорения компиляции
- Добавлено PR !709 (2025-12)
- Позволяет компилятору обрабатывать декларации независимо друг от друга

### 6.19. Инкрементальная компиляция и кэширование

- **TSC incremental**: переиспользование `.tsbuildinfo` для инкрементальной компиляции
- **Linter incremental**: линтер переиспользует механизм инкрементальной компиляции TSC (!653, !656)
- **Кэширование линтера**: `arktsLinterDiagnosticsPerFile` в builder state
- **Linter cache**: PR !800 "addCacheForLinter"
- **Parallel linter**: параллельное выполнение линтера (!211)
- **Language service cache**: использование кэша языкового сервиса для ускорения (!258)

### 6.20. Dynamic import

- Исправления динамических импортов (!598)
- Dynamic import json5 (`4c5ab8bdc4 dynamic import json5`)
- Поддержка `import()` выражений в контексте ETS

### 6.21. Оптимизации производительности и памяти

- **Performance dotting** (§6.13): инструментирование замеров времени в TSC (!572) и линтере (!599)
- **Memory dotting** (§6.13): отслеживание пикового использования памяти (!514)
- **Освобождение TypeChecker'ов**: оптимизация потребления памяти (!405)
- **Управление GC**: принудительный сбор мусора в режиме сборки «только в памяти» (`buildEndOnlyMemoryModeGC`, !840)
- **Оптимизация сигнатур и функций** (!575, !576)
- **Оптимизация эмита** (!826)

---

## Сводная статистика

| Категория | Количество |
|---|---|
| Новые SyntaxKind | 6 |
| Новые AST-интерфейсы | 5 |
| Модифицированные AST-интерфейсы | 1 (Decorator) |
| Новые токены лексера | 2 (struct, lazy) |
| Новые функции парсера | 19+ |
| Модифицированные функции парсера | 16+ |
| EtsFlags (контексты парсера) | 13 |
| Новые функции чекера | 50+ |
| Новые функции эмиттера | 5+ |
| Новые диагностики | 42 |
| Функции в ohApi.ts | 60+ |
| ETS-утилиты в utilities.ts | 15+ |
| ETS-утилиты в utilitiesPublic.ts | 7+ |
| Фабричные функции | 10 |
| Node-test функции | 7 |
| Visitor-функции | 8 |
| CompilerOptions | 22 |
| Модульная система (oh_modules + oh-exports) | Полная поддержка ohpm |
| Инфраструктура профилирования | 2 новых файла (performanceDotting.ts, memorydotting/) |
| Новые файлы компилятора | 4 (ohApi.ts, performanceDotting.ts, memorydotting/, _namespaces/ts*) |
| Системы компилятора | apiAvailable, Sendable, @SuppressWarnings, аннотации, Kit-импорты |
| Оптимизации | Инкрементальная компиляция, кэширование линтера, GC-оптимизация |
| Обфускация | ArkGuard, useTsHar, obfuscation whitelist |
