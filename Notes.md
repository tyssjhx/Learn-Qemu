标记：<span style="background:red; font-weight:bold; color:white">架构相关</span>，<span style="background:green; font-weight:bold; color:white">架构无关</span>

<div style="font-size:3em; text-align:right;">2020.10.16</div>

# C可变参数的使用，以及实现

读源码时读到了C可变参数的相关内容，之前不是很了解C语言可变参数的使用以及实现，整理一下。

## C可变参数的使用方法

可变参数，即函数的参数数量可变。定义函数时候参数列表可以出现“...”来表示不确定数量的参数。具体使用的一个例子如下：

```c
double average(int num,...)
{
 
    va_list valist;
    double sum = 0.0;
    int i;
 
    /* 为 num 个参数初始化 valist */
    va_start(valist, num);
 
    /* 访问所有赋给 valist 的参数 */
    for (i = 0; i < num; i++)
    {
       sum += va_arg(valist, int);
    }
    /* 清理为 valist 保留的内存 */
    va_end(valist);
 
    return sum/num;
}
```

不确定参数数量的函数能工作的一个关键在于如何获取函数调用时的调用者赋予的参数。C语言在头文件stdrag.h中提供了函数va_start、va_arg、va_end以及变量类型va_list来实现参数的获取。

```c
void va_start(va_list ap, last_arg);
type va_arg(va_list ap, type);
void va_end(va_list ap);
```

va_start的第一个参数表示可变参数，第二个参数为函数形参列表中"..."前的最后一个参数。该函数的功能为初始化变量ap。

va_arg用来获得下一个参数的值，type为该参数的类型，并且将ap指向下一个参数。

va_end用来

https://blog.csdn.net/linyt/article/details/2243605

<div style="font-size:3em; text-align:right;">2020.10.29</div>

# 读Qemu源码

读Qemu源码，用户级模拟。

``cpu_exec``函数里两个``while``循环判断条件分别使用函数``cpu_handle_exception``和函数``cpu_handle_interrpu``来判断是否需要处理exception和interrupt。

``cpu_handle_exception``函数返回一个布尔值。判断了cpu->exception_index的值是否小于0，字面意思是**异常号**，小于0则说明应该没有exception？判断内的``replay_has_exception ``函数应该还没有完善，直接返回了``false``。

下一个block判断cpu->exception_index是否大于等于``EXCP_INTERRUPT``，小于0的上面已经返回false。应该是处理大于等于0且大于等于``EXCP_INTERRUPT``的情况，返回``true``后就会退出``cpu_exec``函数，处理时把``cpu->exception_index``赋值给了``ret``，作为cpu_exec的返回值返回了；那else部分处理的应该就是大于等于0且小于``EXCP_INTERRUPT``的情况，这里以user mode only分为了两种情况，最后应该都是通过``do_interrupt``函数来处理exception的。

以hello函数作为例子，观察进入``cpu_handle_exception``的运行过程。

第一次cpu->exception_index小于0，return false。

第二次cpu->exception_index等于128。调用``cc->do_interrupt``处理，之后将``cpu->exception_index ``赋为-1，消除这个标记，return true。

输出Hello World。

第三次cpu->exception_index小于0，return false。

第四次cpu->exception_index等于128。调用``cc->do_interrupt``处理，之后将``cpu->exception_index ``赋为-1，消除这个标记，return true。

那``cpu_handle_interrupt``呢？

第一次return false。

Hello World

第二次return false。

在这个hello函数执行的过程中``cpu_handle_interrupt``貌似没起到什么作用。

看看循环体内部吧。

<div style="font-size:3em; text-align:right;">2020.11.02</div>

程序hello在函数``cpu_exec.c``中的执行流程

setjmp->while(!cpu_handle_exception)->while(!cpu_handle_interruption)->tb_find->cpu_loop_exec_tb->setjmp之后的处理部分->while(!cpu_handle_exception)->退出cpu_exec.c

Hello World

setjmp->while(!cpu_handle_exception)->while(!cpu_handle_interruption)->tb_find->cpu_loop_exec_tb->setjmp之后的处理部分->while(!cpu_handle_exception)->退出cpu_exec.c

这是两个系统调用的处理流程（int0x80）

下面重点看一下两个函数吧

``tb_find()``和``cpu_loop_exec_tb``

``tb_find()``的功能应该是查找翻译后的代码块，假如还没有翻译，应该进行翻译（源代码->tcg）。

``cpu_loop_exec_tb()``的功能应该是执行翻译

弄清楚源代码->tcg->目标代码的过程，和这个过程间的地址转换关系。

``qemu -d op filename`` 可以输出tcg

<img src=".\figures\tb_find.png" alt="tb_find" style="zoom: 50%;" />

接下来仔细读一下``tb_lookup__cpu_state``，重点看查找tb的过程，以及缓存、哈希表是怎么组织的；仔细读一下函数``tb_gen_code``，重点查看由原代码生成tcg代码的过程。看一下函数``tb_add_jump``，了解tb之间的链接是怎么建立的。

<div style="font-size:3em; text-align:right;">2020.11.04</div>

读``tb_lookup__cpu_state``

<img src=".\figures\tb_lookup__cpu_state.png" alt="tb_lookup__cpu_state" style="zoom:50%;" />

可以由cpu访问到cache的头指针，查找cache可以用``atomic_rcu_read``，更改cache可以用``atomic_set``，cache应该是简单地由数组组织起来的。问题：cache的初始化？cpu的初始化？

还有tb结构和tcg代码并不是在一起存储的，即tb结构中不包含翻译过的tcg代码，tcg代码的组织方式？

``cpu_get_tb_cpu_state``函数应该是架构相关的，定义在``/target/i386/cpu.h``里，函数功能是给一些变量赋值。

```c
    *cs_base = env->segs[R_CS].base;
    *pc = *cs_base + env->eip;
    *flags = env->hflags |
        (env->eflags & (IOPL_MASK | TF_MASK | RF_MASK | VM_MASK | AC_MASK));
```

应该和和架构的的内存管理机制有关系。

这里可以补一下x86内存管理机制。

<div style="font-size:3em; text-align:right;">2020.11.05</div>



# 方法论

把宏观微观联系起来，在微观中找到宏观特点的所指。

