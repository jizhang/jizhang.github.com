---
title: Use Bootstrap V5 in Vue 3 Project
categories: Programming
tags: [bootstrap, vue, vite]
---

Bootstrap V5 and Vue 3.x have been released for a while, but the widely used BootstrapVue library is still based on Bootstrap V4 and Vue 2.x. A [new version][1] of BootstrapVue is under development, and there is an alternative project [BootstrapVue 3][2] in alpha version. However, since Bootstrap is mainly a CSS framework, and it has dropped jQuery dependency in V5, it is not that difficult to integrate into a Vue 3.x project on your own. In this article, we will go through the steps of creating such a project.

## Create Vite project

The recommended way of using Vue 3.x is with Vite. Install `yarn` and create from the `vue-ts` template:

```
yarn create vite bootstrap-vue3 --template vue-ts
cd bootstrap-vue3
yarn install
yarn dev
```

## Add Bootstrap dependencies

Bootstrap is published on npm, and it has an extra dependecy Popper, so let's install them both:

```
yarn add bootstrap @popperjs/core
```

You may also need the type definitions:

```
yarn add -D @types/bootstrap
```

## Use Bootstrap CSS

Just add a line to your `App.vue` file and you are free to use Bootstrap CSS right away:

```vue
<script setup lang="ts">
import 'bootstrap/dist/css/bootstrap.min.css'
</script>

<template>
  <button type="button" class="btn btn-primary">Primary</button>
</template>
```

You can also use Sass for further [customization][3].

<!-- more -->


[1]: https://github.com/bootstrap-vue/bootstrap-vue/issues/5196
[2]: https://cdmoro.github.io/bootstrap-vue-3/
[3]: https://getbootstrap.com/docs/5.2/customize/sass/
