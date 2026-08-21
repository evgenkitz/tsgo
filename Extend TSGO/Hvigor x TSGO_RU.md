# Hvigor x TSGO: проектирование механизма единой компиляции

> **Дата: 2026-08-13** **Статус: стадия ревью дизайна**

---

## I. Анализ текущего состояния

### 1.1 Текущая архитектура компиляции ArkTS

```
 Компиляция каждого модуля независимо:

  CompileArkTS(moduleA)                    CompileArkTS(moduleB)
    |                                         |
    +- initDefaultArkCompileConfig()          +- initDefaultArkCompileConfig()
    |  -> генерирует ProjectConfig A          |  -> генерирует ProjectConfig B
    |                                         |
    +- submitArkCompileWork(configA)          +- submitArkCompileWork(configB)
    |  -> workerPool.submit -> worker_thread  |  -> workerPool.submit -> worker_thread
    |                                         |
    +- Внутри Worker'а:                       +- Внутри Worker'а:
       runArkPack(configA)                       runArkPack(configB)
         +- generateConfig()                      +- generateConfig()
         |  -> rollup plugins[]                   |  -> rollup plugins[]
         +- runHvigorCompile()                    +- runHvigorCompile()
         |  -> rollup.build()                     |  -> rollup.build()
         |    +- resolveId                        |    +- resolveId
         |    +- load (ets-loader)                |    +- load (ets-loader)
         |    +- transform (tsc/etsTransform)     |    +- transform (tsc/etsTransform)
         |    +- generateBundle (genAbc)          |    +- generateBundle (genAbc)
         |    +- writeBundle                      |    +- writeBundle
         +- возвращает CompileEvent[]             +- возвращает CompileEvent[]
```

### 1.2 Ключевые узкие места

|**Проблема**|**Причина**|**Влияние**|
|---|---|---|
|**N независимых инстансов tsc**|**каждый воркер независимо загружает** `runArkPack`->`runHvigorCompile`-> rollup -> tsc|**повторный разбор типов, LanguageService нельзя переиспользовать между модулями**|
|**повторный разбор типов между модулями**|**каждый воркер независимо разбирает `.d.ts` зависимых HAR**|**один и тот же `.d.ts` повторно разбирается N воркерами**|
|**накладные расходы холодного старта воркера**|**каждый воркер должен загрузить** `@ohos/hvigor-arkts-compose` + SDK-плагины|**в daemon-режиме смягчается резидентными воркерами, но первая сборка всё равно медленная**|

### 1.3 Ключевые пути кода

```
 Цепочка выполнения задачи CompileArkTS:

 AbstractArkCompile.doTaskAction()          abstract-ark-compile.ts:443
  +- initDefaultArkCompileConfig()         -> генерирует ProjectConfig одного модуля
  +- worker-id-helper.getWorkerIdWithModule()  -> аффинность модуль->воркер
  +- submitArkCompileWork()                run-ark.ts:35
       +- workerPool.submit(task, './job.js/run', { workInput: config })
            +- Внутри Worker'а:
               job.ts:28  run(config)
                 +- runArkPack(config, log, ...)    arkts-pack.ts:207
                     +- generateConfig()            -> rollup config + plugins
                     +- runHvigorCompile(config)    -> rollup.build()
```

### 1.4 Существующая переиспользуемая инфраструктура

|**Инфраструктура**|**Расположение**|**Ценность для TSGO**|
|---|---|---|
|`TaskScheduleOptimizationService`|`task-schedule-optimization.ts:12`|**уже обходит все задачи** `CompileArkTS` и цепочки зависимостей — естественный источник списка модулей|
|`hvigorCore.getCompileDependArr()`|`hvigor-core.ts:99`|**уже собирает Set путей всех задач компиляции**|
|`GlobalProjectDataService.compileProjectModel`|`global-project-data-service.ts:207`|**уже есть полная таблица модулей** `Record<sourceRoot, ModuleInfoType>`|
|`BuildModeManager.targetBuildOptionMap`|`build-mode-manager.ts:59`|**уже есть слитые** `BuildOpt` для каждой пары module x target|
|`WorkerPool`+`pendingPromises`|`worker-pool-impl.ts`+`task-runner.ts:91`|**поддерживает одиночный submit work + ожидание одного promise несколькими задачами**|
|`projectTaskDag`|`task-directed-acyclic-graph.ts:181`|**глобальный DAG, можно запрашивать зависимости задач**|

