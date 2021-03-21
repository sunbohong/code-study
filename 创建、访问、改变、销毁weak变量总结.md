## 创建、访问、改变、销毁 weak 变量总结

#### 我们都知道:

- weak 变量对指向的对象是弱引用, 不会使对象的引用计数+1

- weak 变量指向的对象销毁时指针会被置为 nil

- weak 变量的内存管理是依赖 SideTable(s)、weak_table_t、weak_entry_t
  - SideTables 是静态全局哈希表, 容量固定, iOS 中有 8 张 SideTable
  - 通过对象地址一层一层找, weak 变量的地址, 也就是指针的指针, 存在容量为 4 的 inline_referrers 数组或者 referrers 哈希数组中, 根据数量而定(这里让我联想到 JDK1.8 的 Hash 冲突时的链表+红黑树的切换, 而且我发现苹果喜欢开放定址法, Java 喜欢链地址法), 创建 weak 变量、销毁 weak 变量、销毁对象时都会对这里进行操作

#### 结合 weak 变量的创建、销毁过程我们来串一下这些知识点

```objective-c
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSObject *obj1 = [NSObject new];
        NSObject *obj2 = [NSObject new];
        {
            // 初始化weak变量
            __weak id weak_obj = obj1;
            // 访问weak变量
            NSLog(@"%@", weak_obj);
            // 改变weak变量的指向
            weak_obj = obj2;
            // 作用域结束weak变量销毁
        }
        NSLog(@"断点打在这");
    }
    return 0;
}
```

汇编如下:

![](https://tva1.sinaimg.cn/large/e6c9d24ely1gol000a0uqj20r10dlq6e.jpg)

##### 先看初始化 weak 变量、改变 weak 变量的指向和销毁 weak 变量这三个函数:

```objective-c
id objc_initWeak(id *location, id newObj)
{
    if (!newObj) {
        *location = nil;
        return nil;
    }
    return storeWeak<DontHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object*)newObj);
}
```

```objective-c
id objc_storeWeak(id *location, id newObj)
{
    return storeWeak<DoHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object *)newObj);
}
```

```objective-c
void objc_destroyWeak(id *location)
{
    (void)storeWeak<DoHaveOld, DontHaveNew, DontCrashIfDeallocating>
        (location, nil);
}
```

三个函数都调用了同一个函数:

```objective-c
template <HaveOld haveOld, HaveNew haveNew,
          enum CrashIfDeallocating crashIfDeallocating>
static id
storeWeak(id *location, objc_object *newObj)
{
...
}
```

storeWeak 是一个模版函数, haveOld 参数表示是否有旧值, haveNew 参数表示是否有新值, 指的是 weak 变量的指向. 那就好理解了, 初始化 weak 变量的时候没有旧值有新值, 改变 weak 变量指向的时候既有旧值又有新值, 销毁 weak 变量的时候有旧值没有新值.

截取 storeWeak 函数其中两段主要代码:

```objective-c
// 如果有旧值
if (haveOld) {
    // 把weak变量的地址从它之前指向的对象的weak_entry_t的定长数组或哈希数组中移除
    weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
}
```

```objective-c
// 如果有新值
if (haveNew) {
    newObj = (objc_object *)
    // 把对象和weak变量的指针注册到weak_table_t的weak_entry_t里
    weak_register_no_lock(&newTable->weak_table, (id)newObj, location, crashIfDeallocating ? CrashIfDeallocating : ReturnNilIfDeallocating);

    // 优化过的isa指针的weakly_referenced会被标记为1
    if (!newObj->isTaggedPointerOrNil()) {
        newObj->setWeaklyReferenced_nolock();
    }

    *location = (id)newObj;
}
else {
    // No new value. The storage is not changed.
}
```

总结:

| 函数             | 调用              | haveOld:<br />weak_unregister_no_lock | haveNew:<br />weak_register_no_lock |
| ---------------- | ----------------- | ------------------------------------- | ----------------------------------- |
| objc_initWeak    | weak 变量初始化   | false                                 | true                                |
| objc_storeWeak   | weak 变量指向改变 | true                                  | true                                |
| objc_destroyWeak | weak 变量销毁     | true                                  | false                               |

> objc_destroyWeak 函数是销毁 weak 变量时调用的, 而在对象的 dealloc 时会调用一系列函数:
>
> dealloc->\_objc_rootDealloc->rootDealloc->object_dispose->objc_destructInstance->clearDeallocating->clearDeallocating_slow->weak_clear_no_lock
>
> 最后的 weak_clear_no_lock 函数会遍历 weak_entry_t 的定长数组或哈希数组把所有指向该对象的 weak 指针置为 nil, weak_entry_t 也会被移除
>
> ![](https://tva1.sinaimg.cn/large/e6c9d24egy1golr62vceqj207g09zq3d.jpg)

##### 再来看一下改变 weak 变量的指向时调用的函数 objc_loadWeakRetained 和\_objc_release:

截取 objc_loadWeakRetained 函数中的一段代码:

```objective-c
if (! obj->rootTryRetain()) {
    result = nil;
}
```

rootTryRetain()函数内部会对对象做一次 retain 操作, 而这个操作会被后面调用的\_objc_release 抵消掉, 所以对象的引用计数不会变

##### 最后看一下 MRC 下的汇编

![](https://tva1.sinaimg.cn/large/e6c9d24egy1gol00zk275j20qt0cfact.jpg)

发现访问 weak 变量跟 ARC 下是不同的, 看一下 objc_loadWeak 函数:

```objective-c
id objc_loadWeak(id *location)
{
    if (!*location) return nil;
    return objc_autorelease(objc_loadWeakRetained(location));
}
```

MRC 下 objc_loadWeakRetained 函数的 retain 操作是由添加到 autoreleasepool 抵消的

> 这也解释了《Objective-C 高级编程 iOS 与 OS X 多线程和内存管理》这本书 1.4.2 节提到的: 使用附有\_\_weak 修饰符的变量, 即是使用注册到 autoreleasepool 中的对象
>
> ![](https://tva1.sinaimg.cn/large/e6c9d24ely1gol2961gxnj20ra077mxw.jpg)

简单证明一下:

ARC 下

![](https://tva1.sinaimg.cn/large/e6c9d24egy1gol01jurelj20gt0a30ty.jpg)

MRC 下

![](https://tva1.sinaimg.cn/large/e6c9d24egy1gol01t48zdj20h60ajaba.jpg)
