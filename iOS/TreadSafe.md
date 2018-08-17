#线程安全
##前言
[百度百科](https://baike.baidu.com/item/线程安全/9747724?fr=aladdin):线程安全是多线程编程时的计算机程序代码中的一个概念。在拥有共享数据的多条线程并行执行的程序中，线程安全的代码会通过同步机制保证各个线程都可以正常且正确的执行，不会出现数据污染等意外情况

说白了, 线程安全就是在多线程的情况下, 数据的输出完全符合预期, 且没有偏差, 保证线程安全, 也就是保证任务间的同步.

##原子性

```
所谓原子操作是指不会被线程调度机制打断的操作；这种操作一旦开始，就一直运行到结束，中间不会有任何 context switch
```
也就是说, 加入任务具有原子性, 这个任务在执行的过程中, 是不可以被中断, 期间所持有的资源, 也不可以被其他任务获得, 说白了就是爷们儿运行时候你Y的只能看着, 就是这么牛.

```
具有原子性的汇编指令也可以称作原子操作
```
代码分割到最小的单位就是单句汇编指令, 但是系统中断可以中止正在执行的命令,在汇编语言层面上，提供了LOCK指令前缀来保护指令执行过程层中的数据安全

```
lock addl $0x1 %r8d
```

除此之外，在80486指令集中还有xadd、cmpxchg和xchg等指令是多处理器安全的。加了lock修饰的单条编译指令以及这些特殊的安全指令才算是真正的原子操作.

###atomic
在iOS中, atomic属性内部的锁称为自旋锁, 消耗大量资源(有点类似读写锁的功能)

```
static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy) 
{
   if (offset == 0) {
       object_setClass(self, newValue);
       return;
   }

   id oldValue;
   id *slot = (id*) ((char*)self + offset);

   if (copy) {
       newValue = [newValue copyWithZone:nil];
   } else if (mutableCopy) {
       newValue = [newValue mutableCopyWithZone:nil];
   } else {
       if (*slot == newValue) return;
       newValue = objc_retain(newValue);
   }

   if (!atomic) {
       oldValue = *slot;
       *slot = newValue;
   } else {
       spinlock_t& slotlock = PropertyLocks[slot];
       slotlock.lock();
       oldValue = *slot;
       *slot = newValue;        
       slotlock.unlock();
   }

   objc_release(oldValue);
}

void objc_setProperty(id self, SEL _cmd, ptrdiff_t offset, id newValue, BOOL atomic, signed char shouldCopy) 
{
   bool copy = (shouldCopy && shouldCopy != MUTABLE_COPY);
   bool mutableCopy = (shouldCopy == MUTABLE_COPY);
   reallySetProperty(self, _cmd, newValue, offset, atomic, copy, mutableCopy);
}

```

`atomic`所说的线程安全只是保证`getter`和`setter`方法的线程安全，并不能保证整个对象是线程安全

```
//AtomTest.m

#import "AtomTest.h"
@interface AtomTest()
@property(atomic, assign) NSInteger age;
@end
- (void)test {
    NSInteger temp = 0;
    for (int i = 0 ; i< 10; i++) {
        temp = temp + i;
    }
    for (int i = 0 ; i< 10; i++) {
        temp = temp + i;
    }
    NSLog(@"*** result should be %lu", temp);

    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_async(queue, ^{
        for (int i = 0 ; i< 10; i++) {
            dispatch_async(queue, ^{
                self.age = self.age + i;
                NSLog(@"*** Age in A is %lu", self.age);
            });
        }
    });

    dispatch_async(queue, ^{
        for (int i = 0 ; i< 10; i++) {
            dispatch_async(queue, ^{
                self.age = self.age + i;
                NSLog(@"--- Age in B is %lu", self.age);
            });
        }
    });
}@end

```

输出:

```
2018-08-10 17:43:38.620899+0800 ThreadSafe[19526:403317] *** result should be 90
2018-08-10 17:43:38.621084+0800 ThreadSafe[19526:403369] *** Age in A is 0
2018-08-10 17:43:38.621103+0800 ThreadSafe[19526:403368] *** Age in A is 1
2018-08-10 17:43:38.621116+0800 ThreadSafe[19526:403370] *** Age in A is 3
2018-08-10 17:43:38.621132+0800 ThreadSafe[19526:403367] *** Age in A is 6
2018-08-10 17:43:38.621175+0800 ThreadSafe[19526:403382] *** Age in A is 10
2018-08-10 17:43:38.621205+0800 ThreadSafe[19526:403383] *** Age in A is 15
2018-08-10 17:43:38.621228+0800 ThreadSafe[19526:403384] *** Age in A is 21
2018-08-10 17:43:38.621258+0800 ThreadSafe[19526:403385] *** Age in A is 28
2018-08-10 17:43:38.621285+0800 ThreadSafe[19526:403386] *** Age in A is 36
2018-08-10 17:43:38.621305+0800 ThreadSafe[19526:403387] *** Age in A is 45
2018-08-10 17:43:38.621340+0800 ThreadSafe[19526:403388] --- Age in B is 45
2018-08-10 17:43:38.621382+0800 ThreadSafe[19526:403389] --- Age in B is 46
2018-08-10 17:43:38.621435+0800 ThreadSafe[19526:403391] --- Age in B is 49
2018-08-10 17:43:38.621437+0800 ThreadSafe[19526:403390] --- Age in B is 51
2018-08-10 17:43:38.621478+0800 ThreadSafe[19526:403392] --- Age in B is 55
2018-08-10 17:43:38.621509+0800 ThreadSafe[19526:403393] --- Age in B is 60
2018-08-10 17:43:38.621550+0800 ThreadSafe[19526:403395] --- Age in B is 67
2018-08-10 17:43:38.621574+0800 ThreadSafe[19526:403394] --- Age in B is 73
2018-08-10 17:43:38.621774+0800 ThreadSafe[19526:403397] --- Age in B is 90
2018-08-10 17:43:38.621749+0800 ThreadSafe[19526:403398] --- Age in B is 82
```
运算20次, 结果居然是82, 为什么呢?

将`age`声明为`atomic`, `age`的`getter`和`setter`是原子操作, 但是`self.age = self.age + i;`不是原子操作, 这行赋值的代码至少包含读取(load)，+1(add)，赋值(store)三步操作，当前线程store的时候可能其他线程已经执行了若干次store了(每次运行的结果都不一样, 最后一个也可能是90 也可能不是, 因为此时线程不能如预期一样运行, 是不安全的).

[iOS多线程到底不安全在哪里？](http://mrpeak.cn/blog/ios-thread-safety/)

[iOS中的锁——由属性atomic想到的线程安全](https://www.jianshu.com/p/0b40a63c6436)

[Objective-C 中，atomic原子性一定是安全的吗？](https://www.cnblogs.com/hlgbys/p/5492130.html)

##@synchronized

@synchronized也是一种常用的加锁机制, 但是内部如何实现? 加的又是什么锁呢?

根据网上的资料:

```
@synchronized(obj) {
    // do work
}
```

大概会被转化成:

```
@try {
    objc_sync_enter(obj);
    // do work
} @finally {
    objc_sync_exit(obj);
}
```

关于`objc_sync_enter `和`objc_sync_exit `定义, 可以查看[这里](https://opensource.apple.com/source/objc4/objc4-646/runtime/objc-sync.mm):

```
// Begin synchronizing on 'obj'. 
// Allocates recursive mutex associated with 'obj' if needed.
// Returns OBJC_SYNC_SUCCESS once lock is acquired.  
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



// End synchronizing on 'obj'. 
// Returns OBJC_SYNC_SUCCESS or OBJC_SYNC_NOT_OWNING_THREAD_ERROR
int objc_sync_exit(id obj)
{
    int result = OBJC_SYNC_SUCCESS;
    
    if (obj) {
        SyncData* data = id2data(obj, RELEASE); 
        require_action_string(data != NULL, done, result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR, "id2data failed");
        
        result = recursive_mutex_unlock(&data->mutex);
        require_noerr_string(result, done, "mutex_unlock failed");
    } else {
        // @synchronized(nil) does nothing
    }
	
done:
    if ( result == RECURSIVE_MUTEX_NOT_LOCKED )
         result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;

    return result;
}
```
到此我们可以知道, @synchronized内部添加的是一个递归锁(` Allocates recursive mutex associated with 'obj' if needed.`)

`SyncData `的定义:

```
typedef struct SyncData {
    struct SyncData* nextData;
    id               object;
    int              threadCount;  // number of THREADS using this block
    recursive_mutex_t        mutex;
} SyncData;

typedef struct {
    SyncData *data;
    unsigned int lockCount;  // number of times THIS THREAD locked this block
} SyncCacheItem;

typedef struct SyncCache {
    unsigned int allocated;
    unsigned int used;
    SyncCacheItem list[0];
} SyncCache;

typedef struct {
    SyncData *data;
    spinlock_t lock;

    char align[64 - sizeof (spinlock_t) - sizeof (SyncData *)];
} SyncList __attribute__((aligned(64)));

// Use multiple parallel lists to decrease contention among unrelated objects.
#define COUNT 16
#define HASH(obj) ((((uintptr_t)(obj)) >> 5) & (COUNT - 1))
#define LOCK_FOR_OBJ(obj) sDataLists[HASH(obj)].lock
#define LIST_FOR_OBJ(obj) sDataLists[HASH(obj)].data
static SyncList sDataLists[COUNT];
```

[关于@synchronized 比你想知道的还多](https://www.jianshu.com/p/7026de47dc75)

##信号量和互斥锁

定义:

- 互斥锁

互斥锁应当是排它的，意思是锁在被某个线程获取之后，只有获取锁的线程才能释放这个锁。其他线程必须等到获取锁的线程不再拥有锁之后，才能继续执行。在我使用NSLock的测试中，发现可以unlock其他线程的锁，因此严格来说NSLock并不适合被称作互斥锁

- 信号量

信号量拥有比互斥锁更多的用途。当信号量的value大于0时，所有的线程都能访问临界资源。在线程进入临界区后，value减一，反之亦然。如果信号量初始化为0时，可以看做是等待任务执行完成而非资源保护。value的操作应当是采用原子操作来保证指令的安全性的

区别:

- 信号量用在多线程多任务同步的，一个线程完成了某一个动作就通过信号量告诉别的线程，别的线程再进行某些动作（大家都在sem_wait的时候，就阻塞在 那里）。

- 互斥锁是用在多线程多任务互斥的，一个线程占用了某一个资源，那么别的线程就无法访问，直到这个线程unlock，其他的线程才开始可以利用这个资源


###互斥
[百度百科](https://baike.baidu.com/item/线程互斥/5539748?fr=aladdin):  线程互斥是指某一资源同时只允许一个访问者对其进行访问，具有唯一性和排它性。但互斥无法限制访问者对资源的访问顺序，即访问是无序的

互斥锁（Mutex）是在原子操作API的基础上实现的信号量行为。互斥锁不能进行递归锁定或解锁，能用于交互上下文但是不能用于中断上下文，同一时间只能有一个任务持有互斥锁，而且只有这个任务可以对互斥锁进行解锁。当无法获取锁时，线程进入睡眠等待状态。

互斥锁是信号量的特例。

###自旋

[百度百科](https://baike.baidu.com/item/自旋锁/9137985): 自旋锁是专为防止多处理器并发而引入的一种锁，它在内核中大量应用于中断处理等部分（对于单处理器来说，防止中断处理中的并发可简单采用关闭中断的方式，即在标志寄存器中关闭/打开中断标志位，不需要自旋锁）。

自旋锁不会引起调用者睡眠，如果自旋锁已经被别的执行单元保持，调用者就一直循环在那里看是否该自旋锁的保持者已经释放了锁，"自旋"一词就是因此而得名。

自旋锁的初衷就是：在短期间内进行轻量级的锁定。一个被争用的自旋锁使得请求它的线程在等待锁重新可用的期间进行自旋(特别浪费处理器时间)，所以自旋锁不应该被持有时间过长。如果需要长时间锁定的话, 最好使用信号量。

缺点:
- 自旋锁实际上是忙等锁
- 自旋锁可能导致系统死锁

自旋锁的基本形式如下：

```
spin_lock(&mr_lock);

//临界区
spin_unlock(&mr_lock)
```

### 信号

[百度百科](https://baike.baidu.com/item/信号量): 信号量(Semaphore)，有时被称为信号灯，是在多线程环境下使用的一种设施，是可以用来保证两个或多个关键代码段不被并发调用

信号量的初始值表示有多少个任务可以同时访问共享资源，如果初始值为1，表示只有1个任务可以访问，信号量变成互斥锁（Mutex）。但是互斥锁和信号量又有所区别，互斥锁的加锁和解锁必须在同一线程里对应使用，所以互斥锁只能用于线程的互斥；信号量可以由一个线程释放，另一个线程得到，所以信号量可以用于线程的同步。

描述:

以一个停车场的运作为例。简单起见，假设停车场只有三个车位，一开始三个车位都是空的。这时如果同时来了五辆车，看门人允许其中三辆直接进入，然后放下车拦，剩下的车则必须在入口等待，此后来的车也都不得不在入口处等待。这时，有一辆车离开停车场，看门人得知后，打开车拦，放入外面的一辆进去，如果又离开两辆，则又可以放入两辆，如此往复。

在这个停车场系统中，车位是公共资源，每辆车好比一个线程，看门人起的就是信号量的作用。

信号的性能在自旋和互斥之间，通常的性能表现总是仅次于自旋。

根据超时时间的设置，信号量最终会表现为互斥或者自旋的方式实现。

##如何选择锁

###互斥锁和信号量比较

- 互斥锁功能上基本与二元信号量一样，但是互斥锁占用空间比信号量小，运行效率比信号量高。所以，如果要用于线程间的互斥，优先选择互斥锁。
        
###互斥锁和自旋锁比较
- 互斥锁在无法得到资源时，内核线程会进入睡眠阻塞状态，而自旋锁处于忙等待状态。因此，如果资源被占用的时间较长，使用互斥锁较好，因为可让CPU调度去做其它进程的工作。

- 如果被保护资源需要睡眠的话，那么只能使用互斥锁或者信号量，不能使用自旋锁。而互斥锁的效率又比信号量高，所以这时候最佳选择是互斥锁。

- 中断里面不能使用互斥锁，因为互斥锁在获取不到锁的情况下会进入睡眠，而中断是不能睡眠的。



##参考:

[linux互斥锁简介(内核态)](https://blog.csdn.net/silent123go/article/details/52760492)

[互斥锁的实现](https://blog.csdn.net/hzhzh007/article/details/6532988)
