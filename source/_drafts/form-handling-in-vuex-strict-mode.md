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

## Local Copy

The first solution is to copy the form data from Vuex store to local state, do normal two-way binding, and commit to store when form is submitted.

`src/components/LocalCopy.vue`

```html
<b-form-input v-model="table.table_name" />

<script>
import _ from 'lodash'

export default {
  state: {
    table: _.cloneDeep(this.$store.state.table.table)
  },

  methods: {
    handleSubmit (event) {
      this.$store.commit('table/setTable', this.table)
    }
  }
}
</script>
```

`src/store/table.js`

```javascript
export default {
  mutations: {
    setTable (state, payload) {
      state.table = payload
    }
  }
}
```

There're two caveats in this solution. One is when you try to update the form after committing to store, you'll again get "Error: [vuex] Do not mutate vuex store state outside mutation handlers." It's because the component's local copy is assigned into Vuex store. We can modify the `setTable` mutation to solve it.

```javascript
setTable (state, payload) {
  // assign properties individually
  _.assign(state.table, payload)
  // or, clone the payload
  state.table = _.cloneDeep(payload)
}
```

Another problem is when other components commit changes to Vuex store's `table`, e.g. in a dialog with sub-forms, current component will not be updated. In this case, we'll need to set a watched property.

```html
<script>
export default {
  data () {
    return {
      table: _.cloneDeep(this.$store.state.table.table)
    }
  },

  computed: {
    storeTable () {
      return _.cloneDeep(this.$store.state.table.table)
    }
  },

  watch: {
    storeTable (newValue) {
      this.table = newValue
    }
  }
}
</script>
```

This approach can also bypass the first caveat, because following updates in component's form will not affect the object inside Vuex store.

## References

* https://vuex.vuejs.org/en/forms.html
* https://ypereirareis.github.io/blog/2017/04/25/vuejs-two-way-data-binding-state-management-vuex-strict-mode/
* https://markus.oberlehner.net/blog/form-fields-two-way-data-binding-and-vuex/
* https://forum.vuejs.org/t/vuex-form-best-practices/20084
