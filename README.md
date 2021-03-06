# interview
面试准备总结 转自[掘金](https://juejin.im/post/5e397ccaf265da570b3f1b02)

# runtime相关问题

`runtime`是iOS开发最核心的知识点了，如果下面的问题都解决了，那么对runtime的理解已经很深了。`runtime`已经开源，[objc-runtime](https://github.com/RetVal/objc-runtime).
```
描述：主要是将数据类型的确定由编译时，推迟到运行时。runtime机制使我们直到运行时采取决定一个对象的类别以及调用该类别对象指定的方法。
```
## 结构模型

1.介绍下`runtime`的内存模型(isa、对象、类、metaclass、结构体存储信息等)
```
答：每个对象、类、metaclass 都有一个isa指针。对象isa指针 指向类对象，类对象isa指针 指向metaclass，metaclass的isa指针指向root metaclass对象。
root metaclass的isa指向它自己。
// 实例对象的结构体
struct objc_object {
private:
    isa_t isa;
    ...
}
// isa_t的结构存储
union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }
    Class cls;
 }
// Class的结构体
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
    ...
}

// 在编译期，类的相关方法，属性，协议会被添加到class_ro_t这个只读的结构体中。
// 在运行时，类第一次被调用的时候，class_rw_t会被初始化，category中的内容依然是这个时候被添加进来的。
// class_rw_t不仅仅用来存放运行时添加的信息，编译期确定下来的信息也会被拷贝进去。

```

2.为什么要设计metaclass
```
答：因为实例对象的方法都是相同的，如果存放在每个实例对象的结构体中就会造成内存浪费，所以统一放到类结构体与元类结构体可以更合理。
```

3.`class_copyIvarList`和`class_copyPropertyList`的区别
```
答: class_copyIvarList 可以获取到.h与.m中的属性变量与成员变量，class_copyPropertyList只能获取到属性变量。
```

4.`class_rw_t`和`class_ro_t`的区别
```
答: class_rw_t 是存储着运行时确定下来的属性、方法还有遵循的协议，class_ro_t只存储编译时确定的属性、方法还有遵循的协议。前者包含后者。
```

5.`category`如何被加载的，两个`category`的load方法加载顺序，两个`category`的同名方法的加载顺序
```
答: 1.category在编译时只是一个包含类名，类指针，方法列表，属性列表，协议列表的_category_t结构体；
2.在运行时通过runtime加载分类数据，把分类的方法、属性、协议数据合并到一个大数组中，后参与编译的分类会在数组的前面(i–倒数遍历数组添加的)（这也说明了分类中相同的方法后参与编译的分类会被调用）；
3.合并后的分类数据会插入到原来类数据的前面（相同的方法，分类会优先于原来调用）。
编译顺序是可以手动设置的：TARGETS->BuildPhases->Complle Sources。原类>分类，两个分类则看谁在前面就先编译，先加载load。同名方法后编译的覆盖前面的。
```

6.`category`与`extension`区别，能给NSObject添加`extension`吗，结果如何？
```
答: xtension在编译的时候，它的数据就已经包含在类信息中;category是在运行时，才会将数据合并到类信息中。
前者可以添加属性，后者不能添加属性。因为在运行期，对象的内存布局已经确定，如果添加实例变量会破坏类的内部布局，这对编译性语言是灾难性的。不能给NSObject添加extension，你必须有一个类的源码才能添加一个类的extension。
```

7.消息转发机制，消息转发机制和其他语言的消息机制优劣对比
```
答: OC消息转发机制，动态方法解析-备援接受者-完整的消息转发。
+resolveInstanceMethod->-forwardingTargetForSelector->-(methodSignatrueForSelector&-forwardInvocation)->-doesNotRecognizeSelector

// 可以实现多继承、多重代理、为@dynamic实现方法、动态更新
```

8.在方法调用的时候，`方法查询-> 动态解析 ->消息转发`之前做了什么

9.`IMP`、`SEL`、`Method`的区别与使用场景
```
答：1.SEL就是通过@selector获取到的数据结构，可以简单看成一个方法的名称。
2.IMP就是函数指针，方法的具体实现。typedef id (*IMP)(id, SEL, ...); 
3.method_types就是参数与返回值的数据类型
Method包括以上三个部分。就是SEL与IMP之间做了映射。有了Method和SEL就能找到IMP。
方法调用的过程是从SEL寻找到IMP,然后执行IMP。
使用场景：可以创建一个IMP，直接调用IMP，不走消息发送流程。
方法交换，给类动态添加一个方法。
```

10.`load`与`initialize`方法的区别是什么？在继承关系中他们有什么区别？
```
答: load方法是类被程序装载时调用，initialize是类或它的子类第一次被发送消息时调用。
load方法只会执行一次，initialize可能被执行多次。
load方法不走消息发送流程不会和其他方法一样有继承关系，initialize走消息发送流程有继承关系。
如果分类定义了这俩方法，分类中的load方法会最后调用，分类中的initialize会覆盖子类与父类的。

load: 类先调用，在调用分类。父类先调用，再调用子类。
initialize: 分类先调用，在调用类。父类先调用，再调用子类。
```

11.说一说消息转发的优劣

## 内存管理

1.`weak`的实现原理？`SideTable`的结构是什么样的
```
答: weak_table_t是一个全局weak 引用的表，使用不定类型对象的地址作为 key，用 weak_entry_t 类型结构体对象作为 value 。
weak是Runtime维护了一个hash(哈希)表，用于存储指向某个对象的所有weak指针。weak表其实是一个hash（哈希）表，Key是所指对象的地址，Value是weak指针的地址（这个地址的值是所指对象指针的地址）数组。

```

2.关联对象的应用？系统如何实现关联对象的
```
答：关联对象配合分类，可以给类增加属性变量。关联对象的实现不复杂，保存的方式为一个全局的哈希表，存取都通过查询表找到关联来执行。哈希表的特点就是牺牲空间换取时间，所以执行速度也可以保证。

```

3.关联对象的如何进行内存管理的？关联对象如何实现weak属性？
```
答：在objc_setAssociatedObject实际调用的是_object_set_associative_reference
后者有调用 acquireValue。 首先把新传入的对象，根据协议进行retain/copy，在赋值的过程中获取旧值，在方法结束前release。

使用block捕获的方式
-(void)setWeakvalue:(NSObject *)weakvalue {
    __weak typeof(weakvalue) weakObj = weakvalue;
    typeof(weakvalue) (^block)() = ^(){
        return weakObj;
    };
    objc_setAssociatedObject(self, weakValueKey, block, OBJC_ASSOCIATION_COPY_NONATOMIC);
}
-(NSObject *)weakvalue {
    id (^block)() = objc_getAssociatedObject(self, weakValueKey);
    return block();
}
```
4.`Autoreleasepool`的原理？所使用的数据结构是什么？
```
答: 简单说是双向链表，每张链表头尾相接，有 parent、child指针
每创建一个池子，会在首部创建一个 哨兵 对象,作为标记
最外层池子的顶端会有一个next指针。当链表容量满了，就会在链表的顶端，并指向下一张表。
```

5.`ARC`的实现原理？`ARC`对`retain`和`release`做了哪些优化
```
答：我们都知道，ARC是编译器特性，程序在编译的时候，编译器帮我们在合适的地方插入retain、release等代码以管理对象的引用计数，从而达到自动管理对象生命周期的目的。但是只有编译器是无法单独完成这一工作的，还需要OC运行时库的配合协助。
由于编译器自己加入的retain与release，所以都是c语言的底层实现，不经过发送消息，更高效。

```

6.`ARC`下哪些情况会造成内存泄漏？
```
答: 循环引用，delegate block里用self等。CoreFundation下的要自己手动release的未手动释放的。
```
## 其他

1.`Method Swizzle`的注意事项
```
答: 如果交换方法的原方法没有实现，需要实现一个空操作。不然如果外界调用新方法走旧的实现会出现crash。如果，target不同，方法交换之后方法里的使用了self，类型会变化这时候需要注意。
```

2.属性修饰符`atomic`的内部实现是怎么样的？能保证线程安全吗？
```
答：在set/get方法中使用spinlock_t自旋锁实现。atomic通过这种方法，在运行时保证 set,get方法的原子性。仅仅是保证了set,get方法的原子性。这种线程是不安全的。    self.intA = self.intA + 1;非原子性。
```

3.iOS内省的几个方法有哪些？内部实现原理是什么？
```
答: [object isKindOfClass]、[object isMemberOfClass]
respondsToSelector、instancesRespondToSelector
通过对象isa指针，找到类对象，查看类对象的名称与传入的类对象名称是否一致。
```

4.`class`、`objc_getClass`、`object_getClass`方法有什么区别？
```
答:
object_getClass：获取isa的指向
self.class: self是实例对象返回类对象，如果是类返回类本身。
```

# NSNotification相关

苹果并没有开源相关代码，但是可以读下[GUNStep](https://github.com/gnustep/libs-base)的源码，基本上实现方式很具有参考性

1.实现原理（结构设计、通知如何存储的、`name&observer&SEL`之间的关系等）
```
答: 在iOS中，NSNotification & NSNotificationCenter是使用观察者模式来实现的用于跨层传递消息。
将观察者注册到NSNotificatinonCenter的通知调度表中，然后发送通知时利用标识符name和object识别出调度表中的观察者，然后调用相应的观察者的方法，即传递消息（在Objective-C中对象调用方法，就是传递消息，消息有name或者selector，可以接受参数，而且可能有返回值），如果是基于block创建的通知就调用NSNotification的block。
// 猜想
通知模型 存为数组，不同name的通知 分成不同数组，再以name为key存放到NSSet中。
```

2.通知的发送是同步的还是异步的？
```
答：默认同步的，但是也可以异步。

NSNotification *notification = [NSNotification notificationWithName:kNotificationName
                                                                 object:@"通知说话开始"];
    [[NSNotificationQueue defaultQueue] enqueueNotification:notification
                                               postingStyle:NSPostASAP];
```

3.`NSNotificationCenter`接受消息与发送消息是在同一个线程吗？如何异步发送消息?
```
答：在同一个线程。使用NSNotificationQueue来进行异步发送（NSPostWhenIdle、NSPostASAP）
```


4.`NSNotificationQueue`是异步还是同步发送？在哪个线程同步
```
答：NSPostNow:同步发送，NSPostWhenIdle、NSPostASAP:异步发送。在所发通知那个线程进行同步。
```

5.`NSNotificationQueue`和`runloop`的关系

6.如何保证通知接收的线程是主线程
```
答：在主线程发送通知，或者在接收方法中回到主线程。
```

7.页面销毁后不移除通知会崩溃吗
```
答：会崩溃。会出现找不到方法的异常。
```

8.多次添加同一个通知会是什么结果？多次移除通知呢？
```
答：多次添加一个通知，接收通知时会调用多次。多次移除没问题。
```

9.下面的方式能接收到通知吗？为什么
```objectivec
// 发送通知
[NSNotificationCenter.defaultCenter postNotificationName:@"TestNotification" object:nil];
// 接收通知
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(handleNotification) name:@"testNotification" object:nil];
```
```
答：不能，因为，添加观察者在发送通知之后进行。
```

# Runloop & KVO

## runloop

`runloop`对于一个标准的iOS开发来说都不陌生，应该说熟悉`runloop`是标配，下面就随便列几个典型的问题吧

1.app如何接收到触摸事件的
```
答：1.发生触摸事件时，系统会将该事件加入由UIApplication管理的事件队列中。
2.UIApplication会从队列中取出最前面的事件，并将事件分发下去，通常先发送给应用程序主窗口keywindow;
3.主窗口会在视图层次中找到一个最适合的view来处理这次事件，找到最适合的view之后就会调用视图控件的touches方法来作具体的事件处理。
```
*注意：事件的传递是从上到下（父控件到子控件），事件的响应是从下到上（顺着响应者链条向上传递：子控件到父控件。*

2.为什么只有主线程的`runloop`是开启的
```
答：因为主线程是需要不停处理用户事件等各种source。所以需要开启runloop保活。
```
3.为什么只在主线程刷新UI
```
答：安全+效率：因为UIKit框架不是线程安全的框架，当在多个线程进行UI操作，有可能出现资源抢夺，导致问题。
```

4.`PerformSelector`和`runloop`的关系
```
答：如果在子线程中使用performSelector,如果没有开启runloop是不会执行方法的。需要调用currentRunloop 并且写run才行。
```
```
// 监听runloop的代码
- (void)addObserver
{
    /*
     kCFRunLoopEntry = (1UL << 0),1
     kCFRunLoopBeforeTimers = (1UL << 1),2
     kCFRunLoopBeforeSources = (1UL << 2), 4
     kCFRunLoopBeforeWaiting = (1UL << 5), 32
     kCFRunLoopAfterWaiting = (1UL << 6), 64
     kCFRunLoopExit = (1UL << 7),128
     kCFRunLoopAllActivities = 0x0FFFFFFFU
     */
    CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(CFAllocatorGetDefault(), kCFRunLoopAllActivities, YES, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
        switch (activity) {
            case 1:
            {
                NSLog(@"进入runloop");
            }
                break;
            case 2:
            {
                NSLog(@"timers");
            }
                break;
            case 4:
            {
                NSLog(@"sources");
            }
                break;
            case 32:
            {
                NSLog(@"即将进入休眠");
            }
                break;
            case 64:
            {
                NSLog(@"唤醒");
            }
                break;
            case 128:
            {
                NSLog(@"退出");
            }
                break;
            default:
                break;
        }
    });
    CFRunLoopAddObserver(CFRunLoopGetCurrent(), observer, kCFRunLoopCommonModes);//将观察者添加到common模式下，这样当default模式和UITrackingRunLoopMode两种模式下都有回调。
    self.obsever  = observer;
    CFRelease(observer);
}
```

5.如何使线程保活
```
答：可以给runloop添加一个repeats为YES的timer或者添加一个[runloop addPort:[NSPort port] forMode:NSDefaultRunLoopMode];
```

## KVO

同`runloop`一样，这也是标配的知识点了，同样列几个典型的问题。

1.实现原理
```
答：实现了一个NSNotificaing_Person类，将实例的isa指针指向NSNotificaing_Person。重写set方法，里面调用willChange 与didChange方法触发观察者对象。
```

2.如何手动关闭KVO
```
答：系统方法
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key {
    return NO;
}

```

3.通过KVC修改属性会触发KVO么
```
答：会触发(没有set方法也是可以的)
```

4.哪些情况下使用KVO会出现崩溃，如何防护？
```
答：添加了观察者，忘记在对象销毁时移除。只要不是添加和移除不是成对的出现，那么就会出现crash。
```

5.KVO的优缺点
```
答：优点：
1.能够提供一种简单的方法实现两个对象间的同步。例如：model和view之间同步；

2.能够对非我们创建的对象，即内部对象的状态改变作出响应，而且不需要改变内部对象（SKD对象）的实现；

3.能够提供观察的属性的最新值以及先前值；

4.用key paths来观察属性，因此也可以观察嵌套对象；

5.完成了对观察对象的抽象，因为不需要额外的代码来允许观察值能够被观察
缺点：
1.复杂的“IF”语句要求对象正在观察多个值。这是因为所有的观察代码通过一个方法来指向；
2.我们观察的属性必须使用strings来定义。因此在编译器不会出现警告以及检查；
```

# Block

1.`block`的内部实现，结构体是什么样的
```
答：block本质上也是一个oc对象，他内部也有一个isa指针。block是封装了函数调用以及函数调用环境的OC对象。

```

2.block是类吗？有哪些类型
```
答：block可以看作是类，有全局block、栈block和堆block三种类型。
```

3.一个`int`被`__block`修饰与否的区别？block的变量截获
```
答：如果没有被__block修饰，block截获的只是值拷贝；如果使用__block修饰，会生成一个结构体，复制int的引用地址，达到修改数据。

```

4.`block`在修改`NSMutableArray`,需不需要添加`__block`
```
答：不需要，因为NSMutableArray不是基本数据类型，在block中是指针传递。

// 基本数据类型，用__block修饰的变量编译后会变成结构体实例,这时候的改值等于是改变了结构体实例的成员变量
```

5.怎么进行内存管理的

6.`block`可以用strong修饰吗
```
答：ARC环境下可以使用strong修饰。
```

7.解决循环引用为什么要使用`__strong`、`__weak`修饰
```
答：__weak 修饰 是为了防止循环引用  __weak 修饰的对象在block内部不会对计数器+1
是因为某个特定场景， 比如block中使用了用__weak修饰的对象，但是在执行block代码块的时候__weak修饰的对象已经被释放掉了， 这个时候就会报空指针错误，这个时候就要在block内部用__strong修饰弱引用的对象，这样就不会造成空指针异常。
```

8.`block`发生`copy`时机
```
1.调用Block的copy实例方法

2.Block作为函数返回值返回时

3.将Block赋值给附有__strong修饰符id类型的类或Block类型成员变量时

4.在方法名中含有usingBlock的Cocoa框架方法或Grand Central Dispatch的API中传递Block时
```

9.`block`访问对象类型的`auto变量`时，在ARC和MRC下有什么区别
```
答：// MRC下，__block修饰的变量成为对象后，被block使用后没有强引用的关系，而ARC下有强引用的关系。
```

# 多线程

主要以GCD为主

1.`iOS`开发中有多少类型的线程？分别对比
```
答：GCD、NSThread、NSOperationQueue、pthread 共四种方案
```

2.`GCD`有哪些队列，默认提供哪些队列
```
答：共有五种队列，有串行队列与并行队列，默认提供主队列，以及四种优先级的并行队列。
```

3.`GCD`有哪些方法api
```
dispatch_async()/dispatch_sync()/dispatch_apply()/dispatch_barrier_async()/disptahc_after()/dispatch_group_async()
/dispatch_io_read()/dispatch_semaphore_signal
```

4.`GCD`主线程&主队列的关系
```
答：主队列里的任务一定是在主线程执行的，但是主线程里执行的任务却不一定在主队列。
判断是否在主队列的方法可以使用 dispatch_queue_set_specific和 dispatch_get_specific 这一组方法。
1.第一个测试代码是 DispatchQueue.global().sync(execute: log);
```

5.如何实现同步，有多少方式就说多少
```
答：加锁：互斥锁、自旋锁、递归锁、条件锁或者使用信号量也可以做到同步。
dispatch_barrier_async
dispatch_semaphore_t
@sync
```
6.`dispatch_once`的实现原理
```
答：内部使用信号量，进行线程阻塞，外部传一个int指针地址，当int值为0时执行block 为-1时直接不执行block跳过。其他值则阻塞等待int为-1。
```

7.什么情况下会死锁
```
答：互斥条件（一个资源每次只能被一个进程使用。）、请求和保持条件（一个进程因请求资源而阻塞时，对已获得的资源保持不放。）、不剥夺条件（进程已获得的资源，在末使用完之前，不能强行剥夺。）以及环路等待条件（若干进程之间形成一种头尾相接的循环等待资源关系。）
在某一个串行队列中，同步的向这个队列添加block会死锁。
```

8.有哪些类型的线程锁，分别介绍下作用和使用场景

9.`NSOperationQueue`中的`maxConcurrentOperationCount`默认值
```
答：默认值是-1，就是无限开子线程。
```
10.`NSTimer`、`CADisplayLink`、`dispatch_source_t`的优劣
```
NSTimer 简单易用 以指定的模式注册到runloop后，每当设定的周期时间到达后，runloop会向指定的target发送一次指定的selector消息。周期设置方式简单，但是精度受runloop的阻塞影响导致不精准。
CADisplayLink以特定模式注册到runloop后， 每当屏幕显示内容刷新结束的时候，runloop就会向 CADisplayLink指定的target发送一次指定的selector消息， CADisplayLink类对应的selector就会被调用一次。由于屏幕刷新是固定的所以精度比NSTimer高，周期设置不灵活。60/A
dispatch_source_t精度很高，系统自动触发，系统级别的源。(YYTimer就是这个底层原理)

```


# 视图图像相关

1.`AutoLayout`的原理，性能如何
```
答：设置好约束后，AutoLayout会通过求解多元一次方程，得到view的frame再进行设置。性能没有直接设置frame效果好，随着视图嵌套的增加，性能下降厉害。
```
2.`UIView`与`CAlayer`的区别
```
答：UIView 是对CAlayer的封装，UIView是CAlayer的代理，前者可以响应事件，后者指负责绘制。
1.首先UIView可以响应事件，CALayer不可以响应事件
2.一个 Layer 的 frame 是由它的 anchorPoint,position,bounds,和 transform 共同决定的，而一个 View 的 frame 只是简单的返回 Layer的 frame
3.UIView主要是对显示内容的管理而 CALayer 主要侧重显示内容的绘制
4.在做 iOS 动画的时候，修改非 RootLayer的属性（譬如位置、背景色等）会默认产生隐式动画，而修改UIView则不会。
```

3.事件响应链
```
答：当事件发生时，必须知道谁来响应。再iOS中，又响应者链进行响应，所有事件响应的类都是UIResponder的子类，响应者链是一个由不同对象组成的层次结构，其中的每个对象将依次获得响应事件消息的机会。当发生事件时，事件首先被发送给第一响应者，第一响应者往往是事件发生的视图，也就是用户触摸屏幕的地方。事件将沿着响应者链一直向下传递，直到被接受并做出处理。事件响应链 是事件传递的反方向。

First Responser -- > The Window -- >The Application -- > App Delegate
```

4.`drawrect`与`layoutsubviews`的调用时机
```
答：layoutsubviews 在以下情况会被调用：
1.改变一个UIView的Frame会触发layoutSubviews，当然前提是frame的值设置前后发生了变化。
2、滚动一个UIScrollView引发UIView的重新布局会触发layoutSubviews。
3、旋转Screen会触发父UIView上的layoutSubviews事件。
4、直接调用setNeedsLayout 或者 layoutIfNeeded。
drawrect在以下情况会被调用：
1.drawRect 掉用是在Controller->loadView, Controller->viewDidLoad 两方法之后掉用的.
2.该方法在调用sizeToFit后被调用，所以可以先调用sizeToFit计算出size。然后系统自动调用drawRect:方法。
3.通过设置contentMode属性值为UIViewContentModeRedraw。那么将在每次设置或更改frame的时候自动调用drawRect:。
4.直接调用setNeedsDisplay，或者setNeedsDisplayInRect:触发drawRect:，但是有个前提条件是rect不能为0。
```

5.UI的刷新原理
```
答：CPU和GPU都是由总线连接起来的，在CPU输出的往往是一个位图，上传给GPU。GPU拿到位图之后，会做一个图层的渲染，包括纹理的合成，最后会把结果放到帧缓冲区中， 由视图控制器根据VSync信号（帧同步信号,表示扫描1帧的开始，一帧也就是LCD显示的一个画面。）在指定时间之前，在帧缓冲区当中提取对应显示内容，最后展示到屏幕显示器上。

1.首先对于CPU来说，要完成UI的布局，包括显示或者说绘制，之后会做一些准备工作，最后会将位图提交到GPU上面。
1).Layout：UI布局、文本计算 2).Display：绘制（drawRect）3).Prepare：图片编解码 4).Commit：提交位图

```

6.隐式动画与显示动画的区别

7.什么是离屏渲染
```
答：离屏渲染就是GPU在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作。当我们在设置某些UI视图的图层属性，如果说指令为在未预合成之前，不能用于直接显示的时候呢，那么就触发了离屏渲染。

```

8.`imageName`与`imageWithContentsOfFile`的区别 多个相同的图片会重复加载吗
```
答：imageName获取的文件会缓存起来，相同的图片不会重复加载；imageWithContentsOfFile不会缓存，相同的图片会重复加载。
```

9.图片是什么时候解码的，如何优化
```
答：一般我们使用的图像是JPEG/PNG，这些图像数据不是位图，而是是经过编码压缩后的数据，需要线将它解码转成位图数据，然后才能把位图渲染到屏幕上。

当你用 UIImage 或 CGImageSource 的那几个方法创建图片时，图片数据并不会立刻解码。图片设置到 UIImageView 或者 CALayer.contents 中去，并且 CALayer 被提交到 GPU 前，CGImage 中的数据才会得到解码。

将图片解码放到子线程。
```
10.图片渲染怎么优化
```
答：避免离屏渲染。因为在触发离屏渲染的时候，会增加 GPU 的工作量，而增加 GPU 的工作量很有可能会到导致CPU和GPU工作总耗时超出了16.7ms，那么可能就会导致UI的卡顿和掉帧，那么我们就要避免离屏渲染。
```

11.如果GPU的刷新率超过了iOS屏幕60Hz刷新率是什么现象，怎么解决
```
答：由于iOS 有垂直同步Vsync，所以超过60帧没有什么现象。
```
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

只是列出一些iOS比较核心的开源库，这些库包含了很多高质量的思想，源码学习的时候一定要关注每个框架解决的核心问题是什么，还有它们的优缺点，这样才能算真正理解和吸收

1.AFN

2.SDWebImage

3.CTMediator、其他router库，这些都是常见的路由库，开发中基本上都会用到

## 架构设计

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
```
答：通过协议实现多继承，
```

3.哪些`bug`会导致崩溃，如何防护崩溃
```
KVO/NSNotification/后台任务超时/内存爆满/主线程卡顿超阈值/数组越界/野指针
答：1.方法找不到
2.数组越界、插入空值
3.字典初始化时，key和value数量不一致
4.字典setObject:forKey: removeObjectForKey: key为nil时。
5.使用kvc时 使用了错误的key。
6.偏好设置的key为nil时
7.字符串操作时，传入的下标超出范围。
8.控件addView之前 使用autoLayout进行布局
9.访问野指针
10.死锁
11.除0
```

4.怎么监控崩溃
```
1.有信号发出的监听信号
void registerSignalHandler(void){
    signal(SIGSEGV, handleSignalException);//试图访问未分配给自己的内存, 或试图往没有写权限的内存地址写数据.
    signal(SIGFPE, handleSignalException);//在发生致命的算术运算错误时发出. 不仅包括浮点运算错误, 还包括溢出及除数为0等其它所有的算术的错误。
    signal(SIGBUS, handleSignalException);//非法地址, 包括内存地址对齐(alignment)出错。比如访问一个四个字长的整数, 但其地址不是4的倍数。它与SIGSEGV的区别在于后者是由于对合法存储地址的非法访问触发的(如访问不属于自己存储空间或只读存储空间)。
    signal(SIGPIPE, handleSignalException);//管道破裂。这个信号通常在进程间通信产生，比如采用FIFO(管道)通信的两个进程，读管道没打开或者意外终止就往管道写，写进程会收到SIGPIPE信号。此外用Socket通信的两个进程，写进程在写Socket的时候，读进程已经终止
    signal(SIGHUP, handleSignalException);//本信号在用户终端连接(正常或非正常)结束时发出, 通常是在终端的控制进程结束时, 通知同一session内的各个作业, 这时它们与控制终端不再关联。登录Linux时，系统会分配给登录用户一个终端(Session)。在这个终端运行的所有程序，包括前台进程组和后台进程组，一般都属于这个 Session。当用户退出Linux登录时，前台进程组和后台有对终端输出的进程将会收到SIGHUP信号。这个信号的默认操作为终止进程，因此前台进 程组和后台有终端输出的进程就会中止。不过可以捕获这个信号，比如wget能捕获SIGHUP信号，并忽略它，这样就算退出了Linux登录， wget也 能继续下载。此外，对于与终端脱离关系的守护进程，这个信号用于通知它重新读取配置文件。
    signal(SIGINT, handleSignalException);//程序终止(interrupt)信号, 在用户键入INTR字符(通常是Ctrl-C)时发出，用于通知前台进  程组终止进程。
    signal(SIGQUIT, handleSignalException);//和SIGINT类似, 但由QUIT字符(通常是Ctrl-)来控制. 进程在因收到SIGQUIT退出时会产生core文件, 在这个意义上类似于一个程序错误信号。
    signal(SIGABRT, handleSignalException);//调用abort函数生成的信号。
    signal(SIGILL, handleSignalException);//执行了非法指令. 通常是因为可执行文件本身出现错误, 或者试图执行数据段. 堆栈溢出时也有可能产生这个信号。
}
2.没有信号发出的，就在监听崩溃的临界点记录信息。
baginBackgroundTaskWithExpirationHandler
可以根据beginBackgroundTaskWithExpirationHandler会让后台包活三分钟,先设置一个定时器,在接近三分钟的时候判断后台程序是否还在执行,如果还在执行的话,我们就可以判断该程序即将后台崩溃,进行上报记录,已达到监控的效果
```

5.app的启动过程（考察LLVM的编译过程、静态链接、动态链接、runtime初始化）
```
答：1.加载dyld到App进程
2.加载动态库（包括所依赖的所有动态库）
3.Rebase
4.Bind
5.初始化Objective C Runtime
6.其它的初始化代码
```

6.沙盒目录的每个文件夹划分的作用
```
答：Document: 常用目录，iCloud备份目录，您应该将所有的应用程序数据文件写入到这个目录下。这个目录用于存储用户数据。

Library: 包含两个子目录（caches/Preferences）。
1.Preferences 包含应用程序的偏好设置文件。您不应该直接创建偏好设置文件，而是应该使用NSUserDefaults类来取得和设置应用程序的偏好.
2.Caches 用于存放应用程序专用的支持文件，保存应用程序再次启动过程中需要的信息。
可创建子文件夹。可以用来放置您希望被备份但不希望被用户看到的数据。该路径下的文件夹，除Caches以外，都会被iTunes备份。

Temp: 这个目录用于存放临时文件，保存应用程序再次启动过程中不需要的信息。该路径下的文件不会被iTunes备份。
```

7.简述下`match-o`的文件结构
```
答：包含三个部分：
1.header头部，包含可以执行的CPU架构，比如x86，arm64
2.Load Commands 加载命令，包含文件的组织架构和在虚拟内存中的布局方式
3.Data 数据，包含load commands中需要的各个段(segment)的数据，每个Segment的大小都是Page的整数倍。

__TEXT代码段，只读，包含函数，和只读的字符串，上图中类似__TEXT,__text的都是代码段
__Data数据段，读写，包括可读写的全局变量等，__DATA,__data都是数据段
__LINKEDIT包含了方法和变量的元数据(位置，偏移量)，以及代码签名等信息。
```

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


