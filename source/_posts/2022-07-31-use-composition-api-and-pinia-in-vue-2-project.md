---
title: Use Composition API and Pinia in Vue 2 Project
tags:
  - frontend
  - vue
  - pinia
  - typescript
categories: Programming
date: 2022-07-31 14:07:27
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

## More on states

`ref` is used to define a single state variable, and we have to use `.value` to get and set its value. You can pass an object or array to `ref`, but it is not convenient to change only one member of the state, like changing a field value in a form. So `reactive` would be a better choice here.

```html
<template>
  <div>
    <form @submit.prevent="login()">
      <input v-model="form.username" placeholder="Username" />
      <input v-model="form.password" placeholder="Password" type="password" />
      <button type="submit">Login</button>
    </form>
  </div>
</template>

<script setup lang="ts">
import { reactive } from 'vue'

const form = reactive({
  username: '',
  password: '',
})

function login() {
  console.log({ ...form })
}
</script>
```

`reactive` looks much more like the `data` section in Options API. The difference is you can define multiple `ref` and `reactive`s in one component, place them nearer to where they are used. There are other topics on component state, like `computed` and `watch`, please take a look at the official document [*Reactivity API: Core*][8].

## Define component's `props` and `emits`

Let's wrap login form into a component, to see how `props` and `emits` are defined:

```html
<template><!-- Not changed --></template>

<script setup lang="ts">
import { reactive, defineProps, defineEmits } from 'vue'

export interface Props {
  username: string
  password: string
}

const props = defineProps<Props>()

const emit = defineEmits<{
  (e: 'login', form: Props): void
}>()

const form = reactive({ ...props })

function login() {
  emit('login', { ...form })
}
</script>
```

This component takes `props` as the initial values of form fields, and when the form is submitted, it emits the `login` event to parent component:

```html
<template>
  <LoginForm username="admin" password="admin" @login="login" />
</template>

<script setup lang="ts">
import LoginForm, { type Props } from './LoginForm.vue'

function login(form: Props) {
  console.log(form)
}
</script>
```

We can see `props` and `emits` are both strongly typed, so TS will highlight any violation of the component interface.

Template refs are also supported in Composition API with TS. I wrote a post about wrapping Bootstrap 5 modal into a Vue component, with template ref and `v-model`. Please check out [*Use Bootstrap V5 in Vue 3 Project*][9].

## From Vuex to Pinia

State management library is often used when you want to share states between different components. Rather than *lifting the state up*, we use a dedicated global state store that results in cleaner code and good separation of concerns. A store is also used to interact with backend APIs, and it gives better integration with DevTools. In fact, using a state store has become a standard approach in frontend development.

In Vue 2, the default state management library is Vuex, and that is changing in Vue 3, because you can either use Reactivity API (`ref`, `reactive`, etc.) or Pinia to replace it with. I am not covering every aspect of Vuex or Pinia, just showing you how to convert a daily used Vuex store into new forms. Like this simple user store in Vuex 3.x:

```js
import * as service from '@/services/user'

const types = {
  SAVE: 'save',
}

export default {
  state: {
    username: '',
  },
  actions: {
    async login({ commit }, payload) {
      const response = await service.login(payload)
      commit(types.SAVE, {
        username: response.data.payload.username,
      })
    },
  },
  mutations: {
    [types.SAVE](state, payload) {
      Object.assign(state, payload)
    },
  },
}
```

The `login` method sends username and password to remote API and if login successfully, save the username to its state. Then the state can be shared among components like nav bar, a dropdown of user list, etc. The Pinia version removes the mutation part, thus making the store a little bit simpler:

```ts
import { defineStore } from 'pinia'
import * as service from '@/services/user'

export default defineStore('user', {
  state: () => ({
    username: '',
  }),
  actions: {
    async login(data: object) {
      const response = await service.login(data)
      this.username = response.data.payload.username
    },
  },
})
```

Removing mutation may be the biggest improvement. Pinia also has better type inference out of the box, while in Vuex we need to define complex wrappers around store. Both integrates well with Composition API, because Vuex 4.x is built for Vue 3.x. Detailed comparison can be found on Pinia's [official document][10]. To use the store:

```html
<template>
  <div>Username: {{ store.username }}</div>
</template>

<script setup lang="ts">
import useStore from '@/stores/user'

const store = useStore()

function login() {
  store.login({ username: '', password: '' })
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
[8]: https://vuejs.org/api/reactivity-core.html
[9]: https://shzhangji.com/blog/2022/06/11/use-bootstrap-v5-in-vue3-project/
[10]: https://pinia.vuejs.org/introduction.html#comparison-with-vuex-3-x-4-x
