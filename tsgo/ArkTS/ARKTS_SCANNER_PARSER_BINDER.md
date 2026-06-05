# ArkTS: изменения в сканере, парсере и биндере

> Выжимка из `ARKTS_SPECIFICATION.md` для реализации в tsgo (Go-компилятор ArkTS).
> Охватывает ТОЛЬКО scanner, parser, binder и связанные типы.
> Чекер (checker), линтер, oh_modules, kit-трансформация — за рамками этого документа.

---

## 1. Сканер (`src/compiler/scanner.ts`)

Два новых ключевых слова: `struct` (контекстно-зависимое) и `lazy` (безусловное).

### 1.1. Регистрация ключевых слов

```typescript
struct: SyntaxKind.StructKeyword,   // строка 176 — между string и super
lazy: SyntaxKind.LazyKeyword,       // строка 185 — для import lazy X from "..."
```
`struct` и `lazy` добавлены в карту `textToKeyword`.

**Разница в поведении:**
- `struct` — **контекстно-зависимый**: распознаётся как ключевое слово только в `.ets` файлах (см. 1.3). Вне ETS-контекста — `Identifier`.
- `lazy` — **безусловный**: всегда распознаётся как `LazyKeyword`. Синтаксис `import lazy X from "..."` не существует в стандартном TypeScript, поэтому коллизий нет.

### 1.2. Флаг ETS-контекста (строки 109, 996, 1044, 1059-1061)

```typescript
var inEtsContext: boolean = false;

function setEtsContext(isEtsContext: boolean): void {
    inEtsContext = isEtsContext;
}
```

- `setEtsContext` добавлен в интерфейс `Scanner` (строка 109)
- Переменная `inEtsContext` (строка 996)
- Реализация (строка 1059)

### 1.3. Условная отмена ключевого слова (строки 1586-1589)

```typescript
if (keyword === SyntaxKind.StructKeyword && !inEtsContext) {
    token = SyntaxKind.Identifier;
}
```

**Это все изменения в сканере.** Два новых ключевых слова (`struct`, `lazy`) + флаг ETS-контекста. Вне `.ets`-файлов `struct` — обычный идентификатор. Никаких новых токенов для декораторов (`@Builder`, `@Extend` и т.д.) не добавлялось — они используют стандартный механизм декораторов.

### Резюме для tsgo

- Добавить `struct` и `lazy` в карту ключевых слов
- Хранить флаг `inEtsContext` в сканере
- При lookup: если найдено `StructKeyword` но `!inEtsContext` → вернуть `Identifier`
- `LazyKeyword` — без условий

---

## 2. Парсер (`src/compiler/parser.ts`)

### 2.1. Система флагов EtsFlags

Двухуровневая система контекста:

**Уровень 1 — `NodeFlags.EtsContext`** (`1 << 30`): включён ли режим ArkTS вообще.
Определяется по `ScriptKind.ETS` или расширению `.ets`.

**Уровень 2 — `EtsFlags`** (13 флагов, `src/compiler/types.ts` строки 848-862):

```typescript
export const enum EtsFlags {
    None =                       0,
    StructContext =              1 << 1,   // внутри struct
    EtsExtendComponentsContext = 1 << 2,   // внутри @Extend
    EtsStylesComponentsContext = 1 << 3,   // внутри @Styles
    EtsBuildContext =            1 << 4,   // внутри метода build()
    EtsBuilderContext =          1 << 5,   // внутри @Builder
    EtsStateStylesContext =      1 << 6,   // внутри stateStyles
    EtsComponentsContext =       1 << 7,   // можно парсить EtsComponentExpression
    EtsNewExpressionContext =    1 << 8,   // внутри new-выражения
    UICallbackContext =          1 << 9,   // внутри UI-колбэка (стрелочная функция)
    SyntaxComponentContext =     1 << 10,  // внутри ForEach/LazyForEach
    SyntaxDataSourceContext =    1 << 11,  // внутри первого аргумента ForEach
    NoEtsComponentContext =      1 << 12,  // запрет создания EtsComponentExpression
}
```

Функции для установки/проверки флагов (строки 2054-2325):

```typescript
// Установка (bitwise OR/AND-NOT)
setStructContext(val), setEtsComponentsContext(val), setEtsNewExpressionContext(val),
setEtsExtendComponentsContext(val), setEtsStylesComponentsContext(val),
setEtsBuildContext(val), setEtsBuilderContext(val), setEtsStateStylesContext(val),
setUICallbackContext(val), setSyntaxComponentContext(val),
setSyntaxDataSourceContext(val), setNoEtsComponentContext(val)

// Проверка
inEtsContext(): boolean               // NodeFlags.EtsContext
inStructContext(): boolean            // EtsContext + StructContext
inEtsComponentsContext(): boolean     // EtsContext + EtsComponentsContext
inEtsNewExpressionContext(): boolean  // EtsContext + EtsNewExpressionContext
inEtsExtendComponentsContext()        // EtsContext + EtsExtendComponentsContext
inEtsStylesComponentsContext()        // EtsContext + EtsStylesComponentsContext
inBuildContext()                      // EtsContext + StructContext + EtsBuildContext
inBuilderContext()                    // EtsContext + EtsBuilderContext
inEtsStateStylesContext()             // EtsContext + (build|builder|extend|styles) + EtsStateStylesContext
inUICallbackContext()                 // EtsContext + (build|builder) + UICallbackContext
inSyntaxComponentContext()            // EtsContext + (build|builder) + SyntaxComponentContext
inSyntaxDataSourceContext()           // EtsContext + (build|builder) + SyntaxDataSourceContext
inNoEtsComponentContext()             // EtsContext + (build|builder) + NoEtsComponentContext
inAllowAnnotationContext()            // (EtsContext || Arkguard) && etsAnnotationsEnable
```

### 2.2. Вход в ETS-режим при инициализации SourceFile (строки 1777-1794)

