Document

所谓 Document 对象，是一段 unicode 序列，对应文件中的内容。一个 document 中行的分隔符依然是 \n。

Document 貌似是个弱引用，很容易被 GC 掉，在需要时再被重建。