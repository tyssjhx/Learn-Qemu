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



2020/11/06补一下IA-32的内存管理机制。





<div style="font-size:3em; text-align:right;">2020.11.05</div>

<img src=".\figures\tb_htable_lookup.png" alt="tb_htable_lookup" style="zoom:50%;" />

``tb_htable_lookup``函数用来在哈希表中查找对应的tb，这里计算了一个``phys_pc``，猜测``pc``是GVA，``phys_pc``是GPA。

``phys_pc``==-1是说明找不到物理地址？仔细看一下``get_page_addr_code``，看看地址管理。

最后用了函数``tb_htable_lookup``来在哈希表中查找tb。

参数``tb_ctx``是一个全局变量，``tb_lookup_cmp``是一个比较函数。



<div style="font-size:3em; text-align:right;">2020.11.09</div>

搞清楚``struct TBContext``的结构。

<img src=".\figures\struct TBContext.png" alt="struct TBContext" style="zoom:50%;" />

``htable``中最重要的部分应该是``struct qht_map *map``了，``map``有哈希桶头指针，可以通过偏移访问到hash值对应的哈希桶，其中``n_buckets``指明了有目前哈希桶的个数，为了防止溢出，使用了``return &map->buckets[hash & (map->n_buckets - 1)];``的方式来根据哈希值访问对应的哈希桶。

函数``qht_lookup_custom``

<img src=".\figures\qht_lookup_custom.png" alt="qht_lookup_custom" style="zoom:50%;" />

重点函数是``qht_do_lookup``，先利用这个函数寻找一个tb，之后应该是同步的操作，不是很清楚，当同步出现问题，最后用函数``qht_lookup__slowpath(b, func, userp, hash)``再次查找。

问题：**Qemu同步机制是怎么实现的**？

看一下``qht_do_lookup``函数

<img src=".\figures\qht_do_lookup.png" alt="qht_do_lookup" style="zoom:50%;" />



主题由两个循环组成，第一个循环沿着哈希桶组成的列表向后查找，第二个循环再在每个哈希桶中以哈希值为判断标准进行查找。

几个问题：

1. 这里接在*head后的哈希桶与之前查找的在数组中（线性地址排列）的哈希桶的区别？

   猜测：同一个列表中的桶中的项有同样的``hash & (map->n_buckets - 1)``值，线性地址排列的桶属于不同列表，有不同的``hash & (map->n_buckets - 1)``值。

2. 每个哈希桶中不同为什么还会有不同哈希值的项？

   猜测：应该是因为填入时也是按照这个规则映射的``&map->buckets[hash & (map->n_buckets - 1)]``

证实的话应该要看哈希表的填入过程。

``qht_lookup__slowpath(b, func, userp, hash)``函数的实现也利用了``qht_do_lookup``

<div style="font-size:3em; text-align:right;">2020.11.10</div>

<img src=".\figures\qht_lookup__slowpath.png" alt="qht_lookup__slowpath" style="zoom:50%;" />

``qht_lookup__slowpath``基本上是把之前的一次查找过程加了个do_while的循环，注释里有这样的几行字

```
    /*
     * Removing the do/while from the fastpath gives a 4% perf. increase 	  * when running a 100%-lookup microbenchmark.
     */
```

应该是说这样写会有性能提升？







终于回到了函数``tb_find``，接着读``tb_gen_code``，这个函数的功能是如果没有找到tb，级第一次执行这个tb块，就进行翻译，并返回一个tb。

进入这个函数之前和出这个函数之后在函数``tb_find``之中分别用了``mmap_lock``和``mmap_unlock``函数。

看一下函数``tb_gen_code``

这个函数好长有二百多行，慢慢看吧。

``get_page_addr_code``函数之前也出现过，没有深挖，之后可以仔细看一下？

注释里的``insn``是指令的意思。

cflags每个位都什么意思啊，还有cpu结构体里面每个域都什么意思啊

惊现``go to``语句



<div style="font-size:3em; text-align:right;">2020.11.11</div>

还是先看一下``struct TranslationBlock``吧，之前有看过师兄写的介绍，但好像因为Qemu版本不一样结构体也不太一样了，再整理一下吧

先看一下函数``tcg_tb_alloc``

它的功能是为即将要翻译的tb分配内存空间。

tb结构的内存空间``struct TranslationBlock``和代码(中间代码、翻译过的代码？)的内存空间？

问题：tb之间的连接是怎么样的？

之前已经知道了tb会被挂到``tb_jmp_cache``与``tb_htable``上面。

那么tb之间的连接是怎么样的？

<img src=".\figures\ROUND_UP.png" alt="image-20201111152459905" style="zoom:50%;" />

``#define ROUND_UP(n,d) (((n) + (d) - 1) & -(0 ? (n) : (d)))``

ROUND_UP宏的作用是向上取整，第一个参数n是想要取整的数，第二个参数d说明是多少进制下的取整。

<img src=".\figures\tcg_tb_alloc.png" alt="tcg_tb_alloc" style="zoom:50%;" />

感觉tb的组织、分配也有很多东西可以看，这里的``tcg_rgion_alloc``也可以深挖一下，这里先放过（先把常规情况走一遍），之前的`` get_page_addr_code``函数也没有向下看，在这里标记一下。



<div style="font-size:3em; text-align:right;">2020.11.13</div>

昨天咸鱼了一天，今天来继续看代码吧。

``tcg_func_start``函数主要是改变了一些``tcg_ctx``里的变量值，应该是为之后做准备吧。

``gen_intermediate_code``函数开始进入翻译阶段了。

翻译过程凭空想象的话，感觉首先要找到源代码，然后翻译为IR，之后还要放到准备好的空间中，看看具体是怎么操作的吧。

``gen_intermediate_code``函数申请了一个临时变量`` DisasContext dc``，之后就调用函数``translator_loop``。这里有架构相关的东西了，不同架构传入函数``translator_loop``的第一个参数是不一样的，i386下就是``&i386_tr_ops``，mips下就是``&mips_tr_ops``，``i386_tr_ops``和``mips_tr_ops``的类型都是``static const TranslatorOps``结构，这个结构中的每个域都是一个函数指针，``i386_tr_ops``和``mips_tr_ops``两个变量每个域初始化的函数指针的值不一样。感觉这样可以增加代码的可扩展性。

````c
void gen_intermediate_code(CPUState *cpu, TranslationBlock *tb, int max_insns)
{
    DisasContext dc;

    translator_loop(&i386_tr_ops, &dc.base, cpu, tb, max_insns);
}
````

````c
static const TranslatorOps i386_tr_ops = {
    .init_disas_context = i386_tr_init_disas_context,
    .tb_start           = i386_tr_tb_start,
    .insn_start         = i386_tr_insn_start,
    .breakpoint_check   = i386_tr_breakpoint_check,
    .translate_insn     = i386_tr_translate_insn,
    .tb_stop            = i386_tr_tb_stop,
    .disas_log          = i386_tr_disas_log,
};
````

进入函数

函数``translator_loop``是一个翻译过程的通用流程，不同架构都调用这个函数进行翻译。

函数``translator_loop``函数，以及``i386_tr_ops``里的几个函数是重点，不过有点复杂啊这个函数。



# 方法论

把宏观微观联系起来，在微观中找到宏观特点的所指。

数据结构+算法