```typescript
case ScriptKind.ETS:
    contextFlags = NodeFlags.EtsContext;
    break;
// ...
if (fileName.endsWith(Extension.Ets)) {
    contextFlags = NodeFlags.EtsContext;
}
scanner.setEtsContext(inEtsContext());
```

Два способа активации: явный `ScriptKind.ETS` или расширение `.ets` в имени файла.

### 2.3. Переменные состояния парсера для ETS

```typescript
var etsFlags: EtsFlags;                                              // строка 1574
var extendEtsComponentDeclaration: { name, type, instance } | undef; // строка 1608
var stylesEtsComponentDeclaration: { name, type, instance } | undef; // строка 1610
var currentStructName: string | undefined;                           // строка 1615
var structStylesComponents: Map<string, { structName, kind }>;       // строка 1620
var stateStylesRootNode: string | undefined;                         // строка 1622
var fileStylesComponents: Map<string, SyntaxKind>;                   // строка 1624
var _firstArgumentExpression: boolean;    // getter/setter для отслеживания первого аргумента ForEach
var _repeatEachRest: boolean;            // getter/setter для отслеживания Repeat.each rest
```

Очистка при сбросе состояния (строки 1816-1821):
```typescript
extendEtsComponentDeclaration = undefined;
stylesEtsComponentDeclaration = undefined;
stateStylesRootNode = undefined;
fileStylesComponents.clear();
structStylesComponents.clear();
```

**`firstArgumentExpression`** и **`repeatEachRest`** — геттеры/сеттеры, используемые при парсинге аргументов синтаксических компонентов (ForEach, Repeat) для определения контекста источника данных.

### 2.4. `doInDecoratorContext` — детект ETS-декораторов (строки 2210-2219)

При входе в контекст декоратора проверяется, является ли декоратор `@Extend` или `@Styles`:

```typescript
function doInDecoratorContext(decorators, func) {
    // Если это @Extend(ComponentName) → установка EtsExtendComponentsContext
    if (hasEtsExtendDecoratorNames(decorators, opts)) {
        extendEtsComponentDeclaration = getEtsExtendDecoratorsComponentNames(decorators, opts);
        setEtsExtendComponentsContext(true);
    }
    // Если это @Styles → установка EtsStylesComponentsContext
    if (hasEtsStylesDecoratorNames(decorators, opts)) {
        stylesEtsComponentDeclaration = getEtsStylesDecoratorComponentNames(decorators, opts);
        setEtsStylesComponentsContext(true);
    }
    func();
    // сброс контекстов после выхода
}
```

### 2.5. `parseStructDeclaration` (строки 8670-8742)

```typescript
function parseStructDeclarationOrExpression(pos, hasJSDoc, decorators, modifiers) {
    parseExpected(SyntaxKind.StructKeyword);
    setStructContext(true);
    const name = parseNameOfClassDeclarationOrExpression();
    const typeParameters = parseTypeParameters();
    const heritageClauses = parseHeritageClauses();
    // Если нет heritage clauses и customComponent задан → виртуальный extends
    if (!heritageClauses && customComponent) {
        heritageClauses = createVirtualHeritageClauses(customComponent);
    }
    const members = parseStructMembers(pos);
    clearStructStylesComponents();
    setStructContext(false);
    return factory.createStructDeclaration(...);
}
```

### 2.5. `parseStructMembers` (строки 8806-8849) — виртуальный конструктор

Самая важная часть синтаксического сахара struct:

1. Парсит члены через `parseClassMembers()`
2. Для каждого `PropertyDeclaration` создаёт **виртуальный `PropertySignature`** (с optional `?`)
3. Собирает все PropertySignature в **виртуальный `TypeLiteralNode`**
4. Создаёт виртуальный параметр конструктора: `value?: { prop1?: T1; prop2?: T2; ... }`
5. Добавляет виртуальный параметр `##storage?: LocalStorage`
6. Создаёт виртуальный constructor с этими параметрами
7. Вставляет constructor первым членом

```typescript
// Пример: из
@Component struct Test {
  @State count: number = 0;
  build() { ... }
}

// Автоматически генерируется конструктор:
// constructor(value?: { count?: number }, ##storage?: LocalStorage)
```

**`createVirtualHeritageClauses`** (строка 8732): создаёт синтетический `extends CustomComponent`, если struct не имеет явного наследования.

**`finishVirtualNode`** (строка 8851): обёртка над `finishNode` с `virtual: true` для создания виртуальных AST-узлов.

### 2.6. Struct/Builder-хелперы и auto-readonly

**`isTokenInsideStructBuild`** (строка 8095): проверяет, что имя метода совпадает с настроенным render-методом (по умолчанию `build`).

**`isTokenInsideStructPageTransition`** (строка 8108): проверяет, что имя метода == `pageTransition`.

**`isTokenInsideStructBuilder`**: проверяет, что метод находится внутри struct и имеет `@Builder`.

**`tryParseConstructorDeclaration`**: пытается парсить объявление конструктора struct (если пользователь объявил свой конструктор, виртуальный не генерируется).

**`parseConstructorName`**: парсинг имени конструктора для struct.

**`hasParamAndNoOnceDecorator`** (строка 8471): проверка `@Param` без `@Once` → свойство должно автоматически получить `readonly`.

**`hasEnvDecorator`** (строка 8488): проверка `@Env`/`@CustomEnv` → свойство должно автоматически получить `readonly`.

**`parseClassElement`** (строки 8557-8560): в struct для свойств с `@Param`/`@Env` без `@Once` автоматически добавляется `readonly`:
```typescript
if (inStructContext() && isPropertyDeclaration(node) && hasParamAndNoOnceDecorator(node)) {
    addVirtualReadonlyModifier(node);
}
```

**`parseModifiers`** (строки 8517-8529): инжекция виртуального `readonly` модификатора в список модификаторов свойства.

