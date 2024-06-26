---
layout:     post                    # 使用的布局（不需要改）
title:      update和render两个函数的调用       # 标题 
subtitle:   _update和_render两个函数的调用 #副标题
date:       2022-02-11            # 时间
author:     czk                      # 作者
header-img: img/my/img16.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:  
#标签
    - vue
    
---

#### 前言

书接上文，在`updateComponent`函数最终其实是对`_update`和`_render`两个函数的一次调用
```js
// src/core/instance/lifecycle.js
updateComponent = () => {
  vm._update(vm._render(), hydrating);
};
```

####  _render 函数
`Vue` 的 `_render` 方法是实例的一个私有方法，它用来把实例渲染成一个虚拟 `Node`。

```js
Vue.prototype._render = function(): VNode {
  const vm: Component = this;
  const { render, _parentVnode } = vm.$options;

  if (_parentVnode) {
    vm.$scopedSlots = normalizeScopedSlots(
      _parentVnode.data.scopedSlots,
      vm.$slots,
      vm.$scopedSlots
    );
  }

  // 设置父vnode。这允许渲染函数访问
  vm.$vnode = _parentVnode;
  // render self
  let vnode;
  try {
    // 当父组件patched 的时候
    currentRenderingInstance = vm;
    vnode = render.call(vm._renderProxy, vm.$createElement);
  } catch (e) {
    handleError(e, vm, `render`);
   // 返回错误结果或旧的vnode，以防止渲染错误导致的空白
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== "production" && vm.$options.renderError) {
      try {
        vnode = vm.$options.renderError.call(
          vm._renderProxy,
          vm.$createElement,
          e
        );
      } catch (e) {
        handleError(e, vm, `renderError`);
        vnode = vm._vnode;
      }
    } else {
      vnode = vm._vnode;
    }
  } finally {
    currentRenderingInstance = null;
  }
  // 如果返回的数组只包含一个节点
  if (Array.isArray(vnode) && vnode.length === 1) {
    vnode = vnode[0];
  }
  // 渲染函数出错，返回空的vnode
  if (!(vnode instanceof VNode)) {
    if (process.env.NODE_ENV !== "production" && Array.isArray(vnode)) {
      warn(
        "Multiple root nodes returned from render function. Render function " +
          "should return a single root node.",
        vm
      );
    }
    vnode = createEmptyVNode();
  }
  // 设置父节点
  vnode.parent = _parentVnode;
  return vnode;
};
```
其实这段函数的核心是`vnode = render.call(vm._renderProxy, vm.$createElement)`这里就是`render`方法的调用

- `vm._renderProxy`是什么？

    这里的`_renderProxy`其实是在`Vue.prototype._init`这个函数中定义的,主要如下，
    ```js
    // src/core/instance/init.js
    Vue.prototype._init = function(options?: Object) {
      //...
      if (process.env.NODE_ENV !== "production") {
        // 开发环境下，调用`initProxy`方法，将`vm`作为参数
        initProxy(vm);
      } else {
       // 生产环境下，`vm._renderProxy`就是`vm`本身
        vm._renderProxy = vm; 
      }
      // ...
    };
    ```
    - `initProxy`
    ```js
    // src/core/instance/proxy.js
    let initProxy;
    initProxy = function initProxy(vm) {
      if (hasProxy) {
        // 根据`options.render`和`options.render._withStripped`的值来选择使用
        // `getHandler`还是`hasHandler`
        const options = vm.$options;
        const handlers =
          options.render && options.render._withStripped ? getHandler : hasHandler;
        vm._renderProxy = new Proxy(vm, handlers);
      } else {
        vm._renderProxy = vm;
      }
    };
    ```
    1. 通过`hasProxy`来判断下浏览器是否支持`Proxy`。支持就创建一个`Proxy`对象赋给`vm._renderProxy`；不支持就同生产环境下一样，`vm._renderProxy`就是`vm`本身。
    2. 当使用`vue-loader`解析`.vue`文件时使用`getHandler`,使用`compiler`版本的`Vue.js`则会使用`hasHandler`
    3. 这里代理其实主要是会抛出2种警告提示，一种是在`Vue`中，以`$`或`_`开头的属性不会被代理，因为有可能与内置属性产生冲突。如果你设置的属性以`$`或`_`开头，那么不能直接通过`vm.key`这种形式访问，而是需要通过`vm.$data.key`来访问。 另一种是`key`没有在`data`中定义
    我们就可以对`initProxy`的作用进行一个总结：**在渲染阶段对不合法的数据做判断和处理**。
    
