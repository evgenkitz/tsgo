# Спецификация ArkTS

> Основано на анализе истории коммитов репозитория `third_party_typescript` (форк TypeScript для OpenHarmony)
> Период разработки: февраль 2022 — июнь 2026
> Версия компилятора: 4.9.5-r4 (изначально 4.2.3, обновлён до 4.9.5 в ноябре 2023)

---

## 1. Введение

### 1.1. Что такое ArkTS

ArkTS — это статически типизированное подмножество TypeScript, разработанное для экосистемы OpenHarmony. Компилятор ArkTS (`ohos-typescript`) является форком TypeScript 4.9.5 от Microsoft и расширяет его следующими возможностями:

- **Новые синтаксические конструкции** для декларативного UI (ArkUI): `struct`, `@Builder`, `@Styles`, `@Extend`, `EtsComponentExpression`
- **Новые расширения файлов**: `.ets` (ArkTS source) и `.d.ets` (ArkTS declarations)
- **ArkTS Linter** — система статического анализа, ограничивающая использование динамических возможностей TypeScript/JavaScript
- **Модульная система OpenHarmony**: `oh_modules`, `oh-package.json5`
- **Система Kit-импортов**: преобразование `@kit.ArkUI` в реальные пути SDK
- **Проверка доступности API**: механизм `apiAvailable` для контроля версий API
- **Система Sendable**: статическая проверка безопасности передачи данных между потоками
- **Система аннотаций**: декларативные аннотации (`@Retention`, `@SuppressWarnings`, `@throws`)
- **Оптимизации производительности**: инкрементальная компиляция, кэширование, строгие режимы

### 1.2. История версий

| Версия | Дата | Ключевые изменения |
|--------|------|-------------------|
| **4.2.3-r1** | 2022-02 | Форк TypeScript 4.2.3. Добавление `.ets`, `struct`, `@Builder`, `@Styles`, `@Extend`, `EtsComponentExpression` |
| **4.2.3-r2** | 2022-06 | Поддержка `.d.ets`, `@Concurrent`, `oh_modules`, `oh-package.json5` |
| **4.2.3-r3–r7** | 2022–2023 | ArkTS Linter v1.0, Kit-импорты (`@kit.*`), проверка `apiAvailable` |
| **4.2.3-r8** | 2023-10 | Последний стабильный релиз на базе TS 4.2.3 |
| **4.9.5-r1** | 2023-11 | Апмерж до TypeScript 4.9.5 |
| **4.9.5-r2** | 2023-11 | `@Require`, `@Sendable`, Sendable-правила линтера, `@SuppressWarnings` |
| **4.9.5-r3** | 2024–2025 | ArkTS Linter v1.1, аннотации, `@throws`, `@Retention`, shared modules, import-lazy, taskpool |
| **4.9.5-r4** | 2025–2026 | `CustomEnv`, `WithEnv`, `apiAvailable.getTypeOfNode`, `disableStrictCheckPaths`, изолированные декларации |

### 1.3. Статистика разработки

- **Оригинальных коммитов ArkTS**: ~390 (авторы с доменами `@huawei.com`, `@huawei-partners.com`, `@h-partners.com`)
- **Pull Request'ов**: 858 (формат `!N`, платформа Gitee)
- **Правил линтера**: 110+ тестовых директорий `arkts-*`
- **Ключевых разработчиков**: xucheng46 (47), liyancheng2 (32), lizhonghan1 (27), caiyu30 (27), zhangchen168 (21)

---

## 2. Синтаксические расширения ArkTS

### 2.1. Ключевое слово `struct`

**Назначение:** `struct` — основной строительный блок декларативного UI в ArkTS. Аналог `class`, но с особой семантикой для построения компонентов интерфейса.

**Синтаксис:**
```arkts
@component
struct MyComponent {
  @State count: number = 0;

  build() {
    Column() {
      Text(`Count: ${this.count}`)
      Button('Increment')
        .onClick(() => { this.count++; })
    }
  }
}
```

**Реализация:**
- `SyntaxKind.StructDeclaration` (types.ts:334) — AST-узел, наследующий `ClassLikeDeclarationBase`
- `SyntaxKind.StructKeyword` — лексема `struct` (scanner.ts:176), распознаётся только в контексте `.ets` файлов (scanner.ts:1587)
- Парсер: `parseStructDeclaration()` (parser.ts:8670) — создаёт виртуальный конструктор с параметрами из объявлений свойств, плюс `##storage?: LocalStorage`
- Чекер: `checkStructDeclaration()` (checker.ts:43211) — проверяет имя, резервированные имена компонентов

**Файлы:** `src/compiler/types.ts`, `src/compiler/scanner.ts`, `src/compiler/parser.ts`, `src/compiler/checker.ts`

**Первый PR:** !2 (2022-02-28) — "change ts sourcecode for ets rules"

---

### 2.2. Декоратор `@Builder` / `@BuilderParam`

**Назначение:** `@Builder` помечает метод или функцию как «builder» — она может содержать выражения UI-компонентов. `@BuilderParam` помечает свойство как параметр-builder, принимающий функцию-builder извне.

**Синтаксис:**
```arkts
@Component
struct MyComponent {
  @BuilderParam noParam: () => void;

  @Builder CompC(value: string) {
    Text(value)
  }

  build() {
    Column() {
      this.noParam()
      this.CompC('Hello')
    }
  }
}
```

**Реализация:**
- `EtsFlags.EtsBuilderContext` (types.ts) — флаг контекста
- `hasEtsBuilderDecoratorNames()` (ohApi.ts:349) — проверка наличия декоратора
- Парсер: при обнаружении `@Builder` устанавливает `setEtsBuilderContext(true)` (parser.ts:8024, 8133)
- Чекер: в Builder-контексте декораторы считаются валидными (checker.ts:47811)
- Имена декораторов конфигурируются через `compilerOptions.ets.render.decorator` (по умолчанию `["Builder", "LocalBuilder"]`)

**Первый PR:** !20 (2022-04-22) — "support export @Builder & Styles in export Struct"

---

### 2.3. Декоратор `@Styles`

**Назначение:** Определяет стилевые функции, которые можно применять к компонентам. Может применяться на уровне компонента (метод) или глобально (функция).

**Синтаксис:**
```arkts
@Styles
function globalFancy() {
  .width(100).height(100)
}

@Component
struct MyComponent {
  @Styles componentFancy() {
    .width(100).height(100)
  }

  build() {
    Column() {
      Text('Hello').globalFancy()
      Text('World').componentFancy()
    }
  }
}
```

**Реализация:**
- `EtsFlags.EtsStylesComponentsContext` (types.ts:852)
- `hasEtsStylesDecoratorNames()` (ohApi.ts:319) — проверка декоратора
- Парсер: функция/метод с `@Styles` получает особый контекст (parser.ts:7751, 8135), тип возврата из `compilerOptions.ets.styles.components`
- Чекер: доступ к свойствам `EtsComponentExpression` ищет styles/extend-символы (checker.ts:31094)

**Первый PR:** !20 (2022-04-22)

