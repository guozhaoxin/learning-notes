## GC 复制算法

GC 复制算法会将整个堆分成完全相等的两部分，在分配空间时只使用其中的一半，即 From 区域；当无法分配空间时，会将 From 空间中的活动对象复制到 To 区域中；复制完毕后，交换 From 和 To 区域的指针。

伪代码如下：

```c
void copying(){
  $free = $to_start
  for(r : $roots)
    *r = copy(*r)
  swap($from_start,$to_start)
}

void copy(obj){
if(obj.tag != COPIED)
	copy_data($free, obj, obj.size) 
  obj.tag = COPIED
	obj.forwarding = $free
	$free += obj.size
	for(child : children(obj.forwarding)) 
    *child = copy(*child)
return obj.forwarding 
}
```

首先，from_start end_start 都是全局指针，分别指向两个半堆的其实位置；

其次，每个对象有两个额外的字段，一个是标记对象是否已经被遍历过的标签 tag，一个是指示对象在 to 区域中的位置；

### GC 复制过程

假设初始状态如下：

![gc-copy-1](./images/gc-copy-1.jpg)

初始时， from 堆上有 6 个对象，其中 E 和 F 是垃圾；而 B 和 G 可以通过根直接访问到。

#### 复制 B 对象

首先通过根访问到 B 对象，并将 B 复制到 to 空间中，复制完毕后，整个状态如下：

![gc-copy-2](./images/gc-copy-2.jpg)

可以看到，通过 B 复制出 B‘，根从指向 B 转而指向 B’，B 的 tag 被标记，表示已经复制过，而 B 的forward 字段指向 B’；但是复制出来的 B‘，依然指向 from 空间中的 A；要注意的是，此时的 free 指针已经从 to 的开始移动到了 B‘ 对象的下一个位置；

#### 复制 A 对象

在复制完 B 后，开始对 B 能引用的所有对象进行遍历和复制操作，B 只引用了 A，因此将 A 复制到 to 空间中，成为 A’，状态如下：

![gc-copy-3](./images/gc-copy-3.jpg)

复制 A 和复制 B 很类似，只是 A 没有引用任何对象，因此复制完 A 后，就直接返回了；

