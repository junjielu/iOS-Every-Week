# Run Loop - 陆俊杰

## Key Points

### Run Loop对外接口的结构和作用

Run Loop对外的接口有五个:

- **CFRunLoopRef**
	
	对`Run Loop`对象的引用。
	
- **CFRunLoopModeRef**

	对`Run Loop Mode`的引用。

- **CFRunLoopSourceRef**

	对`Run Loop Source`的引用。

- **CFRunLoopTimerRef**

	对`Run Loop Timer`的引用。

- **CFRunLoopObserverRef**

	对`Run Loop Observer`的引用。
	
一个 `RunLoop` 包含若干个 Mode，每个 Mode 又包含若干个 `Source`/`Timer`/`Observer`。每次调用 `RunLoop` 的主函数时，只能指定其中一个 Mode，这个Mode被称作 `CurrentMode`。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 `Source`/`Timer`/`Observer`，让其互不影响。

---

`Mode`是Run Loop的具体类型：

- **NSDefaultRunLoopMode**
- **NSTrackingRunLoopMode**
- **NSRunLoopCommonModes**

更换Mode会退出旧的Run Loop进入新的Run Loop

---

`Timer`即我们理解的计时器，具体上层封装为`NSTimer`、`CADisplayLink`以及一些方法如：

```
- (void)performSelector:(SEL)aSelector withObject:(id)arg afterDelay:(NSTimeInterval)delay inModes:(NSArray *)modes
```

GCD的计时器并不通过Run Loop实现，而是通过Run Loop监听GCD的消息发送来实现

---

`Observer`是通过KVO实现的观察者，负责观察`Run Loop`的状态并报告给外部。状态枚举为：

```
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
	kCFRunLoopEntry = (1UL << 0)
	kCFRunLoopBeforeTimers = (1UL << 1)
	kCFRunLoopBeforeSources = (1UL << 2)
	kCFRunLoopBeforeWaiting = (1UL << 5)
	kCFRunLoopAfterWaiting = (1UL << 6)
	kCFRunLoopExit = (1UL << 7)
}
```

---

`Source`是`Run Loop`的数据源抽象类，定义了两种`Source`：
- `Source0`：处理App内部事件，如UIEvent。
- `Source1`：由内核管理，`Mach port`驱动,进程之间通讯的数据端口。

### Run Loop的内部逻辑

见Run Loop的迭代执行顺序。

### 理解Mach Port

`Mach Port`可以理解为内核端口，提供了进程间通讯和处理器调度的功能。使用Mach要注意的一点是`Mach`的对象间不能直接调用，只能通过消息传递的方式实现对象间的通信。

一条 Mach 消息是一个二进制数据包，其头部定义了当前端口 `local_port` 和目标端口 `remote_port`，通过`mach_msg()`来进行消息的发送。

> 发送消息：

> ```
> natural_t data;
> mach_port_t port;

> struct {
    mach_msg_header_t header;
    mach_msg_body_t body;
    mach_msg_type_descriptor_t type;
> } message;

> message.header = (mach_msg_header_t) {
    .msgh_remote_port = port,
    .msgh_local_port = MACH_PORT_NULL,
    .msgh_bits = MACH_MSGH_BITS(MACH_MSG_TYPE_COPY_SEND, 0),
    .msgh_size = sizeof(message)
> };

> message.body = (mach_msg_body_t) {
    .msgh_descriptor_count = 1
> };

> message.type = (mach_msg_type_descriptor_t) {
    .pad1 = data,
    .pad2 = sizeof(data)
> };

> mach_msg_return_t error = mach_msg_send(&message.header);

> if (error == MACH_MSG_SUCCESS) {
    // ...
> }
> ```


> 接收消息：
> 
> ```
> mach_port_t port;

> struct {
    mach_msg_header_t header;
    mach_msg_body_t body;
    mach_msg_type_descriptor_t type;
    mach_msg_trailer_t trailer;
> } message;

> mach_msg_return_t error = mach_msg_receive(&message.header);

> if (error == MACH_MSG_SUCCESS) {
    natural_t data = message.type.pad1;
    // ...
> }
> ```

在内核基础上封装的`CFMachPort` / `NSMachPort`可以用做`Run Loop`源

> 使用`CFMessagePort`接收Run Loop的Source消息并执行回调
> 
> ```
> static CFDataRef Callback(CFMessagePortRef port,
                          SInt32 messageID,
                          CFDataRef data,
                          void *info)
> {
    // ...
> }

> CFMessagePortRef localPort =
    CFMessagePortCreateLocal(nil,
                             CFSTR("com.example.app.port.server"),
                             Callback,
                             nil,
                             nil);

> CFRunLoopSourceRef runLoopSource =
    CFMessagePortCreateRunLoopSource(nil, localPort, 0);

> CFRunLoopAddSource(CFRunLoopGetCurrent(),
                   runLoopSource,
                   kCFRunLoopCommonModes);
> ```

> 发送消息
> 
> ```
> CFDataRef data;
> SInt32 messageID = 0x1111; // Arbitrary
> CFTimeInterval timeout = 10.0;

> CFMessagePortRef remotePort =
    CFMessagePortCreateRemote(nil,
                              CFSTR("com.example.app.port.client"));

> SInt32 status =
    CFMessagePortSendRequest(remotePort,
                             messageID,
                             data,
                             timeout,
                             timeout,
                             NULL,
                             NULL);
if (status == kCFMessagePortSuccess) {
    // ...
> }
> ```