---

### 2.4. Декоратор `@Extend`

**Назначение:** Расширяет конкретный тип компонента новыми методами. `@Extend(ComponentName) function funcName(...) { ... }`.

**Синтаксис:**
```arkts
@Extend(Text)
function fancy(fontSize: number) {
  .fontSize(fontSize)
  .fontColor(Color.Red)
}

// Использование:
Text('Hello').fancy(20)
```

**Реализация:**
- `EtsFlags.EtsExtendComponentsContext` (types.ts:851)
- `getEtsExtendDecoratorsComponentNames()` (ohApi.ts:465) — извлекает имя компонента из `@Extend(ComponentName)`
- Парсер: ищет компонент в `compilerOptions.ets.extend.components` (parser.ts:7740)

**Первый PR:** !24 (2022-06-21) — "support export of @Styles & @Extend"

---

### 2.5. Декоратор `@Require`

**Назначение:** Помечает свойство struct как требующее инициализации при создании экземпляра (не нуждается в значении по умолчанию).

**Синтаксис:**
```arkts
@Component
struct Test {
  @Require name: number;  // Не требует инициализации здесь; должно быть передано при создании
  myName: number = this.name;
  build() {}
}
```

**Реализация:**
- `REQUIRE_DECORATOR = 'Require'` (ohApi.ts:1233)
- `checkStructPropertyPosition()` (ohApi.ts:1227) — валидация позиции свойства внутри struct
- Чекер: свойства с `@Require` освобождаются от проверки "used before initialization" (checker.ts:31306-31318)

**Первый PR:** !233 (2023-12) — "support Require"

---

### 2.6. Декоратор `@Sendable`

**Назначение:** Помечает класс, функцию или type alias как "sendable" — безопасный для передачи между потоками/акторами.

**Синтаксис:**
```arkts
@Sendable
class SendableClass implements ISendable {
  public prop: string = '';
  constructor() {}
  myMethod1() {}
}

@Sendable
type SendableType = { data: string };

@Sendable
function sendableFunc(): void {}
```

**Реализация:**
- `isSendableFunctionOrType()` (ohApi.ts:1211) — проверка для функций и type alias
- Чекер: `@Sendable` разрешён на `FunctionDeclaration` и `TypeAliasDeclaration` (utilities.ts:2721)
- `allowImportSendable()` (checker.ts:5138) — контроль импорта Sendable из `.ts` в `.ets`
- `disableSendableCheckRules?: string[]` — опция компилятора для отключения отдельных правил

**Первый PR:** !311 (2024) — "[ArkTS Linter] Initial implementation of Sendable rules"

---

### 2.7. Декоратор `@Retention`

**Назначение:** Управляет политикой сохранения аннотаций. `@Retention({policy: "source"})` означает, что аннотация используется только на этапе компиляции и не попадает в вывод (emit).

**Синтаксис:**
```arkts
@Retention({policy: RetentionPolicy.Source})
annotation Available {
  since: string;
}
```

**Реализация:**
- `isRetentionAnnotationDeclaration()` (checker.ts:40237) — идентификация `@Retention`
- `isSourceRetentionAnnotation()` (checker.ts:47185) — проверка политики source-retention
- `hasSourceRetentionPolicy()` (checker.ts:47122) — проверка наличия `@Retention({policy: "source"})`
- Аннотации с source-retention policy удаляются при emit'е (ohApi.ts:633, 673)

**Первый PR:** !353 (2024) — "Add annotations support, stage 1"

---

### 2.8. Аннотация `@throws`

**Назначение:** Документирует, что функция может выбрасывать исключения. При вызове такой функции компилятор проверяет, что вызывающий код обрабатывает исключения (через try-catch, `.catch()`, или сам аннотирован `@throws`).

**Синтаксис:**
```arkts
/**
 * @throws { BusinessError } 401 - Parameter error
 */
function mayThrow(): void {
  // ...
}

// Вызывающий код должен обработать исключение:
try {
  mayThrow();
} catch (e) {
  // обработка
}
```

**Реализация:**
- `THROWS_TAG = 'throws'` (ohApi.ts:1239)
- `checkThrowableFunction()` (checker.ts:34090) — проверка, что вызов `@throws`-функции находится внутри try-catch, `.catch()`, или другой `@throws`-функции
- `isThrowsHandled()` (checker.ts:34272) — обход AST вверх для поиска try-catch/.catch()
- `shouldSkipTag()` (checker.ts:34253) — фильтрация по версиям (`[since xx]`, `[since xx - yy]`)
- Поддержка `AsyncCallback<T>` и `ErrorCallback<E>` в параметрах функции

**Первый PR:** !663 (2025) — "Add check for throws"

---

### 2.9. Декоратор `@SuppressWarnings`

**Назначение:** Source-retention аннотация, подавляющая определённые предупреждения компилятора на аннотированных объявлениях.

**Синтаксис:**
```arkts
@SuppressWarnings("unused")
function testFunc(): void {
  let unusedVar = 42;
}
```

**Реализация:**
- Относится к категории SourceRetention аннотаций (checker.ts:47197-47199)
- Валидация содержимого через host callback `isSourceRetentionAnnotationContentValid` (checker.ts:40381-40382)
- Удаляется при emit'е (ohApi.ts:673)

**Первый PR:** !810 (2025) — "merge supresswarnings into master"

---

### 2.10. Выражение `EtsComponentExpression`

**Назначение:** AST-узел, представляющий вызов UI-компонента в декларативном синтаксисе ArkUI. Например, `Column() { ... }` или `Text("hello")`.

**Реализация:**
- `SyntaxKind.EtsComponentExpression` (types.ts:286)
- Интерфейс включает: `expression`, `typeArguments`, `arguments`, `body` (types.ts:2563-2568)
- Парсер: `isCurrentTokenAnEtsComponentExpression()` (parser.ts:6865) — проверяет, является ли текущий токен именем компонента из `compilerOptions.ets.components`
- `makeEtsComponentExpression()` (parser.ts:5351) — создаётся из CallExpression с последующей `{` в Builder-контексте
- Вспомогательные функции: `getEtsComponentExpressionInnerCallExpressionNode()`, `getRootEtsComponentInnerCallExpressionNode()` (ohApi.ts:403-456)

---

### 2.11. Расширения файлов `.ets` и `.d.ets`

**Назначение:** `.ets` — исходные файлы ArkTS, `.d.ets` — файлы деклараций ArkTS.

**Реализация:**
- `Extension.Ets = ".ets"`, `Extension.Dets = ".d.ets"` (types.ts:7496-7497)
- `ScriptKind.ETS = 8` (types.ts:7101)
- Включены в `supportedTSExtensions` (utilities.ts:8254): `.ets` и `.d.ets` наравне с `.ts`/`.tsx`/`.d.ts`
- Сканер: `inEtsContext` устанавливается при парсинге `.ets` файлов (parser.ts:1777-1778), включает распознавание `struct` как ключевого слова (scanner.ts:1587)
- Множество проверок по всему компилятору зависят от `ScriptKind.ETS` / `isInEtsFile()`