---

## II. Проектирование механизма взаимодействия с TSGO

### 2.1 Цели проектирования

1. **Единая компиляция**: hvigor собирает конфигурации всех модулей к компиляции и передаёт их TSGO за один раз
2. **Управление переключателем**: включается через свойство `coreParameter.properties`, по умолчанию выключено, исходный путь сохраняется
3. **DAG не ломается**: существующая структура зависимостей задач не меняется, downstream-задачи ничего не замечают
4. **Раздача результатов**: результаты, возвращённые TSGO, раздаются по модулям обратно в соответствующие задачи `CompileArkTS`
5. **Совместимость с инкрементальностью**: сохраняется возможность инкрементального пропуска на уровне модулей

### 2.2 Ядро проектирования: паттерн Shared Barrier (общий барьер + пакетная отправка)

```
 Поток выполнения в режиме TSGO:

  CompileArkTS(A)                CompileArkTS(B)                CompileArkTS(C)
    |                               |                               |
    +- initDefaultArkCompileConfig  +- initDefaultArkCompileConfig  +- initDefaultArkCompileConfig
    |  -> ProjectConfig A           |  -> ProjectConfig B           |  -> ProjectConfig C
    |                               |                               |
    +- coordinator.register(A)      +- coordinator.register(B)      +- coordinator.register(C)
         |                               |                               |
         |  +---------------------------+-------------------------------+
         |  |
         |  v  когда registeredCount === expectedCount:
         |  coordinator.submitBatchWork()
         |  -> workerPool.submit(1 work, workInput: { configs: [A,B,C] })
         |       |
         |       +- Внутри Worker'а: tsgoRunner.run(configs)
         |            -> TSGO единообразно компилирует все модули
         |            -> возвращает Map<moduleName, ModuleCompileResult>
         |
         v  все задачи CompileArkTS ждут один общий batchPromise через механизм pendingPromises:
         (не await внутри doTaskAction, а pendingPromises.add(processingPromise))
         task-runner видит hasPendingPromises → handleTaskPendingPromises
         → Promise.all(pendingPromises).then(onTaskFinished) → разблокирует downstream
```

**Ключевые моменты дизайна**:

- **Новых задач нет**: не нужно вставлять новую задачу `BatchCompileArkTS` в DAG — избегаем рефакторинга DAG
- **CompileArkTS становится «сборщиком конфигураций + потребителем результатов»**: больше не делает независимый submit воркера, а регистрирует конфигурацию + ждёт пакетный результат
- **Последний зарегистрировавшийся триггерит отправку**: когда число регистраций достигает `expectedCount`, последняя выполнившаяся задача `CompileArkTS` запускает пакетную отправку
- **Общий Promise-барьер (через pendingPromises)**: все задачи `CompileArkTS` регистрируют promise обработки результата в `this.pendingPromises`, после завершения TSGO все разблокируются вместе. **Нельзя `await`-ить `batchPromise` в `doTaskAction`** — иначе заблокируется рекурсивное планирование task-runner, последующие задачи `CompileArkTS` не смогут быть извлечены из очереди, и наступит дедлок

> **Ключевое ограничение: doTaskAction не может await-ить долго не разрешающийся promise**
>
> **Цепочка выполнения task-runner:** `runTaskFromQueue → await executeOneTask → await doTaskAction`; **если `doTaskAction` сделает `await` `batchPromise` (не разрешённого), весь task-runner зависнет, последующие задачи `CompileArkTS` никогда не будут запланированы → `expectedCount` никогда не будет достигнут → дедлок**.
>
> **В исходном механизме** `workerPool.submit` — синхронная отправка: в `task.pendingPromises` кладётся promise без await, **`doTaskAction` сразу возвращается, task-runner продолжает планировать следующую задачу.** Режим TSGO обязан переиспользовать тот же паттерн: `this.pendingPromises.add(processingPromise)`, а не `await`.

