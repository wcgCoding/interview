# interview
面试准备总结

转自[掘金](https://juejin.im/post/5e397ccaf265da570b3f1b02)

# runtime相关问题

`runtime`是iOS开发最核心的知识点了，如果下面的问题都解决了，那么对runtime的理解已经很深了。`runtime`已经开源，[objc-runtime](https://github.com/RetVal/objc-runtime).
##结构模型
---
1.介绍下`runtime`的内存模型(isa、对象、类、metaclass、结构体存储信息等)
2.为什么要设计metaclass
3.`class_copyIvarList`和`class_copyPropertyList`的区别
4.`class_rw_t`和`class_ro_t`的区别
5.`category`如何被加载的，两个`category`的load方法加载顺序，两个`category`的同名方法的加载顺序
6.`category`与`extension`区别，能给NSObject添加`extension`吗，结果如何？
7.消息转发机制，消息转发机制和其他语言的消息机制优劣对比
8.在方法调用的时候，`方法查询-> 动态解析 ->消息转发`之前做了什么
9.`IMP`、`SEL`、`Method`的区别与使用场景
10.`load`与`initialize`方法的区别是什么？在继承关系中他们有什么区别？
11.说一说消息转发的优劣

## 内存管理
---
1.`weak`的实现原理？`SideTable`的结构是什么样的
2.关联对象的应用？系统如何实现关联对象的
3.关联对象的如何进行内存管理的？关联对象如何实现weak属性？
4.`Autoreleasepool`的原理？所使用的数据结构是什么？
5.`ARC`的实现原理？`ARC`对`retain`和`release`做了哪些优化
6.`ARC`下哪些情况会造成内存泄漏？

## 其他
---
1.`Method Swizzle`的注意事项
2.属性修饰符`atomic`的内部实现是怎么样的？能保证线程安全吗？
3.iOS内省的几个方法有哪些？内部实现原理是什么？
4.`class`、`objc_getClass`、`object_getClass`方法有什么区别？

# NSNotification相关

苹果并没有开源相关代码，但是可以读下[GUNStep](https://github.com/gnustep/libs-base)的源码，基本上实现方式很具有参考性

1.实现原理（结构设计、通知如何存储的、`name&observer&SEL`之间的关系等）
2.通知的发送是同步的还是异步的？
3.`NSNotificationCenter`接受消息与发送消息是在同一个线程吗？如何异步发送消息?
4.`NSNotificationQueue`是异步还是同步发送？在哪个线程同步
5.`NSNotificationQueue`和`runloop`的关系
6.如何保证通知接收的线程是主线程
7.页面销毁后不移除通知会崩溃吗
8.多次添加同一个通知会是什么结果？多次移除通知呢？
9.下面的方式能接收到通知吗？为什么
```objectivec
// 发送通知
[NSNotificationCenter.defaultCenter postNotificationName:@"TestNotification" object:nil];
// 接收通知
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(handleNotification) name:@"testNotification" object:nil];
```

# Runloop & KVO
## runloop
---
`runloop`对于一个标准的iOS开发来说都不陌生，应该说熟悉`runloop`是标配，下面就随便列几个典型的问题吧
1.app如何接收到触摸事件的
2.为什么只有主线程的`runloop`是开启的
3.为什么只在主线程刷新UI
4.`PerformSelector`和`runloop`的关系
5.如何使线程保活

## KVO
---
同`runloop`一样，这也是标配的知识点了，同样列几个典型的问题。
1.实现原理
2.如何手动关闭KVO
3.通过KVC修改属性会触发KVO么
4.哪些情况下使用KVO会出现崩溃，如何防护？
5.KVO的优缺点

# Block
1.`block`的内部实现，结构体是什么样的
2.block是类吗？有哪些类型
3.一个`int`被`__block`修饰与否的区别？block的变量截获
4.`block`在修改`NSMutableArray`,需不需要添加`__block`
5.怎么进行内存管理的
6.`block`可以用strong修饰吗
7.解决循环引用为什么要使用`__strong`、`__weak`修饰
8.`block`发生`copy`时机
9.`block`访问对象类型的`auto变量`时，在ARC和MRC下有什么区别

# 多线程
主要以GCD为主
1.`iOS`开发中有多少类型的线程？分别对比
2.`GCD`有哪些队列，默认提供哪些队列
3.`GCD`有哪些方法api
4.`GCD`主线程&主队列的关系
5.如何实现同步，有多少方式就说多少
6.`dispatch_once`的实现原理
7.什么情况下会死锁
8.有哪些类型的线程锁，分别介绍下作用和使用场景
9.`NSOperationQueue`中的`maxConcurrentOperationCount`默认值
10.`NSTimer`、`CADisplayLink`、`dispatch_source_t`的优劣

# 视图图像相关
1.`AutoLayout`的原理，性能如何
2.`UIView`与`CAlayer`的区别
3.事件响应链
4.`drawrect`与`layoutsubviews`的调用时机
5.UI的刷新原理
6.隐式动画与显示动画的区别
7.什么是离屏渲染
8.`imageName`与`imageWithContentsOfFile`的区别 多个相同的图片会重复加载吗
9.图片是什么时候解码的，如何优化
10.图片渲染怎么优化
11.如果GPU的刷新率超过了iOS屏幕60Hz刷新率是什么现象，怎么解决

# 性能优化
1.如何做启动优化、如何监控
2.如何做卡顿优化、如何监控
3.如何做耗电优化、如何监控
4.如何做网络优化、如何监控

# 开发证书
1.苹果使用证书的目的是什么
2.AppStore安装app的认证流程
3.开发者怎么在debug模式下把app安装到设备

# 架构设计

## 典型源码的学习
---
只是列出一些iOS比较核心的开源库，这些库包含了很多高质量的思想，源码学习的时候一定要关注每个框架解决的核心问题是什么，还有它们的优缺点，这样才能算真正理解和吸收
1.AFN
2.SDWebImage
3.CTMediator、其他router库，这些都是常见的路由库，开发中基本上都会用到

## 架构设计
---
1.手动埋点、自动埋点和可视化埋点
2.`MVC`、`MVVM`和`MVP`设计模式
3.常见的设计模式
4.单例的弊端
5.常见的路由方案，以及优缺点对比
6.如果保证项目的稳定性
7.设计一个图片缓存框架(LRU)
8.如何设计一个`git diff`
9.设计一个线程池？画出你的架构图
10.你的app架构是什么，有什么优缺点、为什么这么做、怎么改进

# 其他问题
1.`PerformSelector & NSInvocation`优劣对比
2.`OC`如何实现多继承？怎么面向切面
3.哪些`bug`会导致崩溃，如何防护崩溃
4.怎么监控崩溃
5.app的启动过程（考察LLVM的编译过程、静态链接、动态链接、runtime初始化）
6.沙盒目录的每个文件夹划分的作用
7.简述下`match-o`的文件结构


# 系统基础知识
1.进程和线程的区别
2.Https的握手过程
3.什么是`中间人攻击`？怎么预防？
4.`TCP`的握手过程？为什么进行三次握手、四次挥手？
5.堆区和栈区的区别，谁的占用空间大？
6.加密算法：`对称加密算法和非对称加密算法`区别
7.常见的`对称加密和非对称加密算法`有哪些
8.`MD5`、`Sha1`、`Sha256`区别
9.`charles`抓包过程？不使用`charles`，`4G`网络如何抓包

# 数据结构与算法
对于移动开发者来说，一般不会遇到非常难的算法，大多以数据结构为主，笔者列出一些必会的算法，当然有时间了可以去[LeetCode](https://leetcode.com/)上刷刷题
1.八大排序算法
2.栈&队列
3.字符串处理
4.链表
5.二叉树相关操作
6.深搜广搜
7.基本的动态规划题、贪心算法、二分查找

# 总结
未完待续


