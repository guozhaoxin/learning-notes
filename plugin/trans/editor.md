CaretModel 类，用来获取一个个字符对象，而该对象又可以通过 Editor 对象获取到。

当一个 Document 对象被打开时，Editor 会生成一个基于 0 的坐标，记录每行每列的相关信息。