### 2.2.1 Механизм выполнения task-runner и избежание дедлока

**Последовательная рекурсивная цепочка планирования task-runner** (`task-runner.ts:30-89`):

```
 runTaskFromQueue()
  ├─ popRunnableTask()                    // извлекает задачу из очереди готовых
  ├─ await executeOneTask(task)           // ← AWAIT: ждёт завершения doTaskAction
  │    └─ await coreTask.execute()
  │         └─ await this.doTaskAction()
  │
  ├─ hasPendingPromises = task.pendingPromises.get().length > 0
  ├─ if (hasPendingPromises)
  │    handleTaskPendingPromises(task)    // регистрирует асинхронную цепочку, не блокирует
  └─ await runTaskFromQueue()             // ← рекурсивно планирует следующую задачу
```

**Почему исходный механизм не зависает**:

`workerPool.submit` (`worker-pool-impl.ts:270-290`) — **синхронная отправка**:

```
 // Внутри workerPool.submit:
 const promise = new Promise((resolve, reject) => {
  tcb.on(WORK_DONE, () => resolve(true));   // resolve только по завершении воркера
 });
 coreTask.pendingPromises.add(promise);       // ← кладём в pendingPromises, без await
 return tcb;                                  // ← синхронный возврат
```

`doTaskAction` сразу возвращается → `executeOneTask` сразу возвращается → `runTaskFromQueue` видит `hasPendingPromises=true` → вызывает `handleTaskPendingPromises` (регистрирует асинхронную цепочку `Promise.all().then(onTaskFinished)`) → продолжает рекурсию `runTaskFromQueue`, планируя следующую задачу.

**Схема TSGO обязана переиспользовать этот паттерн**:

```
 // Правильная запись:
 this.pendingPromises.add(processingPromise);  // add, без await
 // doTaskAction сразу возвращается

 // Неправильная запись (приведёт к дедлоку):
 const myResult = await myResultPromise;  // batchPromise не разрешён → task-runner зависает → дедлок
```

**Исправленный полный поток выполнения**:

```
 runTaskFromQueue()
  ├─ pop CompileArkTS(A)
  │   doTaskAction: registerConfig(A) → pendingPromises.add(promiseA) → возврат
  ├─ hasPendingPromises=true → handleTaskPendingPromises(A)
  ├─ runTaskFromQueue()
  │   ├─ pop CompileArkTS(B)
  │   │   doTaskAction: registerConfig(B) → pendingPromises.add(promiseB) → возврат
  │   ├─ handleTaskPendingPromises(B)
  │   ├─ runTaskFromQueue()
  │   │   ├─ pop CompileArkTS(C)  ← последняя
  │   │   │   registerConfig(C): count===expected → submitBatchWork() → запускает TSGO
  │   │   │   pendingPromises.add(promiseC) → возврат
  │   │   ├─ handleTaskPendingPromises(C)
  │   │   └─ runTaskFromQueue() ← задач больше нет, возврат
  │   │
  │   │  TSGO компилирует, все pendingPromises не разрешены ...
  │   │
  │   └─ TSGO завершён → batchPromise resolve
  │       ├─ promiseA resolve → Promise.all([promiseA]).then(onTaskFinished(A)) → разблокирует downstream A
  │       ├─ promiseB resolve → Promise.all([promiseB]).then(onTaskFinished(B)) → разблокирует downstream B
  │       └─ promiseC resolve → Promise.all([promiseC]).then(onTaskFinished(C)) → разблокирует downstream C
```

**Проверка зависимостей DAG**:

**Предпосылка схемы: между задачами `CompileArkTS` всех модулей нет DAG-зависимостей, они могут одновременно оказаться в `runnableTaskQueue`.**

**Из** `arkts-task-initializer.ts:78-85`:

```
 declareDepends = (): string[] => {
  return [GenerateLoaderJson.name, CompileResource.name]
    .concat(getBuildProfileDependence(this.targetService));
 }
```