### 2.6. `EtsComponentExpression` — выражения UI-компонентов (строки 6865-6930)

```typescript
// AST-узел (types.ts строки 2563-2569):
interface EtsComponentExpression extends PrimaryExpression, Declaration {
    kind: SyntaxKind.EtsComponentExpression;
    expression: LeftHandSideExpression;      // имя компонента
    typeArguments?: NodeArray<TypeNode>;     // type arguments (виртуальные)
    arguments: NodeArray<Expression>;        // аргументы вызова
    body?: Block;                            // тело (дочерние компоненты)
}
```

**`isCurrentTokenAnEtsComponentExpression`** (строка 6865):
```typescript
function isCurrentTokenAnEtsComponentExpression(): boolean {
    if (!inEtsComponentsContext() || inNoEtsComponentContext()) return false;
    const components = sourceFileCompilerOptions.ets?.components ?? [];
    return components.includes(scanner.getTokenText());
}
```
Проверяет: (1) мы в ETS-компонентном контексте, (2) текущий токен — имя компонента из `compilerOptions.ets.components` (Column, Row, Text, Button, ...).

**`parseEtsComponentExpression`** (строка 6873):
```typescript
function parseEtsComponentExpression(): EtsComponentExpression {
    const name = parseBindingIdentifier();
    const args = parseArgumentList();
    const body = token() === OpenBraceToken ? parseFunctionBlock(None) : undefined;
    return factory.createEtsComponentExpression(name, args, body);
}
```

**Интеграция в `parsePrimaryExpression`** (строка 6928):
```typescript
if (isCurrentTokenAnEtsComponentExpression() && !inEtsNewExpressionContext()) {
    return parseEtsComponentExpression();
}
```

**`makeEtsComponentExpression`** (строка 5351) — преобразование CallExpression + `{`:
```typescript
if ((inBuildContext() || inBuilderContext()) && inUICallbackContext()
    && isCallExpression(expr) && token() === OpenBraceToken) {
    return makeEtsComponentExpression(expr, pos);
}
```
Когда вызов функции в UI-колбэке сопровождается `{`, он переинтерпретируется как компонентное выражение.

### 2.7. Декоратор-управляемые контексты в `parseDeclaration` (строки 7738-7760)

При парсинге объявлений проверяются декораторы для установки ETS-контекстов:

```typescript
// @Extend(ComponentName)
if (hasEtsExtendDecoratorNames(decorators, opts)) {
    extendEtsComponentDeclaration = getEtsExtendDecoratorsComponentNames(...);
    setEtsExtendComponentsContext(true);
}
// @Styles
if (hasEtsStylesDecoratorNames(decorators, opts)) {
    stylesEtsComponentDeclaration = getEtsStylesDecoratorComponentNames(...);
    setEtsStylesComponentsContext(true);
}
// @Builder
if (hasEtsBuilderDecoratorNames(decorators, opts)) {
    setEtsComponentsContext(true);
}
```

### 2.8. `parseFunctionDeclaration` с ETS-контекстом (строки 8012-8058)

```typescript
// 1. @Builder → EtsBuilderContext, UICallbackContext
if (hasEtsBuilderDecoratorNames(...)) setEtsBuilderContext(true), setUICallbackContext(true);

// 2. @Styles → тип возврата из конфигурации
if (inEtsStylesComponentsContext() && stylesEtsComponentDeclaration) {
    typeParameters = parseEtsTypeParameters(pos);
}

// 3. После парсинга — очистка контекстов
setEtsBuilderContext(false);
setEtsExtendComponentsContext(false);
setEtsStylesComponentsContext(false);

// 4. Внедрение виртуального типа возврата, если не указан явно
if (!type && inEtsExtendComponentsContext()) {
    type = parseEtsType(extendEtsComponentDeclaration.type);
}
if (!type && inEtsStylesComponentsContext()) {
    type = parseEtsType(stylesEtsComponentDeclaration.type);
}
```

### 2.9. `parseMethodDeclaration` с ETS-контекстом (строки 8117-8186)

Самая сложная интеграция:

```typescript
// 1. Сохранение контекстов
const savedBuildContext = inBuildContext();
const savedBuilderContext = inBuilderContext();
const savedUICallbackContext = inUICallbackContext();

// 2. Метод build() → EtsBuildContext
if (isBuildMethod(name, opts)) setEtsBuildContext(true);

// 3. @Builder → EtsBuilderContext
if (hasEtsBuilderDecoratorNames(...)) setEtsBuilderContext(true), setUICallbackContext(true);

// 4. @Styles в struct → запись в structStylesComponents
if (inStructContext() && hasEtsStylesDecoratorNames(...)) {
    structStylesComponents.set(methodName, { structName, kind });
    stylesEtsComponentDeclaration = getEtsStylesDecoratorComponentNames(...);
    setEtsStylesComponentsContext(true);
}

// 5. build/pageTransition/@Builder → EtsComponentsContext
if (inStructContext() && isBuildOrPageTransitionOrBuilderMethod(...)) {
    setEtsComponentsContext(true);
}

// 6. Тип возврата для @Styles методов используют parseEtsTypeParameters
if (inEtsStylesComponentsContext() && stylesEtsComponentDeclaration) {
    typeParameters = parseEtsTypeParameters(pos);
}

// 7. Восстановление контекстов после парсинга
setEtsBuildContext(savedBuildContext);
setEtsBuilderContext(savedBuilderContext);
setUICallbackContext(savedUICallbackContext);

// 8. Внедрение виртуального типа возврата
if (!type && inEtsExtendComponentsContext()) type = parseEtsType(extendEtsComponentDeclaration.type);
if (!type && inEtsStylesComponentsContext()) type = parseEtsType(stylesEtsComponentDeclaration.type);
```

### 2.10. Property access в `@Extend`/`@Styles`/stateStyles (строки 6237-6248)

При разборе левой части (member expression):

