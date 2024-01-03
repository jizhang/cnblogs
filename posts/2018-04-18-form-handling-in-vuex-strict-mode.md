---
title: Vuex 严格模式下的表单处理
tags:
  - frontend
  - javascript
  - vue
  - vuex
categories: Programming
date: 2018-04-18 09:08:41
---



![](/images/vue.png)

在使用 Vue 进行表单处理时，我们通常会使用 `v-model` 来建立双向绑定。但是，如果将表单数据交由 Vuex 管理，这时的双向绑定就会引发问题，因为在 **严格模式** 下，Vuex 是不允许在 Mutation 之外的地方修改状态数据的。以下用一个简单的项目举例说明，完整代码可在 GitHub（[链接](https://github.com/jizhang/blog-demo/tree/master/vuex-form)） 查看。

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
<b-form-group label="表名：">
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

当我们在“表名”字段输入文字时，浏览器会报以下错误：

```text
错误：[vuex] 禁止在 Mutation 之外修改 Vuex 状态数据。
    at assert (vuex.esm.js?358c:97)
    at Vue.store._vm.$watch.deep (vuex.esm.js?358c:746)
    at Watcher.run (vue.esm.js?efeb:3233)
```

当然，我们可以选择不开启严格模式，只是这样就无法通过工具追踪到每一次的状态变动了。下面我将列举几种解决方案，描述如何在严格模式下进行表单处理。

<!-- more -->

## 将状态复制到组件中

第一种方案是直接将 Vuex 中的表单数据复制到本地的组件状态中，并在表单和本地状态间建立双向绑定。当用户提交表单时，再将本地数据提交到 Vuex 状态库中。

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

以上方式有两个缺陷。其一，在提交状态更新后，若继续修改表单数据，同样会得到“禁止修改”的错误提示。这是因为 `setTable` 方法将本地状态对象直接传入了 Vuex，我们可以对该方法稍作修改：

```javascript
setTable (state, payload) {
  // 将对象属性逐一赋值给 Vuex
  _.assign(state.table, payload)
  // 或者，克隆整个对象
  state.table = _.cloneDeep(payload)
}
```

第二个问题在于如果其他组件也向 Vuex 提交了数据变动（如弹出的对话框中包含了一个子表单），当前表单的数据不会得到更新。这时，我们就需要用到 Vue 的监听机制了：

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

这个方法还能同时规避第一个问题，因为每当 Vuex 数据更新，本地组件都会重新克隆一份数据。

## 响应表单更新事件并提交数据

一种类似 ReactJS 的做法是，弃用 `v-model`，转而使用 `:value` 展示数据，再通过监听 `@input` 或 `@change` 事件来提交数据变更。这样就从双向绑定转换为了单向数据流，Vuex 状态库自此成为整个应用程序的唯一数据源（Single Source of Truth）。

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

以上方法也是 [Vuex 文档](https://vuex.vuejs.org/en/forms.html) 所推崇的。而根据 [Vue 文档](https://vuejs.org/v2/guide/forms.html) 的介绍，`v-model` 本质上也是一个“监听 - 修改”流程的语法糖而已。

## 使用 Vue 计算属性

Vue 的计算属性（Computed Property）可以配置双向的访问器（Getter / Setter），我们可以利用其建立起 Vuex 状态库和本地组件间的桥梁。其中一个限制在于计算属性无法支持嵌套属性（`table.table_name`），因此我们需要为这些属性设置别名。

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

如果表单字段数目过多，全部列出不免有些繁琐，我们可以创建一些工具函数来实现。首先，在 Vuex 状态库中新增一个可修改任意属性的 Mutation，它接收一个 Lodash 风格的属性路径。

```javascript
mutations: {
  myUpdateField (state, payload) {
    const { path, value } = payload
    _.set(state, path, value)
  }
}
```

在组件中，我们将传入的“别名 - 路径”对转换成相应的 Getter / Setter 访问器。

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

开源社区中已经有人建立了一个名为 [vuex-map-fields](https://github.com/maoberlehner/vuex-map-fields) 的项目，其 `mapFields` 方法就实现了上述功能。

## 参考资料

* https://vuex.vuejs.org/en/forms.html
* https://ypereirareis.github.io/blog/2017/04/25/vuejs-two-way-data-binding-state-management-vuex-strict-mode/
* https://markus.oberlehner.net/blog/form-fields-two-way-data-binding-and-vuex/
* https://forum.vuejs.org/t/vuex-form-best-practices/20084
