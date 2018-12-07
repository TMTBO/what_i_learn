# 锁相关

## 锁性能

![加解锁速度](./resources/lock_benchmark.png)

**注:**
加解锁的速度只表示锁的复杂程度,不代表锁的效率

## 自旋锁实现原理

自旋锁的实现思路很简单，理论上来说只要定义一个全局变量，用来表示锁的可用情况即可，伪代码如下:

```
bool lock = false; // 一开始没有锁上，任何线程都可以申请锁  
do {  
    while(lock); // 如果 lock 为 true 就一直死循环，相当于申请锁
    lock = true; // 挂上锁，这样别的线程就无法获得锁
        Critical section  // 临界区
    lock = false; // 相当于释放锁，这样别的线程可以进入临界区
        Reminder section // 不需要锁保护的代码        
}
```

可惜这段代码存在一个问题: 如果一开始有多个线程同时执行 while 循环，他们都不会在这里卡住，而是继续执行，这样就无法保证锁的可靠性了。解决思路也很简单，只要确保申请锁的过程是原子操作即可。

### 原子操作

狭义上的原子操作表示一条不可打断的操作，也就是说线程在执行操作过程中，不会被操作系统挂起，而是一定会执行完。在单处理器环境下，一条汇编指令显然是原子操作，因为中断也要通过指令来实现。
然而在多处理器的情况下，能够被多个处理器同时执行的操作任然算不上原子操作。因此，真正的原子操作必须由硬件提供支持，比如 x86 平台上如果在指令前面加上 “LOCK” 前缀，对应的机器码在执行时会把总线锁住，使得其他 CPU不能再执行相同操作，从而从硬件层面确保了操作的原子性。
这些非常底层的概念无需完全掌握，我们只要知道上述申请锁的过程，可以用一个原子性操作 test_and_set 来完成，它用伪代码可以这样表示:

```
bool test_and_set (bool *target) {  
    bool rv = *target; 
    *target = TRUE; 
    return rv;
}
```

这段代码的作用是把 target 的值设置为 1，并返回原来的值。当然，在具体实现时，它通过一个原子性的指令来完成。

### 自旋锁总结

如果临界区的执行时间过长，使用自旋锁不是个好主意。之前我们介绍过时间片轮转算法，线程在多种情况下会退出自己的时间片。其中一种是用完了时间片的时间，被操作系统强制抢占。除此以外，当线程进行 I/O 操作，或进入睡眠状态时，都会主动让出时间片。显然在 while 循环中，线程处于忙等状态，白白浪费 CPU 时间，最终因为超时被操作系统抢占时间片。如果临界区执行时间较长，比如是文件读写，这种忙等是毫无必要的。

## 信号量

信号量 dispatch_semaphore_t 的实现原理，它最终会调用到 sem_wait 方法，这个方法在 glibc 中被实现如下:

```
int sem_wait (sem_t *sem) {  
  int *futex = (int *) sem;
  if (atomic_decrement_if_positive (futex) > 0)
    return 0;
  int err = lll_futex_wait (futex, 0);
    return -1;
	)
```

首先会把信号量的值减一，并判断是否大于零。如果大于零，说明不用等待，所以立刻返回。具体的等待操作在 lll_futex_wait 函数中实现，lll 是 low level lock 的简称。这个函数通过汇编代码实现，调用到 SYS_futex 这个系统调用，使线程进入睡眠状态，主动让出时间片，这个函数在互斥锁的实现中，也有可能被用到。
主动让出时间片并不总是代表效率高。让出时间片会导致操作系统切换到另一个线程，这种上下文切换通常需要 10 微秒左右，而且至少需要两次切换。如果等待时间很短，比如只有几个微秒，忙等就比线程睡眠更高效。
可以看到，自旋锁和信号量的实现都非常简单，这也是两者的加解锁耗时分别排在第一和第二的原因。再次强调，加解锁耗时不能准确反应出锁的效率(比如时间片切换就无法发生)，它只能从一定程度上衡量锁的实现复杂程度。

## pthread_mutex

pthread 表示 POSIX thread，定义了一组跨平台的线程相关的 API，pthread_mutex 表示互斥锁。互斥锁的实现原理与信号量非常相似，不是使用忙等，而是阻塞线程并睡眠，需要进行上下文切换。

互斥锁的常见用法如下:

```
pthread_mutexattr_t attr;  
pthread_mutexattr_init(&attr);  
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_NORMAL);  // 定义锁的属性

pthread_mutex_t mutex;  
pthread_mutex_init(&mutex, &attr) // 创建锁

pthread_mutex_lock(&mutex); // 申请锁  
    // 临界区
pthread_mutex_unlock(&mutex); // 释放锁  
```

## 互斥锁的实现

