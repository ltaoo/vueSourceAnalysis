# vue 源码阅读过程

## 起点

```javascript
new Vue({
	el: '#app',
	template: '<App />',
	components: { App },
});
```

从这里开始，发生了什么？

![function Vue](./constructor.png)


## _init

在`_init`方法中，主要就是给`vm`添加了一些属性

- vm._uid = uid++
- vm._isVue = true
- vm._renderProxy = vm
- vm.$options = options
- vm._self = vm

![_init](./init.png)

可以看到，最后面是进行了一系列的初始化函数调用，并且调用了两个生命周期钩子，`beforeCreate`和`created`，这又代表什么含义呢？

### initLifecycle

从名字看是初始化生命周期，实际上也是给`vm`实例添加了很多属性。

- vm.$parent = options.parent
- vm.$root = $parent.root || vm
- vm.$children = []
- vm.$refs = {}
- vm._watcher = null
- vm._inactive = null
- vm._directInactive = false
- vm._isMounted = false = false
- vm._isDestroyed = false
- vm._isBeingDestroyed = false

总的来说，就是`$parent`比较麻烦些，大部分情况应该都是`undefined`吧，不然`vm.$options.parent`是什么时候挂载的？

有点出乎意料的是，在这个函数入口打断点，看到的`vm.$options`不只初始化时传入的属性，还有一些框架添加的，但是在什么时候添加的不清楚。[1]

![initLifecycle](./initLifecycle.png)

### initEvent

从名字就看得出来，在这个函数内会对事件进行初始化，但是，是对什么事件进行初始化？

可以看到是从`$options._parentListeners`上取到所谓的`listeners`，如果存在，就更新到组件上。

但是默认情况是没有的，要怎么添加，才可以在这里得到呢？[2]

![initEvents](./initEvents.png)


### initRender

然后到了这里，从名字来看是初始化`render`？

- vm._vnode = null
- vm.$solts = resolveSlots()
- vm.$scopedSlots = emptyObject
- vm._c = func
- vm.$createElement = func
- vm.$attrs
- vm.$listeners

![initRender](./initRender.png)

在最后，调用`defineReactive`方法，这个方法接收`obj`、`key`和`val`，即一个对象及指定键值对，然后对值注册监听器。

![defineReactive](./defineReactive.png)

那是对我们传入的`data`做监听，实现数据响应吗，不是，从`initRender`这张图片最后可以看到，是对`$attrs`和`$listeners`这两个`vm`上的属性做了这个处理。

所以这里就要深究`Dep`到底是什么，怎么工作的[3]

### Dep

注释是：
> A dep is an observable that can have multiple directives subscribing to it.


每个`Dep`实例有`subs`属性，保存了多个`Watcher`实例，在调用`dep.notify`方法时，会遍历该数组，对每个`Watcher`实例调用`update`方法。


### initInjections

噢，完全不明白这个函数用来做什么，至今为止没有主动用到过`inject`这个配置项，到后面遇到的时候再回过头来看看吧。

![initInjections](./initInjections.png)


### initState

终于到了我们最熟悉的地方，在`initState`函数内，会对`props`、`methods`、`data`、`computed`以及`watch`进行处理，对，就是我们在日常中最常用到的这些，都在`initState`函数内进行处理。

![initState](./initState.png)

可以看到，先判断属性是否存在，然后依次调用

- initProps

#### initProps

初始化`props`，虽然听起来很简单，但实际上也是做了很多事情，先对`props`做验证，然后是返回值（如果传入了就用传入的，否则就用默认的）。

- vm._props = {}
- vm.$options._propKeys = []

![initProps](./initProps.png)

#### initMethods

这是最简单的函数了，在开发环境会做一些错误提示，比如该方法值是`null`，或者已经是一个`props`之类的，在生产环境就直接会将该方法放到`vm`上，如果是`null`就变成一个空函数。

![initMethods](./initMethods.png)

#### initData

其实在`initData`内并没有做什么，主要还是一些重复属性的错误提示，最主要的工作还是由`observe`来做的，后面再对这个进行分析。

![initData](./initData.png)

#### initComputed

很复杂，虽然在看之前就已经预料到会很复杂。

主要做了两件事，1、给每个`key`实例化了一个`Watcher`并保存；2、在`vm`上添加了`computed`同名的`key`，并且在获取值时会调用之前实例化的`watcher`上的方法。

但，为什么函数体内的值发生变化的时候，会触发到自身调用呢？

- vm._computedWatchers  保存了很多个 Watcher 实例。

![initComputed](./initComputed.png)

#### initWatch

初始化监听，很简单，遍历`watch`并给每个键调用`createWatcher`，而在`createWatcher`中其实就是调用了`vm.$watch`，而在`vm.$watch`中，其实最终还是实例化`Watcher`。

![initWatch](./initWatch.png)

### initProvide

只是添加了`vm._provide`这一个属性。

![initProvide](./initProvide.png)

## $mount


OK，前面终于完成了初始化，到目前为止都是平台无关的，接下来就和运行的平台有关了，所以是分为了`web`与`weex`两个不同的`$mount`实现，当然这里只会分析`web`端的实现，这部分的入口在`vue/platforms/web/entry-runtime-with-compiler.js`。

开始吧！

在这个函数内，主要是对`el`、`template`做处理，需要将这两个转换为`render`函数，当然如果已经有`render`函数，会忽略上面两个。

由于存在“编译”，所以还是很复杂，主要逻辑在`compileToFunctions`函数内。

![mount](./mount.png)

### compile

