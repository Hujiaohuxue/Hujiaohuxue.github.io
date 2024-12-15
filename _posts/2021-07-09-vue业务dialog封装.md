---
layout: mypost
title: Vue业务Dialog封装
categories: [前端]
---

在写业务的时候很常见的一个场景就是需要在不同的页面调用同一个表单，常用的交互就是把表单以弹窗的形式展示，但是在每个页面又重复的引入表单组件有时候又很麻烦

![dialog](dialog.png)

解决方案有两个：

1. 在根组件里面引入动态组件，在业务里面通过`this.$root.openDialog(name, props)`去控制动态组件的展示形式

2. 封装成插件的形式去调用，比如`this.$dialog('EditDialog.vue', props)`

当然了，业务 Dialog 组件要有一套规范，props 接收一个 onOk、onCancel 回调，data 里面定义一个 visible 属性

```vue
<template>
  <el-dialog :title="title" :visible.sync="visible" append-to-body>
    <!-- 业务代码 -->
  </el-dialog>
</template>

<script>
export default {
  props: ['onOk', '其他业务需要的属性'],
  data() {
    return {
      visible: false
    }
  }
}
</script>
```

## Vue2 写法

在 Vue2 里面我个人感觉写成插件是比较好用的，实现如下，使用混入做了一些操作，和业务进行解耦

有点不太好的地方是组件是动态插入的，Vue devtools 要刷新下才能看到组件

```js
const mixin = {
  mounted() {
    document.body.appendChild(this.$el)
    this.visible = true
  },
  watch: {
    visible(value) {
      // 动画结束后销毁实例
      if (value === false) {
        setTimeout(() => {
          this.$destroy()
          if (this.$el && this.$el.parentNode) {
            this.$el.parentNode.removeChild(this.$el)
          }
        }, 400)
      }
    }
  }
}

export default {
  install(Vue) {
    Vue.prototype.$dialog = function (name, props) {
      const parentC = this
      // 相对于该插件的位置，静态编译期间会检查的
      import('../components/dialogs/' + name).then(module => {
        const component = { ...module.default }
        const mixins = component.mixins || []
        mixins.push(mixin) // 实现自动打开，动态了混入生命周期函数和销毁操作
        component.mixins = mixins
        component.parent = parentC // vuex 来自 $options.store / options.parent.$store

        const Dialog = Vue.extend(component)
        const dialog = new Dialog({
          propsData: props || {}
        })
        dialog.$mount()
      })
    }
  }
}
```

调用方式如下，注意 onOk 回调的 this 指向，使用箭头函数直接就避免了 😎

```js
this.$dialog('GroupEdit.vue', {
  type: 'edit',
  group: {},
  onOk: () => {
    this.freshList()
  }
})
```

## Vue3 插件版写法

很糟糕的是，由于 Vue3 的升级`Vue.extend`没有了，`$mount`也没有了，组件只能在应用里面去渲染

每个应用之间的数据是隔离的，所以插件什么的都要重新引入。同时如果要交互交互的话也比较麻烦，引入同一个 vuex 实例应该可以，但是没怎试

为了低耦合只能去新建一个应用去挂载渲染

![vue-devtools](vue-devtools.png)

```js
import { createApp, defineComponent } from 'vue'
import ElementPlus from 'element-plus'

const mixin = {
  mounted() {
    document.body.appendChild(this.$el)
    this.visible = true
  },
  watch: {
    visible(value) {
      // 动画结束后销毁实例
      if (value === false) {
        setTimeout(() => {
          this.$.appContext.app.unmount()
        }, 400)
      }
    }
  }
}

export default {
  install(app) {
    app.config.globalProperties.$dialog = (name, props) => {
      import('../components/dialogs/' + name)
        .then(module => {
          const component = module.default
          let mixins = component.mixins || []
          mixins.push(mixin)
          component.mixins = mixins

          return defineComponent(component)
        })
        .then(Dialog => {
          const app = createApp(Dialog, props || {})
          app.use(ElementPlus)
          app.mount(document.createElement('div'))
        })
    }
  }
}
```

## Vue3 动态组件写法

在 Vue3 里面，插件版的写法同样达到了要求，但是完全是一个新引应用了，如果在业务里访问`this.$root`,`vuex`,`router`还是有点麻烦的

所以 Vue3 里面还是动态组件的写法比较好

在根组件引入动态 component，定义一些控制变量

```vue
<template>
  <router-view></router-view>
  <component :is="currentDialog" v-bind="currentDialogProps" />
</template>

<script>
export default {
  data() {
    return {
      currentDialog: null,
      currentDialogProps: null
    }
  }
}
</script>
```

调用的的话`this.$root.$dialog()`,看起来太难看，其实还是可以手动模拟插件的效果的

```js
const app = createApp(App)
const vm = app.mount('#app')

initDialog(app, vm)

function initDialog(app, vm) {
  const mixin = {
    mounted() {
      this.visible = true
    },
    watch: {
      visible(value) {
        // 动画结束后销毁实例
        if (value === false) {
          setTimeout(() => {
            this.$root.currentDialog = null
            this.$root.currentDialogProps = {}
          }, 400)
        }
      }
    }
  }

  app.config.globalProperties.$dialog = (name, props) => {
    import('./components/dialogs/' + name).then(module => {
      const component = module.default
      let mixins = component.mixins || []
      mixins.push(mixin)
      component.mixins = mixins
      // 不需要 defineComponent(component)
      vm.currentDialog = markRaw(component)
      vm.currentDialogProps = markRaw(props || {})
    })
  }
}
```

## Vue3 函数式写法

如果需要共享应用的一些数据，可以通过设置 appContext 来达到效果。可以通过install的方式appContext共享给各个子组件

```js
export default {
  /**
   * install
   * @param {import('vue').App} app
   * @param {*} options
   */
  install(app, options) {
    app.provide('appContext', app._context)
  }
}
```

```js
import { createVNode, render, inject } from 'vue'
import TestDialog from '../components/TestDialog.vue'

const scope = {
  el: null
}

export function useLogin(props) {
  const appContext = inject('appContext')

  function destroy() {
    if (scope.el) {
      render(null, scope.el)
      scope.el.remove()
      scope.el = null
    }
  }

  function open() {
    if (!scope.el) {
      scope.el = document.createElement('div')
      document.body.appendChild(scope.el)
      const vNode = createVNode(TestDialog, {
        ...props,
        destroy: destroy
      })
      vNode.appContext = appContext
      render(vNode, scope.el)
    }
  }

  return open
}
```

## 参考

[Vue 3 with Composition API different object structure for getting context/globalProperties with Vue-CLI 4 in dev build or production build?](https://stackoverflow.com/questions/67168389/vue-3-with-composition-api-different-object-structure-for-getting-context-global)
