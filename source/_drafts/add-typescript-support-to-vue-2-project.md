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

Before you start, upgrade the existing tools to their latest version. `vue-loader` v15 is the [last version][7] that supports Vue 2. Consult the official documents if you encounter migration issues.

```
yarn add vue@^2.7.8
yarn add -D vue-template-compiler@^2.7.8 vue-loader@^15.10.0 webpack@^5.73.0
```

## Install TypeScript and `ts-loader`

First, add `typescript` and `ts-loader` as development dependencies:

```
yarn add -D typescript ts-loader
```

Add `ts-loader` to webpack config:

```js
const config = {
  resolve: {
    extensions: ['.ts', '.js'],
  },
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
      },
      {
        test: /\.ts$/,
        loader: 'ts-loader',
        options: {
          appendTsSuffixTo: [/\.vue$/],
          transpileOnly: true,
        },
      },
    ],
  },
}
```

Now `.ts` files will go through `ts-loader` to get compiled. `vue-loader` will extract `<script lang="ts">` blocks from SFC (Single-File Components) and they also get compiled. The `resolve` and `appendTsSuffixTo` options allow TypeScript to import `.vue` files as modules. `transpileOnly` tells TypeScript compiler *not* to do type checks during compiling. This is for performance reasons, and we will cover it later.

A TypeScript project should have a `tsconfig.json` in the project root. A minimum example would be:

```json
{
  "compilerOptions": {
    "allowJs": true,
    "baseUrl": ".",
    "jsx": "preserve",
    "module": "es2015",
    "moduleResolution": "node",
    "paths": {
      "@/*": ["src/*"]
    },
    "skipLibCheck": true,
    "sourceMap": true,
    "strict": true,
    "target": "es5"
  }
}
```

Options like `baseUrl` and `moduleResolution` tells TypeScript how to find and import a module. `allowJs` allows you to import JavaScript modules in `.ts` files. `skipLibCheck` tells TypeScript to ignore type errors in `node_modules` folder. `strict` turns on extra type checks, such as no implict `any` or `this`.

## Write Vue component with TypeScript

In Vue 2.7, we can use `defineComponent` with Options API to get better type inference. The following example is taken directly from [Vue 3 document][9]. To enable type check in VS Code, install the [Volar][8] extension.

![Vue component in VS Code](/images/typescript/vue-component-in-vs-code.png)

The `count` variable in template is correctly inferred as number type. We can add more type hints to component properties, emits, and event handlers. Please refer to the [document][10] for further details.

Another example would be typing the API request and response data. Take Axios for an instance. This library is currently written in JavaScript, but comes with a [type declaration file][11] that adds type hints to the public API. We can combine it with our custom request/response types.

```ts
interface LoginRequest {
  username: string
  password: string
}

interface LoginResponse {
  userId: number
}

export async function login(data: LoginRequest) {
  return api.post<LoginResponse>('/login', data)
}

// Invoke the API in an async function
const response = await login({ username: 'Jerry', password: '' })
console.log(response.data.userId)
```

If you are using OpenAPI, you can generate typed clients from the specification file. I have written a blog on this topic: *[OpenAPI Workflow with Flask and TypeScript][12]*.

We can also add delaration file to our legacy JavaScript modules. Say there is a `utils.js` module with some function:

```js
export function formatBytes(bytes) {
  if (bytes > 1024) return (bytes / 1024).toFixed(1) + 'K'
  return String(bytes)
}
```

Add the following line to `utils.d.ts` file:

```ts
export function formatBytes(bytes: number): string
```

Now TypeScript will be able to analyze the code:

```ts
import { formatBytes } from '@/utils'

formatBytes('256') // Argument of type 'string' is not assignable to parameter of type 'number'.
```

## Check types during development and build

As mentioned above, the `transpileOnly` option tells `ts-loader` to skip type check so as to speed up the packing process, but obviously drops the benifit of static types. Though IDEs like VS Code + Volar will identify the problems during development, we still need to check types when someone is not using an IDE, or before a pull request is merged. For this purpose, we shall add other two tools:

