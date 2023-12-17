Actions

所有继承了 AnAction 的类，这同样是一个抽象类，子类需要实现它的 actionPerformed() 方法，它本身会接受一个 AnActionEvent 对象；同时建议重载 update()方法；如果使用的是2022.3 版本后的idea，getActionUpdateThread 方法也必须重载。当一个注册的 action，其菜单项被选中时，对应类的 actionPerformed 就会执行。



Extensions

有点类似于 Actions，但它会把注册的类放置在侧边上“

com.intellij.toolWindow 扩展，允许插件注册在工具窗口上；

com.intellij.applicationConfigurable 及 com.intellij.projectConfigurable 扩展，允许插件在设置对话框；

自定义语言插件，使用各种扩展点来支持对特定语言的支持；



如何声明 Extensions

https://plugins.jetbrains.com/docs/intellij/plugin-extensions.html#declaring-extensions



Services

services 好像也是插件，当插件代码调用对应 ComponentManager 实例的 getServices() 方法时。平台会保证一个 service 永远都只有一个实例被导入。

一个 service 如果需要有钩子，则可以实现 Disposable  类，实现其中的 dispose() 方法。

有三种 services，应用级别的、工程级别的、模块级别的，其中模块级别的是最不推荐的，因为容易造成占用大量内存。

