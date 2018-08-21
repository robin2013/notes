#CHECK\_FOR\_FORK()

##写在前面

`CHECK_FOR_FORK()`函数在RunLoop的源代码里有很多地方都用到了, 具体是个什么意思呢?
`CHECK_FOR_FORK()`在`CFRunLoop.m`的定义分为Mac OS(以及嵌入式系统)和其他, 其中的实现逻辑大相径庭


##对于Mac OS和嵌入式系统:

系统的定义:

```
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
extern uint8_t __CF120293;
extern uint8_t __CF120290;
extern void __THE_PROCESS_HAS_FORKED_AND_YOU_CANNOT_USE_THIS_COREFOUNDATION_FUNCTIONALITY___YOU_MUST_EXEC__(void);

#define CHECK_FOR_FORK() do { __CF120290 = true; if (__CF120293) __THE_PROCESS_HAS_FORKED_AND_YOU_CANNOT_USE_THIS_COREFOUNDATION_FUNCTIONALITY___YOU_MUST_EXEC__(); } while (0)
#define CHECK_FOR_FORK_RET(...) do { CHECK_FOR_FORK(); if (__CF120293) return __VA_ARGS__; } while (0)
#define HAS_FORKED() (__CF120293)
#endif

```

__note__:
`DEPLOYMENT_TARGET_EMBEDDED ` 和 `DEPLOYMENT_TARGET_EMBEDDED_MINI `具体的意思找了很久, 都没有很明白的解释, 字面的意思`嵌入式`, 可参见 [TARGET Embedded](http://wiki.freepascal.org/TARGET_Embedded)

###1. 变量说明:

- \_\_CF120290 控制\_\_CF120293是否可以赋值的开关
- \_\_CF120291 多进程标识
- \_\_CF120293 进程fork标识

涉及到的方法:

```
//多进程判断
static void __01121__(void) {
    __CF120291 = pthread_is_threaded_np() ? true : false;
}

// 设置进程fork标识
static void __01123__(void) {
    // Ideally, child-side atfork handlers should be async-cancel-safe, as fork()
    // is async-cancel-safe and can be called from signal handlers.  See also
    // http://standards.ieee.org/reading/ieee/interp/1003-1c-95_int/pasc-1003.1c-37.html
    // This is not a problem for CF.
    if (__CF120290) {
	__CF120293 = true;
#if DEPLOYMENT_TARGET_MACOSX
	if (__CF120291) {
	    CRSetCrashLogMessage2("*** multi-threaded process forked ***");
	} else {
	    CRSetCrashLogMessage2("*** single-threaded process forked ***");
	}
#endif
    }
}
```

以上两个放在`RunTime`的`__CFInitialize()`方法中, 通过`pthread_atfork(__01121__, NULL, __01123__);`设置进程`fork`前后要执行的方法, 其中`__01121__ `在fork前调用, 而`__01123__ `在fork后的子进程中调用.

[pthread_atfork解读](https://blog.csdn.net/codinghonor/article/details/43737869)

###2.`__THE_PROCESS_HAS_FORKED_AND_YOU_CANNOT_USE_THIS_COREFOUNDATION_FUNCTIONALITY___YOU_MUST_EXEC__`方法


```
#define EXEC_WARNING_STRING_1 "The process has forked and you cannot use this CoreFoundation functionality safely. You MUST exec().\n"
#define EXEC_WARNING_STRING_2 "Break on __THE_PROCESS_HAS_FORKED_AND_YOU_CANNOT_USE_THIS_COREFOUNDATION_FUNCTIONALITY___YOU_MUST_EXEC__() to debug.\n"

CF_PRIVATE void __THE_PROCESS_HAS_FORKED_AND_YOU_CANNOT_USE_THIS_COREFOUNDATION_FUNCTIONALITY___YOU_MUST_EXEC__(void) {
    write(2, EXEC_WARNING_STRING_1, sizeof(EXEC_WARNING_STRING_1) - 1);
    write(2, EXEC_WARNING_STRING_2, sizeof(EXEC_WARNING_STRING_2) - 1);
//    HALT;
}
```

这个名字长到令人发指的函数, 其实只干了一件事, 就是向fd=2的句柄(代表打开了一个文件), 写入定义的提示信息, 告诉用户"进程已经被克隆了, 当前的CoreFoundation方法已经不再安全了, 请执行一下exec()置换进程".

[Linux系统函数write（strlen、sizeof与write结合使用的区别](https://blog.csdn.net/sinat_25457161/article/details/48572033)

###3. 总结
CHECK\_FOR\_FORK()的作用就是打开设置开关`__CF120290`, 允许fork进程时, 在子进程中修改`__CF120293`;

当函数正在执行时, 一旦打开`__CF120290`开关, 后边的`if`又捕捉到`__CF120293 `为`true`, 说明进程进行了`fork`, 子进程的方法调用已经不再安全了(fork之后子进程同时运行, 使用的还是父进程的数据, 可能存在问题), 此时, 调用`__THE_PROCESS_HAS_FORKED_AND_YOU_CANNOT_USE_THIS_COREFOUNDATION_FUNCTIONALITY___YOU_MUST_EXEC__`方法打印一个提示信息.

[Threading Programming Guide ](https://link.juejin.im/?target=https%3A%2F%2Fdeveloper.apple.com%2Flibrary%2Fcontent%2Fdocumentation%2FCocoa%2FConceptual%2FMultithreading%2FAboutThreads%2FAboutThreads.html%23%2F%2Fapple_ref%2Fdoc%2Fuid%2F10000057i-CH6-SW2)中有一段描述:

```
Warning: When launching separate processes using the fork function, you must 
always follow a call to fork with a call to exec or a similar function. 
Applications that depend on the Core Foundation, Cocoa, or Core Data frameworks 
(either explicitly or implicitly) must make a subsequent call to an exec function 
or those frameworks may behave improperly.

```

运行一个调用了fork方法的游离进程(应该指的是子进程), 必须紧接着调用一个`exec`或者类似的方法(用于置换子进程), 依赖 Core Foundation, Cocoa, or Core Data 的应用(不管直接或者间接引用)必须执行`exec`或类似操作, 否则这些类库可能无法正常运行.

懵逼了, 感觉云里雾里, 可以看一下`fork`和`exec`的用法, 这个牵涉太多就不在展开, 有兴趣的看这里:
	
[RunLoop 源码阅读](https://juejin.im/post/5aaa15d36fb9a028d82b7d83)

[linux进程之fork 和 exec函数](https://www.cnblogs.com/chris-cp/p/3525070.html)

[程序员必备知识——fork和exec函数详解](https://blog.csdn.net/bad_good_man/article/details/49364947)

[fork和exec的区别](https://blog.csdn.net/xiaofei0859/article/details/77342173)

[Linux 进程概念](https://www.jianshu.com/p/d0bbec93e3eb)

[操作系统之 fork() 函数详解](https://www.jianshu.com/p/484af1700176)

##其他 OS定义:

```

#if !defined(CHECK_FOR_FORK)
#define CHECK_FOR_FORK() do { } while (0)
#endif

#if !defined(CHECK_FOR_FORK_RET)
#define CHECK_FOR_FORK_RET(...) do { } while (0)
#endif

#if !defined(HAS_FORKED)
#define HAS_FORKED() 0
#endif

```
关于这样的定义有什么用呢?

我觉的答案就是没用, 只是因为Runloop只是跨平台的, 为了在代码上兼容.
iOS App 只有一个进程, 你只能创建线程([参见](https://stackoverflow.com/questions/12088155/fork-failed-1-operation-not-permitted))

`do {...} while (0)`的作用就是: 使用`do{...}while(0)`构造后的宏定义不会受到大括号、分号等的影响，总是会按你期望的方式调用运行;

但是这里传递的是空方法体, 所以可能就没毛用了;


[do {...} while (0) 在宏定义中的作用](http://www.cnblogs.com/lanxuezaipiao/p/3535626.html)


另外:

[What's the meaning of CHECK_FOR_FORK?](https://stackoverflow.com/questions/47260563/whats-the-meaning-of-check-for-fork)
感觉说的不太对,CHECK\_FOR\_FORK()是没有任何返回值的.