额，先来说说`compile`，因为`compileToFunctions`是调用`createCompileToFunctionsFn`时传入`compile`得到的。

![compile](./compile.png)

### createCompileToFunctionFn

其实就是返回了`compileToFunctions`这个函数，额外的处理就是多了`cache`闭包，能够缓存一些重复的编译工作？

重点还是`compileToFunctions`函数：

### compileToFunctions

调用了传进来的`compile`，就是上面那个函数得到了`compiled`变量，再调用`createFunction`并传入了`compile.render`就得到了`render`函数。

![createCompileToFunctionsFn](./createCompileToFunctionsFn.png)

### baseCompile

噢，虽然知道了`options.render`函数是调用`compileToFunction`得到，而在这个函数里面实际是调用了`createFunction`并传入`compile.render`得到的，与之对应的`compiled.render`又是通过调用`compile`得到的，但最最最内部，实际还是调用的`baseCompile`对`template`解析得到一切。

而这个`baseCompile`是在调用`createCompilerCreator`函数时传入的，所以说，其实是可以定制化核心的那个编译器？

编译器做的事情看起来蛮简单的，

- 1、ast = parse(template, options)
- 2、optimize(ast, options)
- 3、code = generate(ast, options)

所以又要拆开来看了

#### parse

太复杂了。。。。就是语法分析生成`AST`那一套，可能由于还要处理自定义指令，模板等所以更复杂？

暂时还是不看了，`optimize`也是比较复杂，最后`generate`稍微简单一些。

#### generate

会根据传入的`ast`判断生成何种类型的节点，组件、原生DOM 亦或者是 if for 之类的处理？

不过最终其实只是拼接成了`_c(componentName)`这种形式，所以我们看到的每一个组件都是这样的，这里得到字符串，然后使用`new Function`得到函数，最后挂载到了`options.render`上。

![generate](./generate.png)

### mountComponent

有趣的是，貌似是先在`runtime/index.js`里面声明了`$mount`方法，然后又在`entry-runtime-with-compiler.js`保存了一下，然后覆盖。

这就导致了，完成编译后，会调用`runtime/index.js`的`mount`方法，其实就是调用`mountComponent`方法。

- vm.$el = el
- vm._watcher = new Watcher()
- vm._isMounted = true

![mountComponent](./mountComponent.png)

由于存在两个组件，所以会触发两次`mountComponent`，在第二次断点的时候，会从`new Watcher()`到`updateComponent`这个函数里面来，所以要先看看`new Watcher()`到底做了啥。

最后有这么一段代码：

```javascript
this.value = this.lazy ? undefined : this.get()
```

意思就是说，在实例化`watcher`的时候，如果不是`lazy`，就会立刻去获取值放到`watcher.value`上。而`this.get()`重点是这段：

```javascript
value = this.getter.call(vm, vm)
```

噢，又到了`this.getter`这个函数，而这个函数却是实例化`watcher`时传入的第二个参数！实际上就是我们在`mountComponent`中声明的`updateComponent`函数！

所以接下来就是调用

```javascript
vm._update(vm._render(), hydrating)
```

### vm._render

来到了`core/instance/render.js`这个文件。

最核心的就是调用`render`方法，得到`vnode`，其他的就是添加了一些属性。

- vm.$vnode = _parentVnode

`render`函数是由模板解析得到的，我们直接查看下`vnode`到底是什么

```javascript
vnode = VNode {
    elm: undefined,
    functionalContext: undefined,
    functionalOptions: undefined,
    functionalScopeId: undefined,
    isAsyncPlaceholder: false,
    isCloned: false,
    isComment: false,
    isOnce: false,
    isRootInsert: true,
    isStatic: false,
    key: undefined,
    ns: undefined,
    parent: undefined,
    raw: false,
    tag: "vue-component-1-app",
    text: undefined,
    child: undefined,
    asyncFactory: undefined,
    asyncMeta: undefined,
    children: undefined,
    componentInstance: undefined,
    data: {
        hook: {
            destroy: ƒ destroy(vnode),
            init: ƒ init(vnode, hydrating, parentElm, refElm),
            insert: ƒ insert(vnode),
            prepatch: ƒ prepatch(oldVnode, vnode),
        },
        on: undefined,
    },
    context: VueInstance,
    componentOptions: {
        Ctor: ƒ VueComponent(options),
        children: undefined,
        listeners: undefined,
        propsData: undefined,
        tag: "App",
    },
}
```

重点是`componentOptions`和`data`。

![Vue.prototype._render](./prototypeRender.png)

### vm._update

顾名思义，更新实例。

如果是初始化就在调用`patch`时传入不同的参数。如果是更新，就只传入`prevVnode`和`vnode`，如果父节点是`HOC`？就更新父节点的内容`（vm.$parent.$el = vm.$el）`。

![Vue.prototype._update](./prototypeUpdate.png)

还没有结束，重点在`patch`函数，该函数内会调用`createElm`，该函数又会调用`createComponent`，然后在函数内会取`vnode.data.hook.init`，然后调用这个`init`函数。

![createElm](./patchCreateElm.png)
![createComponent](./createComponent.png)

所以接下来要断点到`init`函数内看看了。

#### vnode.data.hook.init

位于`/core/vdom/create-component.js`文件内。

`init`钩子会调用`prepatch`钩子，而该钩子又调用`updateChildComponent`函数。

![vnode.data.hook.init](./initHook.png)

#### updateChildComponent

最后调用了`$forceUpdate`，而该方法只是调用了`vm._watcher.update()`方法。

![updateChildComponent](./updateChildComponent.png)
 