Межмодульная зависимость `getBuildProfileDependence` (`arkts-task-initializer.ts:220-231`) — это `CREATE_HAR_BUILD_PROFILE` (генерация `BuildProfile.ets`), а **не** `CompileArkTS` модуля HAR.

**Следовательно, `CompileArkTS` HAP не зависит от `CompileArkTS` HAR — задачи `CompileArkTS` всех модулей могут стать готовыми одновременно.**

### 2.3 Детальное проектирование компонентов

#### 2.3.1 TsgoBatchCompileCoordinator (ключевой координатор)

```
 // Расположение: hvigor-ohos-plugin/src/tasks/worker/tsgo-batch-coordinator.ts

 export class TsgoBatchCompileCoordinator {
  private static _instance: TsgoBatchCompileCoordinator;

  // Ожидаемое число регистраций (из TaskScheduleOptimizationService или обхода DAG)
  private _expectedCount: number = 0;
  // Конфигурации зарегистрированных модулей
  private _configs: ProjectConfig[] = [];
  // Имена зарегистрированных модулей (для раздачи результатов)
  private _moduleNames: string[] = [];
  // Общий Promise пакетной компиляции
  private _batchPromise: Promise<Map<string, ModuleCompileResult>> | null = null;
  private _batchResolve!: (result: Map<string, ModuleCompileResult>) => void;
  private _batchReject!: (error: Error) => void;
  // Отправлено ли уже (защита от повторной отправки)
  private _submitted: boolean = false;

  static getInstance(): TsgoBatchCompileCoordinator { ... }

  /** Инициализация: подсчёт числа задач CompileArkTS из DAG */
  init(expectedCount: number): void {
    this._expectedCount = expectedCount;
    this._configs = [];
    this._moduleNames = [];
    this._submitted = false;
    this._batchPromise = new Promise((resolve, reject) => {
      this._batchResolve = resolve;
      this._batchReject = reject;
    });
  }

  /** Регистрация конфигурации модуля, возвращает Promise результата компиляции модуля */
  registerConfig(config: ProjectConfig): Promise<ModuleCompileResult> {
    this._configs.push(config);
    this._moduleNames.push(config.moduleName);

    // Последний зарегистрировавшийся триггерит пакетную отправку
    if (this._configs.length === this._expectedCount && !this._submitted) {
      this._submitted = true;
      this.submitBatchWork();
    }

    // Возвращаем Promise результата этого модуля
    return this._batchPromise.then(batchResult => batchResult.get(config.moduleName)!);
  }

  /** Отправка пакетной компиляции в пул воркеров */
  private async submitBatchWork(): Promise<void> {
    const workerPool = WorkerPoolFactory.getDefaultWorkerPool();
    const workInput = {
      configs: this._configs,
      // Глобальная информация (общая для всех модулей)
      projectArkOption: this._configs[0]?.projectArkOption,
      sdkPath: this._configs[0]?.sdkPath,
      // ...прочая глобальная конфигурация
    };

    // Отправляем в фиксированный воркер (закрепляем workerId=0, чтобы избежать затрат на прогрев TSGO)
    workerPool.submit(null, './tsgo-job.js/run', {
      workInput,
      callback: (result: BatchCompileResult | Error) => {
        if (result instanceof Error) {
          this._batchReject(result);
        } else {
          this._batchResolve(result.moduleResults);
        }
      },
      targetWorkers: [0],  // фиксированный воркер
      useReturnVal: true,
      memorySensitive: true,
      preludeDeps: ['@ohos/hvigor', '@ohos/hvigor-arkts-compose'],
    });
  }

  getBatchPromise(): Promise<Map<string, ModuleCompileResult>> {
    return this._batchPromise!;
  }

  reset(): void { /* очистка состояния для следующей сборки */ }
 }
```

#### 2.3.2 Переделка задачи CompileArkTS (управление переключателем)

