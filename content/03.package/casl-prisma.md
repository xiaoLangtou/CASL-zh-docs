---
title: CASL Prisma
categories: [package]
order: 105
meta:
  keywords: prisma authorization, permission management, casl
  description: >
    使用 CASL 权限管理库进行 Prisma 授权。
    在运行时测试权限并使用 Prisma Where 条件获取可访问的记录
---

# CASL Prisma

[![@casl/prisma NPM version](https://badge.fury.io/js/%40casl%2Fprisma.svg)](https://badge.fury.io/js/%40casl%2Fprisma)
[![](https://img.shields.io/npm/dm/%40casl%2Fprisma.svg)](https://www.npmjs.com/package/%40casl%2Fprisma)
[![Support](https://img.shields.io/badge/Support-github%20discussions-green?style=flat&link=https://github.com/stalniy/casl/discussions)](https://github.com/stalniy/casl/discussions)

该包允许使用 Prisma `WhereInput` 在 [Prisma] 模型上定义 [CASL] 权限。这在 SQL 世界的权限管理方面带来了很大的力量：

1. 我们可以使用 Prisma 查询来定义权限，不再需要学习 MongoDB 查询语言。
2. 此外，我们可以向 SQL 数据库询问诸如"哪些记录可以读取？"或"哪些记录可以更新？"等问题。

## 安装

```sh
npm install @casl/prisma @casl/ability
# or
yarn add @casl/prisma @casl/ability
# or
pnpm add @casl/prisma @casl/ability
```

## 使用方法

该包与其他包有所不同，因为它提供了一个自定义的 `createPrismaAbility` 工厂函数，该函数配置为使用 Prisma [WhereInput](https://www.prisma.io/docs/reference/api-reference/prisma-client-reference#where) 检查权限：

```ts
import { User, Post, Prisma } from '@prisma/client';
import { PureAbility, AbilityBuilder, subject } from '@casl/ability';
import { createPrismaAbility, PrismaQuery, Subjects } from '@casl/prisma';

type AppAbility = PureAbility<[string, Subjects<{
  User: User,
  Post: Post
}>], PrismaQuery>;
const { can, cannot, build } = new AbilityBuilder<AppAbility>(createPrismaAbility);

can('read', 'Post', { authorId: 1 });
cannot('read', 'Post', { title: { startsWith: '[WIP]:' } });

const ability = build();
ability.can('read', 'Post');
ability.can('read', subject('Post', { title: '...', authorId: 1 })));
```

> 请参阅 [CASL 指南](https://casl.js.org/v5/en/guide/intro) 了解如何定义权限。除了条件语言外，其他一切都是相同的。

### 关于 subject 辅助函数的注意事项

因为 Prisma 返回的 DTO 对象没有暴露任何类型信息，我们需要使用 `subject` 辅助函数手动提供该类型，以便 CASL 可以理解对传入对象应用哪些规则。

不幸的是，除了向所有模型添加额外列之外，没有简单的方法来自动化这一点。有关更多详细信息，请查看[此问题](https://github.com/prisma/prisma/issues/5315)。

> 要获取有关对象类型检测的更多详细信息，请阅读 [CASL 主体类型检测](https://casl.js.org/v5/en/guide/subject-type-detection  )

### 关于 Prisma 查询运行时解释器的注意事项

`@casl/prisma` 使用 [ucast](https://github.com/stalniy/ucast) 在 JavaScript 运行时解释 Prisma [WhereInput](https://www.prisma.io/docs/reference/api-reference/prisma-client-reference#where)。但是，有一些注意事项：
- 未实现 JSON 列的相等性
- 未实现数组/列表列的相等性（但是像 `has`、`hasSome` 和 `hasEvery` 这样的操作符应该足够了）
- 在关系上定义条件时，始终指定其中一个操作符（`every`、`none`、`some`、`is` 或 `isNot`）

在接收到查询操作符的无效参数或不支持某些操作的情况下，解释器会抛出 `ParsingQueryError`。

## 查找可访问记录

[Prisma] 和 [CASL] 集成的一个很好的功能是我们可以从数据库中获取用户有权访问的所有记录。要做到这一点，只需使用 `accessibleBy` 辅助函数：

```ts
// ability 是在上面示例中创建的 PrismaAbility 实例

const accessiblePosts = await prisma.post.findMany({
  where: accessibleBy(ability).Post
});
```

该函数接受 `Ability` 实例和 `action`（默认为 `read`），返回一个对象，其中键对应于 Prisma 模型名称，值是从权限规则 `WhereInput` 对象聚合而来的。

**重要提示**：如果用户没有访问任何帖子的权限，`accessibleBy` 会抛出 `ForbiddenError`，所以要准备好捕获它！

要将此与业务逻辑条件结合，只需使用 `AND`：

```ts
const accessiblePosts = await prisma.post.findMany({
  where: {
    AND: [
      accessibleBy(ability).Post,
      { /* 业务相关条件 */ }
    ]
  }
})
```

## TypeScript 支持

该包是用 TypeScript 编写的，提供了全面的 IDE 提示和编译时验证。

> 确保调用 `prisma generate`。`@casl/prisma` 使用 Prisma 生成的类型，因此如果客户端未生成，则什么都不会工作。

此外，还有几个辅助函数使得与 Prisma 和 CASL 一起工作变得容易：

### PrismaQuery

这是一个泛型类型，以通用方式提供 `Prisma.ModelWhereInput`。我们需要传入一个命名模型：

```ts
import { User } from '@prisma/client';
import { Model, PrismaQuery } from '@casl/prisma';

// 几乎与 Prisma.UserWhereInput 相同，除了它是一个高阶类型
type UserWhereInput = PrismaQuery<Model<User, 'User'>>;
```

### Model

只是给模型一个名称。该名称使用 `@casl/ability` 中的 `ForcedSubject<TName>` 辅助函数存储。要使用单独的列或另一种策略来命名模型，请不要使用此辅助函数，因为它只能与 `subject` 辅助函数结合使用。

### Subjects

从传入的对象中创建所有可能主体的联合：

```ts
import { User } from '@prisma/client';
import { Subjects } from '@casl/prisma';

type AppSubjects = Subjects<{
  User: User
}>; // 'User' | Model<User, 'User'>
```

要支持 `all` 的规则定义，我们只需要显式地做：

```ts
type AppSubjects = 'all' | Subjects<{
  User: User
}>; // 'User' | Model<User, 'User'>

type AppAbility = PureAbility<[string, AppSubjects], PrismaQuery>;
```

## 自定义 PrismaClient 输出路径

Prisma 允许[将客户端生成到自定义目录](https://www.prisma.io/docs/concepts/components/prisma-client/working-with-prismaclient/generating-prisma-client#using-a-custom-output-path)，在这种情况下，`@prisma/client` 不再重新导出所需的类型，`@casl/prisma` 无法自动检测和推断类型。在这种情况下，我们需要手动提供所需的类型。假设我们有以下配置：

```prisma
generator client {
  provider = "prisma-client-js"
  output   = "../src/generated/client"
}
```

然后我们需要为 casl-prisma 集成创建一个自定义文件（查看[源代码](https://github.com/stalniy/casl/blob/master/packages/casl-prisma/src/index.ts)获取最新示例）：

```ts
// src/casl-prisma.ts
import {
  type PrismaModel,
  type PrismaQueryFactory,
  type PrismaTypes,
  createAbilityFactory,
} from "@casl/prisma/runtime";
import type { hkt } from "@casl/ability";
import { Prisma } from "./generated/client";

export { accessibleBy, ParsingQueryError } from '@casl/prisma/runtime';
export type { Model, Subjects } from '@casl/prisma/runtime';
export type WhereInput<TModelName extends Prisma.ModelName> = PrismaTypes<Prisma.TypeMap>['WhereInput'][TModelName];
export type PrismaQuery<T extends PrismaModel = PrismaModel> = PrismaQueryFactory<Prisma.TypeMap, T>;
export const createPrismaAbility = createAbilityFactory<Prisma.ModelName, PrismaQuery>();
```

## 想要帮助吗？

想要报告错误、贡献代码或改进文档？太好了！请阅读[贡献指南][contributing]。

如果您想帮助我们维持社区和项目，请考虑[在 Open Collective 上成为财务贡献者](https://opencollective.com/casljs/contribute)

> 详情请参阅[支持 CASL](https://casl.js.org/v5/en/support-casljs)

## 许可证

[MIT 许可证](http://www.opensource.org/licenses/MIT)

[contributing]: https://github.com/stalniy/casl/blob/master/CONTRIBUTING.md
[Prisma]: https://prisma.io/
[CASL]: https://github.com/stalniy/casl