- `vm.$createElement`是什么？
   
   `vm.$createElement`的定义是在`initRender`函数中:
    ```js
    function initRender(vm: Component) {
      // ...
      // 将createElement绑定到这个实例
      // so that we get proper render context inside it.
      // args order: tag, data, children, normalizationType, alwaysNormalize
      // internal version is used by render functions compiled from templates
      vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false);
      // 用户编写的渲染函数。
      vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true);
      // ...
    }
    ```
    
    可以看其实就是分别给实例`vm`加上`_c`和`$createElement`方法。这两个方法都调用了`createElement`方法，只是最后一个参数值不同。
    - 我们调用`$createElement`可以这样写

     ```js
    <div id="app"></div>
        
    <script>
    render: function () {
      return this.$createElement('div', {
         attrs: {
            id: 'app'
          },
      }, this.message)
    },
    data() {
      return {
        message: '我是czkm'
      }
    }
    </script>
    ```
    - 我们平时开发常用的模板字符串
     ```js
    <div id="app">{{ message }}</div>
    <script>
      var app = new Vue({
        el: "#app",
        data() {
          return {
            message: "我是czkm"
          };
        }
      });
    </script>
    ```
    这种使用字符串模板的情况，使用的就是调用`vm._c`的方法了。使用字符串模板，在相关代码执行完前，会先在页面显示 `{{ message }}` ，然后再展示 `我是czkm`；而手写 `render` 函数的话，内部就不会执行把字符串模板转换成 `render` 函数这个操作，并且立即就显示内容
    
    `vm._render` 最终是通过执行 `createElement` 方法只是最后一个参数不同，之后返回`vnode`，它是一个虚拟 `Node`。
    
#### createElement
```js
// src/core/vdom/create-element.js
const SIMPLE_NORMALIZE = 1 // 简单规范化
const ALWAYS_NORMALIZE = 2 // 始终规范化
export function createElement (
  context: Component, // `VNode`当前上下文环境。
  tag: any, // 标签，可以是正常的`HTML`元素标签，也可以是`Component`组件。
  data: any, // `VNode`的数据，其类型为`VNodeData`，定义在根目录`flow/vnode.js`。
  children: any, // `VNode`的子节点
  normalizationType: any, // `children`子节点规范化类型
  alwaysNormalize: boolean // 是否格式化，区别模板编译，还是用户手写render方法
): VNode | Array<VNode> {
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children
    children = data
    data = undefined
  }
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE
  }
  return _createElement(context, tag, data, children, normalizationType)
}
```
 `createElement` 方法实际上是对 `_createElement` 方法的封装，它允许传入的参数更加灵活，在处理这些参数后，调用真正创建 `VNode` 的函数 `_createElement`,多包裹一层的目的是为了让方法达到一种类似于**函数重载**的功能。

对于模板编译调用`_c`时，其`alwaysNormalize`传递的是`false`，`_c`只会在内部使用，因此其方法调用的参数格式无需格式化。

而`$createElement`是用户手写的`render`函数，因为允许用户传递不同形式的参数来调用`$createElement`，所以需要对参数进行格式化。

