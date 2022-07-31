---
title: Use Bootstrap V5 in Vue 3 Project
tags:
  - bootstrap
  - vue
  - vite
categories: Programming
date: 2022-06-11 20:06:26
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

Bootstrap is published on npm, and it has an extra dependency Popper, so let's install them both:

```
yarn add bootstrap @popperjs/core
```

You may also need the type definitions:

```
yarn add -D @types/bootstrap
```

## Use Bootstrap CSS

Just add a line to your `App.vue` file and you are free to use Bootstrap CSS:

```html
<script setup lang="ts">
import 'bootstrap/dist/css/bootstrap.min.css'
</script>

<template>
  <button type="button" class="btn btn-primary">Primary</button>
</template>
```

You can also use Sass for further [customization][3].

<!-- more -->

## Use JavaScript plugins

Bootstrap provides JS plugins to enable interactive components, such as Modal, Toast, etc. There are two ways of using these plugins: through `data` attributes, or create instances programatically. Let's take [Modal][4] for an example.

### Through `data` attributes

First, you need to import the Bootstrap JS. In the following example, we import the individual Modal plugin. You can also import the full Bootstrap JS using `import 'bootstrap'`.

```html
<script setup lang="ts">
import 'bootstrap/js/dist/modal'
</script>

<template>
  <button type="button" class="btn btn-primary" data-bs-toggle="modal" data-bs-target="#exampleModal">
    Launch demo modal 1
  </button>

  <div class="modal fade" id="exampleModal" tabindex="-1" aria-labelledby="exampleModalLabel" aria-hidden="true">
    <div class="modal-dialog">
      <div class="modal-content">
        <div class="modal-header">
          <h5 class="modal-title" id="exampleModalLabel">Modal title</h5>
          <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
        </div>
        <div class="modal-body">
          Woo-hoo, you're reading this text in a modal!
        </div>
        <div class="modal-footer">
          <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
          <button type="button" class="btn btn-primary">Save changes</button>
        </div>
      </div>
    </div>
  </div>
</template>
```

When the *Launch* button is clicked, `data-bs-toggle` tells Bootstrap to show or hide a modal dialog with the element ID indicated by `data-bs-target`. When the *Close* button is clicked, `data-bs-dismiss` indicates hiding the dialog that contains this button. `data` attribute is simple, but not flexible. In practice, we tend to use JS instance instead.

### Through JS instances

From the Bootstrap document, we see the following instruction:

```js
const myModalAlternative = new bootstrap.Modal('#myModal', options)
```

It creates a `Modal` instance on a DOM element with the ID `myModal`, and then we can call the `show` or `hide` methods on it. In Vue, we need to replace the element ID with a [Template Ref][5]:

```html
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { Modal } from 'bootstrap'

const modalRef = ref<HTMLElement | null>(null)
let modal: Modal
onMounted(() => {
  if (modalRef.value) {
    modal = new Modal(modalRef.value)
  }
})

function launchDemoModal() {
  modal.show()
}
</script>

<template>
  <button type="button" class="btn btn-primary" @click="launchDemoModal">
    Launch demo modal 2
  </button>

  <div class="modal fade" tabindex="-1" aria-hidden="true" ref="modalRef">
    <div class="modal-dialog">
      <div class="modal-content">
        <div class="modal-header">
          <h5 class="modal-title">Modal title</h5>
          <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
        </div>
        <div class="modal-body">
          Woo-hoo, you're reading this text in a modal!
        </div>
        <div class="modal-footer">
          <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
          <button type="button" class="btn btn-primary">Save changes</button>
        </div>
      </div>
    </div>
  </div>
</template>
```

The `modalRef` will be set by Vue when component is mounted, at that time we create the Modal instance with the passed-in DOM element. Note `data-bs-dimiss` still works in this example.

### Write a custom component

If you need to use Modal in different places, it is better to wrap it in a component. Create a `components/Modal.vue` file and put the following code in it:

```html
<script setup lang="ts">
import { ref, onMounted, watch } from 'vue'
import { Modal } from 'bootstrap'

const props = defineProps<{
  modelValue: boolean
  title: string
}>()

const emit = defineEmits<{
  (e: 'update:modelValue', modelValue: boolean): void
}>()

const modalRef = ref<HTMLElement | null>(null)
let modal: Modal
onMounted(() => {
  if (modalRef.value) {
    modal = new Modal(modalRef.value)
  }
})

watch(() => props.modelValue, (modelValue) => {
  if (modelValue) {
    modal.show()
  } else {
    modal.hide()
  }
})

function close() {
  emit('update:modelValue', false)
}
</script>

<template>
  <div class="modal fade" tabindex="-1" aria-hidden="true" ref="modalRef">
    <div class="modal-dialog">
      <div class="modal-content">
        <div class="modal-header">
          <h5 class="modal-title">{{ title }}</h5>
          <button type="button" class="btn-close" aria-label="Close" @click="close"></button>
        </div>
        <div class="modal-body">
          <slot />
        </div>
        <div class="modal-footer">
          <slot name="footer" />
        </div>
      </div>
    </div>
  </div>
</template>
```

We use `v-model` to control the visibility of the modal dialog. By watching the value of `modelValue` property, we call corresponding methods on the Modal instance. Also we have replaced the `data-bs-dismiss` with a function that changes the value of `modelValue`, because that should be the single source of truth of the modal state.

Use this component in a demo view:

```html
<script setup lang="ts">
import { ref } from 'vue'
import Modal from '../components/Modal.vue'

const dialogVisible = ref(false)

function launchDemoModal() {
  dialogVisible.value = true
}

function closeModal() {
  dialogVisible.value = false
}

function saveChanges() {
  closeModal()
  alert('Changes saved.')
}
</script>

<template>
  <button type="button" class="btn btn-primary" @click="launchDemoModal">
    Launch demo modal 3
  </button>

  <Modal v-model="dialogVisible" title="Modal title">
    Woo-hoo, you're reading this text in a modal!
    <template #footer>
      <button type="button" class="btn btn-secondary" @click="closeModal">Close</button>
      <button type="button" class="btn btn-primary" @click="saveChanges">Save changes</button>
    </template>
  </Modal>
</template>
```

Check out the [Vue document][6] to learn about component, slot, v-model, etc. Code examples can be found on [GitHub][7].

[1]: https://github.com/bootstrap-vue/bootstrap-vue/issues/5196
[2]: https://cdmoro.github.io/bootstrap-vue-3/
[3]: https://getbootstrap.com/docs/5.2/customize/sass/
[4]: https://getbootstrap.com/docs/5.2/components/modal/
[5]: https://vuejs.org/guide/essentials/template-refs.html
[6]: https://vuejs.org/guide/essentials/component-basics.html
[7]: https://github.com/jizhang/blog-demo/tree/master/bootstrap-vue3