**Первый PR:** !2 (2022-02-28)

---

### 2.12. Импорт `lazy`

**Назначение:** Ленивый импорт модулей — модуль загружается только при первом обращении к его экспортам. Ключевое слово `lazy` распознаётся сканером как `SyntaxKind.LazyKeyword` (scanner.ts:185) **безусловно** (в отличие от `struct`, которое контекстно-зависимо).

**Синтаксис:**
```arkts
import lazy { Component } from '@kit.ArkUI';
import lazy defaultExport from './module';
```

**Реализация:**
- `SyntaxKind.LazyKeyword` (types.ts:213) — новое ключевое слово
- Сканер: `lazy` → `LazyKeyword` (scanner.ts:185), распознаётся всегда, без ETS-контекстного gating
- Эмиттер: `emitImportClause` (emitter.ts:4046) выводит ключевое слово `lazy` перед импортом

**Первый PR:** !421 (2024-10) — "enable import lazy with lazy keyword"

---

### 2.13. Декоратор `@AnimatableExtend`

**Назначение:** Расширение `@Extend` для анимируемых свойств компонентов. `@AnimatableExtend(ComponentName)` аналогичен `@Extend`, но применяется к анимируемым стилям.

**Синтаксис:**
```arkts
@AnimatableExtend(Text)
function animatableFancy(fontSize: number) {
  .fontSize(fontSize)
  .opacity(0.5)
}
```

**Первый PR:** !88 (2023-05) — "Support tsc compiling for ets @AnimatableExtend syntax"

---

### 2.14. Декоратор `@LocalBuilder`

**Назначение:** Вариант `@Builder`, который создаёт локальный builder, не экспортируемый наружу. Распознаётся наравне с `@Builder` через `compilerOptions.ets.render.decorator` (по умолчанию `["Builder", "LocalBuilder"]`).

**Синтаксис:**
```arkts
@Component
struct MyComponent {
  @LocalBuilder
  myLocalBuilder() {
    Text('Local builder content')
  }
}
```

**Реализация:**
- `hasEtsBuilderDecoratorNames()` (ohApi.ts:349) — проверяет и `Builder`, и `LocalBuilder`
- Функционально идентичен `@Builder`, но ограничен областью видимости компонента

---

### 2.15. Декораторы `@Param` и `@Once`

**Назначение:**
- **`@Param`** — помечает свойство struct как параметр, передаваемый извне. Свойства с `@Param` без `@Once` автоматически получают `readonly`.
- **`@Once`** — разрешает свойству с `@Param` быть мутабельным (без `readonly`). Используется только вместе с `@Param`.

**Синтаксис:**
```arkts
@Component
struct MyComponent {
  @Param title: string;           // auto-readonly — не может быть изменено
  @Param @Once mutableCounter: number = 0;  // мутабельное — может изменяться
}
```

**Реализация:**
- `hasParamAndNoOnceDecorator()` (parser.ts:8471) — проверка `@Param` без `@Once`
- `parseClassElement` (parser.ts:8557) — авто-добавление `readonly` модификатора
- `parseModifiers` (parser.ts:8517) — инжекция виртуального `readonly`

---

### 2.16. Декораторы `@Env` и `@CustomEnv`

**Назначение:** Внедрение значений из окружения (environment) в свойства компонента.
- **`@Env`** — стандартные переменные окружения ArkUI
- **`@CustomEnv`** — пользовательские переменные окружения (добавлен в PR !855, апрель 2026)

Свойства с `@Env`/`@CustomEnv` автоматически получают `readonly`.

**Синтаксис:**
```arkts
@Component
struct MyComponent {
  @Env language: string;           // auto-readonly из системного окружения
  @CustomEnv mySetting: number;    // auto-readonly из пользовательского окружения
}
```

**Реализация:**
- `hasEnvDecorator()` (parser.ts:8488) — проверка `@Env`/`@CustomEnv`
- `parseClassElement` / `parseModifiers` — авто-добавление `readonly`

**Первый PR:** !855 (2026-04) — "merge customEnv into master"

---

### 2.17. Синтаксис stateStyles

**Назначение:** stateStyles — специальный синтаксис внутри UI-компонентов для определения стилей, зависящих от состояния компонента (pressed, focused, disabled и т.д.).

**Синтаксис:**
```arkts
Button('Submit')
  .stateStyles({
    pressed: () => {
      .backgroundColor(Color.Grey)
    },
    focused: () => {
      .borderWidth(2)
    }
  })
```

**Реализация:**
- `EtsFlags.EtsStateStylesContext` (types.ts:854) — флаг контекста
- `stateStylesRootNode` (parser.ts:1622) — имя корневого узла в stateStyles-контексте
- При входе в stateStyles-контекст: `.prop` транслируется в виртуальный доступ к `rootNodeInstance.prop` (parser.ts:6237-6248)
- `parseExpected` обход (parser.ts:2520): в stateStyles-контексте отсутствующие токены молча принимаются
- `parseIdentifier` (parser.ts:2850): создание виртуального идентификатора `${rootNode}Instance`

---

### 2.18. Изолированные декларации (`isolateDeclarations`)

**Назначение:** Режим компиляции, при котором декларации типов генерируются изолированно от исходного кода. Поддерживает пошаговую миграцию на строгую типизацию ArkTS.

**Реализация:**
- `Support isolateDeclarations step1` (PR !709, 2025-12)

**Первый PR:** !709 (2025-12)

---

## 3. ArkTS Linter

### 3.1. Архитектура

ArkTS Linter — система статического анализа, проверяющая соответствие кода ArkTS подмножеству TypeScript. Запрещает использование динамических возможностей JavaScript/TypeScript, несовместимых с моделью выполнения ArkTS.

#### Две версии линтера

| | ArkTS Linter 1.0 | ArkTS Linter 1.1 |
|---|---|---|
| **Расположение** | `src/linter/ArkTSLinter_1_0/` | `src/linter/ArkTSLinter_1_1/` |
| **Файлы** | `.ets` только | `.ets` + `.ts` (через `InteropTypescriptLinter`) |
| **FaultID** | 37 категорий ошибок | 80+ категорий |
| **Строгость** | ERROR для всех нарушений | ERROR + WARNING (через `ProblemSeverity`) |
| **Sendable** | Нет | 20+ Sendable-правил |
| **Межмодульные проверки** | Нет | Shared modules, TS↔ETS interop |

#### Интеграция с компилятором

Линтер тесно интегрирован с TypeScript компилятором:

1. **Опция компилятора** `needDoArkTsLinter` включает линтер
2. **Собственный TypeChecker** (`getLinterTypeChecker()`, program.ts:2421) — отдельный экземпляр для линтера
3. **Кэширование**: `arktsLinterDiagnosticsPerFile` (builder.ts) — Map файл→диагностики для инкрементальной компиляции
4. **Входная точка**: `runArkTSLinter()` (LinterRunner.ts) — итерирует исходные файлы, вызывает `TypeScriptLinter.lint()` для `.ets` и `InteropTypescriptLinter.lint()` для `.ts`
5. **Диагностики**: Коды ошибок 28016-28017 относятся к области `ErrorCodeArea.LINTER` (ohApi.ts)