```typescript
// В @Extend-контексте: .prop → virtualIdentifier(extendEtsComponentDeclaration.instance).prop
if (inEtsExtendComponentsContext() && extendEtsComponentDeclaration && token() === DotToken) {
    expression = parseEtsIdentifier(extendEtsComponentDeclaration.instance);
}
// В @Styles-контексте: .prop → virtualIdentifier(stylesEtsComponentDeclaration.instance).prop
if (inEtsStylesComponentsContext() && stylesEtsComponentDeclaration && token() === DotToken) {
    expression = parseEtsIdentifier(stylesEtsComponentDeclaration.instance);
}
// В stateStyles-контексте: .prop → virtualIdentifier(rootNodeInstance).prop
if (stateStylesRootNode && inEtsStateStylesContext() && token() === DotToken) {
    expression = parseEtsIdentifier(`${stateStylesRootNode}Instance`);
}
```

### 2.11. Виртуальные type arguments для компонентов (строки 6730-6809)

При парсинге списка аргументов вызова:

```typescript
// Для свойств stateStyles → установка stateStyles-контекста
if (isStateStylesProperty(...)) setEtsStateStylesContext(true);

// Для ForEach/LazyForEach/Repeat → SyntaxComponentContext
if (isSyntaxComponent(...)) setSyntaxComponentContext(true);

// Атрибуты компонента → виртуальные type arguments
typeArguments = parseEtsTypeArguments(componentName, propertyName);

// @Extend/@Styles → получить виртуальный компонент и внедрить type arguments
const virtualComponent = getVirtualEtsComponent(...);
typeArguments = parseEtsTypeArguments(virtualComponent, propertyName);

// UI-колбэки параметров → SyntaxComponentContext
if (isParamUICallback(...)) setSyntaxComponentContext(true);

// Очистка stateStyles после property access
setEtsStateStylesContext(false);
```

**`isValidVirtualTypeArgumentsContext`** (строка 6808): проверяет, что текущий контекст допускает виртуальные type arguments (ETS-компоненты или @Extend/@Styles).

### 2.12. Стрелочные функции (строки 5429-5441, 5724-5735)

При парсинге стрелочных функций:

```typescript
// Переход SyntaxComponentContext → UICallbackContext
if (inSyntaxComponentContext() || inSyntaxDataSourceContext()) {
    setUICallbackContext(true);
}
if (inSyntaxComponentContext()) {
    setSyntaxComponentContext(false); // consumed
}
// Запрет компонентных выражений в UI-колбэке (по необходимости)
if (!inUICallbackContext()) setNoEtsComponentContext(true);
```

### 2.13. `parseArgumentExpression` — SyntaxDataSourceContext (строки 6957-6976)

```typescript
// Первый аргумент синтаксического компонента → источник данных
if (inSyntaxComponentContext() && firstArgumentExpression) {
    setSyntaxDataSourceContext(true);
}
```

### 2.14. `parseNewExpressionOrNewDotTarget` — сохранение EtsNewExpressionContext (строки 7098, 7117)

```typescript
// Сохранение контекста new-выражения, чтобы предотвратить интерпретацию
// имени компонента как EtsComponentExpression внутри new
const savedEtsNewExpressionContext = inEtsNewExpressionContext();
setEtsNewExpressionContext(true);
// ... парсинг new-выражения ...
setEtsNewExpressionContext(savedEtsNewExpressionContext);
```

### 2.15. `parseArrowFunctionExpressionBody` — исключение struct из ASI (строка 5774)

```typescript
// struct не должен провоцировать automatic semicolon insertion
// внутри стрелочной функции
// (!inEtsContext() || token() !== SyntaxKind.StructKeyword)
```

### 2.16. `forEachChild`-функции для новых узлов

Три новые функции обхода дочерних узлов:

| Функция | Для узла |
|---------|----------|
| `forEachChildInAnnotationPropertyDeclaration` | `AnnotationPropertyDeclaration` |
| `forEachChildInEtsComponentExpression` | `EtsComponentExpression` |
| `forEachChildInAnnotationDeclaration` | `AnnotationDeclaration` |

### 2.17. `setLanguageVersionByFilePath` — ETS-версия языка

### 2.18. Аннотации (`@interface`) — строки 7706-7729, 8188-8201, 8621, 8256

```typescript
// parseAnnotationDeclaration
function parseAnnotationDeclaration(pos, hasJSDoc, decorators, modifiers) {
    parseExpected(AtToken);           // @
    parseExpected(InterfaceKeyword);  // interface
    // проверка: между @ и interface не должно быть пробелов/токенов
    const name = parseBindingIdentifier();
    const members = parseAnnotationMembers(); // ParsingContext.AnnotationMembers
    return factory.createAnnotationDeclaration(...);
}

// parseAnnotationPropertyDeclaration
function parseAnnotationPropertyDeclaration(pos, hasJSDoc, decorators, modifiers) {
    const name = parsePropertyName();
    const type = parseOptionalTypeAnnotation();
    const initializer = parseOptionalInitializer();
    return factory.createAnnotationPropertyDeclaration(...);
}

// parseAnnotationElement (строка 8621) — диспетчер элементов аннотации
// parseAnnotationMembers (строка 8802) — обёртка для списка членов
```

**`isAnnotationMemberStart`** (строка 8256) — валидация членов `@interface`:
- Статические блоки **запрещены**
- Геттеры/сеттеры **запрещены**
- Конструкторы **запрещены**
- Индексные сигнатуры **запрещены**
- Вычисляемые имена свойств **запрещены**
- Разрешены только: `name: type` или `name: type = value`

Аннотации активируются когда `inAllowAnnotationContext()` = true (т.е. ETS-контекст + `etsAnnotationsEnable`).

**`tryParseDecorator`** (строки 8418-8420): пропускает `@interface` в контексте аннотаций (чтобы не парсить аннотацию как декоратор).

