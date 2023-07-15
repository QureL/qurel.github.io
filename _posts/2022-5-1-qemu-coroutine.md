---
layout: post
title: "非局部转移与qemu协程分析"
date:   2022-5-1
tags: [qemu]
comments: true
author: Reynold
---

# 非局部转移

## 解析

qemu-coroutine的ucontext模式是基于setjmp和longjmp实现的，可以完成函数外部的跳转。

### setjmp示例：

```c++
#include <stdio.h>
#include <setjmp.h>

jmp_buf env;

int func1()
{
    printf("in func1\n");
    // if val not 0,then setjmp return val
    // if val is 0, then setjmp return 1
    longjmp(env, 0);
    printf("leave func1");
}

int main()
{
    int ret = 0;

    ret = setjmp(env);
    printf("point1\n");

    if (ret == 0)
    {
        func1(env);
    }
    else
    {
        printf("ret = %d\n", ret);
    }
}
```

运行结果：

```
point1
in func1
point1
ret = 1
```

setjmp第一次调用埋下一个跳转点，保存寄存器环境到env指针中，并且返回0，回到main函数中继续执行；

当调用longjmp后，longjmp会从env指针中恢复setjmp时保存的cpu寄存器环境，最关键的是，将sp寄存器值替换为setjmp时的sp值，并将setjmp的压栈的返回地址存入0(%esp)中，这样再执行ret指令后，pop出来的就是setjmp的返回地址，这样就切换回了setjmp埋下的切换点，并继续执行。

在切回后，setjmp会返回longjmp时设下的val值，如果该值为0，则返回1，借此能较好的区分到底是何时调用的setjmp。

重复调用setjmp，后次的调用会覆盖前次调用的切换点。

### 在x86_64环境下的函数声明：

```c++
 int setjmp(jmp_buf env);
 void longjmp(jmp_buf env, int val);
 typedef struct __jmp_buf_tag jmp_buf[1];
```

> The functions described on this page are used for performing "nonlocal gotos": transferring execution from one
>        function to a predetermined location in another function.  The setjmp() function dynamically  establishes  the
>        target to which control will later be transferred, and longjmp() performs the transfer of execution.

>setjmp() saves the stack context/environment in env for later use by longjmp(3)
>
>setjmp() return 0 if returning directly, and non-zero when returning from longjmp(3) using the saved context.
>
>longjmp() restores the environment saved by the last call of setjmp(3) with the corresponding env argument.

这是里一份基于x86汇编的setjmp实现源码：

```assembly
#include <machine/asm.h>
#define _JBLEN  10      /* size, in longs, of a jmp_buf */
typedef long jmp_buf[_JBLEN]; 
ENTRY(setjmp)
    PIC_PROLOGUE # 屏蔽信号，可不关注
    pushl    $0　　　　　　# sigblock的参数
#ifdef PIC
    call    PIC_PLT(_C_LABEL(sigblock))
#else
    call    _C_LABEL(sigblock)
#endif
    addl    $4,%esp　　　　# 返回上述 pushl $0时的压栈
    PIC_EPILOGUE # 以下是主要代码
    movl    4(%esp),%ecx　　　# 将env地址 -> ecx
    movl    0(%esp),%edx　　  # 返回地址 -> edx
    movl    %edx, 0(%ecx) #将edx、ebx...eax的值保存到env中
    movl    %ebx, 4(%ecx)
    movl    %esp, 8(%ecx)
    movl    %ebp,12(%ecx)
    movl    %esi,16(%ecx)
    movl    %edi,20(%ecx)
    movl    %eax,24(%ecx)　　# eax中保存的是sigblock的返回值  # setjmp返回0
    xorl    %eax,%eax
    ret
END(setjmp)# 到此时，env[0]保存返回地址, env[1]保存ebx的值, ..., env[6]保存eax的值# 下面是longjmp的代码
ENTRY(longjmp)
    movl    4(%esp),%edx　　# 将env的地址 -> edx
    PIC_PROLOGUE
    pushl    24(%edx)　　　　# env[6], 即setjmp中sigblock返回值压栈，当做sigsetmask的参数
#ifdef PIC
    call    PIC_PLT(_C_LABEL(sigsetmask))
#else
    call    _C_LABEL(sigsetmask)
#endif
    addl    $4,%esp　　　　# 恢复堆栈
    PIC_EPILOGUE# 上面 33-43行, 是为了恢复信号# 下面是longjmp的主代码
    movl    4(%esp),%edx　　# 将env地址 -> edx
    movl    8(%esp),%eax　　# val -> eax
    movl    0(%edx),%ecx　　# setjmp函数返回地址 -> ecx# 下面是恢复ebx、esp ... edi等
    movl    4(%edx),%ebx
    movl    8(%edx),%esp
    movl    12(%edx),%ebp
    movl    16(%edx),%esi
    movl    20(%edx),%edi# 下面三行确保eax（即val, longjmp返回到setjmp时的返回值）不为零# 如果eax为零则加一，否则保持原值
    testl    %eax,%eax
    jnz    1f
    incl    %eax
1:    movl    %ecx,0(%esp)#  将ecx(setjmp的返回地址) -> 放在sp指向的栈顶，这样，ret返回后，就跳转到了setjmp的返回地址
    ret
END(longjmp)
```