```
yarn add -D fork-ts-checker-webpack-plugin@^7.2.13 vue-tsc@^0.39.0
```

The [ForkTsCheckerWebpackPlugin][13], as its name suggests, forks a separate process from webpack and do the heavy lifting type check.

```
webpack 5.73.0 compiled successfully in 4177 ms
Type-checking in progress...
ERROR in ./src/services/user.ts:28:13
TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.
    26 | }
    27 |
  > 28 | formatBytes('256')
       |             ^^^^^
    29 |

Found 1 error in 11671 ms.
```

After `yarn start`, local dev server will be available in 4s, and type check takes 11s to finish. The error message will also be displayed on the web page.

![ForkTsCheckerWebpackPlugin](/images/typescript/fork-ts-checker-webpack-plugin.png)

Add this plugin to webpack config, and turn on its support for Vue SFC.

```js
const { VueLoaderPlugin } = require('vue-loader')
const ForkTsCheckerWebpackPlugin = require('fork-ts-checker-webpack-plugin')

const config = {
  plugins: [
    new VueLoaderPlugin(),
    new ForkTsCheckerWebpackPlugin({
      typescript: {
        extensions: {
          vue: true,
        },
      },
    }),
  ],
}
```

However, this plugin only solves the problem during development, we still need a way to do type check before someone merges his code. The solution is to put [`vue-tsc`][14] in the lint phase of your project. `tsc` is the TypeScript Compiler, and `vue-tsc` is a wrapper of that to support compiling TS code block in SFC. Modify the `lint` script in your `package.json` and setup a proper CI pipeline.

```json
{
  scripts: {
    "lint": "eslint --ext .vue,.ts,.js . && vue-tsc --noEmit"
  }
}
```

## More on code style and linting

We usually use `eslint` to enforce various rules of coding convention, and `prettier` for auto formatting. TypeScript also has dedicated lint rules and style guide. Install the necessary eslint plugins:

```
yarn add -D @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

To make it work with [`esling-plugin-vue`][15], use the following `.eslintrc.js` config:

```js
module.exports = {
  extends: [
    'standard',
    'plugin:vue/essential',
    'plugin:@typescript-eslint/recommended',
    'prettier',
  ],
  parser: 'vue-eslint-parser',
  parserOptions: {
    parser: '@typescript-eslint/parser',
  },
}
```

Prettier also has built-in support for TypeScript. The `prettier` plugin in `extends` helps disabling some of the formatting rules. Here is an example of `.prettierrc.json`:

```json
{
  "htmlWhitespaceSensitivity": "ignore",
  "semi": false,
  "singleQuote": true
}
```

And do not forget to add [`husky`][16] and [`lint-staged`][17] to your toolchain, that helps auto linting and formatting your code before it is committed.


[1]: https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes.html
[2]: https://www.typescriptlang.org/cheatsheets
[3]: https://blog.vuejs.org/posts/vue-2-7-naruto.html
[4]: https://v2.vuejs.org/v2/guide/typescript.html
[5]: https://vue-loader.vuejs.org/guide/pre-processors.html#typescript
[6]: https://webpack.js.org/guides/typescript/
[7]: https://github.com/vuejs/vue-loader/issues/1919
[8]: https://marketplace.visualstudio.com/items?itemName=Vue.volar
[9]: https://vuejs.org/guide/typescript/overview.html#definecomponent
[10]: https://vuejs.org/guide/typescript/options-api.html
[11]: https://github.com/axios/axios/blob/v0.27.2/index.d.ts
[12]: https://shzhangji.com/blog/2022/06/19/openapi-workflow-with-flask-and-typescript/
[13]: https://github.com/TypeStrong/fork-ts-checker-webpack-plugin
[14]: https://github.com/johnsoncodehk/volar/tree/master/packages/vue-tsc
[15]: https://eslint.vuejs.org/user-guide/#how-to-use-a-custom-parser
[16]: https://typicode.github.io/husky/#/?id=install
[17]: https://github.com/okonet/lint-staged