В `parseDeclarationWorker` (строка 7805):
```typescript
if (token() === AtToken && inAllowAnnotationContext() && nextToken() === InterfaceKeyword) {
    return parseAnnotationDeclaration(...);
}
```

**`canFollowModifier`** (строка 3014): `@interface` может следовать за модификатором, когда аннотации включены:
```typescript
|| token() === AtToken && inAllowAnnotationContext() && lookAhead(() => nextToken() === InterfaceKeyword)
```

### 2.19. Точки входа `StructKeyword` в парсере

`struct` должен быть валидным стартом во всех этих местах (только в ETS-контексте):

| Функция | Строка | Условие |
|---------|--------|---------|
| `parseStatement` | 7639 | `inEtsContext()` |
| `isStartOfStatement` | 7578 | `inEtsContext()` |
| `isStartOfLeftHandSideExpression` | 5190 | `inEtsContext()` |
| `isDeclaration` | 7471 | `inEtsContext()` |
| `nextTokenCanFollowDefaultKeyword` | 3020 | `inEtsContext()` |
| `isStartOfExpressionStatement` | 5247 | `!inEtsContext() \|\| token !== StructKeyword` (исключение) |

### 2.23. Прочие token-уровневые модификации

`struct` и `@interface` добавляются как валидные токены в следующих проверках:

| Функция | Строка | Изменение |
|---------|--------|----------|
| `isListElement` | — | `StructKeyword` — валидный элемент списка |
| `isListTerminator` | — | `StructKeyword` — терминатор списка (как class) |
| `isReusableClassMember` | — | `StructKeyword` — допускается |
| `canFollowModifier` | 3014 | `@interface` может следовать за модификатором |
| `nextTokenCanFollowDefaultKeyword` | 3020 | `export default struct` разрешён в ETS |
| `isDeclaration` | 7471 | `struct` + `@interface` — начала деклараций |

### 2.24. `SourceFile.markedKitImportRange`

После вызова `processKit()` на этапе парсинга (в `parseSourceFileWorker`, строка 1840), диапазоны трансформированных kit-импортов сохраняются в `sourceFile.markedKitImportRange` для использования чекером и эмиттером.

```typescript
// parseSourceFileWorker (строка 1840):
processKit(factory, statements, sdkPath, sourceFile.markedKitImportRange, ...);
```

### 2.20. Вспомогательные функции парсинга

**`parseEtsIdentifier`** (строка 2882):
```typescript
function parseEtsIdentifier(pos: number): Identifier {
    const text = internIdentifier(stylesEtsComponentDeclaration!.type);
    return finishVirtualNode(factory.createIdentifier(text), pos, pos);
}
```

**`parseEtsType`** (строка 5140): парсит тип как ссылку на тип с очисткой флагов.

**`parseEtsTypeReferenceWorker`** (строка 5154): создаёт виртуальный TypeReferenceNode.

**`parseEtsTypeParameters`** (строка 4145): создаёт массив из одного TypeParameterDeclaration.

**`parseEtsTypeArguments`** (строка 4149): создаёт массив из одного type argument.

### 2.21. `parseExpected` — обход для stateStyles (строка 2520)

```typescript
if (token() !== kind && stateStylesRootNode && inEtsStateStylesContext()) {
    return true;  // молча принять отсутствующий токен
}
```

### 2.22. `parseIdentifier` — виртуальный идентификатор для stateStyles (строки 2850-2858)

```typescript
if (stateStylesRootNode && inEtsStateStylesContext() && token() === DotToken) {
    return finishVirtualNode(
        factory.createIdentifier(`${stateStylesRootNode}Instance`), pos, pos
    );
}
```

---

## 3. Биндер (`src/compiler/binder.ts`)

### 3.1. `StructDeclaration` — как `ClassDeclaration` (строки 2141, 2991, 3564)

```typescript
// ContainerFlags (строка 2141):
case SyntaxKind.StructDeclaration:
    return ContainerFlags.IsContainer;

// bind() (строка 2991):
case SyntaxKind.StructDeclaration:
    inStrictMode = true;
    return bindClassLikeDeclaration(node as ClassLikeDeclaration);

// bindClassLikeDeclaration (строка 3564):
if (node.kind === SyntaxKind.ClassDeclaration || node.kind === SyntaxKind.StructDeclaration) {
    bindBlockScopedDeclaration(node, SymbolFlags.Class, SymbolFlags.ClassExcludes);
}
```

`StructDeclaration` биндится идентично `ClassDeclaration`:
- `SymbolFlags.Class`
- Block-scoped declaration
- Включает strict mode

### 3.2. `AnnotationDeclaration` (строки 2142, 2996, 3600-3615)

```typescript
// ContainerFlags (строка 2142):
case SyntaxKind.AnnotationDeclaration:
    return ContainerFlags.IsContainer;

// bind() (строка 2996):
case SyntaxKind.AnnotationDeclaration:
    return bindAnnotationDeclaration(node as AnnotationDeclaration);

// bindAnnotationDeclaration (строки 3600-3615):
function bindAnnotationDeclaration(node: AnnotationDeclaration): void {
    bindBlockScopedDeclaration(node, SymbolFlags.Class | SymbolFlags.Annotation, SymbolFlags.ClassExcludes);
    const prototypeSymbol = createSymbol(SymbolFlags.Property | SymbolFlags.Prototype, "prototype");
    symbol.exports!.set(prototypeSymbol.escapedName, prototypeSymbol);
    prototypeSymbol.parent = symbol;
}
```

Особенности:
- `SymbolFlags.Class | SymbolFlags.Annotation`
- Создаёт `prototype`-символ (как класс)
- Block-scoped

### 3.3. `AnnotationPropertyDeclaration`

**ContainerFlags** (строка 2189):
```typescript
case SyntaxKind.AnnotationPropertyDeclaration:
    return (node as AnnotationPropertyDeclaration).initializer
        ? ContainerFlags.IsControlFlowContainer
        : 0;
```
Если есть initializer → control flow container (как обычное свойство).