`$createElement`和`_c`最后一个不相同的参数，体现在调用`_c`时对`children`只是进行简单组合，而调用`$createElement`时必须始终对`children`进行格式化。
```js
export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  // ...省略代码
  if (!tag) {
    // in case of component :is set to falsy value
    return createEmptyVNode()
  }
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
  let vnode, ns
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
      // platform built-in elements
      if (process.env.NODE_ENV !== 'production' && isDef(data) && isDef(data.nativeOn)) {
        warn(
          `The .native modifier for v-on is only valid on components but it was used on <${tag}>.`,
          context
        )
      }
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // parent将children normalizes
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
  // ...省略代码
}
```
`_createElement`主要实现了`children` 的规范化以及 `VNode` 的创建

- children 的规范化

因为虚拟`DOM`是树形结构，每一个节点都应该是`VNode`类型，`_createElement` 接收的第 4 个参数 children 是任意类型的，因此我们需要把它们规范成 VNode 类型。

如果没有子节点，那么`children`就是`undefined`。通过`normalizationType`参数来实现的，其中`normalizationType`可能的值有三种：`undefined`表示不进行规范化，`1`表示简单规范化，`2`表示始终规范化。

1. 值为`1`的情况，调用了`simpleNormalizeChildren`，代码如下：

    ```js
    // src/core/vdom/helpers/normalize-children.js
    export function simpleNormalizeChildren (children: any) {
      for (let i = 0; i < children.length; i++) {
        if (Array.isArray(children[i])) {
          return Array.prototype.concat.apply([], children)
        }
      }
      return children
    }
    ```
    `simpleNormalizeChildren`的作用是把多维数组降低一个维度，例如二维数组降低到一维数组，三维数组降低到二维数组，这样做的目的是为了方便后续遍历`children`

2. 值为`2`的情况，它调用了`normalizeChildren`，其代码如下：

    ```js
    // src/core/vdom/helpers/normalize-children.js
    export function normalizeChildren (children: any): ?Array<VNode> {
      return isPrimitive(children)
        ? [createTextVNode(children)]
        : Array.isArray(children)
          ? normalizeArrayChildren(children)
          : undefined
    }
    ```
    `normalizeChildren`作用是判断当`children`是基础类型值的时候，直接返回一个文本节点的`VNode`数组，否则再判断是否为数组，是的话就调用`normalizeArrayChildren`来规范化，不是则其`children`就是`undefined`。

    - `normalizeArrayChildren`
    ```js
    function normalizeArrayChildren (children: any, nestedIndex?: string): Array<VNode> {
      const res = []
      let i, c, lastIndex, last
      for (i = 0; i < children.length; i++) {
        c = children[i]
        if (isUndef(c) || typeof c === 'boolean') continue
        lastIndex = res.length - 1
        last = res[lastIndex]
        //  嵌套
        if (Array.isArray(c)) {
          if (c.length > 0) {
            c = normalizeArrayChildren(c, `${nestedIndex || ''}_${i}`)
            // 合并相邻文本节点
            if (isTextNode(c[0]) && isTextNode(last)) {
              res[lastIndex] = createTextVNode(last.text + (c[0]: any).text)
              c.shift()
            }
            res.push.apply(res, c)
          }
        } else if (isPrimitive(c)) {
          if (isTextNode(last)) {
            // 合并相邻文本节点
            res[lastIndex] = createTextVNode(last.text + c)
          } else if (c !== '') {
            // convert primitive to vnode
            res.push(createTextVNode(c))
          }
        } else {
          if (isTextNode(c) && isTextNode(last)) {
            // 合并相邻文本节点
            res[lastIndex] = createTextVNode(last.text + c.text)
          } else {
            // 嵌套的子数组的默认键(例如由v-for生成)
            if (isTrue(children._isVList) &&
              isDef(c.tag) &&
              isUndef(c.key) &&
              isDef(nestedIndex)) {
              c.key = `__vlist${nestedIndex}_${i}__`
            }
            res.push(c)
          }
        }
      }
      return res
    }
    ```
    上述代码好像看起来很多，其实`normalizeArrayChildren`主要做的了3个判断，我们来分析下。

    首先`normalizeArrayChildren` 接收 2 个参数，`children` 表示要规范的子节点，`nestedIndex` 表示嵌套的索引，因为单个 `child` 可能是一个数组类型。 `normalizeArrayChildren` 主要的逻辑就是遍历 `children`，获得单个节点 `c`，然后对 `c` 的类型判断

    1. 如果是数组类型，则递归调用 `normalizeArrayChildren`，举个例子
    2. 如果是基础类型，调用封装的`createTextVNode`方法来创建一个文本节点，然后`push`到结果数组`res`中。
    3. 如果不属于以上两种情况，那么代表本身已经是`VNode`类型了，直接`push`到结果数组中即可。

    着3种情况都判断了`isTextNode`，表示如果存在两个连续的 `text` 节点，会把它们合并成一个 `text` 节点
    ```js
    // 合并前
    const children = [
      { text: 'Hello ', ... },
      { text: 'czkm', ... },
    ]

    // 合并后
    const children = [
      { text: 'Hello czkm' ... }
    ]
    ```

    经过对 `children` 的规范化，`children` 变成了一个类型为 VNode 的 Array。

