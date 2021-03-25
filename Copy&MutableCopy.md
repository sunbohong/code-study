# Copy & MutableCopy
### copy&mutableCopy介绍
&emsp;&emsp;&emsp;&emsp;`copy`和`mutableCopy`，深浅拷贝，在OC里面是两个协议`NSCopying`和`NSMutableCopying`，分别对应的方法如下：
```Objective-C
@protocol NSCopying

- (id)copyWithZone:(nullable NSZone *)zone;

@end

@protocol NSMutableCopying

- (id)mutableCopyWithZone:(nullable NSZone *)zone;

@end
```
&emsp;&emsp;&emsp;&emsp;在`NSObject`根类里面类对象实现了两个协议方法，而实例对象没有实现，所以不能直接对`NSObject`的实例对象进行`copy`和`mutableCopy`
```
+ (id)copy {
    return (id)self;
}

+ (id)copyWithZone:(struct _NSZone *)zone {
    return (id)self;
}

- (id)copy {
    return [(id)self copyWithZone:nil];
}

+ (id)mutableCopy {
    return (id)self;
}

+ (id)mutableCopyWithZone:(struct _NSZone *)zone {
    return (id)self;
}

- (id)mutableCopy {
    return [(id)self mutableCopyWithZone:nil];
}

```
&emsp;&emsp;&emsp;&emsp;类对象的`copy`和`mutableCopy`都是返回自身，而实例对象的`copy`和`mutableCopy`是调用了自己的`NSCopying`和`NSMutableCopying`协议方法，根类并没有实现实例对象的协议方法，需要子类继承并重写。
所以这个协议方法，想怎么实现就怎么实现，我可以`copy`返回一个新对象的地址，`mutableCopy`返回自身，下面我们可以试一下
```
@interface Person : NSObject <NSCopying , NSMutableCopying>

@end

@implementation Person

- (id)copyWithZone:(nullable NSZone *)zone {
    return [Person new];
}

- (id)mutableCopyWithZone:(NSZone *)zone {
    return self;
}

@end

int main(int argc, const char * argv[]) {

    Person *person = [Person new];
    Person *copyPerson = [person copy];
    Person *mutablePerson = [person mutableCopy];
    NSLog(@"person:%p, copyPerson:%p, mutablePerson:%p", person, copyPerson, mutablePerson);
    
    return 0;
}


```
控制台输出
```
person:0x100518ac0, copyPerson:0x10050e230, mutablePerson:0x100518ac0
```
上面这个例子想说明什么，OC里面的深浅拷贝都是各个类自己实现的。所以像字符串，数组，字典，这三个对其进行`copy`和`mutableCopy`，都会发生什么，那就要看源码了
### 字符串
&emsp;&emsp;&emsp;&emsp;字符串`copy`和`mutableCopy`的实现，我们经常使用的是`NSString`和`NSMutableString`，虽然我们这样写，但是他真正的类是什么，我们看下`demo`
```
        NSString *tagString = [NSString stringWithFormat:@"%@", @"123"];
        /// 这种方式是NSTaggedPointerString
        NSString *tagString1 = [tagString copy];
        NSString *tagString2 = [tagString mutableCopy];
        NSLog(@"tagString-ptr:%p, tagString1-ptr:%p, tagString2-ptr:%p", tagString, tagString1, tagString2);
        
        
        NSString *string = @"fjsaoidjfoiadsjfoidsjoifjdsojfosdajfosfsjfasiojfaodsjfoisjdfjsaf";
        /// 这种方式是__NSCFConstantString
        NSString *string1 = [string copy];
        NSString *string2 = [string mutableCopy];
        NSLog(@"string-ptr:%p, string1-ptr:%p, string2-ptr:%p", string, string1, string2);
        
        NSMutableString *mutaString = [[NSMutableString alloc] initWithString:@"jfioadjsiofjosdajfoidsjaoifjdsoijfosdnfdasfdsjaoifjdoisajf"];
        /// 这种方式是__NSCFString
        NSString *mutaString1 = [mutaString copy];
        NSString *mutaString2 = [mutaString mutableCopy];
```
![](https://raw.githubusercontent.com/VenpleD/Image/master/stringDebugImage.png)
通过上图可以看到
声明了三种字符串，分别是`NSTaggedPointerString`、`__NSCFConstantString`、`__NSCFString`

#### NSTaggedPointerString
`NSTaggedPointerString`的`copy`如何实现
![](https://raw.githubusercontent.com/VenpleD/Image/master/20210325160219.png)
通过一步步运行，最终`NSTaggedPointerString`实现了`copy`协议方法，方法中没有做任何操作，把自己return出去
然后看`NSTaggedPointerString`的`mutableCopy`
通过一步步调用，发现`NSTaggedPointerString`类的父类是`NSString`，
```
Foundation`-[NSString mutableCopyWithZone:]:
->  0x7fff21378c3d <+0>:  pushq  %rbp
    0x7fff21378c3e <+1>:  movq   %rsp, %rbp
    0x7fff21378c41 <+4>:  pushq  %r14
    0x7fff21378c43 <+6>:  pushq  %rbx
    0x7fff21378c44 <+7>:  movq   %rdi, %rbx
    0x7fff21378c47 <+10>: movq   0x67877d62(%rip), %rdi    ; (void *)0x00007fff885d7ac0: NSMutableString // 找到NSMutableString
    0x7fff21378c4e <+17>: movq   0x6786d963(%rip), %rsi    ; "allocWithZone:" // 找到allocWithZone方法
    0x7fff21378c55 <+24>: movq   0x5f214a4c(%rip), %r14    ; (void *)0x00000001001b2400: objc_msgSend
    0x7fff21378c5c <+31>: callq  *%r14 //调用objc_msgSend
    0x7fff21378c5f <+34>: movq   0x6786e01a(%rip), %rsi    ; "initWithString:"
    0x7fff21378c66 <+41>: movq   %rax, %rdi // [NSMutableString allocWithZone]返回结果存入第一个参数
    0x7fff21378c69 <+44>: movq   %rbx, %rdx
    0x7fff21378c6c <+47>: movq   %r14, %rax
    0x7fff21378c6f <+50>: popq   %rbx
    0x7fff21378c70 <+51>: popq   %r14
    0x7fff21378c72 <+53>: popq   %rbp
    0x7fff21378c73 <+54>: jmpq   *%rax //  调用NSMutableString的initWithString方法
```
这里是直接调用了父类的协议方法生成了新的可变字符串返回出去的

#### __NSCFConstantString
`__NSCFConstantString`，常量字符串，程序加载的时候已经被放在了常量区
先看`__NSCFConstantString`的`copy`方法
![](https://raw.githubusercontent.com/VenpleD/Image/master/20210325160930.png)
`__NSCFConstantString`是通过重写`copy`方法来返回自身的，里面没有任何操作
下面是`__NSCFConstantString`的`mutableCopy`实现
```
CoreFoundation`-[__NSCFString mutableCopyWithZone:]:
->  0x7fff20636f0b <+0>:  movq   %rdi, %rdx
    0x7fff20636f0e <+3>:  leaq   0x15662b(%rip), %rax      ; kCFAllocatorDefault
    0x7fff20636f15 <+10>: movq   (%rax), %rdi
    0x7fff20636f18 <+13>: xorl   %esi, %esi
    0x7fff20636f1a <+15>: jmp    0x7fff2060eb54            ; CFStringCreateMutableCopy

```
通过一步步调用，发现`__NSCFConstantString`的父类是`__NSCFString`，自己并没有实现`NSMutableCopy`协议方法，所以这里就调用了父类的协议方法，可以看到是调用了`CFStringCreateMutableCopy`方法，来生成新的字符串，从CoreFoundation里面找源码，如下
```
CFMutableStringRef  CFStringCreateMutableCopy(CFAllocatorRef alloc, CFIndex maxLength, CFStringRef string) {
    CFMutableStringRef newString;

    //  CF_OBJC_FUNCDISPATCHV(__kCFStringTypeID, CFMutableStringRef, (NSString *)string, mutableCopy);

    __CFAssertIsString(string);

    newString = CFStringCreateMutable(alloc, maxLength);
    __CFStringReplace(newString, CFRangeMake(0, 0), string);

    return newString;
}

```

#### __NSCFString
下面来看NSMutableString的copy方法
![](https://raw.githubusercontent.com/VenpleD/Image/master/20210325161836.png)
通过上面可以看到，是调用了`_CFNonObjCStringCreateCopy`来生成一个新的字符串不可变对象返回出去的，源码这里就不探究了
再则就是NSMutableString的mutableCopy方法
![](https://raw.githubusercontent.com/VenpleD/Image/master/20210325162133.png)
通过上面可以看到，是调用`CFStringCreateMutableCopy`来生成一个新的可变字符串的
下面是输出结果
```
tagString-ptr:0xa1e755b9f4c57cb, tagString1-ptr:0xa1e755b9f4c57cb, tagString2-ptr:0x101504550
string-ptr:0x100004080, string1-ptr:0x100004080, string2-ptr:0x101404210
mutaString-ptr:0x101404330, mutaString1-ptr:0x101404440, mutaString2-ptr:0x101404590
```

|  | NSTaggedPointerString | __NSCFConstantString | __NSCFString |
| --- | --- | --- | --- |
| copy | 浅拷贝 | 浅拷贝 | 深拷贝 |
| mutableCopy | 深拷贝 | 深拷贝 | 深拷贝 |

### 数组

接下来看数组相关的`copy`和`mutableCopy`
下面是demo
```
    NSObject *obj1 = [NSObject new];
    NSObject *obj2 = [NSObject new];
    NSArray *array = @[obj1, obj2];
    NSArray *array1 = [array copy];
    NSArray *array2 = [array mutableCopy];
    NSLog(@"array_ptr:%p, array1_ptr:%p, array2_ptr:%p", array, array1, array2);
    NSLog(@"array_obj1:%p, array1_obj1:%p, array2_obj2:%p", array[0], array1[0], array2[0]);
    
    NSMutableArray *mutaArray = [NSMutableArray arrayWithArray:@[obj1, obj2]];
    NSArray *mutaArray1 = [mutaArray copy];
    NSMutableArray *mutaArray2 = [mutaArray mutableCopy];
    NSLog(@"mutaArray_ptr:%p, mutaArray1_ptr:%p, mutaArray2_ptr:%p", mutaArray, mutaArray1, mutaArray2);
    NSLog(@"mutaArray_obj1:%p, mutaArray1_obj1:%p, mutaArray2_obj2:%p", mutaArray[0], mutaArray1[0], mutaArray2[0]);
```
我们下面一步一步看看每种数组的`copy`和`mutableCopy`

#### NSArray
先看`NSArray`的继承关系，如下图，不知道为什么中间有两个__NSArrayI
![](https://raw.githubusercontent.com/VenpleD/Image/master/arrayDebugImage.png)
`[NSArray copy]`，调用的是父类的copy方法
```
CoreFoundation`-[__NSArrayI copy]:
->  0x7fff34132e78 <+0>:  jmp    0x7fff342884ac            ; symbol stub for: objc_retain // 就做了一次retain操作，返回retain的结果
    0x7fff34132e7d <+5>:  nop    
    0x7fff34132e7e <+6>:  nop    
    0x7fff34132e7f <+7>:  nop    
    0x7fff34132e80 <+8>:  nop    
    0x7fff34132e81 <+9>:  nop    
    0x7fff34132e82 <+10>: nop    
    0x7fff34132e83 <+11>: nop    

```
从上面源码可以看到`[NSArray copy]`返回是自身没有创建新的对象

`[NSArray mutableCopy]`,`__NSArrayI`的`mutableCopy`实现
```
CoreFoundation`-[__NSArrayI mutableCopy]:
->  0x7fff341364e2 <+0>:  mov    rax, qword ptr [rip + 0x579be62f] ; __NSArrayI.storage
    0x7fff341364e9 <+7>:  add    rdi, rax
    0x7fff341364ec <+10>: add    rdi, 0x8
    0x7fff341364f0 <+14>: mov    rsi, qword ptr [rdi - 0x8]
    0x7fff341364f4 <+18>: xor    edx, edx
    0x7fff341364f6 <+20>: jmp    0x7fff340ea979            ; __NSArrayM_new // 上面获取数组的容量，然后调用__NSArrayM_new生成新的可变对象
    0x7fff341364fb <+25>: nop    
    0x7fff341364fc <+26>: nop    
    0x7fff341364fd <+27>: nop    
    0x7fff341364fe <+28>: nop    
```
从源码来看，`[NSArray mutableCopy]`返回的是新的对象，并且这个对象是可变数组
#### NSMutableArray
下面是`NSMutableArray`的继承关系图
![](https://raw.githubusercontent.com/VenpleD/Image/master/NSMutableArray.png)

`[NSMutableArray copy]`,`__NSArrayM`的`copy`实现
```
CoreFoundation`-[__NSArrayM copy]:
->  0x7fff340f92ac <+0>:  push   rbp
    0x7fff340f92ad <+1>:  mov    rbp, rsp
    0x7fff340f92b0 <+4>:  push   rbx
    0x7fff340f92b1 <+5>:  push   rax
    0x7fff340f92b2 <+6>:  mov    rbx, rdi
    0x7fff340f92b5 <+9>:  lea    rax, [rip + 0x57a097fc]   ; __cf_tsanReadFunction
    0x7fff340f92bc <+16>: mov    rax, qword ptr [rax]
    0x7fff340f92bf <+19>: test   rax, rax
    0x7fff340f92c2 <+22>: jne    0x7fff340f92d2            ; <+38>
    0x7fff340f92c4 <+24>: mov    rdi, rbx
    0x7fff340f92c7 <+27>: add    rsp, 0x8
    0x7fff340f92cb <+31>: pop    rbx
    0x7fff340f92cc <+32>: pop    rbp
    0x7fff340f92cd <+33>: jmp    0x7fff340f92e7            ; __NSArrayM_copy //这里调用copy方法
    0x7fff340f92d2 <+38>: mov    rsi, qword ptr [rbp + 0x8]
    0x7fff340f92d6 <+42>: lea    rcx, [rip + 0x57a097eb]   ; __CFTSANTagMutableArray
    0x7fff340f92dd <+49>: mov    rdx, qword ptr [rcx]
    0x7fff340f92e0 <+52>: mov    rdi, rbx
    0x7fff340f92e3 <+55>: call   rax
    0x7fff340f92e5 <+57>: jmp    0x7fff340f92c4            ; <+24>
```
下面是`__NSArrayM_copy`的实现，只截取了一部分，最后是调用了`__NSPlaceholderArray`的`initWithArray:range:copyItems:`，但是我一步一步走进去，调到了`[NSArray initWithArray:range:copyItems:]`里面，不知道为什么，这里也就返回的是__NSArrayI对象，不可变数组
![](https://raw.githubusercontent.com/VenpleD/Image/master/mutableArrayCopyImage.png)

下面是`[__NSArrayM mutableCopy]`方法实现，里面貌似用到了copyOnWrite，但是没有找到`_cow_copy`源码，也没有细看里面的逻辑，但是通过结果来看，是创建了新的`__NSArrayM`对象
![](https://raw.githubusercontent.com/VenpleD/Image/master/__NSArrayMMutableCopy.png)

**从上面源码来看，整个数组的copy和mutableCopy都没有涉及到数组存储对象的创建，只是增加了对原数组中对象的引用，**

下面是运行结果
```
2021-03-23 21:59:45.809618+0800 KCObjcBuild[25281:1136156] array_ptr:0x101041310, array1_ptr:0x101041310, array2_ptr:0x101041510
2021-03-23 21:59:45.809748+0800 KCObjcBuild[25281:1136156] array_obj1:0x101005cd0, array1_obj1:0x101005cd0, array2_obj2:0x101005cd0
2021-03-23 21:59:45.809881+0800 KCObjcBuild[25281:1136156] mutaArray_ptr:0x101343cb0, mutaArray1_ptr:0x101343c40, mutaArray2_ptr:0x101342e10
2021-03-23 21:59:45.809982+0800 KCObjcBuild[25281:1136156] mutaArray_obj1:0x101005cd0, mutaArray1_obj1:0x101005cd0, mutaArray2_obj2:0x101005cd0
```

|  | NSArray | NSMutableArray | NSString | NSMutableString |
| --- | --- | --- | --- | --- |
| copy | 浅拷贝（返回自身） | 深拷贝（返回创建的NSArray） | 浅拷贝（返回自身） | 深拷贝（返回创建的__NSCFString） |
| mutableCopy | 深拷贝（返回创建的NSMutableArray） | 深拷贝（返回创建NSMutableArray） | 深拷贝（返回创建的__NSCFString） | 深拷贝（返回创建的__NSCFString） |


个人感觉这些都是系统实现好的方法，记一下就可以了，真的像最开始说的，你可以重写这些方法，想怎么实现就怎么实现，想new新对象，就new