**bindPropertyWorker** (строки 3057-3064):
```typescript
function bindPropertyWorker(node: PropertyDeclaration | PropertySignature | AnnotationPropertyDeclaration) {
    const isOptional = (!isAnnotationPropertyDeclaration(node) && node.questionToken
        ? SymbolFlags.Optional : SymbolFlags.None);
    const annotationPropHasDefaultValue = (isAnnotationPropertyDeclaration(node) && node.initializer !== undefined)
        ? SymbolFlags.Optional : SymbolFlags.None;
    return bindPropertyOrMethodOrAccessor(node, includes | isOptional | annotationPropHasDefaultValue, excludes);
}
```

Отличия `AnnotationPropertyDeclaration` от `PropertyDeclaration`:
- **Нет `questionToken`** — optional не определяется по `?`
- **Optional по initializer'у** — если есть `initializer`, свойство считается optional (`SymbolFlags.Optional`)

### 3.4. `this.prop = x` в конструкторах (строка 3267)

```typescript
case SyntaxKind.AnnotationPropertyDeclaration:
    // обрабатывается как PropertyDeclaration
```
`AnnotationPropertyDeclaration` включено в проверки присваивания `this.prop` в конструкторах.

---

## 4. Типы (`src/compiler/types.ts` → `internal/ast/`)

> **Важно для tsgo:** AST-узлы в typescript-go **генерируются из схемы** `_scripts/ast.json` → `ast_generated.go`.
> Добавление новых узлов (StructDeclaration, EtsComponentExpression, etc.) делается через редактирование `ast.json` и запуск кодогенератора.
> Ручной код в `ast.go` нужен только для узлов с `handWritten: true` (сейчас только `SourceFile`).
> Все struct/factory/visitor/clone/ForEachChild/Is*() — генерируются автоматически.

### 4.1. Новые SyntaxKind

```typescript
StructKeyword = ...             // строка 140 (между ClassKeyword и ConstKeyword)
AnnotationPropertyDeclaration   // строка 235 (между PropertyDeclaration и MethodSignature)
EtsComponentExpression          // строка 286 (между ArrowFunction и DeleteExpression)
StructDeclaration               // строка 334 (перед AnnotationDeclaration)
AnnotationDeclaration           // строка 335 (между StructDeclaration и InterfaceDeclaration)
```

### 4.2. Новые NodeFlags

```typescript
KitImportFlags = 1 << 29    // строка 828 — узлы преобразованных kit-импортов
EtsContext = 1 << 30        // строка 828 — файл парсится как ArkTS
```

`EtsContext` также включён в `ContextFlags` (строка 837):
```typescript
ContextFlags = ... | EtsContext
```

### 4.3. EtsFlags (строки 848-862)

См. раздел 2.1 выше — 13 флагов от `StructContext` до `NoEtsComponentContext`.

### 4.4. Новые SymbolFlags и TypeFlags

```typescript
SymbolFlags.Annotation = 1 << 28    // строка 5539
TypeFlags.Annotation = 1 << 27      // строка 6035 (не используется в парсере/биндере)
```

### 4.5. ScriptKind и Extension

```typescript
ScriptKind.ETS = 8            // строка 7101
Extension.Ets = ".ets"        // строка 7496
Extension.Dets = ".d.ets"     // строка 7497
```

`.ets` и `.d.ets` включены в `supportedTSExtensions` (utilities.ts строка 8254).

### 4.6. Новые AST-интерфейсы

```typescript
// StructDeclaration (строки 3356-3361)
interface StructDeclaration extends ClassLikeDeclarationBase, DeclarationStatement {
    kind: SyntaxKind.StructDeclaration;
    modifiers?: NodeArray<ModifierLike>;
    name?: Identifier;
}

// AnnotationDeclaration (строки 3363-3368)
interface AnnotationDeclaration extends DeclarationStatement {
    kind: SyntaxKind.AnnotationDeclaration;
    modifiers?: NodeArray<ModifierLike>;
    name: Identifier;
    members: NodeArray<AnnotationElement>;
}

// EtsComponentExpression (строки 2563-2569)
interface EtsComponentExpression extends PrimaryExpression, Declaration {
    kind: SyntaxKind.EtsComponentExpression;
    expression: LeftHandSideExpression;
    typeArguments?: NodeArray<TypeNode>;
    arguments: NodeArray<Expression>;
    body?: Block;
}

// AnnotationPropertyDeclaration (строки 1718-1724)
interface AnnotationPropertyDeclaration extends AnnotationElement, JSDocContainer {
    kind: SyntaxKind.AnnotationPropertyDeclaration;
    parent: AnnotationDeclaration;
    name: PropertyName;
    type?: TypeNode;
    initializer?: Expression;
}

// AnnotationElement (строки 3386-3389)
interface AnnotationElement extends NamedDeclaration {
    _annnotationElementBrand: any;
    name: PropertyName;
}

// Annotation — type alias для Decorator (строка 1606)
type Annotation = Decorator;
```

### 4.7. Обновлённые union-типы

| Union-тип | Добавлено |
|-----------|----------|
| `ClassLikeDeclarationBase.kind` | `StructDeclaration`, `AnnotationDeclaration` (строка 3342) |
| `ClassLikeDeclaration` | `StructDeclaration`, `AnnotationDeclaration` (строка 5374) |
| `Declaration` | `StructDeclaration`, `AnnotationDeclaration` (строка 1095) |
| `DeclarationStatement` | `StructDeclaration`, `AnnotationDeclaration` (строка 1329) |
| `Statement` | `StructDeclaration`, `AnnotationDeclaration` (строка 1279) |
| `NamedDeclaration` | `AnnotationPropertyDeclaration`, `AnnotationDeclaration` (строка 1177) |
| `HasChildren` | `AnnotationPropertyDeclaration` (строка 1007) |
| `ModifierSyntaxKind` / `KeywordSyntaxKind` | `StructKeyword` (строка 595) |