## 图解

setjump:

1. 初始时的stack

   ![](https://raw.githubusercontent.com/QureL/qurel.github.io/main/images/kernels/setjmp1.jpg)
   
2. 将寄存器存储进env

   ![](https://raw.githubusercontent.com/QureL/qurel.github.io/main/images/kernels/setjmp2.jpg)

longjmp:

1. 初始时的stack

    ![](https://raw.githubusercontent.com/QureL/qurel.github.io/main/images/kernels/longjmp1.jpg)

2. 从env中恢复寄存器环境，其中 

   1. env中保存的setjmp返回地址 -> ecx
   2. val -> eax
   3. 使用env中保存的栈指针替换esp，栈替换为setjmp调用时压入的栈

3. 如果val为零则+1，否则保持原值，返回val

4. ecx -> esp (栈指针，保存longjmp的返回地址)，使用setjmp的返回地址覆盖longjmp的返回地址



## 附：调用约定

![](https://raw.githubusercontent.com/QureL/qurel.github.io/main/images/kernels/调用约定.jpg)


# qemu coroutine

### 为什么引入协程

QEMU作为虚拟化软件其主体架构采用的是事件驱动模式，在main-loop中监控各种、大量的文件，事件，消息和状态的变化并进行各种操作，当大量的阻塞操作发生时，为不影响VM环境的执行效率，一般都采用异步的方式。而异步方式则需要设定callback函数的调用时机，同时保存大量的执行状态，导致逻辑代码支离破碎，复杂并难以理解。所以最好的解决方式是采用协程的方式将同步方式的代码异步化。

### 得到当前正在执行的协程：qemu_coroutine_self

这个函数可以得到当前线程正在执行的coroutine。由于我们对当前线程执行的coroutine维持了全局变量current，因此这一操作的实现是简单的。

```c
Coroutine *qemu_coroutine_self(void)
{
    if (!current) { 
        current = &leader.base;
    }
    return current; 
} 
```

### 进入一个协程：qemu_coroutine_enter

进入一个coroutine的实现代码如下：

```c
void qemu_coroutine_enter(Coroutine *co, void *opaque)
{
    Coroutine *self = qemu_coroutine_self();
    CoroutineAction ret;

    trace_qemu_coroutine_enter(self, co, opaque);

    if (co->caller) {
        fprintf(stderr, "Co-routine re-entered recursively\n");
        abort();
    }

    co->caller = self;  //协程的调用者，方便协程跳转回原协程
    co->entry_arg = opaque;  //为协程的入口函数赋参数值
    ret = qemu_coroutine_switch(self, co, COROUTINE_ENTER); //切换掉要进入的协程中

    qemu_co_queue_run_restart(co); //在协程yield或者terminate之后，需要执行协程co_wake_up队列中的协程

    switch (ret) {
    case COROUTINE_YIELD:
        return;
    case COROUTINE_TERMINATE:
        trace_qemu_coroutine_terminate(co);
        coroutine_delete(co);
        return;
    default:
        abort();
    }
}

```

### 協程的switch

```c
CoroutineAction __attribute__((noinline))
qemu_coroutine_switch(Coroutine *from_, Coroutine *to_,
                      CoroutineAction action)
{
    CoroutineUContext *from = DO_UPCAST(CoroutineUContext, base, from_);
    CoroutineUContext *to = DO_UPCAST(CoroutineUContext, base, to_);
    int ret;

    current = to_;
    //利用sigsetjmp讲协程的上下文保存
    ret = sigsetjmp(from->env, 0); //返回0則是本協程第一次埋下切換點，之後進行切換到to協程
    if (ret == 0) { 
        siglongjmp(to->env, action);  //利用siglongjmp跳转到目标协程中
    } // 返回值不爲0則説明為其他携程切換回來，直接返回
    return ret;
}
```

### 协程主动yield回其caller

```c++
void coroutine_fn qemu_coroutine_yield(void)
{
    Coroutine *self = qemu_coroutine_self();
    Coroutine *to = self->caller;

    trace_qemu_coroutine_yield(self, to);

    if (!to) {
        fprintf(stderr, "Co-routine is yielding to no one\n");
        abort();
    }

    self->caller = NULL; 
    qemu_coroutine_switch(self, to, COROUTINE_YIELD); //从self返回到to（self->caller）
}
```

sigsetjmp和siglongjmp的定义

```c++
#define sigsetjmp(env, savemask) setjmp(env)
#define siglongjmp(env, val) longjmp(env, val)
```


