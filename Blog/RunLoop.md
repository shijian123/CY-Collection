




## 什么是RunLoop


一般来讲，一个线程一次只能执行一个任务，任务完成后线程就会退出。如果每执行一个任务都创建一个线程会非常浪费资源，所以我们需要一个机制，让线程能随时处理事件但并不退出，通常的代码逻辑为：

```
function loop() {
    initialize();
    do {
        var message = get_next_message();
        process_message(message);
    } while (message != quit);
}
```
这种模型通常被称作 [Event Loop](https://en.wikipedia.org/wiki/Event_loop)。 `Event Loop` 在很多系统和框架里都有实现，比如 Node.js 的事件处理，比如 Windows 程序的消息循环，再比如 OSX/iOS 里的 `RunLoop`，其在没有处理消息时休眠以避免CPU资源占用、在有消息到来时立刻被唤醒。

`RunLoop` 实际上就是一个对象，这个对象管理了其需要处理的事件和消息，并提供了一个入口函数来执行上面 `Event Loop` 的逻辑。线程执行了这个函数后，就会一直处于这个函数内部 “接受消息->等待->处理” 的循环中，直到这个循环结束（比如传入 quit 的消息），函数返回。

OSX/iOS 系统中，`RunLoop`有两个这样的对象：`NSRunLoop` 和 `CFRunLoopRef`。
* `CFRunLoopRef` 是在 `CoreFoundation` 框架内的，它提供了纯 C 函数的 API，这些 API 都是线程安全的。
* `NSRunLoop` 是基于 `CFRunLoopRef` 的封装，提供了面向对象的 API，但是这些 API 不是线程安全的。

CFRunLoopRef 的代码是[开源](https://opensource.apple.com/source/CF/CF-855.17/CFRunLoop.c)的，你可以在[这里](http://opensource.apple.com/tarballs/CF/)下载到整个 CoreFoundation 的源码来查看。Swift 开源后，苹果又维护了[一个跨平台的 CoreFoundation 版本](https://github.com/apple/swift-corelibs-foundation/)，这个版本的源码可能和现有 iOS 系统中的实现略不一样，但更容易编译，而且已经适配了 Linux/Windows。


## RunLoop 与线程的关系

iOS开发遇到两种线程对象：`pthread_t`和`NSThread`，之前苹果文档说`NSThread`是对`pthread_t`的封装，但是现在那份文档已经失效了，有可能现在都是直接包装自更底层的`mach thread`。`CFRunLoop`是基于`pthread`来管理的

苹果不允许直接创建`Runloop`，只提供了两个自动获取的函数，`CFRunloopGetMain()`和`CFRunLoopGetCurrent()`。这两个函数内部的逻辑大概如下：


```
/// 全局的Dictionary，key 是 pthread_t， value 是 CFRunLoopRef
static CFMutableDictionaryRef loopsDic;
/// 访问 loopsDic 时的锁
static CFSpinLock_t loopsLock;
 
/// 获取一个 pthread 对应的 RunLoop。
CFRunLoopRef _CFRunLoopGet(pthread_t thread) {
    OSSpinLockLock(&loopsLock);
    
    if (!loopsDic) {
        // 第一次进入时，初始化全局Dic，并先为主线程创建一个 RunLoop。
        loopsDic = CFDictionaryCreateMutable();
        CFRunLoopRef mainLoop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, pthread_main_thread_np(), mainLoop);
    }
    
    /// 直接从 Dictionary 里获取。
    CFRunLoopRef loop = CFDictionaryGetValue(loopsDic, thread));
    
    if (!loop) {
        /// 取不到时，创建一个
        loop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, thread, loop);
        /// 注册一个回调，当线程销毁时，顺便也销毁其对应的 RunLoop。
        _CFSetTSD(..., thread, loop, __CFFinalizeRunLoop);
    }
    
    OSSpinLockUnLock(&loopsLock);
    return loop;
}
 
CFRunLoopRef CFRunLoopGetMain() {
    return _CFRunLoopGet(pthread_main_thread_np());
}
 
CFRunLoopRef CFRunLoopGetCurrent() {
    return _CFRunLoopGet(pthread_self());
}
```
从上面的代码可以看出:
* 线程和 RunLoop 之间是一一对应的，其关系是保存在一个全局的 Dictionary 里，该字典以线程对象`pthread_t`为`key`，以`RunLoop对象`为`value`
* 线程刚创建时并没有`RunLoop`，如果不主动获取，那它一直都不会有。
* `RunLoop` 的创建是发生在第一次获取时，创建后便会添加进全局字典中，即每一个线程有且仅有一个与之关联的`RunLoop`对象，在线程结束时将`RunLoop`对象销毁。
* 只能在一个线程的内部获取其 RunLoop（主线程除外）。


[苹果官方](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)也给出了一个 RunLoop 模型图，如下：
![](https://upload-images.jianshu.io/upload_images/1426173-e9036de593ce77c4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图也能看出，一个线程会关联一个`RunLoop`对象，`RunLoop`对象会一直循环，直到超时或收到退出指令。在无限循环的过程中会一直处理到来的事件，右侧将事件分为了两类，一类是`Input sources`这部分包括基于端口的`source1`事件，开发者提交的各种`source0`事件，调用`performSelector:onThread:`方法事件，还有一类`Timer sources`，就是常用的定时器事件，这些事件在程序运行期间会不断产生之后会由RunLoop对象检测并负责处理相关事件。


## 关于 RunLoop 的类


在深入了解`RunLoop`的运行机制前，我们先了解下`CoreFoundation`里面关于`RunLoop`的5个类:

* **CFRunLoopRef**：代表 RunLoop 的对象
* **CFRunLoopModeRef**：代表 RunLoop 的运行模式
* **CFRunLoopSourceRef**：就是 RunLoop 模型图中提到的输入源 / 事件源
* **CFRunLoopTimerRef**：就是 RunLoop 模型图中提到的定时源
* **CFRunLoopObserverRef**：观察者，能够监听 RunLoop 的状态改变

它们之间的关系如下:
![](https://upload-images.jianshu.io/upload_images/1426173-41bfb716643323f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看出，一个RunLoop对象（CFRunLoopRef）中包含若干个运行模式（CFRunLoopModeRef）。而每一个运行模式下又包含若干个输入源（CFRunLoopSourceRef）、定时源（CFRunLoopTimerRef）、观察者（CFRunLoopObserverRef）。每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。


### CFRunLoopRef 类

CFRunLoopRef 是 Core Foundation 框架下 RunLoop 对象类。

在Core Foundation中获取RunLoop对象的方法：
* CFRunLoopGetCurrent(); // 获得当前线程的 RunLoop 对象
* CFRunLoopGetMain(); // 获得主线程的 RunLoop 对象

在Foundation下获取RunLoop对象的方法：
* [NSRunLoop currentRunLoop]; // 获得当前线程的 RunLoop 对象
* [NSRunLoop mainRunLoop]; // 获得主线程的 RunLoop 对象

`CFRunLoopMode` 和 `CFRunLoop`的结构:
```
struct __CFRunLoopMode {
    CFStringRef _name;            // Mode Name, 例如 @"kCFRunLoopDefaultMode"
    CFMutableSetRef _sources0;    // Set
    CFMutableSetRef _sources1;    // Set
    CFMutableArrayRef _observers; // Array
    CFMutableArrayRef _timers;    // Array
    ...
};
 
struct __CFRunLoop {
    CFMutableSetRef _commonModes;     // Set
    CFMutableSetRef _commonModeItems; // Set<Source/Observer/Timer>
    CFRunLoopModeRef _currentMode;    // Current Runloop Mode
    CFMutableSetRef _modes;           // Set
    ...
};
```
*  `_commonModes`：是一个集合，里面保存的是具有“Common”的mode的名称
* `_commonModeItems`：也是一个集合，里面保存的是那些需要同步添加到具有“Common”属性的Mode中的`Source/Timer/Observer`集合


### CFRunLoopModeRef 类

系统默认注册了5个Mode：
* kCFRunLoopDefaultMode: App的默认 Mode，通常主线程是在这个 Mode 下运行的。
* UITrackingRunLoopMode: 界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响。
* UIInitializationRunLoopMode: 在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用。
* GSEventReceiveRunLoopMode: 接受系统事件的内部 Mode，通常用不到。
* kCFRunLoopCommonModes: 这是一个占位的 Mode，没有实际作用。

值得注意的是`kCFRunLoopCommonModes`就是被标记为“Common”属性的Mode，可以看出这种模式不是一种真正的模式，仅仅是标识其他模式是否需要同步添加`_commonModeItems`中的`Source/Timer/Observer`。

系统默认将`kCFRunLoopDefaultMode`和`UITrackingRunLoopMode`添加到了`_commonModes`中，即标识为“Common”属性，所以当`RunLoop`运行在这两种模式中会自动同步添加`_commonModeItems`中的`Source/Timer/Observer`。


在[这里](http://iphonedevwiki.net/index.php/CFRunLoop)看到更多的苹果内部的 Mode，但那些 Mode 在开发中就很难遇到了


### CFRunLoopTimerRef 类

CFRunLoopTimerRef是定时源，是基于时间的触发器。当其加入到 RunLoop 时，RunLoop会注册对应的时间点，当时间点到时，RunLoop会被唤醒以执行那个回调，基本上就是NSTimer。

通过对上面三个类的理解，我们可以解释一个关于NSTime的问题

在`ViewController.m`添加下面的代码并运行，每两秒打印“***run***”，但是当我们滑动`UITableView`的时候，会发现定时器在滑动期间，计时停滞了。

```
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.
    // 添加一个tableView
    [self.view addSubview:self.myTableV];
    // 定义一个定时器，
    NSTimer *timer = [NSTimer timerWithTimeInterval:2.0 target:self selector:@selector(run) userInfo:nil repeats:YES];
    // 将定时器添加到当前RunLoop的NSDefaultRunLoopMode下
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
    
}

- (void)run{
    NSLog(@"***run***");
}
```

造成上面问题的原因：
程序刚开始运行的时候，`RunLoop`处于`NSDefaultRunLoopMode`下。当我们滑动的时候，`RunLoop`就结束`NSDefaultRunLoopMode`，切换到了`UITrackingRunLoopMode`模式下，而这个模式下没有添加`NSTimer`，所以`NSTimer`就不工作了。当停止滑动的时候，`RunLoop`就结束了`UITrackingRunLoopMode`模式，又切换回`NSDefaultRunLoopMode`模式，所以NSTimer就又开始正常工作了。


解决方案：
1、将`NSTimer`也添加到`UITrackingRunLoopMode `中，但是方式很不优雅
2、通过上面`CFRunLoopRef `和`CFRunLoopModeRef `的介绍，我们知道只需将`[[NSRunLoop currentRunLoop] addTimer:timer forMode: NSDefaultRunLoopMode];`中的`NSDefaultRunLoopMode `修改为`NSRunLoopCommonModes`也可以解决。因为将`NSTimer`加入到`NSRunLoopCommonModes`中就是把其加入到`_commonModeItems`集合中，这样在滑动时就会自动同步添加`NSTimer`到`UITrackingRunLoopMode`模式下，所以定时器也可以得到执行。



### CFRunLoopSourceRef 类
CFRunLoopSourceRef是事件源，CFRunLoopSourceRef有两种分类方法。

**按照官方文档来分类**

* Port-Based Sources（基于端口）
* Custom Input Sources（自定义）
* Cocoa Perform Selector Sources


**按照函数调用栈来分类**

* Source0 ：非基于Port，它并不能主动触发事件。使用时，你需要先调用` CFRunLoopSourceSignal(source)`，将这个 `Source` 标记为待处理，然后手动调用 `CFRunLoopWakeUp(runloop)` 来唤醒 `RunLoop`，让其处理这个事件。
* Source1：基于Port，被用于通过内核和其他线程相互发送消息。这种 `Source` 能主动唤醒 `RunLoop` 的线程，其回调函数为 `__IOHIDEventSystemClientQueueCallback()`


其实上面的这两种分类方式没有区别，只不过第一种是通过官方理论来分类，第二种是在实际应用中通过调用函数来分类。
下面我们了解一下函数调用栈和Source。
在`ViewController`中新建一个`UIButton`，然后在其触发事件中打断点，在xcode左侧可以看到
![](https://upload-images.jianshu.io/upload_images/1426173-b2768bc1138965e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

点击事件为：首先程序启动，调用17行的`main`函数，`main`函数调用16行`UIApplicationMain`函数，然后一直往上调用函数，最终调用到0行的`clickMethod`函数，即点击函数。

需要注意的一点是我们在12行看到的是`Sources0`，是否意味着点击事件是属于`Sources0`函数的，点击事件就是在`Sources0`中处理的呢？

其实还是系统注册的那个 `source1` 接收`IOHIDEvent`，只是之后在回调 `__IOHIDEventSystemClientQueueCallback() `内触发的 `Source0`，`Source0`再触发的 `_UIApplicationHandleEventQueue()`，所以`UIButton`事件看到是在 `Source0 `内的。


当 RunLoop 进行回调时，一般都是通过一个很长的函数调用出去 (call out), 当你在你的代码中下断点调试时，通常能在调用栈上看到这些函数。下面是这几个函数的整理版本，如果你在调用栈中看到这些长函数名，在这里查找一下就能定位到具体的调用地点了：

```
{
    /// 1. 通知Observers，即将进入RunLoop
    /// 此处有Observer会创建AutoreleasePool: _objc_autoreleasePoolPush();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopEntry);
    do {
 
        /// 2. 通知 Observers: 即将触发 Timer 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeTimers);
        /// 3. 通知 Observers: 即将触发 Source (非基于port的,Source0) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeSources);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 4. 触发 Source0 (非基于port的) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 6. 通知Observers，即将进入休眠
        /// 此处有Observer释放并新建AutoreleasePool: _objc_autoreleasePoolPop(); _objc_autoreleasePoolPush();
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeWaiting);
 
        /// 7. sleep to wait msg.
        mach_msg() -> mach_msg_trap();
        
 
        /// 8. 通知Observers，线程被唤醒
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopAfterWaiting);
 
        /// 9. 如果是被Timer唤醒的，回调Timer
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(timer);
 
        /// 9. 如果是被dispatch唤醒的，执行所有调用 dispatch_async 等方法放入main queue 的 block
        __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(dispatched_block);
 
        /// 9. 如果如果Runloop是被 Source1 (基于port的) 的事件唤醒了，处理这个事件
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);
 
 
    } while (...);
 
    /// 10. 通知Observers，即将退出RunLoop
    /// 此处有Observer释放AutoreleasePool: _objc_autoreleasePoolPop();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopExit);
}

```


### CFRunLoopObserverRef 类
CFRunLoopObserverRef是观察者，每个 Observer 都包含了一个回调（函数指针），当 RunLoop 的状态发生变化时，观察者就能通过回调接受到这个变化。可以观测的时间点有以下几个：

```
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry         = (1UL << 0), // 即将进入Loop 1
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer 2
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source 4
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠 32
    kCFRunLoopAfterWaiting  = (1UL << 6), // 刚从休眠中唤醒 64
    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop 128
    kCFRunLoopAllActivities = 0x0FFFFFFFU // 监听全部状态改变  

}; 
```
上面的 `Source/Timer/Observer `被统称为 `mode item`，一个 `item `可以被同时加入多个 `mode`。但一个 `item` 被重复加入同一个 `mode` 时是不会有效果的。如果一个` mode` 中一个 `item` 都没有，则 `RunLoop` 会直接退出，不进入循环。

## RunLoop 的内部逻辑
根据[苹果文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW23)里的说明，**RunLoop内部逻辑图：**
![](https://upload-images.jianshu.io/upload_images/1426173-82f55b1eb78592bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**RunLoop的具体顺序为：**
1、通知观察者RunLoop已经启动
2、通知观察者即将要开始的Timer
3、通知观察者任何非基于端口的Source0即将触发
4、触发上步准备好的非基于端口的Source0
5、如果有Source1准备好并处于等待状态，立即启动；并进入步骤9
6、通知观察者RunLoop的线程即将进入休眠状态
7、调用 mach_msg 等待接受 mach_port 的消息，线程将进入休眠直到被下面某一个事件唤醒：
	* 一个基于 port 的Source 的事件
	* 一个 Timer 到时间了
	* RunLoop 自身的超时时间到了
	* 被其他什么调用者手动唤醒
8、通知观察者RunLoop的线程刚刚被唤醒
9、处理唤醒时未处理的事件：
	* 如果用户定义的Timer 到时间了，处理定时器事件并重启RunLoop。进入步骤2
	* 如果输入源激活，传递相应的消息
	* 如果RunLoop被显示唤醒而且时间还没超时，重启RunLoop。进入步骤2

10、通知观察者RunLoop即将退出。

RunLoop 的核心就是一个 mach_msg() ，RunLoop 调用这个函数去接收消息，如果没有别人发送 port 消息过来，内核会将线程置于等待状态。

**其内部代码整理如下：**
```
/// 用DefaultMode启动
void CFRunLoopRun(void) {
    CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
}
 
/// 用指定的Mode启动，允许设置RunLoop超时时间
int CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean stopAfterHandle) {
    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);
}
 
/// RunLoop的实现
int CFRunLoopRunSpecific(runloop, modeName, seconds, stopAfterHandle) {
    
    /// 首先根据modeName找到对应mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(runloop, modeName, false);
    /// 如果mode里如果没有input source 与 timer就会返回。就算有observer也是会返回的。
    if (__CFRunLoopModeIsEmpty(currentMode)) return;
    
    /// 1. 通知 Observers: RunLoop 即将进入 loop。
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopEntry);
    
    /// 内部函数，进入loop
    __CFRunLoopRun(runloop, currentMode, seconds, returnAfterSourceHandled) {
        
        Boolean sourceHandledThisLoop = NO;
        int retVal = 0;
        do {
 
            /// 2. 通知 Observers: RunLoop 即将触发 Timer 回调。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeTimers);
            /// 3. 通知 Observers: RunLoop 即将触发 Source0 (非port) 回调。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeSources);
            /// 执行被加入的block
            __CFRunLoopDoBlocks(runloop, currentMode);
            
            /// 4. RunLoop 触发 Source0 (非port) 回调。
            sourceHandledThisLoop = __CFRunLoopDoSources0(runloop, currentMode, stopAfterHandle);
            /// 执行被加入的block
            __CFRunLoopDoBlocks(runloop, currentMode);
 
            /// 5. 如果有 Source1 (基于port) 处于 ready 状态，直接处理这个 Source1 然后跳转去处理消息。
            if (__Source0DidDispatchPortLastTime) {
                Boolean hasMsg = __CFRunLoopServiceMachPort(dispatchPort, &msg)
                if (hasMsg) goto handle_msg;
            }
            
            /// 6. 通知 Observers: RunLoop 的线程即将进入休眠(sleep)。
            if (!sourceHandledThisLoop) {
                __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);
            }
            
            /// 7. 调用 mach_msg 等待接受 mach_port 的消息。线程将进入休眠, 直到被下面某一个事件唤醒。
            /// • 一个基于 port 的Source 的事件。
            /// • 一个 Timer 到时间了
            /// • RunLoop 自身的超时时间到了
            /// • 被其他什么调用者手动唤醒
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort) {
                mach_msg(msg, MACH_RCV_MSG, port); // thread wait for receive msg
            }
 
            /// 8. 通知 Observers: RunLoop 的线程刚刚被唤醒了。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopAfterWaiting);
            
            /// 收到消息，处理消息。
            handle_msg:
 
            /// 9.1 如果一个 Timer 到时间了，触发这个Timer的回调。
            if (msg_is_timer) {
                __CFRunLoopDoTimers(runloop, currentMode, mach_absolute_time())
            } 
 
            /// 9.2 如果有dispatch到main_queue的block，执行block。
            else if (msg_is_dispatch) {
                __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            } 
 
            /// 9.3 如果一个 Source1 (基于port) 发出事件了，处理这个事件
            else {
                CFRunLoopSourceRef source1 = __CFRunLoopModeFindSourceForMachPort(runloop, currentMode, livePort);
                sourceHandledThisLoop = __CFRunLoopDoSource1(runloop, currentMode, source1, msg);
                if (sourceHandledThisLoop) {
                    mach_msg(reply, MACH_SEND_MSG, reply);
                }
            }
            
            /// 执行加入到Loop的block
            __CFRunLoopDoBlocks(runloop, currentMode);
            
 
            if (sourceHandledThisLoop && stopAfterHandle) {
                /// 进入loop时参数说处理完事件就返回。
                retVal = kCFRunLoopRunHandledSource;
            } else if (timeout) {
                /// 超出传入参数标记的超时时间了
                retVal = kCFRunLoopRunTimedOut;
            } else if (__CFRunLoopIsStopped(runloop)) {
                /// 被外部调用者强制停止了
                retVal = kCFRunLoopRunStopped;
            } else if (__CFRunLoopModeIsEmpty(runloop, currentMode)) {
                /// source/timer/observer一个都没有了
                retVal = kCFRunLoopRunFinished;
            }
            
            /// 如果没超时，mode里没空，loop也没被停止，那继续loop。
        } while (retVal == 0);
    }
    
    /// 10. 通知 Observers: RunLoop 即将退出。
    __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
}
```



## 苹果用 RunLoop 实现的功能


### AutoreleasePool

App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。

第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。


### 事件响应
苹果注册了一个 Source1 (基于 mach port 的) 用来接收系统事件，其回调函数为 __IOHIDEventSystemClientQueueCallback()。

当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。这个过程的详细情况可以参考[这里](http://iphonedevwiki.net/index.php/IOHIDFamily)。SpringBoard 只接收按键(锁屏/静音等)，触摸，加速，接近传感器等几种 Event，随后用 mach port 转发给需要的App进程。随后苹果注册的那个 Source1 就会触发回调，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。

_UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。通常事件比如 UIButton 点击、touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的。

### 手势识别
当上面的 _UIApplicationHandleEventQueue() 识别了一个手势时，其首先会调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断。随后系统将对应的 UIGestureRecognizer 标记为待处理。

苹果注册了一个 Observer 监测 BeforeWaiting (Loop即将进入休眠) 事件，这个Observer的回调函数是 _UIGestureRecognizerUpdateObserver()，其内部会获取所有刚被标记为待处理的 GestureRecognizer，并执行GestureRecognizer的回调。

当有 UIGestureRecognizer 的变化(创建/销毁/状态改变)时，这个回调都会进行相应处理。

### 界面更新
当在操作 UI 时，比如改变了 Frame、更新了 UIView/CALayer 的层次时，或者手动调用了 UIView/CALayer 的 setNeedsLayout/setNeedsDisplay方法后，这个 UIView/CALayer 就被标记为待处理，并被提交到一个全局的容器去。

苹果注册了一个 Observer 监听 BeforeWaiting(即将进入休眠) 和 Exit (即将退出Loop) 事件，回调去执行一个很长的函数：
_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()。这个函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面。

这个函数内部的调用栈大概是这样的：

```
_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()
    QuartzCore:CA::Transaction::observer_callback:
        CA::Transaction::commit();
            CA::Context::commit_transaction();
                CA::Layer::layout_and_display_if_needed();
                    CA::Layer::layout_if_needed();
                        [CALayer layoutSublayers];
                            [UIView layoutSubviews];
                    CA::Layer::display_if_needed();
                        [CALayer display];
                            [UIView drawRect];
```


### 定时器
NSTimer 其实就是 CFRunLoopTimerRef，他们之间是 toll-free bridged 的。一个 NSTimer 注册到 RunLoop 后，RunLoop 会为其重复的时间点注册好事件。例如 10:00, 10:10, 10:20 这几个时间点。RunLoop为了节省资源，并不会在非常准确的时间点回调这个Timer。Timer 有个属性叫做 Tolerance (宽容度)，标示了当时间点到后，容许有多少最大误差。

如果某个时间点被错过了，例如执行了一个很长的任务，则那个时间点的回调也会跳过去，不会延后执行。就比如等公交，如果 10:10 时我忙着玩手机错过了那个点的公交，那我只能等 10:20 这一趟了。

CADisplayLink 是一个和屏幕刷新率一致的定时器（但实际实现原理更复杂，和 NSTimer 并不一样，其内部实际是操作了一个 Source）。如果在两次屏幕刷新之间执行了一个长任务，那其中就会有一帧被跳过去（和 NSTimer 相似），造成界面卡顿的感觉。在快速滑动TableView时，即使一帧的卡顿也会让用户有所察觉。Facebook 开源的 AsyncDisplayLink 就是为了解决界面卡顿的问题，其内部也用到了 RunLoop，这个稍后我会再单独写一页博客来分析。



### PerformSelecter
当调用 NSObject 的 performSelecter:afterDelay: 后，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。

当调用 performSelector:onThread: 时，实际上其会创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效


### 关于网络请求
iOS 中，关于网络请求的接口自下至上有如下几层:

```
CFSocket
CFNetwork       ->ASIHttpRequest
NSURLConnection ->AFNetworking
NSURLSession    ->AFNetworking2, Alamofire
```

• CFSocket 是最底层的接口，只负责 socket 通信。
• CFNetwork 是基于 CFSocket 等接口的上层封装，ASIHttpRequest 工作于这一层。
• NSURLConnection 是基于 CFNetwork 的更高层的封装，提供面向对象的接口，AFNetworking 工作于这一层。
• NSURLSession 是 iOS7 中新增的接口，表面上是和 NSURLConnection 并列的，但底层仍然用到了 NSURLConnection 的部分功能 (比如 com.apple.NSURLConnectionLoader 线程)，AFNetworking2 和 Alamofire 工作于这一层。


下面主要介绍下 NSURLConnection 的工作过程。(iOS9已弃用，使用NSURLSession)

通常使用 NSURLConnection 时，你会传入一个 Delegate，当调用了 [connection start] 后，这个 Delegate 就会不停收到事件回调。实际上，start 这个函数的内部会会获取 CurrentRunLoop，然后在其中的 DefaultMode 添加了4个 Source0 (即需要手动触发的Source)。CFMultiplexerSource 是负责各种 Delegate 回调的，CFHTTPCookieStorage 是处理各种 Cookie 的。

当开始网络传输时，我们可以看到 NSURLConnection 创建了两个新线程：com.apple.NSURLConnectionLoader 和 com.apple.CFSocket.private。其中 CFSocket 线程是处理底层 socket 连接的。NSURLConnectionLoader 这个线程内部会使用 RunLoop 来接收底层 socket 的事件，并通过之前添加的 Source0 通知到上层的 Delegate。

![](https://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_network.png)

NSURLConnectionLoader 中的 RunLoop 通过一些基于 mach port 的 Source 接收来自底层 CFSocket 的通知。当收到通知后，其会在合适的时机向 CFMultiplexerSource 等 Source0 发送通知，同时唤醒 Delegate 线程的 RunLoop 来让其处理这些通知。CFMultiplexerSource 会在 Delegate 线程的 RunLoop 对 Delegate 执行实际的回调。





### 事件的响应的根源：
事件的来源是 runloop 注册了一个基于 mach port 的 source1。当发生一个硬件事件时，会生成 IOHIDEvent 对象并注册一个source0
当Runloop被唤醒后处理 Source0，此时回调 __handleHIDEventFetcherDrain() 再转调 __handleEventQueueInternal() 到 __dispatchPreprocessedEventFromEventQueue，来处理并包装成 UIEvent 进行处理和分发。
对于手势识别：当上面的事件进行到 UIWindow 时，window 识别了一个手势，于是向 UIGestureEnvironment 分派该 UIEvent，UIGestureEnvironment 将找到合适的手势对象并发送 Action。
而在之后的一系列相关的Event都将被侦测并转而调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断。
随后系统将对应的 UIGestureRecognizer 标记为待处理。苹果注册了一个 Observer 监测 BeforeWaiting (Loop即将进入休眠) 事件，这个Observer的回调函数是 _UIGestureRecognizerUpdateObserver()，其内部会获取所有刚被标记为待处理的 GestureRecognizer，并执行GestureRecognizer的回调。
当有 UIGestureRecognizer 的变化(创建/销毁/状态改变)时，这个回调都会进行相应处理。



[runloop视频（sunny-孙源）](https://v.youku.com/v_show/id_XODgxODkzODI0.html)

[深入理解RunLoop（YY大神-郭曜源）](https://blog.ibireme.com/2015/05/18/runloop/)

[RunLoop官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)


[解密 Runloop](http://mrpeak.cn/blog/ios-runloop/)

[iOS底层原理探究-Runloop](https://juejin.im/post/5af590c5f265da0b7964f1c2)