对于 pthread_mutex 来说，它的用法和之前没有太大的改变，比较重要的是锁的类型，可以有 PTHREAD_MUTEX_NORMAL、PTHREAD_MUTEX_ERRORCHECK、PTHREAD_MUTEX_RECURSIVE 等等，具体的特性就不做解释了，网上有很多相关资料。

一般情况下，一个线程只能申请一次锁，也只能在获得锁的情况下才能释放锁，多次申请锁或释放未获得的锁都会导致崩溃。假设在已经获得锁的情况下再次申请锁，线程会因为等待锁的释放而进入睡眠状态，因此就不可能再释放锁，从而导致死锁。

然而这种情况经常会发生，比如某个函数申请了锁，在临界区内又递归调用了自己。辛运的是 pthread_mutex 支持递归锁，也就是允许一个线程递归的申请锁，只要把 attr 的类型改成 PTHREAD_MUTEX_RECURSIVE 即可。

## NSLock

NSLock 是 Objective-C 以对象的形式暴露给开发者的一种锁，它的实现非常简单，通过宏，定义了 lock 方法:

```
#define    MLOCK \
- (void) lock\
{\
  int err = pthread_mutex_lock(&_mutex);\
  // 错误处理 ……
}
```

NSLock 只是在内部封装了一个 pthread_mutex，属性为 PTHREAD_MUTEX_ERRORCHECK，它会损失一定性能换来错误提示。

这里使用宏定义的原因是，OC 内部还有其他几种锁，他们的 lock 方法都是一模一样，仅仅是内部 pthread_mutex 互斥锁的类型不同。通过宏定义，可以简化方法的定义。

NSLock 比 pthread_mutex 略慢的原因在于它需要经过方法调用，同时由于缓存的存在，多次方法调用不会对性能产生太大的影响。

## NSCodnition

## NSRecursiveLock

上文已经说过，递归锁也是通过 pthread_mutex_lock 函数来实现，在函数内部会判断锁的类型，如果显示是递归锁，就允许递归调用，仅仅将一个计数器加一，锁的释放过程也是同理。

NSRecursiveLock 与 NSLock 的区别在于内部封装的 pthread_mutex_t 对象的类型不同，前者的类型为 PTHREAD_MUTEX_RECURSIVE。

## @synchronized

这其实是一个 OC 层面的锁， 主要是通过牺牲性能换来语法上的简洁与可读。

我们知道 @synchronized 后面需要紧跟一个 OC 对象，它实际上是把这个对象当做锁来使用。这是通过一个哈希表来实现的，OC 在底层使用了一个互斥锁的数组(你可以理解为锁池)，通过对对象去哈希值来得到对应的互斥锁。

### @synchronized 实现细节

* Questions:

 1. 锁是如何与你传入 @synchronized 的对象关联上的？
 2. @synchronized会保持（retain，增加引用计数）被锁住的对象么？
 3. 假如你传入 @synchronized 的对象在 @synchronized 的 block 里面被释放或者被赋值为 nil 将会怎么样？

@synchronized 的文档告诉我们 @synchronized block 在被保护的代码上暗中添加了一个异常处理。为的是同步某对象时如若抛出异常，锁会被释放掉。

SO 上的这篇帖子 说 @synchronized block 会变成 objc_sync_enter 和 objc_sync_exit 的成对儿调用。我们不知道这些函数是干啥的，但基于这些事实我们可以认为编译器将这样的代码：

```
@synchronized(obj) {
    // do work
}
```

转化成这样:

```
@try {
    objc_sync_enter(obj);
    // do work
} @finally {
    objc_sync_exit(obj);
}
```

`objc_sync_enter` 和 `objc_sync_exit` 是什么鬼？它们是如何实现的？在 `Xcode` 中按住 `Command` 键单击它们，然后进到了 `<objc/objc-sync.h>`，里面有我们感兴趣的这两个函数：

```
/** 
 * Begin synchronizing on 'obj'.  
 * Allocates recursive pthread_mutex associated with 'obj' if needed.
 * 
 * @param obj The object to begin synchronizing on.
 * 
 * @return OBJC_SYNC_SUCCESS once lock is acquired.  
 */
OBJC_EXPORT  int objc_sync_enter(id obj)
    __OSX_AVAILABLE_STARTING(__MAC_10_3, __IPHONE_2_0);

/** 
 * End synchronizing on 'obj'. 
 * 
 * @param obj The objet to end synchronizing on.
 * 
 * @return OBJC_SYNC_SUCCESS or OBJC_SYNC_NOT_OWNING_THREAD_ERROR
 */
OBJC_EXPORT  int objc_sync_exit(id obj)
    __OSX_AVAILABLE_STARTING(__MAC_10_3, __IPHONE_2_0);
```

