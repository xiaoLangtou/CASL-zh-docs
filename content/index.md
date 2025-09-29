---
seo:
  title: CASL.js 中文文档
  description: 同构授权 JavaScript 库，用于限制给定用户可以访问的资源。
---

::u-page-hero{class="dark:bg-gradient-to-b from-neutral-900 to-neutral-950"}
---
orientation: horizontal
---
#top
:hero-background

#title
强大的 [授权]{.text-primary} 库

#description
CASL 是一个同构授权 JavaScript 库，用于限制给定用户可以访问的资源。它被设计为可增量采用，可以轻松扩展到复杂的授权需求。

#links
  :::u-button
  ---
  to: /guide/install
  size: xl
  trailing-icon: i-lucide-arrow-right
  ---
  开始使用
  :::

  :::u-button
  ---
  icon: i-simple-icons-github
  color: neutral
  variant: outline
  size: xl
  to: https://github.com/stalniy/casl
  target: _blank
  ---
  GitHub 仓库
  :::

#default
  :::prose-pre
  ---
  code: |
    export default defineNuxtConfig({
      modules: [
        '@nuxt/ui',
        '@nuxt/content',
        'nuxt-og-image',
        'nuxt-llms'
      ],

      css: ['~/assets/css/main.css']
    })
  filename: nuxt.config.ts
  ---

  ```ts [nuxt.config.ts]
  export default defineNuxtConfig({
    modules: [
      '@nuxt/ui',
      '@nuxt/content',
      'nuxt-og-image',
      'nuxt-llms'
    ],

    css: ['~/assets/css/main.css']
  })
  ```
  :::
::

::u-page-section{class="dark:bg-neutral-950"}
#title
CASL 核心特性

#links
  :::u-button
  ---
  color: neutral
  size: lg
  target: _blank
  to: /guide/intro
  trailingIcon: i-lucide-arrow-right
  variant: subtle
  ---
  了解更多
  :::

#features
  :::u-page-feature
  ---
  icon: i-lucide-shield-check
  ---
  #title
  同构授权

  #description
  在前端和后端使用相同的授权逻辑。一次定义，随处使用，确保客户端和服务器端的一致性。
  :::

  :::u-page-feature
  ---
  icon: i-lucide-zap
  ---
  #title
  高性能

  #description
  轻量级库，零依赖，优化的算法确保快速的权限检查，不会影响应用程序性能。
  :::

  :::u-page-feature
  ---
  icon: i-lucide-puzzle
  ---
  #title
  灵活可扩展

  #description
  支持简单的基于角色的访问控制（RBAC）到复杂的基于属性的访问控制（ABAC），满足各种授权需求。
  :::

  :::u-page-feature
  ---
  icon: i-lucide-code
  ---
  #title
  TypeScript 支持

  #description
  完整的 TypeScript 类型定义，提供出色的开发体验和编译时类型安全。
  :::

  :::u-page-feature
  ---
  icon: i-lucide-database
  ---
  #title
  数据库集成

  #description
  轻松与 MongoDB、SQL 数据库集成，将权限规则转换为数据库查询，实现高效的数据过滤。
  :::

  :::u-page-feature
  ---
  icon: i-lucide-layers-3
  ---
  #title
  框架无关

  #description
  可与任何 JavaScript 框架配合使用：React、Vue、Angular、Node.js 等，提供一致的授权体验。
  :::
::

::u-page-section{class="dark:bg-neutral-950"}
#title
丰富的使用场景

#links
  :::u-button
  ---
  color: neutral
  size: lg
  target: _blank
  to: /cookbook/roles-with-static-permissions
  trailingIcon: i-lucide-arrow-right
  variant: subtle
  ---
  查看示例
  :::

#features
  :::u-page-feature
  ---
  icon: i-lucide-users
  ---
  #title
  用户管理系统

  #description
  控制用户对个人资料、设置和其他用户数据的访问权限，实现细粒度的用户权限管理。
  :::

  :::u-page-feature
  ---
  icon: i-lucide-building
  ---
  #title
  企业应用

  #description
  实现复杂的组织架构权限控制，支持部门、角色、项目等多维度的访问控制需求。
  :::

  :::u-page-feature
  ---
  icon: i-lucide-file-text
  ---
  #title
  内容管理

  #description
  控制用户对文章、评论、媒体文件等内容的创建、编辑、删除和查看权限。
  :::

  :::u-page-feature
  ---
  icon: i-lucide-shopping-cart
  ---
  #title
  电商平台

  #description
  管理商品、订单、支付等业务对象的访问权限，确保数据安全和业务流程的正确性。
  :::

  :::u-page-feature
  ---
  icon: i-lucide-server
  ---
  #title
  API 安全

  #description
  保护 REST API 和 GraphQL 端点，确保只有授权用户才能访问特定的数据和操作。
  :::

  :::u-page-feature
  ---
  icon: i-lucide-smartphone
  ---
  #title
  移动应用

  #description
  在移动应用中实现离线权限检查，提供一致的用户体验和安全保障。
  :::
::

::u-page-section{class="dark:bg-gradient-to-b from-neutral-950 to-neutral-900"}
  :::u-page-c-t-a
  ---
  links:
    - label: 开始使用
      to: '/guide/install'
      trailingIcon: i-lucide-arrow-right
    - label: 查看 GitHub
      to: 'https://github.com/stalniy/casl'
      target: _blank
      variant: subtle
      icon: i-simple-icons-github
  title: 准备好构建安全的应用了吗？
  description: 加入数千名开发者的行列，使用 CASL 构建强大的授权系统。立即开始，保护您的应用程序。
  class: dark:bg-neutral-950
  ---

  :stars-bg
  :::
::