#### Ключевые файлы

| Файл | Назначение |
|------|-----------|
| `src/linter/ArkTSLinter_1_1/TypeScriptLinter.ts` | Основной класс линтера (~114KB) |
| `src/linter/ArkTSLinter_1_1/LinterRunner.ts` | Точка входа, оркестрация |
| `src/linter/ArkTSLinter_1_1/Problems.ts` | FaultID enum (80+ категорий) |
| `src/linter/ArkTSLinter_1_1/InteropTypescriptLinter.ts` | Линтер для `.ts` в смешанных проектах |
| `src/linter/ArkTSLinter_1_1/Utils.ts` | Утилиты (~87KB) |
| `src/linter/Common/ArkUIDecoratorBlackList.ts` | Разрешённые декораторы ArkUI (34 шт.) |

---

### 3.2. Группы правил линтера

#### 3.2.1. Ограничения типов

Запрещают небезопасные типы TypeScript, которые полагаются на динамическую природу JavaScript:

| Правило | Описание |
|---------|----------|
| `arkts-no-any-unknown` | Запрет `any` и `unknown` |
| `arkts-no-conditional-types` | Запрет условных типов |
| `arkts-no-intersection-types` | Запрет пересечений типов (intersection) |
| `arkts-no-mapped-types` | Запрет mapped-типов |
| `arkts-no-indexed-signatures` | Запрет индексированных сигнатур |
| `arkts-no-type-query` | Запрет `typeof` |
| `arkts-no-utility-types` | Ограниченный набор utility-типов |
| `arkts-no-symbol` | Запрет `symbol` |
| `arkts-no-typing-with-this` | Запрет `this` в аннотациях типов |
| `arkts-no-obj-literals-as-types` | Запрет объектных литералов как типов |
| `arkts-limited-esobj` | Ограничения `ESObject` |
| `arkts-instanceof-ref-types` | `instanceof` только для ссылочных типов |
| `arkts-structural-typing` | Контроль структурной типизации |
| `arkts-strict-typing` | Строгая типизация |
| `arkts-strict-typing-required` | Обязательная строгая типизация |

**Пример: `arkts-no-any-unknown`**

Запрещено:
```typescript
let age: unknown = 30;
let aabc: any = 'xiaoming';
```

Разрешено:
```typescript
let age: number = 30;
let aabc: string = 'xiaoming';
```

#### 3.2.2. Ограничения синтаксиса

Запрещают динамические/небезопасные конструкции JavaScript:

| Правило | Описание |
|---------|----------|
| `arkts-no-func-expressions` | Запрет функциональных выражений |
| `arkts-no-generators` | Запрет генераторов |
| `arkts-no-destruct-assignment` | Запрет деструктурирующего присваивания |
| `arkts-no-destruct-decls` | Запрет деструктурирующих объявлений |
| `arkts-no-destruct-params` | Запрет деструктурирующих параметров |
| `arkts-no-var` | Запрет `var` |
| `arkts-no-comma-outside-loops` | Запятая только в циклах |
| `arkts-no-spread` | Запрет spread-оператора |
| `arkts-no-for-in` | Запрет `for..in` |
| `arkts-no-delete` | Запрет `delete` (использовать `= null`) |
| `arkts-no-in` | Запрет оператора `in` |
| `arkts-no-is` | Запрет оператора `is` |
| `arkts-no-standalone-this` | `this` только в методах экземпляра |
| `arkts-no-jsx` | Запрет JSX |
| `arkts-no-regexp-literals` | Запрет литералов RegExp |
| `arkts-no-new-target` | Запрет `new.target` |
| `arkts-no-private-identifiers` | Запрет `#` приватных идентификаторов |
| `arkts-as-casts` | Правила приведения типов |
| `arkts-no-func-apply-bind-call` | Запрет `apply`/`bind`/`call` |
| `arkts-no-noninferrable-arr-literals` | Массивы с невыводимым типом |
| `arkts-no-untyped-obj-literals` | Объектные литералы без контекстного типа |
| `arkts-identifiers-as-prop-names` | Имена свойств — идентификаторы |
| `arkts-no-polymorphic-unops` | Ограничения унарных операторов |
| `arkts-no-as-const` | Запрет `as const` |
| `arkts-no-generic-lambdas` | Запрет generic-лямбд |
| `arkts-no-with` | Запрет оператора `with` |

**Пример: `arkts-no-delete`**

Запрещено:
```typescript
delete obj.a;
```

Разрешено:
```typescript
obj.a = null;
```

#### 3.2.3. Ограничения наследования

| Правило | Описание |
|---------|----------|
| `arkts-extends-only-class` | `extends` — только классы (не интерфейсы) |
| `arkts-implements-only-iface` | `implements` — только интерфейсы |
| `arkts-no-decl-merging` | Запрет слияния объявлений |
| `arkts-no-enum-merging` | Запрет слияния enum'ов |
| `arkts-no-extend-same-prop` | Запрет переопределения свойства |
| `arkts-no-method-reassignment` | Запрет переназначения методов |

#### 3.2.4. Ограничения классов и объектов

| Правило | Описание |
|---------|----------|
| `arkts-no-classes-as-obj` | Класс не может использоваться как значение |
| `arkts-no-class-literals` | Запрет class-выражений |
| `arkts-no-ctor-prop-decls` | Запрет параметров-свойств конструктора |
| `arkts-no-call-signatures` | Запрет call-сигнатур |
| `arkts-no-ctor-signatures-funcs` | Запрет construct-сигнатур в функциях |
| `arkts-no-ctor-signatures-iface` | Запрет construct-сигнатур в интерфейсах |
| `arkts-no-ctor-signatures-type` | Запрет construct-сигнатур в type alias |
| `arkts-no-multiple-static-blocks` | Только один статический блок на класс |
| `arkts-no-prototype-assignment` | Запрет присваивания prototype |

#### 3.2.5. Ограничения модулей и импортов

| Правило | Описание |
|---------|----------|
| `arkts-no-ts-deps` | Ограничения зависимостей TS→TS |
| `arkts-no-module-wildcards` | Запрет wildcard в именах модулей |
| `arkts-no-side-effects-imports` | Запрет side-effect импортов |
| `arkts-no-import-assertions` | Запрет import assertions |
| `arkts-no-import-default-as` | Запрет default-импортов как namespace |
| `arkts-no-misplaced-imports` | Импорты только в начале файла |
| `arkts-no-ns-as-obj` | Запрет namespace как объектов |
| `arkts-no-ns-statements` | Ограничения namespace |
| `arkts-no-require` | Запрет `require()` |
| `arkts-only-imported-variables` | Переменные должны импортироваться явно |
| `arkts-no-export-assignment` | Запрет `export =` |
| `arkts-no-umd` | Запрет UMD |
| `arkts-unique-names` | Уникальные имена в областях видимости |

#### 3.2.6. Правила декораторов

