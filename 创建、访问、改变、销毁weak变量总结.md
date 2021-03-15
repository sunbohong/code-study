## 创建、访问、改变、销毁weak变量总结

#### 我们都知道:

* weak变量对指向的对象是弱引用, 不会使对象的引用计数+1
* weak变量指向的对象销毁时会被置为nil
* weak变量的内存管理是依赖SideTable(s)、weak_table_t、weak_entry_t
    * SideTables是静态全局哈希表, 容量固定, iOS中有8张SideTable
    * 通过对象地址一层一层找, weak变量的地址, 也就是指针的指针, 存在容量为4的inline_referrers数组或者referrers哈希数组中, 根据数量而定(这里让我联想到JDK1.8的Hash冲突时的链表+红黑树的切换, 而且苹果喜欢开放定址法, Java喜欢链地址法), 创建weak变量、销毁weak变量、销毁对象时都会对这里进行操作

#### 结合weak变量的创建、销毁过程我们来串一下这些知识点
```
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
![d31e130a8db7773c743ad94f84c3abb1](创建、访问、改变、销毁weak变量总结.resources/767E36E6-3D54-4B67-9E7D-3A8E54583708.png)
##### 先看初始化weak变量、改变weak变量的指向和销毁weak变量这三个函数:
```
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
```
id objc_storeWeak(id *location, id newObj)
{
    return storeWeak<DoHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object *)newObj);
}
```

```
void objc_destroyWeak(id *location)
{
    (void)storeWeak<DoHaveOld, DontHaveNew, DontCrashIfDeallocating>
        (location, nil);
}
```
三个函数都调用了同一个函数:
```
template <HaveOld haveOld, HaveNew haveNew,
          enum CrashIfDeallocating crashIfDeallocating>
static id 
storeWeak(id *location, objc_object *newObj)
{
...
}
```
storeWeak是一个模版函数, haveOld参数表示是否有旧值, haveNew参数表示是否有新值, 指的是weak变量的指向. 那就好理解了, 初始化weak变量的时候没有旧值有新值, 改变weak变量指向的时候既有旧值又有新值, 销毁weak变量的时候有旧值没有新值. 
截取其中两段主要代码:
```
    // 如果有旧值
    if (haveOld) {
        // 把weak变量的地址从它之前指向的对象的weak_entry_t的定长数组或哈希数组中移除
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }
```
```
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
> objc_destroyWeak函数是销毁weak变量时调用的, 而在对象的dealloc时会调用一系列函数:
> dealloc->_objc_rootDealloc->rootDealloc->object_dispose->objc_destructInstance->clearDeallocating->clearDeallocating_slow->weak_clear_no_lock
> 最后的weak_clear_no_lock函数会遍历weak_entry_t的定长数组或哈希数组把所有指向该对象的weak指针置为nil, weak_entry_t也会被移除

##### 再来看一下改变weak变量的指向时调用的函数objc_loadWeakRetained和_objc_release:

截取objc_loadWeakRetained函数中的一段代码:
```
    if (! obj->rootTryRetain()) {
        result = nil;
    }
```
rootTryRetain()函数内部会做一次retain操作, 而这个操作会被后面调用的_objc_release抵消掉
##### 最后看一下MRC下的汇编
![9acbb591ce8a58c04507c212613c6274](创建、访问、改变、销毁weak变量总结.resources/635209B6-955B-4AF4-879D-9C9815FD3D66.png)
发现访问weak变量跟ARC下是不同的, 看一下objc_loadWeak函数:
```
id objc_loadWeak(id *location)
{
    if (!*location) return nil;
    return objc_autorelease(objc_loadWeakRetained(location));
}
```
MRC下objc_loadWeakRetained函数的retain操作是由添加到autoreleasepool抵消的
> 这也证明了《Objective-C高级编程iOS与OS X多线程和内存管理》这本书1.4.2节提到的: 使用附有__weak修饰符的变量, 即是使用注册到autoreleasepool中的对象

简单证明一下:
// ARC
![1089d9e51089086ef6ca9955c08c4579](创建、访问、改变、销毁weak变量总结.resources/4204F0CB-B38E-4505-A600-A737427CB0EC.png)
// MRC
![b9f84550a437b7fc07d732ce930f0bfb](创建、访问、改变、销毁weak变量总结.resources/C0A033F4-7B09-49C6-BCAE-622EE4A89B09.png)



