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

Another important advantage is better type inference. With Vue 2, TypeScript has a difficulty in inferring types from Options API, so we have to use `Vue.extend` or [class-based components][7]. Though Vue 2.7 backports `defineComponent` that improves this situation, Composition API still provides a more natural and consice way to define types, for it only consists of plain variables and functions. So in this article, I will use TypeScript as the demo language. If your legacy project hasn't adopted TypeScript yet, you can check out my previous post *[Add TypeScript Support to Vue 2 Project][6]*.

For maintainers of larger projects, Composition API also brings better code reuse through custom composable functions, as well as smaller JS bundle and better performance. And last but not least, you can always use both APIs in one project. The Vue team has no plan to remove Options API.

## From Options API to Composition API

The transformation is not difficult, so long as you see the connection between these two APIs. Let's start with a simple component:

```html
<template>
  <div>
    Count: {{ count }}
    <button @click="increment()">Increment</button>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'

export default defineComponent({
  data() {
    return {
      count: 0,
    }
  },
  mounted() {
    this.count = 1
  },
  methods: {
    increment() {
      this.count += 1
    },
  },
})
</script>
```

There is a state, a lifecycle hook, and one method. The Composition API version is:

```html
<template><!-- Not changed --></template>

<script lang="ts">
import { defineComponent, ref, onMounted } from 'vue'

export default defineComponent({
  setup() {
    const count = ref(0)

    onMounted(() => {
      count.value = 1
    })

    function increment() {
      count.value += 1
    }

    return { count, increment }
  },
})
</script>
```

State becomes a `ref`; the `mounted` lifecycle hook becomes an `onMounted` function call; `increment` becomes a plain function. All logics go into the `setup` function of the component definition, and the returned variables can be used in template (`count`, `increment`). You may wonder if you can mix the Composition API with Options API in the same component. The answer is yes, but it is not a good practice, so do it judiciously.

To further simplify the definition, use the syntax sugar `<script setup>`, also available in Vue 2.7:

```html
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const count = ref(0)

onMounted(() => {
  count.value = 1
})

function increment() {
  count.value += 1
}
</script>
```

[1]: https://github.com/vuejs/composition-api
[2]: https://github.com/antfu/unplugin-vue2-script-setup
[3]: https://github.com/vueuse/vue-demi
[4]: https://blog.vuejs.org/posts/vue-2-7-naruto.html
[5]: https://vuejs.org/guide/extras/composition-api-faq.html
[6]: https://shzhangji.com/blog/2022/07/24/add-typescript-support-to-vue-2-project/
[7]: https://github.com/vuejs/vue-class-component