文件底部的一句话提醒着我们：苹果工程师也是人啊哈哈

```
// The wait/notify functions have never worked correctly and no longer exist.
OBJC_EXPORT  int objc_sync_wait(id obj, long long milliSecondsMaxWait) 
    UNAVAILABLE_ATTRIBUTE;
OBJC_EXPORT  int objc_sync_notify(id obj) 
    UNAVAILABLE_ATTRIBUTE;
OBJC_EXPORT  int objc_sync_notifyAll(id obj) 
    UNAVAILABLE_ATTRIBUTE;
```

不过，objc_sync_enter 的文档告诉我们一些新东西： @synchronized 结构在工作时为传入的对象分配了一个递归锁。分配工作何时发生，如何发生呢？它怎样处理 nil？幸运的是 Objective-C runtime 是开源的，所以我们可以马上阅读源码并找到答案！

注：递归锁在被同一线程重复获取时不会产生死锁。你可以在这找到一个它工作原理的精巧案例。有个叫做 NSRecursiveLock 的现成的类也是这样的，你可以试试。

你可以在[这里](http://www.opensource.apple.com/source/objc4/objc4-646/runtime/objc-sync.mm)找到 objc-sync 的全部源码，但我要带着你看源码，让你屌的飞起。我们先从文件顶部的数据结构开始看。在代码块的下方我将立刻做出解释，所以尝试理解代码时别花太长时间哦。

```
typedef struct SyncData {
    id object;
    recursive_mutex_t mutex;
    struct SyncData* nextData;
    int threadCount;
} SyncData;

typedef struct SyncList {
    SyncData *data;
    spinlock_t lock;
} SyncList;

// Use multiple parallel lists to decrease contention among unrelated objects.
#define COUNT 16
#define HASH(obj) ((((uintptr_t)(obj)) >> 5) & (COUNT - 1))
#define LOCK_FOR_OBJ(obj) sDataLists[HASH(obj)].lock
#define LIST_FOR_OBJ(obj) sDataLists[HASH(obj)].data
static SyncList sDataLists[COUNT];
```

一开始，我们有一个 struct SyncData 的定义。这个结构体包含一个 object（嗯就是我们给 @synchronized 传入的那个对象）和一个有关联的 recursive_mutex_t，它就是那个跟 object 关联在一起的锁。每个 SyncData 也包含一个指向另一个 SyncData 对象的指针，叫做 nextData，所以你可以把每个 SyncData 结构体看做是链表中的一个元素。最后，每个 SyncData 包含一个 threadCount，这个 SyncData 对象中的锁会被一些线程使用或等待，threadCount 就是此时这些线程的数量。它很有用处，因为 SyncData 结构体会被缓存，threadCount==0 就暗示了这个 SyncData 实例可以被复用。

下面是 struct SyncList 的定义。正如我在上面提过，你可以把 SyncData 当做是链表中的节点。每个 SyncList 结构体都有个指向 SyncData 节点链表头部的指针，也有一个用于防止多个线程对此列表做并发修改的锁。

上面代码块的最后一行是 sDataLists 的声明 - 一个 SyncList 结构体数组，大小为16。通过定义的一个哈希算法将传入对象映射到数组上的一个下标。值得注意的是这个哈希算法设计的很巧妙，是将对象指针在内存的地址转化为无符号整型并右移五位，再跟 0xF 做按位与运算，这样结果不会超出数组大小。 LOCK_FOR_OBJ(obj) 和 LIST_FOR_OBJ(obj) 这俩宏就更好理解了，先是哈希出对象的数组下标，然后取出数组对应元素的 lock 或 data。一切都是这么顺理成章哈。

当你调用 objc_sync_enter(obj) 时，它用 obj 内存地址的哈希值查找合适的 SyncData，然后将其上锁。当你调用 objc_sync_exit(obj) 时，它查找合适的 SyncData 并将其解锁。

噢耶！现在我们知道了 @synchronized 如何将一个锁和你正在同步的对象关联起来，我希望聊聊当一个对象在 @synchronized block 当中被释放或设为 nil 时会发生什么。

如果你看了源码，你会注意到 objc_sync_enter 里面没有 retain 和 release。所以它要么没有保持传递给它的对象，要么或是在 ARC 下被编译。我们可以用下面的代码来做个测试：

```
NSDate *test = [NSDate date];
// This should always be `1`
NSLog(@"%@", @([test retainCount]));

@synchronized (test) {

    // This will be `2` if `@synchronized` somehow
    // retains `test`
    NSLog(@"%@", @([test retainCount]));
}
```

两次输出结果都是 1。那么 objc_sync_enter 貌似是没保持被传入的对象啊。这就有趣了。如果你正在同步的对象被释放了，然后有可能另一个新的对象在此处（被释放对象的内存地址）被分配内存。有可能某个其他的线程试着去同步那个新的对象（就是那个在被释放的旧对象的内存地址上刚刚新创建的对象）。在这种情况下，另一个线程将会阻塞，直到当前线程结束它的同步 block。这看起来并不是很糟。这听起来像是这种事情实现者早就知道并予以接受。我没有遇到过任何好的替代方案。

假如对象在 “synchronized block” 中被设成 nil 呢？我们回顾下我们“拿衣服（naive）”的实现吧：

```
NSString *test = @"test";
@try {
    // Allocates a lock for test and locks it
    objc_sync_enter(test);
    test = nil;
} @finally {
    // Passed `nil`, so the lock allocated in `objc_sync_enter`
    // above is never unlocked or deallocated
    objc_sync_exit(test);   
}
```

objc_sync_enter 被调用时传入的是 test 而 objc_sync_exit 被调用时传入的是 nil。而传入 nil 的时候 objc_sync_exit 是个空操作，所以将不会有人释放锁。这真操蛋！

如果 Objective-C 容易受这种情况的影响，我们知道么？下面的代码调用 @synchronized 并在 @synchronized block 中将一个指针设为 nil。然后在后台线程对指向同一个对象的指针调用 @synchronized。如果在 @synchronized block 中设置一个对象为 nil 会让锁死锁，那么在第二个 @synchronized 中的代码将永远不会执行。我们将不会在控制台中看见任何东西打印出来。

```
NSNumber *number = @(1);
NSNumber *thisPtrWillGoToNil = number;

@synchronized (thisPtrWillGoToNil) {
    /**
     * Here we set the thing that we're synchronizing on to `nil`. If
     * implemented naively, the object would be passed to `objc_sync_enter`
     * and `nil` would be passed to `objc_sync_exit`, causing a lock to
     * never be released.
     */
    thisPtrWillGoToNil = nil;
}

dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0), ^ {

    NSCAssert(![NSThread isMainThread], @"Must be run on background thread");

    /**
     * If, as mentioned in the comment above, the synchronized lock is never
     * released, then we expect to wait forever below as we try to acquire
     * the lock associated with `number`.
     *
     * This doesn't happen, so we conclude that `@synchronized` must deal
     * with this correctly.
     */
    @synchronized (number) {
        NSLog(@"This line does indeed get printed to stdout");
    }

});
```

当我们执行上面的代码时，那行代码确实打印到控制台了！所以 Objective-C 很好地处理了这种情形。我打赌是编译器做了类似下面的事情来解决这事儿的。

```
NSString *test = @"test";
id synchronizeTarget = (id)test;
@try {
    objc_sync_enter(synchronizeTarget);
    test = nil;
} @finally {
    objc_sync_exit(synchronizeTarget);
}
```

用这种方式实现的话，传递给 objc_sync_enter 和 objc_sync_exit 总是相同的对象。他们在传入 nil 时都是空操作。这带来了个棘手的 debug 场景：如果你向 @synchronized 传递 nil，那么你就不会得到任何锁而且你的代码将不会是线程安全的！如果你想知道为什么你正收到出乎意料的竞态（race），确保你没向你的 @synchronized 传入 nil。你可以在 objc_sync_nil 上设置一个符号断点来达到此目的。objc_sync_nil 是一个空方法，当 objc_sync_enter 函数被传入 nil 时会被调用，折让 debug 更容易些。

```
int objc_sync_enter(id obj)
{
    int result = OBJC_SYNC_SUCCESS;

    if (obj) {
        SyncData* data = id2data(obj, ACQUIRE);
        require_action_string(data != NULL, done, result = OBJC_SYNC_NOT_INITIALIZED, "id2data failed");
	
        result = recursive_mutex_lock(&data->mutex);
        require_noerr_string(result, done, "mutex_lock failed");
    } else {
        // @synchronized(nil) does nothing
        if (DebugNilSync) {
            _objc_inform("NIL SYNC DEBUG: @synchronized(nil); set a breakpoint on objc_sync_nil to debug");
        }
        objc_sync_nil();
    }

done: 
    return result;
}
```

这回答了我眼下的问题。

1. 你调用 sychronized 的每个对象，Objective-C runtime 都会为其分配一个递归锁并存储在哈希表中。
2. 如果在 sychronized 内部对象被释放或被设为 nil 看起来都 OK。不过这没在文档中说明，所以我不会再生产代码中依赖这条。
3. 注意不要向你的 sychronized block 传入 nil！这将会从代码中移走线程安全。你可以通过在 objc_sync_nil 上加断点来查看是否发生了这样的事情。