| Правило | Описание |
|---------|----------|
| `arkts-no-decorators-except-arkui` | Только декораторы ArkUI разрешены |

**Разрешённые декораторы** (34 шт., из `ArkUIDecoratorBlackList.ts`):

`Entry`, `Component`, `Reusable`, `CustomDialog`, `Consume`, `Link`, `LocalStorageLink`, `LocalStorageProp`, `ObjectLink`, `Prop`, `Provide`, `State`, `StorageLink`, `StorageProp`, `Builder`, `LocalBuilder`, `BuilderParam`, `Observed`, `Require`, `Sendable`, `Track`, `ComponentV2`, `ObservedV2`, `Trace`, `Local`, `Param`, `Once`, `Event`, `Monitor`, `Provider`, `Consumer`, `Computed`, `Type`, `Env`

#### 3.2.7. Правила Sendable (v1.1)

| Правило | Описание |
|---------|----------|
| `arkts-sendable-class-decorator` | Правила декоратора `@Sendable` на классах |
| `arkts-sendable-class-inheritance` | Правила наследования Sendable |
| `arkts-sendable-prop-types` | Разрешённые типы свойств |
| `arkts-sendable-explicit-field-type` | Обязательная аннотация типа полей |
| `arkts-sendable-definite-assignment` | Определённое присваивание |
| `arkts-sendable-obj-init` | Инициализация Sendable-объектов |
| `arkts-sendable-generic-types` | Generic-типы в Sendable |
| `arkts-sendable-closure-export` | Экспорт замыканий |
| `arkts-sendable-imported-variables` | Импортированные переменные |
| `arkts-sendable-function-decorator` | `@Sendable` на функциях |
| `arkts-sendable-function-assignment` | Присваивание Sendable-функций |
| `arkts-sendable-function-property` | Свойства Sendable-функций |
| `arkts-sendable-function-overload-decorator` | Перегрузка Sendable-функций |
| `arkts-sendable-function-as-expr` | Sendable-функция как выражение |
| `arkts-sendable-function-imported-variables` | Импортированные переменные в Sendable-функциях |
| `arkts-sendable-typealias-declaration` | Объявление Sendable type alias |
| `arkts-sendable-typealias-decorator` | Декоратор Sendable type alias |
| `arkts-sendable-as-expr` | Sendable в выражениях |
| `arkts-sendable-computed-prop-name` | Запрет вычисляемых имён свойств |
| `arkts-sendable-decorator-limited` | Ограничения декораторов на Sendable |
| `arkts-sendable-beta-compatible` | Совместимость с beta-версиями |
| `arkts-no-ts-sendable-type-inheritance` | Наследование Sendable из TS |

#### 3.2.8. Shared-модули и межмодульные проверки

| Правило | Описание |
|---------|----------|
| `arkts-shared-module-exports` | Правила экспорта shared-модулей |
| `arkts-no-ts-deps` | Запрет TS→TS зависимостей |
| `arkts-taskpool-concurrent-function-args` | Аргументы TaskPool-функций должны быть Concurrent |

#### 3.2.9. Прочие правила

| Правило | Описание |
|---------|----------|
| `arkts-limited-stdlib` | Ограниченный набор API стандартной библиотеки |
| `arkts-limited-throw` | Ограничения throw |
| `arkts-no-types-in-catch` | Запрет типов в catch |
| `arkts-no-aliases-by-index` | Запрет доступа по индексу |
| `arkts-no-props-by-index` | Запрет свойств по индексу |
| `arkts-no-func-props` | Запрет объявлений свойств-функций |
| `arkts-no-implicit-return-types` | Обязательные явные типы возврата |
| `arkts-no-definite-assignment` | Ограничения definite assignment |
| `arkts-no-ambient-decls` | Запрет ambient-объявлений |
| `arkts-no-globalthis` | Запрет `globalThis` |
| `arkts-no-inferred-generic-params` | Запрет выведенных generic-параметров |
| `arkts-strict` | Строгий режим |
| `arkts-as-casts` | Правила приведения типов |

---

## 4. Модульная система OpenHarmony

### 4.1. `oh_modules`

**Назначение:** `oh_modules` — замена `node_modules` в экосистеме OpenHarmony. Пакетный менеджер `ohpm` устанавливает зависимости в `oh_modules` вместо `node_modules`.

**Реализация:**
- Параметризация через `compilerOptions.packageManagerType = "ohpm"` (types.ts:6957)
- `isOhpm()` (ohApi.ts:235) — проверка типа пакетного менеджера
- `getModuleByPMType()` (ohApi.ts:252) — возвращает `"oh_modules"` или `"node_modules"`
- Все пути модулей параметризованы: `isOHModules()`, `isOHModulesDirectory()`, `isOHModulesReference()` (ohApi.ts:239-288)
- В модульном резолвере: `commonPackageFolders` включает `"oh_modules"` (utilities.ts:7895)
- `isTargetModulesDerectory()` (ohApi.ts) — `true` и для `node_modules`, и для `oh_modules`

**Файлы:** `src/compiler/ohApi.ts`, `src/compiler/moduleNameResolver.ts`, `src/compiler/utilities.ts`

**Первый PR:** !45 (2022-12-29) — "Support oh_modules and oh-package.json5"

---

### 4.2. `oh-package.json5`

**Назначение:** Формат манифеста пакета в формате JSON5 (расширенный JSON с поддержкой комментариев, trailing commas, и т.д.). Аналог `package.json`.

**Реализация:**
- `getPackageJsonByPMType()` (ohApi.ts:259) — возвращает `"oh-package.json5"` или `"package.json"`
- В `getPackageJsonInfo()` (moduleNameResolver.ts:2008): JSON5-файлы парсятся через `require("json5").parse()` вместо стандартного `readJson()`
- Поля как у `package.json`: `types`, `main`, `exports`, `versionPaths`
- Поле `oh-exports` — контроль экспорта (см. раздел 4.3)

**Первый PR:** !55 (2023) — "Support json5 format file parsing"

---

### 4.3. `oh-exports`

**Назначение:** Поле в `oh-package.json5`, которое ограничивает, какие модули могут быть импортированы из пакета внешним кодом.

**Реализация:**
- `ResolvedModule.isNotOhExport?: boolean` (types.ts:7439-7440)
- Если модуль разрешён, но не указан в `oh-exports` → `isNotOhExport = true`
- В program.ts (3822): модули с `isNotOhExport` исключаются из программы
- В checker.ts (4944): попытка импорта такого модуля даёт ошибку `Cannot_find_module_0_This_module_is_not_exported`

---

### 4.4. Ограничения импортов между TS и ETS

- **TS → ETS запрещён** (checker.ts:4952-4966): `.ts` файлы не могут импортировать `.ets`/`.d.ets` модули
- Диагностика: `Importing_ArkTS_files_in_JS_and_TS_files_is_forbidden`
- Исключение: `.d.ts` файлы внутри SDK и `.ts` файлы с `tsImportSendableEnable` могут импортировать `.ets` (checker.ts:5138-5145)
- В oh_modules подавляются все диагностики, кроме "forbidden import ArkTS" (program.ts:2712-2722)

