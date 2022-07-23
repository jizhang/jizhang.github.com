---
title: Add TypeScript Support to Vue 2 Project
tags: [frontend, typescript, vue, webpack, eslint]
categories: Programming
---

Now that TypeScript has become the de facto standard in frontend development, new projects and third-party libraries are mostly built on its ecosystem. For existing projects, TypeScript can also be applied gradually. Just add the toolchain, and start writing or rewriting part of your application. In this article, I will walk you through the steps of adding TypeScript to a Vue 2 project, since I myself is working on a legacy project, and TypeScript has brought a lot of benefits.

## Prerequisites

For those who are new to TypeScript, I recommend you read the guide *[TypeScript for JavaScript Programmers][1]*. In short, TypeScript is a superset of JavaScript. It adds type hints to variables, as well as other syntax like class, interface, decorator, and some of them are already merged into ECMAScript. When compiling, TypeScript can do static type check. It will try to infer the variable type as much as possible, or you need to define the type explicitly. Here is the official [TypeScript Cheat Sheet][2].

![TypeScript Cheat Sheet - Interface](/images/typescript/cheat-sheet-interface.png)

<!-- more -->

You should also be familiar with [Vue][4], [vue-loader][5], and [webpack][6]. Vue 2 already has good support for TypeScript, and the recently published [Vue 2.7][3] backported a lot of useful features from Vue 3, like composition API, `<script setup>`, and `defineComponent`, further improving the developer experience of TypeScript in Vue.

## Install TypeScript and `ts-loader`



[1]: https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes.html
[2]: https://www.typescriptlang.org/cheatsheets
[3]: https://blog.vuejs.org/posts/vue-2-7-naruto.html
[4]: https://v2.vuejs.org/v2/guide/typescript.html
[5]: https://vue-loader.vuejs.org/guide/pre-processors.html#typescript
[6]: https://webpack.js.org/guides/typescript/
