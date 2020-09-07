
---

### `[[Objc alloc] init]`

一般情况下，我们初始化一个对象时会调用 `[[Obj alloc] init]` 来进行初始化，就算是对其进行重写为 `[[Objc alloc] initWithXXX:xxx]` 还是会在内部去调用 `[[super alloc] init]` 方法，那么，alloc和init到底扮演了一个什么样对角色呢？

---

#### `alloc`

打开xcode的汇编模式，然后在初始化`NSObject对象`时打一个断点，运行后，xcode会停在我们打得断点的地方，并展示出当前汇编进程：
![alloc汇编进程](https://upload-images.jianshu.io/upload_images/11398671-9443f8948ac7ef3b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，在12行调用来一个方法`objc_alloc_init`,我们去到`runtime`的源码(781版本)中搜索这个方法：
![objc_alloc_init](https://upload-images.jianshu.io/upload_images/11398671-d86f73f8630b952a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
然后跟着一路点进去：
![callalloc](https://upload-images.jianshu.io/upload_images/11398671-fb26653d2ccefd9e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
因为现在都是使用的objc2版本也就是`Modern`,所以会走`_objc_rootAllocWithZone`这个方法，所以我们跟着点进去：
![_objc_rootAllocWithZone](https://upload-images.jianshu.io/upload_images/11398671-8dd2a5026d19317c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![_class_createInstanceFromZone](https://upload-images.jianshu.io/upload_images/11398671-09f5aeb32e7a172a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
