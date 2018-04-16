---
title: Form Handling in Vuex Strict Mode
categories: Programming
tags: [frontend, javascript, vue, vuex]
---

![](/images/vue.png)

When handling form inputs in Vue, we usually use `v-model` to achieve two-way binding. But if we want to put form data into Vuex store, two-way binding becomes a problem, since in **strict mode**, Vuex doesn't allow state change outside mutation handlers. Take the following snippet for instance, while full code can be found on GitHub ([link](https://github.com/jizhang/vuex-form)).

`src/store/table.js`

```javascript
export default {
  state: {
    namespaced: true,
    table: {
      table_name: ''
    }
  }
}
```

`src/components/NonStrict.vue`

```html
<b-form-group label="Table Name:">
  <b-form-input v-model="table.table_name" />
</b-form-group>

<script>
import { mapState } from 'vuex'

export default {
  computed: {
    ...mapState('table', [
      'table'
    ])
  }
}
</script>
```

When we input something in "Table Name" field, an error will be thrown in browser's console:

```text
Error: [vuex] Do not mutate vuex store state outside mutation handlers.
    at assert (vuex.esm.js?358c:97)
    at Vue.store._vm.$watch.deep (vuex.esm.js?358c:746)
    at Watcher.run (vue.esm.js?efeb:3233)
```

Apart from not using strict mode at all, which is fine if you're ready to lose some benefits of tracking every mutation to the store, there're several ways to solve this error. In this article, we'll explore these solutions, and explain how they work.

<!-- more -->

## References

* https://vuex.vuejs.org/en/forms.html
* https://markus.oberlehner.net/blog/form-fields-two-way-data-binding-and-vuex/
* https://ypereirareis.github.io/blog/2017/04/25/vuejs-two-way-data-binding-state-management-vuex-strict-mode/
* https://github.com/maoberlehner/vuex-map-fields
* https://forum.vuejs.org/t/vuex-form-best-practices/20084