---

## 5. Система Kit-импортов

### 5.1. Концепция

Kit-импорты — механизм компилятора, преобразующий символические имена `@kit.X` в реальные пути к файлам деклараций SDK на этапе парсинга, до начала проверки типов.

**Синтаксис:**
```typescript
import { Button, Text } from '@kit.ArkUI';
import { image } from '@kit.ImageKit';
```

После трансформации `@kit.ArkUI` заменяется на конкретный путь к `.d.ts`/`.d.ets` файлу в SDK.

### 5.2. Реализация

- `KIT_PREFIX = '@kit.'` (ohApi.ts:1235)
- `processKit()` (ohApi.ts:1588) — вызывается из парсера после парсинга всех выражений (parser.ts:1837)
- `getKitJsonObject()` (ohApi.ts:1265) — загружает конфигурацию kit'а из JSON-файла SDK (`kit_configs/<kitName>.json`)
- `createImportDeclarationForKit()` (ohApi.ts:1320) — создаёт новый import с реальным путём
- Трансформированные узлы помечаются `NodeFlags.KitImportFlags` и `virtual = true`
- Кэширование: `kitJsonCache` (ohApi.ts:1255)
- White-list API-модулей для дополнения атрибутов: `@ohos.arkui.*`, `@hms.hds.*` (ohApi.ts:1571)
- White-list Kit-модулей: `@kit.ArkUI`, `@kit.MediaLibraryKit`, `@kit.UIDesignKit` (ohApi.ts:1582)

### 5.3. Пути SDK

- `getSdkPath()` (ohApi.ts:1258) — извлекает путь SDK из `compilerOptions.etsLoaderPath`
- Для SDK 1.2 (mixed compiler) используется путь `openharmony/ets/dynamic/build-tools/ets-loader/kit_configs`

**Файлы:** `src/compiler/ohApi.ts`, `src/compiler/parser.ts`, `src/compiler/program.ts`

**Первый PR:** !46 (2023) — первые упоминания kit-обработки в связке с form typecheck framework

---

## 6. Проверка доступности API (`apiAvailable`)

### 6.1. Назначение

Механизм `apiAvailable` проверяет, что используемые API доступны в целевой версии SDK. Позволяет компилятору выдавать предупреждения при использовании API, которые помечены как доступные только с определённой версии.

### 6.2. Реализация

- `isApiAvailableVersionSpecifications` (checker.ts:1451) — host callback для проверки версий API
- `checkApiAvailableVersion()` (checker.ts:47221) — основная функция проверки
- `apiAvailableGetTypeOfNode()` (checker.ts:47216) — получение типа узла для проверки (добавлена поддежка `getTypeOfNode` в PR !858, май 2026)
- При несоответствии версии: `This_API_has_been_Special_Markings_exercise_caution_when_using_this_API`
- Интегрировано с системой аннотаций через `@Available(since: "...")` и `@Retention({policy: "source"})`

### 6.3. Варианты использования

- Проверка минимальной версии API: `@Available(since: "12")`
- Проверка диапазона версий: `@Available(since: "12", until: "15")`
- Source-retention: аннотации не попадают в скомпилированный вывод

**Файлы:** `src/compiler/checker.ts`, `src/compiler/ohApi.ts`

**Ключевые PR'ы:** !842 "merge 20260315-sharesourcefile into master", !848 "merge interfaceApiAvailable_master into master", !858 "merge interfaceApiAvailable_master into master"

---

## 7. Система Sendable

### 7.1. Назначение

Sendable — система статической типизации для конкурентного программирования в ArkTS. Гарантирует, что данные, передаваемые между потоками (через TaskPool и другие механизмы), являются безопасными для разделяемого доступа.

### 7.2. Ключевые концепции

- **`@Sendable` декоратор**: помечает класс, функцию или type alias как безопасный для передачи между потоками
- **`ISendable` интерфейс**: маркерный интерфейс, которому могут соответствовать только `@Sendable` классы
- **Sendable-правила линтера**: 20+ правил, проверяющих корректность Sendable-кода
- **TaskPool**: проверка аргументов TaskPool-функций — должны быть Sendable (`arkts-taskpool-concurrent-function-args`)

### 7.3. Ограничения Sendable

1. Поля Sendable-классов должны иметь явную аннотацию типа
2. Sendable-классы не могут использовать вычисляемые имена свойств
3. Наследование: только `@Sendable` класс может реализовывать `ISendable`
4. Sendable-объекты при добавлении в `collections.Array` должны быть Sendable
5. Импортированные переменные в Sendable-замыканиях проверяются

### 7.4. Реализация

- `isSendableFunctionOrType()` (ohApi.ts:1211) — идентификация Sendable-функций и типов
- `allowImportSendable()` (checker.ts:5138) — контроль импорта Sendable между TS и ETS
- `nodeCanBeDecorated()` (utilities.ts:2721) — `@Sendable` разрешён на `FunctionDeclaration` и `TypeAliasDeclaration`
- `disableSendableCheckRules?: string[]` — гранулярное отключение правил
- `tsImportSendableEnable?: boolean` — глобальное разрешение импорта Sendable из TS

**Ключевые PR'ы:** !311 "[ArkTS Linter] Initial implementation of Sendable rules", !373 "Implement Sendable closure", !410 "Implement the object push to collection.Array has to be sendable", !415 "Sendable Function & Sendable TypeAlias"

---

## 8. Система аннотаций

### 8.1. Назначение

Аннотации — декларативный механизм метаданных в ArkTS, аналогичный аннотациям в Java. Поддерживаются декларации аннотаций, свойства аннотаций, и их применение к объявлениям.

### 8.2. Синтаксис

```arkts
@Retention({policy: RetentionPolicy.Source})
annotation Available {
  since: string;
}

@Available({since: "12"})
class MyComponent {
  // ...
}
```

### 8.3. Типы AST

- `SyntaxKind.AnnotationDeclaration` — объявление аннотации
- `SyntaxKind.AnnotationPropertyDeclaration` — свойство аннотации
- Аннотации могут иметь константные значения свойств

### 8.4. Source-retention

Аннотации с `@Retention({policy: "source"})`:
- Проверяются на этапе компиляции
- **НЕ попадают** в вывод (emit) — удаляются трансформером (ohApi.ts:633, 673)
- Используются для: `@Available`, `@SuppressWarnings`

### 8.5. Проверки аннотаций

- Аннотации не могут использоваться в HAR-пакетах с JS-выводом
- `@Retention` может применяться только к декларациям аннотаций
- Конструкторы аннотаций проверяются на корректность
- Пустые аннотации (`@Anno()`) поддерживаются

**Ключевые PR'ы:** !353 "Add annotations support, stage 1", !509 "TSC annotation fix", !646 "TSC annotation bugfix"

---

## 9. Инфраструктура компилятора

### 9.1. Строгие режимы

