---
title: CASL Angular
categories: [package]
order: 115
meta:
  keywords: ~
  description: ~
---

# CASL Angular

[![@casl/angular NPM version](https://badge.fury.io/js/%40casl%2Fangular.svg)](https://badge.fury.io/js/%40casl%2Fangular)
[![](https://img.shields.io/npm/dm/%40casl%2Fangular.svg)](https://www.npmjs.com/package/%40casl%2Fangular)
[![Support](https://img.shields.io/badge/Support-github%20discussions-green?style=flat&link=https://github.com/stalniy/casl/discussions)](https://github.com/stalniy/casl/discussions)

此包允许将 `@casl/ability` 与 [Angular] 应用程序集成。它提供了已弃用的 `AblePipe`、`AblePurePipe` 以及新的 `AbilityService` 和 `AbilityServiceSignal` 给 Angular 模板，因此您可以根据用户查看权限来显示或隐藏组件、按钮等。

## 安装

```sh
npm install @casl/angular @casl/ability
# or
yarn add @casl/angular @casl/ability
# or
pnpm add @casl/angular @casl/ability
```

## 配置 AppModule

要将管道添加到应用程序的模板中，您需要导入所需的管道

```ts @{data-filename="app.module.ts"}
import { NgModule } from '@angular/core';
import { AblePipe } from '@casl/angular';
import { createMongoAbility, PureAbility } from '@casl/ability';

@NgModule({
  imports: [
    // other modules
    AblePipe
  ],
  providers: [
    { provide: PureAbility, useValue: createMongoAbility() }
  ]
  // other properties
})
export class AppModule {}
```

> 阅读 [CASL 和 TypeScript](https://casl.js.org/v5/en/advanced/typescript) 以获取有关 `MongoAbility` 类型配置的更多详细信息。

## 更新 Ability 实例

大多数需要权限检查支持的应用程序都有类似 `AuthService`、`LoginService` 或 `Session` 服务（您可以随意命名）的东西，它负责用户登录/注销功能。每当用户登录（和注销）时，我们需要使用新规则更新 `Ability` 实例。

让我们假设服务器在登录时返回带有角色的用户：

```ts @{data-filename="Session.ts"}
import { PureAbility, AbilityBuilder } from '@casl/ability';
import { Injectable } from '@angular/core';

@Injectable({ provideIn: 'root' })
export class Session {
  private token: string

  constructor(@Inject(PureAbility) private ability: MongoAbility) {}

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
    const { can, rules } = new AbilityBuilder(createMongoAbility);

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

> 查看 [定义规则](https://casl.js.org/v5/en/guide/define-rules) 以获取有关如何定义 `Ability` 的更多信息

然后在 `LoginComponent` 中使用此 `Session` 服务：

```ts
import { Component } from '@angular/core';
import { Session } from '../services/Session';

@Component({
  selector: 'login-form',
  template: `
    <form (ngSubmit)="login()">
      <input type="email" [(ngModel)]="email" />
      <input type="password" [(ngModel)]="password" />
      <button type="submit">Login</button>
    </form>
  `
})
export class LoginForm {
  email: string;
  password: string;

  constructor(private session: Session) {}

  login() {
    const { email, password } = this;
    return this.session.login({ email, password });
  }
}
```

## 使用 AbilityService 在模板中检查权限

`AbilityService` 是一个提供 `ability$` 可观察对象的服务。此可观察对象注入在 DI 中提供的 `PureAbility` 实例，并在每次规则更改时发出它。这允许高效地使用权限检查，特别是在我们使用 `ChangeDetectionStrategy.OnPush` 的情况下。

让我们首先看看如何在任何组件中使用它：

```ts
@Component({
  selector: 'my-home',
  template: `
    <ng-container *ngIf="ability$ | async as ability">
      <h1>Home Page</h1>
      <button *ngIf="ability.can('create', 'Post')">Create Post</button>
    </ng-container>
  `
})
export class HomeComponent {
  readonly ability$: Observable<AppAbility>;

  constructor(abilityService: AbilityService<AppAbility>) {
    this.ability$ = abilityService.ability$;
  }
}
```

它也可以安全地在 `*ngFor` 和其他指令内使用。如果我们使用 `ChangeDetectionStrategy.OnPush`，它将为我们提供额外的性能改进，因为 `ability.can(...)` 不会在不需要时被调用。

从性能角度来看，这种方法效果很好，因为它每个组件只创建一个订阅（而不是像 `ablePure` 管道那样每次检查都创建），并且不需要我们的组件使用 `Default` 或 `OnPush` 策略。

但让我们也看看如何使用管道进行权限检查以及其性能影响：

## Signals 支持

@casl/angular 的最新版本还支持新的 `signal`。要在您的应用程序中使用它，请使用 `AbilityServiceSignal` 而不是 `AbilityService`：

```ts
import { AbilityServiceSignal } from '@casl/angular';
import { AppAbility } from './AppAbility';

@Component({
  selector: 'my-home',
  template: `
      <h1>Home Page</h1>
      <button *ngIf="can('create', 'Post')">Create Post</button>
  `
})
export class HomeComponent {
  private readonly abilityService = inject<AbilityServiceSignal<AppAbility>>(AbilityServiceSignal);
  protected readonly can = this.abilityService.can;
}
```

## 使用管道在模板中检查权限（已弃用）

要在任何模板中检查权限，您可以使用 `AblePipe`：

```html
<div *ngIf="'create' | able: 'Post'">
  <a (click)="createPost()">Add Post</a>
</div>
```

### 为什么使用管道而不是指令？

指令不能用于将值传递给其他组件的输入。例如，我们需要根据用户创建帖子的能力来启用或禁用按钮。使用指令我们无法做到这一点，但我们可以使用管道来做到这一点：

```html
<button [disabled]="!('create' | able: 'Post')">Add Post</button>
```

### 性能考虑

`@casl/angular` 中有 2 个管道：

* `able` - 非纯管道
* `ablePure` - 纯管道

那么，我们应该何时使用哪个？

> 如果您有疑问，请对操作和主题类型检查使用 `ablePure`，对所有其他情况使用 `able`

根据 Angular 文档，纯管道仅在其参数更改时才被调用。这意味着您**不能将可变对象与纯管道一起使用**，因为这些对象中的更改不会触发纯管道重新评估。但好处是 Angular 为整个应用程序只创建一个纯管道实例并在组件之间重用它，这样可以节省组件实例化时间和内存占用。

由于 [Angular 中的开放功能](https://github.com/angular/angular/issues/15041)，我们需要将 `ablePure` 管道的结果传递给 `async` 管道。因此，不是

```html
<div *ngIf="'create' | ablePure: 'Todo'">...</div>
```

我们需要写：

```html
<div *ngIf="'create' | ablePure: 'Todo' | async">...</div>
```

> `ablePure` 管道返回一个 `Observable<boolean>`，因此 `async` 管道可以有效地解包它

对于改变应用程序状态的应用程序，我们需要使用非纯的 `able` 管道，因为它可以检测对象属性的更改。不用担心，按操作和主题类型进行的检查非常快，在 O(1) 时间内完成。按操作和主题对象进行的检查性能稍慢，取决于特定主题类型的规则数量和使用的条件，但通常这不会成为应用程序的瓶颈。

## TypeScript 支持

此包是用 TypeScript 编写的，因此它会警告您错误的使用。

在 Angular 应用程序中使用特定于应用程序的能力可能有点繁琐，因为在注入 `Ability` 实例的任何地方都需要导入其泛型参数：

```ts
import { Ability } from '@casl/ability';
import { Component } from '@angular/core';
import { AppAbilities } from '../services/AppAbility';

@Component({
  selector: 'todo-item'
})
export class TodoItem {
  constructor(
    private ability: Ability<AppAbilities>
  ) {}
}
```

为了让生活更轻松，您可以使用 `AbilityClass<TAbility>` 类来利用伴随对象模式：

```ts @{data-filename="AppAbility.ts"}
import { Ability, AbilityClass } from '@casl/ability';

type Actions = 'create' | 'read' | 'update' | 'delete';
type Subjects = 'Article' | 'User';

export type AppAbility = MongoAbility<[Actions, Subjects]>;
export const AppAbility = PureAbility as AbilityClass<AppAbility>;
```

并在您的应用程序中随处使用 `AppAbility`：

```ts @{data-filename="AppModule.ts"}
import { NgModule } from '@angular/core';
import { AppAbility } from './services/AppAbility';

@NgModule({
  // other configuration
  providers: [
    { provide: AppAbility, useValue: createMongoAbility() },
  ]
})
export class AppModule {}
```

## 想要帮助？

想要报告错误、贡献代码或改进文档？太好了！请阅读 [贡献指南][contributing]。

如果您想帮助我们维持社区和项目，请考虑 [在 Open Collective 上成为财务贡献者](https://opencollective.com/casljs/contribute)

> 查看 [支持 CASL](https://casl.js.org/v5/en/support-casljs) 了解详情

## 许可证

[MIT 许可证](http://www.opensource.org/licenses/MIT)

[contributing]: https://github.com/stalniy/casl/blob/master/CONTRIBUTING.md
[Angular]: https://angular.io/