- `VNode` 的创建
    ```js
    let vnode, ns
    if (typeof tag === 'string') {
      let Ctor
      ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
      if (config.isReservedTag(tag)) {
        // platform built-in elements
        vnode = new VNode(
          config.parsePlatformTagName(tag), data, children,
          undefined, undefined, context
        )
      } else if (isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
        // component
        vnode = createComponent(Ctor, data, context, children, tag)
      } else {
       // 未知或未列出的名称空间元素
       // 在运行时检查，因为它可能被分配一个命名空间
        vnode = new VNode(
          tag, data, children,
          undefined, undefined, context
        )
      }
    } else {
      // direct component options / constructor
      vnode = createComponent(tag, data, context, children)
    }
    ```
    创建`VNode`节点的逻辑有两大分支，`tag`为`string`类型和`component`类型
    
     `tag`如果是 `string` 类型，判断如果是内置的一些节点(例如`html`或`svg`标签)，则直接创建一个普通 `VNode`如果不是则尝试在已经全局或者局部注册的组件中去匹配，匹配成功则使用`createComponent`去创建组件节点。`tag`  如果是  `Component` 类型，则直接调用 `createComponent` 创建一个组件类型的 `VNode`。例如：
     ```js
    <template>
      <div id="app">
        <div>{{msg}}</div>
        <hello-world :msg="msg" />
        <test/>
      </div>
    </template>
    <script>
    import HelloWorld from '@/components/HelloWorld.vue'
    export default {
      name: 'App',
      data () {
        return {
          msg: 'message',
        }
      },
      components: {
        HelloWorld
      }
    }
    </script>
    ```

    这里的`tag`为`test`，但`div`这种内置标签节点，又不像`hello-world`是已经局部注册过的组件，它属于未知的标签。这里之所以直接创建未知标签的`VNode`而不是报错，这是因为子节点在`createElement`的过程中，有可能父节点会为其提供一个`namespace`，真正做未知标签校验的过程发生在`path`阶段。
   
  
####  _update 函数
回到 `mountComponent` 函数， `vm._render` 创建了一个 `VNode`，接下来就是要把这个 `VNode` 渲染成一个真实的` DOM`,而这个过程正是通过`vm._update`来完成的。

`Vue` 的 `_update` 是实例的一个私有方法，它被调用的时机有 2 个，一个是首次渲染，一个是数据更新的时候,主要作用是把 `VNode` 渲染成真实的 `DOM`