- **`strictCheckerOnly`**: включает строгую проверку типов только для `.ets` файлов
- **`disableStrictCheckPaths`**: пути, исключённые из строгой проверки (добавлено в !853, апрель 2026)
- **Strict mode for OH modules** (`enableStrictOHModuleCheck`): строгая проверка `oh_modules` (!733, !739)
- **Strict diagnostics to getSemanticDiagnostics** (!847)
- Линтер автоматически форсирует: `strictNullChecks`, `strictFunctionTypes`, `strictPropertyInitialization`, `noImplicitReturns` (checker.ts:1456-1466)

### 9.2. Инкрементальная компиляция

- **TSC incremental**: переиспользование `.tsbuildinfo` для инкрементальной компиляции
- **Linter incremental**: линтер переиспользует механизм инкрементальной компиляции TSC (!653, !656)
- **Кэширование линтера**: `arktsLinterDiagnosticsPerFile` в builder state
- **Linter cache** (!800: "addCacheForLinter")
- **Parallel linter**: параллельное выполнение линтера (!211)

### 9.3. Оптимизации производительности

- **Performance dotting**: инструментирование производительности в TSC (!572) и линтере (!599)
- **Memory dotting**: отслеживание пикового использования памяти (!514, `src/compiler/memorydotting/`)
- **Освобождение TypeChecker'ов**: оптимизация потребления памяти (!405)
- **Оптимизация сигнатур и функций** (!575, !576)
- **Ускорение инкрементальной сборки**: параллельный линтер + кэш языкового сервиса (!258)
- **Управление GC**: принудительный сбор мусора в режиме сборки только в памяти (!840)

### 9.4. Обработка `.so` файлов

- Игнорирование ошибок для модулей `.so` (!69, !78)
- Опция `tsImportSoCheck` для контроля импорта `.so`
- Проверка типов `.so` файлов: `feature/enable-so-file-type-check` (!844)

### 9.5. SourceMap и обфускация

- **ArkGuard**: поддержка обфускации для тестов (`--enable-arkguard`, !362)
- **Compact sourcemap**: исправление некорректных sourcemap при обфускации (!490)
- **useTsHar**: флаги для обфускации в контексте аннотаций (!657)
- **Obfuscation whitelist**: поддержка wildcard'ов в белом списке обфускации (!359)

### 9.6. Build-система и инфраструктура

- **BUILD.gn**: интеграция с системой сборки OpenHarmony
- **bundle.json**: метаданные пакета для OpenHarmony (!128)
- **OAT.xml**: Open Source Audit Tool для проверки лицензий (!1, !96)
- **Docker**: поддержка Docker-окружения
- **Python-скрипты сборки**: `compile_typescript.py`

### 9.7. Импорт и модули

- **Import-lazy**: ленивая загрузка модулей с ключевым словом `lazy` (!421, !554, !577)
- **Dynamic import**: исправление динамических импортов (!598)
- **Rollup shared TSC AST**: переиспользование AST между модулями (!851)
- **Isolate declarations**: поддержка изолированных деклараций (!709)

### 9.8. RollupSharedTSCAST и CustomEnv

- **RollupSharedTSCAST**: переиспользование общего AST между компиляциями для ускорения (!851)
- **CustomEnv**: поддержка пользовательских переменных окружения в TSC (!855, апрель 2026)
- **WithEnv**: адаптация TSC для работы с переменными окружения (!791572b4, май 2026)

---

## 10. Хронология версий

| Дата | PR | Событие |
|------|-----|---------|
| 2022-02-07 | !1 | Форк: добавление OAT.xml, README.OpenSource |
| 2022-02-28 | !2 | Первый код ArkTS: "change ts sourcecode for ets rules" |
| 2022-03 | !6, !7, !9, !13 | Ранние исправления: method overload, pageTransition, parse styles, if component |
| 2022-04-22 | !20 | `@Builder` и `@Styles` в `export Struct` |
| 2022-06-21 | !24 | `@Styles` и `@Extend` |
| 2022-12-29 | !40 | `.d.ets` файлы и сохранение ArkUI-декораторов |
| 2022-12-29 | !43 | `@Concurrent` декоратор |
| 2022-12-29 | !45 | `oh_modules` и `oh-package.json5` |
| 2023-02 | !46 | Form typecheck framework (начало kit-обработки) |
| 2023-05 | !86 | `struct` как идентификатор в TS/JS |
| 2023-05 | !88 | `@AnimatableExtend` |
| 2023-07 | !99 | ArkTSLinter в компиляции |
| 2023-07 | !100 | Запрет импорта `.ets` из `.ts`/`.d.ts` |
| 2023-07 | !106 | ESObject как алиас `any` |
| 2023-08 | !116 | Standard mode для ArkTS Module |
| 2023-10 | !191 | Версия 4.2.3-r8 |
| 2023-11-03 | !195 | **Апмерж до TypeScript 4.9.5** |
| 2023-11-08 | !200 | Версия 4.9.5-r2 |
| 2023-12 | !211 | Параллельный ArkTS Linter |
| 2023-12 | !233 | Декоратор `@Require` |
| 2024-01 | !231 | Инкрементальный API для ets_checker |
| 2024-01 | !266 | Флаг `skip tsc oh_modules check` |
| 2024-03 | !270 | Изоляция свойств ArkUI |
| 2024-03 | !283 | Строгая/нестрогая проверка на одной программе |
| 2024-04 | !311 | **Первая реализация Sendable-правил** |
| 2024-04 | !323 | Sendable API, collections.Array |
| 2024-05 | !353 | **Система аннотаций, stage 1** |
| 2024-06 | !356 | `arkts-no-side-effect-import` |
| 2024-06 | !365 | Правила shared-модулей |
| 2024-07 | !373 | Sendable-замыкания |
| 2024-08 | !386 | TS import ETS |
| 2024-09 | !410 | Object push to collection.Array must be sendable |
| 2024-10 | !415 | Sendable Function & Sendable TypeAlias |
| 2024-10 | !421 | Import-lazy с ключевым словом `lazy` |
| 2024-11 | !432 | Ограничения `@Sendable` декоратора |
| 2025-01 | !460 | Оптимизированная строгая модель |
| 2025-02 | !509 | Исправления аннотаций TSC |
| 2025-04 | !514 | Memory dotting |
| 2025-05 | !548 | `@Require` не требует инициализации |
| 2025-06 | !572 | Performance dotting в TSC |
| 2025-07 | !653 | Linter переиспользует инкрементальный механизм TSC |
| 2025-08 | !663 | **Проверка `@throws`** |
| 2025-09 | !670 | Intl.NumberFormat.formatRange() |
| 2025-10 | !680 | TaskPool function args must be Concurrent |
| 2025-11 | !698 | Find Record from static |
| 2025-12 | !709 | Isolate declarations |
| 2026-01 | !733 | enableStrictOHModuleCheck |
| 2026-02 | !743 | Checklist |
| 2026-03 | !800 | addCacheForLinter |
| 2026-03 | !804 | oh_export |
| 2026-03 | !810 | `@SuppressWarnings` |
| 2026-04 | !842 | share resource file |
| 2026-04 | !849 | `@Retention` |
| 2026-04 | !853 | `disableStrictCheckPaths` |
| 2026-04 | !855 | `CustomEnv` |
| 2026-05 | !851 | RollupSharedTSCAST |
| 2026-05 | !858 | `interfaceApiAvailable` + `getTypeOfNode` |