```
 // abstract-ark-compile.ts -- переделка doTaskAction

 protected async doTaskAction(): Promise<void> {
  this.validateModuleJsonAbstract();
  const config: ProjectConfig = await this.initDefaultArkCompileConfig();

  // ===== Ветвь переключателя TSGO =====
  if (coreParameter.properties.ohosTsgoCompileEnabled) {
    await this.doTsgoBatchCompile(config);
    return;  // исходный путь не выполняем
  }

  // ===== Исходный путь (без изменений) =====
  // ... исходная логика submitArkCompileWork ...
 }

 /** Путь пакетной компиляции TSGO */
 private async doTsgoBatchCompile(config: ProjectConfig): Promise<void> {
  const coordinator = TsgoBatchCompileCoordinator.getInstance();

  // 1. Регистрируем конфигурацию, получаем Promise результата этого модуля (без await!)
  const myResultPromise = coordinator.registerConfig(config);

  // 2. Регистрируем обработку результата как pending promise (тот же механизм, что у workerPool.submit)
  //    await myResultPromise нельзя —— иначе заблокируется рекурсивное планирование task-runner, дедлок
  const processingPromise = myResultPromise.then(result => {
    this.handleCompileEvents(result.compileEvents, this.durationEvent, false);
    return this.writeBuildMode(config);
  });
  this.pendingPromises.add(processingPromise);

  // 3. doTaskAction сразу возвращается, task-runner продолжает планировать следующую задачу CompileArkTS
  //    task-runner видит hasPendingPromises=true → handleTaskPendingPromises
  //    → Promise.all(pendingPromises).then(onTaskFinished) → разблокирует downstream

  // 4. Компиляция Widget (можно пакетно, можно независимо)
  if (this.needSubmitArkTsWidget) {
    // конфигурация widget тоже регистрируется в coordinator
  }
 }
```

#### 2.3.3 Входной скрипт воркера TSGO

```
 // Расположение: hvigor-ohos-plugin/src/tasks/worker/tsgo-job.ts

 export const run = async (
  input: { configs: ProjectConfig[] },
  cacheStoreManager?: CacheStoreManager
 ): Promise<BatchCompileResult | Error> => {
  const { configs } = input;

  try {
    // Строим параметры единой компиляции TSGO
    const tsgoOptions: TsgoCompileOptions = {
      modules: configs.map(config => ({
        moduleName: config.moduleName,
        modulePath: config.modulePath,
        sourceRoot: config.projectPath,
        entryObj: getCompileInputObject(config),
        outputDir: config.aceModuleBuild,
        buildMode: config.buildMode,
        arkOptions: config.arkOptions,
        tscConfig: config.arkOptions?.tscConfig,
        externalApiPaths: config.externalApiPaths,
        entryModules: config.entryModules,
        // ... прочие per-module конфигурации
      })),
      // Глобальная конфигурация
      sdkPath: configs[0].sdkPath,
      compileProjectModel: GlobalProjectDataService.getInstance().getCompileProjectModel(),
      cachePath: configs[0].cachePath,
      // ... прочая глобальная конфигурация
    };

    // Вызываем единую компиляцию TSGO
    const tsgoResult = await runTsgoCompile(tsgoOptions);

    // Преобразуем в per-module Map результатов
    const moduleResults = new Map<string, ModuleCompileResult>();
    for (const moduleResult of tsgoResult.moduleResults) {
      moduleResults.set(moduleResult.moduleName, moduleResult);
    }

    return { moduleResults, isSuccess: true };
  } catch (error) {
    return error as Error;
  }
 };
```

#### 2.3.4 Контракт адаптационного интерфейса на стороне TSGO

