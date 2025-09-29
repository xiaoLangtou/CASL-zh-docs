---
title: CASL Aurelia
categories: [package]
order: 130
meta:
  keywords: ~
  description: ~
---

# CASL Aurelia

[![@casl/aurelia NPM version](https://badge.fury.io/js/%40casl%2Faurelia.svg)](https://badge.fury.io/js/%40casl%2Faurelia)
[![](https://img.shields.io/npm/dm/%40casl%2Faurelia.svg)](https://www.npmjs.com/package/%40casl%2Faurelia)
[![Support](https://img.shields.io/badge/Support-github%20discussions-green?style=flat&link=https://github.com/stalniy/casl/discussions)](https://github.com/stalniy/casl/discussions)

该包允许将 `@casl/ability` 与 [Aurelia] 应用程序集成。它为 Aurelia 模板提供了 `AbleValueConverter` 和**已弃用的** `CanValueConverter`，因此您可以根据用户查看它们的能力来显示或隐藏组件、按钮等。

## 安装

```sh
npm install @casl/aurelia @casl/ability
# or
yarn add @casl/aurelia @casl/ability
# or
pnpm add @casl/aurelia @casl/ability
```

## 开始使用

`@casl/aurelia` 导出 `configure` 函数，该函数满足 [Aurelia 插件](https://aurelia.io/docs/plugins/write-new-plugin) 的要求。因此，您可以在 `plugin` 函数中传递它：

```js @{data-filename="main.js"}
import ability from './services/ability';

export function configure(aurelia) {
  aurelia.use
    .standardConfiguration()
    .developmentLogging()
    .plugin('@casl/aurelia', ability); // <-- add plugin

  aurelia.start().then(() => aurelia.setRoot());
}
```

该插件接受一个可选的第二个参数，即您应用程序的 ability 实例。您也可以通过直接调用 `container` 的 API 来注册实例，但请确保注册 `PureAbility` 键，值转换器通过此键请求 `Ability` 实例。这允许应用程序开发人员决定如何配置操作、主题和条件。这也是从 tree shaking 中获得最大收益的唯一方法（例如，如果您不需要条件，您可以使用 `PureAbility` 并摆脱 `sift` 库）。

```js
import { Ability, PureAbility } from '@casl/ability';
import ability from './services/ability';

export function configure(aurelia) {
  aurelia.use
    .standardConfiguration()
    .developmentLogging()
    .plugin('@casl/aurelia'); // <-- add plugin

  aurelia.container.registerInstance(PureAbility, ability);
  aurelia.container.registerInstance(Ability, ability);
  aurelia.start().then(() => aurelia.setRoot());
}
```

> Read [CASL and TypeScript](https://casl.js.org/v5/en/advanced/typescript) to get more details about `Ability` type configuration.

## 更新 Ability 实例

大多数需要权限检查支持的应用程序都有类似 `AuthService` 或 `LoginService` 或 `Session` 服务（随您命名）的东西，负责用户登录/注销功能。每当用户登录（和注销）时，我们需要使用新规则更新 `Ability` 实例。

让我们想象服务器在登录时返回具有角色的用户：

```ts @{data-filename="Session.ts"}
import { autoinject } from 'aurelia-framework';
import { Ability, AbilityBuilder } from '@casl/ability';

@autoinject
export class Session {
  private token: string

  constructor(private ability: Ability) {}

  login(details) {
    const params = { method: 'POST', body: JSON.stringify(details) };
    return fetch('path/to/api/login', params)
      .then(response => response.json())
      .then((session) => {
        this.updateAbility(session.user);
        this.token = session.token;
      });
  }

  private updateAbility(user) {
    const { can, rules } = new AbilityBuilder(Ability);

    if (user.role === 'admin') {
      can('manage', 'all');
    } else {
      can('read', 'all');
    }

    this.ability.update(rules);
  }

  logout() {
    this.token = null;
    this.ability.update([]);
  }
}
```

> See [Define rules](https://casl.js.org/v5/en/guide/define-rules) to get more information of how to define `Ability`

然后在 `LoginComponent` 中使用这个 `Session` 服务：

```ts
import { autoinject, bindable } from 'aurelia-framework';
import { Session } from '../services/Session';

@autoinject
export class LoginFormCustomElement {
  @bindable email: string;
  @bindable password: string;

  constructor(private session: Session) {}

  login() {
    const { email, password } = this;
    return this.session.login({ email, password });
  }
}
```

## 在模板中检查权限

要在任何模板中检查权限，您可以使用 `AbleValueConverter`：

```html
<div if.bind="'create' | able: 'Post'">
  <a click.trigger="createPost()">Add Post</a>
</div>
```

> 您可以将 `if` 中的表达式读作"if creatable Post"

或者使用**已弃用的** `CanPipe`：

```html
<div *ngIf="'Post' | can: 'create'">
  <a click.trigger="createPost()">Add Post</a>
</div>
```

`CanValueConverter` 被弃用是因为它的可读性较差，并且更难与 `Ability` 的 `can` 方法支持的所有类型定义集成。这就是为什么 `CanValueConverter` 的类型比 `AbleValueConverter` 更弱的原因。

## 为什么使用值转换器而不是自定义属性？

自定义属性不能用于将值传递到其他组件的输入中。例如，我们需要根据用户创建帖子的能力来启用或禁用按钮。使用指令我们无法做到这一点，但我们可以使用值转换器来做到这一点：

```html
<button disabled.bind="!('create' | able: 'Post')">Add Post</button>
```

Aurelia 中的值转换器在性能方面非常好，如果它们的参数没有改变，它们就不会被调用。它们还支持信号绑定，因此当您更新 `Ability` 实例时可以轻松更新。

## TypeScript 支持

该包是用 TypeScript 编写的，因此它会警告您错误的用法。

在 Aurelia 应用程序中使用特定于应用程序的权限可能有点繁琐，因为在注入 `Ability` 实例的任何地方，您都需要导入其泛型参数：

```ts
import { Ability } from '@casl/ability';
import { autoinject } from 'aurelia-framework';
import { AppAbilities } from '../services/AppAbility';

@autoinject
export class TodoItemCustomElement {
  constructor(private ability: Ability<AppAbilities>) {}
}
```

为了让生活更轻松，您可以创建一个单独的类而不是创建单独的类型：

```ts @{data-filename="AppAbility.ts"}
import { Ability } from '@casl/ability';

type Actions = 'create' | 'read' | 'update' | 'delete';
type Subjects = 'Article' | 'User'

export type AppAbilities = [Actions, Subjects];

export class AppAbility extends Ability<AppAbilities> {
}
```

并在 Aurelia 的容器中注册这个类：

```ts @{data-filename="main.ts"}
import { AppAbility } from './services/AppAbility';

export function configure(aurelia) {
  aurelia.use
    .standardConfiguration()
    .developmentLogging()
    .plugin('@casl/aurelia'); // <-- add plugin

  const ability = new AppAbility();
  aurelia.container.registerInstance(PureAbility, ability);
  aurelia.container.registerInstance(AppAbility, ability);
  aurelia.start().then(() => aurelia.setRoot());
}
```

## 想要帮助？

想要报告错误、贡献代码或改进文档？太好了！请阅读[贡献指南][contributing]。

如果您想帮助我们维持社区和项目，请考虑[在 Open Collective 上成为财务贡献者](https://opencollective.com/casljs/contribute)

> 查看 [Support CASL](https://casl.js.org/v5/en/support-casljs) 了解详情

## 许可证

[MIT License](http://www.opensource.org/licenses/MIT)

[contributing]: https://github.com/stalniy/casl/blob/master/CONTRIBUTING.md
[Aurelia]: https://aurelia.io/

