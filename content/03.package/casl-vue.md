---
title: CASL Vue
categories: [package]
order: 125
meta:
  keywords: ~
  description: ~
---

# CASL Vue

[![@casl/vue NPM version](https://badge.fury.io/js/%40casl%2Fvue.svg)](https://badge.fury.io/js/%40casl%2Fvue)
[![](https://img.shields.io/npm/dm/%40casl%2Fvue.svg)](https://www.npmjs.com/package/%40casl%2Fvue)
[![Support](https://img.shields.io/badge/Support-github%20discussions-green?style=flat&link=https://github.com/stalniy/casl/discussions)](https://github.com/stalniy/casl/discussions)

此包允许将 `@casl/ability` 与 [Vue 3] 应用程序集成。因此，您可以根据用户查看 UI 元素的权限来显示或隐藏它们。

## 安装

**对于 Vue 2.x**：

```sh
npm install @casl/vue@1.x @casl/ability
# 或
yarn add @casl/vue@1.x @casl/ability
# 或
pnpm add @casl/vue@1.x @casl/ability
```

**对于 Vue 3.x**：

```sh
npm install @casl/vue @casl/ability
# 或
yarn add @casl/vue @casl/ability
# 或
pnpm add @casl/vue @casl/ability
```

## 开始使用

此包提供了一个 Vue 插件、几个用于新 [Vue Composition API](https://v3.vuejs.org/guide/composition-api-introduction.html) 的 hooks 和 `Can` 组件。

### 插件

插件提供响应式 `Ability` 实例，并可选择定义 `$ability` 和 `$can` 全局属性，与 Vue 2.x 的方式相同。与之前版本的唯一区别是它需要将 `Ability` 实例作为必需参数传递：

```js
import { createApp } from 'vue';
import { abilitiesPlugin } from '@casl/vue';
import ability from './services/ability';

createApp()
  .use(abilitiesPlugin, ability, {
    useGlobalProperties: true
  })
  .mount('#app');
```

之后，我们可以在任何组件中使用 `$ability` 或 `$can` 方法：

```html
<template>
  <div v-if="$can('create', 'Post')">
    <a @click="createPost">Add Post</a>
  </div>
</template>
```

`globalProperties` 与全局变量的概念相同，这可能会让生活变得复杂一些，因为任何组件都可以访问它们（即隐式依赖），我们需要通过前缀确保它们不会引入名称冲突。因此，我们可以使用 [provide/inject API](https://v3.vuejs.org/guide/component-provide-inject.html) 来注入 `$ability`，而不是将 `$ability` 和 `$can` 暴露为全局变量：

```js
createApp()
  .use(abilitiesPlugin, ability)
  .mount('#app');
```

要在组件中注入 `Ability` 实例，我们可以使用 `ABILITY_TOKEN`：

```html
<template>
  <div>
    <div v-if="$ability.can('create', 'Post')">
      <a @click="createPost">Add Post</a>
    </div>
  </div>
</template>

<script>
import { ABILITY_TOKEN } from '@casl/vue';

export default {
  inject: {
    $ability: { from: ABILITY_TOKEN }
  }
}
</script>
```

这稍微冗长一些，但允许我们明确表达。这与新的 [Composition API](https://v3.vuejs.org/guide/composition-api-introduction.html) 配合得特别好：

```html
<template>
  <div>
    <div v-if="can('create', 'Post')">
      <a @click="createPost">Add Post</a>
    </div>
  </div>
</template>

<script>
import { useAbility } from '@casl/vue';

export default {
  setup() {
    // some code
    const { can } = useAbility();

    return {
      // other props
      can
    };
  }
}
</script>
```

### provideAbility 钩子

很少情况下，我们可能需要为组件子树提供不同的 `Ability` 实例，为此我们可以使用 `provideAbility` hook：

```html
<template>
  <!-- a template -->
</template>

<script>
import { provideAbility } from '@casl/vue';
import { defineAbility } from '@casl/ability';

export default {
  setup() {
    const myCustomAbility = defineAbility((can) => {
      // ...
    });

    provideAbility(myCustomAbility)
  }
}
</script>
```

> 查看 [CASL 指南](https://casl.js.org/v5/en/guide/intro) 了解如何定义 `Ability` 实例。


### Can 组件

我们可以通过使用 `Can` 组件来检查应用程序中的权限，这是另一种方式。`Can` 组件不会被插件自动注册，因此我们可以决定是否要使用组件或 `v-if` + `$can` 方法。此外，这有助于 tree shaking 在我们决定不使用它时将其移除。

要全局注册组件，我们可以使用全局 API（我们也可以在使用它的组件中本地注册组件）：

```js
import { Can, abilitiesPlugin } from '@casl/vue';

createApp()
  .use(abilitiesPlugin, ability)
  .component(Can.name, Can) // 组件注册
  .mount('#app');
```

这是我们如何使用它：

```html
<template>
  <Can I="create" a="Post">
    <a @click="createPost">Add Post</a>
  </Can>
</template>
```

它接受默认插槽和 5 个属性：

* `do` - 操作名称（例如，`read`、`update`）。有一个别名 `I`
* `on` - 检查的主体。有 `a`、`an`、`this` 别名
* `field` - 检查的字段

  ```html
  <template>
    <Can I="read" :this="post" field="title">
      Yes, you can do this! ;)
    </Can>
  </template>
  ```

* `not` - 反转权限检查，如果用户无法执行某些操作则显示 UI：

  ```html
  <template>
    <Can not I="create" a="Post">
      You are not allowed to create a post
    </Can>
  </template>
  ```

* `passThrough` - 无论 `ability.can` 返回什么都渲染子元素。这对于基于 `Can` 创建自定义组件很有用。例如，如果您需要根据用户权限禁用按钮：

  ```html
  <template>
    <div>
      <Can I="delete" a="Post" passThrough v-slot="{ allowed }">
        <button :disabled="!allowed">Delete post</button>
      </Can>
    </div>
  </template>
  ```

#### 属性名称和别名

从上面的代码中可以看出，组件名称及其属性名称和值创建了一个英语句子，实际上是一个问题。上面的示例读作"Can I delete a Post？"。

还有其他几个属性别名允许构造可读的问题。以下是帮助您做到这一点的指导：

* 当您按类型检查时使用 `a`（或 `an`）别名

  ```html
  <Can I="read" a="Post">...</Can>
  ```

* 当您检查特定实例上的操作时使用 `this` 别名。因此，问题可以读作"Can I read this *particular* post？"

  ```html
  <Can I="read" :this="post">...</Can>
  ```

* 如果您感到无聊并且不想让您的代码更具可读性，请使用 `do` 和 `on` :)

  ```html
  <Can do="read" :on="post">...</Can>
  <Can do="read" :on="post" field="title">...</Can>
  ```

#### 组件 vs 响应式 Ability

让我们考虑两种解决方案的优缺点以便做出决定。

**Can 组件**：

**优点**：
* 声明式
* 可以缓存权限检查结果直到 props 或 ability 改变（目前不支持）

**缺点**：
* 创建成本更高
* 在模板中增加嵌套
* 在复杂布尔表达式中更难使用
* 更难将权限检查作为 prop 传递给另一个组件

**响应式 Ability**：

**优点**：
* 易于使用
* 在模板中使用 `v-if` 声明式
* 易于作为 prop 传递给另一个组件
* 易于在复杂布尔表达式中使用（在 js 或模板中）

**缺点**：
* 检查成本更高，条件在每次重新渲染时都会重新评估

尽管响应式权限检查稍微昂贵一些，但它们仍然非常快，建议使用响应式权限而不是 `<Can>` 组件。

## TypeScript 支持

该包是用 TypeScript 编写的，所以不用担心您需要记住所有的属性和别名。如果您使用 TypeScript，您的 IDE 会建议您正确的用法，TypeScript 会在您犯错时警告您。

在 Vue 应用程序中使用 TypeScript 有几种方式，取决于您的偏好。但让我们首先定义我们的 `AppAbility` 类型：


```ts @{data-filename="AppAbility.ts"}
import { Ability, AbilityClass } from '@casl/ability';

type Actions = 'create' | 'read' | 'update' | 'delete';
type Subjects = 'Article' | 'User'

export type AppAbility = Ability<[Actions, Subjects]>;
export const AppAbility = Ability as AbilityClass<AppAbility>;
```

### 扩展 Vue 类型

TypeScript 没有其他方式在不扩展的情况下知道全局属性的类型。为此，让我们添加 `src/shims-ability.d.ts` 文件，内容如下：

```ts @{data-filename="shims-ability.d.ts"}
import { AppAbility } from './AppAbility'

declare module 'vue' {
  interface ComponentCustomProperties {
    $ability: AppAbility;
    $can(this: this, ...args: Parameters<this['$ability']['can']>): boolean;
  }
}
```

### 组合式 API

使用组合式 API，我们不需要扩展 Vue 类型，可以使用 `useAbility` hook：

```ts
import { useAbility } from '@casl/vue';
import { AppAbility } from './AppAbility';

export default {
  setup(props) {
    const { can } = useAbility<AppAbility>();

    return () => can('read', 'Post') ? 'Yes' : 'No';
  }
}
```

此外，我们可以创建一个单独的 `useAppAbility` hook，这样我们就不需要在每个想要检查权限的组件中导入 `useAbility` 和 `AppAbility`，而只需导入一个 hook：


```ts @{data-filename="hooks/useAppAbility.ts"}
import { useAbility } from '@casl/vue';
import { AppAbility } from '../AppAbility';

export const useAppAbility = () => useAbility<AppAbility>();
```

### 选项式 API

也可以在选项式 API 中使用 `@casl/vue` 和 TypeScript。默认情况下，`ABILITY_TOKEN` 的类型是 `InjectionKey<Ability>`，要将其转换为 `InjectionKey<AppAbility>`，我们需要使用一个单独的变量：

```ts @{data-filename="AppAbility.ts"}
import { InjectionKey } from 'vue';
import { ABILITY_TOKEN } from '@casl/vue';

// 定义 `AppAbility` 的前面内容

export const TOKEN = ABILITY_TOKEN as InjectionKey<AppAbility>;
```

现在，当我们注入 `AppAbility` 实例时，我们将拥有正确的类型：

```html
<script lang="ts">
import { defineComponent } from 'vue';
import { TOKEN } from './AppAbility';

export default defineComponent({
  inject: {
    ability: { from: TOKEN }
  },
  created() {
    this.ability // AppAbility
  }
});
</script>
```

> Read [Vue TypeScript](https://v3.vuejs.org/guide/typescript-support.html) for more details.

## 更新 Ability 实例

大多数具有授权逻辑的应用程序需要在用户登录或注销时使用新规则更新 `Ability` 实例。通常您会在 `LoginComponent` 中执行此操作：

```html
<template>
  <form @submit.prevent="login">
    <input v-model="email" type="email" placeholder="Email" />
    <input v-model="password" type="password" placeholder="Password" />
    <button type="submit">Login</button>
  </form>
</template>

<script>
import { useAbility } from '@casl/vue'

export default {
  name: 'LoginForm',
  setup() {
    const ability = useAbility()

    return {
      ability,
      email: '',
      password: ''
    }
  },
  methods: {
    login() {
      const params = {
        email: this.email,
        password: this.password
      }

      return fetch('path/to/api/login', { method: 'POST', body: JSON.stringify(params) })
        .then(response => response.json())
        .then(session => this.ability.update(session.rules))
    }
  }
}
</script>
```

> See [Define rules](https://casl.js.org/v5/en/guide/define-rules) to get more information of how to define `Ability`

## 想要帮助？

想要报告错误、贡献代码或改进文档？太好了！请阅读我们的[贡献指南][contributing]，然后查看标记为 <kbd>help wanted</kbd> 或 <kbd>good first issue</kbd> 的问题。

如果您想帮助我们维持社区和项目，请考虑[在 Open Collective 上成为财务贡献者](https://opencollective.com/casljs/contribute)

> 查看 [Support CASL](https://casl.js.org/v5/en/support-casljs) 了解详情

## 许可证

[MIT License](http://www.opensource.org/licenses/MIT)

[contributing]: https://github.com/stalniy/casl/blob/master/CONTRIBUTING.md
[Vue 3]: https://v3.vuejs.org/guide/introduction.html