```
 // Контракт данных hvigor -> TSGO

 interface TsgoCompileOptions {
  /** Конфигурации всех модулей к компиляции */
  modules: TsgoModuleConfig[];
  /** Путь SDK */
  sdkPath: string;
  /** Глобальная таблица модулей (sourceRoot -> ModuleInfo) */
  compileProjectModel: ArkCompileProjectModelType;
  /** Путь кэша */
  cachePath: string;
  /** Режим сборки */
  buildMode: string;
 }

 interface TsgoModuleConfig {
  moduleName: string;
  modulePath: string;
  sourceRoot: string;
  /** Вход компиляции (формат rollup input или собственный формат TSGO) */
  entryObj: Record<string, string>;
  /** Каталог вывода */
  outputDir: string;
  /** Опции компиляции ArkTS (соответствует BuildOpt.arkOptions) */
  arkOptions: ArkOptions;
  /** Конфигурация TSC */
  tscConfig?: TscConfig;
  /** Пути внешних API (.d.ts зависимых HAR) */
  externalApiPaths: string[];
  /** Список входных модулей */
  entryModules: string[];
  /** Опции обфускации */
  obfuscationOptions?: ObfuscationOptions;
  /** Информация о путях модуля */
  aceModuleRoot: string;
  aceModuleBuild: string;
  /** Путь карты исходников */
  sourceMapPath?: string;
  /** Компиляция ли widget */
  widgetCompile?: string;
 }

 // Контракт возврата TSGO -> hvigor

 interface BatchCompileResult {
  moduleResults: Map<string, ModuleCompileResult>;
  isSuccess: boolean;
  error?: Error;
 }

 interface ModuleCompileResult {
  moduleName: string;
  compileEvents: CompileEvent[];
  compileLogEvents: LogEvent[];
  isSuccess: boolean;
  error?: Error;
 }
```

---

## III. Момент инициализации и подсчёт модулей

```
 Жизненный цикл сборки:

  init()                          -- создание моделей Project/Module
  configuration()                 -- оценка hvigorfile, привязка плагинов, регистрация задач
    +- хук nodesEvaluated         -- конфигурации всех модулей готовы
  buildTaskGraph()                -- построение DAG
    +- хук taskGraphResolved      -- DAG готов, можно посчитать число задач CompileArkTS
      +- [точка инициализации TSGO]
          TsgoBatchCompileCoordinator.getInstance().init(
            countCompileArkTsTasks(projectTaskDag)
          )
  execute()                       -- выполнение задач
    +- CompileArkTS(A/B/C...)     -- регистрация конфигурации + ожидание пакетного результата
```

**Реализация подсчёта модулей**:

```
 // Регистрируем в хуке taskGraphResolved
 function initTsgoCoordinator(): void {
  if (!coreParameter.properties.ohosTsgoCompileEnabled) return;

  const compileArkTsTasks = hvigorCore.getCompileDependArr();
  let count = 0;
  for (const taskPath of compileArkTsTasks) {
    if (taskPath.includes('CompileArkTS')) {
      count++;
    }
  }
  TsgoBatchCompileCoordinator.getInstance().init(count);
 }
```

---

## IV. Адаптация инкрементальной компиляции

```
 Обработка инкрементальных сценариев:

 1. Инкрементальный пропуск на уровне модулей (сохраняется):
    - declareInputs/declareOutputs у IncrementalTask не меняются
    - модули UP-TO-DATE не выполняют doTaskAction -> не регистрируют конфигурацию
    - нужно скорректировать expectedCount: считать только фактически выполняемые задачи CompileArkTS

 2. Внутренняя инкрементальность TSGO (адаптация на стороне TSGO):
    - TSGO получает полный список модулей, но внутри может пропускать неизменённые модули по hash файлов
    - LanguageService TSGO изначально поддерживает инкрементальную компиляцию (переиспользование кэша между модулями)

 3. Динамическая корректировка expectedCount:
    - после taskGraphResolved, до execute, обойти флаги isUpToDate задач CompileArkTS в DAG
    - считать expectedCount только по задачам !isUpToDate
```

---

## V. Стратегия обработки ошибок

```
 Механизм распространения ошибок:

  Ошибка компиляции TSGO
    |
    +- Ошибка одного модуля:
    |  coordinator._batchReject(error)
    |  -> все задачи CompileArkTS, ожидающие batchPromise, получают reject
    |  -> срабатывает onTaskFailed в task-runner
    |  -> сборка прерывается
    |
    +- Глобальная ошибка (например, падение процесса TSGO):
       событие WORK_ERROR в worker-action
       -> callback TCB вызывает reject
       -> распространение как выше
```

