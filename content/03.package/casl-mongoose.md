---
title: CASL Mongoose
categories: [package]
order: 110
meta:
  keywords: ~
  description: ~
---

# CASL Mongoose

[![@casl/mongoose NPM version](https://badge.fury.io/js/%40casl%2Fmongoose.svg)](https://badge.fury.io/js/%40casl%2Fmongoose)
[![](https://img.shields.io/npm/dm/%40casl%2Fmongoose.svg)](https://www.npmjs.com/package/%40casl%2Fmongoose)
[![Support](https://img.shields.io/badge/Support-github%20discussions-green?style=flat&link=https://github.com/stalniy/casl/discussions)](https://github.com/stalniy/casl/discussions)

该包集成了 [CASL] 和 [MongoDB]。换句话说，它允许基于 CASL 规则从 MongoDB 获取记录，并回答诸如"哪些记录可以读取？"或"哪些记录可以更新？"等问题。

## 安装

```sh
npm install @casl/mongoose @casl/ability
# or
yarn add @casl/mongoose @casl/ability
# or
pnpm add @casl/mongoose @casl/ability
```

## 使用方法

`@casl/mongoose` 不仅可以与 [mongoose] 集成，还可以通过新的 `accessibleBy` 辅助函数与任何 [MongoDB] JS 驱动程序集成。

### `accessibleBy` 辅助函数

这个巧妙的辅助函数允许将权限规则转换为 MongoDB 查询，并仅从数据库中获取可访问的记录。它可以与 mongoose 或 [MongoDB 适配器][mongo-adapter] 一起使用：


#### MongoDB 适配器

```js
const { accessibleBy } = require('@casl/mongoose');
const { MongoClient } = require('mongodb');
const ability = require('./ability');

async function main() {
  const db = await MongoClient.connect('mongodb://localhost:27017/blog');
  let posts;

  try {
    posts = await db.collection('posts').find(accessibleBy(ability, 'update').ofType('Post'));
  } finally {
    db.close();
  }

  console.log(posts);
}
```

This can also be combined with other conditions with help of `$and` operator:

```js
posts = await db.collection('posts').find({
  $and: [
    accessibleBy(ability, 'update').ofType('Post'),
    { public: true }
  ]
});
```

**Important!**: never use spread operator (i.e., `...`) to combine conditions provided by `accessibleBy` with something else because you may accidentally overwrite properties that restrict access to particular records:

```js
// returns { authorId: 1 }
const permissionRestrictedConditions = accessibleBy(ability, 'update').ofType('Post');

// DANGER DO NOT DO THIS (see above use $and)
const query = {
  // This is bad and potentially wrong code
  ...permissionRestrictedConditions, 
  authorId: 2
};
```

In the case above, we overwrote `authorId` property and basically allowed non-authorized access to posts of author with `id = 2`

If there are no permissions defined for particular action/subjectType, `accessibleBy` will return `{ $expr: { $eq: [0, 1] } }` and when it's sent to MongoDB, database will return an empty result set.

#### Mongoose


```js
const Post = require('./Post') // mongoose model
const ability = require('./ability') // defines Ability instance

async function main() {
  const accessiblePosts = await Post.find(accessibleBy(ability).ofType('Post'));
  console.log(accessiblePosts);
}
```

从历史上看，`@casl/mongoose` 旨在与 [mongoose] 进行超级简单的集成，但现在由于在 TS 中处理 mongoose 类型的复杂性，我们将其重新定位为更具 MongoDB 特定性的包。这些插件仍然提供但已弃用，我们鼓励您在应用程序级别编写自己的插件或使用 `accessibleBy` 和 `accessibleFieldsBy` 辅助函数

### 可访问记录插件

> **已弃用**：此插件已弃用，推荐使用 `accessibleBy` 辅助函数。它将在下一个主要版本中被移除。

此插件为 mongoose `Model` 和 `Query` 添加了 `accessibleBy` 方法。`accessibleBy` 方法返回 `Query` 实例，因此您可以将其与其他方法链式调用。

要全局添加此插件：

```js
const { accessibleRecordsPlugin } = require('@casl/mongoose');
const mongoose = require('mongoose');

mongoose.plugin(accessibleRecordsPlugin);
```

> Make sure to add the plugin before calling `mongoose.model(...)` method. Mongoose won't add global plugins to models that where created before calling `mongoose.plugin()`.

或者添加到特定模型：

```js @{data-filename="Post.js"}
const mongoose = require('mongoose')
const { accessibleRecordsPlugin } = require('@casl/mongoose')

const Post = new mongoose.Schema({
  title: String,
  author: String
})

Post.plugin(accessibleRecordsPlugin)

module.exports = mongoose.model('Post', Post)
```

之后，我们可以在 `Post` 上使用 `accessibleBy` 方法获取可访问的记录：

```js
const Post = require('./Post')
const ability = require('./ability') // defines Ability instance

async function main() {
  const accessiblePosts = await Post.accessibleBy(ability);
  console.log(accessiblePosts);
}
```

> See [CASL guide](https://casl.js.org/v5/en/guide/intro) to learn how to define abilities

or on existing query instance:

```js
const Post = require('./Post');
const ability = require('./ability');

async function main() {
  const accessiblePosts = await Post.find({ status: 'draft' })
    .accessibleBy(ability)
    .select('title');
  console.log(accessiblePosts);
}
```

`accessibleBy` returns an instance of `mongoose.Query` and that means you can chain it with any `mongoose.Query`'s method (e.g., `select`, `limit`, `sort`). By default, `accessibleBy` constructs a query based on the list of rules for `read` action but we can change this by providing the 2nd optional argument:

```js
const Post = require('./Post');
const ability = require('./ability');

async function main() {
  const postsThatCanBeUpdated = await Post.accessibleBy(ability, 'update');
  console.log(postsThatCanBeUpdated);
}
```

> `accessibleBy` is built on top of `rulesToQuery` function from `@casl/ability/extra`. Read [Ability to database query](https://casl.js.org/v5/en/advanced/ability-to-database-query) to get insights of how it works.

如果用户没有执行特定操作的权限，CASL 将抛出 `ForbiddenError` 并且不会向 MongoDB 发送请求。为了额外的安全性，它还会添加 `__forbiddenByCasl__: 1` 条件。

例如，让我们查找用户可以删除的所有帖子（我们没有为删除定义权限）：

```js
const { defineAbility } = require('@casl/ability');
const mongoose = require('mongoose');
const Post = require('./Post');

mongoose.set('debug', true);

const ability = defineAbility(can => can('read', 'Post', { private: false }));

async function main() {
  try {
    const posts = await Post.accessibleBy(ability, 'delete');
  } catch (error) {
    console.log(error) // ForbiddenError;
  }
}
```

We can also use the resulting conditions in [aggregation pipeline](https://mongoosejs.com/docs/api.html#aggregate_Aggregate):

```js
const Post = require('./Post');
const ability = require('./ability');

async function main() {
  const query = Post.accessibleBy(ability)
    .where({ status: 'draft' })
    .getQuery();
  const result = await Post.aggregate([
    {
      $match: {
        $and: [
          query,
          // other aggregate conditions
        ]
      }
    },
    // other pipelines here
  ]);
  console.log(result);
}
```

or in [mapReduce](https://mongoosejs.com/docs/api.html#model_Model.mapReduce):

```js
const Post = require('./Post');
const ability = require('./ability');

async function main() {
  const query = Post.accessibleBy(ability)
    .where({ status: 'draft' })
    .getQuery();
  const result = await Post.mapReduce({
    query: {
      $and: [
        query,
        // other conditions
      ]
    },
    map: () => emit(this.title, 1);
    reduce: (_, items) => items.length;
  });
  console.log(result);
}
```

### 可访问字段插件

`accessibleFieldsPlugin` 是一个插件，它为模型的实例和静态方法添加了 `accessibleFieldsBy` 方法，并允许检索所有可访问的字段。当我们需要在响应中仅发送模型的可访问部分时，这非常有用：

```js
const { accessibleFieldsPlugin } = require('@casl/mongoose');
const mongoose = require('mongoose');
const pick = require('lodash/pick');
const ability = require('./ability');
const app = require('./app'); // express app

mongoose.plugin(accessibleFieldsPlugin);

const Post = require('./Post');

app.get('/api/posts/:id', async (req, res) => {
  const post = await Post.accessibleBy(ability).findByPk(req.params.id);
  res.send(pick(post, post.accessibleFieldsBy(ability))
});
```

Model 类上也存在同名方法。但**重要的是**要理解它们之间的区别。静态方法不考虑条件！它遵循与 `Ability` 的 `can` 方法相同的[检查逻辑](https://casl.js.org/v5/en/guide/intro#checking-logic)。让我们看一个例子来回顾：

```js
const { defineAbility } = require('@casl/ability');
const Post = require('./Post');

const ability = defineAbility((can) => {
  can('read', 'Post', ['title'], { private: true });
  can('read', 'Post', ['title', 'description'], { private: false });
});
const post = new Post({ private: true, title: 'Private post' });

Post.accessibleFieldsBy(ability); // ['title', 'description']
post.accessibleFieldsBy(ability); // ['title']
```

As you can see, a static method returns all fields that can be read for all posts. At the same time, an instance method returns fields that can be read from this particular `post` instance. That's why there is no much sense (except you want to reduce traffic between app and database) to pass the result of static method into `mongoose.Query`'s `select` method because eventually you will need to call `accessibleFieldsBy` on every instance.

### `accessibleFieldsBy` 辅助函数

此辅助函数允许获取特定主体类型的可访问字段。当您想要获取主体类型的可访问字段而无需创建该主体类型的实例时，这很有用。

```js
import { accessibleFieldsBy } from '@casl/mongoose';

const fields = accessibleFieldsBy(ability, 'read').ofType('Post');
```

您也可以使用 `of` 方法来获取特定主体实例的可访问字段：

```js
const post = new Post({ title: 'Hello', author: 'me' });
const fields = accessibleFieldsBy(ability, 'read').of(post);
```

`accessibleFieldsBy` 是一个配套辅助函数，允许仅获取特定主体类型或主体的可访问字段：

```ts
import { accessibleFieldsBy } from '@casl/mongoose';
import { Post } from './models';

accessibleFieldsBy(ability).ofType('Post') // returns accessible fields for Post model
accessibleFieldsBy(ability).ofType(Post) // also possible to pass class if classes are used for rule definition
accessibleFieldsBy(ability).of(new Post()) // returns accessible fields for Post model
```

This helper is pre-configured to get all fields from `Model.schema.paths`, if this is not desired or you need to restrict public fields your app work with, you need to define your own custom helper:

```ts
import { AnyMongoAbility, Generics } from "@casl/ability";
import { AccessibleFields, GetSubjectTypeAllFieldsExtractor } from "@casl/ability/extra";
import mongoose from 'mongoose';

const getSubjectTypeAllFieldsExtractor: GetSubjectTypeAllFieldsExtractor = (type) => {
  /** custom implementation of returning all fields */
};

export function accessibleFieldsBy<T extends AnyMongoAbility>(
  ability: T,
  action: Parameters<T['rulesFor']>[0] = 'read'
): AccessibleFields<Extract<Generics<T>['abilities'], unknown[]>[1]> {
  return new AccessibleFields(ability, action, getSubjectTypeAllFieldsExtractor);
}
```

## mongoose 中的 TypeScript 支持

该包是用 TypeScript 编写的，这使得使用插件和 `toMongoQuery` 辅助函数变得更容易，因为 IDE 提供了有用的提示。让我们看看它的实际应用！

假设我们有一个可以描述为以下的 `Post` 实体：

```ts
import mongoose from 'mongoose';

export interface Post extends mongoose.Document {
  title: string
  content: string
  published: boolean
}

const PostSchema = new mongoose.Schema<Post>({
  title: String,
  content: String,
  published: Boolean
});

export const Post = mongoose.model('Post', PostSchema);
```

要使用 `accessibleBy` 方法扩展 `Post` 模型，只需包含相应的插件（全局或在 `Post` 中本地）并使用相应的 `Model` 类型即可。因此，让我们更改示例，使其包含 `accessibleRecordsPlugin`：

```ts
import { accessibleRecordsPlugin, AccessibleRecordModel } from '@casl/mongoose';

// 所有之前的代码，除了最后一行

PostSchema.plugin(accessibleRecordsPlugin);

export const Post = mongoose.model<Post, AccessibleRecordModel<Post>>('Post', PostSchema);

// 现在我们可以安全地使用 `Post.accessibleBy` 方法。
Post.accessibleBy(/* parameters */)
Post.where(/* parameters */).accessibleBy(/* parameters */);
```

以类似的方式，我们可以使用 `AccessibleFieldsModel` 和 `AccessibleFieldsDocument` 类型包含 `accessibleFieldsPlugin`：

```ts
import {
  accessibleFieldsPlugin,
  AccessibleFieldsModel,
  AccessibleFieldsDocument
} from '@casl/mongoose';
import * as mongoose from 'mongoose';

export interface Post extends AccessibleFieldsDocument {
  // 与之前示例相同的 Post 定义
}

const PostSchema = new mongoose.Schema<Post>({
  // 与之前示例相同的 Post 模式定义
})

PostSchema.plugin(accessibleFieldsPlugin);

export const Post = mongoose.model<Post, AccessibleFieldsModel<Post>>('Post', PostSchema);

// 现在我们可以安全地使用 `Post.accessibleFieldsBy` 方法和 `post.accessibleFieldsBy`
Post.accessibleFieldsBy(/* parameters */);
const post = new Post();
post.accessibleFieldsBy(/* parameters */);
```

如果我们想要包含两个插件，我们可以使用提供两个插件方法的 `AccessibleModel` 类型：

```ts
import {
  accessibleFieldsPlugin,
  accessibleRecordsPlugin,
  AccessibleModel,
  AccessibleFieldsDocument
} from '@casl/mongoose';
import * as mongoose from 'mongoose';

export interface Post extends AccessibleFieldsDocument {
  // 与之前示例相同的 Post 定义
}

const PostSchema = new mongoose.Schema<Post>({
  // 与之前示例相同的 Post 模式定义
});
PostSchema.plugin(accessibleFieldsPlugin);
PostSchema.plugin(accessibleRecordsPlugin);

export const Post = mongoose.model<Post, AccessibleModel<Post>>('Post', PostSchema);
```

这允许我们安全地使用 `accessibleBy` 和 `accessibleFieldsBy` 两种方法。

## 想要帮助吗？

想要报告错误、贡献代码或改进文档？太好了！请阅读[贡献指南][contributing]。

如果您想帮助我们维持社区和项目，请考虑[在 Open Collective 上成为财务贡献者](https://opencollective.com/casljs/contribute)

> 详情请参阅[支持 CASL](https://casl.js.org/v5/en/support-casljs)

## 许可证

[MIT 许可证](http://www.opensource.org/licenses/MIT)

[contributing]: https://github.com/stalniy/casl/blob/master/CONTRIBUTING.md
[mongoose]: http://mongoosejs.com/
[mongo-adapter]: https://mongodb.github.io/node-mongodb-native/
[CASL]: https://github.com/stalniy/casl
[MongoDB]: https://www.mongodb.com/
