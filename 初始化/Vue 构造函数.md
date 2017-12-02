## Vue 构造函数

当然，只要写过`js`的人都能看出来，上面的代码调用了`Vue`构造函数，实例化了一个实例。所以第一步就是查看构造函数内做了什么。

这是第一个难点，`codepen`使用了`vue.min.js`，而使用`vue-cli`初始化的`webpack`项目使用了`vue/dist/vue.esm.js`，但我们想看的是源码，前两者显然都是打包后的文件。所以在`main.js`文件内，不是`import Vue from 'vue'`，而是

```javascript
import Vue from './lib/vue/platforms/web/entry-runtime-with-compiler'
```

当然，我要先展示下我的目录结构：

```bash
.
├── build
├── config
├── src
│   ├── assets
│   ├── components
│   ├── main.js        <-------------------- 入口文件在这
│   └── lib
│       └── vue
│           ├── compiler
│           │   ├── codegen
│           │   ├── directives
│           │   └── parser
│           ├── core
│           │   ├── components
│           │   ├── global-api
│           │   ├── instance
│           │   │   └── render-helpers
│           │   ├── observer
│           │   ├── util
│           │   └── vdom
│           │       ├── helpers
│           │       └── modules
│           ├── platforms
│           │   ├── web
│           │   │   ├── compiler
│           │   │   │   ├── directives
│           │   │   │   └── modules
│           │   │   ├── runtime
│           │   │   │   ├── components
│           │   │   │   ├── directives
│           │   │   │   └── modules
│           │   │   ├── server
│           │   │   │   ├── directives
│           │   │   │   └── modules
│           │   │   └── util
│           │   └── weex
│           │       ├── compiler
│           │       │   ├── directives
│           │       │   └── modules
│           │       ├── runtime
│           │       │   ├── components
│           │       │   ├── directives
│           │       │   └── modules
│           │       └── util
│           ├── server
│           │   ├── bundle-renderer
│           │   ├── optimizing-compiler
│           │   ├── template-renderer
│           │   └── webpack-plugin
│           ├── sfc
│           └── shared
└── static
```

在`main.js`文件同级，我增加了`lib`文件，并将`vue`整个源码拷贝到了这里，额，记得还有一些文件依赖关系要修改一下，修改`build/webpack.base.conf.js`文件，主要是增加了`alias`后面五行内容：

```javascript
resolve: {
  extensions: ['.js', '.vue', '.json'],
  alias: {
    'vue$': 'vue/dist/vue.esm.js',
    '@': resolve('src'),
    'core': resolve('src/lib/vue/core'),
    'shared': resolve('src/lib/vue/shared'),
    'sfc': resolve('src/lib/vue/sfc'),
    'web': resolve('src/lib/vue/platforms/web'),
    'compiler': resolve('src/lib/vue/compiler'),
  }
},
```

幸运的话`npm run dev`就能够跑起项目。

> 其实我后面意识到应该使用 [https://github.com/vuejs/vue ](https://github.com/vuejs/vue) 这个仓库而不是自己初始化一个。



如果从`entry-runtime-with-compiler`这个文件看起，可以看到从`./runtime/index`导入`Vue`，然后进行一些处理，而查看`./runtime/index`发现`Vue`又是从`core/index`导入的。

其实从文件目录就能看出大概，无论是`entry-runtime-with-compiler`还是`./runtime/index`都是位于`web`这个目录，表示这里的文件都是和`web`平台相关的，对`Vue`所做的处理也是和`web`平台相关的；与之相对的`core`部分，就是平台无关的。



所以先看这部分，打开`core/index`可以看到`Vue`来自`instance/index`。



![](/assets/constructor.png)`process.env.NODE_ENV !== 'production'`会在后面大量出现，该代码是用来判断当前是否为生产环境（即线上环境），反之则是开发环境或者测试环境，当然大部分情况都是开发环境。指向`yes`表示就是我们的开发环境，所以会有`warn`，即控制台报错。



在函数最后，调用了`this._init`方法。