---

## Приложение A: Полный список правил ArkTS Linter

### A.1. Правила ограничения типов (14)
`arkts-no-any-unknown`, `arkts-no-conditional-types`, `arkts-no-intersection-types`, `arkts-no-mapped-types`, `arkts-no-indexed-signatures`, `arkts-no-type-query`, `arkts-no-utility-types`, `arkts-no-symbol`, `arkts-no-typing-with-this`, `arkts-no-obj-literals-as-types`, `arkts-limited-esobj`, `arkts-instanceof-ref-types`, `arkts-structural-typing`, `arkts-strict-typing`, `arkts-strict-typing-required`

### A.2. Правила ограничения синтаксиса (24)
`arkts-no-func-expressions`, `arkts-no-nested-funcs`, `arkts-no-generators`, `arkts-no-destruct-assignment`, `arkts-no-destruct-decls`, `arkts-no-destruct-params`, `arkts-no-var`, `arkts-no-comma-outside-loops`, `arkts-no-spread`, `arkts-no-for-in`, `arkts-no-delete`, `arkts-no-in`, `arkts-no-is`, `arkts-no-standalone-this`, `arkts-no-jsx`, `arkts-no-regexp-literals`, `arkts-no-new-target`, `arkts-no-private-identifiers`, `arkts-no-as-const`, `arkts-no-func-apply-bind-call`, `arkts-no-noninferrable-arr-literals`, `arkts-no-untyped-obj-literals`, `arkts-identifiers-as-prop-names`, `arkts-no-polymorphic-unops`

### A.3. Правила классов и наследования (12)
`arkts-extends-only-class`, `arkts-implements-only-iface`, `arkts-no-decl-merging`, `arkts-no-enum-merging`, `arkts-no-enum-mixed-types`, `arkts-no-extend-same-prop`, `arkts-no-method-reassignment`, `arkts-no-classes-as-obj`, `arkts-no-class-literals`, `arkts-no-ctor-prop-decls`, `arkts-no-call-signatures`, `arkts-no-ctor-signatures-*` (3), `arkts-no-multiple-static-blocks`, `arkts-no-prototype-assignment`

### A.4. Правила модулей и импортов (12)
`arkts-no-ts-deps`, `arkts-no-module-wildcards`, `arkts-no-side-effects-imports`, `arkts-no-import-assertions`, `arkts-no-import-default-as`, `arkts-no-misplaced-imports`, `arkts-no-ns-as-obj`, `arkts-no-ns-statements`, `arkts-no-require`, `arkts-only-imported-variables`, `arkts-no-export-assignment`, `arkts-no-umd`, `arkts-unique-names`

### A.5. Правила Sendable (22)
`arkts-sendable-class-decorator`, `arkts-sendable-class-inheritance`, `arkts-sendable-prop-types`, `arkts-sendable-explicit-field-type`, `arkts-sendable-definite-assignment`, `arkts-sendable-obj-init`, `arkts-sendable-generic-types`, `arkts-sendable-closure-export`, `arkts-sendable-imported-variables`, `arkts-sendable-function-decorator`, `arkts-sendable-function-assignment`, `arkts-sendable-function-property`, `arkts-sendable-function-overload-decorator`, `arkts-sendable-function-as-expr`, `arkts-sendable-function-imported-variables`, `arkts-sendable-typealias-declaration`, `arkts-sendable-typealias-decorator`, `arkts-sendable-as-expr`, `arkts-sendable-computed-prop-name`, `arkts-sendable-decorator-limited`, `arkts-sendable-beta-compatible`, `arkts-no-ts-sendable-type-inheritance`

### A.6. Прочие правила (10)
`arkts-limited-stdlib`, `arkts-limited-throw`, `arkts-no-types-in-catch`, `arkts-no-decorators-except-arkui`, `arkts-no-aliases-by-index`, `arkts-no-props-by-index`, `arkts-no-func-props`, `arkts-no-implicit-return-types`, `arkts-no-definite-assignment`, `arkts-no-ambient-decls`, `arkts-no-globalthis`, `arkts-no-inferred-generic-params`, `arkts-strict`, `arkts-as-casts`

---

## Приложение B: Ключевые файлы исходного кода

| Файл | Назначение | Размер |
|------|-----------|--------|
| `src/compiler/ohApi.ts` | ArkTS-специфичные API: декораторы, Sendable, kit-трансформация, oh_modules, @throws | ~2122 строк |
| `src/compiler/types.ts` | SyntaxKind, AST-интерфейсы, Extension, ScriptKind, CompilerOptions | Стандартный + ArkTS-дополнения |
| `src/compiler/parser.ts` | Парсинг struct, Builder/Styles/Extend, EtsComponentExpression, kit-трансформация | Стандартный + ArkTS-дополнения |
| `src/compiler/checker.ts` | Проверка типов, apiAvailable, @throws, Sendable, аннотации | Стандартный + ArkTS-дополнения |
| `src/compiler/scanner.ts` | Лексический анализ: `struct` в ETS-контексте | Стандартный + ArkTS-дополнения |
| `src/compiler/utilities.ts` | ScriptKind, isInETSFile, isCalledStructDeclaration | Стандартный + ArkTS-дополнения |
| `src/compiler/program.ts` | Linter-интеграция, фильтрация диагностик, oh_modules | Стандартный + ArkTS-дополнения |
| `src/compiler/moduleNameResolver.ts` | Разрешение oh_modules, oh-package.json5 | Стандартный + ArkTS-дополнения |
| `src/linter/ArkTSLinter_1_1/TypeScriptLinter.ts` | Основной класс линтера v1.1 | ~114KB |
| `src/linter/ArkTSLinter_1_1/Problems.ts` | FaultID enum (80+ категорий) | - |
| `src/linter/ArkTSLinter_1_1/InteropTypescriptLinter.ts` | Линтер для `.ts` в смешанных проектах | ~19KB |
| `src/linter/Common/ArkUIDecoratorBlackList.ts` | Белый список декораторов ArkUI (34 шт.) | - |
| `src/compiler/performanceDotting.ts` | Инструментирование производительности | Huawei |
| `src/compiler/memorydotting/memoryDotting.ts` | Инструментирование памяти | Huawei |

---

## Приложение C: Разработчики ArkTS

Топ-10 разработчиков по количеству коммитов (Huawei-оригинальные коммиты):

| Разработчик | Коммитов |
|-------------|----------|
| xucheng46 | 47 |
| liyancheng2 | 32 |
| lizhonghan1 | 27 |
| caiyu30 | 27 |
| zhangchen168 | 21 |
| lihong | ~20 |
| houhaoyu | ~15 |
| wangyunxiao8 | ~12 |
| yenan10 | ~12 |
| shanweiqian | ~10 |

Всего: ~47 разработчиков Huawei + партнёров.