### Run Loop的迭代执行顺序

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
    /// 如果mode里没有source/timer/observer, 直接返回。
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
            
            /// 通知 Observers: RunLoop 的线程即将进入休眠(sleep)。
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

### Apple使用Run Loop实现的功能

---

#### **AutoreleasePool**：
	
App启动后，苹果在主线程 `RunLoop` 里注册了两个 `Observer`，其回调都是 `_wrapRunLoopWithAutoreleasePoolHandler()`。

第一个 `Observer` 监视的事件是 `Entry`，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 `Observer` 监视了两个事件： `BeforeWaiting`时调用`_objc_autoreleasePoolPop()` 和 `_objc_autoreleasePoolPush()` 释放旧的池并创建新池；`Exit`时调用 `_objc_autoreleasePoolPop()` 来释放自动释放池。其优先级最低，保证其释放池子发生在其他所有回调之后。

---

#### **事件响应**

苹果注册了一个 `Source1` 用来接收系统事件，其回调函数为 `__IOHIDEventSystemClientQueueCallback()`。

当一个硬件事件(触摸/锁屏/摇晃等)发生后，用 `mach_port` 发送给需要的App进程，`Source1` 就会触发回调，并调用 `_UIApplicationHandleEventQueue()` 进行应用内部的分发。

---

#### **界面更新**

当在进行UI操作时，比如改变了 Frame、更新了 `UIView`/`CALayer` 的层次时，或者手动调用了 `UIView`/`CALayer` 的 `setNeedsLayout`/`setNeedsDisplay`方法后，这个 UIView/CALayer 就被标记为待处理，并被提交到一个全局的容器去。

苹果注册了一个 `Observer` 监听 `BeforeWaiting` 和 `Exit`  事件，回调去执行一个很长的函数：
`_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()`。这个函数里会遍历所有待处理的 `UIView`/`CAlayer` 以执行实际的绘制和调整，并更新UI界面。

---

#### **计时器**

`NSTimer` 其实就是 `CFRunLoopTimerRef`。一个 `NSTimer` 注册到 `RunLoop` 后，`RunLoop` 会为其重复的时间点注册好事件。

---

### 如何在实际操作中去使用Run Loop

实践请阅读`AFNetworking`源码中关于创建常驻自定义线程及`Run Loop`的实践，这里贴一下代码：

```
+ (void)networkRequestThreadEntryPoint:(id)__unused object {
    @autoreleasepool {
        [[NSThread currentThread] setName:@"AFNetworking"];
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}
 
+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });
    return _networkRequestThread;
}
```


## Questions

### 如何让Run Loop挂起or唤醒？

通过`mach_port`调用等待消息，此时如果没有别人发送 port 消息过来，内核会将线程置于等待状态，从而将Run Loop挂起。

通过指定用于唤醒的`mach_port`端口，向其发送msg，唤醒trap状态使Run Loop被唤醒。

> 一般来说，唤醒是通过其他线程向端口发送msg来唤醒Run Loop的

### 为什么NSTimer完全依赖于Run Loop？GCD的计时器和Run Loop有关系吗？

GCD的计时器实现并不基于`Run Loop`，但却和Run Loop结合地很紧密。GCD中dispatch到main queue的block会被分发到main run loop进行调用，

### 主线程的函数是通过Run Loop哪些函数调起的？

- **__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__**
- **__CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__**
- **__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__**
- **__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__**
- **__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_CALLBACK_FUNCTION__**
- **__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_CALLBACK_FUNCTION__**

### source0和source1有什么区别？

- `Source0` 包含了一个回调，它并不能主动触发事件。使用时，先调用 `CFRunLoopSourceSignal(source)`，将这个 `Source` 标记为待处理，然后手动调用 `CFRunLoopWakeUp(runloop)` 来唤醒 `RunLoop`，让其处理这个事件。

- `Source1` 包含了一个 `mach_port` 和一个回调，被用于通过内核和其他线程相互发送消息。

### 解释一下Run Loop的几种mode。

- **NSDefaultRunLoopMode**
	
	处理除了`NSConnection`之外的其他输入数据源的`Mode`，一般状态下的`RunLoop`均为这种`Mode`
	
- **UITrackingRunLoopMode**
	
	处理跟踪事件的`Mode`，可以使用这种Mode去创建Timer并在跟踪过程中销毁，常见的滑动操作即为`UITrackingRunLoopMode `

- **NSRunLoopCommonModes**

	`NSDefaultRunLoopMode`、`UITrackingRunLoopMode `和自定义的被标记 _Common_ 属性的Mode的集合。每当 RunLoop 的内容发生变化时，RunLoop 都会自动将 _commonModeItems 里的 Source/Observer/Timer 同步到具有 "Common" 标记的所有Mode里。

### 一个thread能对应几个Run Loop？

一个`thread`只对应一个`Run Loop`，然而`Run Loop`是可以嵌套的，所以可以对应无数个`Run Loop`

### Observer的CFRunLoopActivity的枚举分别代表了什么状态？

```
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry         = (1UL << 0), // 即将进入Loop
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠
    kCFRunLoopAfterWaiting  = (1UL << 6), // 刚从休眠中唤醒
    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop
};
```

### 从Run Loop角度解释一下autorelease对象在何时释放？

通过`Observer`在两次`Run Loop`的两次sleep间对`AutoreleasePool`进行`Pop`和`Push`，将loop中产生的`autorelease`对象释放。