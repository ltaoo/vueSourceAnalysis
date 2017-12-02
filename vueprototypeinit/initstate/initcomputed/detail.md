

其实\`computed\`一直对我来说是很神奇的存在，比如这样的代码：



```javascript
new Vue({
	el: '#app',
	template: '<div><input v-model="name" />{{message}}</div>',
	data() {
		return {
			name: 'ltaoo',
			nickname: 'something',
		};
	},
	computed: {
		message() {
			return 'Hello ' + this.name; 
		},
	},
})
```

当改变输入框内的值时，对应的\`message\`也会改变。



我好奇的是，为什么知道\`name\`变化的时候要调用\`message\`获取新的值？



从上面的源码来看，显示实例化了一个\`watcher\`保存到\`vm.\_computedWatchers\`上，而\`Watcher\`构造函数内，将\`this\`存放到\`vm.watchers\`里面。



然后断点会到\`defineComputed\`里面，然后再到\`createComputedGetter\('message'\)\`这个函数，这个函数返回了一个函数\`computedGetter\`。



```javascript
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```



所以\`message\`这个\`key\`的\`getter\`实际上就是\`computedGetter\`这个函数了，在获取\`message\`的时候，就会调用它。（初始化时调用\`render\`也会触发这个函数的调用，以获取\`message\`值显示在页面上。）



额，我们知道，在\`initComputed\`内实例化的\`watcher\`存放在了\`vm.\_computedWatchers\`上，\`computedGetter\`函数内\`取出了这个\`watcher\`，判断是否\`dirty\`



\`dirty === true\`所以会调用\`evaluate\(\)\`，然后会调用\`message\`拿到值，但此时这个值就已经是\`ltaoo1\`了！



或者再看看这个组件吧：



```javascript
(function() {
    with (this) {
        return _c('div', [
            _c('input', {
                directives: [
                    {
                        name: 'model',
                        rawName: 'v-model',
                        value: name,
                        expression: 'name',
                    },
                ],
                domProps: { value: name },
                on: {
                    input: function($event) {
                        if ($event.target.composing) return;
                        name = $event.target.value;
                    },
                },
            }),
            _v(_s(message)),
        ]);
    }
});
```



\`with\`用以延长作用域链，所以\`\_c\`实际上是\`this.\_c\`，其他同理。

\`\_s === toString\`，\`\_v === createTextVNode\`，都和\`message\`没关系，还是不知道为什么\`message\`会改变。



其实是\`name\`改变后，会调用\`render\`重新计算\`vnode\`，导致的\`message\`改变？



但是虽然断点会到\`computedGetter\`这里，表示的确有去取\`message\`这个变量，但是在去的过程中，\`watcher.dirty === false\`，所以不调用\`message\`这个函数，导致\`message\`值并没有发生改变。



但为什么\`watcher.dirty === false\`而之前会是\`true\`呢？



因为调用了\`update\`，为什么\`watcher\`上的\`update\`会被调用呢？



因为\`name\`值发生了改变，调用了对应的\`setter\`，在\`setter\`内会调用\`dep.notify\`，该方法会遍历自身\`deps\`，调用\`dep.update\`，什么时候\`dep\`被加到\`name\`的\`deps\`上？



在第一次渲染\`vnode\`的时候，要获取到\`message\`的值，要会触发\`computedGetter\`，此时\`watcher.dirty === true\`（因为在初始化的时候\`dirty === lazy\`，而\`computed\`对应的\`watcher\`就是\`lazy\`），所以会调用\`evaluate\`，在这个方法内，调用了\`watcher.get\(\)\`，在\`get\`方法内，会将\`Dep.target = watcher\`，然后调用\`message\`函数，由于函数内要获取\`this.name\`，所以就会触发\`name\`的\`getter\`，而在\`getter\`内，判断\`Dep.target\`是否存在，如果存在，就将\`Dep.target\`放到闭包内的\`dep.subs\`上。



所以，就实现了依赖关系。



> 所以要尽量避免\`computed\`对应的函数内有无关的取值。