**Оптимизация**: TSGO должен по возможности возвращать per-module ошибки, а не общий отказ. Даже если компиляция модуля A упала, результаты модуля B всё ещё пригодны (зависит от бизнес-стратегии).

---

## VI. Адаптация Spinner / логов

```
 Поведение Spinner (режим TSGO):

  Вариант A (рекомендуется): единый Spinner
    - первый CompileArkTS запускает spinner "TSGO Batch Compiling..."
    - внутренние callback'и прогресса TSGO обновляют состояние spinner
    - после завершения пакета — stop spinner

  Вариант B: сохранить per-module spinner
    - каждый CompileArkTS запускает свой spinner
    - но в период ожидания batchPromise spinner в состоянии "waiting"
    - после завершения пакета останавливаются по очереди
```

---

## VII. Область влияния изменений и совместимость

### 7.1 Список изменений

|**Файл**|**Тип изменения**|**Пояснение**|
|---|---|---|
|`abstract-ark-compile.ts:443`|**изменение** doTaskAction|**новая ветвь TSGO, управление переключателем**|
|`tsgo-batch-coordinator.ts`|**новый**|**синглтон ключевого координатора**|
|`tsgo-job.ts`|**новый**|**входной скрипт воркера**|
|`run-ark.ts`|**без изменений**|**сохраняем исходный submitArkCompileWork**|
|`job.ts`|**без изменений**|**сохраняем исходную функцию run**|
|`worker-id-helper.ts`|**без изменений**|**в режиме TSGO не используется, но не удаляется**|
|`global-core-parameters.ts:124`|**изменение**|**добавляем свойство** `ohosTsgoCompileEnabled`|
|`task-schedule-optimization.ts`|**без изменений**|**переиспользуем существующий compileDependArr**|
|`arkts-pack.ts`|**без изменений**|**сохраняем исходный runArkPack**|
|**DAG / TaskControlCenter**|**без изменений**|**изменений структуры DAG нет**|

### 7.2 Гарантии совместимости (по спецификации AGENTS.md)

1. **Существующую логику не удаляем**: `submitArkCompileWork`, `runArkPack`, `worker-id-helper` полностью сохраняются
2. **Поведение по умолчанию не меняем**: `ohosTsgoCompileEnabled` по умолчанию `false`, работает исходный путь
3. **Управление переключателем**: через `coreParameter.properties.ohosTsgoCompileEnabled`
4. **Старые интерфейсы сохраняем**: исходный путь `AbstractArkCompile.doTaskAction` полностью сохраняется

### 7.3 Что нужно адаптировать на стороне TSGO

|**Пункт адаптации**|**Пояснение**|
|---|---|
|**Приём пакетных конфигураций модулей**|**принимать** `TsgoModuleConfig[]` вместо одного `ProjectConfig`|
|**Единый LanguageService**|**внутри одного процесса TSGO LanguageService общий для всех модулей (переиспользование разбора типов между модулями)**|
|**per-module вывод артефактов**|**выводить ABC-байткод и sourceMap каждого модуля в его** `aceModuleBuild`|
|**per-module возврат результатов**|**возвращать** `Map<moduleName, ModuleCompileResult>`|
|**Callback прогресса**|**опционально: сообщать прогресс компиляции через** `parentPort.postMessage`|
|**Адаптация кэша**|**принимать** `cacheStoreManager` от hvigor, переиспользовать персистентный кэш daemon-режима|

---

## VIII. Оценка прироста производительности

```
 Сценарий: проект из 10 модулей (5 HAR + 1 HAP + 1 HSP + 3 библиотеки зависимостей)

 Текущий режим (N независимых компиляций):
  +- холодный старт воркера x 6 (сначала HAR, затем HAP/HSP)
  +- инициализация tsc LanguageService x 6
  +- разбор SDK .d.ts x 6 (каждый воркер независимо разбирает одни и те же декларации SDK)
  +- инициализация цепочки rollup-плагинов x 6
  +- фактический параллелизм: ~2-3 (ограничен зависимостями DAG)

 Режим TSGO (1 единая компиляция):
  +- холодный старт воркера x 1
  +- инициализация TSGO LanguageService x 1 (общий для модулей)
  +- разбор SDK .d.ts x 1 (глобальный кэш)
  +- внутренний параллелизм TSGO (рантайм Go от природы многоядерный)
  +- нет ожиданий зависимостей DAG (все модули компилируются единообразно)

 Ожидаемый прирост:
  +- первая сборка: ускорение 40-60% (в основном за счёт инициализации tsc и дедупликации разбора типов)
  +- инкрементальная сборка: ускорение 20-30% (инкрементальная компиляция TSGO + кэш между модулями)
  +- режим daemon: ускорение 30-50% (резидентный процесс TSGO + переиспользование LanguageService)
```