### 4.8. CompilerOptions для ETS

```typescript
// В CompilerOptions (строки 6956-6977):
ets?: EtsOptions;
etsLoaderPath?: string;
tsImportSendableEnable?: boolean;
etsAnnotationsEnable?: boolean;
disableSendableCheckRules?: string[];
strictCheckerOnly?: boolean;
```

```typescript
// EtsOptions (строки 6981-7013):
interface EtsOptions {
    render: { method: string[]; decorator: string[] };
    components: string[];
    libs: string[];
    extend: {
        decorator: string[];
        components: { name: string; type: string; instance: string }[];
    };
    styles: {
        decorator: string;
        component: { name: string; type: string; instance: string };
        property: string;
    };
    concurrent: { decorator: string };
    customComponent?: string;
    propertyDecorators: { name: string; needInitialization: boolean }[];
    emitDecorators: { name: string; emitParameters: boolean }[];
    syntaxComponents: {
        paramsUICallback: string[];
        attrUICallback: { name: string; attributes: string[] }[];
    };
}
```

### 4.9. `CheckMode.SkipEtsComponentBody` (типы, строка 1268)

```typescript
CheckMode.SkipEtsComponentBody = 1 << 7  // пропуск тела ETS-компонента при проверке типов
```
Используется чекером для пропуска тел EtsComponentExpression при разрешении сигнатур (оптимизация).

### 4.10. `Decorator.annotationDeclaration` (types.ts, строка 1598)

```typescript
interface Decorator {
    // ...
    readonly annotationDeclaration?: AnnotationDeclaration;  // ссылка на декларацию аннотации
}
```
Поле связывает декоратор с соответствующей AnnotationDeclaration (если декоратор является аннотацией). Используется чекером для `resolveAnnotation`.

### 4.11. `SourceFile.markedKitImportRange`

```typescript
interface SourceFile {
    // ...
    markedKitImportRange?: KitImportRange[];  // диапазоны трансформированных kit-импортов
}
```

### 4.12. `ResolvedModule.isNotOhExport`

```typescript
// types.ts строка 7439
interface ResolvedModule {
    // ...
    isNotOhExport?: boolean;  // модуль не в oh-exports
}
```

### 4.13. `nodeCanBeDecorated` (utilities.ts строка 2721)

```typescript
// @Sendable разрешён на FunctionDeclaration и TypeAliasDeclaration
function nodeCanBeDecorated(node, ...): boolean {
    // ...
    return isArkTsDecorator(...) || isSendableFunctionOrType(...);
}
```

### 4.14. Фабричные функции (`factory/nodeFactory.ts`, ~100 строк)

Новые factory-функции для создания ETS-узлов:

| Функция | Узел |
|---------|------|
| `createStructDeclaration` / `updateStructDeclaration` | `StructDeclaration` |
| `createAnnotationDeclaration` / `updateAnnotationDeclaration` | `AnnotationDeclaration` |
| `createAnnotationPropertyDeclaration` / `updateAnnotationPropertyDeclaration` | `AnnotationPropertyDeclaration` |
| `createEtsComponentExpression` / `updateEtsComponentExpression` | `EtsComponentExpression` |

Модифицированы:
- `createDecorator` / `updateDecorator` — добавлен параметр `annotationDeclaration`

### 4.15. Node-test функции (`factory/nodeTests.ts`)

| Функция | Проверка |
|---------|----------|
| `isStructDeclaration(node)` | `kind === SyntaxKind.StructDeclaration` |
| `isAnnotationDeclaration(node)` | `kind === SyntaxKind.AnnotationDeclaration` |
| `isAnnotationPropertyDeclaration(node)` | `kind === SyntaxKind.AnnotationPropertyDeclaration` |
| `isEtsComponentExpression(node)` | `kind === SyntaxKind.EtsComponentExpression` |
| `isAnnotation(node)` | `Decorator` с непустым `annotationDeclaration` |
| `isDecoratorOrAnnotation(node)` | Базовый тест для `Decorator` |
| `isDecorator(node)` | `Decorator` **без** `annotationDeclaration` |

### 4.16. Visitor-функции (`visitorPublic.ts`)

Новые case в visitor'ах для обхода ETS-узлов:

| Синтаксис | Добавлен в |
|-----------|-----------|
| `SyntaxKind.Decorator` | Обход декораторов с `annotationDeclaration` |
| `SyntaxKind.AnnotationPropertyDeclaration` | Обход свойств аннотаций |
| `SyntaxKind.EtsComponentExpression` | Обход UI-компонентных выражений |
| `SyntaxKind.StructDeclaration` | Обход struct |
| `SyntaxKind.AnnotationDeclaration` | Обход аннотаций |

### 4.17. `ohApi.ts` — функции, используемые парсером

Функции из `src/compiler/ohApi.ts` (1915 строк), вызываемые напрямую из парсера:

**Декораторы:**

| Функция | Назначение |
|---------|-----------|
| `hasEtsExtendDecoratorNames(decorators, opts)` | Поиск `@Extend` |
| `hasEtsStylesDecoratorNames(decorators, opts)` | Поиск `@Styles` |
| `hasEtsBuildDecoratorNames(decorators, opts)` | Поиск render-метода |
| `hasEtsBuilderDecoratorNames(decorators, opts)` | Поиск `@Builder`/`@LocalBuilder` |
| `hasEtsConcurrentDecoratorNames(decorators, opts)` | Поиск `@Concurrent` |
| `isTokenInsideBuilder(decorators)` | Проверка `@Builder` на узле |
| `getEtsExtendDecoratorsComponentNames(...)` | Извлечение имени компонента из `@Extend(Name)` |
| `getEtsStylesDecoratorComponentNames(...)` | Извлечение имени компонента из `@Styles` |
| `isArkTsDecorator(node)` | Проверка: декоратор является ArkTS-декоратором |
| `isEtsFunctionDecorators(decorators, opts)` | Проверка на ETS-декораторы функции |
| `getReservedDecoratorsOfEtsFile(decorators)` | Фильтрация ETS-декораторов |
| `getReservedDecoratorsOfStructDeclaration(decorators)` | Фильтрация декораторов struct |
| `ensureEtsDecorators` / `concatenateDecoratorsAndModifiers` / `getEffectiveDecorators` | Манипуляции с декораторами |

