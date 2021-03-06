
---

### OC-runtime 简述

在oc中，`runtime`是由c/c++/汇编所编写的一套c语言的API，在oc的不断完善之下发展出了两个大版本，分别是旧版本`Legacy`,和现代版本`Modern`，在开源代码中分别以`old`和`new`的尾缀展示，在 [这里(开源代码)](https://opensource.apple.com/tarballs/objc4/)可以下载到runtime的源码（由apple所维护的）,[这里(官方文档)](https://developer.apple.com/documentation/objectivec/objective-c_runtime#//apple_ref/doc/uid/TP40001418-CH1g-126286)可以找到官方文档 ！

---

### Command+R


oc代码的运行过程为：OC语言 -> c++ -> 汇编语言 -> 机器语言 

在xcode中，代码会先`编译`，然后`运行`，这两个过程分别对应为`编译时`和`运行时`，`编译时`会检查代码是否有明显的错误，例如未定义的类名或不规范的函数使用，`运行时`负责后续所以操作，与很多错误`编译时`并无法检测出来，`运行时`便会将错误收集并使当前的程序`Crash`，以维持系统程序的稳定

---

### 为了阅读源码需要了解的概念

#### 位运算 `&`

在计算机编程中，最基本的单位为`位`,8位构成一个字节,而`位运算`便是在`位`的基础上进行运算，`&`运算符代表`与运算`，对应了电路学中的`与门`，其运算规则为
```
    0 & 0 = 0
    0 & 1 = 0
    1 & 0 = 0
    1 & 1 = 1
```

#### 位运算 `|`
`|`运算符代表`或运算`，对应了电路学中的`或门`，其运算规则为
```
    0 & 0 = 0
    0 & 1 = 1
    1 & 0 = 1
    1 & 1 = 1
```

#### 位运算 `<<`
`|`运算符代表`左移运算`，是指将某个`字节`中的所有`位`向左边移动特定的位置，例如 
```
1 << 1
0000 0001 -> 0000 0010
所以 1 << 1 = 2

4 << 2
0000 0100 -> 0001 0000
所以 4 << 2 = 16

5 << 3
0000 0101 -> 0010 1000
所以 5 << 3 = 40

```

#### 位域 `:1`
```
    struct {
        char one: 1;
        char two: 2;
        char three: 3;
    } Onetwothree
```
上面的结构体中，因为`one` `two` `three`分别指定了`位域`，且`位域`为`1`，也就是它们分别占了`1位`，一共占用了`3位`，出于`对齐原则`，结构体`Onetwothree`便占用了`8位`，也就是`一个字节`。

#### 共用体 `Union`
```
    union {
        uint32_t I;
        float F;
    } T;
```
`共用体`与`结构体`表达式相类似，单其所有成员共用一块`内存`，也就是其中任意一个值改变，其他的值也会随之改变，共用体所占的内存取决于成员最大的占用内存

#### 编译
```
    
    clang -rewrite-objc main.m -o -main.mm

    // 针对平台进行优化
    xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m -o main.mm
```
如果出现Foundation库丢失的情况，是因为mac未指定xcode，无法定位到xcode包，执行`sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer/`便可以解决这个问题。

