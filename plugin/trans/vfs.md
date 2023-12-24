Virtual File System

主要目的是：

1 提供通用的 API，无论实际的文件在哪里存储；

2 记录文件修改历史，保留不同的版本；

3 在一个 VFS 文件中，提供一种持久化数据的可能性；

VFS 会用快照的方式，在每次用户请求一个文件时，将其从磁盘上加载到内存，并异步更新相关内容到磁盘上。但是这个快照是应用级别，而不是工程级别的。所以如果一个文件被多个工程打开了，只有一份内容会被存储在 VFS 中。

当通过 VFS 访问一个文件或者文件夹时，相关信息会被加载到快照中；否则不会加载实际内容，只会加载一些元信息，如时间戳、路径、属性等。这也就是为什么有时候一些被删除的文件会继续显示或者一些新文件不显示的原因。

VFS 会异步将更新通过 refresh operation 阶段刷到磁盘上。

更新可以同步，也可以异步；建议使用异步，同步可能有死锁问题。



Virtual File System Events

所有 VFS 中的变化，都会被当成是 virtual file system events。这些事件被分发线程(dispatch thread)和写操作触发。

为了监听 VFS 事件，可以实现 BulkFileListener ，并订阅 VirtualFileManager.VFS_CHANGES 主题。

VFS 事件在每个变化之前和之后发送，用户可以在事件之前获取到文件的旧内容。而当一个删除时间发生并写到磁盘上后，会触发一个 beforeFileDeletion 事件，而用户依然可以通过 VFS API 获取到删除时的文件内容，这估计就是可以立马在 idea 中恢复它的原因吧。

最后要注意的是，refresh operation 只会对快照中的文件发送更新事件。比如，用户访问一个文件夹，但是没有通过 VirtualFile.getChildren() 访问其内容，则用户不会接到该文件夹中创建了文件的通知。而对文件的操作同理。（这段说的不清不楚）



如何创建一个  Virtual File 对象

正常情况下，不需要自己创建。如果需要的话，可以使用 VirtualFile.createChildData() 创建一个 VirtualFile 实例，然后用 VirtualFile.setBinaryContent() 向该文件对象中写入数据。

