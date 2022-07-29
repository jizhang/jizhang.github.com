---
title: Use Composition API and Pinia in Vue 2 Project
tags: [frontend, vue, pinia, typescript]
categories: Programming
---

Composition API is one of the major features of Vue 3, and it greatly changes how we organize code. Vue 3 also introduces Pinia as the recommended state management library, superceding Vuex that now enters maintenance mode. It would be nice if we can use these cool features in Vue 2 project, since migration of legacy project could be difficult and costly. Fortunately, the community has tried hard to bring Vue 3 features back to Vue 2, like [`@vue/composition-api`][1], [`unplugin-vue2-script-setup`][2] and [`vue-demi`][3]. Recently, [Vue 2.7][4] is released and backports features like Composition API, `<script setup>`, `defineComponent`, etc. This article will show you how to change your code from Options API to Composition API, from Vuex to Pinia.

## Why Composition API

The main advantage of Composition API is that you can organize your code in a more flexiable way. Previously with Options API, we can only group codes by `data`, `methods`, and hooks, while with Composition API, codes constituting one feature can be put together. There is a nice figure in the official document *[Composition API FAQ][5]* that illustrates how code blocks look differently after applying Composition API.

![Options API vs. Composition API](/images/typescript/composition-api-after.png)

<!-- more -->

[1]: https://github.com/vuejs/composition-api
[2]: https://github.com/antfu/unplugin-vue2-script-setup
[3]: https://github.com/vueuse/vue-demi
[4]: https://blog.vuejs.org/posts/vue-2-7-naruto.html
[5]: https://vuejs.org/guide/extras/composition-api-faq.html