---

## IX. Рекомендуемый путь внедрения

```
 Phase 1: инфраструктура
  +- реализовать TsgoBatchCompileCoordinator
  +- реализовать вход воркера tsgo-job.ts
  +- добавить переключатель ohosTsgoCompileEnabled
  +- переделать AbstractArkCompile.doTaskAction, добавить ветвь TSGO

 Phase 2: адаптация TSGO
  +- определить интерфейсы TsgoCompileOptions / TsgoModuleConfig
  +- на стороне TSGO реализовать чтение пакета модулей и единую компиляцию
  +- на стороне TSGO реализовать per-module вывод артефактов
  +- на стороне TSGO реализовать возврат Map результатов

 Phase 3: интеграционная проверка
  +- адаптация инкрементальной компиляции (динамическая корректировка expectedCount)
  +- адаптация компиляции Widget
  +- адаптация Spinner / логов
  +- проверка обработки ошибок
  +- end-to-end тесты

 Phase 4: оптимизация производительности
  +- резидентный TSGO-воркер + переиспользование daemon
  +- настройка внутреннего параллелизма TSGO
  +- оптимизация стратегии кэширования
  +- бенчмарки сравнения производительности
```

---

## X. Ссылки на ключевые файлы

|**Назначение**|**Путь файла**|**Строка**|
|---|---|---|
|**Точка входа выполнения задачи CompileArkTS**|`hvigor-ohos-plugin/src/tasks/abstract-ark-compile.ts`|**443**|
|**Отправка work одного модуля**|`hvigor-ohos-plugin/src/tasks/worker/run-ark.ts`|**35**|
|**Выполнение компиляции внутри воркера**|`hvigor-ohos-plugin/src/tasks/worker/job.ts`|**28**|
|**Точка входа runArkPack**|`hvigor-arkts-compose/src/arkts-pack.ts`|**207**|
|**Построение цепочки rollup-плагинов**|`hvigor-arkts-compose/src/arkts-pack.ts`|**470**|
|**Регистрация задач компиляции**|`hvigor-ohos-plugin/src/tasks/manager/arkts-task-initializer.ts`|**33**|
|**Сбор цепочки зависимостей компиляции**|`hvigor-ohos-plugin/src/plugin/hooks/task-schedule-optimization.ts`|**44**|
|**Построение глобальной таблицы модулей**|`hvigor-ohos-plugin/src/tasks/service/global-project-data-service.ts`|**207**|
|**Управление слиянием buildOption**|`hvigor-ohos-plugin/src/project/build-option/build-mode-manager.ts`|**58**|
|**Определение ProjectConfig**|`hvigor-arkts-compose/src/project-config/project-config.ts`|**24**|
|**Аффинность модуль->воркер**|`hvigor-ohos-plugin/src/tasks/worker/worker-id-helper.ts`|**6**|
|**Фабрика пула воркеров**|`hvigor/src/base/internal/pool/worker-pool/worker-pool-factory.ts`|**26**|
|**Глобальные параметры**|`hvigor/src/base/internal/data/global-core-parameters.ts`|**124**|
|**Последовательное планирование задач**|`hvigor/src/base/internal/task/core/task-runner.ts`|**30**|
|**Механизм pendingPromises**|`hvigor/src/base/internal/task/core/task-runner.ts`|**91**|

---

**Версия документа**: 1.0 **Последнее обновление**: 2026-08-13 **Команда-владелец**: команда разработки Hvigor
