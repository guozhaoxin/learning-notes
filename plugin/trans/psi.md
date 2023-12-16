PSI：Program Structure Interface。

PSI 文件是种根结构，代表了某一特定语言类型的内容。

所有 PSI 类的基类都是 PsiFile，这是一个 interface。PSI 对象的可见范围是工程级别的，同一个文件，如果在多个窗口中同时打开，会共享相同的 Document 和 VritualFile 实例，但是每个工程都有自己的一个 PSI 对象。

有多种获取 PSI 对象的方法：

https://plugins.jetbrains.com/docs/intellij/psi-files.html#how-do-i-get-a-psi-file