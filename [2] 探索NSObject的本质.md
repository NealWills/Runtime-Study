
---

### `NSObject类`

我们已经知道，在程序编译时，OC代码会先编译为c/c++代码，所以我门来看下，一个NSObject对象到底是什么。

新建一个 macOS - Command Terminal Tool命令行工具，将`main.m`中的`init main`方法修改为下面这样:
```
int main(int argc, const char * argv[]) {

    NSObject *objc = [[NSObject alloc] init];

    return 0;

}
```
然后编译为C++文件,在main.m同级目录下可以看到新生成的main.mm文件
```
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m -o main.mm
```
可以看到，`init main`方法被编译为
```
int main(int argc, const char * argv[]) {

    NSObject *objc = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init"));

    return 0;

}
```
在main.mm文件中搜索`NSObject`可以搜索出一个结构体
```
struct NSObject_IMPL {
	Class isa;
};
```
可以看到，实际上在C++中，`NSObject`是一个命名为`NSObject_IMPL`的结构体，`IMPL`是`implement`的缩写，也就是`NSObject`的`底层实现`，它的成员只有一个`Class`类型的`isa`，通过查看runtime的源码，发现`Class`实际上是`typedef struct objc_class *Class;`,也是一个结构体。
所以，我们可以得出结论，`NSObject类的本质是结构体`。

---

`Person类`

拓展一下，Person类继承于NSObject，那么Person对象的本质又是什么呢？
我们新增一个类，并将`main.m`中的`init`方法修改为
```


@interface Person : NSObject

@end

@implementation Person

@end

int main(int argc, char * argv[]) {
    
    @autoreleasepool {
        
        Person *person = [[Person alloc] init];
        
    }
    return 0;
}
```
然后重新将代码编译为C++代码，便可以找到`Person_IMPL`，它的结构是这样的：
```
    //ifndef _REWRITER_typedef_Person
    //define _REWRITER_typedef_Person
    typedef struct objc_object Person;
    typedef struct {} _objc_exc_Person;
    //endif

    struct Person_IMPL {
        struct NSObject_IMPL NSObject_IVARS;
    };
```
因为并没有声明其他参数，所以在它的实现中，就只有一个成员:`NSObject_IMPL`类型的结构体, 那如果加上几个成员呢？
```

@interface Person : NSObject {
    @public
    NSString *name;
    int age;
}

struct Person_IMPL {
	struct NSObject_IMPL NSObject_IVARS;
	NSString *name;
	int age;
};
```

### 内存空间

我们可以用`malloc`模块和`runtime`模块来打印出`申请的内存空间`和`实际使用的空间`
```
NSLog(@"%zd - %zd", class_getInstanceSize([NSObject class]), class_getInstanceSize([person class]));
        
NSLog(@"%zd - %zd", malloc_size((__bridge const void *)objc), malloc_size((__bridge const void *)person));
```

```

struct NSObject_IMPL {
    Class isa;
};

struct Person_IMPL {
    struct NSObject_IMPL NSObject_IVARS;
    NSString *name;
    int age;
};

//打印的结果为

// class_getInstanceSize  8 - 24
// malloc_size 16 - 32

```

```
struct NSObject_IMPL {
    Class isa;
};

struct Person_IMPL {
    struct NSObject_IMPL NSObject_IVARS;
//    NSString *name;
    int age;
};

//打印的结果为

// class_getInstanceSize  8 - 16
// malloc_size 16 - 16

```

```
struct NSObject_IMPL {
    Class isa;
};

struct Person_IMPL {
    struct NSObject_IMPL NSObject_IVARS;
    NSString *name;
//    int age;

//打印的结果为

// class_getInstanceSize  8 - 16
// malloc_size 16 - 16

};

```