**Контекст:**

| Функция | Назначение |
|---------|-----------|
| `isInEtsFile(node)` | Проверка ETS-файла |
| `isInEtsFileWithOriginal(node)` | Проверка с учётом исходного узла |
| `inEtsStylesContext(node)` | Проверка Styles-контекста вне парсера |

**Kit-трансформация (~440 строк в ohApi.ts):**

| Функция | Назначение |
|---------|-----------|
| `processKit(...)` | Трансформация `@kit.*` импортов |
| `getSdkPath(opts)` | Путь к SDK |
| `getKitJsonObject(sdkPath, kitName)` | Чтение kit-конфигурации из JSON |
| `createImportDeclarationForKit(...)` | Создание реального import из kit-имени |
| `markKitImport(...)` | Маркировка трансформированного импорта |
| `processKitStatementSuccess(...)` | Обработка успешно трансформированного kit-импорта |
| `supplementNamedBindings(...)` | Дополнение атрибутных биндингов |
| `preProcessSpecifiedImportDeclaration(...)` | Предварительная обработка импортов |
| `processExtendComponentMap(...)` | Обработка карты @Extend-компонентов |


---

## 5. Резюме для реализации в tsgo

### Сканер

1. `struct` и `lazy` в карте ключевых слов → `StructKeyword`, `LazyKeyword`
2. Флаг `inEtsContext` + метод `setEtsContext`
3. При lookup: если `StructKeyword && !inEtsContext` → `Identifier`
4. `LazyKeyword` — без условий

### Парсер

1. **Два уровня флагов**: `NodeFlags.EtsContext` + 13 `EtsFlags`
2. **Вход в ETS-режим** по `ScriptKind.ETS` или `.ets` расширению
3. **`StructDeclaration`**: парсинг, виртуальный конструктор из свойств + `LocalStorage`
4. **Struct/Builder-хелперы**: `isTokenInsideStructBuild`, `isTokenInsideStructPageTransition`, `isTokenInsideStructBuilder`, `tryParseConstructorDeclaration`, `parseConstructorName`
5. **Auto-readonly**: `hasParamAndNoOnceDecorator`, `hasEnvDecorator`, инжекция в `parseClassElement`/`parseModifiers`
6. **`createVirtualHeritageClauses`**: синтетический `extends CustomComponent`
7. **`finishVirtualNode`**: создание виртуальных узлов
8. **`doInDecoratorContext`**: детект `@Extend`/`@Styles`
9. **`EtsComponentExpression`**: `ComponentName(args) { body }` и `CallExpression + {`
10. **Декоратор-управляемые контексты**: `@Builder`, `@Extend(Name)`, `@Styles`
11. **Виртуальные узлы**: type arguments, property access для @Extend/@Styles, return types, stateStyles-идентификаторы
12. **`AnnotationDeclaration`** (`@interface`): парсинг + `isAnnotationMemberStart` + `parseAnnotationElement` + `tryParseDecorator`
13. **Стрелочные функции**: проброс UICallbackContext/NoEtsComponentContext, исключение struct из ASI
14. **`parseNewExpressionOrNewDotTarget`**: сохранение EtsNewExpressionContext
15. **`firstArgumentExpression` / `repeatEachRest`**: отслеживание первого аргумента ForEach
16. **`StructKeyword`** в 6+ точках проверки statement/declaration start + token-уровневых модификациях
17. **`isValidVirtualTypeArgumentsContext`**: проверка контекста для виртуальных type arguments
18. **3 `forEachChild`-функции** для новых узлов
19. **`setLanguageVersionByFilePath`**: ETS-версия
20. **`SourceFile.markedKitImportRange`**: сохранение диапазонов kit-трансформации
21. **`processKit`** (~440 строк): вызов в `parseSourceFileWorker`

### Биндер

1. `StructDeclaration` → `bindClassLikeDeclaration` с `SymbolFlags.Class`
2. `AnnotationDeclaration` → `bindAnnotationDeclaration` с `SymbolFlags.Class | SymbolFlags.Annotation`
3. `AnnotationPropertyDeclaration` → optional по initializer'у (не по `?`)
4. `AnnotationPropertyDeclaration` в `this.prop = x` проверках

### Types

- 6 новых SyntaxKind (StructKeyword, LazyKeyword + 4 узла: AnnotationPropertyDeclaration, EtsComponentExpression, StructDeclaration, AnnotationDeclaration)
- 13 EtsFlags, 2 NodeFlags, 1 SymbolFlags, 1 CheckMode
- 4 новых AST-интерфейса + AnnotationElement + Annotation (type alias)
- `Decorator.annotationDeclaration` — связь декоратор→аннотация
- `SourceFile.markedKitImportRange` — kit-диапазоны
- `ScriptKind.ETS`, `Extension.Ets`, `Extension.Dets`
- `EtsOptions` — конфигурация всех ETS-фич
- 10 фабричных функций, 7 node-test функций, 8 visitor-функций

### ohApi.ts (подмножество для парсера)

- 13+ функций декораторов (`hasEts*DecoratorNames`, `isArkTsDecorator`, `isEtsFunctionDecorators`)
- 3 функции контекста (`isInEtsFile`, `isInEtsFileWithOriginal`, `inEtsStylesContext`)
- 9+ функций kit-трансформации (`processKit` + хелперы)
