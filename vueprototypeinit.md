## Vue.prototype.\_init



![](/assets/init.png)

流程图看起来很长，实际代码不多，可以看到出现了三次判断是否为生产环境，为了简化理解可以只看右边的条件分支。

做的事情也很简单，给`vm`即实例添加了多个属性，这里用红色做了标记。然后调用了多个`init`开头的函数：



* initLifecycle
* initEvents
* initRender
* initInjections
* initProvide

在调用这些函数过程中还调用了两个生命周期钩子`beforeCreate`和`created`。这就表示在`initRender`函数后，就开始`create`，直到`initProvide`调用了，完成了`create`变成了`created`，那在`Vue`中，`create`表示创建什么呢，继续往下看。



> 在构造函数内调用\_init方法，看起来很普通，很正常。但《JavaScript设计模式与开发实践》在_“模板方法模式”_一节，将`Vue.prototype._init`称为“模板方法” 。