```js
// src/core/instance/lifecycle.js
 Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  // 页面的挂载点，真实的元素
  const prevEl = vm.$el
  // 老 VNode
  const prevVnode = vm._vnode
  const restoreActiveInstance = setActiveInstance(vm)
  // 新 VNode
  vm._vnode = vnode
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // 老 VNode 不存在，表示首次渲染，即初始化页面时走这里
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // 响应式数据更新时，即更新页面时走这里
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  restoreActiveInstance()
  // update __vue__ reference
  if (prevEl) {
    prevEl.__vue__ = null
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm
  }
  // if parent is an HOC, update its $el as well
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el
  }
  // updated hook is called by the scheduler to ensure that children are
  // updated in a parent's updated hook.
}
```


`_update`方法使用`setActiveInstance`来设置当前激活的实例，使用`restoreActiveInstance`来恢复，`setActiveInstance`方法定义如下

```js
const restoreActiveInstance = setActiveInstance(vm)
export function setActiveInstance(vm: Component) {
    // 定义了闭包变量保存了当前激活的实例
  const prevActiveInstance = activeInstance
  activeInstance = vm
  return () => {
  //这个函数的目的是用来恢复`activeInstance`到上一个缓存下来的激活实例
    activeInstance = prevActiveInstance
  }
}
```
`_update`代码并不是很多，其核心就是调用`__patch__`方法，`__patch__` 函数在不同平台会有不同的定义

```js
// 判断了当前是否处于浏览器环境
Vue.prototype.__patch__ = inBrowser ? patch : noop
```
使用`inBrowser`判断了当前是否处于浏览器环境，如果是则赋值为`path`，否则就是`noop`空函数。这样判断是因为`Vue`还可以运行在`node`服务端

    ```js
    import * as nodeOps from 'web/runtime/node-ops'
    //`node-ops.js`文件中封装的方法，实际上就是对真实`DOM`操作的一层封装，传递`nodeOps`的
    // 目的是为了在虚拟`DOM`转成真实`DOM`节点的过程中提供便利
    import { createPatchFunction } from 'core/vdom/patch'
    //`baseModules`是对模板标签上`ref`和`directives`各种操作的封装
    import baseModules from 'core/vdom/modules/index'
    // `platformModules`是对模板标签上`class`、`style`、`attr`以及`events`等操作的封装。
    import platformModules from 'web/runtime/modules/index'

    // modules 是由 platformModules 和 baseModules 合并而来
    // 里面定义了一些模块的钩子函数的实现
    const modules = platformModules.concat(baseModules)

    // 传入平台特有的一些操作，然后返回一个 patch 函数
    export const patch: Function = createPatchFunction({ nodeOps, modules })
    ```
来对`_update`做一个总结：

1. 首次渲染和派发更新重新渲染的`patch`是差异的，表现为首次渲染时提供的根节点是一个真实的`DOM`元素，在派发更新重新渲染时提供的是一个`VNode`
    ```js
      //......
    if (!prevVnode) {
        // 老 VNode 不存在，表示首次渲染，即初始化页面时走这里
        vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
      } else {
        // 响应式数据更新时，即更新页面时走这里
        vm.$el = vm.__patch__(prevVnode, vnode)
      }
      //......
    ```
2. 父子组件递归渲染的时候，首先渲染子组件，子组件渲染完毕后才会去渲染父组件，在这过程中，`activeInstance`始终指向当前渲染的组件实例。同时根据父子组件递归渲染的顺序，父组件`created`后,子组件才开始渲染，具体的的执行顺序为：
    ```js
    // parent beforeCreate
    // parent created
    // parent beforeMount
    // child beforeCreate
    // child created
    // child beforeMount
    // child mounted
    // parent mounted
    ```
 3. `render`函数执行会得到一个`VNode`的树形结构，`update`的作用就是把这个虚拟`DOM`节点树转换成真实的`DOM`节点树。
 
![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3796b310eecd451ea94df2ec8e618da0~tplv-k3u1fbpfcp-watermark.image?)
