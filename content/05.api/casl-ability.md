---
title: "@casl/ability API"
categories: [api]
order: 10
meta:
  keywords: ~
  description: ~
---

`@casl/ability` 包含 2 个模块：

* 核心模块，提供 `PureAbility` 和其他核心类

  ```ts
  import * as core from '@casl/ability';
  ```

* 额外模块，提供额外的辅助函数

  ```ts
  import * as extra from '@casl/ability/extra';
  ```

本页面仅描述核心包的 API 文档。查看 [@casl/ability/extra API](/api/casl-ability-extra) 以获取有关辅助函数的信息。

> 本页面上的所有示例都使用 [TypeScript](https://www.typescriptlang.org/)

## PureAbility

`PureAbility` 是一个实现权限检查功能的基类。前缀"Pure"与函数式编程无关。它只是意味着这个类没有预定义的配置。

它是一个接受 2 个参数的泛型类：

1. `Abilities` 要么是表示可能操作的字符串字面量类型，要么是表示所有可能主体上所有可能操作的 2 元素元组。默认等于 `[string, Subject]`
2. `Conditions` 是条件的形状。对此参数没有限制，因此它可以是任何东西。

例如：

```ts
import { PureAbility, Abilities } from '@casl/ability';

type ClaimAbility = PureAbility<string>;
type AppAbility = PureAbility<[string, string]>;
```

**另请参阅**：[TypeScript 支持](/advanced/01.typescript)

### PureAbility constructor

* **Parameters** (`A` is shortened from `Abilities`):
  * `rules: RawRuleFrom<A, Conditions>[] = []`
  * `options: AbilityOptions<A, Conditions> = {}`
    * `detectSubjectType?: (subject?: Subject) => string`
      allows to check [subject type detection](../../guide/subject-type-detection  ) logic
    * `conditionsMatcher?: ConditionsMatcher<Conditions>`
      allows to change the language to match conditions. See [Customize Ability](../../advanced/customize-ability) for details
    * `fieldMatcher?: FieldMatcher`
      allows to change the logic how fields are matched. See [Custom field matcher](../../advanced/customize-ability#custom-field-matcher) for details
    * `resolveAction?: ResolveAction<Normalize<A>[0]>`
      allows to pass a function that resolves aliases to actions. See [Define action aliases](../../guide/define-aliases) for details

* **Usage**:

  ```ts
  const ability = new PureAbility<['read' | 'update', 'Article']>([
    { action: 'read', subject: 'Article' },
    { action: 'update', subject: 'Article' },
  ]);
  ```
* **See also**: [Define rules](../../guide/define-rules)

### update

Updates rules of `PureAbility` instance. This method completely replaces all previously define rules with new ones.

* **Parameters**:
  * `rules: RawRuleFrom<A, Conditions>[]`
* **Returns**: `this`
* **Emits**:
  * `update` - before updating instance
  * `updated` - after instance has been updated
* **Usage**\
  Use it when you need to update permissions. In the common scenario, you need to do this on login, logout and when user permissions are changed.

  ```ts
  import { PureAbility } from '@casl/ability';

  const ability = new PureAbility([{ action: 'manage', subject: 'all' }]);
  ability.update([]); // took back all permissions
  ```
* **See also**: [CASL guide](/guide/intro)

### can of PureAbility

Checks that the provided action and subject satisfy permissions. Depending on `Abilities` generic parameter this function accepts either single `action` argument (when `Abilities` is a string) or 3 arguments (when `Abilities` is a tuple).

* **Parameters**
  * `action: string` - an action to check
  * `subject: Subject` - a subject to check
  * `field?: string` - a field to check
* **Returns**: `boolean`
* **Usage**:

  ```ts
  import { PureAbility } from '@casl/ability';

  const claimAbility = new PureAbility<'read' | 'update'>();
  // the 1st generic is a string, so the method accepts only single parameter
  claimAbility.can('read');

  const ability = new PureAbility<['read' | 'update', 'Article']>();
  // the 1st argument is a tuple, so the method accepts 2-3 parameters
  ability.can('read', 'Article');
  ```
* **See also**: [CASL guide](/guide/intro)

### cannot of PureAbility

This method works the same way as [can](#can-of-pure-ability) but returns inverted result.

### relevantRuleFor

This method returns a rule that matches provided action, subject and field. If rule cannot be found it returns `null`. May be useful for debugging.

* **Parameters**: accepts the same parameters as [can](#can-of-pure-ability)
* **Returns**: `Rule<Abilities, Conditions> | null`
* **See also**: [Debugging and testing](../../advanced/debugging-testing)

### rulesFor

This method returns all registered rules for provided action, subject and field. Useful for debugging and for extensions. Contrary to `relevantRuleFor` accepts subject type as the 2nd argument or:

* **Parameters**:
  * `action: string` - an action to check
  * `subjectType: SubjectType` - a subject type to check (this one is optional if `Abilities` is a string literal type)
  * `field?: string` - a field to check
* **Returns**: `Rule<Abilities, Conditions>[]`

### possibleRulesFor

Similar to [rulesFor](#rules-for) but it accepts only up to 2 parameters (depending on the type of 1st generic parameter). Returns all possible rules ignoring field level restrictions. Useful for debugging and for extensions.

* **Parameters**:
  * `action: string`
  * `subjectType: SubjectType`
* **Returns**: `Rule<Abilities, Conditions>[]`

### on

Allows to register event handler on specific event. Currently, only 2 events are supported:

* `update`, triggered before an instance is updated
* `updated`, triggered after an instance is updated

* **Parameters**:
  * `event: 'update' | 'updated'`
  * `handler: (event: Event) => void`
* **Returns**: a function that removes event handler
* **Usage**\
  Useful for frontend frameworks integration. Usually you have a single ability instance in the app and a lot of places which need to recheck permissions when the instance's rules are changed.

### rules property of PureAbility

Returns an array of `RawRule`s passed in `PureAbility` constructor.

### detectSubjectType

Can be used to detect subject type of object. Works for both subject types and subject instances.

* **Parameters**:
  * `subject: Subject`
* **Returns**: `string`

## Ability (deprecated by createMongoAbility)

`Ability` extends [`PureAbility`](#pure-ability). It sets default values for 2 options:

* `conditionsMatcher` into [`mongoQueryMatcher`](#mongo-query-matcher)
* `fieldMatcher` into [`fieldPatternMatcher`](#field-pattern-matcher).

It also enforces `MongoQuery` restriction on the `Conditions` generic parameter. By default, it is `Ability<Abilities, MongoQuery>`.

## createMongoAbility

This is a factory function that creates an instance of `PureAbility` with Mongo-like conditions to restrict access.

## AbilityBuilder

This class allows to construct `PureAbility` instance in declarative way. It accepts a single generic parameter `T extends AnyAbility`. Usually, we don't need to provide it, as TypeScript will always infer it for us (just don't forget to pass class of `Ability` in constructor).

### AbilityBuilder constructor

* **Parameters**:
  * `AbilityType: AbilityClass<TAbility>`
* **Usage**:

  ```ts
  import { AbilityBuilder, createMongoAbility } from '@casl/ability';

  const { can, build } = new AbilityBuilder(createMongoAbility);
  ```
* **See also**: [Define rules](../../guide/define-rules)

### can of AbilityBuilder

Registers a `RawRule` instance in `rules` array. Depending on the passed in generic parameter, this function accepts either single `action` argument (when `Abilities` generic of `PureAbility` is a string) or 3 arguments (when `Abilities` is a tuple). In general it accepts 1-4 parameters.

* **Parameters**: this method has 2 overloads
  * `action: string | string[]`
  * `subjectType: string | Function`
  * `fields: string[]`
  * `conditions: Conditions`
  * **and**
  * `action: string | string[]`
  * `subjectType: string | Function`
  * `conditions: Conditions`
* **Returns** `RuleBuilder`, a class that allows to further change constructed `RawRule` (e.g., add forbidden reason).
* **Usage**:

  ```ts
  import { AbilityBuilder, PureAbility, createMongoAbility, AbilityClass } from '@casl/ability';

  // action only Ability type
  type ClaimAbility = PureAbility<'read' | 'update'>;
  const ClaimAbility = PureAbility as AbilityClass<ClaimAbility>;
  const { can, build } = new AbilityBuilder(ClaimAbility);

  can('read');
  can('update');

  // or action and subject Ability type
  const { can, build } = new AbilityBuilder(createMongoAbility);

  can('read', 'Article', { private: true });
  can('read', 'User', ['firstName', 'lastName']);

  const ability = build();
  ```
* **See also**: [Define rules](../../guide/define-rules)

### cannot of AbilityBuilder

Registers an inverted `RawRule` inside `AbilityBuilder`. Accepts the same parameters and has the same behavior as [can method](#can-of-ability-builder).

### build

Builds an instance of provided `Ability` type.

* **Parameters**:
  * `options?: AbilityOptionsOf<TAbility>`
* **Returns** a new `TAbility` instance

### rules property of AbilityBuilder

Contains an array of `RawRule`s registered by [`can`](#can-of-ability-builder) and [`cannot`](#cannot-of-ability-builder) methods.

## defineAbility

This function allows to define [`MongoAbility`](#ability) instance in a compact form. Cannot be used to create `PureAbility` instances. It's very useful for writing tests and documentation.

* **Signature** (`T` is `TAbility`):
  * `<T extends AnyAbility>(define: DSL<T, void>, options?: AbilityOptionsOf<T>) => T`
  * `<T extends AnyAbility>(define: DSL<T, Promise<void>>, options?: AbilityOptionsOf<T>) => Promise<T>`
* **See also**: [Define rules](../../guide/define-rules)

## ForbiddenError

This class is useful if you want to stop execution of the next code in case if user doesn't have permission to do something. Has private `constructor`, so cannot be instantiated with `new` (in TypeScript). It's a generic class that accepts `TAbility` as a single parameter. For example:

```ts
import { ForbiddenError, defineAbility } from '@casl/ability';

const ability = defineAbility((can) => {
  can('read', 'User')
});

ForbiddenError.from(ability).throwUnlessCan('read', 'Article');
// fetch article from database
```

### static from

Creates a `ForbiddenError` instance from provided `Ability` instance.

* **Parameters**:
  * `ability: TAbility`
* **Returns** `ForbiddenError` instance

### static setDefaultMessage

Allows to change default error message for all `ForbiddenError`s. By default, the error is `Cannot execute "${error.action}" on "${error.subjectType}"`

* **Parameters**:
  * `messageOrFn: string | GetErrorMessage` - message or function that returns string
* **Returns** void
* **Usage**:

  ```ts
  import { ForbiddenError } from '@casl/ability';

  ForbiddenError.setDefaultMessage('Not authorized');
  // or more verbose
  ForbiddenError.setDefaultMessage(error => `You are not allowed to ${error.action} on ${error.subjectType}`);
  ```

### setMessage

Changes message of a particular `ForbiddenError` instance.

* **Parameters**:
  * `message: string`
* **Returns** `this`
* **Usage**:

  ```ts
  import { ForbiddenError } from '@casl/ability';
  import ability from './appAbility';

  ForbiddenError.from(ability)
    .setMessage('You cannot update posts')
    .throwUnlessCan('update', 'Post');
  ```

### throwUnlessCan

Accepts the same parameters as [PureAbility's can method](#can-of-pure-ability). Throws a `ForbiddenError` if user cannot do the provided action on provided subject.

## getDefaultErrorMessage

A function that returns default error message for `ForbiddenError`. Is useful to revert back default message after setting it using [`ForbiddenError.setDefaultMessage`](#static-set-default-message).

## fieldPatternMatcher

This factory function accepts an array of fields and creates a function that matches fields by patterns using wildcards (i.e., `*`). It's used as a default field matcher option in `Ability` class.

* **Factory Parameters**:
  * `fields: string[]`
* **Factory Returns** `MatchField`
* **Matcher Parameters**:
  * `field: string`
* **Matcher Returns** `boolean`
* **Usage**

  ```ts
  import { fieldPatternMatcher } from '@casl/ability';

  const matchField = fieldPatternMatcher(['name', 'email', 'address.**']);
  console.log(matchField('name')); // true
  console.log(matchField('address.street')); // true
  ```

* **See also**: [Ability API](#ability), [Restricting field access](../../guide/restricting-fields), [Customize Ability](../../advanced/customize-ability)

## mongoQueryMatcher

This factory function creates a matcher that matches subjects based on [MongoDB query language](http://docs.mongodb.org/manual/reference/operator/query/). It's used as a default field matcher option in `Ability` class.

* **Factory Parameters**:
  * `conditions: MongoQuery`
* **Factory Returns** `ConditionsMatcher<MongoQuery>`
* **Matcher Parameters**:
  * `object: Record<PropertyKey, any>`
* **Matcher Returns** `boolean`
* **Usage**\
  Can be passed as `conditionsMatcher` option to `Ability` and `PureAbility` classes

  ```ts
  import { mongoQueryMatcher } from '@casl/ability';

  const matchConditions = mongoQueryMatcher({ authorId: 1, private: true });
  console.log(matchConditions({ authorId: 2 })); // false
  console.log(matchConditions({ authorId: 1, private: true })); // true
  ```

* **See also**: [Ability API](#ability), [Conditions in depth](../../guide/conditions-in-depth), [Customize Ability](../../advanced/customize-ability)

## buildMongoQueryMatcher

This is a factory of factory. It allows to extend `mongoQueryMatcher` with custom mongo operators. See [ucast](https://github.com/stalniy/ucast) for details

* **Factory Parameters**:
  * `parsingInstructions: Record<string, ParsingInstruction>`
  * `interpreters: Record<string, JsOperator>`
* **Factory Returns** extended `mongoQueryMatcher`
* **Usage**\
  The result of this function can be passed as `conditionsMatcher` option to `Ability` and `PureAbility` classes.
* **See also**: [mongoQueryMatcher API](#mongo-query-matcher), [Customize Ability](../../advanced/customize-ability)

## createAliasResolver

Creates a function that resolves alias to real actions. Can be passed as `resolveAction` option to `Ability` and `PureAbility` classes.

* **Parameters**:
  * `aliasMap: AliasMap`
* **Returns** `(action: string | string[]) => string | string[]`
* **See also**: [Define Action Aliases](../../guide/define-aliases)

## detectSubjectType

The default subject type detection logic. Can be passed as `detectSubjectType` option to `Ability` and `PureAbility` classes. Can be used to extend the default logic. The checking logic is the following:

* if `subject` is `undefined`, returns `all`
* if `subject` is a `ForcedSubject`, returns its forced type
* if `subject` is an object, returns `constructor.modelName || constructor.name`

* **Parameters**:
  * `subject?: {}`
* **Returns** a string (i.e., subject type)
* **See also**: [Subject type detection](../../guide/subject-type-detection  )

## subject

Sets subject type for a plain object. You shouldn't use this if you use classes or define custom `detectSubjectType` algorithm. Accepts 2 generic parameters `TSubjectType` and `TObject`.

* **Parameters**:
  * `subjectType: TSubjectType`
  * `object: TObject`
* **Returns** `TObject & ForcesSubject<TSubjectType>`
* **See also**: [Subject type detection](../../guide/subject-type-detection  )
