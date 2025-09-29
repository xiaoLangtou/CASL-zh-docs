---
title: "@casl/ability/extra API"
categories: [api]
order: 15
meta:
  keywords: ~
  description: ~
---

## rulesToQuery

这是一个辅助迭代器函数，允许将权限中的条件聚合到数据库查询中。

* **参数**：
  * `ability: TAbility`
  * `action: string`
  * `subjectType: SubjectType`
  * `convert: (rule: RuleOf<TAbility>) => object`
* **返回值** 如果用户不被允许在指定的 `subjectType` 上运行指定的 `action`，则返回 `null`，否则返回一个包含可选 `$and` 和 `$or` 字段的对象。`$and` 包含来自反向规则的转换结果，`$or` 包含直接规则的结果。
* **另请参阅**：[能力到数据库查询](../../advanced/ability-to-database-query)，[@casl/mongoose](../../package/casl-mongoose#accessible-records-plugin)

## rulesToAST

此函数将规则转换为 [ucast](https://github.com/stalniy/ucast) AST。

* **参数**：
  * `ability: TAbility`
  * `action: string`
  * `subjectType: SubjectType`
* **返回值** 如果用户不被允许在指定的 `subjectType` 上运行指定的 `action`，则返回 `null`，否则返回 AST。

## rulesToFields

这是一个辅助函数，允许从 `PureAbility` 条件中提取字段值。这对于从新对象的权限中提取默认值可能很有用。

* **参数**：
  * `ability: TAbility`
  * `action: string`
  * `subjectType: SubjectType`
* **返回值** 一个包含条件值的对象。
* **用法**\
  此函数处理不是对象的条件值。确保在结果对象上调用 `ability.can`：

  ```ts
  import { defineAbility } from '@casl/ability';
  import { rulesToFields } from '@casl/ability/extra';

  const ability = defineAbility((can) => {
    can('read', 'Article', { authorId: 1 });
    can('read', 'Article', { public: true });
    can('read', 'Article', { title: { $regex: /^\[Draft\]/i } });
  });

  const defaultValues = rulesToFields(ability, 'read', 'Article');
  console.log(defaultValues); // { public: true, authorId: 1 }

  const newArticle = {
    ...defaultValues,
    title: '...',
    description: '...'
  };
  ```

## permittedFieldsOf

This function returns fields of `subject` which specified `action` may be applied on. Accepts single generic parameter `TAbility` (`T` to be short).

* **Parameters**
  * `ability: T`
  * `action: string`
  * `subject: Subject`
  * `options: PermittedFieldsOptions<T>`
* **Returns** an array of fields
* **Usage**\
  This function is especially useful for backend API because it allows to filter out disallowed properties from request body (e.g., in [expressjs](https://expressjs.com/) middleware)

  ```ts
  import { defineAbility } from '@casl/ability';
  import { permittedFieldsOf } from '@casl/ability/extra';
  import { pick, isEmpty } from 'lodash';

  const ability = defineAbility((can) => {
    can('read', 'Article');
    can('update', 'Article', ['title', 'description']);
  });

  app.patch('/api/articles/:id', async (req, res) => {
    const updatableFields = permittedFieldsOf(ability, 'update', 'Article', {
      fieldsFrom: rule => rule.fields || [/* list of all fields for Article */]
    });
    const changes = pick(req.body, updatableFields);

    if (isEmpty(changes)) {
      res.status(400).send({ message: 'Nothing to update' });
      return;
    }

    await updateArticleById(id, changes);
  });
  ```

  So, now even if user try to send fields that he is not allowed to update, `permittedFieldsOf` will filter them out!

* **See also**: [Restricting fields access](../../guide/restricting-fields), [@casl/mongoose](../../package/casl-mongoose#accessible-fields-plugin)

## packRules

This function **reduces serialized rules size in 2 times** (in comparison to its raw representation), by converting objects to arrays. This is useful if you plan to cache rules in [JWT](https://en.wikipedia.org/wiki/JSON_Web_Token) token.

> Don’t use result returned by packRules directly, its format is not public and may change in future versions.

* **Parameters**:
  * `rules: TRawRule[]`
  * `packSubject?: (subjectType: SubjectType) => string` - we need to pass this parameter only if we use classes as subject types. It should return subject type's string representation.
* **Returns** `PackRule<TRawRule>[]`
* **Usage**

  ```ts
  import { packRules } from '@casl/ability/extra';
  import jwt from 'jsonwebtoken';
  import { defineRulesFor } from '../services/appAbility';

  app.post('/session', (req, res) => {
    const token = jwt.sign({
      id: req.user.id,
      rules: packRules(defineRulesFor(req.user))
    }, 'jwt secret', { expiresIn: '1d' });

    res.send({ token });
  });
  ```
* **See also**: [Cache abilities](../../cookbook/cache-rules), [`unpackRules`](#unpack-rules)

## unpackRules

This function unpacks rules previously packed by [`packRules`](#pack-rules), so they can be consumed by `PureAbility` instance.

* **Parameters**:
  * `rules: PackRule<TRawRule>[]`
  * `unpackSubject?: (type: string) => SubjectType` - we need to pass this parameter only if we use classes as subject types. It should return subject type out of its string representation.
* **Returns** `TRawRule[]`
* **Usage**\
  If backend sends packed rules, we need to use `unpackRules` before passing them into `PureAbility` instance:

  ```ts
  import { unpackRules } from '@casl/ability/extra'
  import jwt from 'jsonwebtoken';
  import ability from '../services/appAbility';

  export default class LoginComponent {
    login(params) {
      return http.post('/session')
        .then((response) => {
          const token = jwt.decode(response.token);
          ability.update(unpackRules(token.rules))
        });
    }
  }
  ```
* **See also**: [Cache abilities](../../cookbook/cache-rules), [`packRules`](#pack-rules)
