---
title: Form Handling in Vuex Strict Mode
tags:
  - frontend
  - javascript
  - vue
  - vuex
categories: Programming
date: 2018-04-17 14:13:40
---


![](/images/vue.png)

When handling form inputs in Vue, we usually use `v-model` to achieve two-way binding. But if we want to put form data into Vuex store, two-way binding becomes a problem, since in **strict mode**, Vuex doesn't allow state change outside mutation handlers. Take the following snippet for instance, while full code can be found on GitHub ([link](https://github.com/jizhang/blog-demo/tree/master/vuex-form)).

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

The first solution is to copy the form data from Vuex store to local state, do normal two-way binding, and commit to store when user submits the form.

`src/components/LocalCopy.vue`

```html
<b-form-input v-model="table.table_name" />

<script>
import _ from 'lodash'

export default {
  data () {
    return {
      table: _.cloneDeep(this.$store.state.table.table)
    }
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

## Explicit Update

A ReactJS-like approach is to commit data on input / change event, i.e. use one-way data binding instead of two-way, and let Vuex store become the single source of truth of your application.

`src/components/ExplicitUpdate.vue`

```html
<b-form-input :value="table.table_name" @input="updateTableForm({ table_name: $event })" />

<script>
export default {
  computed: {
    ...mapState('table', [
      'table'
    ])
  },

  methods: {
    ...mapMutations('table', [
      'updateTableForm'
    ])
  }
}
</script>
```

`src/store/table.js`

```javascript
export table {
  mutations: {
    updateTableForm (state, payload) {
      _.assign(state.table, payload)
    }
  }
}
```

This is also the recommended way of form handling in Vuex [doc](https://vuex.vuejs.org/en/forms.html), and according to Vue's [doc](https://vuejs.org/v2/guide/forms.html), `v-model` is essentially a syntax sugar for updating data on user input events.

## Computed Property

Vue's computed property supports getter and setter, we can use it as a bridge between Vuex store and component. One limitation is computed property doesn't support nested property, so we need to make aliases for nested states.

`src/components/ComputedProperty.vue`

```html
<b-form-input v-model="tableName" />
<b-form-select v-model="tableCategory" />

<script>
export default {
  computed: {
    tableName: {
      get () {
        return this.$store.state.table.table.table_name
      },
      set (value) {
        this.updateTableForm({ table_name: value })
      }
    },

    tableCategory: {
      get () {
        return this.$store.state.table.table.category
      },
      set (value) {
        this.updateTableForm({ category: value })
      }
    },
  },

  methods: {
    ...mapMutations('table', [
      'updateTableForm'
    ])
  }
}
</script>
```

When there're a lot of fields, it becomes quite verbose to list them all. We may create some utilities for this purpose. First, in Vuex store, we add a common mutation that can set arbitrary state indicated by a lodash-style path.

```javascript
mutations: {
  myUpdateField (state, payload) {
    const { path, value } = payload
    _.set(state, path, value)
  }
}
```

Then in component, we write a function that takes alias / path pairs, and creates getter / setter for them.

```javascript
const mapFields = (namespace, fields) => {
  return _.mapValues(fields, path => {
    return {
      get () {
        return _.get(this.$store.state[namespace], path)
      },
      set (value) {
        this.$store.commit(`${namespace}/myUpdateField`, { path, value })
      }
    }
  })
}

export default {
  computed: {
    ...mapFields('table', {
      tableName: 'table.table_name',
      tableCategory: 'table.category',
    })
  }
}
```

In fact, someone's already created a project named [vuex-map-fields](https://github.com/maoberlehner/vuex-map-fields), whose `mapFields` utility does exactly the same thing.

## References

* https://vuex.vuejs.org/en/forms.html
* https://ypereirareis.github.io/blog/2017/04/25/vuejs-two-way-data-binding-state-management-vuex-strict-mode/
* https://markus.oberlehner.net/blog/form-fields-two-way-data-binding-and-vuex/
* https://forum.vuejs.org/t/vuex-form-best-practices/20084
