# Copy & MutableCopy
`copy`和`mutableCopy`，深浅拷贝，在OC里面是两个协议方法`NSCopying`和`NSMutableCopying`，分别对应的方法如下：
```Objective-C
@protocol NSCopying

- (id)copyWithZone:(nullable NSZone *)zone;

@end

@protocol NSMutableCopying

- (id)mutableCopyWithZone:(nullable NSZone *)zone;

@end
```
在`NSObject`根类里面类对象实现了两个协议方法，而对象没有实现，所以不能直接对对象进行copy和mutableCopy
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
类对象的`copy`和`mutableCopy`都是返回自身，而实例对象的`copy`和`mutableCopy`是调用了自己的`NSCopying`和`NSMutableCopying`协议方法，并且根类并没有实现实例对象的协议方法，所以需要子类继承并重写。
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
上面这个例子想说明什么，OC里面的深浅拷贝都是各个类自己实现的，所以像字符串，数组，字典，这三个对其进行`copy`和`mutableCopy`，都会发生什么，那就要看源码了
第一个例子，字符串`copy`和`mutableCopy`的实现
对外开放的经常别使用的是`NSString`和`NSMutableString`
虽然我们这样写，但是他真正的类是什么，我们看下
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

声明了三种字符串，分别是`NSTaggedPointerString`、`__NSCFConstantString`、`__NSCFString`，打印结果
```
tagString-ptr:0xa1e755b9f4c57cb, tagString1-ptr:0xa1e755b9f4c57cb, tagString2-ptr:0x101504550
string-ptr:0x100004080, string1-ptr:0x100004080, string2-ptr:0x101404210
mutaString-ptr:0x101404330, mutaString1-ptr:0x101404440, mutaString2-ptr:0x101404590
```
`NSTaggedPointerString`，优化后的指针，`copy`返回自己，`mutableCopy`返回的其实是`__NSCFString`
`__NSCFConstantString`，常量字符串，程序以加载的时候就存在了，他们的`mutableCopy`也是`__NSCFstring`
下面是`NSTaggedPointerString`类型字符串调用`mutableCopy`的汇编码（没找到实现源码），`NSTaggedPointerString`类的父类是`NSString`，
```
Foundation`-[NSString mutableCopyWithZone:]:
->  0x7fff21378c3d <+0>:  pushq  %rbp
    0x7fff21378c3e <+1>:  movq   %rsp, %rbp
    0x7fff21378c41 <+4>:  pushq  %r14
    0x7fff21378c43 <+6>:  pushq  %rbx
    0x7fff21378c44 <+7>:  movq   %rdi, %rbx
    0x7fff21378c47 <+10>: movq   0x67877d62(%rip), %rdi    ; (void *)0x00007fff885d7ac0: NSMutableString
    0x7fff21378c4e <+17>: movq   0x6786d963(%rip), %rsi    ; "allocWithZone:"
    0x7fff21378c55 <+24>: movq   0x5f214a4c(%rip), %r14    ; (void *)0x00000001001b2400: objc_msgSend
    0x7fff21378c5c <+31>: callq  *%r14
    0x7fff21378c5f <+34>: movq   0x6786e01a(%rip), %rsi    ; "initWithString:"
    0x7fff21378c66 <+41>: movq   %rax, %rdi
    0x7fff21378c69 <+44>: movq   %rbx, %rdx
    0x7fff21378c6c <+47>: movq   %r14, %rax
    0x7fff21378c6f <+50>: popq   %rbx
    0x7fff21378c70 <+51>: popq   %r14
    0x7fff21378c72 <+53>: popq   %rbp
    0x7fff21378c73 <+54>: jmpq   *%rax
```
这里面大概的意思就是生成了`NSMutableString`，返回出去的，而`NSMutableString`的父类就是`__NSCFstring`
`__NSCFConstantString`的父类是`__NSCFString`，自己没有实现`NSMutableCopy`协议方法，所以对此类型字符串调用`mutableCopy`，直接就是父类`__NSCFString`的实现，下面是汇编码
```
CoreFoundation`-[__NSCFString mutableCopyWithZone:]:
->  0x7fff20636f0b <+0>:  movq   %rdi, %rdx
    0x7fff20636f0e <+3>:  leaq   0x15662b(%rip), %rax      ; kCFAllocatorDefault
    0x7fff20636f15 <+10>: movq   (%rax), %rdi
    0x7fff20636f18 <+13>: xorl   %esi, %esi
    0x7fff20636f1a <+15>: jmp    0x7fff2060eb54            ; CFStringCreateMutableCopy

```
可以看到是调用了`CFStringCreateMutableCopy`方法，从CoreFoundation里面找源码
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
生成了，新的对象，并且返回了
上面都是`mutableCopy`协议方法的调用实现，也就是子类如果不实现，就调用父类
同样，`copy`协议方法也一样，子类不实现，就调用父类，但是`NSTaggedPointerString`实现了`copy`协议方法，返回自身，`__NSCFConstantString`重写了`copy`方法，返回自身，所以就没有调用父类`__NSCFString`的copy协议方法，所以我们`NSMutableString`的`copy`就是生成了新的对象，只不过子类重写了，没有调到这里
下面是`__NSCFConstantString`的`copy`方法
```
CoreFoundation`-[__NSCFConstantString copy]:
->  0x7fff205cc722 <+0>: movq   %rdi, %rax /// rdi寄存器放的是第一个参数，也就是self，然后移动到rax，rax是返回寄存器，也就是调用此方法的返回值
    0x7fff205cc725 <+3>: retq 
