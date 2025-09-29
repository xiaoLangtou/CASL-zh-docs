---
title: CASL React
categories: [package]
order: 120
meta:
  keywords: ~
  description: ~
---

# CASL React

[![@casl/react NPM version](https://badge.fury.io/js/%40casl%2Freact.svg)](https://badge.fury.io/js/%40casl%2Freact)
[![](https://img.shields.io/npm/dm/%40casl%2Freact.svg)](https://www.npmjs.com/package/%40casl%2Freact)
[![Support](https://img.shields.io/badge/Support-github%20discussions-green?style=flat&link=https://github.com/stalniy/casl/discussions)](https://github.com/stalniy/casl/discussions)

此包允许将 `@casl/ability` 与 [React] 应用程序集成。它提供了 `Can` 组件，允许根据用户查看权限来隐藏或显示 UI 元素。

> `@casl/react` 与 [React Native](https://reactnative.dev/) 完美配合

## 安装

```sh
npm install @casl/react @casl/ability
# or
yarn add @casl/react @casl/ability
# or
pnpm add @casl/react @casl/ability
```

## Can 组件

它接受子元素和 6 个属性：

* `do` - 操作名称（例如，`read`、`update`）。有一个别名 `I`
* `on` - 检查的主题。有 `a`、`an`、`this` 别名
* `field` - 检查的字段

  ```jsx
  export default ({ post }) => <Can I="read" this={post} field="title">
    Yes, you can do this! ;)
  </Can>
  ```

* `not` - 反转权限检查，如果用户无法执行某些操作则显示 UI：

  ```jsx
  export default () => <Can not I="create" a="Post">
    You are not allowed to create a post
  </Can>
  ```

* `passThrough` - 无论 `ability.can` 返回什么都渲染子元素。这对于基于 `Can` 创建自定义组件很有用。例如，如果您需要根据用户权限禁用按钮：

  ```jsx
  export default () => (
    <Can I="create" a="Post" passThrough>
      {allowed => <button disabled={!allowed}>Save</button>}
    </Can>
  )
  ```

* `ability` - 用于检查权限的 `Ability` 实例
* `children` - 要隐藏或渲染的元素。可以是渲染函数：

  ```jsx
  export default () => <Can I="create" a="Post" ability={ability}>
    {() => <button onClick={this.createPost}>Create Post</button>}
  </Can>
  ```

  或 React 元素：

  ```jsx
  export default () => <Can I="create" a="Post" ability={ability}>
    <button onClick={this.createPost}>Create Post</button>
  </Can>
  ```

> 最好将子元素作为渲染函数传递，因为如果用户没有执行某些操作的权限（在上面的情况下是 `create Post`），它不会创建额外的 React 元素

不要被组件接受的属性数量吓到，我们将讨论如何绑定其中一些属性。

### 将 Can 绑定到特定的 Ability 实例

在每个 `Can` 组件中传递 `ability` 会很不方便。这就是为什么有 2 个函数允许绑定 `Can` 来使用特定的 `Ability` 实例：

* `createCanBoundTo`\
  此函数是为了支持 React < 16.4.0 版本而创建的，这些版本没有 [Context API][react-ctx-api]。可以这样使用：

  ```js @{data-filename="Can.js"}
  import { createCanBoundTo } from '@casl/react';
  import ability from './ability';

  export const Can = createCanBoundTo(ability);
  ```
* `createContextualCan`\
  此函数是为了支持 [React 的 Context API][react-ctx-api] 而创建的，可以这样使用：

  ```js @{data-filename="Can.js"}
  import { createContext } from 'react';
  import { createContextualCan } from '@casl/react';

  export const AbilityContext = createContext();
  export const Can = createContextualCan(AbilityContext.Consumer);
  ```

这两种方法几乎相同，第二种稍微好一些，因为它允许您为应用程序的不同部分提供不同的 `Ability` 实例，并使用 [`contextType` 静态属性](https://reactjs.org/docs/context.html#classcontexttype) 注入权限。根据您使用的 React 版本选择您的方式。

> 在本指南中，我们将使用 `createContextualCan`，因为它涵盖了现代 React 开发中的更多情况。

为了完成设置，我们需要通过 `AbilityContext.Provider` 提供一个 `Ability` 实例：

```jsx @{data-filename="App.jsx"}
import { AbilityContext } from './Can'
import ability from './ability'

export default function App({ props }) {
  return (
    <AbilityContext.Provider value={ability}>
      <TodoApp />
    </AbilityContext.Provider>
  )
}
```

> 查看 [CASL 指南](https://casl.js.org/v6/en/guide/intro) 了解如何定义 `Ability` 实例。

然后使用我们的 `Can` 组件：

```jsx
import React, { Component } from 'react'
import { Can } from './Can'

export class TodoApp extends Component {
  createTodo = () => {
    // implement logic to show new todo form
  };

  render() {
    return (
      <Can I="create" a="Todo">
        <button onClick={this.createTodo}>Create Todo</button>
      </Can>
    )
  }
}
```

### 命令式访问 Ability 实例

有时组件中的逻辑可能有点复杂，因此您无法使用 `<Can>` 组件。在这种情况下，您可以使用 [React 的 `contextType` 组件属性](https://reactjs.org/docs/context.html#classcontexttype)：

```jsx
import React, { Component } from 'react'
import { AbilityContext } from './Can'

export class TodoApp extends Component {
  createTodo = () => {
    // logic to show new todo form
  };

  render() {
    return (
      <div>
        {this.context.can('create', 'Todo') &&
          <button onClick={this.createTodo}>Create Todo</button>}
      </div>
    );
  }
}

TodoApp.contextType = AbilityContext;
```

或 `useContext` hook：

```jsx
import React, { useContext } from 'react';
import { AbilityContext } from './Can'

export default () => {
  const createTodo = () => { /* logic to show new todo form */ };
  const ability = useContext(AbilityContext);

  return (
    <div>
      {ability.can('create', 'Todo') &&
        <button onClick={createTodo}>Create Todo</button>}
    </div>
  );
}
```

在这种情况下，当您想要更新用户权限时，您需要创建一个新的 `Ability` 实例（不要使用 `update` 方法，在这种情况下它不会触发重新渲染），或者您需要强制重新渲染整个应用程序。

为了让事情变得更容易，`@casl/react` 提供了 `useAbility` hook，它接受 `React.Context` 作为唯一参数（与 `useContext` 相同），但当您更新 `Ability` 规则时，会在使用此 hook 的组件中触发重新渲染。上面的示例可以重写为：

```jsx
import { useAbility } from '@casl/react';
import { AbilityContext } from './Can'

export default () => {
  const createTodo = () => { /* logic to show new todo form */ };
  const ability = useAbility(AbilityContext);

  return (
    <div>
      {ability.can('create', 'Todo') &&
        <button onClick={createTodo}>Create Todo</button>}
    </div>
  );
}
```

### 在 React < 16.4 中使用 TypeScript 的注意事项

If you use TypeScript and React < 16.4 make sure to create a file `contextAPIPatch.d.ts` file with the next content:

```ts
declare module 'react' {
  export type Consumer<T> = any;
}
```

and include it in your `tscofig.json`, otherwise your app won't compile:

```json
{
  // other configuration options
  "include": [
    "src/**/*",
    "./contextAPIPatch.d.ts" // <-- add this line
  ]
}
```

### 属性名称和别名

从上面的代码中可以看出，组件名称及其属性名称和值创建了一个英语句子，实际上是一个问题。例如，下面的代码读作 `Can I create a Post`：

```jsx
export default () => <Can I="create" a="Post">
  <button onClick={...}>Create Post</button>
</Can>
```

还有其他几个属性别名允许构造可读的问题：

* 当您按类型检查时使用 `a`（或 `an`）别名

  ```jsx
  export default () => <Can I="read" a="Post">...</Can>
  ```

* 当您检查特定实例上的操作时，使用 `this` 别名而不是 `a`。因此，问题可以读作"Can I read this *particular* post？"

  ```jsx
  // `this.props.post` 是 `Post` 类的实例（即模型实例）
  export default () => <Can I="read" this={this.props.post}>...</Can>
  ```

* 如果您感到无聊并且不想让您的代码更具可读性，请使用 `do` 和 `on` ;)

  ```jsx
  // `this.props.post` 是 `Post` 类的实例（即模型实例）
  export default () => <Can do="read" on={this.props.post}>...</Can>

  // 或按字段检查
  export default () => <Can do="read" on={this.props.post} field="title">...</Can>
  ```

## TypeScript 支持

该包是用 TypeScript 编写的，所以不用担心您需要记住所有的属性和别名。如果您使用 TypeScript，您的 IDE 会建议您正确的用法，TypeScript 会在您犯错时警告您。

## 更新 Ability 实例

大多数需要权限检查支持的应用程序都有类似 `AuthService` 或 `LoginService` 或 `Session` 服务（随您喜欢命名）的东西，负责用户登录/注销功能。每当用户登录（和注销）时，我们需要使用新规则更新 `Ability` 实例。通常您会在 `LoginComponent` 中执行此操作。

让我们想象服务器在登录时返回带有角色的用户：

```ts @{data-filename="Login.jsx"}
import { AbilityBuilder, Ability } from '@casl/ability';
import React, { useState, useContext } from 'react';
import { AbilityContext } from './Can';

function updateAbility(ability, user) {
  const { can, rules } = new AbilityBuilder(Ability);

  if (user.role === 'admin') {
    can('manage', 'all');
  } else {
    can('read', 'all');
  }

  ability.update(rules);
}

export default () => {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const ability = useContext(AbilityContext);
  const login = () => {
    const params = {
      method: 'POST',
      body: JSON.stringify({ username, password })
    };
    return fetch('path/to/api/login', params)
      .then(response => response.json())
      .then(({ user }) => updateAbility(ability, user));
  };

  return (
    <form>
      {/* input fields */}
      <button onClick={login}>Login</button>
    </form>
  );
};
```

> See [Define rules](https://casl.js.org/v6/en/guide/define-rules) to get more information of how to define `Ability`

## 在 hooks 中使用 `useAbility`

在 hook 依赖项中使用 `const ability = useAbility(AbilityContext)` 的返回值 `ability` 不会在规则更新时触发重新渲染。您必须指定 `ability.rules`：

```jsx
const posts = React.useMemo(() => getPosts(ability), [ability.rules]);
// ✅ 调用 ability.update 将更新帖子列表
```

**注意**：如果您在 hooks 内部使用 `useAbility` hook，请确保调用 `ability.update(rules)` 而不是创建新的 `Ability` 实例。否则，`useAbility` hook 将无法触发重新渲染。

## 想要帮助？

想要报告错误、贡献代码或改进文档？太好了！阅读我们的[贡献指南][contributing]，然后查看标记为 <kbd>help wanted</kbd> 或 <kbd>good first issue</kbd> 的问题。

如果您想帮助我们维持社区和项目，请考虑[在 Open Collective 上成为财务贡献者](https://opencollective.com/casljs/contribute)

> 详情请参阅 [支持 CASL](https://casl.js.org/v6/en/support-casljs)

## 许可证

[MIT 许可证](http://www.opensource.org/licenses/MIT)

[contributing]: https://github.com/stalniy/casl/blob/master/CONTRIBUTING.md
[React]: https://reactjs.org/
[react-ctx-api]: https://reactjs.org/docs/context.html
