
<h1>ArkUI Declarative Core Language Specification </h1>

ArkUI mechanisms for application state, UI rendering and update.

Editor: Guido Grassel (guido.grassel@huawei.com), Huawei RnD Finland
Last Major Change: June 28, 2023.

This is a 'living' specification. We aim to update this Wiki document whenever 
adding new functionality, fixing bugs in the spec or framework implementation,
and we frequently add clarifications and new/better examples.  If you find a 
bug or something seems unclear, please file an issue.

The English version of this document at https://gitee.com/arkui-finland/arkui-edsl-core-spec/blob/master/arkui-core-spec.md is the reference. It should be preferred
over any translation because of risk of missing update, translation mistake or omission.

# Specification Quick Links

* [@Component](general-ui-spec.md#222-provisions-for-defining-a-custom-component)
* [@Entry](general-ui-spec.md#223-default-page-entry-component-entry)
* [build()](general-ui-spec.md#23-domain-specific-syntax-of-the-component-build-function)
* [@Builder](general-ui-spec.md#243-provisions-for-custom-builder-functions)
* [@BuilderParam](general-ui-spec.md#247-provisions-for-using-builderparam)
* [aboutToAppear, aboutToDisappear](general-ui-spec.md#226-lifecycle-callback-functions)
* [@Extend](general-ui-spec.md#25-extending-built-in-components-with-extend)
* [if else](rendering-control-syntax.md#713-provisions-for-using-if-else-and-else-if)
* [ForEach](rendering-control-syntax.md#724-provisions-for-using-foreach)
* [Local variable init](intro-state-mgmt.md#321-provisions-for-local-variable-initialization)
* [Variable init from the parent](intro-state-mgmt.md#322-provisions-for-variable-initialization-from-the-parent)
* [Variable update from the parent](intro-state-mgmt.md#323-provisions-for-variable-update-from-the-parent)
* [@State](manage-state-component.md#411-provisions-for-using-state)
* [@Prop](manage-state-component.md#426-provisions-for-using-prop)
* [@Link](manage-state-component.md#431-provisions-for-using-link)
* [@Provide, @Consume](manage-state-component.md#441-provisions-for-using-provide-and-consume)
* [@ObjectLink, @Observed](manage-state-component.md#458-provisions-for-using-observed-and-objectlink)
* [@LocalStorageLink](ui-state-storages.md#513-provisions-for-using-localstorage-api)
* [@LocalStorageProp](ui-state-storages.md#513-provisions-for-using-localstorage-api)
* [@StorageProp](ui-state-storages.md#523-provisions-for-using-appstorage-api)
* [@StorageLink](ui-state-storages.md#523-provisions-for-using-appstorage-api)
* [@Watch](other-state-mgmt.md#611--provisions-for-using-watch)
* [$$](other-state-mgmt.md#631-provisions-for-using-)
* [SubscribableAbstract](other-state-mgmt.md#622-provisions-for-using-subscribableabstract)
* [LocalStorage](ui-state-storages.md#513-provisions-for-using-localstorage-api)
* [AppStorage](ui-state-storages.md#523-provisions-for-using-appstorage-api)
* [PersistentStorage](ui-state-storages.md#543-provisions-for-using-persistentstorage)
* [Environment](ui-state-storages.md#551-provisions-for-using-environment)

# Full Table of Contents

  - [Specification Quick Links](arkui-core-spec.md#specification-quick-links)
  - [Full Table of Contents](arkui-core-spec.md#full-table-of-contents)
  - [Revision History](arkui-core-spec.md#revision-history)
  - [1 Summary](arkui-core-spec.md#1-summary)
  - [2 General UI Specifications](general-ui-spec.md#2-general-ui-specifications)
    - [2.1 Declarative UI Description](general-ui-spec.md#21-declarative-ui-description)
      - [2.1.1 Creating components](general-ui-spec.md#211-creating-components)
      - [2.1.2 Attribute functions](general-ui-spec.md#212-attribute-functions)
      - [2.1.3 Handling events](general-ui-spec.md#213-handling-events)
    - [2.2 Subdivision into custom components](general-ui-spec.md#22-subdivision-into-custom-components)
      - [2.2.1 Introduction](general-ui-spec.md#221-introduction)
      - [2.2.2 Provisions for defining a custom component](general-ui-spec.md#222-provisions-for-defining-a-custom-component)
      - [2.2.3 Default page entry component `@Entry`](general-ui-spec.md#223-default-page-entry-component-entry)
      - [2.2.4 Provisions for custom component parameters](general-ui-spec.md#224-provisions-for-custom-component-parameters)
      - [2.2.5 Custom Component Lifecycle](general-ui-spec.md#225-custom-component-lifecycle)
      - [2.2.6 Lifecycle callback functions](general-ui-spec.md#226-lifecycle-callback-functions)
    - [2.3 Domain specific syntax of the `@Component build` and `@Builder` functions](general-ui-spec.md#23-domain-specific-syntax-of-the-component-build-and-builder-functions)
    - [2.4 Custom build functions `@Builder`](general-ui-spec.md#24-custom-build-functions-builder)
      - [2.4.1 Example for custom build functions with by-reference parameter passing](general-ui-spec.md#241-example-for-custom-build-functions-with-by-reference-parameter-passing)
      - [2.4.2 Explanations on how ArkUI supports by-reference parameter passing](general-ui-spec.md#242-explanations-on-how-arkui-supports-by-reference-parameter-passing)
      - [2.4.3 Provisions for custom builder functions](general-ui-spec.md#243-provisions-for-custom-builder-functions)
      - [2.4.4 Provisions for custom builder functions by-reference parameter passing](general-ui-spec.md#244-provisions-for-custom-builder-functions-by-reference-parameter-passing)
      - [2.4.5 Provisions for custom builder functions by-value parameter passing](general-ui-spec.md#245-provisions-for-custom-builder-functions-by-value-parameter-passing)
      - [2.4.6 Custom component variable holding a @Builder function reference  `@BuilderParam` and `BuilderType<C>`](general-ui-spec.md#246-custom-component-variable-holding-a-builder-function-reference--builderparam-and-buildertypec)
      - [2.4.7 `@Builder` function as parameter to another `@Builder` function](general-ui-spec.md#247-builder-function-as-parameter-to-another-builder-function)
      - [2.4.8 Provisions for using `@BuilderParam`](general-ui-spec.md#248-provisions-for-using-builderparam)
    - [2.5 Extending built-in components with `@Extend`](general-ui-spec.md#25-extending-built-in-components-with-extend)
    - [2.6 Definition of resusable styles with @Styles](general-ui-spec.md#26-definition-of-resusable-styles-with-styles)
  - [3. Introduction to State Management](intro-state-mgmt.md#3-introduction-to-state-management)
    - [3.1 ArkUI decorators overview](intro-state-mgmt.md#31-arkui-decorators-overview)
    - [3.2 General provisions for decorated variables](intro-state-mgmt.md#32-general-provisions-for-decorated-variables)
      - [3.2.1 General provisions for variable type](intro-state-mgmt.md#321-general-provisions-for-variable-type)
      - [3.2.2 TypeScript union types and `type` keyword](intro-state-mgmt.md#322-typescript-union-types-and-type-keyword)
      - [3.2.3 Provisions for `undefined` and `null`](intro-state-mgmt.md#323-provisions-for-undefined-and-null)
      - [3.2.4 Permissive type checking for API 10 vs. strict type checking in subsequent API versions](intro-state-mgmt.md#324-permissive-type-checking-for-api-10-vs-strict-type-checking-in-subsequent-api-versions)
      - [3.2.5 Variable types and observed changes for variables decorated with  `@State`, `@Provide`, `@Link`, `@Consume`, or `@Prop`](intro-state-mgmt.md#325-variable-types-and-observed-changes-for-variables-decorated-with--state-provide-link-consume-or-prop)
      - [3.2.6 Variable types and observed changes for variables decorated with  `@ObjectLink`:](intro-state-mgmt.md#326-variable-types-and-observed-changes-for-variables-decorated-with--objectlink)
        - [3.2.7 Variable types and observed changes for variables decorated with `@StorageLink`, `@StorageProp`, `@LocalStorageLink`, or `@LocalStorageProp`](intro-state-mgmt.md#327-variable-types-and-observed-changes-for-variables-decorated-with-storagelink-storageprop-localstoragelink-or-localstorageprop)
      - [3.2.8 Provisions for local variable initialization](intro-state-mgmt.md#328-provisions-for-local-variable-initialization)
      - [3.2.9 Provisions for variable initialization from the parent](intro-state-mgmt.md#329-provisions-for-variable-initialization-from-the-parent)
      - [3.2.10 Provisions for variable update from the parent](intro-state-mgmt.md#3210-provisions-for-variable-update-from-the-parent)
    - [3.3 Tutorial - application state in ArkUI](intro-state-mgmt.md#33-tutorial---application-state-in-arkui)
      - [3.3.1 Model - View - View Model Pattern](intro-state-mgmt.md#331-model---view---view-model-pattern)
      - [3.3.2 Who should own a ViewModel root data item - `@State`, `@Provide`, `@LocalStorageLink/Prop`, or `@StorageLink/Prop`?](intro-state-mgmt.md#332-who-should-own-a-viewmodel-root-data-item---state-provide-localstoragelinkprop-or-storagelinkprop)
      - [3.3.3 `@Prop` and `@ObjectLink` use for nested data structures](intro-state-mgmt.md#333-prop-and-objectlink-use-for-nested-data-structures)
      - [3.3.4 The difference between `@Prop` and `@ObjectLink` used for nested data structures](intro-state-mgmt.md#334-the-difference-between-prop-and-objectlink-used-for-nested-data-structures)
      - [3.3.5 Application example for nested ViewModel](intro-state-mgmt.md#335-application-example-for-nested-viewmodel)
      - [3.3.6 Example for the difference between `@Prop` shallow copy and deep copy](intro-state-mgmt.md#336-example-for-the-difference-between-prop-shallow-copy-and-deep-copy)
    - [3.4 Good and Bad Practices in State Management](intro-state-mgmt.md#34-good-and-bad-practices-in-state-management)
      - [3.4.1 The basics](intro-state-mgmt.md#341-the-basics)
      - [3.4.2 Miss Nested Object Property Changes Case 1](intro-state-mgmt.md#342-miss-nested-object-property-changes-case-1)
      - [3.4.3 Missed Nested Object Property Updates Case 2](intro-state-mgmt.md#343-missed-nested-object-property-updates-case-2)
      - [3.4.4 `@Prop` object copy semantics](intro-state-mgmt.md#344-prop-object-copy-semantics)
      - [3.4.5 Application must not mutate application state during `build`](intro-state-mgmt.md#345-application-must-not-mutate-application-state-during-build)
      - [3.4.6 UIUpdater - Force UI update with artificial app state](intro-state-mgmt.md#346-uiupdater---force-ui-update-with-artificial-app-state)
      - [3.4.7 Duplicate Array ids caused by adding Array element in ForEach - case 1](intro-state-mgmt.md#347-duplicate-array-ids-caused-by-adding-array-element-in-foreach---case-1)
      - [3.4.8 Duplicate array element update in ForEach - case 2](intro-state-mgmt.md#348-duplicate-array-element-update-in-foreach---case-2)
      - [3.4.9 Array of Objects with `@StorageLink` / `@LocalStorageLink`](intro-state-mgmt.md#349-array-of-objects-with-storagelink--localstoragelink)
  - [4. Managing State owned by a Component](manage-state-component.md#4-managing-state-owned-by-a-component)
    - [4.1 State owned by a Component `@State`](manage-state-component.md#41-state-owned-by-a-component-state)
      - [4.1.1 Provisions for using `@State`](manage-state-component.md#411-provisions-for-using-state)
    - [4.2 Component state uni-directionally synced from parent component `@Prop`](manage-state-component.md#42-component-state-uni-directionally-synced-from-parent-component-prop)
      - [4.2.1 Scenario - Simple type `@Prop` synced from `@State` in parent component](manage-state-component.md#421-scenario---simple-type-prop-synced-from-state-in-parent-component)
      - [4.2.2 Scenario - Simple type `@Prop` with local init and no sync from parent](manage-state-component.md#422-scenario---simple-type-prop-with-local-init-and-no-sync-from-parent)
      - [4.2.3 Scenario - Simple type `@Prop` synch'd from `@State` Array item in parent component](manage-state-component.md#423-scenario---simple-type-prop-synchd-from-state-array-item-in-parent-component)
      - [4.2.4 Scenario - Class object type `@Prop` synch'd from `@State` class object property in parent component](manage-state-component.md#424-scenario---class-object-type-prop-synchd-from-state-class-object-property-in-parent-component)
      - [4.2.5 Scenario - Class type `@Prop` synced from `@State` array item in parent component](manage-state-component.md#425-scenario---class-type-prop-synced-from-state-array-item-in-parent-component)
      - [4.2.6 Provisions for using `@Prop`](manage-state-component.md#426-provisions-for-using-prop)
    - [4.3 Bi-bidirectional syncing with a variable of parent component `@Link`](manage-state-component.md#43-bi-bidirectional-syncing-with-a-variable-of-parent-component-link)
      - [4.3.1 Provisions for using `@Link`](manage-state-component.md#431-provisions-for-using-link)
      - [4.3.2 Example for `@Link` with simple and with class types](manage-state-component.md#432-example-for-link-with-simple-and-with-class-types)
      - [4.3.3 Example for `@Link` with Array type](manage-state-component.md#433-example-for-link-with-array-type)
    - [4.4 Bi-directionally syncing state with descendent components `@Provide` and `@Consume`](manage-state-component.md#44-bi-directionally-syncing-state-with-descendent-components-provide-and-consume)
      - [4.4.1 Provisions for using `@Provide` and `@Consume`](manage-state-component.md#441-provisions-for-using-provide-and-consume)
    - [4.5 Object reference with `@ObjectLink`](manage-state-component.md#45-object-reference-with-objectlink)
      - [4.5.1 Introduction to usage scenarios for `@ObjectLink`](manage-state-component.md#451-introduction-to-usage-scenarios-for-objectlink)
      - [4.5.2 `@ObjectLink` variable and its source variable of parent component both refer to the same object](manage-state-component.md#452-objectlink-variable-and-its-source-variable-of-parent-component-both-refer-to-the-same-object)
    - [4.5.3 Observing Nested Class Object Property Changes `@Observed` and `@ObjectLink` - Introduction](manage-state-component.md#453-observing-nested-class-object-property-changes-observed-and-objectlink---introduction)
      - [4.5.4 `@ObjectLink` and `@Observed` Class Decorator Scenario Nested Objects](manage-state-component.md#454-objectlink-and-observed-class-decorator-scenario-nested-objects)
      - [4.5.5 `@ObjectLink` and `@Observed` class decorator Scenario Array of Objects](manage-state-component.md#455-objectlink-and-observed-class-decorator-scenario-array-of-objects)
      - [4.5.6 `@ObjectLink` and `@Observed` class decorator Scenario Array of Arrays](manage-state-component.md#456-objectlink-and-observed-class-decorator-scenario-array-of-arrays)
      - [4.5.7 `@ObjectLink` and `@Observed` class decorator Scenario arbitrary deep nesting](manage-state-component.md#457-objectlink-and-observed-class-decorator-scenario-arbitrary-deep-nesting)
      - [4.5.8 Provisions for using `@Observed` and `@ObjectLink`](manage-state-component.md#458-provisions-for-using-observed-and-objectlink)
      - [4.5.9 Avoid pitfall of union type sync source of `@ObjectLink`](manage-state-component.md#459-avoid-pitfall-of-union-type-sync-source-of-objectlink)
    - [4.6 Regular TypeScript component variable](manage-state-component.md#46-regular-typescript-component-variable)
    - [4.7 More examples](manage-state-component.md#47-more-examples)
      - [4.7.1 Decorated @Component variables of type `Map`](manage-state-component.md#471-decorated-component-variables-of-type-map)
      - [4.7.2 Decorated @Component variables of type `Set`](manage-state-component.md#472-decorated-component-variables-of-type-set)
  - [5 UI State Storages](ui-state-storages.md#5-ui-state-storages)
    - [5.1 Storage API for UI state `LocalStorage`](ui-state-storages.md#51-storage-api-for-ui-state-localstorage)
      - [5.1.1 Example for using `LocalStorage` from app logic](ui-state-storages.md#511-example-for-using-localstorage-from-app-logic)
      - [5.1.2 Example for using `LocalStorage` from inside UI](ui-state-storages.md#512-example-for-using-localstorage-from-inside-ui)
      - [5.1.3 Provisions for using `LocalStorage` API](ui-state-storages.md#513-provisions-for-using-localstorage-api)
      - [5.1.4  Sharing a LocalStorage instance from Ability to one or several Views](ui-state-storages.md#514--sharing-a-localstorage-instance-from-ability-to-one-or-several-views)
    - [5.2 Storage API for application-wide UI state `AppStorage`](ui-state-storages.md#52-storage-api-for-application-wide-ui-state-appstorage)
      - [5.2.1 Example for using `AppStorage` and `LocalStorage` from app logic](ui-state-storages.md#521-example-for-using-appstorage-and-localstorage-from-app-logic)
      - [5.2.2  Example for using `AppStorage` and `LocalStorage` from inside UI](ui-state-storages.md#522--example-for-using-appstorage-and-localstorage-from-inside-ui)
      - [5.2.3 Provisions for using `AppStorage` API](ui-state-storages.md#523-provisions-for-using-appstorage-api)
    - [5.3 Supporting APIs to `AppStorage` and `LocalStorage`](ui-state-storages.md#53-supporting-apis-to-appstorage-and-localstorage)
      - [5.3.1 `SubscribedAbstractProperty<T>`](ui-state-storages.md#531-subscribedabstractpropertyt)
      - [5.3.2 Add a `LocalStorage` instance to a `@Component` using `@Entry`](ui-state-storages.md#532-add-a-localstorage-instance-to-a-component-using-entry)
    - [5.4 API for persisting application properties `PersistentStorage`](ui-state-storages.md#54-api-for-persisting-application-properties-persistentstorage)
      - [5.4.1 Example](ui-state-storages.md#541-example)
      - [5.4.2 Wrong use - Access property in `AppStorage` before `PersistentStorage`](ui-state-storages.md#542-wrong-use---access-property-in-appstorage-before-persistentstorage)
      - [5.4.3 Provisions for using `PersistentStorage`](ui-state-storages.md#543-provisions-for-using-persistentstorage)
    - [5.5 API for importing device environment `Environment`](ui-state-storages.md#55-api-for-importing-device-environment-environment)
      - [5.5.1 Provisions for using `Environment`](ui-state-storages.md#551-provisions-for-using-environment)
    - [5.6 Overview: Synchronization between `AppStorage` / `LocalStorage` and UI Components](ui-state-storages.md#56-overview-synchronization-between-appstorage--localstorage-and-ui-components)
    - [5.7 Synchronization between `AppStorage` and UI Components using `@StorageLink` and `@StorageProp` decorators](ui-state-storages.md#57-synchronization-between-appstorage-and-ui-components-using-storagelink-and-storageprop-decorators)
      - [5.7.1 Example of use](ui-state-storages.md#571-example-of-use)
      - [5.7.2 Provisions for using `@StorageLink` and `@StorageProp`](ui-state-storages.md#572-provisions-for-using-storagelink-and-storageprop)
    - [5.8 Synchronization between `LocalStorage` and UI Components using  `@LocalStorageLink`and `@LocalStorageProp` decorators](ui-state-storages.md#58-synchronization-between-localstorage-and-ui-components-using--localstoragelinkand-localstorageprop-decorators)
      - [5.8.1 First example of use](ui-state-storages.md#581-first-example-of-use)
      - [5.8.2 Provisions for using `@LocalStorageLink` and `@LocalStorageProp`](ui-state-storages.md#582-provisions-for-using-localstoragelink-and-localstorageprop)
      - [5.8.3 Second example of use](ui-state-storages.md#583-second-example-of-use)
    - [5.9 `LocalStorage` and `AppStorage` access combined](ui-state-storages.md#59-localstorage-and-appstorage-access-combined)
    - [5.10 Example combining `@StorageLink`, `@LocalStorageLink`, `@Provide` - `@Consume` and `@State`](ui-state-storages.md#510-example-combining-storagelink-localstoragelink-provide---consume-and-state)
  - [6 Other State Management functionality](other-state-mgmt.md#6-other-state-management-functionality)
    - [6.1 Get notified of state variable changes `@Watch`](other-state-mgmt.md#61-get-notified-of-state-variable-changes-watch)
      - [6.1.1  Provisions for using `@Watch`](other-state-mgmt.md#611--provisions-for-using-watch)
      - [6.1.2 `@Watch` example with `@Link`](other-state-mgmt.md#612-watch-example-with-link)
      - [6.1.3 `@Watch` and custom component update example](other-state-mgmt.md#613-watch-and-custom-component-update-example)
    - [6.2 Application defined observed object variables `SubscribableAbstract`](other-state-mgmt.md#62-application-defined-observed-object-variables-subscribableabstract)
      - [6.2.1 Scenarios when to define an own `SubscribableAbstract` class](other-state-mgmt.md#621-scenarios-when-to-define-an-own-subscribableabstract-class)
      - [6.2.2 Provisions for using `SubscribableAbstract`](other-state-mgmt.md#622-provisions-for-using-subscribableabstract)
    - [6.3  Built-in Component Two-way Value Synchronization - '$$'](other-state-mgmt.md#63--built-in-component-two-way-value-synchronization---)
      - [6.3.1 Provisions for using `$$`](other-state-mgmt.md#631-provisions-for-using-)
    - [6.4 AnimatableData](other-state-mgmt.md#64-animatabledata)
      - [6.4.1 `IAnimatableArithmetic` interface](other-state-mgmt.md#641-ianimatablearithmetic-interface)
      - [6.4.2  `@AnimatableExtend` special `@Extend` function](other-state-mgmt.md#642--animatableextend-special-extend-function)
      - [6.4.3 Example](other-state-mgmt.md#643-example)
  - [7 Rendering Control Syntax](rendering-control-syntax.md#7-rendering-control-syntax)
    - [7.1 Conditional Rendering with `if`](rendering-control-syntax.md#71-conditional-rendering-with-if)
      - [7.1.1 `if  else ` and sub-component state](rendering-control-syntax.md#711-if--else--and-sub-component-state)
      - [7.1.2 Nested `if` statements](rendering-control-syntax.md#712-nested-if-statements)
      - [7.1.3 Provisions for using `if`, `else`, and `else if`](rendering-control-syntax.md#713-provisions-for-using-if-else-and-else-if)
    - [7.2 Repeated Content with `ForEach`](rendering-control-syntax.md#72-repeated-content-with-foreach)
      - [7.2.1 `ForEach` and subcomponent state](rendering-control-syntax.md#721-foreach-and-subcomponent-state)
      - [7.2.2 Nested use of `ForEach`](rendering-control-syntax.md#722-nested-use-of-foreach)
      - [7.2.3 Example of ForEach using optional `index` parameter](rendering-control-syntax.md#723-example-of-foreach-using-optional-index-parameter)
      - [7.2.4 Provisions for using `ForEach`](rendering-control-syntax.md#724-provisions-for-using-foreach)
      - [7.2.5 Common pitfall with using `ForEach`](rendering-control-syntax.md#725-common-pitfall-with-using-foreach)
  - [8 SwiftUI - eDSL Feature comparison](swiftui-feature-comparison.md#8-swiftui---edsl-feature-comparison)
    - [8.1 StateManagement comparison](swiftui-feature-comparison.md#81-statemanagement-comparison)
    - [8.2 Rendering comparison](swiftui-feature-comparison.md#82-rendering-comparison)
<!-- TOC END -->
# Revision History

| Revision Date | Revision Version | Revision Description / Modification description | Author |
| ---------------------------------- | ------------------------- | ------------------------------------------------------------ | --------------- |
| 2020.8.9 | 0.1 | Draft completed | Yang Jiandong 00404506 |
| 2020.9.17 | 0.3 | Minimized set | Yu Zhiqiang 00389682 |
| 2020.9.24 | 0.5 | Modified the sample information. | Yang Jiandong 00404506 |
| 200.11.05 | 0.6 | Added the facade description and removed the common module description. | Yang Jiandong 00404506 |
| 2020.11.06 | 0.7 | Deleted the redundant Facade information. Specify that the component needs builder as the UI body flag, and all attribute values are in small camel case. | Yu Zhiqiang 00389682 |
| 2020.12.1 | 0.8 | Updated the unified design approach. Updated Facade and decorator-related descriptions, including replacing and updating @Component to @Facade, replacing and updating @Prop to @Inherit, updating @State/@Event/@Listener, and deleting @Lifecycle/ @Builder/ @Bind/ @Callable | Yang Jiandong 00404506 |
| 2020.12.1 | 0.9 | Retained the external naming rules of @Component and @Prop, and deleted the description of some non-core features. | Yu Zhiqiang 00389682 |
| 2020.12.8 | 0.10 | Reorganize the directory structure | Sun Fei 00579415 |
| 2020.12.14 | 0.11 | Modified the prop information. The prop supports only simple types and is transferred from the parent component by value. | Yu Zhiqiang (employee ID: 00389682) |
| 2020.12.16 | 0.12 | Added the definition of @Binding decorator | Yu Litao 00447391 |
| 200.12.21 | 1.0 | Modified the version number | Wu Yonghui 00321266 |
| 2021.01.07 | 1.1 | Modified the struct object description. | Wu Yonghui 00321266 |
| 2021.01.12 | 1.2 | Machine translation to English | Guido Grassel, 00323864 |
| 2021.01.20 | 1.3 | @State, @Prop, @Binding clarified, ForEach clarified. Lifecycle methods and other issues remain | Guido Grassel 00323864 |
| 2021.01.27 | 1.4 | Renamed @Binding to @Link | Guido Grassel 00323864 |
| 2021.02.03 | 1.5 | Accepted chapter 2.5, 2.6, 2.7 | Yu Zhiqiqng, Wang Lei, proposer: Guido Grassel 00323864 |
| 2021.03.15 | 1.6 | Update new features added by HQ | Sun Fei 00579415 |
| 2021.03.18 | 1.7 | New features proposal for discussion and decision | Guido Grassel 00323864 |
| 2021.03.31 | 1.8 | New version accepted March 31 after HQ review. | Yu Zhiqiqng, Sun Fei, proposer: Guido Grassel 00323864 |
| 2021.06.23 | 1.9 | Copied eDSL spec text from several design docs | Guido Grassel 00323864 |
| 2021.07.02 | 1.10 | Many clarifications based on framework implementation experience and app developer feedback. Quick facts tables for most features | Guido Grassel 00323864 |
| 2021.07.04 | 1.11 | Updated AppStorage, PersistentStorage APIs based on review comments; full details for Environment API" | Guido Grassel 00323864 |
| 2021.07.20 | 1.12 | Updated PersistentStorage and Environment API function names. Clarified but did not change semantics. | Guido Grassel 00323864 |
| 2021.07.27 | 1.13 | Allow @Link source to be also @StorageLink | Guido Grassel 00323864 |
| 2022.02.14 | 1.14 | add `LocalStorage`, `@LocalStorageLink`, `@LocalStorageProp`, refactor description of`AppStorage` and related variable decorators accordingly, cleanup never completed section about Ability data sources and sinks. | Gido Grassel 00323864 |
| 2022.02.15 | 1.15 | add `LocalStorageLookup` | Guido Grassel 00323864 |
| 2022.03.03 | 1.16 | remove `LocalStorageLookup`, add `LocalStorage.GetShared()` no change form what has been defined in the design doc. | Guido Grassel 00323864 |
| 2022.03.23 | 1.16 | added  section 6.1.3 with clarification on @Watch processing order | Guido Grassel 00323864 |
| 2022.10.06 | 1.17 | added section 7.2.3 clarifying example for ForEach item with index | Guido Grassel 00323864 |
| 2022.11.08 | 1.18 | ForEach with optionakl index parameter<br/>AppStorage/LocalStorage link, prop return value SubscribedAbstractProperty | Guido Grassel 00323864 |
| 2022.12.16 | 2.0 | Review of the entire document, many revised and new examples, plentry of clarifications esp. in reference of the partial update optimisation. Moved to gitee.com pages. | Guido Grassel 00323864, Henna Myllys wx1107779 |
| 2022.12.20 | 2.1 | rules for initialization and update, changes to chapter 3 and to exch 'provisions of use ...' section in chapters 4 nd 5. | Guido Grassel 00323864, Henna Myllys wx1107779 |
| 2022.12.20 | 2.2 | section 2.4 @Builder and @BuilderParam copied from the design doc | Guido Grassel 00323864 |
| 2023.01.04 | 2.3 | Section 5 revisited for @StorageProp and @LocalStorageProp | Henna Myllys wx1107779 |
| 2023.01.11 | 2.4 | new sub-feature for `@Prop`, chapter 4.2: `@Prop` optional to init local, then optional to provide sync source | Tatu Tomppo wx908032, Guido Grassel 00323864 |
| 2023.01.18 | 2.5 | add missing update from @Prop local init and init from parent in section 3.2 | Tatu Tomppo wx908032 |
| 2023.01.18 | 2.6 | add section 3.3 Tutorial, explanations how to read to chapter 1 | Guido Grassel 00323864 |
| 2023.01.23 | 2.7 | Corrected typographical errors and fixed build errors of examples in sections 2.2.6, 4.5.2 and 7.2.2 | Vidhya Pria Arunkumar 00824002 |
| 2023.02.09 | 2.8 | Core Spec organized to multiple small files and support to generate ToC from script is added | Vidhya Pria Arunkumar 00824002 |
| 2023.02.23 | 2.9 | updates and additions to section 2.4 @BuilderParam and BuilderType | Guido Grassel 00323864 |
| 2023.03.22 | 2.10 | "@ObjectLink and @State ets reference the same object | Guido Grassel 00323864 |
| 2023.04.02 | 2.11 | `@Prop` shallow-copy for API rel.9 and deep-copy to achieve proper one-way sync for API rel.10 | Guido Grassel 00323864 |
| 2023.04.28| 2.12 | `AppStorage`, `PersistentStorage`, and `Environment` all public static API function names changed from upper case to lower case first letter to comply with newly introduced OHOS API style guidelines. | Henna Myllys wx1107779 |
| 2023.05.05 | 2.13 | Add `@Observed` class decorator implementation notes | Guido Grassel 00323864 |
||| Add support for `Date` type to `@State, @Link, @Prop, @Provide, @Consume, @ObjectLink` decorator definition, for API rel. 9. Support for storage related decorators still to be implemented for API rel. 10 ||
||| `SubscribableAbstract` for API rel. 10, typo in class name also corrected. ||
||| Add clarification on API extension from API rel to 10: `AppStorage` and `LocalStorage` static functions `prop` and `setAndProp` supported types `S` only up to API rel. 9, all types included in `T` starting API rel. 10. ||
| 2023.05.09 | 2.14 | new chapter 3.4 Good and Bad Practices in State Management | Vidhya Pria Arunkumar 00824002, Guido Grassel 00323864 |
| 2023.06.28 | 2.15 | Add support for `Map` and `Set` to `@State, @Link, @Prop, @Provide, @Consume` decorator definitions, for API rel. 10. Clarifications on observed value changes depending on the variable type. | Henna Myllys hwx1107779, Guido Grassel 00323864 |
| 2023.06.28 | 2.16 | Add `undefined`, `null` support, union types, permissive type check API 10 and strict type check API 11. Arising new pitfalls for ForEaxh and for `@ObjectLink`  | Guido Grassel 00323864 |
| 2023.06.28 | 2.17 | add specification for AnimatableData feature, new section 6.4 | Guido Grassel 00323864, Sarath Singapati 00336022 |
| 2023.06.28 | 2.18 | add `@StorageLink/Prop` and `@LocalStoageLink/Prop` cause delayed re0render of inactive @Component | Guido Grassel 00323864 |


# 1 Summary

This document defines the core mechanisms and features of the ArkUI Declarative UI Framework. It describes the declarative UI description, UI state management, initial UI rendering and UI update logic in response to state changes.

This document provides reference specifications for application developers to develop UIs. For specification details of UI components, their attribute functions and supported events see the ArkUI Components specification.

A first-time reader is recommended to read (in this order):
* [Chapter 2](general-ui-spec.md#2-general-ui-specifications)
* [Section 3.1](intro-state-mgmt.md#31-arkui-decorators-overview)
* [Section 3.3 includes a tutorial](intro-state-mgmt.md#33-tutorial---application-state-in-arkui)
* [Chapter 4](manage-state-component.md#4-managing-state-owned-by-a-component)
* [Chapter 5](ui-state-storages.md#5-ui-state-storages)
* [Chapter 6](other-state-mgmt.md#6-other-state-management-functionality)
* [Section 3.2](intro-state-mgmt.md#32-summary-of-provisions-for-decorated-variable-initialization-and-update) for provisions for variable local init, init from parent and update as needed while reading chapters 4 and 5
























---

# 2 General UI Specifications

The development framework provides a series of basic components that are combined and extended declaratively to describe the UI of the application. It also provides data binding and event processing mechanisms to help developers implement application interaction logic.

The following is an example code of an Hello World page:

```typescript
// An example of displaying Hello World. After you click the button, Hello Ace is displayed.
@Entry
@Component
struct Hello {
    @State myText: string = 'World';

    build() {
        Column() {
            Text('Hello')
                .fontSize(100)
            Text(this.myText)
                .fontSize(100)
            Divider()
            Button() {
              Text('Click me')
            }
              .onClick(() => {
                this.myText = 'Ace'
              })
              .width(500)
              .height(200)
              .backgroundColor(Color.Red)
        }
    }
}
```
The preceding example code describes the structure of a simple page and introduces the following basic concepts:
- Decorators: Decorated variables or classes to assign special meanings to them. For example, `@Entry`, `@Component` and `@State` in the preceding example are decorators.
- Customized component: Reusable UI unit, which can be combined with other components, such as the `struct Hello` decorated by `@Component`.
- UI Description: Declaratively describes the UI structure, such as the code block in the `build()` method. ArkUI defines its own syntax.
- Built-in component: default built-in basic components and layout components in the framework, which can be directly invoked by developers, such as `Column`, `Text`, `Divider` and `Button`.
- Attribute function: used to configure component attributes, such as `fontSize()`, `width()`, `height()` and `backgroundColor()`.
- Event method: used to add the component response logic to an event. The logic is set through the event method, for example, `onClick()` following `Button`.

## 2.1 Declarative UI Description

### 2.1.1 Creating components

Create components without the need for writing the `new` operator, provide mandatory and optional parameters:

```typescript
Column() {
    Text('item 1')
    Divider() // No parameter configuration of the divider component
    Text('item 2')
}
```

### 2.1.2 Attribute functions

The purpose of attribute functions is to configure styling and other properties of builtin components.

Example: set the `fontSize` attribute of the `Text` component as follows:
```Typescript
Text('123')
    .fontSize(12)
```

Attribute functions are chainable. Thereby setting multiple attributes can be done efficiently.
It is a good practice to use a separate line for each attribute function.

Example: Set multiple attributes of the `Image` component at the same time:
```Typescript
Image('a.jpg')
    .alt('error.jpg')
    .width(100)
    .height(100)
```

Attribute functions accept constant expressions, variables, enumeration types and other computed expressions just like regular TypeScript functions:

```Typescript
// Size, count, and offset are private variables defined in the component.
Text('hello')
    .fontSize(this.size)

Image('a.jpg')
    .width(this.count % 2 === 0 ? 100:200)
    .height(this.offset + 100)
```

For some attribute functions the framework also pre-defines some enumeration types. See the ACE Declarative Framework specification for attribute functions and permissible enumeration types.

Examples: Configure the `color` and `fontWeight` attributes of the Text component as follows:

```Typescript
Text('hello')
    .fontSize(20)
    .color(Color.Red)
    .fontWeight(FontWeight.Bold)
```

### 2.1.3 Handling events

Use attribute functions to define event handler functions. Button component and its `onClick` attribute function is just one example event ACE supports. See the ACE Declarative specification for reference on UI components, supported events and the attribute functions for defining the event handler. 

Example: Install a click event handler on a Button component:

```typescript
@State counter : number = 0l

build() {
    ...
    Button("Click me!")
        .onClick(() => {
            this.counter += 2
        })
}
```
This example will invoke the lambda function whenever the user touches the button. 
Event handler functions typically mutate @Component state. A @Component state variable change will cause the UI to be updated. This update happens asynchronously.

In this example, `counter` is a private data variable defined of the custom component.

`this` is available and refers to the owning component instance (more one components in the following sections).

Provisions for asynchronous processing in event handlers:
* Use of ES6 Promise and use of callback functions inside event handler is allowed. Such function is allowed to mutate app state. As the Promise resolve/failure or callback function execute asynchronously it could happen that the framework has marked the component and its state variables for deletion before the function is executed. Delayed function accessing component state can lead to unexpected side effects. Developers should pay attention and avoid callback functions and Promise functions to access any state after the component has been deleted. See the section about @Component lifecycle functions for further details and an example how to use `async` functions properly [link](./general-ui-spec.md#226-lifecycle-callback-functions) .
* Use of `async await` is not supported inside event handler function bodies for performance reasons.

## 2.2 Subdivision into custom components

### 2.2.1 Introduction

Defining the entire application UI with just predefined components would lead to monolithic design, hard to maintain and poor in execution performance. The purpose of `@Component struct` expression is for application to define their own components. A `@Component` defined by the application is also referred to as _custom component_.

A custom component can combine other components. It describes the UI structure by implementing the `build` function. 

The component-based solution has the following features:
- Combinable: allows developers to use built-in components and other components in a combined manner, as well as common attributes and methods.
- Reusable: can be reused by other components and used in different parent components or containers as different instances.
- A custom component has a well-defined lifecycle. Lifecycle callback functions allow the component to perform its own action upon specific lifecycle events.
- Data-driven update: Following the declarative UI paradigm, a custom component holds some state. Its `build` function defines how to render the UI based on the component state. Its UI is re-generated by executing (parts of) its build function whenever its state changes. The component is said to 'render' when its `build` function is executed first time after creating the component. The component is said to 're-render' when its `build` function is executed again as a consequence of some of its state changing.

```typescript
@Component
struct HelloComponent {
    @State say: string = 'Hello, World!';
    build() {
        Row() {
            Text(this.say)
                .fontSize(20)
                .fontColor(Color.Red)
        }
        .backgroundColor(Color.Green)
    }
}
```

The ArkUI framework executed the `build` function of `MyComponent` after the component instance has been created.

`MyComponent` can be created from other custom components and rendered within its layout.
Example: Create two instances of `HelloComponent` and combine it into a Column layout with a Text component.

```typescript
@Component
struct ParentComponent {
    build() {
        Column() {
            Text('ArkUI says')
              .fontSize(20)
            HelloComponent({say: "Hello, World!"})
            Divider()
             .height(10)
            HelloComponent({say: "你好!"})
        }
    }
}
```

Execution of the `ParentComponent` `build` function, i.e. rendering of `ParentComponent` will create two instances of `HelloComponent`, provide initializing values for `say` component state variable. The UI framework will create two instances of `HelloComponent` and render each separately. 


### 2.2.2 Provisions for defining a custom component

Definition of a custom component must start with `@Component struct` followed by the component name. The name must start with a capital letter and be at least two characters long. The name must be unique within the application. No global variable, function, type, class or other component with the same name is allowed. 

A custom component body must be enclosed by curly brackets `{....}`.

A custom component must implement the `build` function. It may implement additional `@Builder` custom build functions.  See the next section for the special rules governing definition of `build` function.

A custom component may implement the `aboutToAppear` and `aboutToDisappear` lifecycle callback functions.

A component must implement no constructor. The `ace-ets2bundle` compiler creates a constructor for framework internal purposes.

A component may implement other member functions and variables in the same way as an ES6 class with a few limitations:
* `readonly` variables are allowed as long as they are not decorated (with e.g. `@State`).
* Special naming conventions: Component member variable and function names must not start with the letter `$`, because `$` creates a reference for initialising a `@Link`. Naming a member `$$` is not allowed, this name is reserved for use in `@Builder` functions.
* Access to member variables and functins is always private. Defining `private` access is optional. Defining access other than `private` is a syntax error.
* Component variables and functions can not be static.

Much more about the use of component variables and ArkUI variable decorators is defined later in this specification in chapters 3, 4, and 5. 

Generally `this` refers to the component instance. It can be used for member variable and function access.

JavaScript -style single line (`//`) and multi-line (`/* ... */`) comments are allowed.


### 2.2.3 Default page entry component `@Entry`

A custom component decorated with `@Entry` is used as the default entry component of the page. At most one component can be decorated with `@Entry` in a single source file. The Page Navigation API provides own means for specifying the entry component of a page. The specification therein overwrites the @Entry decoration.  When the page is loaded, the entry component is created and rendered.  The usage of `@Entry` is as follows:

```Typescript
@Component
struct HideComponent {
    build() {
        Column() {
            Text('goodbye')
                .fontColor(Color.Blue)
        }
    }
}

@Entry
@Component
struct MyComponent {
    build() {
        Column() {
            Text('hello world')
                .fontColor(Color.Red)
        }
    }
}
```

The `@Entry` decorator accepts an optional parameter of type `LocalStorage`, see section 5.3.2 .

### 2.2.4 Provisions for custom component parameters

When creating or updating a custom component from a `build` or `@Builder` function, parameters can be supplied to set a variable of the component.


Example to create an instance of `MyComponent` and initialize its `countDownFrom` variable with the value `10` (Example of section 2.2.4):

```TypeScript
MyComponent({countDownFrom: 10, color: this.someColor})
```

Only defined variables of that @Component are allowed. Some @Component variables require initialization, for others initialization from the parent is optional, for some initialization is prohibited. Each variable decorator defines its own initialization rules. A summary table is available from chapter 3.2.

### 2.2.5 Custom Component Lifecycle

The life cycle of a custom component is as follows

Custom component creation and rendering:
1. Custom component creation: An instance of a custom component is created either by the UI framework (when it is the root component of a page) or when its parent component renders.
2. Initialization of component variables with locally defined default values, if supplied. This happens in document order.
3. Variable initialization with component constructor parameter, if supplied. 
4. The custom component is scheduled for initial render asynchronously.

The framework initiates initial render:
4. If defined, the components's `aboutToAppear` function is executed.
5. The `build` function of a component is executed the first time. The component is 'rendered'. Rendering creates instances of further sub-components as described in steps 1 - 4. - While executing build first time the framework observes read (get) access on each state variable. The framework constructs two mapping tables:
    * state variable -> UI component (and `ForEach` and `if`)
    * UI component -> update function for this component. A lambda, subset of build function, that just creates one UI component and executes its attribute functions.

An event handler is executed, mutates a state variable of the component. Or a property in LocalStorage / AppStorage changes and causes a linked state variable to change value. 
6. The framework observes the variable change. Marks the component needing re-render. Re-render will happen asynchronously. Before re-render actually happens more state variables might change. All needed updates will be performed at once.

The framework initiates initial re-render:
7. Using the two mapping tables created in step 5 the framework, and knowledge of the changed state variable the framework executes update lambda functions only for those UI components (and `ForEach` and `if`) that depend on changed state variables. While executing these lamdas the framework updates said two mapping tables.


Custom component deletion. A Custom component gets deleted because of a branch change in `if` or an update to `ForEach`.
8. Before deleting a component its `aboutToDisappear` lifecycle function is called.
9. The custom component and all its variables are deleted. Any `@Link, @Prop, @StorageLink`, etc. variables unregister from their sync sources.

### 2.2.6 Lifecycle callback functions

Callback functions notify about the lifecycle of a custom component. These can be defined as private member functions of a custom component. The run-time calls these functions at defined times. These functions must not be called from the application.

The following example illustrates the purpose of lifecycle functions:

```typescript
@Component
struct CountDownTimerComponent {

  @State countDownFrom : number = 0;
  private timerId : number = -1;

   aboutToAppear() : void  {
    this.timerId = setInterval(() => {
      if (this.countDownFrom <= 0) {
         clearTimeout(this.timerId);
      }
      this.countDownFrom -= 1;
    }, 1000); // decr counter by 1 every second
  }

   aboutToDisappear() : void {
    if (this.timerId > 0) {
      clearTimeout(this.timerId);
      this.timerId = -1;
    }
  }

  build() {
    Column() {
      Text(`${this.countDownFrom} sec left`)
    }
  }
}
```

Above example shows that lifecycle functions are essential to allow the `CountDownTimerComponent` to manage its timer resource and to be self contained.

A similar function could be made with a component that uses Fetch to asynchronously load content from the network. It should initiate Fetch on `aboutToAppear` and cancel any pending Fetch on `aboutToDisappear`. Writing the eDSL of such component is left as an exercise to the reader.

Life cycle functions are defined as follows:

| Function | Comment  |
|----------|----------|
| `aboutToAppear` | Function executes after creation of a new instance of a custom component and before execution of its `build` function. <br/> Mutation of a state variable in the `aboutToAppear` function is allowed. These changes will be used in the subsequent execution of `build`. |
| `aboutToDisappear` | Function executes just before deletion of a custom component. <br/> Mutation of a state variable in the `aboutToDisappear` function is not recommended. |

`this` is available and refers to the component instance.

 **Async functions and await** 

Life cycle functions, like any function called during the component build process, must do minimal processing to shorten the time for UI initial render.
Making a call from a life cycle functions to an `async` function is allowed. However, awaiting on a Promise (using `await`) is not allowed and also not supported. 

The following example provides a boilerplate how to properly use an `async` function to modify app state:
1. Call the async function, but do not use `await`. Pass any app state to it as needed. Use `Promise.then` to obtain the async function return value and update any app state.
2. The execution flow continues right away, without waiting for the function to complete. The lifecycle function ends.
3. Upon the async function completion the Promise resolves and the lambda provided to `then` executes. The async function return value is a parameter of this lambda. Use the paramer to update any app state with the async function return value.

```TypeScript

// async function returns the given parameter with a delay of 1sec.
async function asyncProcessMock(param: number) : Promise<number> {
  const param_ = param;
  return new Promise(resolve => setTimeout(() => {
    resolve(param_);
  }, 2000));
}

@Entry
@Component
struct ComponentAsync {
  @State someInputState : number = 47;
  @State someModifiedState : number = 0;


  aboutToAppear() {
    console.log("before async function call");
    asyncProcessMock(this.someInputState).then((asyncFunctionReturnValue : number) => {
      console.log(`async function returns ${asyncFunctionReturnValue}, updating state`);
      this.someModifiedState = asyncFunctionReturnValue;
    })
    console.log("after async function call");
  }

  build() {
    Column() {
      Text(`async func input ${this.someInputState}, async func modifies state ${this.someModifiedState}`)
    }
  }
}
```

Note the order of log messages to understand the processing order:
```Text
Execution of aboutToAppear
1. before async function call"
2. after async function call

// with a delay of 1 sec
// execution of Promise resolve lambda 
3. async function returns 47, updating state
```

 **async function execution from `aboutToDisappear`:** 
Developer take note of thsi special restriction for `aboutToDisappear`: In case of an asynchronous operation (a Promise or a callback function) being started from `aboutToDisappear` function the custom component will remain in the closure of the Promise resolve and failure functions until after that function has been called (which prevents the component from being garbage collected). Mutating component's state variables inside the a Promise resolve or failure function (or callback function) is _not allowed_. The reason is that the framework has already deleted some of the references to this component and its state when the function is executed asynchronously.

## 2.3 Domain specific syntax of the `@Component build` and `@Builder` functions

Every custom component must define how to render in its `build` function. The function takes no parameters and returns nothing. The function body follows a special domain specific syntax in the `ArkUI eDSL`. The purpose of the eDSL is two-fold:
- shorter, easier to write
- flexibility for run-time optimization thru compilation (with `ace-ets2bundle`)

The same rules apply to the `@Component` `build`, `@Builder` function, `ForEach` item build function and `if else` build functions for each branch. 

The `build` function body is enclosed by curly brackets `{ ... }`.

Exactly one top level component is required:
The root component is typically a container component, like `Column`. `if` or `ForEach` are not allowed at the top level. Same rules apply to `ForEach` item build function and `if else` build functions, except these allow for one or more top level component.

At most one component can be defined per line:
E.g. `Text("foo"); Image("bar.jpeg")` in one line is a syntax error.

Chaining of attribute functions:
Attribute functions can be chained. It is good practice to use an own line for each attribute function as well.  The semicolon (`;`) after the last attribute function can be skipped.

Single line (`//`) and multi-line (`/* ... */`) comments are allowed.

Code inside `build` and functions called from `build` _must not mutate any application state_, e.g. `Text('${count++}')` is a syntax error.

Use of any other TypeScript expression is disallowed. 
For instance
* use of `console.log` is not allowed
* No JavaScript expressions
* No local variable inside `build`.
* No function calls, with four exceptions: 
    - calling an `@Builder` function, for details see definition of `@Builder` functions.
    - calling the `@Builder` function stored in a `@BuilderParam` variable of `BuilderType<C>` type, for details see definition of `@BuilderParam` variable decorator.
    - call to attribute, event handler declaration, `@Extend`, or `@Styles` functions
    - create sub-@Components
    - TS functions that supply parameter values to afo  rementioned functions, e.g. `Text(this.label.substring(15))).width(Math.min(this.a,100))`
Remember: None of these functions must not mutate any application state. Example: .

Examples of what is not allowed:

```typescript
build() {
    let a: number = 1        << invalid: variable declaration not allowed
    console.log(`a: ${a}`)   << invalid: console.log not allowed
    Column() {
        Text('Hello ${this.myName.toUpperCase()}')  << ok.
        Text('Hello'); Text('World');     << invalid, two components on the same line
    }
    {                   <<  creation of alocal scope not allowed
        this.doSomeCalculations()     << invalid: no function calls except @Builder functions
    }                   << creation of a local scope not allowed
    Text(this.calcTextValue())  << ok calling a TS function to calc the parameter value is generally ok

    new MyOtherCustomComponent()  << do not use 'new'
}
```

Below `build` is invalid, _exactly one_ top level component is required.
```typescript
build() {
   Text('Hello ${this.myName.toUpperCase()}') 
   Text(this.calcTextValue())    << invalid 2nd top level component
}
```

Below `build` is also invalid, no `if` or `ForEach` at the top level, 
exactly one top level component is required.
```typescript
build() {
    if (this.foo) {    << invalid, ForEach and if not allowed on build top level
      Text('Hello ${this.myName.toUpperCase()}') 
    } else {
      Text(this.calcTextValue()) 
    }
}
```

`switch` is not supported, therefore the following example is invalid. Use `if .. else ` instead:
```typescript
build() {
  Column() {
    switch(expression) {
      case 1:
        Text("...")
      case 2: 
        Image("...")
      default:
        Text("...")
    }
  }
}
```

The following is valid only if `aFunction` is a `@Builder` function. If it is a regular function it is not allowed:
```typescript
build() {
  Column() {
    this.aFunction()
  }
}
```

Expressions are not allowed, this makes the following example invalid:
```typescript
build() {
  Column() {
    (this.aVar > 10) ? Text("...") : Image("...")
  }
}
```

## 2.4 Custom build functions `@Builder`

As explained the `@Component build` function defines how to render the UI of a custom component, it follows special syntax rules. An `@Builder` decorated function is a special function that serves similar purpose as the `build` function, i.e. to render some components. The `@Builder` function body follows the same syntax rules as the `build` function.  To simplify language we refer to a `@Builder` decorated function also as a 'custom builder function'.

ArkUI Declarative supports two variations of custom builder functions: Custom builder function owned by a custom component and global custom builder function. The `@BuilderParam` variable decorator is required for a `@Component` member variable to hold a function reference to a `@Builder` function.

### 2.4.1 Example for custom build functions with by-reference parameter passing

Lets study below example:
* `PropChild` component calls the global custom builder function `ABuilder` passes a non-decorated member variable and the `@Prop` member variable.
* `LinkChild` calls the same function, passes a constant for `label` parameter and the `@Link` member variable.
* `ABuilder` created the `@Component` `CreatedByBuilder` that expects a reference to a state variable to establish a two-way sync with its `link` state variable.

The way we expect this example to work is that
* When `s` changes in `Parent` it updates the `@Prop ps` and `@Link ls` in both sub-components, 
* change of `@Prop ps` or `@Link ls`is expected to `paramB1` of the builder function 
* change of `paramB1`  builder function parameter performs the expected two-way sync with `CreatedByBuilder link`. 

Here we see a special feature of builder function parameters that use the `$$` mechanism: 
Regular TS/JS function parameters are always _by-value_, means a simple type variable value is copied, and also an Object reference is copied into the special function `arguments` array. If builder functions used by-value parameter passing, then, the `@Link link` source would not be the state variable itself as expected, but just its copied string value. ArkUI special  `$$`  mechanism causes _by-reference_ parameter passing. C++ developers know the difference when adding a `&` after a C++ function parameter.


```TypeScript
@Component
struct CreatedByBuilder {

  label : string = "no label";
  @Link link : string;

  build() {
    Text(`CreatedByBuilder ${this.label} ${this.link}`)
      .width(200).height(100)
  }
}

@Builder function ABuilder( $$ : { paramA1: string, paramB1 : string } ) {
  Row() {
    Text(`@UseStateVarByReference: ${$$.paramB1}`)
      .width(200).height(100)

    CreatedByBuilder({ label: $$.paramA1, link: $$.$paramB1 } )
  }
}

@Component
struct LinkChild {

  @Link ls : string;
  private label : string = "LinkChild";

  build() {
    Column() {
      ABuilder({paramA1: this.label, paramB1: this.ls })
    }
    .width(400).height(200)
    .onClick(() => {
      console.log("LinkChild ls changing");
      this.ls = (this.ls == "Hello!") ? "Hej" : "Hello!";
    })
  }
}

@Component
struct PropChild {
  @Prop ps : string;

  build() {
    Column() {
      ABuilder({paramA1: "PropChild", paramB1: this.ps })
    }
    .width(400).height(200)
    .onClick(() => {
      console.log("PropChild ps changing");
      this.ps = (this.ps == "Hello!") ? "Hej" : "Hello!";
    })
  }
}

@Entry
@Component
struct Parent {
  @State s : string = "Hello!";

  build() {
    Column() {
      LinkChild({ls : this.$s})
      Divider().height(10)
      PropChild({ps : this.s})
      Divider().height(10)
      Text(`Parent state'${this.s}'`)
        .width(400).height(50)
    }
  }
}
```

### 2.4.2 Explanations on how ArkUI supports by-reference parameter passing

This section is informative (i.e. non-normative), it may change in future versions of the ArkUI compiler and framework.

JS does not have by-reference passing like C++, so how does ArkUI achieve by-reference passing? - The $$ mechanism is part of the ArkUI eDSL. The ArkUI compiler changes the eDSL to regular JS. In the JS compiler output the builder function calls change:

eDSL function signature
```TypeScript
ABuilder( $$ : { paramA1: string, paramB1 : string } );
```

compiler intermediate TS function signature (the compiler output is JS, the intermediate TS is more telling because of the still present type information):
```TypeScript
ABuilder( $$ : { paramA1: string, paramB1 : string } ) : void;
```

Nothing changes, e.g. any parameter use in function body `$$.paramA1` is expected to return the string value.

The magic is in the compiled version of the function call, and in a ES6 Proxy for the `$$` Object:


eDSL function call
```TypeScript
ABuilder({paramA1: "PropChild", paramB1: this.ps })
 ...
ABuilder({paramA1: this.label, paramB1: this.ls })
```

compiler intermediate TS for function call (simplified):
```TypeScript
ABuilder( $$ : { paramA1: (): string => "PropChild", paramB1 : () : string => this.__ps} ) : void;
...
ABuilder({paramA1: (): string => this.label, paramB1: () : string => this.__ls})
```

So, each parameter turns into a lambda with `this` referring to the calling custom component. When executed the lambda returns a constant value, regular or state variable:
* `{ paramA1(): string => "PropChild" ... }`  - lambda returns constant string
* `{ paramA1(): string => this.label ... }`   - lambda returns the value of `this.label`
* `{ paramB1: () : string => this.__ps ... `} - lambda returns the `@Prop ps` state variable framework implementation object(`__variableName`, never access it directly from your app code!)
* `{ paramB1: () : string => this.__ls ... `} - dito for the component `@Link ls`

The builder function body implementation (see function signature above) expects a value not a function, this is where a ES6 Proxy wrapped around the `$$` Object comes to the rescue:

A much simplified function implementation just for first above builder call:
```JavaScript
function makeProxy($$ : Object) : { paramA1: string, paramB1 : string } {
   return new Proxy( /* Handler */ {
      get (target, prop) {
        if (prop=="paramA1") {
          return target["paramA1"]();  // execute lambda, will return constant "PropChild"
        }
        if (prop=="paramB1") {
          return target["paramB1"]().get() // lambda return this.__ps. To get value, call get() to return string value.
        }
        if (prop=="__paramB1") {
          return target["paramB1"]() // lambda return this.__ps, the framework implementation of @Prop.
                                    // this is good to create a two-way sync with '@Link link'
        }
      }
   }, /* proxied Object */ $$);
}

// make the call to builder 
ABuilder( $$ : makeBuilder({ paramA1: (): string => "PropChild" as string, paramB1 : () : string => this.__ps as string} ));
```
Note also, the `makeProxy` function return type is what is expected by the `ABuilder` function signature.

The framework includes one `makeProxy` function implementation, it uses TS genetics. There is no need to generate a special proxy for every builder function.

### 2.4.3 Provisions for custom builder functions

***`@Builder` function inside a custom component:***

Defining one or several custom builder (`@Builder` decorated) functions inside a custom component is permissable:
- It can be thought of like a private, special kind of member functions of that component. 
- The name of a custom builder function inside a custom component must start with a _lowercase_ letter and be at least two characters long. 
- It can be called from the owning component's 'build' or another custom builder (within that custom component) function only. 
- Inside the custom builder function body, `this` refers to the owning component. Component state variables are accessible from within the custom builder function implementation. Using the custom components state variables is recommended over parameter passing where app logic allows.

Syntax for defining 
`@Builder myBuilderFunction({ ... })`


The syntax for invocation 
`this.myBuilderFunction({ ...})`. 
The use of `this` to call a component's custom builder function is required to distinguish from calling a global custom builder function.


***Global `@Builder` functions:***

Syntax for defining 
`@Builder function 
MyGlobalBuilderFunction({ ... })`


The syntax for invocation 
`MyGlobalBuilderFunction({ ...})` 

A global custom builder functions is much the same as a component-level custom builder function but it is accessible from the entire application.`this` is not available and can _not_ be bound (by using `bind`) either.
- The name of a global custom builder function must start with a capital letter and be at least two characters long. The name needs to be unique within the application. No framework component, no custom component, no type, class or global TS function or variable with the same name is allowed.

Adding a global custom builder function to an application can be a lightweight alternative to defining a custom component. Use of a global custom builder function is recommended if no own state (beyond function parameters) is required.

> `@Builder, @BuilderParam, and BuilderType` are features for API9, for which end-to-end support is still unter development.

### 2.4.4 Provisions for custom builder functions by-reference parameter passing

Always use by-reference variable passing, by-value passing is depreciated!
It is not allowed either to combine both types in a single function.

The rules:
- Named (or object) notation is to be used, must use `$$` as object name.
- A caller must provide all defined `@Builder` functions parameters. 
- Optional parameters (`?`) are not supported, because ArkUI does not allow rendering UI components from `undefined` value.
- Parameter names must not start with `$` character.
- types must match, `undefined` or `null` constants as well as expressions evaluating to these values are not allowed.
- All parameters are immutable (1). Their purpose is to enable UI updates. If mutability is required the custom builder should be replaced by a custom component with a @Link variable.
- A `@Builder` function always returns `void`.
- For defining a builder function with no parameters the short-hand signature `() => void` can be used.  The ArkUI compiler will expand the short-hand to `($: {}) => void` .

(1) ArkUI might support mutable parameters in the future. This allowed scenarios where the parameter is assigned a new value in event handler implementation.

Permissable use of a builder function parameter in function implementation depends on the value or expressions it is passed.

| JS expression in function invocation | builder impl can access value | builder impl can init `@Prop` | can init `@ObjectLink` | can init `@Link` | UI comps created by builder will update |
|--------------------------------------|------------------|----------------|----------------|----------------|----------------|
| `"Hello"`, `47`, `{ a: 101 }` - constant value | yes | N/A | N/A | N/A | no |
| `this.regular` - non-decorated (non-state) member variable of some type | yes | N/A | N/A | N/A | no |
| lambda function `(...args) : T => { .... }`  | yes | N/A | N/A | N/A | no |
| `2 * this.regular` - expression involving non-state member variable | yes | no | no | no | no |
| `this.aState` - state variable | yes | if decorator of `aState` allows  | if decorator of `aState` allows | if decorator of `aState` allows | yes |
| `this.aBuilderParam` - `@BuilderParam` variable of type `BuilderType<C>` | yes  | N/A | N/A | N/A | N/A |
| `sqr(this.ths.regular + this.aState2)` - state variable(s) in expression | yes | N/A | N/A | N/A | yes when a state var used in expression changes |
| `item` in `ForEach` item builder, from state var `Array<T>` | yes | yes, see `@Prop` for full details | yes if `T` is `@Observed` decorated class object | N/A | yes | 
| `this.arr[47]` state var `Array<T>` item | yes | as above | as above | as above | yes |
| `this.obj.b` state var Object, `b : T` | yes | yes, see `@Prop` for full details | yes if `T` is `@Observed` decorated class object | N/A | yes |

Developer advise: When developing a `@Builder` function for other app developers to use, it is important to clarify what kinds of input it expects. E.g. when  your builder function uses a parameter to create a `@Link` you need to make it clear that this parameter must only be initialized with a state variable from which a `@Link` can be created.  And when calling a `@Builder` function, double check from its implementation what each parameter is used for.

> `@Builder, @BuilderParam, and BuilderType` are features for API9, for which end-to-end support is still unter development.

### 2.4.5 Provisions for custom builder functions by-value parameter passing

Always use by-reference variable passing, by-value passing is depreciated!
It is not allowed to combine both types in a single function.

`@Builder` functions by-value parameters are the same as regular TS function parameters.

Changes to state variables, whose value is supplied to the builder function, will not cause components to update that are created within the builder function. Therefore, the use of by-value parameters is no recommended to be used with ArkUI API rel. 9 or newer, future versions will remove support.

The rules:
- A caller must provide all defined @Builder functions parameters. 
- Optional parameters (`?`) are not supported, because ArkUI does not allow rendering UI components from `undefined` value.
- types must match, `undefined` or `null` constants as well as expressions evaluating to these values are not allowed.
- All parameters are mutable but changes wil not sync back to calling component.

> `@Builder, @BuilderParam, and BuilderType` are features for API9, for which end-to-end support is still unter development.

### 2.4.6 Custom component variable holding a @Builder function reference  `@BuilderParam` and `BuilderType<C>`

`@BuilderParam` is the variable decorator to be used for custom component member variables of type reference to `@Builder` function.

The framework defines `BuilderType` as `type BuilderType<C extends Object> = ($$ : C) => void`.

A `@BuilderParam` variable type is always of specialization of `BuilderType<C>`.
For example:
`@BuilderParam aBuilder : BuilderType<{label: string}>` can hold a reference to `@Builder function SomeBuilder($$ : { label: string })`  but no reference to `@Builder function SomeOtherBuilder1($$ : { })` or `@Builder function SomeBuilde1r($$ : { label: string, n : number })` because of function parameter type mismatch.

In the following example custom component `CC` has two `@BuilderParam` member variables.
When creating an instance of `CC` these variables are initialized just like any regular (non-decorated) member variable of a custom component.
 
```TypeScript  
@Builder function GlobalBuilder0() {  // short for @Builder function GlobalBuilder0($$ : {}) 
  Text("0 - global builder")
        .width(400)
        .height(50)
        .backgroundColor(Color.Yellow)
}

@Builder function GlobalBuilder1($$ : {label: string }) {
  Text($$.label)
        .width(400)
        .height(50)
        .backgroundColor(Color.Blue)
}

@Component
struct CC {

  @Prop size_ : number;

  @Builder doNothingBuilder() { 
    // does nothing
  };
  
  @BuilderParam aBuilder0 : BuilderType<{}> = this.doNothingBuilder;
  @BuilderParam aBuilder1 : BuilderType<{ label : string }>;
  
  build() {
    Column() {
      this.aBuilder0()       // short for this.aBuilder0({})
      this.aBuilder1({label: "1 - global Builder label" } )
    }
    .height(3*50)
  }
}

@Entry
@Component
struct CCUser {

  @State size_ : number = 1;

  @Builder componentBuilder() {   // short for @Builder componentBuilder( $$ : {})
      Text(`2 + 3 - component builder`)
        .width(400)
        .height(50)
        .backgroundColor(Color.Green)
    }

  build() {
    Column() {
      Text("Start")

      CC({size_: 1, aBuilder0: GlobalBuilder0, aBuilder1: GlobalBuilder1})

      Text("End")
    }
  }
}
```

### 2.4.7 `@Builder` function as parameter to another `@Builder` function

In few app scenarios it might be useful to supply a `@Builder` function reference to another `@Builder` function.
This is what happens in these lines of the following function:

```TypeScript
@Component
struct Child {

  ...
  @BuilderParam aBuilderParam : BuilderType<{param1 : string}>;

  @Builder childBuilder($$ : { builderFunc: BuilderType<{param1 : string}> }) {
    $$.builderFunc({ param1: this.childState})    
  }

  build() {
    Column() {
      this.childBuilder({ builderFunc: this.aBuilderParam })
  ...
```

`childBuilder` `@Builder` function gets invoked with the a reference to another `@Builder` function.
It is essential that the `childBuilder` be defined with the `BuilderType<{...}>` type. This is how to 
UI compileridentifies the function as a `@Builder` function and allows calling it from the `@Builder` function body:
 `$$.builderFunc({ param1: this.childState}) `


The full example source code:

```TypeScript

@Component
struct Child {

  @State childState : string = "childState value";
  @BuilderParam aBuilderParam : BuilderType<{}>;

  @Builder childBuilder($$ : { builderFunc: BuilderOneStringType }) {
    $$.builderFunc({ param1: this.childState})    
  }

  build() {
    Column() {
      this.childBuilder({ builderFunc: this.aBuilderParam })
      Button("Mod this Child state")
      .height(75)
      .onClick(() => {
        this.childState = `C${this.childState}`;
      })
    }
    .height(3*50)
  }
} 



@Builder function GlobalBuilderAsParam($$ : { param1: string }) {
      Text( `GlobalBuilderAsParam builderParam ${$$.param1} .` )
        .width("100%")
        .height(60)
        .backgroundColor(Color.Green)
    }


@Entry
@Component
struct Parent {

  @State parentState : string = "parent state value";
  
  @Builder parentComponentBuilder($$ : { param1: string }) {
      Text( `ParentCompBuilder parentState: ${this.parentState} builderParam ${$$.param1} .` )
        .width("100%")
        .height(60)
        .backgroundColor(Color.Green)
    }

  build() {
    Column() {
      Text("Component Builder func param to Component Builder func")
      Child({ aBuilderParam: this.parentComponentBuilder})
      Button("Mod Parent state")
        .height(75)
        .onClick(() => {
          this.parentState = `P${this.parentState}`;
        })
      Divider().height(8)
      Text("Global builder func param to Component Builder func")
      Child({ aBuilderParam: GlobalBuilderAsParam })
    }
  }
}
```
### 2.4.8 Provisions for using `@BuilderParam`

`@BuilderParam` is the component variable decorator for `@Builder` function references:

`@BuildParam` variable initialization:
* Can only assign a `@Builder` function reference or undefined. Attempt to assign a regular TS function is a syntax error.
* Default variable value is `undefined`.
* Variable is optional to initialize locally. Initialization with a custom component member `@Builder` function or a global @Builder function reference.
* Variable is optional to initialize from the parent.
* In case of `undefined` variable value the framework skips function execution.
* A custom component builder function can use `this` in its function body. When passing the reference to this function to a `@BuilderParam` variable of a sub-component the framework ensures `this` is bound correctly. The app should _not_ call `bind(this)` by itself.

`@BuildParam` variable type definition:
* The type of a `@BuilderParam` must always be `BuilderType<C>` and `C` must be an `Object` with zero or more properties. The type of such property can be simple, enum, object (incl. array and function) type. The use of user type specification for `@BuilderParam` is depreciated, including specifications of the form `BuilderParam aBuilder = ($$ : { a: string, b: ClassA }) => void;` 
* The type of the `@BuilderParam` variable and the assigned function's signature must match exactly. By-reference function parameters are supported. `BuilderType<{}>` and `@Builder aBuilder()` match types because this is a short hand for `@Builder aBuilder($$ : {})`.
 
Invoking a @BuilderParam function:
* To invoke a `@BuilderParam aBuilder : BuilderType<C>` function from inside component `build()` use `this.aBuilder({...})`.
* To invoke a `aBuilderFuncParam : BuilderType<C>` function from inside another @Builder function use  `$$.aBuilderFuncParam({...})`.
* In both above cases, the framework checks `this.aBuilder` / `aBuilderFuncParam` for `undefined` value, and skips the call.
* It is not possible to invoke a function of type other than `BuilderType<C>`.

> `@Builder, @BuilderParam, and BuilderType` are features for API9, for which end-to-end support is still unter development.

## 2.5 Extending built-in components with `@Extend`

`@Extend` adds a new attribute function to a framework UI component. `@Extend` makes the function available in the entire application. The new attribute function is a composition of existing attribute functions and pre-defined values, like shown in below example:

```typescript
@Extend(Text) 
function fancy(color:string){
    .backgroundColor(color)
}

@Extend(Text)
function superFancy(size:number){
    .fontSize(size)
    .fancy(Color.Red)
}

@Extend(Button)
function fancy(color: string){
    .backgroundColor(color)
    .width(200)
    .height(100)
}
```

Above example adds two new attribute functions to the `Text` component and one new to `Button` component.
A custom attribute function can invoke all pre-defined and custom attribute functions permissible for the extended component. 

No other syntax elements are available, e.g. use of `If` is not permissable. Calls to functions other than attribute functions are not allowed. `this` is not available inside a custom attribute function body. `@Extend` can not be applied to a custom component. 

A custom attribute function must be defined before it can be used. @Extend adds the function globally to all instances of a built-in component. It is therefore advisable to give a custom attribute function a descriptive and long enough name to avoid name collisions with other extensions.

This custom attribute function can be used just the same way as any of the pre-defined attribute functions. Regular TypeScript provisions for function parameters apply.

There is no problem with UI updates when passing a state variable as an `@Extend` function parameter. Passing the variable to the function generates a 'get' call on this variable. That is sufficient for the framework to realize the dependency from the state variable to the component onto which the custom attribute function is applied.

```typescript
@Component
struct FancyUse {
    build() {
        Row() {
            Text("Just Fancy")
              .fancy(Color.Yellow)
            Text("Super Fancy Text")
              .superFancy(24)
              .height(70)
            Button("Fancy Button")
              .fancy(Color.Green)
        }
    }
}
```

## 2.6 Definition of resusable styles with @Styles 

>
> chapter still to be added
>


---

# 3. Introduction to State Management

In the _declarative UI programming style the UI is a function of applications state_. The developer defines how to render the current application state. The ArkUI Declarative UI framework provides many possibilities for easily defining UI dependencies on application state.


Custom components own variables. The framework provides multiple variable decorators with different semantics. A variable _must be decorated_ whenever rendering of one or several components depends on this variable. Failure to do so will still give correct initial rendering but will result in _loss of UI updates_.

Some glossary: A 'state variable' is a variable, on which UI rendering depends. Hence a non-state variable could be a helper variable in some calculation. All state variables need to be decorated. Therefore, we use the terms 'state variable' and 'decorated variable' like synonyms.

## 3.1 ArkUI decorators overview 

ArkUI decorator overview, details in chapters 3 and 4:

`T` refers to the following types: class, `SubscribaleAbstract`, number, boolean, string and enum; or Array of one of these types.

`@State`: An @State decorated variable holds some state _owned_ by this component. It can be the source of one or two way data sync with child components.  Whenever the variable changes (e.g. caused an event handler function execution, or data input in an entry field) dependent component will be updated. Allowed types are `T`.

`@Prop` and `@Observed` class decorator: Use the `@Prop` variable decorator to create a one-way sync with a variable of its @Component's parent.  Its value sync from the variable of the parent to the @Prop variable. The `@Prop` variable is mutable, but changes do not sync to the parent. The `@Prop` source must be a decorated variable (details see section about `@Prop`) itself, item of decorated array, or property of decorated class object. If the @Prop itself is a class object or array in the latter two scenarios then its class must be decorated with the `@Observed` class decorator.  Allowed types of a `@Prop` variable are `T`, must match the type of the sync source.

`@Link`: Use the @Link variable decorator for two-way data sync with a variable of its parent @Component: When the `@Link` variable value changes its source is updated as well, and also when the source updates the `@Link` will do as well. Allowed types are `T`, must match the type of the sync source.

`@Provide` and `@Consume`: `@Provide(alias)` works the same as `@State` except it makes the variable available as a sync source to all descendent @Components. The descendent @Components who can bind to the provided variable using `@Consume(alias)`. Provided and consuming variable names (or alias if used) and variable types must match.  `@Provide` and `@Consume` variables sync two-ways. Allowed types are `T`, type of the provided and consumed variable must be the same.

`#BuilderParam` decorates a component variable of special type `BuilderType<C>`. This variable can hold a reference to a `@Builder` function.

`@ObjectLink` variable and `@Observed` class decorators are for two-way data sync in scenarios involving nested objects or arrays:
A child @Component to render the UI from an `@ObjectLink` decorated variable of class object type. That class needs to be decorated with the `@Observed` class decorator. This variable to sync (two-way) object property changes with a source in parent @Component. The source can be an item inside decorated array type variable, or it is a property inside decorated class object type variable. `@ObjectLink` must be of class object type, must match the class of the source. 

Note that decorating a class with `@Observed` alone has no effect. Combined use with `@ObjectLink` for two-way sync or with `@Prop` for one-way sync is required. 

Application-wide state

`LocalStorage` is an in-memory 'database' for application state declared by the applications and typically used to share state across multiple applications pages or Activities.  

`AppStorage` is a special `LocalStorage` singleton instance created by the ArkUI framework. It should be used for application state that is used throughout the UI of the entire app.

`@StorageLink(propertyName)`: variable decorator works like `@Consume(alias)` with the difference that the linked property with given name is obtained from the `AppStorage`. Its a two-way value synchronization between the component and the property inside `AppStorage`. Allowed types are T. The decorated variable type and the type of the property in AppStorage must match.

`@StorageProp(propertyName)` variable decorator also synchronizes (single direction) a component property from `AppStorage`. A value change in `AppStorage` updates the property in the component. Changes to the decorated variable do not sync back to `AppStorage`. Allowed types are T. The decorated variable type and the type of the property in AppStorage must match. 

`AppStorage` class has an API for business logic implementation to add, read, modify and remove app state properties. Changes made thru this API cause said state data synchronization with UI components.

The UI framework provides more ready-made solutions for common concerns of business logic in regard to UI state:

`PersistentStorage` persists selected `AppStorage` properties on device disk. The framework offers an API for the app to decide which AppStorage properties should be persisted with the help of `PersistentStorage`. UI or business logic do not access properties in `PersistentStorage` directly. All property access is to `AppStorage` from where changes sync to `PersistentStorage` automatically.

`Environment` is a source of immutable properties that describe the environment in which the app is running. `Environment` sync's all environment property changes to `AppStorage`. The properties it adds to `AppStorage` are all immutable and can be linked to using `@StorageProp`.

## 3.2 General provisions for decorated variables

Thsi section states provisions common to multiple variable decorators. - A first-time reader of this spec might skip over section 3.2 and come back to it later when reference is needed.
* [Section 3.2.1](intro-state-mgmt.md#321-provisions-for-variable-type) permissable types depending on the variable decorator
* [Section 3.2.2](intro-state-mgmt.md#322-provisions-for-local-variable-initialization) summarizes the rules for local initialization, 
* [Section 3.2.3](intro-state-mgmt.md#323-provisions-for-variable-initialization-from-the-parent) summarizes for initializing a variable in a child @Component from a variable in the parent @Component. 
* [Section 3.2.4](intro-state-mgmt.md#324-provisions-for-variable-update-from-the-parent) summarizes for sub-sequent value update. These tables provide good reference. 



### 3.2.1 General provisions for variable type

The purpose of decorating a variable is to mark it as input to @Component rendering. All variables must be of declared type, and TS type `any` is not allowed.
The framework observes changes of decorated variables and triggers component rerendering automatically. What kinds of changes it can observe depends on the variable type.

`ArkUITypes` type includes all types that ArkUI can generally supports. There are restrictions though, for specific variable decorators.

The `ArkUIType` formal specification in TS can be found below, in human language the types are (further details later in this spec).
* simple types: `number, string, boolean`, enum with `string` or `number` values 
* JS builtin types: `Date`, `Map` (from API 10), `Set` (from API 10), `Array`, and `[]`
* Application-defined classes that prefer to notify about relevant value changes by themselves. These classes need to extend from the abstract base class `SubscribableAbstract` defined by ArkUI.
* Other application-defined ES6 classes and Objects with purely TS implementation (no C++). 

> As of Aug'23 the framework implementation restricts support for ES6 classes extending from Map, Set, and Date to @Observed decoration.
Such classes can not have own methods or other properties. Furthermore, we are being blocked from committing missing code for Set and Map change observation. Hence, do not use Map and Set until we have removed this note from the Core Spec.

```TypeScript
type ArkUITypes = ArkUISimpleTypes | ArkUIObjectTypes | ArkUIOwnTypesNoUndefined;
type ArkUISimpleTypes = ArkUISimpleTypesNoUndefined | undefined;
type ArkUIObjectTypes = ArkUIObjectTypesNoUndefined | undefined | null;
type ArkUISimpleTypesNoUndefined = number | string | boolean | Color;
type ArkUIObjectTypesNoUndefined = Date | Array< ArkUITypesNoUndefined > | ArkUITypesNoUndefined[] | Object | Map<ArkUITypesNoUndefined, ArkUITypes> | Set<ArkUITypesNoUndefined> | SubscribableAbstract | Resource;
type ArkUIOwnTypesNoUndefined = Resource | Color | ResourceStr | Length | ResourceColor;
type ArkUITypesNoUndefined = ArkUISimpleTypesNoUndefined | ArkUIObjectTypesNoUndefined | ArkUIOwnTypesNoUndefined;
```

Object means here only Objects that are ES6 classes or JS objects and implemented purely in TS/JS (no native implementation).

`SubscribableAbstract` is defined in [section 6.2.2](https://gitee.com/arkui-finland/arkui-edsl-core-spec/blob/master/other-state-mgmt.md#622-provisions-for-using-subscribableabstract):
```TypeScript
declare abstract class SubscribableAbstract {
    constructor();
    protected notifyPropertyHasChanged(propName: string, newValue: any): void;
    public numberOfSubscribers(): number;
}
```

Types defined in conjunction with UI components APIs
```TypeScript
type ResourceStr = string | Resource;
type Length = string | number | Resource;
type ResourceColor = Color | number | string | Resource;

class Color {
    static readonly White = "#ffffffff";
    static readonly Black = "#ff000000";
    static readonly Blue = "#ff0000ff";
    static readonly Brown = "#ffa52a2a";
    static readonly Gray = "#ff808080";
    static readonly Green = "#ff008000";
    static readonly Grey = "#ff808080";
    static readonly Orange = "#ffffa500";
    static readonly Pink = "#ffffc0cb";
    static readonly Red = "#ffff0000";
    static readonly Yellow = "#ffffff00";
    static readonly LightYellow = "#ffffffaa";
} 

interface Resource {
    readonly id: number;
    readonly type: number;
    readonly params?: any[];
    readonly bundleName: string;
    readonly moduleName: string;
}
```

An incomplete list of classes that can not be used in conjunction with variable decorators (because of their native implementation): `Boolean`, `Number`, `String`, `WeakMap`, `WeakSet`, `WeakRef`, `Promise`. Also `any` and `unknown` are not allowed.

### 3.2.2 TypeScript union types and `type` keyword

ArkUI supports TS union types, e.g. `@State textLabel : string | Resource = $r("Hello") || "Hello";` means `textLabel` can either take the return value of `$r` which is `Resource` or a `string` value. 

ArkUI, just as ArkTS, currently does not support TS intersection types.

ArkUI supports custom types defined by the TS `type` keyword, the same example as above using `type`: `type StringResource = string | Resource;` and the variable declaration use the custom type `@State textLabel : StringResource = $r("Hello") || "Hello";`

For further explanation please refer to TS documentation.

### 3.2.3 Provisions for `undefined` and `null`

ArkUI supports `undefined` and `null` as follows:
* `undefined` and `null` use must be made explicit in the type definition, e.g. `@State s : string | undefined = undefined` is ok, @State s : string = undefined` is not recommended and will create a type mismatch error is strict mode (see next section).
* `undefined` is supported in union types with simple types `number | string | boolean`, and with object types.
* `null` is supported only in union types that include at least one object type. This restriction is in line with how `null` is used in TS. Note also that `typeof null == 'object'`.

Note: TS supports marking class properties as  _optional_ using the `?` notation in declaration. It adds `undefined` automatically to their type, and dds a default initialization with `undefined`. ArkUI might add support later, and define the semantics of default initialization for each variable decorator.


### 3.2.4 Permissive type checking for API 10 vs. strict type checking in subsequent API versions

Up to API 10 ArkUI implements permissive type support similar to TypeScript `tsc` non-strict ("permissive") compilation support.  Permissive means:
- `undefined` or `null` value can be assigned even though the variable type declaration does not specify, e.g. `@State s : string = undefined` is not recommended but supported.
- assignment of a value of different type than specified is not recommended but supported, as long as the assigned value is of a permissable type for this decorator.
- assignment of value of illegal type might result in application termination with JS exception.

The permissive framework behaviour has been introduced for API 10 because several ArkUI system apps would otherwise fail with type mismatch error. 

From API 11 ArkUI will switch to strict typing. Applications are therefore strongly advised to declare variable types property, modify values only within the range of declared type. Pay special attention to these two scenario's:
* If a variable needs to accept 'undefined', you need to declare it: `@State arr : ClassA[] | undefined = undefined;`. 
* If values of multiple basic types, use TypeScript union type of two basic types, e.g. `@State textLabel : string | Resource = $r("Hello") || "Hello";` and if also `undefined` assignment needs to be allowed: `@State textLabel : string | Resource | undefined = $r("Hello") || "Hello";`


### 3.2.5 Variable types and observed changes for variables decorated with  `@State`, `@Provide`, `@Link`, `@Consume`, or `@Prop`

| permissable types | observed value changes | 
|-------------------------|------------------------|
| `number, string, boolean`, enum with `string` or `number` values | Assign a new value. |
| `Date`  | Assign a new `Date` object. <br/> Set a new `Date` value using methods `setFullYear`, `setMonth`, `setDate`, `setHours`, `setMinutes`, `setSeconds`, `setMilliseconds`, `setTime`, `setUTCFullYear`, `setUTCMonth`, `setUTCDate`, `setUTCHours`, `setUTCMinutes`, `setUTCSeconds`, `setUTCMilliseconds` |
| `Map<ArkUITypesNoUndefined, ArkUITypes>`  (added in release of API 10) | Assign new `Map` object. <br/> Modifications made to `Map` object internal storage by function `set`, `clear`, `delete` are observed. Note: `[]` operators must not be used as these do not modify the Map object's internal storage but treat the Map object like a regular Object. |
| `Set<ArkUITypesNoUndefined>` (added in release of API 10) | Assign a new `Set` object. <br/> Modifications made to `Set` object internal storage by function `add`, `clear`, `delete` are observed. Note: `[]` operators must not be used, same argument as for Map. |
| `Array<ArkUITypes>` or `[]` of permissable type | Assign new array. <br> Assign new array item value using `[]` operator. <br> Adding, removing, or updating array item with methods `copyWithin`, `fill`, `reverse`, `sort`, `splice`, `push`, `pop`, `shift`, `unshift`. |
| Instance of `SubscribableAbstract` | Assign a new `SubscribableAbstract` object. <br/> All object property changes notified by the `SubscribableAbstract` object |
| Application-defined ES6 class | Assign a new class instance. <br/> value assignment to object property for all those properties returned by `Object.keys(classObject)` Note: this requirement means member variables defined to have a `get` and `set` function are not observed. The background to this restriction is that the ArkUI framework uses ES6 internally to observe object property changes.  |
| Application defined ES6 class extending `Date`, `Map`, `Set`, or `Array` class | Assign a new class instance.  <br/> value assignment to object property of the extended class with the same restriction as mentioned for class objects. <br> `Date`, `Map`, `Set`, or `Array` altering functions as listed above. As of Aug'2023 the framework implementation supports classes extending from Map, Set, and Date only for the purpose of @Observed clss decoration. Classes can not have own properties or functions |
| Types used by ArkUI components:<br>`ResourceStr`, `Length`,  `ResourceColor `, `Color`| new value assignment |
| JS object (no functions), i.e. instance is `typeof obj=='object'`' other than those object types listed above, | Assign a new object.  <br/> value assignment to object property with the same restriction as mentioned for class objects. |
| union type of permissible type <br/>incl. union types with `undefined` or `null`, e.g. `string \| undefined` or `ClassA \| null` | Assign a new value. <br/>Type of current value determines what changes will be observed. |


### 3.2.6 Variable types and observed changes for variables decorated with  `@ObjectLink`:

| `@ObjectLink` permissible types | observed value changes | 
|--------------------------------|------------------------|
| Application-defined ES6 class decorated with `@Observed`. | Assign a new class instance. <br/> value assignment to object property for all those properties returned by `Object.keys(classObject)` Note: this requirement means member variables defined to have a `get` and `set` function are not observed. The background to this restriction is that the ArkUI framework uses ES6 internally to observe object property changes.  |
| `@Observed` decorated class extending from `Date`  | Same as general ES6 class and in addition: <br/> Set a new `Date` value using methods `setFullYear`, `setMonth`, `setDate`, `setHours`, `setMinutes`, `setSeconds`, `setMilliseconds`, `setTime`, `setUTCFullYear`, `setUTCMonth`, `setUTCDate`, `setUTCHours`, `setUTCMinutes`, `setUTCSeconds`, `setUTCMilliseconds` |
| `@Observed` decorated class extending from  `Map`  (API 10) | Same as general ES6 class and in addition: <br/> Modifications made to `Map` object internal storage by function `set`, `clear`, `delete` are observed. Note: `[]` operators must not be used as these do not modify the Map object's internal storage but treat the Map object like a regular Object. |
| `@Observed` decorated class extending from `Set` (API 10) | Same as general ES6 class and in addition: <br/> Modifications made to `Set` object internal storage by function `add`, `clear`, `delete` are observed. Note: `[]` operators must not be used, same argument as for Map. |
| `Observed` decorated class extending from `Array` | Same as general ES6 class and in addition: <br> Assign new array item value using `[]` operator. <br> Adding, removing, or updating array item with methods `copyWithin`, `fill`, `reverse`, `sort`, `splice`, `push`, `pop`, `shift`, `unshift`. |
| Instance of `SubscribableAbstract` (@Observed class decorator not needed) | Assign a new `SubscribableAbstract` object. <br/> All object property changes notified by the `SubscribableAbstract` object |
| union type of permissable type for @ObjectLink <br/>incl. union types with `undefined` or `null`, e.g. `ClassA \| undefined` or `ClassA \| null` | Assign a new value. <br/>Type of current value determines what changes will be observed. |

See definition of @ObjectLink for details.



#### 3.2.7 Variable types and observed changes for variables decorated with `@StorageLink`, `@StorageProp`, `@LocalStorageLink`, or `@LocalStorageProp`

| permissable type | observed value changes | 
|-------------------------|------------------------|
| `number, string, boolean, enum` | Assign a new value. |
| `Array` or `[]` of permissable type | Assign new array. <br> Assign new array item value using `[]` operator. <br> Adding, removing, or updating array item with methods `copyWithin`, `fill`, `reverse`, `sort`, `splice`, `push`, `pop`, `shift`, `unshift`. |
| Application-defined ES6 class | Assign a new class instance. <br/> value assignment to object property for all those properties returned by `Object.keys(classObject)` Note: this requirement means member variables defined to have a `get` and `set` function are not observed. The background to this restriction is that the ArkUI framework uses ES6 internally to observe object property changes.  |
| Application defined ES6 class extending `Array` class | Assign a new class instance.  <br/> value assignment to object property of the extended class with the same restriction as mentioned for class objects. <br> `Array` altering functions as listed above. |
| JS object, i.e. instance is `typeof obj=='object'` other than those object types listed above. No Date, set, Map, and no functions. | Assign a new object.  <br/> value assignment to object property with the same restriction as mentioned for class objects. |

`undefined` and `null` value and union types are not, yet, supported.

Future versions of ArkUI will complete the support for types currently only supported for aforementioned decorators.


### 3.2.8 Provisions for local variable initialization

When creating a custom component the order of initialization is 
1. Local initialization, if supplied
2. Initialization from the parent, if supplied
After both steps all types of variables must be initialized with the exception regular and `@BuilderParam` are allowed but not recommended to be left uninitialized. 

***Local initialization***:

| Decorator | local initialization |
|-----------|----------------------|
| regular   | Optional, highly recommended |
| @State | Mandatory |
| @Prop | Optional. NOTE: Parent initialization will override optional local initialization. |
| @Link | Not allowed |
| @ObjectLink | Not allowed |
| @Provide | Mandatory |
| @Consume | Not allowed |
| @BuilderParam | Optional |
| @StorageLink<br />, @StorageProp<br />, @LocalStorageLink<br />, @LocalStorageProp | Mandatory, the value is only used if property is not found from `AppStorage` / from `LocalStorage` |

### 3.2.9 Provisions for variable initialization from the parent

Decorated variables in the parent @Component can be used to initialize and subsequently also update a decorated variables in a child @Component. 

The following two tables summaries the rules for decorated variable initialization, source variable in the parent @Component (columns) and initialized variable in the child @Component (rows). Here it is assumed that both parent variable and child variable are of the same type. What is permissable depends on the decorators of both variables:

| **Variable in child component** | **regular, not decorated** | **@State** | **@Link** | **@Prop** | **@Provide** | **@Consume** | **@ObjectLink** | **@StorageLink** | **@StorageProp** | **@LocalStorageLink** | **@LocalStorageProp** |
|---------------------------------|----------------------------|------------|-----------|-----------|--------------|--------------|------------------|------------------|------------------|-----------------------|------------------------|
| **regular**                    | OK                         | OK         | OK        | OK        | -            | -            | OK               | OK               | -                | -                     | -                      |
| **@State**                     | OK                         | OK         | OK        | OK        | OK           | OK           | OK               | OK               | OK               | OK                    | OK                     |
| **@Link**                      | -                          | OK         | OK       | OK        | OK           | OK           | -                | OK                | OK               | OK                    | OK                     |
| **@Prop**                      | OK                         | OK         | OK        | OK        | OK           | OK           | OK               | OK               | OK               | OK                    | OK                     |
| **@Provide**                   | OK                         | OK         | OK        | OK        | OK           | OK           | OK               | OK               | OK               | OK                    | OK                     |
| **@Consume**                   | -                          | -          | -         | -         | -            | -            | -                | -                | -                | -                     | -                      |
| **@ObjectLink**  | -                          | OK(1)      | OK(1)    | OK(1)     | OK(1)        | OK(1)        | OK(1)            | OK(1)             | OK(1)            | OK(1)                | OK(1)                  |

| **Note**  | **Explanation**                                                               |
|-----------|-------------------------------------------------------------------------------|
| **(1)** | Object type only, no simple type supported. Use `@Prop` instead. Decorated variable in parent and `@ObjectLink` variable in the child component refer to the same object.  |


`@StorageLink, @StorageProp, @LocalStorageLink, @LocalStorageProp` are not allowed to be initialized from the parent component.

`@BuilderParam` variable in child component can optionally be initialized from the parent with either a `@Builder` function reference or with the value of another `@BuilderParam` variable.

Some of the child @Component decorated variables can be initialized with a source variable from parent @Component that is of different type, as long as the actual value will match the child variable type. This is valid for @State, @Link, @Prop and @Provide decorated variable in the parent @Component. What is permissible depends on the decorators of both variables:

| **Variable in Child component** | **obj.a simple** | **obj.obj2 object** | **arr[i] simple** | **arr[i] object** |
|---------------------------------|-----------|--------------|-------------------|-------------------|
| **regular**                    | OK        | OK           | OK                | OK                |
| **@State**                     | OK        | OK           | OK                | OK                |
| **@Link**                      | -         | -            | -                 | -                 |
| **@Prop**                      | OK        | OK           | OK                | OK                |
| **@Provide**                   | OK        | OK           | OK                | OK                |
| **@Consume**                   | -         | -            | -                 | -                 |
| **@ObjectLink**                | N/A       | OK(2)        | N/A               | OK(2)             |


| **Note**  | **Explanation**                                                               |
|-----------|-------------------------------------------------------------------------------|
| **(2)** | array item / inner object must be class object, class decorted with `@Observed` |


### 3.2.10 Provisions for variable update from the parent

When the source variable (columns) changes the child @Component (rows) might be updated and the value change might sync to the variable in the child @Component. Each decorator has own rules: no sync, one-way sync from parent to child or two-way sync:

| **Variable in Child component** | **regular, not decorated** | **@State**      | **@Link**       | **@Prop**       | **@Provide**    | **@Consume**    | **@ObjectLink** | **@StorageLink** | **@LocalStorageLink** |
|---------------------------------|----------------------------|-----------------|-----------------|-----------------|-----------------|-----------------|------------------|------------------|-----------------------|
| **regular**                    | -                          | -               | -               | -               | -               | -               | -                | -                | -                     |
| **@State**                     | -                          | -               | -               | -               | -               | -               | -                | -                | -                     |
| **@Link**                      | -                          | two-way         | two-way         | two-way         | two-way         | two-way         | -                | two-way          | two-way               |
| **@Prop**                      | -                          | parent -> child | parent -> child | parent -> child | parent -> child | parent -> child | OK                | parent -> child  | parent -> child       |
| **@Provide**                   | -                          | -               | -               | -               | -               | -               | -                | -                | -                     |
| **@Consume**                   | -                          | -               | -               | -               | -               | -               | -                | -                | -                     |
| **@ObjectLink**                | -                          | object ref (3)  | object ref (3)  | object ref (3)  | object ref (3)  | object ref (3)  | object ref (3)   | object ref (3)   | object ref (3)        |

| **Note**  | **Explanation**                                                               |
|-----------|-------------------------------------------------------------------------------|
| **(3)** | reference to same object (no copy is made): new object assignment syncs from parent to `@ObjectLink` variable in child component only <br> object property change syncs both ways. |

The rules for update when the source variable is of object or array type:

| **Variable in Child component** | **obj.a**       | **obj.obj2**    | **arr[i] simple** | **arr[i] object** |
|---------------------------------|-----------------|-----------------|-------------------|-------------------|
| **regular**                    | -               | -               | -                 | -                 |
| **@State**                     | -               | -               | -                 | -                 |
| **@Link**                      | -               | -               | -                 | -                 |
| **@Prop**                      | parent -> child | parent -> child | parent -> child   | parent -> child   |
| **@Provide**                   | -               | -               | -                 | -                 |
| **@Consume**                   | -               | -               | -                 | -                 |
| **@ObjectLink**                | N/A             | parent -> child | N/A               | parent -> child   |

A `@BuilderParam` variable in child component does not get updated from the parent.

## 3.3 Tutorial - application state in ArkUI

This tutorial offers newcomers a help with ArkUI to choose the most suited state variable decorator for their application.
This tutorial is not meant to replicate the specification of decorators. You should still read the specification chapters 
for each decorator to get full understanding of each decorator.

### 3.3.1 Model - View - View Model Pattern 

The application state, from which to render the UI, is often more complex. It often consists of arrays or objects, with nested objects inside and nested combinations of these. In this situation it is advisable to align the app architecture to the Model - View - View Model (MVVM, [see Wikipedia](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel) ) design pattern:

* View - the @Components in ArkUI
* Model - The model stores data and related logic. It represents data that is being transferred between controller components or any other related business logic. The structure of Model data is often pre-defined by the used database or cloud API.
* ViewModel - In ArkUI, the ViewModel gets stored in decorated variables of @Components, LocalStorage, and AppStorage, from which the @Components are rendered by executing their `build` function. ViewModel classes expose functions called from event handler functions and `@Watch` callback functions to mutate the data. These functions also sync back changes to the Model. The important point is that the structure of the ViewModel should always be designed to optimally support the construction and update of `@Components`. This is the motivation for separate Model and ViewModel.

Many issues with UI construction and update are caused by the fact that the ViewModel is not designed to optimally support @Component render. Or developers try to force fit @Component rendering to a Model, which has a pre-determined structure that is badly suited to support @Component render. For instance, if an application reads tables from an SQL database into its in-memory data Model, then, this this Model is almost guaranteed to be not suitable for rendering ArkUI components from it. Such application needs a separate View-Model:

```text
+------------------+          +------------------+           +------------------+
|                  |          |                  |   render  |                  |
|      Model       |  two-way |    ViewModel     |  -------> |    View (UI)     |
|                  | <------> | - decorated vars | <-------  | - exec build()   |
|                  |   sync   | - LocalStorage   |  event-   | - exec @Builder's|
|                  |          | - AppStorage     |  handlers |                  |
+------------------+          +------------------+           +------------------+
```

Following-on from above example involving an SQL database, the application should be designed to 
* use a Model that is optimized for efficient database operations
* use a ViewModel optimizes for efficient UI updates using ArkUI state management functionality
* deploy converters/adapters that can convert initially read Model data to create the initial ViewModel and subsequently update the ViewModel with Model updates or additional data read from the database. Converters/adapters that can convert ViewModel data to Model data are required for all scenarios where data gets added or modified by the UI. In many application scenarios on a subset of the data can be added or modified in the UI.

While this design requires some extra implementation effort up-front compared to force fitting the UI to the SQL database schema, an application developer can expect a clear payoff with simplified UI design and implementation and better UI performance.

### 3.3.2 Who should own a ViewModel root data item - `@State`, `@Provide`, `@LocalStorageLink/Prop`, or `@StorageLink/Prop`?

The ViewModel typically consists of multiple top-level data items. Who should own these and what variable decorator to choose?

`@State` and `@Provide` decorated variables as well as `LocalStorage` and `AppStorage` are all good to 'own' and share some top level ViewModel data item - which one to choose when? - The answer is it depends how far the state needs to be shared across custom components. Four options from smallest to largest scope of sharing: 
1. `@State` decorated @Component variable, 
2. `@Provide` decorated @Component variable, or
3. use LocalStorage and `@LocalStorageLink/Prop` decorated @Component variable, or
4. use AppStorage and `@StorageLink/Prop` decorated @Component variable.

We discuss these options in detail:

1. The `@State testNum` decorated variable can be shared with one or several child components, which components use either `@Prop`, `@Link`, or `@ObjectLink` to establish a one-way or two way sync.

The following example uses `@State` in `Parent` root component to own a ViewModel data item. It uses two-way sync with `@Link testNum` variables in `LinkChild` and `Sibling` sub-components. Furthermore, `LinkLinkChild @Link` creates a two-way sync with `@Link testNum` and `LinkPropChild` creates a one-way sync.

Hence, when the `@Link testNum`  in `LinkChild` changes, then the change will sync first to `Parent` and from there to `Sibling`. `@Link testNum` change also syncs to `LinkLinkChild`  and  `LinkPropChild`.

Worth noting to understand the difference of `@State` decorator and following options: 
- in order to share `@State testNum` with a child of child component (2nd level descendant) it needs to be passed to the child component and from there to the child of child.
- sharing is done by passing the variable as a parameter to the child constructor.

```TypeScript
@Component
struct LinkLinkChild {
  @Link @Watch("testNumChange") testNumGrand: number;

  testNumChange(propName:string ): void {
    console.log(`LinkLinkChild: testNumGrand value ${this.testNumGrand}`);
  }

  build() {
      Text(`LinkLinkChild: ${this.testNumGrand}`)
  }
}


@Component
struct PropLinkChild {
  @Prop @Watch("testNumChange") testNumGrand: number;

  testNumChange(propName:string ): void {
    console.log(`PropLinkChild: testNumGrand value ${this.testNumGrand}`);
  }

  build() {
      Text(`PropLinkChild: ${this.testNumGrand}`)
        .height(70)
        .backgroundColor(Color.Red)
        .onClick(() => {
          this.testNumGrand+=1;
        })
  }
}


@Component
struct Sibling {
  @Link @Watch("testNumChange") testNum: number;

  testNumChange(propName:string ): void {
    console.log(`Sibling: testNumChange value ${this.testNum}`);
  }

  build() {
      Text(`Sibling: ${this.testNum}`)
  }
}

@Component
struct LinkChild {
  @Link @Watch("testNumChange") testNum: number;

  testNumChange(propName:string ): void {
    console.log(`LinkChild: testNumChange value ${this.testNum}`);
  }

  build() {
    Column() {
      Button('incr testNum')
      .onClick(() => {
        console.log(`LinkChild: before value change value ${this.testNum}`);
        this.testNum=this.testNum+1
        console.log(`LinkChild: after value change value ${this.testNum}`);
      })
      Text(`LinkChild: ${this.testNum}`)
      LinkLinkChild({testNumGrand: this.$testNum})
      PropLinkChild({testNumGrand: this.testNum})
    }
    .height(200).width(200)
  }
} 


@Entry
@Component
struct Parent {
  @State @Watch("testNumChange1") testNum: number = 1;
  testNumChange1(propName : string ): void {
    console.log(`Parent: testNumChange value ${this.testNum}`)
  }

  build() {
    Column() {
      LinkChild({testNum: this.$testNum})
      Sibling({testNum: this.$testNum})
    }
  }
}
```

2. An `@Provide` component variable shares with any descendant component. That descendant component uses `@Consume` to create a two-way sync.  So `@Provide` - `@Consume` pattern is preferred over using `@State` - `@Link` - `@Link` from parent to child to child-of-child component. Both sync the same, but the former is less code to write. `@Provide` - `@Consume` is good for sharing state within a single page UI component tree.

The following is a modified version of above using `@Provide` and `@Consume` instead of `@State` and `@Link`. The visible conveniences of `@Consume` is that it `connects` with `@Provide` in an ancestor component without passing its reference in the component constructor call: `LinkChild({ /* empty */ })`. Except for the initialization `@Provide` works really the same as `@State`, and `Consume` works the same as `@Link`:

```TypeScript
@Component
struct LinkLinkChild {
  @Consume @Watch("testNumChange") testNum: number;

  testNumChange(propName:string ): void {
    console.log(`LinkLinkChild: testNum value ${this.testNum}`);
  }

  build() {
      Text(`LinkLinkChild: ${this.testNum}`)
  }
}


@Component
struct PropLinkChild {
  @Prop @Watch("testNumChange") testNumGrand: number;

  testNumChange(propName:string ): void {
    console.log(`PropLinkChild: testNumGrand value ${this.testNumGrand}`);
  }

  build() {
      Text(`PropLinkChild: ${this.testNumGrand}`)
        .height(70)
        .backgroundColor(Color.Red)
        .onClick(() => {
          this.testNumGrand+=1;
        })
  }
}


@Component
struct Sibling {
  @Consume @Watch("testNumChange") testNum: number;

  testNumChange(propName:string ): void {
    console.log(`Sibling: testNumChange value ${this.testNum}`);
  }

  build() {
      Text(`Sibling: ${this.testNum}`)
  }
}

@Component
struct LinkChild {
  @Consume @Watch("testNumChange") testNum: number;

  testNumChange(propName:string ): void {
    console.log(`LinkChild: testNumChange value ${this.testNum}`);
  }

  build() {
    Column() {
      Button('incr testNum')
      .onClick(() => {
        console.log(`LinkChild: before value change value ${this.testNum}`);
        this.testNum=this.testNum+1
        console.log(`LinkChild: after value change value ${this.testNum}`);
      })
      Text(`LinkChild: ${this.testNum}`)
      LinkLinkChild({ /* empty */ })
      PropLinkChild({testNumGrand: this.testNum})
    }
    .height(200).width(200)
  }
} 


@Entry
@Component
struct Parent {
  @Provide @Watch("testNumChange1") testNum: number = 1;
  testNumChange1(propName : string ): void {
    console.log(`Parent: testNumChange value ${this.testNum}`)
  }

  build() {
    Column() {
      LinkChild({ /* empty */ })
      Sibling({ /* empty */ })
    }
  }
}
```

3. `@LocalStorageLink` and `@LocalStorageProp` create a two-way or a one-way sync with a property in a `LocalStorage` instance.  You can think of a `LocalStorage` object as a Map of `@State` variables. The `LocalStorage` object can be shared across several pages of an ArkUI application. Therefore use `@LocalStorageLink`,  `@LocalStorageProp` and `LocalStorage` for sharing state across more than one page of your application.

Another modified example. It creates a LocalStorage instance, injects it into the root component by `@Entry(storage)`. Upon initializing the   `@LocalStorageLink` variable in the `Parent` component it creates the property in the `LocalStorage` instance and sets the specified initial value: ` @LocalStorageLink("testNum") testNum: number = 1;`.

`LocalStorage` can be though of as a Map of `@State` variables with property names acting as keys. An `@LocalStorageLink` decorated variable behaves like a `@Link` to that `@State` -like value in the map.

```TypeScript
@Component
struct LinkLinkChild {
    @LocalStorageLink("testNum") @Watch("testNumChange") testNum: number = 1;

  testNumChange(propName:string ): void {
    console.log(`LinkLinkChild: testNum value ${this.testNum}`);
  }

  build() {
      Text(`LinkLinkChild: ${this.testNum}`)
  }
}


@Component
struct PropLinkChild {
  @LocalStorageProp("testNum") @Watch("testNumChange") testNumGrand: number = 1;

  testNumChange(propName:string ): void {
    console.log(`PropLinkChild: testNumGrand value ${this.testNumGrand}`);
  }

  build() {
      Text(`PropLinkChild: ${this.testNumGrand}`)
        .height(70)
        .backgroundColor(Color.Red)
        .onClick(() => {
          this.testNumGrand+=1;
        })
  }
}


@Component
struct Sibling {
  @LocalStorageLink("testNum") @Watch("testNumChange") testNum: number = 1;

  testNumChange(propName:string ): void {
    console.log(`Sibling: testNumChange value ${this.testNum}`);
  }

  build() {
      Text(`Sibling: ${this.testNum}`)
  }
}

@Component
struct LinkChild {
  @LocalStorageLink("testNum") @Watch("testNumChange") testNum: number = 1;

  testNumChange(propName:string ): void {
    console.log(`LinkChild: testNumChange value ${this.testNum}`);
  }

  build() {
    Column() {
      Button('incr testNum')
      .onClick(() => {
        console.log(`LinkChild: before value change value ${this.testNum}`);
        this.testNum=this.testNum+1
        console.log(`LinkChild: after value change value ${this.testNum}`);
      })
      Text(`LinkChild: ${this.testNum}`)
      LinkLinkChild({ /* empty */ })
      PropLinkChild({ /* empty */ })
    }
    .height(200).width(200)
  }
} 

// create LocalStorage object to hold the data
const storage = new LocalStorage();

@Entry(storage)
@Component
struct Parent {
  @LocalStorageLink("testNum") @Watch("testNumChange1") testNum: number = 1;
  testNumChange1(propName : string ): void {
    console.log(`Parent: testNumChange value ${this.testNum}`)
  }

  build() {
    Column() {
      LinkChild({ /* empty */  })
      Sibling({ /* empty */ })
    }
  }
}
```

4. `@StorageLink` and `@StorageProp` create a two-way or a one-way sync with a property in a `AppStorage`. `AppStorage` is a singleton `LocalStorage` object, which the framework creates at app startup. Use `@StorageLink` and `@StorageProp` to share state across many of not all pages of an ArkUI application. The framework can be told to persist specific properties in `AppStorage` to file using the `PersistentStorage` API. Therefore, also use `@StorageLink` and `@StorageProp` for any data that should be restored when the user re-starts the application.

The modifications to the example to use `@StorageLink` and `@StorageProp`  in conjunction with `AppStorage` is left as an exercise to the reader. 


### 3.3.3 `@Prop` and `@ObjectLink` use for nested data structures

In all but the most simple cases a ViewModel data item is of complex type: e.g. an array of objects, or class object properties of type class object (nested class objects) or combination of these. 

This is where it is essential the @Component structure and the ViewModel structure to be designed together:

It is recommended to design a separate @Component for rendering each array or object. This means an array of objects, or an object whose property is of class object type itself required a @Component for rendering the outer array / object and another @Component for rendering the class object nested inside the array / the object. 

Why this recommendation? - Because `@Prop`, `@Link`, `@ObjectLink` decorated variables can only observe value changes on _one level_:
- for class objects: 
    * assignment of a new Object is an observed change: `this.obj = new ClassObj(...)`
    * Object property change is an observed change: `this.obj.a = new ClassA(...)`
    * 2nd level object property change is _not_ observed `this.obj.a.b = 47` is _not_ observed.
- for Array:
    * assignment of new Array is an observed change: `this.arr = [ ... ]`
    * deletion, insertion and replacement of array item is an observed change: Example of an array of objects: `this.arr[1] = new ClassA(); this.arr.pop(); this.arr.push(new ClassA(...)), this.arr.sort(...)` are all observed changes.
    * A 2nd level object property change is _not_ observed: `this.arr[1].b = 47` is _not_ observed.

Use of separate parent @Component and child @Component are recommended. - We talked already about the decorator to be used in the parent @Component, the one that 'owns' this ViewModel item: Use either `@State`, `@Provide`, `@LocalStorageLink/Prop` or `@StorageLink/Prop`. 

What decorator to chose for the class object nested inside? - Use `@ObjectLink` or in special cases use `@Prop`: `@ObjectLink` gets initialized with a reference to the object nested inside, its use is more efficient. `@Prop` gets initialized with a deep copy of the object nested inside to realize the one-way sync semantics. That's why `@ObjectLink` is recommended.

The `@ObjectLink` or `@Prop` variable stores the class object nested inside. This class must be decorated with the `@Observed` class decorator. If the class decoration is missing UI updates will not work. - What is the magic of `@Observed`? - `@Observed` creates a custom `constructor` function for the decorated class. This function creates an instance of the class and wraps it with an ES6 Proxy. This Proxy, which is implemented by the ArkUI framework, transparently intercepts all 'get' and 'set' operations on the wrapped object's properties. 'set' access to observe property value values. 'get' access to realize what UI component rendering read (use) the object, and thereby be smart about minimal scope UI updates.

How to use @Observed class decorator with an Array nested inside an outer Array or Object? -  So, the class of the object nested inside should be decorated with `@Observed`. What if the type nested inside is an Array? How to decorate an Array? - Well, in JavaScript Arrays are Objects. Here is the generic solution:
```TypeScript
@Observed class ObservedArray<T> extends Array<T> {
    constructor(args: any[]) {
        super(...args);
    }
    /* otherwise empty */
}
```
and use from outer ViewModel class like this:

```TypeScript
class Outer {
  innerArrayProp : ObservedArray<string>;
  ...
}
```

`@ObjectLink` realizes a two-way sync (because it gets initialized with a reference to the source object) and `@Prop` realized a one-way sync. Means when the `@Prop` gets initialized with a deep copy of the nested object. Why can a `@Prop` be assigned a new object, and `@ObjectLink` can not? - Assigning a new object to a `@Prop` variable simply overwrites the local copy. To realize two-way sync semantics an assignment to `@ObjectLink` would need to sync back and update the containing object property or array item in the parent component. That's something that can not be realized in TypeScript/JavaScript.


### 3.3.4 The difference between `@Prop` and `@ObjectLink` used for nested data structures

The following is an example of Array of class object. The key design Elements:
1. parent @Component `ViewB` renders the `@State arrA : Array<ClassA>`.
    * `@State` can observe assignments of new Array, Array item insertions, deletions and replacements.
2. child @Components `ViewA` render an object of `ClassA` each
3. class decorator `@Observed ClassA` in combination with `@ObjectLink a: ClassA;`
    * can observe changes of `ClassA` objects nested inside the Array. Consequences of not using 
        * the nested object property assignment in ViewB `this.arrA[Math.floor(this.arrA.length/2)].c = 10;` would not be observed and respective `ViewA` instance would not update.
        * There are two `ViewA` instances each rendering the first and the last item in the array. They render the same `ClassA` object. - Property assignment `this.a.c += 1;` in one `ViewA` instance would not cause other `ViewA` instance that renders the same ClassA object to update. (Note ) 

```TypeScript
let NextID  : number= 1;

@Observed 
class ClassA {
    public id : number;
    public c: number;

    constructor(c: number) {
        this.id = NextID++;
        this.c = c;
    }
}

@Component
struct ViewA {

  @ObjectLink a: ClassA;
  label : string = "ViewA1";

  build() {
     Row() {
        Button(`ViewA [${this.label}] this.a.c= ${this.a.c} +1`)
        .onClick(() => {
            // change object property
            this.a.c += 1;
        })
     }
  }
}

@Entry
@Component 
struct ViewB {

  @State arrA : ClassA[] = [ new ClassA(0), new ClassA(0) ];

  build() {
     Column() {
        ForEach (this.arrA,
            (item) => {
                ViewA({ label: `#${item.id}`, a: item })
            },
            (item) => item.id.toString()
        )

        Divider().height(10)

        if (this.arrA.length) {
            ViewA({ label: `ViewA this.arrA[first]`, a: this.arrA[0] })
            ViewA({ label: `ViewA this.arrA[last]`, a: this.arrA[this.arrA.length-1] })
        }

        Divider().height(10)

            Button(`ViewB: reset array`)
            .onClick(() => {
                // case: replace entire Array, will be observed by this.arrA
                this.arrA = [ new ClassA(0), new ClassA(0) ];
            })  
            Button(`array push`)
            .onClick(() => {
                // case insert new item to array, will be observed by this.arrA
                this.arrA.push(new ClassA(0))
            })
            Button(`array shift`)
            .onClick(() => {
                // case delete item from array, will be observed by this.arrA
                this.arrA.shift()
            })  
            Button(`ViewB: chg item property in middle`)
            .onClick(() => {
                // case  replace item in array, change will be observed by this.arrA
                this.arrA[Math.floor(this.arrA.length/2)] = new ClassA(11);
            })
            Button(`ViewB: chg item property in middle`)
            .onClick(() => {
                // case nested object propertyy will be observed by @ObjectLink a in ViewA
                this.arrA[Math.floor(this.arrA.length/2)].c = 10;
            })
        }
  }
}
```

A modification of above example. Replace `@ObjectLink` with `@Prop` in `ViewA`. - Think by yourself, how does the behaviour of the application change?

```TypeScript
@Component
struct ViewA {

  @Prop a: ClassA;
  label : string = "ViewA1";

  build() {
     Row() {
        Button(`ViewA [${this.label}] this.a.c= ${this.a.c} +1`)
        .onClick(() => {
            // change object property
            this.a.c += 1;
        })
     }
  }
}
```

Do you understand what changes? - Hint: Pressing all Buttons has the same affect as before except one Button. Which Button is it and what happens differently?

Solution: The Button in `ViewA` causes an UI update to the containing View only. If the rendered `ClassA` object is the first or last array item the property value change does not propagate to the other `ViewA` instance. The reason is the one way sync behaviour of `@Prop`. The modified `ClassA` object is a copy, it is not the object inside the parent's `@State arrA : Array<ClassA>` and it is not the same object as rendered by the other `ViewA` instance.

### 3.3.5 Application example for nested ViewModel

Only proceed to this example after understanding what has been told so far and after doing some own experimentation. The following  example dives deeper in application design for nested ViewModel. Especially we explain how one @Component can render an Object and additionally another Object nested inside.

Application requirements:

We want a page with a simple address book that shows the following info. Details are those of selected Contact or "Me".
Pressing "Edit" makes all data fields of "Details" editable. Only after clicking "Save" changes are used, and "Me" or Contact list item is updated, if the name was changed.

```text
|----------------------|
| "Me:"                |
| User's name, phone[0]|    just the user's own name + first phone, here, selectable
|----------------------|
| "Contacts:"          |
| Contact 1, phone[0]  |   list of contact names+first phone number, selectable
| Contact 2, phone[0]  |
| ...                  |
| Contact N,, phone[0] |
|----------------------|
| "Edit:"              |   either view selected contact details, editable
| Name                 |
| Street               |
| City, Zip            |
| Phone 1, Phone 2, .. |   zero or more phone numbers.
| <"Save changes">     |  apply changes only on "Save"
|----------------------|
```

The ViewModel needs to include:
- me : stores a Person
- contacts - stores a list of Person
* selected : reference to Person
* detailsMode - takes one of three values: showNone, readMode or editMode
- Person (class)
    * name : string
    * address : Address 
    * phones: list of Phone
  - Address (class)
    * street : string
    * zip : number
    * city : string
- Phone : string following international number phone format


A couple of noteworthy points about this example:
1. `AddressBookView` should update whenever either `me` object or `contacts` array update. If we used `@Link addrBook : AddressBook` then change of `this.addrBook.me.name` would be a 2nd level object property change and not be observed. Therefore, we use `@ObjectLink me : Person`, and `@ObjectLink contacts :  ObservedArray<Person>` and these changes will be observed.
2. PersonView needs to update when person name (`Person.name`) or first phone number (`Person.phones[0]`) updates. Changes on two different levels, an `@Link person : Person` would not detect the change of the phone number. To circumvent this View update problem we use two `@ObjectLink person : Person` and `@ObjectLink phone : ObservedArray<string>;` variables. Note how these are initialized and update from the parent. `@ObjectLink` must be class or array type and it must be initialized with an array item or object property of decorated variable in the parent. 
5. `PersonEditView` exploits the one-way sync of `@Prop` to obtain a local copy of the `Person` object. This local copy receives all the edit changes on TextInput.onChange events. 
6. On user selecting the "Save Change" option all the changes are copied to the two-way sync `@Link refToPerson`. Note how the changes are made: 
    * Doing like this `this.selectedPerson = new Person (this.name, this.address, this.phones)` would _not update_ the referenced Person object. 
    * Also doing like this is incorrect: `this.selectedPerson.address = this.address`. This would assign the `@Prop addreess` object. Subsequent local edits would be applied before being saved.
    * The same arguments why not do this `this.selectedPerson.phones = new ObservedArray<string>(this.phones)`. This would assign `@Prop phones` array to `this.selectedPerson.phones`. `ObservedArray` constructor does not copy the items of the input Array.

```TypeScript
// ViewModel classes ---------------------------

let nextId = 0;

@Observed class ObservedArray<T> extends Array<T> {
    constructor(args?: any[]) {
        console.log(`ObservedArray: ${JSON.stringify(args)} `)
        if (Array.isArray(args)) {
            super( ...args);
        } else {
            super(args)

        }
    }
}

@Observed class Address {
    street : string;
    zip: number;
    city : string;

    constructor(street : string,
        zip: number,
        city : string) {
            this.street = street;
            this.zip = zip;
            this.city = city;
        }
}

@Observed class Person {
    id_ : string;
    name: string;
    address : Address;
    phones: ObservedArray<string>;

    constructor(name: string,
            street : string,
            zip: number,
            city : string,
            phones: string[]) {
        this.id_ = `${nextId}`;
        nextId++;
        this.name = name;
        this.address = new Address(street, zip, city);
        this.phones = new ObservedArray<string>(phones);
    }
}

class AddressBook {
    me : Person;
    contacts : ObservedArray<Person>;

    constructor(me : Person, contacts : Person[]) {
        this.me = me;
        this.contacts = new ObservedArray<Person>(contacts);
    }
}


// @Components -----------------------

// renders the name of a Person object and the first number in the phones ObservedArray<string> 
// For also the phone number to update we need two @ObjectLink here, person and phones, 
// can not use this.person.phones. Changes of inner Array not observed.
// onClick updates selectedPerson also in AddressBookView, PersonEditView
@Component
struct PersonView {

    @ObjectLink person : Person;
    @ObjectLink phones :  ObservedArray<string>;
    
    @Link selectedPerson : Person;

    build() {
        Flex({ direction: FlexDirection.Row, justifyContent: FlexAlign.SpaceBetween }) {
          Text(this.person.name)
          if (this.phones.length) {
            Text(this.phones[0])
          }
        }
        .height(55)
        .backgroundColor(this.selectedPerson.name == this.person.name ? "#ffa0a0" : "#ffffff")
        .onClick(() => {
            this.selectedPerson = this.person;
        })
    }
}

// Renders all details
// @Prop get initialized from parent AddressBookView, TextInput onChange modifies local copies only
// on "Save Changes" copy all data from @Prop to @ObjectLink, syncs to selectedPerson in other @Components.
@Component
struct PersonEditView {

    @Consume addrBook : AddressBook;

    /* Person object and sub-objects owned by the parent Component */
    @Link selectedPerson: Person;

    /* editing on local copy until save is handled */
    @Prop name: string;
    @Prop address : Address;
    @Prop phones : ObservedArray<string>;

    selectedPersonIndex() : number {
        return this.addrBook.contacts.findIndex((person) => person.id_ == this.selectedPerson.id_);
    }

    build() {
        Column() {
            TextInput({ text: this.name})
                .onChange((value) => {
                    this.name = value;
                  })
            TextInput({text: this.address.street})
                .onChange((value) => {
                    this.address.street = value;
                })

            TextInput({text: this.address.city})
                .onChange((value) => {
                    this.address.city = value;
                })

            TextInput({text: this.address.zip.toString()})
                .onChange((value) => {
                    const result = parseInt(value);
                    this.address.zip= isNaN(result) ? 0 : result;
                })

            if(this.phones.length>0) {
                ForEach(this.phones,
                        (phone, index) => {
                            TextInput({text: phone})
                                .width(150)
                                .onChange((value) => {
                                    console.log(`${index}. ${value} value has changed`)
                                    this.phones[index] = value;
                                })
                        },
                        (phone, index) => `${index}-${phone}`
                ) 
            }

            Flex({ direction: FlexDirection.Row, justifyContent: FlexAlign.SpaceBetween }) {
                Text("Save Changes")
                    .onClick(() => {
                        // copy values from local copy to the provided ref to Person object owned by 
                        // parent Component. Avoid creating new Objects, modify properties of existing
                        this.selectedPerson.name = this.name;
                        this.selectedPerson.address.street = this.address.street
                        this.selectedPerson.address.city   =  this.address.city
                        this.selectedPerson.address.zip    = this.address.zip
                        this.phones.forEach((phone : string, index : number) => { this.selectedPerson.phones[index] = phone } );
                    })
                if (this.selectedPersonIndex()!=-1) {
                    Text("Delete Contact")
                        .onClick(() => {
                            let index = this.selectedPersonIndex();
                            console.log(`delete contact at index ${index}`);

                            // delete found comtact
                            this.addrBook.contacts.splice(index, 1);

                            // determin new selectedPerson
                            index = (index < this.addrBook.contacts.length) ? index : index-1;

                            // if no contact left, set me as selectedPerson
                            this.selectedPerson = (index>=0) ? this.addrBook.contacts[index] : this.addrBook.me;
                        })
                }
            }

        }
    }
}


@Component
struct AddressBookView {

    @ObjectLink me : Person;
    @ObjectLink contacts : ObservedArray<Person>;
    @State selectedPerson: Person = undefined;

    aboutToAppear() {
        this.selectedPerson = this.me;
    }
    
    build() {
        Flex({ direction: FlexDirection.Column, justifyContent: FlexAlign.Start}) {
            Text("Me:")
            PersonView({person: this.me, phones: this.me.phones, selectedPerson: this.$selectedPerson})

            Divider().height(8)

            Flex({ direction: FlexDirection.Row, justifyContent: FlexAlign.SpaceBetween }) {
                Text("Contacts:")
                Text("Add")
                    .onClick(() => {
                        this.selectedPerson = new Person ("", "", 0, "", [ "+358"]);
                        this.contacts.push(this.selectedPerson);
                    })
            }
                .height(50)

            ForEach(this.contacts,
                contact => {
                    PersonView({person: contact, phones: contact.phones, selectedPerson: this.$selectedPerson})
                },
                contact => contact.id_
            )

            Divider().height(8)

            Text("Edit:")
            PersonEditView({ selectedPerson: this.$selectedPerson, name: this.selectedPerson.name, address: this.selectedPerson.address, phones: this.selectedPerson.phones })
        }
            .borderStyle(BorderStyle.Solid).borderWidth(5).borderColor(0xAFEEEE).borderRadius(5)
    }
}
 
@Entry
@Component
struct PageEntry {

    @Provide addrBook : AddressBook = new AddressBook(
        new Person("Guido Grassel", "Itamerenkatu 9", 180, "Helsinki", [ "+358441234567", "+35891234567", "+49621234567889"]),
        [
            new Person("Oleg Beletski", "Itamerenkatu 9", 180, "Helsinki", [ "+358449876543", "+3589456789"]),
            new Person("Sarath Singapati", "Itamerenkatu 9", 180, "Helsinki", [ "+358509876543", "+358910101010"]),
            new Person("Vidhya Pria Arunkumar", "Itamerenkatu 9", 180, "Helsinki", [ "+358400908070", "+35894445555"]),
        ]);

    build() {
        AddressBookView({ me: this.addrBook.me, contacts: this.addrBook.contacts, selectedPerson: this.addrBook.me})
    }
}
```


### 3.3.6 Example for the difference between `@Prop` shallow copy and deep copy

`@Prop` needs to make a copy of an object provided by its source during initialisation and sub-subsequent sync. 
API release 9 uses shallow copy for copying objects. Starting with API 10 the framework uses deep copy. There is 
plenty of documentation on shallow and deep copy of JS objects available. Readers unfamiliar with these terms are
advised to familiarise themselves. The purpose of this section is to illustrate the implications on ArkUI 
state management behaviour.

To experiment with API 9 versus API 10 edit the property `"targetAPIVersion"` in the HAP `module.json` file.

API 9:

Changing `ClassA` properties or assigning a new `ClassA` object in component `PropClassAArray` 
(middle part of the screen), causes update also to the `@ObjectLink` of component `ObjectLinkClassA` created 
from `stateClassAArray` items (bottom part of the screen).

This updates happens because shallow copy does not copy the `ClassA` objects inside the `PropClassAArray` 
`@Prop objArray` array. It only copies the array of references to `ClassA` objects. Hence, the change is made on the 
same object as said `@ObjectLink`.

API 10:

When running with API 10, above changes inside `PropClassAArray` do _not update_ the `@ObjectLink` of 
component `ObjectLinkClassA`. The reason is that the `ClassA` objects inside `@Prop objArray` array
and the objects of the parent `stateClassAArray` are different. They are different because deep copy 
has copied them when initialising the `PropClassAArray` component `@Prop objArray` array.

```TypeScript
let nextId = 0;

@Observed 
class ClassA  {
    id : number;
    a : number;
    constructor(a : number = 0) {
        this.id = nextId++;
        this.a = a;
    }
}

@Component
struct PropClassA {
    @Prop obj : ClassA = new ClassA();

    build() {
        Column() {
            Text(`PropClassA: obj: ${this.obj.a}`)
            .onClick(() => {
                this.obj.a += 1;
                stateMgmtConsole.debug(`PropClassA onClick obj changed to  ${this.obj.a}`)
            })
        }.border({width: 3, color: Color.Red})
    }
}

@Component
struct ObjectLinkClassA {

    @ObjectLink obj : ClassA;

    build() {
        Row() {
            Text(`ObjectLink: obj: ${this.obj.a}`)
            .height(100)
            .onClick(() => {
                this.obj.a +=1;
                stateMgmtConsole.debug(`ObjectLink onClick ClassA property changed to  ${this.obj.a}`)
            })
        }.border({width: 3, color: Color.Red})
    }
}

@Component
struct PropClassAArray {

    @Prop objArray : Array<ClassA> = [];

    build() {
        Column() {
            Text(`green box: @Prop : Array<ObjectClassA> item [0] + [1]`)
            Row() {
                ObjectLinkClassA({ obj:  this.objArray[0] })
                Text("[0] Assign new ClassA")
                    .height(100)
                    .onClick(() => {
                        this.objArray[0] = new ClassA();
                        stateMgmtConsole.debug(`PropClassAArray[0] onClick ClassA object assign ${this.objArray[0].a}`)
                    })
                    Text("Change ClassA property")
                            .height(100)
                            .onClick(() => {
                                this.objArray[0].a += 1;
                                stateMgmtConsole.debug(`PropClassAArray[1] onClick ClassA property change  ${this.objArray[1].a}`)
                            })
            }
            Row() {
                ObjectLinkClassA({ obj:  this.objArray[1] })
                Text("[0] Assign new ClassA")
                    .height(100)
                    .onClick(() => {
                        this.objArray[1] = new ClassA();
                        stateMgmtConsole.debug(`PropClassAArray[1] onClick ClassA object assign ${this.objArray[1].a}`)
                    })
                    Text("Change ClassA property")
                            .height(100)
                            .onClick(() => {
                                this.objArray[1].a += 1;
                                stateMgmtConsole.debug(`PropClassAArray[1] onClick ClassA property change  ${this.objArray[1].a}`)
                            })
            }
        }.border({width: 3, color: Color.Green})
    }
}

@Entry
@Component
struct StateClassAArray {

    @State stateClassAArray : Array<ClassA> = [ new ClassA(), new ClassA() ];
    @State stateClassA : ClassA = new ClassA();

    build() {
        Column() {
            Column() {
                Text(`StateClassAArray: @State ClassA: ${this.stateClassA.a}`)
                .height(65)
                .onClick(() => {
                    this.stateClassA.a += 1;
                    stateMgmtConsole.debug(`StateClassAArray onClick stateClassA changed to  ${this.stateClassA.a}`)
                })
                ObjectLinkClassA({ obj: this.stateClassA })
                PropClassA({ obj: this.stateClassA })
            }
                .border({width: 3, color: Color.Yellow})

            Column() {
                Text("Red box: @ObjectLink from @State array item[0] + item [1]")
                Row() {
                    ObjectLinkClassA({obj : this.stateClassAArray[0] })
                    Text("Assign new ClassA")
                            .height(100)
                            .onClick(() => {
                                this.stateClassAArray[0] = new ClassA();
                                stateMgmtConsole.debug(`StateClassAArray[0] onClick ClassA object assign ${this.stateClassAArray[0].a}`)
                            })
                    Text("Change ClassA property")
                            .height(100)
                            .onClick(() => {
                                this.stateClassAArray[0].a += 1;
                                stateMgmtConsole.debug(`StateClassAArray onClick stateClassAArray[0] changed to  ${this.stateClassAArray[0].a}`)
                            })
                }
                Row() {
                    ObjectLinkClassA({obj : this.stateClassAArray[1] })
                    Text("Assign new ClassA")
                            .height(100)
                            .onClick(() => {
                                this.stateClassAArray[1] = new ClassA();
                                stateMgmtConsole.debug(`StateClassAArray[1] onClick ClassA object assign ${this.stateClassAArray[1].a}`)
                            })
                    Text("Change ClassA property")
                            .height(100)
                            .onClick(() => {
                                this.stateClassAArray[1].a += 1;
                                stateMgmtConsole.debug(`StateClassAArray onClick stateClassAArray[1] changed to  ${this.stateClassAArray[1].a}`)
                            })
                }

                Divider().height(5)

                PropClassAArray({ objArray: this.stateClassAArray })

            }
                .border({width: 3, color: Color.Blue})
        }
    }
}
```

## 3.4 Good and Bad Practices in State Management

The aim of this section, is to empower app developers to improve the quality of their applications, particularly in regards to effective state management. This resource provides a comprehensive list of common scenarios and potential pitfalls faced by developers, along with actionable solutions. Furthermore, poorly-designed source code is contrasted with well-crafted examples, accompanied by step-by-step instructions on how to remedy any issues.

### 3.4.1 The basics

Correct the mistakes in the following example.
```TypeScript
class ClassA {
  public c : number = 0;
  constructor (c : number) {
    this.c = c;
  }
}

@Component
struct LinkChild {
  @Link testNum: number;

  build() {
    Text(`LinkChild testNum ${this.testNum}`)
  }
}

@Component
struct ObjectLinkChild {
  @ObjectLink testNum: ClassA;

  build() {
    Text(`ObjectLinkChild testNum ${this.testNum.c}`)
      .onClick(() => {
        this.testNum = new ClassA(47);
      })
  }
}


@Component
struct PropChild1 {
  @Prop testNum: ClassA = new ClassA(1);

  build() {
    Text(`PropChild1 testNum ${this.testNum.c}`)
      .onClick(() => {
        this.testNum = new ClassA(48);
      })
  }
}

@Component
struct PropChild2 {
  @Prop testNum: ClassA;

  build() {
    Text(`PropChild2 testNum ${this.testNum.c}`)
        .onClick(() => {
          this.testNum.c += 1;
      })
}
}

@Component
struct PropChild3 {
  @Prop testNum: ClassA;

  build() {
    Text(`PropChild3 testNum ${this.testNum.c}`)
  }
}

@Entry
@Component
struct Parent {
  @State @Watch("testNumChange1") testNum: ClassA = new ClassA(1);

  testNumChange1(propName : string ): void {
    console.log(`Parent: testNumChange value ${this.testNum.c}`)
  }

  build() {
    Column() {
      Text(`Parent testNum ${this.testNum.c}`)
        .onClick(() => {
          this.testNum.c += 1;
        })

      LinkChild({testNum: this.testNum.a})
      ObjectLinkChild({testNum: this.testNum})
      PropChild1()
      PropChild2()
      PropChild3({testNum: this.testNum})
    }
  }
}
```


Several mistakes:
1. `LinkChild`: Invalid `@Link testNum: number` and init from parent `LinkChild({testNum: this.testNum.a})`. @Link source must be a decorated variable, here `@Link testNum: ClassA` and init from parent `LinkChild({testNum: this.testNum})`.
2. `PropChild2` has no local initialisation, it is mandatory to init from Parent `PropChild1({testNum: this.testNum})`.
3. `PropChild3` does not change `@Prop testNum`. @Prop has the overhead of variable value copy. No need here. Use @ObjectLink instead. It is more lightweight than `@Link` and especially `@Prop`.
4. click handler in `ObjectLinkChild` attempts an assignment `this.testNum = new ClassA(47);`. `@ObjectLink variables can not be assigned a new value. Object property changes are allowed.
5. `@Observed` class decorator for `ClassA` is strictly speaking not needed here because we init the @ObjectLink testNum from an @State variable. It would be needed if an Object nested inside would be used for initialisation.
 

 The corrected example:

 ```TypeScript
 @ObjectLink
class ClassA {
  public c : number = 0;
  constructor (c : number) {
    this.c = c;
  }
}

@Component
struct LinkChild {
  @Link testNum: ClassA;

  build() {
    Text(`LinkChild testNum ${this.testNum?.c}`)
  }
}

@Component
struct ObjectLinkChild {
  @ObjectLink testNum: ClassA;

  build() {
    Text(`ObjectLinkChild testNum ${this.testNum?.c}`)
      .onClick(() => {
        this.testNum.c += 1;
      })
  }
}


@Component
struct PropChild1 {
  @Prop testNum: ClassA = new ClassA(1);

  build() {
    Text(`PropChild1 testNum ${this.testNum?.c}`)
      .onClick(() => {
        this.testNum = new ClassA(48);
      })
  }
}

@Component
struct PropChild2 {
  @Prop testNum: ClassA;

  build() {
    Text(`PropChild2 testNum ${this.testNum?.c}`)
        .onClick(() => {
          this.testNum.c += 1;
      })
}
}

@Component
struct ObjectLinkChild3 {
  @ObjectLink testNum: ClassA;

  build() {
    Text(`ObjectLinkChild3 testNum ${this.testNum.c}`)
  }
}

@Entry
@Component
struct Parent {
  @State @Watch("testNumChange1") testNum: ClassA = new ClassA(1);

  testNumChange1(propName : string ): void {
    console.log(`Parent: testNumChange value ${this.testNum?.c}`)
  }

  build() {
    Column() {
    Text(`Parent testNum ${this.testNum?.c}`)
        .onClick(() => {
          this.testNum.c += 1;
      })

      LinkChild({testNum: this.testNum})
      ObjectLinkChild({testNum: this.testNum})
      PropChild1()
      PropChild2({testNum: this.testNum})
      ObjectLinkChild3({testNum: this.testNum})
    }
  }
}
 ```



### 3.4.2 Miss Nested Object Property Changes Case 1

Some UI updates do not work in this example, can you find the mistake?

```TypeScript
class ClassA {
  a: number;
  constructor(a: number) {
    this.a = a;
  }
  getA() : number {
    return this.a; }
  setA( a: number ) : void {
    this.a = a; }
}

class ClassC {
  c: number;
  constructor(c: number) {
    this.c = c;
  }
  getC() : number {
    return this.c; }
  setC(c : number) : void {
    this.c = c; }
}

class ClassB extends ClassA {
  b: number = 47;
  c: ClassC;

  constructor(a: number, b: number, c: number) {
    super(a);
    this.b = b;
    this.c = new ClassC(c);
  }

  getB() : number {
    return this.b; }
  setB(b : number) : void {
    this.b = b; }

  getC() : number {
    return this.c.getC(); }
  setC(c : number) : void {
    return this.c.setC(c); }
}


@Entry
@Component
struct MyView {

    @State b : ClassB = new ClassB(10, 20, 30);

    build() {
        Column({space:10}) {
            Text(`a: ${this.b.a}`)
             Button("Change ClassA.a")
            .onClick(() => {
                this.b.a +=1;
            })

            Text(`b: ${this.b.b}`)
            Button("Change ClassB.b")
            .onClick(() => {
                this.b.b += 1;
            })

            Text(`c: ${this.b.c.c}`)
            Button("Change ClassB.ClassC.c")
            .onClick(() => {
                this.b.c.c += 1;
            })
        }
     }
}
```

The issue: `Text('c: ${this.b.c.c}')` does not update.
 
 Explanation: `@State b : ClassB` can observe changes of properties `this.b` object only:  `this.b.a`, `this.b.b`, and `this.b.c` properties. But `@State b` can not observe changes of _nested_ object `this.b.c`: Therefore a change of property `this.b.c.c` (property `c` of nested object `c : ClassC` ) is not updated.

To observe the nested changes of `ClassC` and to update the UI accordingly, the following design changes are needed
- construct a sub-component for rendering instances of `ClassC`. 
- this sub-component may either use `@ObjectLink c : ClassC` or `@Prop c : ClassC`. `@ObjectLink` is to be used unless the sub-view needs to make _local_ modifications to its `ClassC` object.
- nested `ClassC` must be decorated with `@Observed`. The effect of the class decorator is that when creating the `ClassC` object inside the `ClassB` (here `new ClassB(10, 20, 30)` ) it gets wrapped inside a ES6 Proxy. That Proxy will notify the ClassC property change (here: `this.b.c.c += 1`) to the `@ObjectLink`.


``` Typescript
class ClassA {
  a: number;
  constructor(a: number) {
    this.a = a;
  }
  getA() : number {
    return this.a; }
  setA( a: number ) : void {
    this.a = a; }
}

@Observed        // class decorator added
class ClassC {
  c: number;
  constructor(c: number) {
    this.c = c;
  }
  getC() : number {
    return this.c; }
  setC(c : number) : void {
    this.c = c; }
}

class ClassB extends ClassA {
  b: number = 47;
  c: ClassC;

  constructor(a: number, b: number, c: number) {
    super(a);
    this.b = b;
    this.c = new ClassC(c);
  }

  getB() : number {
    return this.b; }
  setB(b : number) : void {
    this.b = b; }
  getC() : number {
    return this.c.getC(); }
  setC(c : number) : void {
    return this.c.setC(c); }
}

@Component            // own @Component for rendering ClassC added
struct ViewClassC {

    @ObjectLink c : ClassC;
    build() {
        Column({space:10}) {
            Text(`c: ${this.c.getC()}`)
            Button("Change C")
                .onClick(() => {
                    this.c.setC(this.c.getC()+1);
                })
        }
    }
}

@Entry
@Component
struct MyView {

    @State b : ClassB = new ClassB(10, 20, 30);

    build() {
        Column({space:10}) {
            Text(`a: ${this.b.a}`)
             Button("Change ClassA.a")
            .onClick(() => {
                this.b.a +=1;
            })

            Text(`b: ${this.b.b}`)
            Button("Change ClassB.b")
            .onClick(() => {
                this.b.b += 1;
            })

            ViewClassC({c: this.b.c})   // replacement for Text(`c: ${this.b.c.c}`)
            Button("Change ClassB.ClassC.c")
            .onClick(() => {
                this.b.c.c += 1;
            })
        }
     }
}
```



### 3.4.3 Missed Nested Object Property Updates Case 2

Taking the lesson from the previous case we create a sub-component with a `@ObjectLink` decorated variable for rendering a nested view model object, and we decorate the class with `@Observed`. However, a particular UI update still does not work. - Can you find it in the following code?

``` Typescript
let nextId = 1;

@Observed
class SubCounter {
  counter : number;

  incrSubCounter(c: number) {
    this.counter = this.counter+c;
  }

  setSubCounter(c : number) {
    this.counter = c;
  }

  constructor(c : number) {
    this.counter = c;
  }
}

@Observed
class Counter {
  id : number;
  counter : number;
  subCounter : SubCounter;

  incrCounter() {
    this.counter++;
  }

  incrSubCounter(c: number) {
    this.subCounter.incrSubCounter(c);
  }

  setSubCounter(c: number) : void {
    this.subCounter.setSubCounter(c);
  }

  constructor(c : number) {
    this.id = nextId++;
    this.counter = c;
    this.subCounter = new SubCounter(c);
  }
}

@Component
struct CounterComp {

  @ObjectLink value: Counter;

   build() {
    Column(space: 10) {
      Text(`${this.value.counter}`)
        .fontSize(25)
        .onClick(()=>{
          this.value.incrCounter();
        })
        Text(`${this.value.subCounter.counter}`)
        .onClick(()=>{
            this.value.incrSubCounter(1);
        })
        Divider().height(2)
    }
  }
}

@Entry
@Component
struct ParentComp {
  @State counter: Counter[] = [ new Counter(1), new Counter(2),  new Counter(3) ];

  build() {
    Row() {
      Column() {
        CounterComp({value: this.counter[0]})
        CounterComp({value: this.counter[1]})
        CounterComp({value: this.counter[2]})

        Divider().height(5)

        ForEach(this.counter,
          item => {
            CounterComp({value: item})
          },
          item => item.id.toString()
        )

        Divider().height(5)

        Text('Parent: reset entire counter')
        .fontSize(20).height(50)
        .onClick(()=>{
          this.counter = [ new Counter(1), new Counter(2), new Counter(3) ];
        })
        Text('Parent: incr counter[0].counter')
        .fontSize(20).height(50)
        .onClick(()=>{
          // 1st click handler
          this.counter[0].incrCounter();
          this.counter[0].incrSubCounter(10);  // This works to increment the SubCounter value by 10
        })

        Text('Parent: set.counter to 10')
        .fontSize(20).height(50)
        .onClick(()=>{
          this.counter[0].setSubCounter(10); // This DOES NOT work to set the SubCounter value by 10
        })
      }
    }
  }
}
```

In the example given above, check the Text click event handlers of the `ParentComp`. With 'onClick' of `Text('Parent: incr counter[0].counter')`, the `this.counter[0].incrSubCounter(10)` call, which is expected to increase the value of the `SubCounter` class by 10, works as expected. However, the `this.counter[0].setSubCounter(10)` call in 'onClick' of `Text('Parent: set.counter to 10')`, which is supposed to reset the counter value of the `SubCounter` class by 10, does not work.

Both `incrSubCounter` and `setSubCounter` are functions of the same `SubCounter` nested class. Calling `incrSubCounter` in the first click handler function updates the UI correctly. Calling `setSubCounter` in the second click handler function does not update the UI, i.e. it does not update `Text('${this.value.subCounter.counter}')`. Why does the Text update work in one case, and why does it not work in the other case?

Explanation:

Neither `incrSubCounter` nor `setSubCounter` can trigger an update of `Text('${this.value.subCounter.counter}')` because the `@ObjectLink value : Counter` does not observe property changes of the nested `this.value.subCounter` `SubCounter` object. We learn this in case #1. 
However, the call `this.counter[0].incrCounter()` in the 1st click handler marks `this.value` of having changed. This is what triggers `Text('${this.value.subCounter.counter}')` to update.

Try by removing the line `this.counter[0].incrCounter()` from the 1st click handler.

The solution:

Now to update the value in the `SubCounter` class directly, such that the `this.counter[0].setSubCounter(10)` operation works, you can follow the approach described below.

The important change is
```TypeScript 
  @ObjectLink value: Counter;
  @ObjectLink subValue: SubCounter;
```
It causes both the `Counter` as well as the nested `SubCounter` object to be observed. Therefore, UI updates work as expected. UI updates also continue to work when `this.counter[0].incrCounter()` is removed.

This trick can be used to achieve 'two levels' of observation, outer object and nested object observation. This can only be when using `@ObjectLink` decorator not with `@Prop`, because `@Prop` copies the object. See the next case.

``` Typescript
let nextId = 1;

@Observed
class SubCounter {
  counter : number;

  incrSubCounter(c: number) {
    this.counter = this.counter+c;
  }

  setSubCounter(c : number) {
    this.counter = c;
  }

  constructor(c : number) {
    this.counter = c;
  }
}


@Observed
class Counter {
  id : number;
  counter : number;
  subCounter : SubCounter;

  incrCounter() {
    this.counter++;
  }

  incrSubCounter(c: number) {
    this.subCounter.incrSubCounter(c);
  }

  setSubCounter(c: number) : void {
    this.subCounter.setSubCounter(c);
  }

  constructor(c : number) {
    this.id = nextId++;
    this.counter = c;
    this.subCounter = new SubCounter(c);
  }
}

@Component
struct CounterComp {

  @ObjectLink value: Counter;
  @ObjectLink subValue: SubCounter;

   build() {
    Column(space: 10) {
      Text(`${this.value.counter}`)
        .fontSize(25)
        .onClick(()=>{
          this.value.incrCounter();
        })
        Text(`${this.subValue.counter}`)
        .onClick(()=>{
            this.subValue.incrSubCounter(1);
        })
        Divider().height(2)
    }
  }
}

@Entry
@Component
struct ParentComp {
  @State counter: Counter[] = [ new Counter(1), new Counter(2),  new Counter(3) ];

  build() {
    Row() {
      Column() {
        CounterComp({value: this.counter[0], subValue: this.counter[0].subCounter})
        CounterComp({value: this.counter[1], subValue: this.counter[1].subCounter})
        CounterComp({value: this.counter[2], subValue: this.counter[2].subCounter})

        Divider().height(5)

        ForEach(this.counter,
          item => {
            CounterComp({value: item, subValue: item.subCounter})
          },
          item => item.id.toString()
        )

        Divider().height(5)

        Text('Parent: reset entire counter')
        .fontSize(20).height(50)
        .onClick(()=>{
          this.counter = [ new Counter(1), new Counter(2), new Counter(3) ];
        })
        Text('Parent: incr counter[0].counter')
        .fontSize(20).height(50)
        .onClick(()=>{
          this.counter[0].incrCounter();
          this.counter[0].incrSubCounter(10);  // This works to increment the SubCounter value by 10
        })

        Text('Parent: set.counter to 10')
        .fontSize(20).height(50)
        .onClick(()=>{
          this.counter[0].setSubCounter(10); // This DOES NOT work to set the SubCounter value by 10
        })
      }
    }
  }
}
```

### 3.4.4 `@Prop` object copy semantics

In the previous case, the `CounterComp` component has variable decorated with `@ObjectLink`. `@ObjectLink` is a reference to the object owned by the @State in the `ParentComp` component
Changes made to these variables within `CounterComp` are also observed in the `ParentComp` component.

Replace the `@ObjectLink` with `@Prop` decorator, add some more UI output to notice the issue: 
2nd click handler updates UI ok, 3rd click handler leads to no UI update. There is an issue with application logic. - Can you find it? 

``` Typescript
@Component
struct CounterComp {

  @Prop value: Counter;
  @Prop subValue: SubCounter;
  
   build() {
    Column(space: 10) {
      Text(`this.value.incrCounter(): this.value.counter: ${this.value.counter}`)
        .fontSize(20)
        .onClick(()=>{
          // 1st click handler
          this.value.incrCounter();
        })
        Text(`this.subValue.counter: ${this.subValue.counter}`)
        .onClick(()=>{
            // 2nd click handler
            this.subValue.incrSubCounter(7);
        })
        Text(`this.value.subCounter.counter: ${this.value.subCounter.counter}`)
        .onClick(()=>{
            // 3rd click handler
            this.value.incrSubCounter(77);
        })
        Divider().height(2)
    }
  }
}


@Entry
@Component
struct ParentComp {
  @State counter: Counter[] = [ new Counter(1), new Counter(2),  new Counter(3) ];

  build() {
    Row() {
      Column() {
        CounterComp({value: this.counter[0], subValue: this.counter[0].subCounter})
        CounterComp({value: this.counter[1], subValue: this.counter[1].subCounter})
        CounterComp({value: this.counter[2], subValue: this.counter[2].subCounter})

        Divider().height(5)

        ForEach(this.counter,
          item => {
            CounterComp({value: item, subValue: item.subCounter})
          },
          item => item.id.toString()
        )

        Divider().height(5)

        Text('Parent: reset entire counter')
        .fontSize(20).height(50)
        .onClick(()=>{
          this.counter = [ new Counter(1), new Counter(2), new Counter(3) ];
        })
        Text('Parent: incr counter[0].counter')
        .fontSize(20).height(50)
        .onClick(()=>{
          this.counter[0].incrCounter();
          this.counter[0].incrSubCounter(10);
        })

        Text('Parent: set.counter to 10')
        .fontSize(20).height(50)
        .onClick(()=>{
          this.counter[0].setSubCounter(10);
        })
      }
    }
  }
}
```

The issue:

`@Prop` creates a local copy of the variables. Therefore, `this.value.subCounter` is not the same object as `this.subValue`. Therefore, `this.value.incrSubCounter()` does not modify the copy of `SubCounter` object that is rendered: `Text('${this.value.subCounter.counter}')`.

The Solution:

The solution is designed to preserve the one-way sync for `value` from Parent to CounterComp.
We need one deep-copy of the `value : Counter` object and we must avoid the second copy for SubCounter object. 

This can be done
- use just one `@Prop counter : Counter` in `CounterComp`
- add another sub-component `SubCounterComp` with an `@ObjectLink subCounter: SubCounter` . This `@ObjectLink` makes sure `SubCounter` object property changes are observed and UI updates ok.  `@ObjectLink` shares the same `SubCounter` object with `CounterComp`.

``` Typescript
let nextId = 1;

@Observed
class SubCounter {
  counter : number;

  incrSubCounter(c: number) {
    this.counter = this.counter+c;
  }

  setSubCounter(c : number) {
    this.counter = c;
  }

  constructor(c : number) {
    this.counter = c;
  }
}

@Observed
class Counter {
  id : number;
  counter : number;
  subCounter : SubCounter;

  incrCounter() {
    this.counter++;
  }

  incrSubCounter(c: number) {
    this.subCounter.incrSubCounter(c);
  }

  setSubCounter(c: number) : void {
    this.subCounter.setSubCounter(c);
  }

  constructor(c : number) {
    this.id = nextId++;
    this.counter = c;
    this.subCounter = new SubCounter(c);
  }
}

@Component
struct SubCounterComp {

  @ObjectLink subValue: SubCounter;
  
   build() {
      Text(`SubCounterComp: this.subValue.counter: ${this.subValue.counter}`)
      .onClick(()=>{
          // 2nd click handler
          this.subValue.incrSubCounter(7);
      })
  }
}

@Component
struct CounterComp {

  @Prop value: Counter;
  
   build() {
    Column(space: 10) {
      Text(`this.value.incrCounter(): this.value.counter: ${this.value.counter}`)
        .fontSize(20)
        .onClick(()=>{
          // 1st click handler
          this.value.incrCounter();
        })
        
        SubCounterComp({ subValue: this.value.subCounter })
        
        Text(`this.value.incrSubCounter()`)
        .onClick(()=>{
            // 3rd click handler
            this.value.incrSubCounter(77);
        })
        Divider().height(2)
    }
  }
}


@Entry
@Component
struct ParentComp {
  @State counter: Counter[] = [ new Counter(1), new Counter(2),  new Counter(3) ];

  build() {
    Row() {
      Column() {
        CounterComp({value: this.counter[0]})
        CounterComp({value: this.counter[1]})
        CounterComp({value: this.counter[2]})

        Divider().height(5)

        ForEach(this.counter,
          item => {
            CounterComp({value: item})
          },
          item => item.id.toString()
        )

        Divider().height(5)

        Text('Parent: reset entire counter')
        .fontSize(20).height(50)
        .onClick(()=>{
          this.counter = [ new Counter(1), new Counter(2), new Counter(3) ];
        })
        Text('Parent: incr counter[0].counter')
        .fontSize(20).height(50)
        .onClick(()=>{
          this.counter[0].incrCounter();
          this.counter[0].incrSubCounter(10);  // This works to increment the SubCounter value by 10
        })

        Text('Parent: set.counter to 10') // can not render this.value.subCounter.subValue here, changes would not be observed
        .fontSize(20).height(50)
        .onClick(()=>{
          this.counter[0].setSubCounter(10); // This DOES NOT work to set the SubCounter value by 10
        })
      }
    }
  }
}
```


### 3.4.5 Application must not mutate application state during `build`

``` Typescript

/* Notice: Development practices to avoid */

@Entry
@Component
struct CompA {
@State col1 : string = Color.Yellow;
@State col2 : string = Color.Green;
@State count : number = 1;

    build() {
        Column() {
           //count variable mutated directly in Text
            Text(`${this.count++}`)
            .width(50)
            .height(50)
            .fontColor(this.col1)
            .onClick(() => {
                this.col2 = Color.Red;
            })
        }
        .backgroundColor(this.col2)
    }
}
```

When tested on the target environment, the behavior of the above application in regard to the Text label `Text('${this.count}')` may vary based on whether a full or partial update is performed.
- The framework might go into an infinite re-render loop, because each rendering of the Text mutates the app state that causes another re-render cycle to start.
- Whenever `this.col2` changes, the _entire_ build function gets executed, thereby, `this.count` Text label binding changes. This is how previously used ArkUI full update solution behaves. However, in current ArkUI partial update solution only the Column gets updates, the Text binding does not change.
- Whenever `this.col1` changes, the _entire_ Text component gets updates, _all_ its attribute functions execute. This is the behaviour of current ArkUI partial update solution. However, in the future, we may change to an update solution where only `.fontColor(this.col1)` attribute function is executed, the Text label function is not.

The recommended approach to develop the application is given below:

``` Typescript

/* Recommended approach */

@Entry
@Component
struct CompA {
  @State col1 : string = Color.Yellow;
  @State col2 : string = Color.Green;
  @State count : number = 1;

  build() {
    Column() {
      Text(`${this.count}`)
        .width(50)
        .height(50)
        .backgroundColor(this.col1)
        .onClick(() => {
          this.count++;
      })
    }
    .backgroundColor(this.col2)
  }
}
```

The recommended approach to app development involves handling the `count++` operation within the actions of the click event handler. This ensures that the count is incremented only when a specific action is performed.

Reader take note that mutating applications state inside the build function can be more hidden than in above example:
- mutate state in a `@Builder`, `@Extend` or `@Style` function 
- mutating state in a JS function called to compute a parameter , e.g. `Text('${this.calcLabel()}')`
- use of functions that make _in-place_ modifications to an Array. For instance 

```TypeScript
@State arr : Array<..> = [ ... ];
ForEach(this.arr.sort().filter(....), 
  item=> { 
  ...
})
```
`sort()` modifies the Array `this.arr` in-place. Only subsequent call to `filter(..)` returns a new  Array.

The correct implementation is
```TypeScript
ForEach(this.arr.filter(....).sort(), 
  item=> { 
  ...
})
```
`filter(..)` returns a new  Array, which `.sort()` will modify. `this.arr` remains unchanged.

 **Learning lesson** : ArkUI forbids mutation of any  state variables during the build process. The motivation is to leave room for optimisations of the re-render process and to avoid ambiguities in application behaviour.


### 3.4.6 UIUpdater - Force UI update with artificial app state

``` Typescript
 /* Not recommended version */
@Entry
@Component
struct CompA {
  @State needsUpdate : boolean = true;  
  realState1 : Array<number> = [ 4, 1, 3, 2 ];  // not decorated to mark as app state
  realState2 : string = Color.Yellow;

  updateUI(param : any) : any {
        const triggerAGet = this.needsUpdate;
        return param;
    }

  build() {
    Column({ space: 20 }) {
        ForEach(this.updateUI(this.realState1),
        item => {
          Text(`${item}`)
        })
        Text("add item")
          .onClick(() => {
            // mutating realState1 does not trigger UI update
            this.realState1.push(this.realState1[this.realState1.length-1] + 1);

            // trigger UI update
            this.needsUpdate = !this.needsUpdate;
        })
        Text("chg color")
          .onClick(() => {
            // mutating realstate2 does not trigger UI update
            this.realState2 = this.realState2==Color.Yellow ? Color.Red : Color.Yellow;

            // trigger UI update
            this.needsUpdate = !this.needsUpdate;
        })
     }.backgroundColor(this.updateUI(this.realState2))
     .width(200).height(500)
  }
}
```

The issues with above implementation are
- The app wants to control the UI update logic, but in ArkUI it should be left to the framework to detect app state changes and to applu the needed UI updates. 
- `this.needsUpdate` is an artificial UI state used solely to force the framework to make specific UI updates. Variables `this.realState1, this.realState2` are not decorated, hence, the framework does not interpret these as app state. 
- the app has sub-optimal UI update performance: mutating `this.needsUpdate`will update UI even if there is no need.
- the app logic is harder to understand than necessary.

To fix this issue, the `realState1` and `realState2` variables shall be made `@State` decorated. Once this is done, there is need for variable `needsUpdate` anymore:

``` Typescript
 /* Recommended version */
@Entry
@Component
struct CompA {
  @State realState1 : Array<number> = [ 4, 1, 3, 2 ];
  @State realState2 : string = Color.Yellow;

  updateUI(param : any) : any {
        const triggerAGet = param;
        return param;
    }

  build() {
    Column({ space: 20 }) {
        ForEach(this.updateUI(this.realState1),
        item => {
          Text(`${item}`)
        })
        Text("add item")
          .onClick(() => {
            // mutating realstate1 triggers UI update
            this.realState1.push(this.realState1[this.realState1.length-1] + 1);

        })
        Text("chg color")
          .onClick(() => {
            // mutating realstate2 triggers UI update
            this.realState2 = this.realState2==Color.Yellow ? Color.Red : Color.Yellow;
        })
     }.backgroundColor(this.updateUI(this.realState2))
     .width(200).height(500)
  }
}
```

### 3.4.7 Duplicate Array ids caused by adding Array element in ForEach - case 1

The below example is using the `ForEach` method to iterate over each element of an array `this.arr` and displays them in `Text` and performs addition of an array element on click of `Text('Add arr element')`.

``` Typescript
@Entry
@Component
struct Index {
  @State arr: number[] = [1,2,3];

  build() {
      Column() {
       ForEach(this.arr,
                (item) => {
                    Text(`Item ${item}`)
                 },
                item => item.toString())
        Text('Add arr element')
        .fontSize(20)
        .onClick(()=>{
            this.arr.push(4);
            console.log("Arr elements: ", this.arr);
        })
      }
    }
}
```

In the above example, when you click on the `Text('Add arr element')` element twice, the array `this.arr` gets added with the number `4` each time. However, with the third parameter `(item) => item.toString()` in `ForEach` loop that is expecting unique Array id values. The requirement is that Array ids be unique and stable. 
- unique: The array id generation function is expected to compute a different value for each array item.  
- stable: The framework considers an array item be replaced or changed when its item id has changed.
The framework warns about duplicate ids. The framework behaviour is undefined, especially the UI update may not work in this case. 
  
 In our example the framework will not display the newly added text element for the second time or later. This is because the item is no longer unique as it contains the same element `4` more than once.

But if you remove the third optional parameter `(item) => item.toString()` from the `ForEach` loop, the `Text` element gets updated with the newly added array elements on each 'onClick'. This is because the framework uses the default Array id generation function. Current implementation uses this function as the default  `(item: any, index : number) => '${index}__${JSON.stringify(item)}';`. It is more permissive but leads to unnecessary UI updates. So, apps are advised to define their own Array id function.


### 3.4.8 Duplicate array element update in ForEach - case 2

The below example defines two components: `Index` and `Child`. `Index` has an array property called arr that initially contains the numbers 1, 2, 3 and `Child` has a @Prop `value` that takes the arr element as input from its parent `Index` component.

``` Typescript

/* Notice: Development practices to avoid */

@Component
struct Child {
  @Prop value: number;
  build() {
    Text(`${this.value}`)
      .fontSize(50)
      .onClick(()=>{this.value++})
  }
}
@Entry
@Component
struct Index {
  @State arr: number[] = [1,2,3];

  build() {
    Row() {
      Column() {
        Child({value: this.arr[0]})
        Child({value: this.arr[1]})
        Child({value: this.arr[2]})

        Divider().height(5)

        ForEach(this.arr,
          item => {
            Child({value: item})
          },
          item => item.toString()
        )
        Text('Parent: replace entire arr')
        .fontSize(50)
        .onClick(()=>{
          // note that both arrays include the item '3'.
          // this means id does not change,
          // means ForEach fill not update that Child instance
          // and consequently its @Prop is not updated from parent
          this.arr = (this.arr[0] == 1) ? [3,4,5] : [1,2,3];
        })
      }
    }
  }
}
```

When the onClick handler of the Text component "Parent: replace entire arr" is triggered, the state `arr` is replaced with new set of elements containing either the numbers 3, 4, and 5 or the numbers 1, 2, and 3 depending on the current value of the first element of arr. However, the `Child` component created inside the ForEach loop is not updated with the new value input.

This happens because both the old and new arrays contain an element with the same value (i.e., the number 3), and the identifier generated for this element in the parent component does not change. As a result, the ForEach loop does not recognize that the corresponding `Child` instance needs to be updated with the new value input, and the @Prop in this Child component does not update accordingly.

To see this behavior in action, you can replace the duplicate element "3" with a unique value in the arr state array, which should result in the expected behavior of properly replacing the array based on the first element.


### 3.4.9 Array of Objects with `@StorageLink` / `@LocalStorageLink`

TBC, Guido



---

# 4. Managing State owned by a Component

> This chapter has been updated, especially the sections about extended @Prop and @ObjectLink have undergone substantial changes.
> It is ready to be used
> a few details still need to be checked against the spec
> - initialization and update from parent @Component. What decorated variables in parent can be used to init what decorated variable in child @Component. The stated info seems correct but incomplete, e.g. especially @Provide and @Consume need checking.
> - SubscribableAbstract class exists, need to test it still works, and need to review the spec where it is allowed to be used.
> many ETS examples have already been checked, for others this remains to be done. Publish the source code of all examples.
> Move `Watch` to chapter 4.



## 4.1 State owned by a Component `@State`

An `@State` decorated variable includes some state that is used to render the custom component. It can be of simple or class object type. 

An `@State` decorated variable, like all other decorated variables in ArkUI Declarative, are private and only accessible from within the component.  Its type and its local initialization must be specified. Initialization from the parent component using the named parameter mechanism is optional. 

An `@State` decorated variable _owning_ some state means two things: 
1. An one-way and two-way data sync relationship can be created from an `@State` variable to a `@Prop`, `@Link` or `@ObjectLink` decorated variable in child component. 
2. The` @State` decorated variable lifecycle is the same as that of its owning custom component.

The following is an example of a simple type `@State` variable:

```typescript
@Entry
@Component
struct MyComponent {

    @State count: number = 0;

    // MyComponent provides a method for modifying the @State status data member.
    private toggleClick() {
        this.count += 1;
    }

    build() {
        Column() {
            Button() {
                Text(`click times: ${this.count}`)
                    .fontSize(10)
            }.onClick(this.toggleClick.bind(this))
        }
    }
}
```

The following is a class type example for `@State` variable:

```typescript
// Customize the status data class.
class Model {
    public value: string;
    constructor(value: string) {
        this.value = value;
    }
}

@Entry
@Component
struct EntryComponent {
    build() {
        Column() {
            // any named parameter specified here will overwrite the 
            // locally defined default value on first render
            MyComponent({ count: 1, increaseBy: 2 })  
            MyComponent({ title: new Model('Hello, World 2'), count: 7 })
        }
    }
}

@Component
struct MyComponent {

    @State title: Model = new Model('Hello World');
    @State count: number = 0;
    private increaseBy : number = 1;

    build() {
        Column() {
            Text(`${this.title.value}`)
            Button() {
                Text(`Click to change title`).fontSize(10)
            }.onClick(() => {
                // update the @state variable, will cause above Text to update
                this.title.value = this.title.value == 'Hello Ace' ? 'Hello World' : 'Hello Ace';
            })

            Button() {
                Text(`Click to increase count=${this.count}`).fontSize(10)
            }.onClick(() => {
                // update the @state variable, will cause above Text to update
                this.count += this.increaseBy;
            })
        } // Column
    } // build
}
```

In the preceding example, the user-defined component `MyComponent` defines the `@State` state variables `count` and `title` of simple (number) and class object types. When the value of `count` or `title` changes, the owning `MyComponent` instance will re-render and affected `Text` components will update. 


### 4.1.1 Provisions for using `@State`

| @State variable decorator | Comment |
|---|---|
| custom component variable decorator | |  
| decorator parameters | none |
| type of sync | No sync with any kind of variable in parent component. |
| permissible variable types and observed changes | See [section 3.2.5]((intro-state-mgmt.md#325-variable-types-and-observed-changes-for-variables-decorated-with--state-provide-link-consume-or-prop)) |
| local initialization | mandatory, see [section 3.2.8]((intro-state-mgmt.md#328-provisions-for-local-variable-initialization))  |
| initialization from parent component | Allowed, see [section 3.2.9]((intro-state-mgmt.md#329-provisions-for-variable-initialization-from-the-parent)) |
| Update from parent | None |
| Can initialize and update sub-components | permissible, this is the typical case, see [section 3.2.8](intro-state-mgmt.md#328-provisions-for-variable-initialization-from-the-parent) and [section 3.2.9](intro-state-mgmt.md#329-provisions-for-variable-update-from-the-parent) for details |
| access | private, accessible only within the owning component | 

An `@State` decorated variable, like all other decorated or non-decorated component variables, shares the lifecycle of its owning component.

When creating a new instance of the owning component (first render), the processing steps are the following to determine the initial `@State` variable value.
1. Apply the locally defined default value
2. Apply the named parameter value, if one is supplied. Named parameters is the mechanism of how to set a custom initial value for component member variable. Providing a custom initial value is optional for `@State` decorated variables.

When updating an existing instance of the owning component (re-render), the processing steps are the following to determine the `@State` variable value.
1. The variable value before re-render is maintained.
2. _No_ update to @State variable value.


## 4.2 Component state uni-directionally synced from parent component `@Prop`

`@Prop` has the same semantics as `@State` with the differences of how the variable must be initialized and how it is updated:
*  An `@Prop` decorated variable must be initialised with a simple-type or class-type value provided by its parent component. It must not be initialized locally. 
* An `@Prop` variable is allowed to be modified locally, but the change does not propagate back to its parent component. 
* Whenever that data source changes, the `@Prop` decorated variable gets updated, and any locally made changes are overwritten. Hence, the sync of value is uni-directional from the parent to the owning component.

### 4.2.1 Scenario - Simple type `@Prop` synced from `@State` in parent component

The most basic scenario that has been supported from the start of ArkUI:  The value of an `@State` decorated variable in parent component initialises a `@Prop` decorated variable in child component. The `@State` variable value also updates the `@Prop` variable whenever the `@State` changes. Changes to `@Prop` decorated variable do not affect the value of its source `@State`. Instead of an `@State` decoration the source could also be decorated with `@Link` or `@Prop`, mechanisms for syncing the `@Prop` would be the same.

The type of the source and the `@Prop` variable must be the same, both simple types and class are permissible.

Example (`propState.ets`):

```typescript
@Component
struct CountDownComponent {

    @Prop count: number;
    costOfOneAttempt: number = 1;

    build() {
        Column() {
            if (this.count> 0) {
                Text(`You have ${this.count} Nuggets left`)
            } else {
                Text("Game over!")
            }

            Button() {
                Text("Try again")
            }.onClick(() => {
                this.count -= this.costOfOneAttempt;
            })
        }
    }
}

@Entry
@Component
struct ParentComponent {

    @State countDownStartValue: number = 10; 

    build() {
        Column() {
            Text(`Grant ${this.countDownStartValue} nuggets to play.`)
            Button() {
                Text("+1 - Nuggets in New Game")
            }.onClick(() => {
                this.countDownStartValue += 1;
            })

            Button() {
                Text("-1  - Nuggets in New Game")
            }.onClick(() => {
                this.countDownStartValue -= 1;
            })

            CountDownComponent({ count: this.countDownStartValue, costOfOneAttempt: 2 })
        }
    }
}
```

On initial render, when creating `CountDownComponent` sub-component its `@Prop counter` variable is initialised from `ParentComponent` `@State countDownStartValue` variable. 


When pressing "+1" or "-1" Button the `@State countDownStartValue` of `ParentComponent` changes. This will cause `ParentComponent` to re-render. At the minumum `CountDownComponent` will be updated because of its changed parameter.

Updating `CountDownComponent` will update the value of `@Prop counter`. This is mentioned one-way sync mechanism from parent variable to @Prop decorated variable. 

Because of the change of `@Prop counter` child component `CountDownComponent` will re-render as well. At a minimum, the `if` statement's conditions `(this.counter> 0)` is-evaluated and Text sub-component inside the `if` would be updated.


One special case of multiple `@Prop` variables update from parent needs to be pointed out:
Just one decorated source variables change in the parent is enough to cause the parent component to re-render. This re-render sync's _all_ `@Prop` variables with the source' values. To illustrate, lets modify above example:

```TypeScript
@Component
struct CountDownComponent {

    @Prop count: number;
    @Prop costOfOneAttempt: number = 1;
    ...
}

struct ParentComponent {

    @State countDownStartValue: number = 10; 
    @State costOfOneAttempt : number = 2;

    build() {
        ...
        CountDownComponent({ 
            /* @Prop */ count: this.countDownStartValue, 
            /* Prop */ costOfOneAttempt: this. costOfOneAttempt
        })
    }
}
```
When either `State countDownStartValue` or `@State costOfOneAttempt` changes, then `ParentComponent` will re-render. The re-render will update `CountDownComponent` component instance, this update will cause both `@Prop` to be synch'd with the value of their source `@State` variables.



### 4.2.2 Scenario - Simple type `@Prop` with local init and no sync from parent

Sometimes a reusable component developer wants to provide a local initialization to a `@Prop` and leave it to the using app to decide if it wants to establish a sync relationship with a variable in its parent component. Therefore `@Prop` allows for optional local initialization. If and only if local initialization is provided, giving a sync source in the parent becomes optional.

The following example includes two `@Prop` variables in child component. 
* `@Prop customCounter` has no local init, therefore its required that using parent @Component provides a source from where to init the `@Prop` and update the `@Prop` when the source value changes. 
* `@Prop customCounter2` has a local init. In this situation it is still allowed but not mandatory to specify a sync source in the parent when creating the child component.

It is the responsibility of the ETS transpiler to verify in the ETS source code that sync source is given in the constructor call for any @Prop without local init. Below example can also be used to verify that behaviour by enabling the latter two `CustomComponent` constructor calls.

```TypeScript

@Component
struct CustomComponent {

    // Parent provides the initialization for this. 'customCounter' gets updates from parent.
    @Prop customCounter: number;
    // Parent does not give initialization for this. So this must be initialized here.
    @Prop customCounter2: number = 5;

    build() {
        Column() {

            Row() {
                Text(`From Main: ${this.customCounter}`).width(90).height(40).fontColor("#FF0010")
            }

            Row() {
                Button("Click to change locally !").width(480).height(60).margin({top:10})
                    .onClick(() => {this.customCounter2++})
            }.height(100).width(480)

            Row() {
                Text(`Custom Local: ${this.customCounter2}`).width(90).height(40).fontColor("#FF0010")
            }
        }
     }
}

@Entry
@Component
struct MainProgram {

    @State mainCounter : number = 10;

    build() {
        Column() {
            Row() {
               Column() {
                  Button("Click to change number").width(480).height(60).margin({top:10, bottom:10})
                  .onClick(() => {
                            this.mainCounter++
                        })
                }
             }
             Row() {
                Column() {
                    // Here we only initialize the @Props that required init from parent because it lacks local init.
                    CustomComponent({customCounter: this.mainCounter})

                    // Here we initialize both @Props from this parent component.
                    // init from parent will overwrite local init of customCounter2
                    CustomComponent({customCounter: this.mainCounter, customCounter2: this.mainCounter})

                    // error case
                    // Here we only initialize one of @Prop. customCounter is not initialized here or in custom component, this is an error.
                    // Expected ETS transpiler output: Error @Prop: customCounter must be initialized in parent or in custom component
                    // CustomComponent({customCounter2: this.mainCounter})

                    // error case
                    // mandatory initialization of customCounter missing
                    // Expected ETS transpiler output: Error @Prop: customCounter must be initialized in parent or in custom component
                    // CustomComponent()

                    // error case:
                    // Here we only initialize one of @Props to null or undefined. This is not allowed.
                    // Output: Error @Prop: customCounter must be initialized in parent or in custom component
                    // CustomComponent({customCounter: null})
                    // CustomComponent({customCounter: undefined})
                }.width('40%')
            }
            Row() { Text("").width(480).height(10)}
        }
    }
}
```

### 4.2.3 Scenario - Simple type `@Prop` synch'd from `@State` Array item in parent component

A frequently used pattern is to use an `Array<T>` item in parent component as data source to initialize and update a `@Prop` decorated variable.


Example (`propForEach.ets`):

```typescript
@Component
struct Child {
  
  @Prop value: number;

  build() {
    Text(`${this.value}`)
      .fontSize(50)
      .onClick(()=>{this.value++})
  }
}

@Entry
@Component
struct Index {
  @State arr: number[] = [1,2,3];

  build() {
    Row() {
      Column() {
        Child({value: this.arr[0]})
        Child({value: this.arr[1]})
        Child({value: this.arr[2]})

        Divider().height(5)

        ForEach(this.arr, 
          item => {
            Child({value: item})
          }, 
          item => item.toString()
        )
        Text('replace entire arr')
        .fontSize(50)
        .onClick(()=>{
          // note that both arrays include the item '3'.
          this.arr = this.arr[0] == 1 ? [3,4,5] : [1,2,3];
        })
      }
    }
  }
}
```



Initial render creates 6 instances of `Child` component. Each `@Prop` is initialized with a _copy_ of an array item.
`Child` `onclick` event handler changes the local variable `value`.

Lets assume we clicked so many times that all local values be '7'

```text
7
7
7
----
7
7
7
```

After clicking `replace entire arr`, the screen will show the following - howcome?

```text
3
4
5
----
7
4
5
```

`Child({value: this.arr[0]})` component update syncs `this.arr[0]` to the instance' `@Prop` variable because `this.arr[0]` has changed. The same happens for `Child({value: this.arr[1]})` and `Child({value: this.arr[1]})`.

`this.arr` change causes `ForEach` to update: Array item with id '3' is retained in this update, array items with ids  '1' and '2'  are deleted and array items with id '4' and '5' added. This implies that `Child` generated for item '3' will _not_ update and its `@Prop` will not be synced from its (unchanged!) source. Two `Chlld` component instances will be deleted and two new ones be created.


### 4.2.4 Scenario - Class object type `@Prop` synch'd from `@State` class object property in parent component


Example (`propObjectShared.ets`)

A library with a single book and two customers Each user can mark the book as read, but this does not affect the user readers.
Technically speaking, local changes to the `@Prop book` object do not sync back to `@State book` in `Library` component.

`Book` class can but does not need to be decorated with `Observed` in this example. That's only needed for nested structures, as we explain in the next section.

```typescript
class Book {
    public title: string;
    public pages: number;
    public readIt : boolean = false;

    constructor(title: string, pages: number) {
        this.title = title;
        this.pages = pages;
    }
}

@Component
struct ReaderComp {
    @Prop book: Book;

    build() {
        Row() {
            Text(this.book.title)
            Text(` ... has ${this.book.pages} pages!`)
            Text(` ... ${this.book.readIt ? "I have read" : "I have not read it"}`)
              .onClick(() => this.book.readIt = true )
        }
    }
}

@Entry
@Component
struct Library {
    @State book: Book =  new Book("100 secrects of C++", 765);

    build() {
        Column() {
            ReaderComp({ book: this.book })
            ReaderComp({ book: this.book })
        }
    }
}
```

### 4.2.5 Scenario - Class type `@Prop` synced from `@State` array item in parent component

Example (`propObjectArray.ets`):

With the previous examples understood there is only one detail that needs explanation.
'Mark unread for everyone' event handler changes a property inside the Book object that is contained
in the `@State allBooks` array. `@State` decorator will not observe this property change, because 
it is a nested property of 2nd level. It can only observe properties on first level. Hence, the 
`ReaderComp` will not be updated. Nonetheless, it is expected that the `@Prop` is reset to 
the value of its changed source array book item. 

This is the purpose of the  `@Observed` class decorator. `@Observed` causes all instances of the 
decorated class to be wrapped with an opaque  proxy object. This proxy can detect 
all property changes inside the wrapped object. If this happens the proxy notifies the `@Prop`
and the `@Prop` value will be updated. 

```typescript
let nextId : number = 1;

@Observed
class Book {
    public id : number;
    public title: string;
    public pages: number;
	public readIt : boolean = false;

    constructor(title: string, pages: number) {
        this.id = nextId++;
        this.title = title;
        this.pages = pages;
    }
}

@Component
struct ReaderComp {
	@Prop book: Book;

	build() {
		Row() {
			Text(this.book.title)
			Text(` ... has ${this.book.pages} pages!`)
			Text(` ... ${this.book.readIt ? "I have read" : "I have not read it"}`)
			  .onClick(() => this.book.readIt = true )
		}
	}
}

@Entry
@Component
struct Library {
	@State allBooks: Book[] = [ new Book("100 secrects of C++", 765), new Book("Effective C++", 651), new Book("The C++ programming language", 1765) ];

	build() {
        Column() {
			Text(`Library's all time favorite`)
			ReaderComp({ book: this.allBooks[2] })
			
			Divider()
			
			Text(`Books on loan to a reader`)
		    ForEach(this.allBooks, 
                book => {
                    ReaderComp({ book: book })
                },
                book => book.id
            )

            Button(`Add new`)
            .onClick(() => {
                this.allBooks.push(new Book("The C++ Standard Library", 512));
            })
            Button(`Remove first book`)
            .onClick(() => {
                this.allBooks.shift();
            })
            Button(`Mark unread for everyone`)
            .onClick(() => {
                this.allBooks.forEach((book) => book.readIt = false)
            })
        }
    }
}
```

### 4.2.6 Provisions for using `@Prop`

| @Prop variable decorator | Comment |
|---|---|
| custom component variable decorator | |  
| decorator parameters | none |
| permissible type and observed changes | See [section 3.2.5](intro-state-mgmt.md#325-variable-types-and-observed-changes-for-variables-decorated-with--state-provide-link-consume-or-prop) |
| local initialization | Optional (added in API 10, before forbidden). See [section 3.2.8]((intro-state-mgmt.md#328-provisions-for-local-variable-initialization))  |
| initialization from parent component | Optional if local initialization exists. Mandatory otherwise. 
|| Valid: `CompA({ /* aProp not provided */ }) `
|| Invalid: `CompA: ({ aProp: undefined }) `
|| Invalid: `CompA: ({ aProp: null })` . |
| Sync with parent ("sync source") | One-way sync from parent if initialized from parent. The sync source in parent must be a decorated variable of same type, or an array item of decorated variable, or an object property of decorated variable. See [section 3.2.9](intro-state-mgmt.md#329-provisions-for-variable-initialization-from-the-parent), and [section 3.2.10](intro-state-mgmt.md#3210-provisions-for-variable-update-from-the-parent)  | 
| Can initialize and update sub-components | permissible, see [section 3.2.8](intro-state-mgmt.md#328-provisions-for-variable-initialization-from-the-parent) and [section 3.2.9](intro-state-mgmt.md#329-provisions-for-variable-update-from-the-parent) for details |
| access | A `@Prop` variable is private, accessible only within the component | 

An `@Prop` decorated variable shares the life cycle of their owning component.

To understand the `@Prop` variable value initialization and update mechanism, it is necessary to consider the parent component and the child component that owns the `@Prop` variable.

1. Initial render: The execution of the parent component's `build` function creates a new instance of the child component. The initialization is the following:  The `@Prop` decorated variable is initialized. A copy is made to enable changes to remain local, see the deepcopy specification below. An class object or array type `@Prop` variables keep track of the source and observe object property / array item changes to it.

2. Update: 
When the parent component has provided a source for @Prop variable then the following happens:
    * Observed `@Prop` source change, or source component object property changes, or source array item changes: As mentioned `@Prop` keeps track of its source.  When @Prop observes a change to its source, its local value is re-initialized with a deep copy (see (1)). Any changes of the local value are overwritten.
    * Parent component re-render: A parent component re-render will update the child component when needed. An owning component update resets the `@Prop` source, which can trigger the process described in (2)

`@Prop` observed changes to its local value in the same way as `@State` does.


**The `@Prop` deep copy (starting API rel. 10)**
`@Prop` realizes a one-way sync, its value can be modified locally. Hence, any object needs to be deep-copied that is provided by its parent component (on initial render or subsequent update). The deep copy works with
- those object properties that `Object.keys(propObj)` returns
- object prototype (e.g. functions are copied ok)
- nested objects
- `Set`, `Map`, `Date`, `Array`, array (i.e. created with `[]`)
- graph-shaped data models: data models that include objects referenced from multiple places (e.g. Object multiple times in an array, references back to a 'parent', 'root',  or to a 'sibling' node).

Copy does not work for:
- objects that are implemented in native other than `Set`, `Map`, `Date` mentioned above

**The `@Prop` shallow copy in API rel. 9:**
API 9 release wrongly implements a shallow copy solution for `@Prop` as follows. the implementation is wrong because it is _not_ a proper _one-way sync_:
- For object-type `@Prop`: shallow copy copies those object properties that `Object.keys(propObj)` returns
- for array type `@Prop`: shallow copy copies array items
- object/array prototype (e.g. functions are copied ok)
- copy stops at the top level object or array, references are to _original_ nested objects and arrays inside.
    * an outer array is copied. If it includes references to original objects/arrays, those nested objects and arrays are not copied. 
    * an outer object is copied. Object-type properties values reference to the original objects. Likewise, array-type property values reference the original array. That is because those nested objects and arrays are not copied ("shallow copy").
- for `Date` type: creates new `Date` object and sets its value from the `Date` value from the original.
- `Map`, `Set` builtin types are unsupported, so are any other objects / classes with native implementation.
 
> Shallow-copy instead of proper deep-copy was a bug while preparing API rel. 9. Since many applications seem to 'rely' on this wrong behaviour it was decided to keep shallow-copy for API rel. 9 . Applications upgrading to API rel. 10 will need to be fixed to either work properly with `@Prop` deep-copy. Alternatively apps could be changed to use the generalized `@ObjectLink` decorator: For object type state variables `@ObjectLink` works in the same way as `@Prop` except that `@ObjectLink` shares a reference to the same object with its source state variable. No copy is made. See [section 4.5](manage-state-component.md#45-object-reference-with-objectlink) especially the example in [section 4.5.2](manage-state-component.md#452-objectlink-variable-and-its-source-variable-of-parent-component-both-refer-to-the-same-object).


## 4.3 Bi-bidirectional syncing with a variable of parent component `@Link`


An `@Link` decorated variables in a child component shares the same value with a variable in parent component.
That source variable must be decorated with `@State`, `@StorageLink`, or `@Link`.

An example (`propLink.ets`) of using `@Link` for bidirectional communication, and also provides another example for the use of `@Prop`:

```typescript
@Component
struct ChildA {
    
    @Prop counterVal : number;
    
    build() {
        Button(`ChildA: this.counterVal(${this.counterVal}) + 1`)
            .onClick(() => this.counterVal += 1)
            .width(400).height(100) 
    }
}

@Component
struct ChildB {
    
    @Link counterRef : number;
    
    build() {
        Button(`ChildB: this.counterRef(${this.counterRef}) + 1`)
            .onClick(() => this.counterRef += 1)
            .width(400).height(100) 
    }
}

@Entry
@Component
struct ParentView {

    initialCounterValue : number = 1;
    @State counter : number = this.initialCounterValue;

    build() {
        Column() {
            ChildA({ counterVal: this.counter })

            ChildB({ counterRef: this.counter })
            ChildB({ counterRef: this.counter })

            Text(`Parent: this.counter= ${this.counter}`)
             .width(400).height(100) 
        }
    }
}
```

ParentView owns the `counter` state, established a one-way data sync with ChildA, and a two-way data sync with ChildB sub-components' counterRef link. Whenever `counterRef` changes in an instance of ChildB, `counter` in ParentView updates automatically, and from there also the `counterRef` in the other ChildB component updates. Also counterVal in ChildA instance is updated from its source `counter` in ParentView. As state in all three components has changed all of them need to re-render, starting with the `ParentView`.

### 4.3.1 Provisions for using `@Link`

| @Link variable decorator | Comment |
|---|---|
| custom component variable decorator | |  
| decorator parameters | none |
| type of sync | two-way: from an `@State`, @StorageLink or another `@Link` decorate variable in the parent component to this variable and also into the other direction |
| permissible variable types and observed changes | See [section 3.2.5](intro-state-mgmt.md#325-variable-types-and-observed-changes-for-variables-decorated-with--state-provide-link-consume-or-prop) |
| local initialization | forbidden |
| initialization and update from parent component | Mandatory to init from parent. Initialization from parent established a two-way sync. Starting with API9 the syntax is `aLink: this.aState` to init a variable `aLink` in child component from variable `aState` in parent component. Also `aLink: this.$aState` and `aLink: $aState` syntax is supported, but it is depreciated and should not be used anymore. For rules what parent component source variable decorator can init a @Link see [section 3.2.9](intro-state-mgmt.md#329-provisions-for-variable-initialization-from-the-parent), and [section 3.2.10](intro-state-mgmt.md#3210-provisions-for-variable-update-from-the-parent)  |
| Can initialize and update sub-components | permissible, see [section 3.2.8](intro-state-mgmt.md#328-provisions-for-variable-initialization-from-the-parent) and [section 3.2.9](intro-state-mgmt.md#329-provisions-for-variable-update-from-the-parent) for details |
| access | A `@Link` variable is private, accessible only within the component | 

An `@Link` decorated variable share the lifecycle of their owning component.

To understand the `@Link` variable value initialization and update mechanism, it is necessary to consider the parent component and the child component that owns the `@Link` variable.

Initial render: The execution of the parent component's `build` function creates a new instance of the child component (child component's first render). The initialization is the following:
* An @State or @Link decorated variable of the parent must be specified to initialize the child's @Link variable. The child's @Link variable value and its source variable are kept in sync (two-way data synchronization).

Update of the `@Link` source: The execution of the parent component's `build` function updates an existing instance of the child component (child component is said to 're-render'). The child's `@Link` variable value is determined:
* The child's `@Link` variable is updated via the named parameter mechanism.

Update of the `@Link`: The child's @Link variable is set a new value. Processing steps:
1. The source @State or @Link variable value of the parent is updated.
2. The parent component re-renders. No variable update via the named parameter mechanism is needed, the child's @Link is up-to-date.

### 4.3.2 Example for `@Link` with simple and with class types

The following example is for `@Link` of both simple type and class type. Compared to the opening example, the new functionality is inside `GreenButton`. The UI framework observes both object property and object value mutations of an `@Link` variable.

```typescript
class GreenButtonState {
    width: number = 0;

    constructor(width:number) {
        this.width = width;
    }
}

@Component
struct GreenButton {
    @Link greenButtonState: GreenButtonState;

    build() {
        Button("Green Button")
        .width(this.greenButtonState.width)
        .height(150.0)
        .backgroundColor("#00ff00")
        .onClick(() => {
            if (this.greenButtonState.width < 700) {
                this.greenButtonState.width += 125; // update variable's object property, change is observed
            } else {
                this.greenButtonState = new GreenButtonState(100);  // update variable with new class object, change is observed
            }
            console.log("onClick handler on GreenButton, updated value this.greenButtonState.width: " + this.greenButtonState.width);
        })
    }
}

@Component
struct RedButton {
    @State redButtonState: number = 100;

    build() {
        Button("Red Button")
        .width(this.redButtonState)
        .height(150.0)
        .backgroundColor("#ff0000")
        .onClick(() => {
            this.redButtonState = (this.redButtonState < 700) ? this.redButtonState + 80 : 100;
            console.log("onClick handler on RedButton, updated value this.redButtonState: " + this.redButtonState);
        })
    }
}

@Component
struct YellowButton {
    @Prop yellowButtonState: number;

    build(){
        Button("Yellow Button")
        .width(this.yellowButtonState)
        .height(150.0)
        .backgroundColor("#ffff00")
        .onClick(() => {
            this.yellowButtonState += 50.0;
            console.log("onClick handler on YellowButton, updated value this.yellowButtonState: " + this.yellowButtonState);
        })
    }
}

@Entry
@Component
struct ShufflingContainer {
    @State shuffle: boolean = false;
    @State greenButtonState: GreenButtonState = new GreenButtonState(300);
    @State yellowButtonProp: number = 100;

    build() {
        Column(){
           Button(`Parent View: ${this.shuffle ? 'Shuffle to Red before Green' : 'Shuffle to Green before Red'}`)
            .width(700.0)
            .height(150.0)
            .onClick(() => {
                this.shuffle = !this.shuffle;
                console.log("onClick handler on ShufflingContainer, updated value this.shuffle: " + this.shuffle);
            })

            Button("Parent View: Set yellowButtonProp")
            .width(700.0)
            .height(150.0)
            .onClick(() => {
                this.yellowButtonProp = (this.yellowButtonProp < 700) ? this.yellowButtonProp + 100 : 100;
                console.log("onClick handler on ShufflingContainer, updated value of yellowButtonProp: " + this.yellowButtonProp);
            })

            if(this.shuffle) {
                GreenButton({ greenButtonState: this.greenButtonState })
                RedButton()
                YellowButton({ yellowButtonState: this.yellowButtonProp })
            } else {
                RedButton()
                YellowButton({ yellowButtonState: this.yellowButtonProp })
                GreenButton({ greenButtonState: this.greenButtonState })
            }
        }
    }
}
```

### 4.3.3 Example for `@Link` with Array type

Another example with `@Link` of `Array<number>` value type:

```typescript
@Component
struct Child {
    @Link items: number[];
    build() {
        Column() {
            Button() {
                Text("Button1: push")
            }.onClick(() => {
                this.items.push(this.items.length+1);
            })
            Button() {
                Text("Button2: replace whole item")
            }.onClick(() => {
                this.items = [100, 200, 300];
            })
        }
    }
}

@Entry
@Component
struct Parent {
    @State arr: number[] = [1, 2, 3];
    build() {
        Column() {
            Child({ items: this.arr })
            ForEach(this.arr,
                item => {
                    Text(`${item}`)
                },
                item => item.toString()
            )
        }
    }
}
```

The framework observed adding, deleting and replacing array items.

It is  important to note that the variable type of the `@Link` and `@State` variables is the same: `number[]` in above example.  It is for instance _not_ permissible to define the `@Link` variable in Child as type `number` (`@Link item : number`), and create Child components for each array item like below. `@Prop` or `@Observed` should be used depending on application semantics.


## 4.4 Bi-directionally syncing state with descendent components `@Provide` and `@Consume`


An `@Provide` decorated state variable  works the same as an `@State` decorated variable with the following additional features:

This state variable becomes available to all descendent components of providing component automatically. The variable is said to be 'provided' to other components.  The convenience advantage of using @Provide is therefore that developer does not need to pass a variable from component to component multiple times.

An descendent component gains access to provided state variable by decorating a TS variable with `@Consume`. This establishes a two-way data sync between the provided and the consumed  variable. This sync works the same as combination of `@State` and `@Link` does. The only difference is that the UI framework can make the connection across multiple levels of the UI parent-child hierarchy.

```Typescript
@Component
struct CompD {

    @Consume reviewVotes : number;
    build() {
        Column() {
            Text(`reviewVotes(${this.reviewVotes})`)
            Button(`reviewVotes(${this.reviewVotes}), give +1`)
                .onClick(() => this.reviewVotes += 1 )
        }
        .border({ width: 3, color: Color.Black })
        .margin(5)
    }
}

@Component
struct CompC {

    build() {
        Row({ space: 5 }) {
            CompD()
            CompD()
        }
    }
}

@Component
struct CompB {

    build() {
        CompC()
    }
}

@Entry
@Component
struct CompA {
    @Provide reviewVotes : number = 0;

    build() {
        Column() {
            Button(`reviewVotes(${this.reviewVotes}), give +1`)
               .onClick(() => this.reviewVotes += 1)
            CompB()
        }
    }
}
```

In above example the variable `reviewVotes` is provided by the entry component `CompA` to all other components in the UI tree. To provide it its variable reviewVotes is decorated by `@Provide`. In order to sync with the provided variable `CompA` and `CompC` use the `@Consume` decorator.


### 4.4.1 Provisions for using `@Provide` and `@Consume`

***`@Provide` decorator***

Same rules apply for `@Provide` as for `@State`. `@Provide` difference is that it functions as sync source also for further descendant (child of child) components:

|   | Comment |
|---|---|
| custom component variable decorator | |  
| decorator parameters | alias: constant string, optional to specify (note, the string is not quoted, see example). If the alias is specified, the variable is provided under the alias name only. If the alias is not specified the variable is provided under the variable name. |
| type of sync | two-way: from the provided variable to all consumed variables (see @Consume), and the opposite direction. Same two-way sync behaviour as combination of `@State` and `@Link`. |
| permissible variable types and observed changes | See [section 3.2.5](intro-state-mgmt.md#325-variable-types-and-observed-changes-for-variables-decorated-with--state-provide-link-consume-or-prop) |
| local initialization | mandatory |
| initialization from parent component | Optional. see  see [section 3.2.9](intro-state-mgmt.md#329-provisions-for-variable-initialization-from-the-parent)  |
| Update from parent | None |
| Sync with descendant component | two-way with `@Consume` variable in ancestor. Can not initialize or update sub-components otherwise. |
| access | A @Provide variable is private, accessible only within the component | 

The rules for initialization and update of a @Provide variable are the same as those for @State.

***`@Consume` decorator***

|   | Comment |
|---|---|
| custom component variable decorator | |  
| decorator parameters | alias: constant string, optional to specify (note, the string is not quoted, see example). If the alias is specified, a variable must be provided under this name to match. Otherwise it is the @Consume variable name that needs to match. |
| type of sync | two-way: from the provided variable (See @Provide) to the @Consume decorated variable, and the opposite direction. Same two-way sync behaviour as combination of @State and @Link. |
| permissible variable types and observed changes | See [section 3.2.5](intro-state-mgmt.md#325-variable-types-and-observed-changes-for-variables-decorated-with--state-provide-link-consume-or-prop) |
| local initialization | forbidden |
| initialization from parent component | forbidden |
| Sync with ancestor component | two-way with `@Provide` variable in ancestor  |
| Can initialize and update sub-components | permissible, see [section 3.2.8](intro-state-mgmt.md#328-provisions-for-variable-initialization-from-the-parent) and [section 3.2.9](intro-state-mgmt.md#329-provisions-for-variable-update-from-the-parent) for details |
| access | A @Consume variable is private, accessible only within the component | 

Further clarification on the use of `@Provide` and  `@Consume`:
* An `@Provide` decorated variable can have an additional `@Watch` decorator, but no other additional decorator.
* The same component can provide several variables. The same component can consume multiple provided variables.
* A component providing two variables under the same name is a syntax error.
* A component providing a variable under the same name as its parent, or parent of parent (ancestor) is a syntax error.
* A component consuming a variable that neither its parent nor parent of parent (ancestors) provide under stated name is also a syntax error. 
* These syntax errors should be reported by the compiler.

The rules for initialization and update of a @Consume variable are the same as those for @Link, with the exception of how the source variable is determined amongst the component's ancestors.

## 4.5 Object reference with `@ObjectLink`

### 4.5.1 Introduction to usage scenarios for `@ObjectLink`

`@ObjectLink` variable decorator only supports object type variables, not simple type number, boolean, or string.
`@ObjectLink` needs to be initialized from variable in parent component, we call this its source variable. The source variable can be `@State`, `@Link`, `@Prop`, or `@ObjectLink` decorated.

`@ObjectLink` variable can not be assigned a new value
* `this.objectLink = new ClassA(..)` is not possible

`@ObjectLink` variable object property value changes are possible
* `this.objectLink.a++` is possible.

`ObjectLink` can be used in two scenarios. 
1. Reference to source: `@ObjectLink` variable and its source variable of parent component both refer to the same object. 
2. Reference to array item or object property of source: 
    * `@ObjectLink` source variable in parent component is of type `Array<T extends Object>`. `@ObjectLink` includes a reference to an _array item_, the @ObectLink variable is of type `T extends Object`.
   * `@ObjectLink` source variable is an object but not an array. The object includes a property `aProp: T extends Object`. `@ObjectLink` includes a reference to that property value, the `@ObjectLink` variable is of type `T extends Object`.

The following example is to show that the same child component with an `@Observed` decorated variable can support both scenarios:
* `@ObjectLink` reference object of source variable (scenario #1), and
* `@ObjectLink` reference object which is an item in array type source variable (or value of an object property)

```TypeScript
@Observed
class ClassA {
    public a: number;
    constructor(a?: number) {
        this.a = a ? a : 0;
    }
}

@Component
struct Child {
    @ObjectLink objLink: ClassA;

    build() {
        Row({ space: 20 }) {
            Text(`objLink: ${JSON.stringify(this.objLink)}`)
            Button(`a++`)
                .onClick(() => { this.objLink.a++; })
        }
        .border({ width: 3, color: Color.Red })
    }
}

@Entry
@Component
struct Parent {
    @State arr: ClassA[] = [new ClassA(123), new ClassA(222), new ClassA(300)];
    @State single: ClassA = new ClassA(1);

    build() {
        Column({ space: 20 }) {
            Row({ space: 10 }) {
                Text(`arr: ${JSON.stringify(this.arr)}`)
                Button(`arr[1].a++`)
                .onClick(() => { this.arr[1].a++; })
                Button(`assign`)
                .onClick(() => { this.arr[1] = new ClassA(10*this.arr[1].a); })
            }

            Row({ space: 10 }) {
                Text(`single: ${JSON.stringify(this.single)}`)
                Button(`single.a++`)
                .onClick(() => { this.single.a++; })
                Button(`assign`)
                .onClick(() => { this.single = new ClassA(10*this.single.a); })
            }
            Child({ objLink: this.arr[1] })
            Child({ objLink: this.single})
        }
        .width('100%')
    }
}
```

The key lines of source code for this showcase are the following:

`@Observed` class decoration
```TypeScript
    @Observed
    class ClassA {
```

In the parent component:
```TypeScript
    @State arr: ClassA[] = [new ClassA(123), new ClassA(222), new ClassA(300)];
    @State single: ClassA = new ClassA(1);
```

And inside its `build` function:
* the first `Child` component initialisation (and update) is according to scenario #2 array of objects, `@ObjectLink` having a reference to array item index 1.
* the second `Child` component initialization  (and update) showcases scenario #1, `@ObjectLink` having a reference to the same object as the @State in the parent.

```TypeScript
    Child({ objLink: this.arr[1] })
    Child({ objLink: this.single})
```

This initializes and updates an `@ObjectLink` variable in the child component:
```TypeScript
    @ObjectLink objLink: ClassA;
```

We will discuss scenario #1 in more detail in sections 4.5.2 first, followed by discussion of scenario #2 from section 4.5.3 .

### 4.5.2 `@ObjectLink` variable and its source variable of parent component both refer to the same object


The following example is to compare `@ObjectLink`, `@Prop`, and `@Link` sync behaviour.

```TypeScript
@Observed
class ClassA {
    public a: number;
    constructor(a?: number) {
        this.a = a ? a : 0;
    }
}

@Component
struct Child {
    @Link link: ClassA;
    @Prop prop: ClassA;
    @ObjectLink objLink: ClassA;

    build() {
        Column({ space: 10 }) {
            Row({ space: 10 }) {
                Text(`link: ${JSON.stringify(this.link)}`)
                Button(`a++`)
                .onClick(() => { this.link.a++; })
                Button(`assign`)
                .onClick(() => { this.link = new ClassA(10*this.link.a); })
            }
            Row({ space: 10 }) {
                Text(`prop: ${JSON.stringify(this.prop)}`)
                Button(`a++`)
                .onClick(() => { this.prop.a++; })
                Button(`assign`)
                .onClick(() => { this.prop = new ClassA(10*this.prop.a); })
            }
            Row({ space: 10 }) {
                Text(`objLink: ${JSON.stringify(this.objLink)}`)
                Button(`a++`)
                .onClick(() => { this.objLink.a++; })
                Text(`assign not possible`)
            }
        }
        .border({ width: 3, color: Color.Red })
    }
}

@Entry
@Component
struct Parent {
    @State parent: ClassA = new ClassA(123);

    build() {
        Column({ space: 20 }) {
            Row({ space: 10 }) {
                Text(`parent: ${JSON.stringify(this.parent)}`)
                Button(`a++`)
                .onClick(() => { this.parent.a++; })
                Button(`assign`)
                .onClick(() => { this.parent = new ClassA(10*this.parent.a); })
            }
            Child({ link: this.$parent, prop: this.parent, objLink: this.parent })
        }
        .width('100%')
    }
}
```

Sync from decorated variable in child to `@State` in parent component

| type of change |  `@Link` | `@ObjectLink` |`@Prop` |
|--|--|--|--|
| assign new object<br/>`this.obj = new ClassA(..)` | local change, and <br> updates @State value | assignment of new object not possible | local change only |
| change property value <br/> `this.obj.a++` | local change, and <br> updates `@State` value | local change, and <br> updates `@State` value | local change only |

When to use  `@Link`, `@Prop`, or `@ObjectLink` decorator?

| characteristic |  `@Link` | `@ObjectLink` | `@Prop` |
|--|--|--|--|
| child component only reads value, makes no changes | can use | recommended | do not use |
| child component assigns new object, source in parent component should change  | must use | does not support | does not support |
| child component makes object property changes only, source in parent component should change | can use | recommended | does not support |
| child component makes local changes, changes must not affect source in parent Component | does not support | does not support | must use |
| performance of initialization and update from source in parent | same for `@ObjectLink` and `@Link` | same for `@ObjectLink` and `@Link` | need to deep copy object, slower than  `@ObjectLink` and `@Link` |


## 4.5.3 Observing Nested Class Object Property Changes `@Observed` and `@ObjectLink` - Introduction

`@Observed` was already mentioned in connection with `@Prop` variable. An `@Prop` decorated variable of type class object inside a sub-view whose source in parent component is an item of an array or a property of an object. That array or object must be decorated with `@State`, `@Link`.   `@Prop` creates a one way sync from said source to the decorated variable.

`@ObjectLink` also decorates a class object type variable. It has the same requirements for its source: said array item or object property. - An `@ObjectLink` variable takes no local copy of its source. The difference of `@ObjectLink` are therefore two-fold:
- an `@ObjectLink` variable can not be assigned a new object, i.e. `this.myObjectLink = new MyObject()` is not supported.
- an `@ObjectLink` variable allows object property changes. These reflect in the source.
For C/C++ developers, an `@ObjectLink` can be understood like a pointer to the source object inside the parent component. 

### 4.5.4 `@ObjectLink` and `@Observed` Class Decorator Scenario Nested Objects

Example (`objectLinkNestedObjects.ets`):

Consider the following data structure of nested class objects:

```typescript

let NextID  : number= 1;

@Observed 
class ClassA {
    static 
    public id : number;
    public c: number;

    constructor(c: number) {
        this.id = NextID++;
        this.c = c;
    }
}

@Observed 
class ClassB {
    public a: ClassA;

    constructor(a: ClassA) {
        this.a = a;
    }
}
```

The following component hierarchy renders this data structure. 

```typescript

@Component
struct ViewA {

  @ObjectLink a : ClassA;
  label : string = "ViewA1";

  build() {
     Row() {
        Button(`ViewA [${this.label}] this.a.c=${this.a.c} +1`)
        .onClick(() => {
            this.a.c += 1;
        })
     }
  }
}

@Entry
@Component 
struct ViewB {
  @State b : ClassB = new ClassB(new ClassA(0));

  build() {
     Column() {
       ViewA({ label: "ViewA #1", a: this.b.a })
       ViewA({ label: "ViewA #2", a: this.b.a })

       Button(`ViewB: this.b.a.c+= 1`)
        .onClick(() => {
            this.b.a.c +=1;
        })  
        Button(`ViewB: this.b.a = new ClassA(0)`)
        .onClick(() => {
            this.b.a = new ClassA(0);
        })  
        Button(`ViewB: this.b = new ClassB(ClassA(0))`)
        .onClick(() => {
            this.b = new ClassB(new ClassA(0));
        }) 
    }
  } 
}
```

Lets review the app state changes inside event handlers of `ViewB`:
*  will update both instances of `ViewA`, and this in turn will update its `@ObjectLink a`, thereby update the dependent Button label.
* assign `this.b.a = ...` - this is also observed as a change of the `@State b`, same update change as above.
* assign `this.b.a.c = ...` - this is _not_ observed as a change of the `@State b`, because c is a property of nested object of `ClassA`. This is where the opaque proxy wrapped around the `ClassA` object (caused by the `@Observed` class decorator as mentioned in connection with `@Prop` already) detects the property `c` change and marks  as having changed.

And also the event handler inside `ViewA`:
* assign `this.a.c` - this is a change of `@ObjectLink a` which causes the Button label to be updated. `@ObjectLink` - alike `@Prop` does not take a copy of its source - but it is like a 'C/C++ pointer' to the ClassA object of its source. Therefore, its the same assignment as `this.b.a.c = ...` above. Also the other instance of `ViewA` updates.
* assign `this.a = new ClassA(...)` is _not_ possible, `@ObjectLink`variables are read-only.


### 4.5.5 `@ObjectLink` and `@Observed` class decorator Scenario Array of Objects

Array of object is a frequently used data structure. 

Example (`objectLinkArrayOfObject2.ets`):

This example is about an `@State` decorated Array of `ClassA` objects in `ViewB`. A sub-view has an `@ObjectLink` to a `ClassA` object, where there are two `ViewA` instances that share the same `ClassA` object.

```TypeScript
let NextID  : number= 1;

@Observed 
class ClassA {
    static 
    public id : number;
    public c: number;

    constructor(c: number) {
        this.id = NextID++;
        this.c = c;
    }
}
@Component
struct ViewA {

  @ObjectLink a: ClassA;
  label : string = "ViewA1";

  build() {
     Row() {
        Button(`ViewA [${this.label}] this.a.c= ${this.a.c} +1`)
        .onClick(() => {
            this.a.c += 1;
        })
     }
  }
}

@Entry
@Component 
struct ViewB {

  @State arrA : ClassA[] = [ new ClassA(0), new ClassA(0) ];

  build() {
     Column() {
        ForEach (this.arrA,
            (item) => {
                ViewA({ label: `#${item.id}`, a: item })
            },
            (item) => item.id.toString()
        )

        ViewA({ label: `ViewA this.arrA[first]`, a: this.arrA[0] })
        ViewA({ label: `ViewA this.arrA[last]`, a: this.arrA[this.arrA.length-1] })

        Button(`ViewB: reset array`)
            .onClick(() => {
                this.arrA = [ new ClassA(0), new ClassA(0) ];
            })  
            Button(`ViewB: push`)
            .onClick(() => {
                this.arrA.push(new ClassA(0))
            })
            Button(`ViewB: shift`)
            .onClick(() => {
                this.arrA.shift()
            })  
            Button(`ViewB: chg item property in middle`)
            .onClick(() => {
                this.arrA[Math.floor(this.arrA.length/2)].c = 10;
            })
            Button(`ViewB: chg item property in middle`)
            .onClick(() => {
                this.arrA[Math.floor(this.arrA.length/2)] = new ClassA(11);
            })
        }
  }
}
```

Lets again look at the app state changes made in event handlers:
* `this.arrA[Math.floor(this.arrA.length/2)] = new ClassA(..)` - this change is observed by `@State a` causes three updates
   1. `ForEach`: the newly `ClassA` object has a changed `id` therefore the array item is recognized as changed, the ForEach item builder will execute, create a new `ViewA` component instance. 
   2. The two `ViewA` instances below that use an array `[ ]` operator on `@State a`.
*  `this.arrA.push(new ClassA(0))` -  - this change is observed by `@State a` causes again three  updates but with different effect:
    1. `ForEach`: the newly added `ClassA` object has a unknown id. The item builder function is executed ....
    2. The first `ViewA` instances below that use an array `[ ]` operator on `@State a`: The array has changed, but the first array item is unchanged, therefore that `@ObjectLink` 'pointer' value is unchanged. 
   3. The second `ViewA` instances below: The last array item has changed, therefore that `@ObjectLink` 'pointer' value changes. 
* `this.arrA[Math.floor(this.arrA.length/2)].c` - this is a property change of nested `ClassA` object. The change is handled by `@ObjectLink`.

### 4.5.6 `@ObjectLink` and `@Observed` class decorator Scenario Array of Arrays

`@Observed` class decoration is required for the nested `Array`. The trick is to declare a class that extends from `Array`:
`class StringArray extends Array<String> {}` and create using `new StringArray()`. The use of the `new` operator is required for `@Observed` class decorator to work properly, because TS class decorators overwrite the constructor function.

```TypeScript
@Observed
class StringArray extends Array<String> {
}

@Component
struct ItemPage {
  @ObjectLink itemArr: StringArray;

  build() {
    Row() {
        Text("ItemPage")
            .width(100).height(100)

        ForEach(this.itemArr, 
            item => {
                Text(item)
                    .width(100).height(100)
            },
            item => item
        )
    }
  }
}

@Entry
@Component
struct IndexPage {
  @State arr: Array<StringArray> = [ new StringArray(), new StringArray(), new StringArray() ];

  build() {
    Column() {
      ItemPage({ itemArr: this.arr[0] })
      ItemPage({ itemArr: this.arr[1] })
      ItemPage({ itemArr: this.arr[2] })
      
      Divider()

      ForEach(this.arr,
        itemArr => {
            ItemPage({ itemArr: itemArr })
        },
        itemArr => itemArr[0]
      )

      Divider()

      Button("update")
        .onClick(() => {
            console.error("Update all items in arr");
            if (this.arr[0][0] != undefined) {
                // we should have a real id to use with ForEach, but we do no
                // so need to make sure the pushed strings are unique.
                this.arr[0].push(`${this.arr[0].slice(-1).pop()}${this.arr[0].slice(-1).pop()}`);
                this.arr[1].push(`${this.arr[1].slice(-1).pop()}${this.arr[1].slice(-1).pop()}`);
                this.arr[2].push(`${this.arr[2].slice(-1).pop()}${this.arr[2].slice(-1).pop()}`);
            } else {
                this.arr[0].push("Hello");
                this.arr[1].push("World");
                this.arr[2].push("!");
            }
        })
    }
  }
}
```

The update scenario:
* `this.arr[...].push(...) ` - observed change of `@ObjectLink itemArr` triggers update of  `ForEach` in `ItemPage` , it detects the added array item by the newly added item id.


### 4.5.7 `@ObjectLink` and `@Observed` class decorator Scenario arbitrary deep nesting

The following _advanced example_ shows the full power of `@ObjectLink` and `@Observed`, how it can handle arbitrarily deep nested structures of objects and arrays. The example is a simplified version of Dynamicview, a solution for dynamically creating any ArkUI component from  given view model.

file `/lib/dynamicView.ets`

`DVModel` class describes UI compontent, its attributes and events, and its children.
Each child is describes by an `DVModel` object, hence, the structure is recursive.

```TypeScript
/**
 * Dynamic View creation
 * from a recursive data structure
 *
 * exported @Component: DynamicView
 * exported view model classes:
 * - DVModelContainer
 * - DVModel
 * - DVModelParameters
 * - DVModelEvents
 * - DVModelChildren
 *
 */

/**
* View Model classes 
*/

@Observed
class export DVModelParameters extends Object {
  /* empty, just get any instance wrapped inside an ObservedObject
  with the help of the decoration */
}

@Observed
class DVModelEvents extends Object {
  /* empty, just get any instance wrapped inside an ObservedObject
  with the help of the decoration */
}


@Observed
class DVModelChildren extends Array<DVModel> {
  /* empty, just get any instance wrapped inside an ObservedObject
  with the help of the decoration */
}

let nextId : number = 1;

@Observed export class DVModel {
  id_ : number;
  compType : string;
  params   : DVModelParameters;
  events   : DVModelEvents;
  children : DVModelChildren;

  constructor(compType : string, params : DVModelParameters, events : DVModelEvents, children: DVModelChildren) {
    this.id_ = nextId++;
    this.compType = compType;
    this.params = params;
    this.events = events;
    this.children = children;
  }
}

// includes the root DVModel objects.
export class DVModelContainer {
  model : DVModel;

  constructor(model : DVModel) {
    this.model = model;
  }
}
```

`DynamicView` is the @Component that does all the work:

The following 4 features are the key solution elements for dynamic View 
construction and update:

1. The if statement decides which framework component to create.
  We can not use a factory function here, because that would requite calling 
  a regular function inside build() or a @Builder function.

2. Take note of the @Builder for Row, Column containers:
  These functions create DynamicView Views inside a DynamicView 
  view. This behaviour is why we talk about DynamicView as a 'recursive' View.
  All @Builder functions are member functions of the DynamicView @Component to 
  retain access ('this.xyz') to its decorated state variables.
  
3. The @Extend functions execute attribute and event handler registration functions 
  for all attributes and events permissable on the framework component, irrespective 
  if DVModelParameters or DVModelEvents objects includes a value or not. If not
  the attribute or event is set to 'undefined' by intention. This is required to unset
  any previously set value.

4. The scope ('this') of any lambda registered as an event hander function, e.g. for onClick, 
  is the @Component, in which the DVModel object is initialized. This said, it is advised to initialize 
  the DVModel object in the @Component that is parent to outmost DynamicView. Thereby,
  any event handler function is able to mutate decorated state variables of that @Component


file `/lib/dynamicView.ets` continued:

```TypeSCript
@Extend(Text) 
function text_attrs(params: DVModelParameters, events : DVModelEvents) {
  .width(params["width"])  
  .height(params["height"])  
  .backgroundColor(params["backgroundColor"])
  .onClick(function() {
    let self = this;
    return events["onClick"];
  }() )
  .fontColor(params["fontColor"], "a")
}

@Extend(Image) 
function image_attrs(params: DVModelParameters, events : DVModelEvents) {
  .width(params["width"])  
  .height(params["height"])  
  .backgroundColor(params["backgroundColor"])
  .onClick(function() {
    let self = this;
    return events["onClick"];
  }() )
}

@Extend(Button) 
function button_attrs(params: DVModelParameters, events : DVModelEvents) {
  .width(params["width"])  
  .height(params["height"])  
  .backgroundColor(params["backgroundColor"])
  .onClick(function() {
    let self = this;
    return events["onClick"];
  }() )
}

@Extend(Column) 
function column_attrs(params: DVModelParameters, events : DVModelEvents) {
  .width(params["width"])  
  .height(params["height"])  
  .backgroundColor(params["backgroundColor"])
  .onClick(function() {
    let self = this;
    return events["onClick"];
  }() )
}

@Extend(Row) 
function row_attrs(params: DVModelParameters, events : DVModelEvents) {
  .width(params["width"])  
  .height(params["height"])  
  .backgroundColor(params["backgroundColor"])
  .onClick(function() {
    let self = this;
    return events["onClick"];
  }() )
}

@Component
export struct DynamicView {
    @ObjectLink model : DVModel;
    @ObjectLink children : DVModelChildren;
    @ObjectLink params : DVModelParameters;
    @ObjectLink events : DVModelEvents;

@Builder buildChildren() {
    ForEach(this.children,
      child => {
        DynamicView({ model: child, params: child.params, events: child.events, children: child.children })
      },
      child=>`${child.id_}`
    )
}

@Builder buildRow() {
    Row() {
      this.buildChildren()
    }
    .row_attrs(this.params, this.events)
}

@Builder buildColumn() {
    Column() {
      this.buildChildren()
  }
  .column_attrs(this.params, this.events)
}

@Builder buildText() {
    Text(`${this.params["value"]}`)
      .text_attrs(this.params, this.events)
}

@Builder buildImage() {
    Image(this.params["src"])
      .image_attrs(this.params, this.events)
}

// Button with label
@Builder buildButton() {
    Button(this.params["value"])
      .button_attrs(this.params, this.events)
}

    build(){
      if (this.model.compType=="Column") {
        this.buildColumn()
      } else if (this.model.compType=="Row") {
        this.buildRow()
      } else if (this.model.compType=="Text") {
        this.buildText()
    } else if (this.model.compType=="Image") {
        this.buildImage()
    } else if (this.model.compType=="Button") {
        this.buildButton()
    }
  }
}
```

Creating DVModel classes is permissible, but using a converter from JSON is more convenient:

file: `json/dynamicViewJson.ts`

```TypeScript
import { DVModel, DVModelParameters, DVModelEvents, DVModelChildren } from  "../lib/dynamicView.ets";

export function createDVModelFromJson(json : Object) : DVModel {

    /* private use helper functions */

    function createChildrenFrom(children : Array<Object>) : DVModelChildren {
        let result = new DVModelChildren();
        if (Array.isArray(children)) {
          (children as Array<Object>).forEach(child => { 
            const childView = createDVModelFromJson(child);
            if (childView != undefined) {
              result.push(childView); 
            }
          });
        }
        return result;
    }

    function createAttributesFrom(attributes : Object) : DVModelParameters {
      let result = new DVModelParameters();
      if ((typeof attributes == "object") && (!Array.isArray(attributes)))  {
        Object.keys(attributes).forEach(k => result[k]=attributes[k]);
      }
      return result;
    }

    function createEventsFrom(events : Object) : DVModelEvents {
      let result = new DVModelEvents();
      if ((typeof events== "object") && (!Array.isArray(events)))  {
        Object.keys(events).forEach(k => result[k]=events[k]);
      }
      return result;
    }

   if (typeof json !== 'object') {
    console.error("createDVModelFromJson: input is not JSON");
    return undefined;
   }
   
   return new DVModel(
      json["compType"],
      createAttributesFrom(json["attributes"]),
      createEventsFrom(json["events"]),
      createChildrenFrom(json["children"])
   );
}
```

How to use the converter is best illustrated with a short example:
Note here the use of @State in `DynamicContainerView ` and the initialisation of multiple @ObjectLink 
variables in `DynamicView`: `this.container.model : DVModel`, `this.container.model.params : DVModelParameters` and 
` this.container.model.events : DVModeEvents`, `this.container.model.children : DVModelChildren`. 
The use of multiple @ObjectLink variables enables the `DynamicView` component to take care of UI updates when one of these object changes.
If `DynamicView` just used `@ObjectLink DVModel` it could not update when children, events, or attributes 
nested objects / array change.

```TypeScript
@Entry
@Component
struct DynamicContainerView {
   
  private model : DVModel = createDVModelFromJson(
      {
          compType: "Text", 
          attributes: { value: "Hello, World!", height: "100%", width: "100%" },
          events: { 
            onClick: (evt) => { 
              console.log("onClick changes the language");
              this.container.model.params["value"] 
                = this.container.model.params["value"] === "Hello, World!"
                  ? "你好" 
                  : "Hello, World!";
            }
          }
      }
    );

  @State container : DVModelContainer = new DVModelContainer(this.model);

  build(){
    Column() {
      DynamicView({model: this.container.model, params: this.container.model.params, events: this.container.model.events, children: this.container.model.children })
    }
      .width(300).height(300)
  }
}

```
### 4.5.8 Provisions for using `@Observed` and `@ObjectLink`

 **`@Observed` decorator class decorator** 

`@Observed` decorates a class, i.e. it must be prepended to the class definition.

| `@Observed` decorator class decorator | Comment |
|---|---|
| decorator parameters | none |
| class decorator      | decorates an ES6 / TS class. Must use `new` to create a class object. |

Implementation notes: 
1. In TypeScript a class decorator is implemented by replacing the class' constructor function. The `@Observed` framework `Observed` function subclasses the decorated class for the purpose of wrapping all class instances inside an ES6 Proxy. The Proxy observed object property changes and notifies the change to the `@ObjectLink` or `@Prop` observed variable implementation. All those property changes are observed that ES6 Proxy can observe. Those are changes of properties returned by `Object.keys(proxiesObj)`.  
2. Chaining `Observed` with other class decorators onto the same class might cause issues.
3. The `@Observed` decorator modifies the class prototype chain.

 **`@ObjectLink` variable decorator** 

An `@ObjectLink` decorated variable must be of object type. Simple type variables are not supported. Use `@Prop` instead

| `@ObjectLink` variable decorator  | Comment |
|---|---|
| custom component variable decorator | |  
| decorator parameters | none |
| permissible variable types and observed changed | Object types, `undefined`, `null`. No simple types. See below table |
| special limitation | object property changes are allowed, these sync to sync peer. New value assignment is not possible. |
| local initialization | forbidden |
| initialization from parent component | mandatory to specify. The referenced source object must be of the same class as the decorated variable. Two options<br/>
1. The `@State`, `@Link`, `@Provide`, `@Consume` or `@ObjectLink` decorated variable in parent component and the `@ObjectLink` refer to the sme object. That especially means the type of both variables must be the same.<br/>
2. nested class instance, class is decorated with `@Observed`<br/>
    * The referenced object must be an array item. The outer array must be a `@State`, `@Link`, `@Provide`, `@Consume` or `@ObjectLink` decorated variable. <br> 
    * The referenced object must be an object property value. The outer object must be a `@State`, `@Link`, `@Provide`, `@Consume` or `@ObjectLink` decorated variable. <br/>See [section 3.2.9](intro-state-mgmt.md#329-provisions-for-variable-initialization-from-the-parent) and [section 3.2.10](intro-state-mgmt.md#3210-provisions-for-variable-update-from-the-parent) |
| Sync with source object | object property changes sync two-way. An object assignment in the parent component updates the `@ObjectLink` decorated variable in the child component. |
| Can initialize sub-components | permissible |


| `@ObjectLink` permissable type | observed value changes | 
|--------------------------------|------------------------|
| Application-defined ES6 class decorated with `@Observed`. | Assign a new class instance. <br/> value assignment to object property for all those properties returned by `Object.keys(classObject)` Note: this requirement means member variables defined to have a `get` and `set` function are not observed. The background to this restriction is that the ArkUI framework uses ES6 internally to observe object property changes.  |
| `@Observed` decorated class extending from `Date`  | Same as general ES6 class and in addition: <br/> Set a new `Date` value using methods `setFullYear`, `setMonth`, `setDate`, `setHours`, `setMinutes`, `setSeconds`, `setMilliseconds`, `setTime`, `setUTCFullYear`, `setUTCMonth`, `setUTCDate`, `setUTCHours`, `setUTCMinutes`, `setUTCSeconds`, `setUTCMilliseconds` |
| `@Observed` decorated class extending from  `Map`  (API 10) | Same as general ES6 class and in addition: <br/> Modifications made to `Map` object internal storage by function `set`, `clear`, `delete` are observed. Note: `[]` operators must not be used as these do not modify the Map object's internal storage but treat the Map object like a regular Object. |
| `@Observed` decorated class extending from `Set` (API 10) | Same as general ES6 class and in addition: <br/> Modifications made to `Set` object internal storage by function `add`, `clear`, `delete` are observed. Note: `[]` operators must not be used, same argument as for Map. |
| `Observed` decorated class extending from `Array` | Same as general ES6 class and in addition: <br> Assign new array item value using `[]` operator. <br> Adding, removing, or updating array item with methods `copyWithin`, `fill`, `reverse`, `sort`, `splice`, `push`, `pop`, `shift`, `unshift`. |
| Instance of `SubscribableAbstract` (@Observed class decorator not needed) | Assign a new `SubscribableAbstract` object. <br/> All object property changes notified by the `SubscribableAbstract` object |

An `@ObjectLink` decorated variable accepts changes to its properties, e.g. `this.objLink.a= ..` is ok, but assignment is not allowed, e.g. `this.objLink=....` is not ok. (Note: We are thinking if and how to enable assignment, feedback welcome)

### 4.5.9 Avoid pitfall of union type sync source of `@ObjectLink`

`@ObjectLink` type must not be simple type.  App developers are advised to pay attention to sync source of an `@ObjectLink` that is of simple and object union type:
* Array of mixed simple and object union type items pose a risk of initializing or updating an @ObjectLink with a simple type value. 
* The same applies with an object property in parent used to init and update an `@ObjectLink`, if this property is of mixed simple and object union type.

Example how to avoid type mismatch:

```TypeScript
type MixedType = string | Resource;

function getResourceOrDefault(resourceName : string, defaultString : string ) : MixedType {
    return $r(resourceName) || defaultString;
}

@Component struct Parent {
     @State mixedArray : Array<MixedType> = [ getResourceOrDefault(...), getResourceOrDefault(...), getResourceOrDefault(...)];
     build() {
        Column() {
            ForEach(this.mixedArray,
              item => {
                Child({ objectLink: (item instance of Resource) ? item as Resource : undefined, simpleProp: (typeof item ==="string") ? item as string : undefined })
              }
            )
        }
     }
}

@Component struct Child {
    @ObjectLink objectLink : Resource;
    @Prop simpleProp : string;
    
    build() {
            Text(this.objectLink || this.simpleProp)
        }
    }
}
```

`ObjectLink` limitation to object types makes app design unnecessarily complex. Future versions of ArkUI may provide a solution.


## 4.6 Regular TypeScript component variable

A component can also have variables without any decorators, referred to as 'regular' TS variable.

| non-decorated  | Comment |
|---|---|
| permissible variable types | All TS types except `any` are allowed, specifying the type is recommended to give the compiler the possibility for optimizations that exploit fixed, known type. |
| observed changes | no observation |
| default value | recommended to specify |
| initialize and update via named parameter mechanism | optional |
| can use to initialize sub-components | allowed for regular variables and @State, @Provide variables. |
| update from parent component | none | 
| access | private, optional to specify. Specifying other access than private is a syntax error. | 

A regular variable, like all other variables, shares the lifecycle of their owning component.

When creating a new instance of the owning component (first render), the processing steps are the following to determine the @State variable value.
1. Apply the locally define default value (if defined).
2. Apply the named parameter value, if one is supplied. 
This process allows for an uninitialized variable.

When updating an existing instance of the owning component (re-render), the processing steps are the following to determine the regular variable value.
1. The variable value before re-render is maintained.
2. No update from parent


## 4.7 More examples

### 4.7.1 Decorated @Component variables of type `Map`

The following example shows how to use `Map` with appropriate decorators like `@State`, `@Link` and `@Prop`. Support for `Map` is release in API 10, as it requires deep copying. In the example a `Map<number, number>` is created and shared from Parent component to its LinkChild and PropChild components. User can insert new items from Parent component and experiment with clearing the `Map` from the child components.

Developers take note to always use `Map` functions `get` and `set`. Do not use the `[]` operator, ArkUI does not monitor 'get' and 'set' access to `Map` objects this way. The background is that the `[]` operator accesses JS object properties, not properties in the `Map`s own storage area, as [this MDN article explains](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map#setting_object_properties).

```typescript
@Component
struct LinkChild {
    @Link data: Map<number,number>;

    build() {
        Column() {
            Text(`@Link data items 1-3, actual size ${this.data.size}`)
            if (this.data.has(1)) {
                Text(`${this.data.get(1)}`)
            }
            if (this.data.has(2)) {
                Text(`${this.data.get(2)}`)
            }
            if (this.data.has(3)) {
                Text(`${this.data.get(3)}`)
            }

            Button("clear").width(300).height(70)
            .onClick(() => { 
                this.data.clear();
            })
        }
        .border({ color: Color.Blue, width: 3})
    }
}

@Component
struct PropChild {
    @Prop data: Map<number,number>;

    build() {
        Column() {
            Text(`@Prop data items 1-3, actual size ${this.data.size}`)
            if (this.data.has(1)) {
                Text(`${this.data.get(1)}`)
            }
            if (this.data.has(2)) {
                Text(`${this.data.get(2)}`)
            }
            if (this.data.has(3)) {
                Text(`${this.data.get(3)}`)
            }

            Button("clear").width(300).height(70)
            .onClick(() => { 
                this.data.clear(); 
            })
        }
        .border({ color: Color.Red, width: 3})
    }
}

let next = 4;

@Entry
@Component
struct Parent {
    @State data: Map<number,number> = new Map([ [1,1], [2,2] ]);

    build() {
        Column({ space: 10 }) {
            Text(`Parent data 1-3, actual size ${this.data.size}.`)
            if (this.data.has(1)) {
                Text(`${this.data.get(1)}`)
            }
            if (this.data.has(2)) {
                Text(`${this.data.get(2)}`)
            }
            if (this.data.has(3)) {
                Text(`${this.data.get(3)}`)
            }

            Button("set").width(300).height(70)
            .onClick(() => { 
                this.data.set(next, next++);
            })
            Button("delete 1").width(300).height(70)
            .onClick(() => { 
                if (this.data.has(1)) {
                    this.data.delete(1);
                }
            })
            Button("change 2").width(300).height(70)
            .onClick(() => { 
                if (this.data.has(2)) {
                    let val = this.data.get(2);
                    this.data.set(2, 10*val);
                }
            })

            LinkChild({ data: this.$data })
            PropChild({ data: this.data })
        }
    }
}
```

### 4.7.2 Decorated @Component variables of type `Set`

The following, a bit more complex example shows how to use `Set` with appropriate decorators like `@State`, `@Link` and `@Prop`. To help UI development, a new class `MySet` is extended from `Set` and some helper functions are defined there. `MySet` can be used to store any type `T`, but here we use user-defined class `ClassA`, that is also decorated with `@Observed` decorator.

```typescript
@Observed
class ClassA {
    public id: number;
    public msg: string;
    constructor(id?: number, msg?: string) {
        this.id = id ? id : next++;
        this.msg = msg ? msg : "";
    }
}

class MySet<T> extends Set<T> {
    constructor(...args: any[]) {
        super(...args);
    }
    get items(): T[] {
        let arr = [];
        for (const item of this) {
            arr.push(item);
        }
        return arr;
    }
}
```

Otherwise the example is quite similar to the `Map` example in the previous chapter. Parent component shares its `MySet` instance to LinkChild and PropChild components. As the `MySet` is decorated with `@Provide` in the Parent component, the ConsumeChild component finds the `MySet` and can use it. As `Set` stores object references, the example generates one single instance of `ClassA` that can be used for experimenting inserting and deleting an item from the `MySet`. 

```typescript
@Component
struct LinkChild {
    @Link single: ClassA;
    @Link data: MySet<ClassA>;

    build() {
        Column() {
            Text(`@Link data.size ${this.data.size}`)
            ForEach(this.data.items, item => {
                Text(`#${item.id} -- ${item.msg}`)
            },
                item => item.id
            )

            Button("clear").width(300).height(70)
            .onClick(() => { 
                this.data.clear();
            })
            Button("add single object").width(300).height(70)
            .onClick(() => { 
                this.data.add(this.single);
            })
        }
        .border({ color: Color.Blue, width: 3})
    }
}

@Component
struct ConsumeChild {
    @Consume data: MySet<ClassA>;

    build() {
        Column({space: 10}) {
            Text(`@Consume child finds following ids`)
            Row({ space: 3}) {
                ForEach(this.data.items, item => {
                    Text(`#${item.id}`)
                },
                    item => item.id
                )
            }
        }
    }
}

@Component
struct PropChild {
    @Prop data: MySet<ClassA>;

    build() {
        Column() {
            Text(`@Prop data.size ${this.data.size}`)
            ForEach(this.data.items, item => {
                Text(`#${item.id} -- ${item.msg}`)
            },
                item => item.id
            )

            ConsumeChild()

            Button("clear").width(300).height(70)
            .onClick(() => { 
                this.data.clear(); 
            })
        }
        .border({ color: Color.Red, width: 3})
    }
}

let next = 4;

@Entry
@Component
struct Parent {
    @State single: ClassA;
    @Provide data: MySet<ClassA> = new MySet([this.single, new ClassA(3, "33")]);

    build() {
      List() {
        Column({ space: 10 }) {
            Text(`Parent data.size ${this.data.size}`)
            ForEach(this.data.items, item => {
                Text(`#${item.id} -- ${item.msg}`)
            },
                item => item.id
            )

            Button("add new").width(300).height(70)
            .onClick(() => { 
                this.data.add(new ClassA(next, `this is item ${next}`));
                next++;
            })
            Button("delete single object").width(300).height(70)
            .onClick(() => { 
                if (this.data.has(this.single)) {
                    this.data.delete(this.single);
                }
            })

            Divider()

            LinkChild({ single: this.$single, data: this.$data })
            PropChild({ data: this.data })
        }
      }
    }
}
```

---

# 5 UI State Storages

## 5.1 Storage API for UI state `LocalStorage`

`LocalStorage` is a TS class of the framework.  Its purpose is to provide in-memory storage for mutable state properties that are required to construct a specific part of the application UI, e.g. the UI of one Ability.

An application can create multiple instances of `LocalStorage`, use each instance with parts of the UI, e.g. one `LocalStorage` instance for each Ability. An entry (top-level) `@Component`  can be assigned at most one instance of `LocalStorage`. All child instances of this custom component automatically gain access to the same `LocalStorage` instance.  A `@Component` has access to at most one `LocalStorage` instance and to central `AppStorage` (see next section). The same `LocalStorage` object can be assigned to multiple entry `@Component`s. 

The application decides about the life cycle of a `LocalStorage` object. The JS Engine will garbage collect a `LocalStorage` object when the app relates the last reference to it, which includes deleting the last custom component.
persist
### 5.1.1 Example for using `LocalStorage` from app logic

Example for using `LocalStorage` from app logic.

```typescript
let storage = new LocalStorage({"PropA": 47}); // create new instance and init with given object
var propA : SubscribedAbstractProperty<number> = storage.get<number>("PropA")         // propA == 47
var link1 : SubscribedAbstractProperty<number> = storage.link<number>("PropA");       // link1.get() == 47
var link2 : SubscribedAbstractProperty<number> = storage.link<number>("PropA");       // link2.get() == 47
var prop  : SubscribedAbstractProperty<number> = storage.prop<number>("PropA");       // prop.get() = 47

//
link1.set<number>(48);      // two-way sync: link1.get() == link2.get() == prop.get() == 48
prop.set<number>(1);        // one-way sync: prop.get()=1; but link1.get() == link2.get() == 48
// info about the type '<number>' can be skipped also
link1.set(49);             // two-way sync: link1.get() == link2.get() == prop.get() == 49

// before variables link1, link2, and prop go out of scope and will be deleted by GC 
// these variables need to terminate the sync relationship with LocalStorage that was 
// created by prop() and link() calls. This is the purpose of the 
// SubscribedAbstractProperty.aboutToBeDeleted() function
link1.aboutToBeDeleted();
link2.aboutToBeDeleted();
prop.aboutToBeDeleted();
```

### 5.1.2 Example for using `LocalStorage` from inside UI

The `@LocalStorageLink` variable decorator works together with the `LocalStorage` instance of this `@Component` or its ancestor. 
It creates two-way data sync with a property in `LocalStorage`. In the following example, the full spec can be found in the section about `@LocalStorageLink` decorator.

```typescript
// create new instance and init with given object
let storage = new LocalStorage({"PropA": 47});

@Component
struct Child {

    // @LocalStorageLink variable decorator creates two-way link with property "PropA" in LocalStorage
    @LocalStorageLink("PropA") storLink2 : number = 1;

    build() {
        Button(`Child from LocalStorage ${this.storLink2}`)
        // changes will sync with  "PropA" in LocalStorage and with Parent.storLink1
          .onClick(() => this.storLink2 += 1 )
    }
}

// make LocalStorage accessible from the @Component
@Entry(storage)
@Component
struct CompA {
   
    // @LocalStorageLink variable decorator creates two-way link with property "PropA" in LocalStorage
    @LocalStorageLink("PropA") storLink1 : number = 1;

    build() {
        Column({ space: 15 }) {
            Button(`Parent from LocalStorage ${this.storLink1}`)    // initial value from LocalStorage will be 47, because "PropA" initialized already
                 .onClick(() => this.storLink1 += 1 ) 
        
            // Child @Component receives access to CompA' LocalStorage instance automatically.
            Child() 
       }
   }
}
```

### 5.1.3 Provisions for using `LocalStorage` API

The API of `LocalStorage` is as follows. 

Type `T` can be class, number, boolean, string or Array of these types. Type `S` can be number, boolean, string (simple types only). Once created the type of a named property never changes.  Subsequent calls to `Set` must set a value of same type.

All properties in `LocalStorage` are mutable.

| Member Function | Parameter | Return value | Definition |
|------------|--------|--------|---------|
| `GetShared` | none | `LocalStorage` or `undefined` | Get the `LocalStorage` instance available in the current execution context if any. See example `Sharing a LocalStorage instance from Ability to one or several Views`. |
| constructor | obj? : Object | N/A | create a new `LocalStorage` object. Optionally initialize with the given Object. All object properties returned by `Object.keys(obj)` and their values will be added to `LocalStorage`. |
| `has` | key : String | boolean | Return true of the property with given name exists in `LocalStorage` |
| `get<T>`  | key : String | `T` or `undefined` | Get the property with the given name, if it exists |
| `set<T>` | key : String, <br> newValue : `T` | boolean | If a property with the given name exists set its value and return true. Otherwise don't set anything and return false. `newValue` must be of type `T` and must not be `undefined` or `null` |
| `setOrCreate<T>` | key : String, <br> newValue : `T` | boolean | if a property with given name exists: Update its value and return true. If no property with given name exists: Create a new property with given default value inside `LocalStorage`. `newValue` must be a of type `T`, `undefined` or `null` are not allowed. Return true. |
| `link<T>` | key : String | `SubscribedAbstractProperty<T>` \| `undefined` | If a property with given key exists, returns a two-way data binding to this property. Means value changes will be synched from the using variable (or Component) to the `LocalStorage` and from `LocalStorage` instance to any variable (or Component). The call returns `undefined` if a property with this key does not exist. The type can be any type `T`. The function has additional optional parameters that are currently reserved for framework internal use of `link`/`linkAndSet`. |
| `setAndLink<T>` | key : String <br> defaultValue: `T` | `SubscribedAbstractProperty<T>` | if a property with given key exists, determine the return value as defined for `link`. If the property does not exist, create a property with given default type and value `defaultValue` (see `setOrCreate`) and return a link to it (see `link`). `defaultValue` must be a of type `T`, `undefined` or `null` are not allowed. |
| `prop<S>` | key : String | `SubscribedAbstractProperty<S>` | if a property with given key exists, returns a one-way data binding to this property. Means value changes will be synched from `LocalStorage` to any variable (or Component). The property value must be of simple type `S`. The function has additional optional parameters that are currently reserved for framework internal use of `prop`/`setAndProp`.  br><br>From API rel. 10 onwards, in addition to types `S` also types `T` are supported.  |
| `prop<T>` since API rel. 10 | key : String | `SubscribedAbstractProperty<T>` | if a property with given key exists, returns a one-way data binding to this property. Means value changes will be synched from `LocalStorage` to any variable (or Component). The property value must be of simple type `S`. The function has additional optional parameters that are currently reserved for framework internal use of `prop`/`setAndProp`. |
| `setAndProp<S>` | key : String <br> defaultValue: `S` | `SubscribedAbstractProperty<S>` | if a property with given key exists returns a one-way data binding to this property (see `prop()`. If the property does not exist, create a property with given default value `defaultValue` (see `setOrCreate`) and return a prop to it (see `prop`). The `defaultValue` must be a of type `S`, `undefined` or `null` are not allowed.  <br><br>From API rel. 10 onwards, in addition to types `S` also types `T` are supported. |
| `setAndProp<T>` since API rel. 10 | key : String <br> defaultValue: `T` | `SubscribedAbstractProperty<T>` | if a property with given key exists returns a one-way data binding to this property (see `prop()`. If the property does not exist, create a property with given default value `defaultValue` (see `setOrCreate`) and return a prop to it (see `prop`). The `defaultValue` must be a of type `T`, `undefined` or `null` are not allowed. <br><br>From API rel. 10 onwards, in addition to types `S` also types `T` are supported. |
| `delete` | key : String | boolean | Delete the property with given name, if it exists and return true.  Precondition is that it has no subscribers anymore. If property does not exist or still has subscribers, then, do nothing and return false |
| `keys` | none | `IterableIterator<string>` | Return an iterator over all property keys, see `Map.keys()`  |
| `size` | none | number | number of properties, same as `Map.size()` |
| `clear` | none | boolean | delete _all_ properties from the `LocalStorage` instance. Precondition is that there are no subscribers anymore. The method returns false and deletes no properties if there is one or several property that still has subscribers. |

Once created the type of a named property never changes. Subsequent calls to `set` must set a value of same type.


### 5.1.4  Sharing a LocalStorage instance from Ability to one or several Views

Create a `LocalStorage` instance in owning Ability and `windowStage.createUIContent`.

```TypeScript
class MainAbility extends Ability {

  private readonly etsFilePath = "pages/index";

  // the key could be anything, but filePath is convenient when each Page has its own LocalStorage
  private storage = new LocalStorage({ /* initializing object */ });

  onCreate(...) {
    // do nothing related to AppStorage and LocalStorage
  }

  onWindowStageCreate(windowStage) {  
    // createUIContent adds the given storage object to the launching page of etsFilePath
    windowStage.createUIContent(this.context, this.etsFilePath, this.storage);
  }

  onWindowStageDelete(...) {  // TODO add actual callback name for the reserve of onWindowStageDelete
      this.storage.aboutToBeDeleted();
  }
}
```

`createUIContent` creates the LocalStorage instance, it becomes available in `CompA` component and all its descendant components automatically:

```TypeScript
@Entry
@Component struct CompA {

  // can access LocalStorage instance using 
  // @LocalStorageLink/Prop decorated variables
  @LocalStorageLink("propA") varA : number = 1;
  
  build() {
    ...
  }
}
```

Developer advise: It is good practise to always create a LocalStorage with meaningful default values
that serves as a backup when providing Ability work abnormally. It is also useful for unit testing the page.

## 5.2 Storage API for application-wide UI state `AppStorage`

`AppStorage` is a special, singleton `LocalStorage` object within the application that is created by the UI framework on application startup. Its purpose is to provide the central storage for mutable application UI state properties. `AppStorage` contains _all_ those state properties that  need to be accessible _throughout_ the application _to construct the UI_. The storage retains all properties and their values as long as the application remains running. Properties are accessed using a unique key string value.

UI Components synchronize application state data properties with the `AppStorage`. Implementation of application business logic can access `AppStorage` as well.

Selected state properties of `AppStorage` can be synched with different data sources or data sinks. Those data sources and sinks can be local on the device or remote, and have different capabilities, such as data persistence (see `PersistentStorage`). These data sources and sinks are implemented in the business logic, separate from the UI. Link those `AppStorage` properties to `PersistentStorage` whose values should be kept until application re-start.


### 5.2.1 Example for using `AppStorage` and `LocalStorage` from app logic

Example for using `AppStorage` from app logic. AppStorage is a singleton, its static API functions resemble those (non-static) functions of `LocalStorage`:

```typescript
AppStorage.setOrCreate("PropA", 47);         // create singleton instance on demand and init with given property

let storage = new LocalStorage({"PropA": 17}); // create new instance of LocalStorage and init with given object

var propA  = AppStorage.get("PropA")         // propA in AppStorage == 47, propA in LocalStorage == 17
var link1  = AppStorage.link("PropA");       // link1.get() == 47
var link2  = AppStorage.link("PropA");       // link2.get() == 47
var prop   = AppStorage.prop("PropA");    // prop.get() = 47

link1.set(48);      // two-way sync: link1.get() == link2.get() == prop.get() == 48
prop.set(1);        // one-way sync: prop.get()=1; but link1.get() == link2.get() == 48
link1.set(49);      // two-way sync: link1.get() == link2.get() == prop.get() == 49

storage.get("PropA")  // == 17 setting propA in AppStorage does not affect LocalStorage
storage.set("PropA", 101);   // Setting propA in LocalStorage does not affect AppStorage property and links, props using it as source: 
storage.get("PropA")     // == 101

AppStorage.get("PropA")  // == 49
link1.get()              // == 49
link2.get()              // == 49
prop.get()               // == 49
```


### 5.2.2  Example for using `AppStorage` and `LocalStorage` from inside UI

The `@StorageLink` variable decorator works together with the `AppStorage` in the same way as `@LocalStorageLink` works together with `LocalStorage`. It creates two-way data sync with a property in `AppStorage`:

```typescript
// set PropA in AppStorage, if the singleton does not exist yet, it will be created on the fly
AppStorage.setOrCreate("PropA", 47); 

// create new instance and init with given object
let storage = new LocalStorage({"PropA": 48}); 

@Entry(storage)
@Component
struct CompA {
   
    // @StorageLink variable decorator creates two-way link with property "PropA" in AppStorage
    // initial value will be 47, because set in above code
    @StorageLink("PropA") storLink : number = 1;

    // @LocalStorageLink variable decorator creates two-way link with property "PropA" in LocalStorage
    // initial value will be 48, because set in above code
    @LocalStorageLink("PropA") localStorLink : number = 1;

    build() {
        Column({ space: 20 }) {
        Text(`From AppStorage ${this.storLink}`)
            // changes will sync with "PropA" in AppStorage, not in LocalStorage
            .onClick(() => this.storLink+=1 ) 

        Text(`From LocalStorage ${this.localStorLink}`)
            // changes will sync with "PropA" in LocalStorage, not in AppStorage
            .onClick(() => this.localStorLink+=1 ) 
       }
   }
}
```

 ### 5.2.3 Provisions for using `AppStorage` API

The API of `AppStorage` is similar to that of `LocalStorage`. Since `AppStorage` is a singleton, all methods are static ones. 

Type `T` can be Class, number, boolean, string, or Array of these types. Type `S` can be number, boolean, string (simple types only). Once created the type of a named property never changes.  Subsequent calls to `set` must set a value of same type.

| (static) Method  | Parameter | Return value | Definition |
|------------|--------|--------|---------|
| `createSingleton` | obj : Object | void | `AppStorage` is initialized on first access. Use of `createSingleton(obj)` is optional. It is useful to initialize `AppStorage` with the given `Object` at the start of the application. All object properties returned by `Object.keys(obj)` and their values will be added to `AppStorage`. |
| `link<T>` <br><br>renamed from<br>`Link<T>` (deprecated) | key : String | `SubscribedAbstractProperty<T>` \| `undefined` | If a property with given key exists and if it is mutable, returns a two-way data binding to this property. Means value changes will be synched from the using variable (or Component) to the `AppStorage` and from `AppStorage` to any variable (or Component). The call returns `undefined` if a property with this key does not exist or if the property is read-only. The type can be of any type `T`. The function has additional optional parameters that are reserved for framework internal use of `link`/`setAndLink`. |
| `setAndLink<T>` <br><br>renamed from<br>`SetAndLink<T>` (deprecated) | key : String <br> defaultValue: `T` | SubscribedAbstractProperty<T> | if a property with given key exists, determine the return value as defined for `link`. If the property does not exist, create a mutable property with given default type and value `defaultValue` (see `setOrCreate`) and return a link to it (see `link`). The default value must be a of type `T`. E.g. `undefined` or `null` are not allowed. |
| `prop<S>` <br><br>renamed from<br>`Prop<S>` (deprecated) | key : String | `SubscribedAbstractProperty<S>` | if a mutable or immutable property with given key exists, returns a one-way data binding to this property. Means value changes will be synched from `AppStorage` to any variable (or Component). The default value must be a of type `S`. The function has additional optional parameters that are reserved for framework internal use of `prop`/`setAndProp`  <br><br>From API rel. 10 onwards, in addition to types `S` also types `T` are supported. |
| `prop<T>` since API rel. 10 | key : String | `SubscribedAbstractProperty<T>` | if a mutable or immutable property with given key exists, returns a one-way data binding to this property. Means value changes will be synched from `AppStorage` to any variable (or Component). The default value must be a of type `T`. The function has additional optional parameters that are reserved for framework internal use of `prop`/`setAndProp`|
| `setAndProp<S>` <br><br>renamed from<br>`SetAndProp<S>` (deprecated) | key : String <br> defaultValue: `S` | `SubscribedAbstractProperty<S>` | if a property with given key exists, returns a one-way data binding to this property (see `prop`). If the property does not exist, create a mutable property with given default value `defaultValue` (see `setOrCreate`) and return a prop to it (see `prop`). The default value must be a of type `S`. E.g. `undefined` or `null` are not allowed. <br><br>From API rel. 10 onwards, in addition to types `S` also types `T` are supported. |
| `setAndProp<T>` since API rel. 10| key : String <br> defaultValue: `T` | `SubscribedAbstractProperty<T>` | if a property with given key exists, returns a one-way data binding to this property (see `prop`. If the property does not exist, create a mutable property with given default value `defaultValue` (see `setOrCreate`) and return a prop to it (see `prop`). The default value must be a of type `T`. E.g. `undefined` or `null` are not allowed. |
| `has` <br><br>renamed from<br>`Has` (deprecated) | key : String | boolean | Return true of the property with given name exists in AppStorage |
| `get<T>` <br><br>renamed from<br>`Get<T>` (deprecated) | key : String | `T` \| `undefined` | Get the property with the given name, if it exists |
| `set<T>` <br><br>renamed from<br>`Set<T>` (deprecated) | key : String, <br> newValue : `T` | boolean | If a mutable property with the given name exists, set its value and return true. Otherwise don't set anything and return false. `newValue` must be a of type `T`, `undefined` or `null` are not allowed. |
| `setOrCreate<T>` <br><br>renamed from<br>`SetOrCreate<T>` (deprecated) | key : String, <br> newValue : `T` | boolean | if a property with given name exists: If it is mutable, update its value and return true. If immutable, return false (failure). If no property with given name exists: Create a new mutable property with given default value inside AppStorage. The `newValue` must be a of type `T`,  `undefined` or `null` are not allowed. Return true. |
| `delete` <br><br>renamed from<br>`Delete` (deprecated) | key : String | boolean | Delete the property with given name, if it exists and return true. Precondition is that there are no subscribers anymore. If property does not exist or it still has subscribers, do nothing and return false. |
| `keys` <br><br>renamed from<br>`Keys` (deprecated) | none | `IterableIterator<string>` | Return an iterator over all property keys, see `Map.keys()`  |
| `clear` <br><br>renamed from<br>`Clear` (deprecated) | none | boolean | delete all properties from the AppStorage. Precondition is that there are no subscribers anymore. The method returns false and deletes no properties if there is any property that still has subscribers. |
| `IsMutable` (deprecated) | key : String | boolean | returns true if the property exists and is mutable, otherwise false. Note: Function currently not implemented, all AppStorage properties are mutable. |

Static function names starting with an upper case letter are still supported but depreciated. Applications should be updated to use lower case static function names. This change was made in May 2023 to comply with a new OHOS API style guideline.

An earlier version of the specification made mention of immutable AppStorage properties. This paragraph was deleted because the feature was never implemented and there are no plans to do so.

`AppStorage` working together with `PersistentStorage` and  `Environment`:
- A call to `PersistentStorage.link()` _after_ creating the property in `AppStorage` (the opposite order of calls is what is recommended) uses the type and value in `AppStorage` and overwrites any property with the same name in `PersistentStorage`.
- A call to `Environment.envProp()` _after_ creating the property in `AppStorage` will fail.

## 5.3 Supporting APIs to `AppStorage` and `LocalStorage`

### 5.3.1 `SubscribedAbstractProperty<T>`

`ObservedPropertyAbstract<T>` is the common base class for observed properties, and properties implementing one-way or two way sync in the framework. An instance can be obtained using one of the following aforementioned API function

| Storage | Function | Return value | Description |
|---|---|---|---|
| LocalStorage | `link<T>`, `setAndLink<T>` | `SubscribedAbstractProperty<T>` | Two-way sync |
| LocalStorage | `prop<S>`, `setAndProp<S>` | `SubscribedAbstractProperty<T>` | One-way sync |
| AppStorage | `link<T>`, `setAndLink<T>` | `SubscribedAbstractProperty<T>` | Two-way sync |
| AppStorage | `prop<S>`, `setAndProp<S>` | `SubscribedAbstractProperty<T>` | One-way sync |

| `SubscribedAbstractProperty<T>` | Parameter | Return value | Description |
|---|---|---|---|
| `get<T>` | none | `T` | get value |
| `set<T>` | newValue : T | void | set new value of type `T`. `T` must always be the same | 
| `aboutToBeDeleted` | none | void | notify the framework the `SubscribedAbstractProperty<T>` object will go out of scope. This call ends to two-way sync relationship with the property in `LocalStorage` or `AppStorage` by unsubscribing from its source. |
| `info` | none | string | return the property name, e.g. `let link = AppStorage.link("foo")` then `link.info() == "foo"` |
| `numberOfSubscribers` | none | number | number of subscribers |



### 5.3.2 Add a `LocalStorage` instance to a `@Component` using `@Entry`

There are two solutions on how to create and add an instance of `LocalStorage` instance to a `@Component`. Which one to choose depends 
if the creation and the use of an `LocalStorage` instance spans across multiple source files of the application project.

The first solution using an `@Entry` component decorator parameter works only when the `LocalStorage` instance is created in the same eDSL source file as the top-level ('entry') `@Component`:

The `@Entry` decorator defines the default top-level `@Component`, see section 2.2.2 . The `@Entry` decorator accepts an optional parameter of type `LocalStorage`. This adds the storage object to the custom component, from where all its descendant custom components inherit access.

| @Entry | Description |
|----|---|
| storage : LocalStorage | adds the storage object to the decorated custom component |

The following example shows how to create a `LocalStorage` using the constructor 
and how to add it to the `CompA` top-level component using its `@Entry` decorator:

```TypeScript
let storage = new LocalStorage({ "propA": 47 });

@Entry(storage)
@Component struct CompA {

  // can access 'storage' instance using 
  // @LocalStorageLink/Prop decorated variales
  @LocalStorageLink("propA") varA : number = 1;
  
  build() {
    ...
  }
}
```


## 5.4 API for persisting application properties `PersistentStorage`

`PersistentStorage` is an optional singleton object within the application. The purpose of this object is to persist selected `AppStorage` properties so that their values are the same upon application re-start as they were when the application was closed.  The framework creates the instance of `PersistentStorage` (in a lazy way) on first access.

The most important API function of `PersistentStorage` is `link(..)`. It creates a two-way sync for a property in `AppStorage`. There are a few additional API functions to manage persisted properties. Business logic always get or set properties through `AppStorage`. 

### 5.4.1 Example 

The important API call is `PersistentStorage.persistProp("aProp", 50)` - it configures AppStorage property `aProp` to be persistent to file. This API call is meant to be included into the application's business logic. Read more about how to use this API in the following:

```TypeScript
// application business logic, executed on app startup before constructing the the UI
// instruct to persist the property "aProp" of AppStorage 
// means whenever the property changes in AppStorage the property and its new value are 
// saved to local file.
// if upon executing this call the property does not exist in AppStorage
// create it and init with value 50;
PersistentStorage.persistProp("aProp", 50); 

@Entry
@Component
struct Index {
  // use property "aProp" from AppStorage 
  // to construct the UI of this Component
  // if the property does not exist in AppStorage, yet, 
  // create it and init with value 47
  @StorageLink("aProp") n: number = 47;

  build() {
    Row() {
      Column() {
        Text(`${this.n}`)
          .onClick(() => {
            // increment the decorated variable
            // the two-way sync semantics of @StorageLink decorator will cause 
            // the changed value to be synched back to AppStorage "aProp" property.
            // According to the PersistentStorage.persistProp("aProp", ..) command the 
            // changed value from AppStorage will automatically be persisted to file.
            this.n += 1; // 51
          })
      }
    }
  }
}
```

Scenario #1: First run after fresh app installation
1. Execution of line `PersistentStorage.persistProp("aProp", 50);`: the property "aProp" is not found on PersistentStorage disk, because the app has just been installed. It is also not available in `AppStorage`. The semantics are that the property will be created in AppStorage and initialized with given value `50` (last parameter of `persistProp` function). The property and its value is written to disk.
2. The `Index` @Component gets created, and subsequently the `@StorageLink("aProp") n ` gets created. The search for AppStorage property "aProp" succeeds, therefore, `n` is initialized with the value 50 found in AppStorage.

On click handler execution
1. `@StorageLink("aProp") n` value is increased, triggers Index component re-render.
2. `@StorageLink` semantics imply sync the value back to the property in AppStorage.
3. The change of AppStorage value "aProp" triggers any other `@StorageLink` of `@StorageProp` variables linking to this property to be updated (here we have no other such variables)
4. Because of the given command `PersistentStorage.persistProp("aProp", ...)` the change of AppStorage value "aProp" triggers PersistentStorage to write the property and its changed value to file.

Scenario #2: Subsequent app run (start the app another time):
1. Execution of line `PersistentStorage.persistProp("aProp", 50);`: the property "aProp" is found on PersistentStorage disk. The property is added to AppStorage with the value found on disk.
2. The `Index` @Component gets created, and subsequently the `@StorageLink("aProp") n ` gets created. The search for AppStorage property "aProp" succeeds, therefore, `n` is initialized with the value found in AppStorage.

The actions upon click handler execution are the same as in scenario #1.

> As of March 2 PersistentStorage has a _bug_ for number and boolean type properties. Upon 2nd application run the number type variable is read back from disk into AppStorage as a string type property. 


### 5.4.2 Wrong use - Access property in `AppStorage` before `PersistentStorage`

See step 1 of scenario #2 above. To ensure a property value found on PersistentStorage disk gets used, it is imperative to configure PersistentStorage with `persistProp` or `persistProps` function  _before_ accessing AppStorage thru its programming API and also before creating any UI components.

An example of using AppStorage API in the _wrong_ order:

```TypeScript
let aProp = AppStorage.setOrCreate("aProp", 47);   // WRONG order, always configure property with PersistentStorage first!
P, 48);
```

* `AppStorage.setOrCreate("aProp", 47)`: property "aProp" is created in `AppStorage`, its type is number and its value is set to the specified default 47.
* `PersistentStorage.persistsProp("aProp", 48)`: Conflict: A property with same name but different value is available in `AppStorage` and in `PersistentStorage` disk (available from previous run). `AppStorage` _overrides_ `PersistentStorage` disk. The value from previous app run is lost.

 
 ### 5.4.3 Provisions for using `PersistentStorage`

The API is the following. All methods are static:

| (static) Method | Parameter | return value | definition |
|------------|-----------|------------|------------|
| `persistProp` <br><br>renamed from<br>`PersistProp` (deprecated) | key : string<br> defaultValue: `T` | void | instructs the named property in `AppStorage` to be persisted to file. This call should always be made before accessing AppStorage by either using its API or creating any component with a `@StorageLink` or `@StorageProp` decorated variable. The resolution order to determine the property type and value is: <br/> 1. Property with given name exists on PersistentStorage file. If found, create a property with given name in AppStorage and set its value with the value found. <br>2. Property does not exist on file. Create a property with given name in AppStorage and set its value with the defaultValue parameter. |
| `deleteProp` <br><br>renamed from<br>`DeleteProp` (deprecated) | key : string | void | Inverse  of `PersistProp`. The property gets deleted from the persistent storage and subsequent value changes in `AppStorage` have no effect on `PersistentStorage`.  |
| `persistProps` <br><br>renamed from<br>`PersistProps` (deprecated) | keys : `{ key: string,`<br>`defaultValue: any}[]` | void | Convenience function suitable to especially at app startup to initialize all links of  `PersistentStorage`. Calls `persistProp(key, defaultValue)` for each of the given key + defaultValue pairs.  |
| `keys` <br><br>renamed from<br>`Keys` (deprecated) | | void | returns an `Array<string>` of all persisted property keys |

Static function names starting with an upper case letter are still supported but depreciated. Applications should be updated to use lower case static function names. This change was made in May 2023 to comply with a new OHOS API style guideline.

Permissible types and values of AppStorage properties that work with PersistentStorage:
* number, string, boolean, enum simple types
* Objects that can be reconstructed by combination of JSON.stringify() and JSON.parse(). This rules out built-in types such as Date, Map, Set because their value will not be re-constructed correctly.
* nested Objects (Array of objects, object type property inside object, etc.) may not work as the framework is currently not able to detect value changes of _nested_ Objects (incl. Arrays) in AppStorage.
* `undefined` and `null` value is not supported.

Developers are advised that persisting data is a comparably slow operation. An application should avoid
- persisting large data sets, and
- persisting variables that are very frequently changing

The `PersistentStorage` implementation may throttle the frequency of persisting property changes when the process of persisting changes becomes too heavy.

> The current implementation of  `PersistentStorage`  is functionally complete but little attention has been paid to performance and data integrity (e.g. forceful application termination while write is on-going can lead to data loss). Therefore, the size of saved data needs to be kept small.

## 5.5 API for importing device environment `Environment`

`Environment` is a singleton object created by the framework on application start. It provides a range of application state properties to `AppStorage` that describe the device environment in which the application is executing. `Environment` and its properties are immutable. All property values are of simple type only.

Example:

```typescript
Environment.envProp("languageCode", "cn")
```
The UI framework implementation takes care of updating an `AppStorage` property that has been initialized with `Environment.envProp(..)`. Whenever its value changes in the device environment it will update its value in `AppStorage`.

Access from UI components like this. To keep a Component variable updated with changes in the device environment, this variable should be decorated with `@StorageProp`.
```TypeScript
@StorageProp("languageCode") lang : string = "cn";
```
The chain of updates is from  device internals to `Environment` to `AppStorage` to the decorated variable in the UI component.  An `@StorageProp` decorated variable can be locally modified but the change will not be updated to `AppStorage`.

Example by use in the business logic:
```TypeScript
const lang = AppStorage.prop("languageCode")   // property "languageCode" provided from 'Environment' to `AppStorage` 

if (lang == "cn") {
    console.log("你好")
} else {
    console.log("Hello!")
}
```

### 5.5.1 Provisions for using `Environment`

The API is the following. All methods are static:

| (static) Method | Parameter | return value | definition |
|------------|-----------|------------|------------|
| `envProp` <br><br>renamed from<br>`EnvProp` (deprecated) | key : string<br>defaultValue: `S` | boolean | creates a new property in `AppStorage`. The UI framework implementation takes care of updating its value whenever the named device environment property changes. Recommended use is at app startup.  The function call fails and returns false if a property with given name exists in `AppStorage` already. It is _wrong_ API use to access a property with given name in `AppStorage` before calling `Environment.envProp`. |
| `envProps` <br><br>renamed from<br>`EnvProps` (deprecated) | keys : `{ key: string,`<br>`defaultValue: any}[]` | void | Convenience function to initialize all device environment properties that app will use in one batch. Calls `envProp(key, defaultValue)` for each of the given key + defaultValue pairs. Recommended use is at app startup.  |
| `keys` <br><br>renamed from<br>`Keys` (deprecated) | | void | returns an `Array<string>` of all environment property keys |

> Note: `set(key, value)` function is required by the UI framework to make property value updates, but it is not meant for use by an app. 

Static function names starting with an upper case letter are still supported but depreciated. Applications should be updated to use lower case static function names. This change was made in May 2023 to comply with a new OHOS API style guideline.

**The built-in environment values of the framework are as follows:**

> Still to be reviewed and updated acc. to implementation.
> Action item HQ: Update below table with supported device environment properties.

| Key  | Data Type       | Description |
|--------------------|---------------|-------------|
| accessibilityEnabled | boolean         | Check whether the accessibility  screen reading is enabled.  |
| colorMode            | ColorMode enum  | Dark mode type. The  options are ColorMode.Light: light color. ColorMode.Dark: dark color. |
| fontScale            | number          | Font size scale. Range: [0.85, 1.45]                         |
| fontWeightScale      | number          | Font weight scale. Range：[0.6, 1.6]                       |
| layoutDirection      | LayoutDirection | layout direction type: including LayoutDirection.LTR:  left to right. LayoutDirection.RTL: right-to-left. |
| languageCode         | string          | Indicates the current system language value. The value must be lowercase letters, for example, zh |


## 5.6 Overview: Synchronization between `AppStorage` / `LocalStorage` and UI Components 

In previous chapters we defined already how a component variable can be synced with a `@State` decorated variable in a parent or ancestor component. This is the purpose of the `@Prop`, `@Link` and `@Consume` decorators.

This section defines how to sync a Component variable with `LocalStorage` and its special  singleton `AppStorage`:
* To access `AppStorage` a custom component variable needs to be decorated with either `@StorageLink` or `@StorageProp`.
* To access `LocalStorage` use variables decorators `@LocalStorageLink` or `@LocalStorageProp`.

## 5.7 Synchronization between `AppStorage` and UI Components using `@StorageLink` and `@StorageProp` decorators

By decorating the variable with `@StorageLink(key)` a two way data synchronization is established. `key` identifies the property in `AppStorage`. When creating the owning Component, the variable is initialized with the value in `AppStorage`. Local variable initialization is mandatory. If a property with given `key` is missing from `AppStorage` then it will be added with stated initializing value.
 A change made in the UI is synched back to `AppStorage` and from there to any other binding instance, such as `PersistentStorage` or other Components that bind to it with either `@StorageLink` or `StorageProp`. 

By decorating the variable with `@StorageProp(key)` a one way data synchronization from `AppStorage` to the decorated variable is established. The rules for initialization are the same as for `@StorageLink(key)`. A change can be made in the UI but it will not be synched to `AppStorage`. An update from `AppStorage` will overwrite local changes.

`@StorageLink(key)` and `@StorageProp(key)` support number, string, boolean, class object and array of these types. Observed changes value assignment for all types, object property changes, array item replacements, additions, and deletions, for details see [section 3.2.5](intro-state-mgmt.md#325-variable-types-and-observed-changes-for-variables-decorated-with--state-provide-link-consume-or-prop)

### 5.7.1 Example of use

Example for using `@StorageLink`.

```typescript

// in non-UI logic
AppStorage.setOrCreate("PropA", 47); 
let linkToPropA = AppStorage.link("PropA"); 

@Entry
@Component
struct CompA {
   
    // @StorageLink variable decorator creates two-way link with property "PropA" in AppStorage
    // initial value will be 47, because set in above code
    @StorageLink("PropA") storLink : number = 1;

    build() {
        Column() {
          Text(`incr @StorageLink variable`)
            // updates "PropA" in AppStorage
            .onClick(() => this.storLink += 1 )

          // bad practise and will not always work to use global vars in UI
          Text(`@StorageLink: ${this.storLink} - linkToPropA: ${linkToPropA.get()}`)
        }
    }
}
```

### 5.7.2 Provisions for using `@StorageLink` and `@StorageProp`

***`@StorageLink` decorator***

| `@StorageLink` variable decorator | Comment |
|---|---|
| custom component variable decorator | |  
| decorator parameters | key: constant string, mandatory to specify (note, the string is quoted, see example) |
| type of sync | two-way: from `AppStorage` property to component variable and back |
| permissible variable types and observed changes | See  [section 3.2.5](intro-state-mgmt.md#325-variable-types-and-observed-changes-for-variables-decorated-with--state-provide-link-consume-or-prop) || local initialization | must specify, serves as initializing default if the property does not exist in `AppStorage`, yet. |
| initialize and update from parent | forbidden |
| can use to initialize sub-components | allowed, see section 3.2 for details. |
| access | A `@StorageLink` variable is private, accessible only within the component | 

> TODO need to check if Subscribale is permitted value type

***`@StorageProp` decorator***

| `@StorageProp` variable decorator | Comment |
|---|---|
| custom component variable decorator | |  
| decorator parameters | key: constant string, mandatory to specify (note, the string is quoted) |
| type of sync | one-way: from `AppStorage` property to component variable. The component variable can be changed locally, but an update from `AppStorage` will overwrite local changes. |
| permissible variable types and observed changes | See  [section 3.2.5](intro-state-mgmt.md#325-variable-types-and-observed-changes-for-variables-decorated-with--state-provide-link-consume-or-prop) |
| local initialization | mandatory to specify, gets used as initializing default if the property does not exist in `AppStorage`, yet. |
| initialize and update from parent | forbidden |
| can use to initialize sub-components | allowed, see section 3.2 for details |
| access | A `@StorageProp` variable is private, accessible only within the component | 

`@StorageLink` or `@StorageProp` value changes trigger `@Component` re-render with a delay in two scenario:
1. When the owning @Component is inactive because the containing page is not visible, amd 
2. When the owning @Component is inactive because the containing Tab item inside a `Tabs` component is not visible.

In both scenarios the behaviour is as follows.
* sync of the `@StorageLink` or `@StorageProp` variable value is normal, e.g. change of the AppStorage property value updates the variable normally.
* a variable value change is observed normal, but
* an observed change does not trigger `@Watch` function execution and re-render of the owning `@Component` immediately. Instead the owning @Component is notified with a delay when it becomes active, and it will execute @Watch function and re-render subsequently. 

Developers should take note that 
* delayed `@Watch` execution can change the execution order of application functions, e.g a timer function or other event from platform may execute before the `@Watch` function.
* @Component re-render updates `@Prop` and `@ObjectLink` value in a child. These variable value updates get delayed subsequently as well.


## 5.8 Synchronization between `LocalStorage` and UI Components using  `@LocalStorageLink`and `@LocalStorageProp` decorators

`@LocalStorageLink`and `@LocalStorageProp` decorators work similarly with `LocalStorage` as aforementioned decorators work with `AppStorage`:

`@LocalStorageLink(key)` variable decorator creates a two-way sync with the named property in the @Component's `LocalStorage`. 

`@LocalStorageProp` creates a one-way data sync from `LocalStorage` to the variable.


### 5.8.1 First example of use

Example for using `@LocalStorageLink` with `LocalStorage`.
The example is intentionally similar to the one in section 5.10.1 to show similarities and differences.

```typescript

// in non-UI logic
let storage = new LocalStorage({"PropA": 47}); 
let linkToPropA = storage.link("PropA"); 

@Entry(storage)
@Component
struct CompA {
   
    // @LocalStorageLink variable decorator creates two-way link with property "PropA" in @Component's LocalStorage
    // initial value will be 47, because set in above code
    @LocalStorageLink("PropA") storLink : number = 1;

    build() {
        Column() {
          Text(`incr @LocalStorageLink variable`)
            // updates "PropA" in AppStorage
            .onClick(() => this.storLink+=1 )

          // bad practise and will not always work to use global vars in UI
          Text(`@LocalStorageLink: ${this.storLink} - linkToPropA: ${linkToPropA.get()}`)
        }
    }
}
```

### 5.8.2 Provisions for using `@LocalStorageLink` and `@LocalStorageProp`

A top-level custom component can be initialized with at most one `LocalStorage` instance. All its children and children of children will inherit access to this storage object. `@LocalStorageLink`and `@LocalStorageProp` decorators sync the decorated variable with this storage instance. If a component has no `LocalStorage` instance (neither by explicit initialization nor by inheritance from an ancestor), then, the framework will implicitly create an (empty) `LocalStorage` instance. Descendent components inherit this instance.

***@LocalStorageLink decorator***

| `@LocalStorageLink` variable decorator | Comment |
|---|---|
| custom component variable decorator | |  
| decorator parameters | key: constant string, mandatory to specify (note, the string is quoted) |
| type of sync | two-way: from `LocalStorage` property to component variable and back |
| permissible variable types and observed changes | See  [section 3.2.5](intro-state-mgmt.md#325-variable-types-and-observed-changes-for-variables-decorated-with--state-provide-link-consume-or-prop) || default value | must specify, serves as initializing default if the property does not exist in `LocalStorage` instance, yet. |
| initialize and update from parent | forbidden |
| can use to initialize sub-components | allowed, see section 3.2 for details  |
| access | A @LocalStorageLink variable is private, accessible only within the component | 

> TODO need to check if Subscribale is permitted value type

***@LocalStorageProp decorator***

| `@LocalStorageProp` variable decorator | Comment |
|---|---|
| custom component variable decorator | |  
| decorator parameters | key: constant string, mandatory to specify (note, the string is quoted) |
| type of sync | one-way: from `LocalStorage` property to component variable. The component variable can be changed locally, but an update from `LocalStorage` will overwrite local changes. |
| permissible variable types | class objects, string, number, boolean, and arrays of these types, the type must be specified. The type of the property in `LocalStorage` and the decorated variable type must be the same.  `undefined` and `null value are not supported. |
| default value | must specify, serves as initializing default if the property does not exist in `LocalStorage`, yet. |
| observed changes | for boolean, string, number type value: any value change <br> for class object type value: set new object value, and value changes of all those object properties that `Object.keys(observedObject)` returns.<br> for Array type value: adding, deleting, updating array item |
| initialize and update from parent | forbidden |
| can use to initialize sub-components | allowed, see section 3.2 for details  |
| access | A `@LocalStorageProp` variable is private, accessible only within the component | 

`@LocalStorageLink` or `@LocalStorageProp` value changes trigger @Component re-render with a delay in the sme way as defined for `@StorageLink` and `@StorageProp`:
1. When the owning @Component is inactive because the containing page is not visible, amd 
2. When the owning @Component is inactive because the containing Tab item inside a `Tabs` component is not visible.

In both scenarios the behaviour is as follows.
* sync of the `@StorageLink` or `@StorageProp` variable value is normal, e.g. change of the AppStorage property value updates the variable normally.
* a variable value change is observed normal, but
* an observed change does not trigger `@Watch` function execution and re-render of the owning `@Component` immediately. Instead the owning @Component is notified with a delay when it becomes active, and it will execute @Watch function and re-render subsequently. 

Developers should take note that 
* delayed `@Watch` execution can change the execution order of application functions, e.g a timer function or other event from platform may execute before the `@Watch` function.
* @Component re-render updates `@Prop` and `@ObjectLink` value in a child. These variable value updates get delayed subsequently as well.

### 5.8.3 Second example of use

This example demonstrates shared access to the same `LocalStorage` instance in a parent and two sibling child custom components.
Each variable has a two-way sync with the same `countStorage` property in the shared `LocalStorage` instance.

filename: `ets/localStorageLink.ets`

```TypeScript
let storage = new LocalStorage({ countStorage: 1 });
console.log(`Created and initialized LocalStorage: readback from LocalStorage prop 'countStorage': ${storage.get<number>('countStorage')}.`);

@Component
struct Child {
  // give the Child instance a name, ver updates
  label : string = "no name";

  // two-way sync with  "countStorage" in LocalStorage
  // call @Watch function upon changes (made locally also elsewhere)
  @LocalStorageLink("countStorage") @Watch("onPlayCountChanged") playCountLink : number = 0;

  // to check if @Watch works, see onPlayCountChanged
  @State playCountChk : number = this.playCountLink;

  // @Watch function
  onPlayCountChanged(chgPropName : string) {
    console.log(`@Watch cb: ${chgPropName} changed, new value ${this[chgPropName]}.`);
    this.playCountChk = this[chgPropName];
  }

  build() {
    Row() {
      Text(this.label)
        .width(50).height(60).fontSize(12)

      Text(`playCountLink ${this.playCountLink}: inc by 1`)
            .onClick(() => {
              console.log(`click handler start: playCountLink ${this.playCountLink} .`)
              this.playCountLink += 1;
              console.log(`click handler after: playCountLink ${this.playCountLink} .`)
              console.log(`readback from LocalStorage prop 'countStorage': ${storage.get<number>('countStorage')}.`);
             })
              .width(200).height(60).fontSize(12)
       
       Text(`@Watch'ed: ${this.playCountChk}`)
          .width(50).height(60).fontSize(12)

      }.width(300).height(60)
  }
}

@Entry(storage)
@Component
struct Parent {

    @LocalStorageLink("countStorage") playCount : number = 0;
    build() {
        Column() {
            Row() {
                Text("Parent")
                    .width(50).height(60).fontSize(12)
                Text(`playCount ${this.playCount} dec by 1`)
                    .onClick(() => {
                        console.log(`Parent click handler start: playCount ${this.playCount}.`)
                        this.playCount -= 1;
                        console.log(`click handler after: playCount ${this.playCount}.`)
                        console.log(`readback from LocalStorage prop 'countStorage': ${storage.get<number>('countStorage')}`);
                    })
                    .width(250).height(60).fontSize(12)
            }.width(300).height(60)

            Row() {
                Text("LocalStorage")
                    .width(50).height(60).fontSize(12)
                Text(`countStorage ${this.playCount} incr by 1`)
                    .onClick(() => {
                        console.log(`StorageLink click handler start: `)
                        storage.set<number>('countStorage', 1 + storage.get<number>('countStorage'));
                        console.log(`readback from LocalStorage prop 'countStorage': ${storage.get<number>('countStorage')}.`);
                    })
                    .width(250).height(60).fontSize(12)
            }.width(300).height(60)

            Child({ label: "ChildA" })
            Child({ label: "ChildB" })

            Text(`playCount in LocalStorage for debug ${storage.get<number>("countStorage")}`)
            .width(300).height(60).fontSize(12)
        }
    }
}
```

## 5.9 `LocalStorage` and `AppStorage` access combined

The following example exemplifies use of `@LocalStorageLink`, `@LocalStorageProp`, `@StorageLink` and `@StorageProp` in combination.
The former two share the same `color` source property in `LocalStorage`, the latter a different source property in `AppStorage` despite the same property name. Worth pointing out that the parameter of the `@Watch` callback function parameter is the `@Component` variable name.

Filename: `ets/watchStorages.ets`

```TypeScript
function getRandomColor() : string {
    return "#ff" + rndHexStr(255) + rndHexStr(255) + rndHexStr(255);
}

function rndHexStr(max : number) : string {
    return Math.floor(Math.random() * max).toString(16);
}

// Create and initialize AppStorage singleton
// property name 'color'
AppStorage.setOrCreate("color", getRandomColor());

// Create and initialize LocalStorage object
// property name 'color'
let storage = new LocalStorage({ color: getRandomColor() });

console.log(`Created and initialized LocalStorage and AppStorage: read-back 
  LocalStorage 'color': ${storage.get<string>('color')}
  AppStorage 'color': ${AppStorage.get<string>('color')}
  Values are different? ${storage.get<string>('color') != AppStorage.get<string>('color') ? 'yes' : 'no (likely a framework bug)'} .`);


@Entry(storage)
@Component
struct Parent {

// @StorageProp and @LocalStorageProp have one-way sync from AppStorage/from LocalStorage, @Watch catches local changes
@StorageLink("color")      @Watch("onStateChange") storLinkColor      : string = getRandomColor();
@StorageProp("color")      @Watch("onStateChange") storPropColor      : string = getRandomColor();  
@LocalStorageLink("color") @Watch("onStateChange") localStorLinkColor : string = getRandomColor();
@LocalStorageProp("color") @Watch("onStateChange") localStorPropColor : string = getRandomColor();  

// set by onStateChange to the name / the value of the last changed variable.
// see @Watch onStateChange cb function
@State lastChangedVariableName  : string = "n/a";
@State lastChangedVariableValue : string = "n/a";

onStateChange(watchedVarName : string) {
  // copy var name and its value to @State variables for display
  this.lastChangedVariableName = watchedVarName;
  this.lastChangedVariableValue = this[watchedVarName];
}

build() {
    Column() {
      Text("AppStorage.link: 'storLinkColor'")
        .width(200).height(60).fontSize(12)
        .backgroundColor(this.storLinkColor)
        .onClick(() => this.storLinkColor = getRandomColor() )

      Text("AppStorage.prop: 'storPropColor'")
        .width(200).height(60).fontSize(12)
        .backgroundColor(this.storPropColor)
        .onClick(() => this.storPropColor = getRandomColor() )

      Divider()
       .width(200).height(10)

      Text("LocalStorage.link: 'localStorLinkColor'")
        .width(200).height(60).fontSize(12)
        .backgroundColor(this.localStorLinkColor)
        .onClick(() => this.localStorLinkColor = getRandomColor() )

      Text("LocalStorage.prop: 'localStorPropColor'")
        .width(200).height(60).fontSize(12)
        .backgroundColor(this.localStorPropColor)
        .onClick(() => this.localStorPropColor = getRandomColor() )

      Divider()
       .width(200).height(10)

      Text(`@Watch Last @Component var changed: ${this.lastChangedVariableName}`)
        .width(200).height(60).fontSize(12)
        .backgroundColor(this.lastChangedVariableValue)
    }
  }
}
```

## 5.10 Example combining `@StorageLink`, `@LocalStorageLink`, `@Provide` - `@Consume` and `@State`

`AppStorage` property names, property names in each `LocalStorage` instance, `@Provide` and `@Consume` alias names, and variable names used inside a custom components all have their own, disjoint namespace. Most importantly, this means that the same property names, alias names and variable names can be used without any ambiguity. 

The following example exemplifies just this. Use of such confusing same names is not recommended for regular app development, though:

File `ets/storagesCombined.ets`

```TypeScript
function getRandomColor() : string {
    return "#ff" + rndHexStr(255) + rndHexStr(255) + rndHexStr(255);
}

function rndHexStr(max : number) : string {
    return Math.floor(Math.random() * max).toString(16);
}

// Create and initialize AppStorage singleton
// property name 'color'
AppStorage.setOrCreate("color", getRandomColor());

// Create and initialize LocalStorage object
// property name 'color'
let storage = new LocalStorage({ color: getRandomColor() });

console.log(`Created and initialized LocalStorage and AppStorage: read-back 
  LocalStorage 'color': ${storage.get<string>('color')}
  AppStorage 'color': ${AppStorage.get<string>('color')}
  Values are different? ${storage.get<string>('color') != AppStorage.get<string>('color') ? 'yes' : 'no (likely a framework bug)'} .`);

@Component
struct Consumer {
  label : string = "no name";
  // @Provide - @Consume alias name 'color'
  @Consume("color") consumeColor : string;

  build() {
      Text("@Consume: 'consumeColor'")
        .width(200).height(60).fontSize(12)
        .backgroundColor(this.consumeColor)
        .onClick(() => this.consumeColor = getRandomColor() )
  }
}


@Entry(storage)
@Component
struct Parent {

// the name 'color is used in 4 separated namespaces:
// 1. AppStorage property name;
// 2. LocalStorage instance property name,
// 3. @Provide - @Consume alias name, scope: UI descendent tree,
// 4. @Component @State variable name
@StorageLink("color")      @Watch("onStateChange")  storColor      : string = getRandomColor();
@LocalStorageLink("color") @Watch("onStateChange")  localStorColor : string = getRandomColor();
@Provide("color")          @Watch("onStateChange")  provideColor   : string = getRandomColor();
@State                     @Watch("onStateChange")  color          : string = getRandomColor();

// set by onStateChange to the name / the value of the last changed variable.
// see @Watch onStateChange cb function
@State lastChangedVariableName  : string = "n/a";
@State lastChangedVariableValue : string = "n/a";

onStateChange(watchedVarName : string) {
  // copy var name and its value to @State variables for display
  this.lastChangedVariableName = watchedVarName;
  this.lastChangedVariableValue = this[watchedVarName];
}

build() {
    Column() {
      Text("AppStorage: 'storColor'")
        .width(200).height(60).fontSize(12)
        .backgroundColor(this.storColor)
        .onClick(() => this.storColor = getRandomColor() )

      Text("LocalStorage: 'localStorColor'")
        .width(200).height(60).fontSize(12)
        .backgroundColor(this.localStorColor)
        .onClick(() => this.localStorColor = getRandomColor() )

      Text("@Provide: 'provideColor'")
        .width(200).height(60).fontSize(12)
        .backgroundColor(this.provideColor)
        .onClick(() => this.provideColor = getRandomColor() )

      Consumer({ label: "@Consume" })

      Text("@State: 'color'")
        .width(200).height(60).fontSize(12)
        .backgroundColor(this.color)
        .onClick(() => this.color = getRandomColor() )

      Divider()
       .width(200).height(10)

      Text(`@Watch Last @Component var changed: ${this.lastChangedVariableName}`)
        .width(200).height(60).fontSize(12)
        .backgroundColor(this.lastChangedVariableValue)
    }
  }
}
```


---

# 6 Other State Management functionality

## 6.1 Get notified of state variable changes `@Watch`


An application can request to be notified whenever the value of decorated variable changes. The callback function is called after the value change has occurred.

Introductory Example of `@Watch` with `@State`:

```TypeScript
@Entry
@Component
struct Basket {

    @State @Watch("onBasketUpdated") shopBasket : Array<number>= [ 7, 12, 47, 3 ];
    @State totalPurchase : number = this.updateTotal();

    updateTotal() : number {
        let total = this.shopBasket.reduce((sum, i) =>  sum + i , 0);

        // Apply discount if over 100EUR
        if (total >= 100) {
            total = 0.9 * total;
        }
        return total;
    }
    
    // @Watch cb
    onBasketUpdated(propName: string) : void {
        this.totalPurchase = this.updateTotal();
    }

    build() {
        Column() {
            Text(`Total: ${this.totalPurchase.toFixed(2)} €`)

            Button("Add to basket")
            .onClick(() => { 
                this.shopBasket.push(Math.round(100 * Math.random()));
            })
        }
    }
}
```

In above example the `Basket` custom component function `onBasketUpdated` is called after the watched `@State` variable `shopBasket` has been updated. It updates the `totalPurchase` variable. Custom component re-render happens asynchronously some time afterwards. The steps are the following:
1. event handler mutates `shopBasket` array
2. ArkUI state management framework calls `onBasketUpdated`
3. `onBasketUpdated` updates `totalPurchase`. 
4. In response to the two `@State` decorated variables changing, `Basket` component re-renders and the Text component showing the total will be updated.

### 6.1.1  Provisions for using `@Watch`


| @Watch supplementary variable decorator | Comment |
|----|-----|
| decorator parameter | Exactly one callback function must be supplied. Constant string, the string is quoted (see examples). Reference to a `(string) => void` custom component member function.  |
| Custom component variables that can be decorated | all decorated state variables. regular variables can not be watched. |
| Order of decorators | The `@State`, `@Prop`, `@Link`, etc decorator must precede the `@Watch` decorator`.  |

Callback function syntax:

| type | Comment |
|------------|---------|
| `(changedPropertyName : string) => void` | The function is a member function of the custom component. It takes the property name as a string input parameter. This is useful when using the same function as calpack to several watched properties. The function returns nothing. Failure of the application to implement the function is a syntax error that the eDSL compiler reports. |


* Developers are advised to pay attention to the risk of infinite loops. Loops can be caused by the property update callback function directly or indirectly mutating the same variable.
* To avoid loops mutating the @Watch decorated state variable inside the callback handler must be avoided.
* Developers should pay attention to performance. The property value update function delays component re-render (see processing steps above). The callback function should only perform quick computations. 
* calling an `async` function from a @Watch function is allowed. Same rules as for @Component lifecycle functions apply, see also the example in therein, [link](./general-ui-spec.md#226-lifecycle-callback-functions) .
* awaiting on a Promising (using `await`) inside a @Watch function is not allowed and not supported.

The ArkUI framework processes steps in this order:
1. an event handler function updates an observed variable value. OR. The linked property changes in LocalStorage/AppStorage.
2. property `@Watch` callback function executes synchronously after variable mutation 
    2.1 in case the callback function mutates other watched variables, execute their variable @Watch callback functions in the same and in other custom components.
3. event handler function of step 1 continues 
4. custom component owning the mutated variable of step 1 re-renders, so do other custom components whose state variables changed (through subsequent `@Watch handler` functions in step 2.1, or through `@Link`, etc., value synchronisation mechanisms).

A `@Watch` function is not called upon custom component variable initialization.
A @Watch function is called upon child custom component variable (e.g. `@Link` varibale) update by its parent (as happens during child component first render and re-render).

```TypeScript
@Component Child {
  @State @Watch(cb) count = 1;
}

@Component Parent {
  @State count = 1;
}
```


### 6.1.2 `@Watch` example with `@Link`

The following example is meant to clarify the processing steps in case of watching a `@Link` in a child component:

```TypeScript
class PurchaseItem {
  static NextId : number = 0;

  public id: number;
  public price: number;

  constructor(price : number) {
    this.id = PurchaseItem.NextId++;
    this.price = price;
  }
}

@Component
struct BasketViewer {

    @Link @Watch("onBasketUpdated") shopBasket: PurchaseItem[];
    @State totalPurchase : number = 0;

    updateTotal() : number {
        let total = this.shopBasket.reduce((sum, i) =>  sum + i.price , 0);

        // Apply discount if over 100EUR
        if (total >= 100) {
            total = 0.9 * total;
        }
        return total;
    }
    
    // @Watch cb
    onBasketUpdated(propName: string) : void {
        this.totalPurchase = this.updateTotal();
    }

    build() {
        Column() {
            ForEach(this.shopBasket,
                (item) => {
                   Text(`Price: ${item.price.toFixed(2)} €`)
                },
                item => item.id.toString()
            )
            Text(`Total: ${this.totalPurchase.toFixed(2)} €`)
        }
    }
}


@Entry
@Component
struct BasketModifier {

    @State shopBasket : PurchaseItem[] = [];

    build() {
        Column() {
            Button("Add to basket")
              .onClick(() => { this.shopBasket.push(new PurchaseItem(Math.round(100 * Math.random()))) })
            BasketViewer({ shopBasket: this.$shopBasket })
        }
    }
}
```

The processing steps
1. `BasketModifier` component Button.onClick adds an item to BasketModifier.shopBasket
2. `@Link` `BasketViewer.shopBasket` value changes 
2. state management framework calls `@Watch` function `BasketViewer.onBasketUpdated`. It updates BasketViewer.totalPurchase
3. Because of `@Link shopBasket` and `@State totalPurchase` variables changed, State management framework marks `BasketViewer` component for re-rendering. Re-rendering will happen asynchronously.
4. Because of `@State shopBasket` variable change, state management framework marks `BasketModifier` component for re-rendering. Re-rendering will happen asynchronously.

### 6.1.3 `@Watch` and custom component update example

The purpose of the following example is to clarify processing steps of component update and @Watch. Note that `count` is a `@State` in _both_ components. No @Link is used.

> GGr This example shows ACE `@Prop` decoration is not needed. The same functionality can be achieved as shown for count below.

```Typescript
@Component
struct TotalView {

   @Prop @Watch("onCountUpdated") count : number = 0;
   @State total : number = 0;

    // @Watch cb
    onCountUpdated(propName: string) : void {
      this.total += this.count;
    }

    build() {
       Text(`Total: ${this.total}`)
    }
}


@Entry
@Component
struct CountModifier {

    @State count : number = 0;

    build() {
        Column() {
            Button("add to basket")
              .onClick(() => { this.count++ })
            TotalView({ count: this.count })
        }
    }
}
```

Processing steps:

The processing steps
1. `CountModifier` component Button.onClick increases `CountModifier.count=1`;
2. Because of `@State count` variable change, state management framework marks `CountModifier` component for re-rendering. Re-rendering will happen asynchronously.
3. `CountModifier` re-renders, updates `CountView(count: this.count)`.
4. As part of the update of `CountView, CountView.count` value changes to 1.
5. State management framework invokes `@Watch` function `CountView.onCountUpdated`. This function updates `CountView.total`.
6. 2. Because of `@State count` and `total` variables change, state management framework marks `CountView` component for re-rendering. Re-rendering will happen asynchronously.
7. CountView re-renders, updates the `Text` component.


## 6.2 Application defined observed object variables `SubscribableAbstract`

 `SubscribableAbstract` allows an application to _notify relevant view model state changes by itself_ to the framework that require a UI update:

Abstract class `SubscribableAbstract` is part of the ArkUI SDK. An application extend `SubscribableAbstract` to implement its own View Model class that _notifies state changes by itself_. `SubscribableAbstract` notifies the decorated variable that a relevant change has occurred. Subsequently the decorated variable will trigger the same UI update logic as if the framework had observed a application state value change.

Currently `SubscribableAbstract` can be used together with `@State, @Link, @Prop, @ObjectLink, @Provide, and @Consume` decorated variables.

### 6.2.1 Scenarios when to define an own `SubscribableAbstract` class

As explained earlier in this specification `@State, @Link`, and other decorated variables can observe changes of class objects or arrays. `Object.keys()` enumerates the properties of a given TS object. Changes to all these object properties will be observed. Adding, removing or setting an Array item is observed. 

In some cases, though, this is not suitable for the application. This is where an application is recommended to have  its own `SubscribableAbstract` class:
* When framework's ES6 proxy can not detect a state change:
    - The class exposes a computed property (`get myProp() { return this.query(..); }`) that is used to construct the UI but `Object.keys(..)` will not include, hence the framework's built-in proxy functionality will not observe its change.
    - The class has a native implementation.
* The object includes some properties the UI does not depend on, hence, their mutation could lead to unnecessary UI refreshes.
* The application follows a design where UI implementation is not allowed to mutate state directly, but instead invoke 'actions' inside application logic that will update the state and notify the framework about the state change.

The following example illustrates:

```TypeScript

// application defined object that extends ObservedClass
// when sensing an 'action' randomize the object mutates
// and notifies the framework.
// Application can also mutate a parameter 'rangeTo' 
// which will also fire the 'randomize' action.
// Naturally 'random' property can not be set.
//
class RandomObservedObject extends SubscribableAbstract {

    private rangeTo_: number = 6;
    private randomNumber_: number = -1;

    constructor(rangeTo: number) {
        super();
        this.rangeTo_ = rangeTo;
        console.log("RandomObservedObject constructor done");
    }

    // action leading to mutation of state
    // with application specific logic
    public randomize(): void {
        this.randomNumber_ = Math.floor(Math.random() * this.rangeTo_);
        this.notifyPropertyHasChanged("random", this.random);
        console.log(`RandomObservedObject randomize this.random${this.random}`);
    }

    get random() {
        return this.randomNumber_;
    }

    get rangeTo(): number {
        return this.rangeTo_;
    }

    // set upper limit of range 0...N 
    //  Note mutating this property is not notified as a change
    //  because UI does not depend on rangeTo
    set rangeTo(newValue: number) {
        console.log(`RandomObservedObject rangeTo begin: this.random${this.random}`);
        this.rangeTo_ = newValue;
        console.log(`RandomObservedObject rangeTo end: this.random${this.random}`);
    }
}

// shared instance
let randomObject = new RandomObservedObject(6);


@Component
struct DiceLinkView {
    @Link randomizer: RandomObservedObject;
    label: string;

    build() {
        Button(`[${this.label} (Link)] Your number is: ${this.randomizer.random + 1}.`)
            .width(300)
            .height(100)
            .onClick(() => this.randomizer.randomize())
    }
}

@Component
struct DiceView {
    @State randomizer: RandomObservedObject = randomObject;
    label: string = "unknown";

    build() {
        Row() {
            Text(`[${this.label}]: Your number is: ${this.randomizer.random + 1}.`)
                .width(150)
                .height(100)
            DiceLinkView({ label: this.label, randomizer: this.$randomizer })
        }
    }
}

@Entry
@Component
struct MainView {
    @State randomizer: RandomObservedObject = randomObject;
    @State numberOfDice: number = 1;

    build() {
        Column() {
            DiceView({ label: "DiceView #1" })
            DiceView({ label: "DiceView #2" })

            Button("Throw the Dice!")
                .onClick(() => this.randomizer.randomize())
                .width(400)
                .height(100)
            Button(`Number of Dice ${this.numberOfDice}`)
                .onClick(() => {
                    this.numberOfDice = this.numberOfDice == 1 ? 2 : 1;
                    this.randomizer.rangeTo = this.numberOfDice == 1 ? 6 : 12;
                })
                .width(400)
                .height(100)
        }
    }
}
```

### 6.2.2 Provisions for using `SubscribableAbstract`

`SubscribableAbstract` is an ArkUI SDK class added for API rel. 10:

`SubscribableAbstract` is an abstract class that manages subscribers to value changes. These subscribers are the framework implementations of  `@State, @Link, @Prop, @ObjectLink, @Provide, and @Consume` decorated variables. A decorated variable shares the lifecycle of its owning component. When assigned an object of instance `SubscribableAbstract`, at variable creation or when re-assigning a new value later, the decorated variable subscribes to this object, when unassigned or when getting deleted it unsubscribes.

 About lifecycle of an `SubscribableAbstract` object: It is legal for two decorated variables to share the same instance to a `SubscribableAbstract` object. Each such decorated variable implementation makes its own subscription to the `SubscribableAbstract` object. Hence, when both variables have unsubscribed the `SubscribableAbstract` may do its own de-initialization, e.g. release held external resources.
 
How to extend:
A class that extends `SubscribableAbstract`manages the 'get' and 'set' to its properties by itself. The subclass needs to notify all relevant value changes to the framework for the UI to be updated. For reasons of efficiency notification should only be given for class properties that are used to generate the UI. 
* As with an derived class in ES6 it must call `super()` in its constructor to let the base class initialize itself.
* A subclass must call `notifyPropertyHasChanged` after a relevant property has changes. Subsequently the framework will notify all dependent components to re-render.


| Function | Parameters | Definition |
|----------|------------|------------|
| SubscribableAbstract constructor | none | subclass must call `super` |
| `protected notifyPropertyHasChanged` | `propName: string` <br/> `newValue : any` | name of changed property <br />value of the changed property after the change <br />the sub-class must call this function _after_ a property change. This triggers any necessary UI update. |
| `public numberOfSubscribers` | returns `number` | Function current number of subscribers |

How to use:
* `State, @Link, @Prop, @ObjectLink, @Provide, and @Consume`  decorated variables can hold an Object that is instance of `SubscribableAbstract`.
* Implementation note: Support for `SubscribableAbstract` for storage related decorators might be added later.


## 6.3  Built-in Component Two-way Value Synchronization - '$$'

The purpose of the `$$` operator is to provide a TS variable by-reference to a built-in component. The variable value and the internal state of that component are kept in sync. What that internal state is depends on the component. For example, for `TextField` it is the current input string value, for slider it is the slider numeric value.

### 6.3.1 Provisions for using `$$`

Use of the `$$` operator on different kinds of TS variables:

| case | notation |
|------|----------|
| component member variable | `$$this.varName` |
| global variable | `$$globalVarName` |
| object property by reference | `$$obj.aProperty` or `$$this.obj.aProperty` |
| array item | `$$arr[7]` |
| array item's property | `$$arr[7].aProperty` |

Further provisions for `$$`:
- The type of the provided variable must match the expected type of the using component. There is no implicit type conversion. Only simple types `number`, `string` and `boolean` are allowed.
- The variable can be a decorated or regular variable. If the variable is a global one it must be in scope as long as the accessing component remains created. 
- The variable must be writable (this excludes `@StorageProp` decorated variables).
- The normal component re-rendering process updates the bound built-in component's internal value. Hence, for a change of the bound TS variable to update the component internally, this change must be observed (see decorated variables).


## 6.4 AnimatableData

The way how to animate changes of application-defined custom data types is as follows:
1. The application class needs to implement `IAnimatableArithmetic` interface, it provides the `add`, `substract`, `multiply`, and `equal` function.
2. Implement a `@AnimatableExtend` function. The framework calls this function on each frame.


### 6.4.1 `IAnimatableArithmetic` interface

ArkUI supports animatable data of any application defined data model type. The only requirement is that the app's data model class implement the `IAnimatableArithmetic` interface.

The ArkUI SDK provides the following interface declaration.

```TypeScript
interface IAnimatableArithmetic<T> {
    // this + rhs => result
    plus(rhs: IAnimatableArithmetic<T>) : IAnimatableArithmetic<T>;

    // this - rhs => result
    subtract(rhs : IAnimatableArithmetic<T>) : IAnimatableArithmetic<T>;

    // this * scale => result
    multiply(scale: number) : IAnimatableArithmetic<T>;

    // this is same as rhs by deep comparison
    equals(rhs: IAnimatableArithmetic<T>): boolean;
}
```

For example for a list of (x, y) points as needed to draw a polyline the app needs to implement the
required matrix add, subtract, scale and equals calculations.


### 6.4.2  `@AnimatableExtend` special `@Extend` function

The purpose of an `@AnimatableExtend` decorated function is rendering animated data using one of ArkUI built-in UI components.

An `@AnimatableExtend` decorated function is a specialized global `@Extend` function. All rules for @Extend functions apply, and in addition.
* The only function parameter provides the animatable data, i.e. the parameter type must be instance of `IAnimatableArithmetic`
* The extended framework UI component renders the animatable data.
  
App developers are advised to keep this function extremely lightweight because it will be executed on every frame.

### 6.4.3 Example


Application defines its own AnimatableData class, here `PointVector`:

```TypeScript
// we have definition of type Point in the framework like this:
type Point = [ number, number ];

// PointClass = Array with two number entries
class PointClass extends Array<number> {
    constructor(value: Point) {
        super(value[0], value[1])
    }

    plus(rhs: PointClass): PointClass {
        let result = new Array<number>() as Point;
        for (let i = 0; i < 2; i++) {
            result.push(rhs[i] + this[i])
        }
        return new PointClass(result);
    }

    subtract(rhs: PointClass): PointClass {
        let result = new Array<number>() as Point;
        for (let i = 0; i < 2; i++) {
            result.push(this[i] - rhs[i]);
        }
        return new PointClass(result);
    }

    multiply(scale: number): PointClass {
        let result = new Array<number>() as Point;
        for (let i = 0; i < 2; i++) {
            result.push(this[i] * scale)
        }
        return new PointClass(result);
    }
}


class PointVector extends Array<PointClass> implements IAnimatableArithmetic<Array<Point>> {
    constructor(initialValue: Array<Point>) {
        super();
        if (initialValue.length) {
            initialValue.forEach(p => this.push(new PointClass(p)))
        }
    }

    // implement the IAnimatableArithmetic interface
    plus(rhs: PointVector): PointVector {
        let result = new PointVector([]);
        const len = Math.min(this.length, rhs.length)
        for (let i = 0; i < len; i++) {
            result.push(this[i].plus(rhs[i]))
        }
        return result;
    }
    subtract(rhs: PointVector): PointVector {
        let result = new PointVector([]);
        const len = Math.min(this.length, rhs.length)
        for (let i = 0; i < len; i++) {
            result.push(this[i].subtract(rhs[i]))
        }
        return result;
    }
    multiply(scale: number): PointVector {
        let result = new PointVector([]);
        for (let i = 0; i < this.length; i++) {
           result.push(this[i].multiply(scale))
        }
        return result;
    }
    equals(rhs: PointVector): boolean {
        if (this.length !== rhs.length) {
            return false;
        }
        for (let index = 0, size = this.length; index < size; ++index) {
            if (this[index][0] !== rhs[index][0] || this[index][1] !== rhs[index][1]) {
                return false;
            }
        }
        return true;
    }
}
```

Application defines `@AnimatableExtend` function, here it extends `Polyline` component:

```TypeScript
@AnimatableExtend(Polyline) function animatablePoints(points : PointVector) {
    .points(points)
}
```

Example for putting everything to use:

```TypeScript

function getRandomInt(max) {
    return Math.floor(Math.random() * max);
  }

@Entry
@Component struct AnimatedShape {

  
  // the animatable data
  @State points : PointVector = new PointVector([
      [0, getRandomInt(200)],
      [20, getRandomInt(200)],
      [40, getRandomInt(200)],
      [60, getRandomInt(200)],
      [80, getRandomInt(200)]
  ]);

  // some other state that we  change with animation
  @State color : string = Color.Green;

  build() {
      Column() {
              Polyline()
                  .width(400).height(400)
                  // change fill attribute value with animation
                  .fill(this.color)
                  .stroke(Color.Red)

                  // call the @AnimatableExtend function
                  .animatablePoints(this.points)

                  // change backgroundColor attribute value with animation
                  .backgroundColor(this.color)
                  .animation({duration: 3000, delay: 0, curve: "ease"})

              Button("Replace")
                  .width(200).height(70)
                  .onClick(() => {
                          // state change, causes Polyline to update
                          this.color = (this.color == Color.Green) ? Color.Pink : Color.Green;
                          this.points = new PointVector([
                                [0, getRandomInt(200)],
                                [20, getRandomInt(200)],
                                [40, getRandomInt(200)],
                                [60, getRandomInt(200)],
                                [80, getRandomInt(200)]
                            ]);
                   })
                Button("Scale")
                  .width(200).height(70)
                  .onClick(() => {
                          // state change, causes Polyline to update
                          this.color = (this.color == Color.Green) ? Color.Pink : Color.Green;
                          const scale = 0.5 + Math.random();
                          this.points = this.points.multiply(scale)
                   })
              Button("Replace Color")
                  .width(200).height(70)
                  .onClick(() => {
                          this.color = (this.color == Color.Green) ? Color.Pink : Color.Green;
                   })
       }
    }
  }
````



---

# 7 Rendering Control Syntax

ArkUI Declarative includes provisions for conditional content using  `if ... else if ... else ...` and for repeated content using `ForEach`. 

## 7.1 Conditional Rendering with `if`

 `if ... else if ... else ...` makes rendering of content conditional of a boolean expression. 

```typescript
@Entry
@Component 
struct ViewA {
    @State count : number = 0;
    build() {
        Column() {
            Text(`count=${this.count}.`)
            if (this.count > 0) {
                Text(`count is positive`)
                    .fontColor(Color.Green)
            } 
            Button("increase count")
              .onClick(() => this.count++)
            Button("decrease count")
              .onClick(() => this.count--)
        }
    }
}
```

Each branch of `if` includes a build function. Such build function must create one or more child components.
At initial render `if` will execute at most one build function, add the generated children to its parent component.

`if` updates whenever a state variable used inside the `if` condition or the `else if`  condition changes, re-evaluates the condiiton(s). If the evaluation of the conditions changes it means that another branch of `if` needs to be build. The framework will:
1. remove all previously rendered components (of earlier branch)
2. execute the build function of the branch, add the generated children to its parent component.

In above example, if `count` increases from 0 to 1, then, `if` updates, the conditon `count > 0` is re-evaluated, the evaluation result changes from `false` to `true`. Therefore, the positive branch build function will execute, it creates two `Text` components and prepends them to the `Column`.  If later `count` changes back to 0, then, these `Text` components will be removed from the `Column`. Since there is no `else` branch, no new build function will be executed.


### 7.1.1 `if  else ` and sub-component state

Consider this example involving `if ... else ...` and a sub-component with an @State variable:

``` Typescript
@Component
struct CounterView {

  @State counter : number = 0;
  label: string = "unknown";

  build() {
    Row() {
        Text(`${this.label}`)
        Button(`counter ${this.counter} +1`)
        .onClick( () => this.counter+=1 )
    }
  }
}

@Entry
@Component
struct MainView {

  @State toggle : boolean = true;

  build() {
    Column() {
        if (this.toggle) {
          CounterView({label: "CounterView #positive"})
        } else {
          CounterView({label: "CounterView #negative"})
        }
        Button(`toggle ${this.toggle}`)
          .onClick(() => this.toggle = !this.toggle )
    }
  }
}
```

On first render the `CounterView` '#positive' sub-component is created. It carries some of state, the `@State counter`. When mutating `CounterView.counter` state variable, the `CounterView` '#positive' sub-component re-renders, the state variable value is preserved.
When the value of `MainView.toggle` state variable changes to `false`, the `if` inside `MainView` parent updates and subsequently the `CounterView` '#positive' sub-component will be removed. Instead a new `CounterView` instance '#negative' will be created. Its own `@State counter` variable is set to initial value 0. 

Therefore, it is important to understand that `CounterView` '#positive' and `CounterView`'#negative' are _two distinct instances_ of the same custom component. When `if` branches change,  there is _no_ updating of an existing sub-components and _no_ preservation of state. 

Below test case shows the required modifications if the value of `counter` be preserved when the 
if condition changes:

``` Typescript
@Component
struct CounterView {

  @Link counter : number;
  label: string = "unknown";

  build() {
    Row() {
        Text(`${this.label}`)
        Button(`counter ${this.counter} +1`)
        .onClick( () => this.counter+=1 )
    }
  }
}

@Entry
@Component
struct MainView {

  @State toggle : boolean = true;
  @State counter : number = 0;

  build() {
    Column() {
        if (this.toggle) {
          CounterView({ counter: this.$counter, label: "CounterView #positive" })
        } else {
          CounterView({ counter: this.$counter, label: "CounterView #negative" })
        }
        Button(`toggle ${this.toggle}`)
          .onClick(() => this.toggle = !this.toggle )
    }
  }
}
```

Here, the `@State counter` variable is owned by the parent component. Therefore, it is not destroyed 
when a `CounterView` component instance is deleted. `CounterView` component refers to the state by a 
`@Link`. This technique is sometimes referred to as 'pushing up the state in the component tree`.
What this expression tries to describe is that state must be moved from a child to its parent (or parent of parent) to avoid losing it when conditional content gets destroyed (and also when repeated content get destroyed, explained in the next chapter)

### 7.1.2 Nested `if` statements

Nested `if` statements are permissible. The nesting makes no difference to the rule about the parent component. Also in this case the parent component is the Column container.

```TypeScript
@Entry
@Component
struct CompA {
    @State toggle : boolean = false;
    @State toggleColor: boolean = false;

    build() {
        Column() {
            Text("Before")
              .fontSize(15)

            if (this.toggle) {
               Text("Top True, positive 1 top")
                 .backgroundColor("#aaffaa").fontSize(20)

               if (this.toggleColor) {
                  Text("Top True, Nested True, positive COLOR  Nested ")
                  .backgroundColor("#00aaaa").fontSize(15)
               } else {
                  Text("Top True, Nested False, Negative COLOR  Nested ")
                  .backgroundColor("#aaaaff").fontSize(15)
               } // inner if
            } else {
               Text("Top false, negative top level").fontSize(20)
               .backgroundColor("#ffaaaa")
          
               if (this.toggleColor) {
                  Text("positive COLOR  Nested ")
                  .backgroundColor("#00aaaa").fontSize(15)
               } else {
                  Text("Negative COLOR  Nested ")
                  .backgroundColor("#aaaaff").fontSize(15)
           }
            }
            Text("After")
              .fontSize(15)
            Button("Toggle Outer")
              .onClick(() => { this.toggle = !this.toggle; } )

            Button("Toggle Inner")
              .onClick(() => { this.toggleColor = !this.toggleColor; } )

        }
    }
}
```

### 7.1.3 Provisions for using `if`, `else`, and `else if`

`if`, `else` and `else if` statements are supported. 

It is permissible to use `if` within any container component. 
`if`, `else` and `else if` statements are 'transparent' when it comes to the parent child relationship of components. Rules about permissible child components must be followed also when there is one or several `if` statement between the parent and the child component. 

The build function inside each branch must follow the special rules for build functions. Each such build function must create one or several top level components. An empty build functions is a syntax error.

`if` updates whenever a state variable used in its conditions changes value. `if` updates as follows:
1. evaluation of `if` and `else if` condition(s). If branch does not change, stop here. If branch changes:
2. delete all previously built sub-components.
3. execute the build function of the new branch and add obtained components to `if` parent container. In case of missing but applicable `else` branch do not build anything.

Developer advise:
* A condition can include Typescript expressions. As for any expression inside build functions such expression must not change any application state.


## 7.2 Repeated Content with `ForEach`

`ForEach` enables repeated content based on `array` type data:

```typescript
@Entry
@Component
struct MyComponent {
    @State arr: number[] = [10, 20, 30];

    build() {
        Column() {
            ForEach(this.arr,
                    (item) => {
                        Text(`item value: ${item}`)
                        Divider()
                    },
                    (item) => item.toString() 
            )
        }
    }
}
```

The above example generates a Text and a Divider component for each item of the `this.arr` array. `ForEach` has the same 'transparent' characteristic as `if` in regard to the component parent-child relationship: The logical parent component of the Text and Divider is the Column component.

`ForEach` has two mandatory and one optional parameter. 
* The 1st parameter must be of type `Array<T>`. 
* The 2nd and 3rd parameter are lambda functions that each will be called for each array item of type `T`.
* The 2nd parameter is the child build function. It is the same kind of special function as a @Component `build` function. 
* The 3rd, optional parameter creates a _unique_ index for each array item. It plays an important role how the framework identifies array changes. The exact definition follows later in this chapter.

### 7.2.1 `ForEach` and subcomponent state

Let's look at a bit more complex example:

```TypeScript
@Component
struct CounterView {
  label : string;
  @State count : number = 0;

build() {
      Button(`${this.label}-${this.count} click +1`)
        .width(300).height(40)
        .backgroundColor("#a0ffa0")
        .onClick(() => { this.count++ } )
  }
}


@Entry
@Component
struct MainView {
    @State arr: number[] = Array.from(Array(10).keys()); // [0.,.9]
    nextUnused : number = this.arr.length;

    build() {
        Column() {
            Button(`push new item`)
            .onClick(() => {
               this.arr.push(this.nextUnused++)
            })
            .width(300).height(40)

            Button(`pop last item`)
            .onClick(() => {
               this.arr.pop()
            })
            .width(300).height(40)

            Button(`prepend new item (unshift)`)
            .onClick(() => {
               this.arr.unshift(this.nextUnused++)
            })
           .width(300).height(40)

            Button(`remove first item (shift)`)
            .onClick(() => {
               this.arr.shift()
            })
            .width(300).height(40)

            Button(`insert at pos ${Math.floor(this.arr.length/2)}`)
            .onClick(() => {
               this.arr.splice(Math.floor(this.arr.length/2), 0, this.nextUnused++);
            })
            .width(300).height(40)

            Button(`remove at pos ${Math.floor(this.arr.length/2)}`)
            .onClick(() => {
               this.arr.splice(Math.floor(this.arr.length/2), 1);
            })
            .width(300).height(40)

            Button(`set at pos ${Math.floor(this.arr.length/2)} to ${this.nextUnused}`)
            .onClick(() => {
               this.arr[Math.floor(this.arr.length/2)]=this.nextUnused++;
            })
            .width(300).height(40)

            ForEach(this.arr,
                    (item) => {
                        CounterView({label: item.toString()})
                    },
                    (item) => item.toString()
            )
        }
    }
}
```

`MainView` owns an @State array of numbers. Adding, deleting and replacing an array item are observed mutation events. Whenever one of these happens the `ForEach` inside `MainView` is updated.

`CounterView` has some own state: `count : number`. When a new instance of `CounterView` is created and when an existing instance is updated?

This is the purpose of the item index function. This function creates an unique and persistent id for each array item: The framework determins if an item inside an array has changed by the id generated for this item. As long as the id is the same, the item value is assumed unchanged; its index position could have changed. For this to work two array items may never compute to the same id !

Using the computed item id, the framework can distinguish newly created, deleted and retained array items:
1. The framework will remove UI components for a removed array item.
2. The framework executes the item build function _only for newly added_ array items.
3. The framework will not execute the item build function for retained array items. If the item index within the array has changed it will just _move_ its UI components according to the new ordering, but will not update the UI components.
 

The item index function is recommended but optional to specify.  _Generated ids must be unique_, means the same id must not be computed for two items within the array. The id must be different even if two array items have same value.
* if the array item value changes the id must change

Common pitfalls: Violation of above rules for item id is the most common app development error. The mistake especially common for `Array<number>` when duplicate numbers are added during execution. 


Example: As mentioned already the id generation function is optional. `ForEach` in above example without the item index function:

```TypeScript
ForEach(this.arr,
        (item) => {
            CounterView(label: item.toString())
        }
)
```

If no item id function is provided the framework tries to be smart to detect specific array changes when updating `ForEach`. However, it might delete subcomponents and re-execute the item build function for array items that have moved (changed index) within the array. In above example, this changes application behavior in regard to the `counter` state of `CounterView`. When creating a new instance of `CounterView` the value of `counter` will  initialized with `0`.

Whenever it is important to preserve state of repeated sub-components, then, state needs to be 'pushed up the component tree' as explained for conditional sub-components already (see 'if'). The following example shows how to use `@ObjectLink` for doing so:

```TypeScript
let NextID : number = 0;

@Observed class MyCounter {
    public id : number;
    public c: number;

    constructor(c: number) {
      this.id = NextID++;
        this.c = c;
    }
}

@Component
struct CounterView {

    @ObjectLink counter : MyCounter;
    label : string = "CounterView";

    build() {
        Button(`CounterView [${this.label}] this.counter.c=${this.counter.c} +1`)
        .width(200).height(50)
        .onClick(() => {
            this.counter.c += 1;
        })
    }
}

@Entry
@Component
struct MainView {
  @State firstIndex : number = 0;
  @State counters : Array<MyCounter> = [new MyCounter(0), new MyCounter(0), new MyCounter(0),
                                        new MyCounter(0), new MyCounter(0) ];

  build() {
     Column() {
        ForEach (this.counters.slice(this.firstIndex, this.firstIndex+3),
            (item) => {
                CounterView({label: `Counter item #${item.id}`, counter: item})
            },
            (item) => item.id.toString()
        )

        Button(`Counters: shift up`)
            .width(200).height(50)
            .onClick(() => {
                this.firstIndex = Math.min(this.firstIndex+1, this.counters.length-3);
            })

        Button(`counters: shift down`)
            .width(200).height(50)
            .onClick(() => {
                this.firstIndex = Math.max(0, this.firstIndex-1);
            })
    }
  }
}
```

When increasing `firstIndex`, `ForEach` inside `Mainview` is updated and the `CounterView` sub-component associated with item id `firstIndex-1` gets deleted. For array item with id `firstindex + 3` a new `CounterView` sub-component instance is created. The counter state is preserved because `CounterView` instances do not own the state, they just link to the state owned by their parent component. Therefore, deleting an instance does not delete the state.

### 7.2.2 Nested use of `ForEach`

Nesting `ForEach` inside another `ForEach` in the same component is allowed, but not recommended.
It is better to split the component into two and have each `build` function include just one `ForEach`.

A _bad_ example how nested use of `ForEach`:

```TypeScript
class Month {
  year: number;
  month: number;
  days: number[];

  constructor(year: number, month: number, dayArray : number[]) {
    this.year = year;
    this.month = month;
    this.days = dayArray;
  }

  // ...
}

@Entry
@Component
struct CalendarData {
  // simulate with 6 months
  @State calendar : Month[] = [
    new Month(2020, 1, [...Array(31).keys()]),
    new Month(2020, 2, [...Array(28).keys()]),
    new Month(2020, 3, [...Array(31).keys()]),
    new Month(2020, 4, [...Array(30).keys()]),
    new Month(2020, 5, [...Array(31).keys()]),
    new Month(2020, 6, [...Array(30).keys()])
  ]

  build() {
    Column() {
      Button() {
        Text('next month')
      }.onClick(() => {
        this.calendar.shift()
        this.calendar.push(new Month(2020, 7, [...Array(31).keys()]))
      })

      ForEach(this.calendar,
        (item: Month) => {
          ForEach(item.days,
            (day : number) => {
              // build day block
            },
            (day : number) => day.toString()
          ) // inner ForEach
        },
        (item: Month) => (item.year * 12 + item.month).toString() // field is used together with year and month as the unique ID of the month.
      ) // outer ForEach
    }
  }
}
```

It is _not recommended_ to adopt above example. It is syntactically correct. However, it has two issues: 
1. Badly readable code
2. For a data structure of months and days of a year it might not be a requirement, but more generally speaking: The framework will not observe property changes to Month objects, including any changes to the days array; therefore the framework will not update the UI.

The recommended application design is to split 'Calendar' into 'Year', 'Month' and 'Day' sub-components.
Define a 'Day' model class to hold info about a day, and decorate this class `@Observed`. `DayView` component to utilize a `ObjectLink` decorate variable to link to data about a day. Do the same for `MonthView` and `Month` model class.

### 7.2.3 Example of ForEach using optional `index` parameter

It is possible to use optional `index` parameters in item build and id generation functions. 

```TypeScript
@Entry
@Component
struct ForEachWithIndex {

    @State arr : number[] = [4, 3, 1, 5];

    build() {
        Column() {
            ForEach(this.arr,
              (it, indx) => {
                Text(`Item: ${indx} - ${it}`)
              },
              (it, indx)  => {
                return `${indx} - ${it}`
              }
            )
        }
    }
}

Output:
Item: 0-4
Item: 1-3
Item: 2-1
Item: 3-5
```

The correct construction of the id generation function is essential. When `index` is used in the item generation function, then, the `index` parameter should also be used in the id generation function to produce a) unique ids and b) an id for given source array item that changes when its index position within the array changes.

This example also illustrates the significant performance disadvantage caused by the `index` parameter. If an item is moved within the source array without modification, the dependent UI still requires rebuilding because of the changed `index`. E.g. with use of index sorting, an array merely requires unmodified child UI node of `ForEach` to be _moved_ to correct slot, which is a lightweight operation for the framework. When using `index` all child UI nodes needs to be re-build, which is much more heavy weight. Therefore, three advices:
1. be considerate about using `index` in the item generation function
2. when using the `index` in the item build function then also use it in the id generation function
3. only specify the `index` parameter in the function signature if it is actually used in the function body.


### 7.2.4 Provisions for using `ForEach`

*1. Array parameter*:

The syntax is `Array<T>`:

The first parameter must be an `Array<T>`. Empty array is allowed and creates no children. A function that returns an array is allowed as well, e.g. `arr.slice(1, 3)`. As with anything expression inside a build function, such a function must not mutate any application state. Developers take caution that this requirement forbids functions that modify the array _in place_, such as `Array.splice`, `Array.sort`, or `Array.reverse`.

*2. Child build function mandatory parameter*:

The syntax is:
```JavaScript
`(item, index?) => void`
```

The second parameter is the child build lambda function. 
* `item` will be of type `T` as in first parameter `Array<T>`. Optional number parameter will be of type number. Types are not specified.
* provisions for build functions apply the function body. 
* the function must generate one or more top level components
* It generates one or several child components for given array item. Single and list of child components must be enclosed in curly braces `{ .... }`. 
* The component type of any sub-component must be allowed inside the parent container component of `ForEach` (e.g. `LitemItem` is allowed only when the `ForEach` parent is `List`)
* The child build function is allowed to return an `if` or another `ForEach`.  `ForEach` can be placed inside `if`.
* Optional `index` parameter should only be specified in the function signature if used in its body.

Developer advise: 
* Developers are advised to make no assumption on the order of item build functions. The execution order may not be the order of items inside the array.
*  Make no assumption either when items are build the first time. Currently initial render of `ForEach` builds all array items when the @Component renders first time, but future framework versions might change this behaviour to a more lazy behaviour.
* Using `index` has severe negative impact on UI update performance, try to avoid where feasible.
* When using `index` in the item generation function, also use `index` in the item index function.


*3. Item id generator function parameter*:

The syntax is:
```JavaScript
`(item, index)? => string`
```
Item wil be of type `T` as in first parameter `Array<T>`. index wil be of number type. Types do not need to be specified.

The optional third parameter is the item id generator anonymous function. It generates a unique and stable ID for given array item (and for its index position inside the array if the item build function uses `index`): 
* Two items inside the same array must never compute the same ID.
* If `index` is not used, then, an item's ID must not change when the item's position within the array changes. However, if `index` is used, then the ID must change when a mutated item is moved within the array.
* When an item is replaced by a new one (with different value), the ID of the replaced and the ID of the new item must be different. 
* if `index` is used in the item build function than it should also be used in the id generation function.
* The id generator function is not allowed to mutate any component state.
* The id generator function is optional. However, for performance reasons, it is *recommended to provide*, which enables the framework to better identify array changes. 
* When leaving away the id generation function the app needs to ensure that `JSON.stringify` can work with each item of the `ForEach` source array. That's because the framework uses a default id generation function that relies on `JSON.stringify` to stringifies each array item. 

### 7.2.5 Common pitfall with using `ForEach`

The most common mistake app developers make in connection with `ForEach` is that the id generation function returns the same value for two array items. Even if the value of two array items is the same, the id generation function must still return different value.

Adding an item once will success, but a second click will lead to sublicate '4' in id generation.:
```
@State arr : Array<number> = [1, 2, 3]
...
ForEach(this.arr, (item) => {
  ....
  },
  item => item.toString()
)
...
Button("add item")
  .onClick(() => {
    this.arr.push(4) // only works once!
  }
```

When the `ForEach` source array can include `undefined` items additional pitfalls arise:
* make sure the item build function works with `undefined` item value.
* make sure the id generation function generates _unique ids_ also in case of one or more `undefined` items.


Another pitfall is to use of `Array` functions that modify the array in place. These violates the rule that build functions must not mutate app state:

```TypeScript
ForEach(this.arr.reverse(), (item) => {
  ....
  },
  item => item.toString()
)
```

The use of an array of objects is very common.  The Text label in the following example will not update.
The reason is that the `@State arrOfObjects` observed array changes but no object property changes within.  `@State` decorator will not catch the change to property `aProp` in `this.arrOfObjects[0].aProp`:

```TyeScript
@State arrOfObjects : Array<ClassA> = [...]
...
ForEach(this.arrOfObjects, (item) => {
  Text(this.arrOfObjects.aProp)
  },
  item => item.id
)
...
Button("modify property")
  .onClick(() => {
    this.arrOfObjects[0].aProp
  }
```

The solution is to
* decorate `ClassA` with `@Observed` and 
* to render each instance of `ClassA` in a sub-component that has an `@ObjectLink classAObject : ClassA`. See examples on `@ObjectLink`.


---

# 8 SwiftUI - eDSL Feature comparison

> This section is non-normative. It needs some updating as well

SwiftUI - eDSL feature comparison. Readers need to be familiar with SwiftUI. To catchup with SwiftUI the following articles should be helpful: https://www.swiftbysundell.com/articles/swiftui-state-management-guide/ and https://developer.apple.com/documentation/swiftui/state-and-data-flow.

## 8.1 StateManagement comparison

SwiftUI State Mgmt and Re-rendering:
![SwiftUI State Mgmt and Re-rendering](figures/SwiftUI-SaDF-Overview.png)

| SwiftUI | eDSL | Actions | Comment |
|---------|-------|---|-----------------------------|
| @State  | @State | - |Property wrapper for local state data of this view. Observed variable set operations but no object property mutations. This is what @ObservedObject and @StateObject are for. https://www.hackingwithswift.com/quick-start/swiftui/what-is-the-state-property-wrapper |
| @StateObject | @State | - | Wraps a class-type variable. The class needs to confirm to ObservableObject protocol. The View allocates and owns the instance. New in XCode 12.5. https://www.hackingwithswift.com/quick-start/swiftui/what-is-the-stateobject-property-wrapper |
| @ObservedObject | @Link | - | Wraps a class-type variable. The class needs to confirm to ObservableObject protocol. The View does not own the instance. Gets initialized via the constructor. https://www.hackingwithswift.com/quick-start/swiftui/how-to-use-observedobject-to-manage-state-from-external-objects |
| @Binding | @Link | - | Wraps a local variable to create a two way data binding on a by value data type. Initialized via the constructor. https://developer.apple.com/documentation/swiftui/binding, and  https://www.hackingwithswift.com/quick-start/swiftui/what-is-the-binding-property-wrapper |
| View.environmentObject function | @Provide | implement | Supplies an ObservableObject to a view subhierarchy. https://developer.apple.com/documentation/swiftui/view/environmentobject(_:), and https://www.hackingwithswift.com/quick-start/swiftui/how-to-use-environmentobject-to-share-data-between-views |
| @EnvironmentObject | @Consume | -  | A property wrapper type for an observable object supplied by a parent or ancestor view. Match by class name.https://developer.apple.com/documentation/swiftui/environmentobject, and  https://developer.apple.com/documentation/swiftui/environmentobject |
| ObservableObject protocol | @Observed, @ObjectLink | - | To observe class instances the class needs to confirm to this protocol and needs to publish one or more properties with @Published. Initialized via the constructor. https://www.hackingwithswift.com/quick-start/swiftui/observable-objects-environment-objects-and-published |
| @Published | N/A | - | Make a class property observable in SwiftUI. In eDSL all class properties are observed. Will not support. https://www.hackingwithswift.com/quick-start/swiftui/what-is-the-published-property-wrapper |
| objectWillChange | SubscribaleAbstract base class | - | allows app to implement a class with own logic when state mutation should be notified. TODO in SwiftUI |
| N/A | @Prop | - | Not needed in SwiftUI, SwiftUI allows constructors: pass regular variable to component and init a @State decorated variable in its constructor |
| create link reference - $ | $ | - | create a binding (SwiftUI)/LinkReference(ACE) for two-way data sync when passing a @State or @Link variable to a child custom component/View in its constructor. Functionality in SwiftUI and in ACE is about the same |
| by-reference parameter - $ | `$$` | - | pass a state variable by-reference when creating a built-in component or calling an attribute/modifier function | 
| @Watch(cbFunc) | `@Warch(cbFunc)` | implement | Callback function invoked upon specific state variable mutation. SwiftUI API design to bind this variable to a View can be confusing.  https://www.hackingwithswift.com/quick-start/swiftui/how-to-run-some-code-when-state-changes-using-onchange |
| onAppear | aboutToAppear | - | SwiftUI: called after the View has appeared. https://www.vadimbulavin.com/swiftui-view-lifecycle/ |
| onDisappear | aboutToDisAppear | - | SwiftUI: called after the View has disappeared |
| ForEach | ForEach | - | |
| if, if .. else .. | if, if .. else .. | - | 
| @AppStorage, UserDefaults | ~ @StorageLink + AppStorage + PersistentStorage | see below | SwiftUI @AppStorage(key) is to read/write name - value pairs to UserDefaults, which is a persistent storage. UserDefault provides also an API. |
 | @SceneStorage | N/A | - | ACE does not have notion of a Scene, hence the distinction between scene specific property value and app-wide value can not be made. Will not / can not support. |
| @Environment, EnvironmentValue | AppStorage + Environment | - | Obtain info about the UI's environment. Will be different from device to device, some props might also change on the same device. Note: The mechanism is powerful in SwiftUI because the framework provides a large number of helpful  environment properties. ACE spec and implementation needs to focus on defining environment properties and that are of benefit to app developers. | AppStorage | see above | implement | ACE uses AppStorage as common abstraction layer that separates UI from business logic. AppStorage has a TS API for business logic to provide the implementation to sync btw properties btw AppStorage and  local and cloud-based data stores. @StorageLink(key) and @StorageProp(Key) to bind from UI to properties in AppStorage |
| FetchRequest, FetchedResults property wrappers | AppStorage TS API + HTTP Fetch | see below | ACE and SwiftUI take different approaches how to connect the UI to data stored in the cloud. SwiftUI gives the possibility to formulate HTTP Fetch requests inside UI logic. We chose to mandate a clear separation of UI and business logic.  SwiftUI @FetchRequest example: https://www.hackingwithswift.com/books/ios-swiftui/how-to-combine-core-data-and-swiftui |
| AppStorage + API | - | - | |
| Environment | - | - |  |
| PersistentStorage | - | - |  |
| @StorageLink, @StorageProp | - | - | |

## 8.2 Rendering comparison

The comparison of rendering solution in a table can only be superficial

| SwiftUI | eDSL | Actions | Comment |
|---------|-------|---|-----------------------------|
| View | @Component struct | - | View is a regular Swift language struct. In Swift a struct is a by-value type. Its properties are immutable. @Component is a TS class or struct. Either is an (by-reference) Object and its properties are mutable. There is more room to compiler optimizations in Swift due to this difference. |
| @Component constructors not allowed | View.init (constructor) | left for future | SwiftUI supports: View constructor to define the Component/View API, e.g. what component members can be initialized and what not. Implement custom initialization logic. Both are missing in eDSL. |
| state variables | state variable | - | Both Views and @Components can have state variables |
| only immutable regular Swift variables | mutable regular TS variables | - | @Component can have regular and mutable TS variables, View properties are immutable (State variables are kept separately from the View) | 
| modifier functions | attribute functions | - | SwiftUI modifier functions are defined as Extensions of View. |
| order of modifier functions matters | order of modifier functions does not matter, except for .animate | build process re-architect, complete .animate | The order in which modifier functions are applied matters in swiftUI, e.g, Column{...}.background(Color.red).frame(200, 200) and Column{...}.frame(200, 200).background(Color.red) wlll draw a red box of different size. The only case where the order of attribute function matters in ACE is with .animate. Here ACE follows the SwiftUI examples |
| View Extension | @Extend app defined attribute functions | consider improvements in the future | View extensions are more powerful. ACE @Extend functions only support the case of applying common styling  |
| ViewModifier protocol, .modifier(ViewModifier) | N/A | left for future | A View Extension can not have own state, because it is just a function. When this is insufficient, SwiftUI allows to implement more complex functionality in an own class and to provide an instance to the `.modifier` modifier function. Support from the build process is needed to maintain that instance. |
| - | @Extend | - | Custom defined attribute functions in ACE. Simpler to write than View Extensions in SwiftUI, but also not as powerful |
| Variable of type View | N/A | left for future | In SwiftUI a View is a regular Swift type, therefore a variable of type View is permissible |
| function return value of type View | @Builder | implement | These functions are regular Swift functions in SwiftUI. Not special limitations. @Builder functions can only be called from inside build(), same limitations to @Builder function as for build   |
| modifier function with parameter of type View | N/A | left for future | Examples for such modifier functions: .alert, background, .overlay |
| build | body | left for future to make more flexible | In SwiftUI the build function is a regular Swift function. Its DSL is implemented in Swift. Any app function can use the same DSL or even create its own one to generate a View hierarchy. To allow app to define additional build functions, ACE supports @Builder. For defining a DSL in Swift see https://www.swiftbysundell.com/articles/deep-dive-into-swift-function-builders/ |
| comments allowed inside body | no comments allowed inside build | - | |
| regular Swift code only in event handlers | TS code only in event handlers | - |
| widthAnimation function inside event handler | `animateTo` inside event handler | - | widthAnimation is mentioned here because it requires deep support inside the build process. The ACE solution is expected to be similar once implemented |
| withTransaction | N/A/ left for future | |
| call functions from body function that return a View. | @Builder | - | Lightweight alternative to custom component without own state in ACE. Can call these functions from build() |
| GeometryReader | GeometryReader | prototyping | get info about available layout space and use to render sub-components. GeometryReader requires deep support inside the build and layout process |