```
下面是`NSTaggedPointerString`的`copy`方法，也是直接返回自身
```
CoreFoundation`-[NSTaggedPointerString copyWithZone:]:
->  0x7fff206056c6 <+0>: movq   %rdi, %rax /// rdi寄存器放的是第一个参数，也就是self，然后移动到rax，rax是返回寄存器，也就是调用此方法的返回值
    0x7fff206056c9 <+3>: retq   
    0x7fff206056ca <+4>: nop    
    0x7fff206056cb <+5>: nop    

```
接下来看数组相关的`copy`和`mutableCopy`
```
    NSObject *obj1 = [NSObject new];
    NSObject *obj2 = [NSObject new];
    NSArray *array = @[obj1, obj2];
    NSArray *array1 = [array copy];
    NSArray *array2 = [array1 mutableCopy];
    NSLog(@"array_ptr:%p, array1_ptr:%p, array2_ptr:%p", array, array1, array2);
    NSLog(@"array_obj1:%p, array1_obj1:%p, array2_obj2:%p", array[0], array1[0], array2[0]);
    
    NSMutableArray *mutaArray = [NSMutableArray arrayWithArray:@[obj1, obj2]];
    NSArray *mutaArray1 = [mutaArray copy];
    NSMutableArray *mutaArray2 = [mutaArray mutableCopy];
    NSLog(@"mutaArray_ptr:%p, mutaArray1_ptr:%p, mutaArray2_ptr:%p", mutaArray, mutaArray1, mutaArray2);
    NSLog(@"mutaArray_obj1:%p, mutaArray1_obj1:%p, mutaArray2_obj2:%p", mutaArray[0], mutaArray1[0], mutaArray2[0]);
```
先看结果
```
2021-03-23 21:59:45.809618+0800 KCObjcBuild[25281:1136156] array_ptr:0x101041310, array1_ptr:0x101041310, array2_ptr:0x101041510
2021-03-23 21:59:45.809748+0800 KCObjcBuild[25281:1136156] array_obj1:0x101005cd0, array1_obj1:0x101005cd0, array2_obj2:0x101005cd0
2021-03-23 21:59:45.809881+0800 KCObjcBuild[25281:1136156] mutaArray_ptr:0x101343cb0, mutaArray1_ptr:0x101343c40, mutaArray2_ptr:0x101342e10
2021-03-23 21:59:45.809982+0800 KCObjcBuild[25281:1136156] mutaArray_obj1:0x101005cd0, mutaArray1_obj1:0x101005cd0, mutaArray2_obj2:0x101005cd0
```
从结果来看，`NSArray`的`copy`是浅拷贝，`mutableCopy`是深拷贝，`NSMutableArray`的`copy`是深拷贝，结果是不可变数组，`NSMutableArray`的`mutableCopy`是深拷贝，返回的是可变数组
下面是`NSArray`父类`__NSArrayI`的`copy`实现
```
CoreFoundation`-[__NSArrayI copy]:
->  0x7fff34132e78 <+0>:  jmp    0x7fff342884ac            ; symbol stub for: objc_retain
    0x7fff34132e7d <+5>:  nop    
    0x7fff34132e7e <+6>:  nop    
    0x7fff34132e7f <+7>:  nop    
    0x7fff34132e80 <+8>:  nop    
    0x7fff34132e81 <+9>:  nop    
    0x7fff34132e82 <+10>: nop    
    0x7fff34132e83 <+11>: nop    

```

下面是`NSArray`父类`__NSArrayI`的`mutableCopy`实现
```
CoreFoundation`-[__NSArrayI mutableCopy]:
->  0x7fff341364e2 <+0>:  mov    rax, qword ptr [rip + 0x579be62f] ; __NSArrayI.storage
    0x7fff341364e9 <+7>:  add    rdi, rax
    0x7fff341364ec <+10>: add    rdi, 0x8
    0x7fff341364f0 <+14>: mov    rsi, qword ptr [rdi - 0x8]
    0x7fff341364f4 <+18>: xor    edx, edx
    0x7fff341364f6 <+20>: jmp    0x7fff340ea979            ; __NSArrayM_new
    0x7fff341364fb <+25>: nop    
    0x7fff341364fc <+26>: nop    
    0x7fff341364fd <+27>: nop    
    0x7fff341364fe <+28>: nop    
```

下面是`NSMutableArray`的父类`__NSArrayM`的`copy`实现
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
    0x7fff340f92cd <+33>: jmp    0x7fff340f92e7            ; __NSArrayM_copy
    0x7fff340f92d2 <+38>: mov    rsi, qword ptr [rbp + 0x8]
    0x7fff340f92d6 <+42>: lea    rcx, [rip + 0x57a097eb]   ; __CFTSANTagMutableArray
    0x7fff340f92dd <+49>: mov    rdx, qword ptr [rcx]
    0x7fff340f92e0 <+52>: mov    rdi, rbx
    0x7fff340f92e3 <+55>: call   rax
    0x7fff340f92e5 <+57>: jmp    0x7fff340f92c4            ; <+24>
```
个人感觉这些都是系统实现好的方法，记一下就可以了，真的像最开始说的，你可以重写这些方法，想怎么实现就怎么实现，想new新对象，就new
