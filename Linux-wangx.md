# LINUX

[TOC]

# 1 Linux C代码的编译

## 1.1 编译

C语言程序的编译包括【预处理、编译、汇编、链接】这几个步骤

完整编译的命令如下:

```bash
$ gcc hello.c -o hello
```

-o参数表示输出文件的名字。

除了完整编译之外我们还可以分步完成编译过程:

预处理

得到预处理后的文件，一般用.i表示：

```bash
$ gcc -E hello.c
```
编译

得到编译后的文件，一般用.s表示：

```bash
$ gcc -S hello.c
```

汇编

得到汇编后的文件，一般用.o表示

```bash
$ gcc -c hello.c
```

链接

链接过程由链接器自动完成，也可以手动链接

## 1.2 动态库

编译动态库的指令如下:

```bash
$ gcc hello.c -fPIC -shared -o libhello.so
```

动态库的名称有固定的格式要求需要为libxxx.so，方便后面链接的时候使用。

链接动态库的方式如下：

```bash
$ gcc main.c -L. -lhello -o main
```

-l指的是动态库名字 动态库的名字需要去掉lib前缀和.so后缀，这也是动态库在编译的时候有命名要求的原因

-L指的是libhello.so的路径

我们使用ldd命令查看main

```bash
$ ldd main
        linux-vdso.so.1 (0x00007fff1bdb1000)
        libhello.so => not found
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f0661a1c000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f066200f000)
```

当你使用动态库时需要用一下命令指定.so的路径

```bash
$ export LD_LIBRARY_PATH=`pwd`
```

我们再使用ldd命令查看main

```bash
$ ldd main-so                      
        linux-vdso.so.1 (0x00007ffda9974000)
        libhello.so => /root/libhello.so (0x00007fce3498b000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fce3459a000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fce34d8f000)
```

[动态库测试demo](#动态库测试demo)

## 1.3 静态库

编译静态库的指令如下:

先使用-c参数将库文件编译为.o文件

```bash
$ gcc hello.c -c -o hello.o
```

再使用ar将.o文件打包为静态库，如有多个可以一起打包

```bash
$ ar -rc libhello.a hello.o
```

注意打包为静态库是，库的名字必须为libxxx.a，方便后面链接使用

静态链接指令如下:

```bash
$ gcc main.c -lhello -L. -static -o main
```

[静态库测试demo](#静态库测试demo)

## 1.4 动态库和静态库链接问题

如果-L指定的目录下只存在libhello.a不存在libhello.so我们来看看执行一下两条命令生成的main的区别

```bash
$ gcc main.c -lhello -L. -static -o main
```

我们用ldd命令查看

```bash
$ ldd main
		not a dynamic executable
```

你会发现它是一个完全静态链接的库

而你如果不加-static的话会出现一下结果

```bash
$ gcc main.c -lhello -L. -o main
```

同样用ldd命令查看

```bash
$ ldd main
        linux-vdso.so.1 (0x00007ffdf9bbc000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f8c0fd46000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f8c10339000)
```

此时是因为你在最终编译main.c时仍然使用了动态链接，系统库/lib64/ld-linux-x86-64.so.2等被动态链接到了程序里，而hello.c中的函数由于我们把它编成了静态库，所以hello.c中的函数还是被静态链接到了main中。



如果在当前目录下同时存在libhello.so和libhello.a，那么如果你执行

```bash
$ gcc main.c -lhello -L. -static -o main
```

则会链接libhello.a，并且把系统库也静态链接到main中

```bash
$ ldd main
		not a dynamic executable
```

如果你执行

```bash
$ gcc main.c -lhello -L. -o main
```

则会链接libhello.so，并且把系统库也动态链接到main中

```bash
$ ldd main                    
        linux-vdso.so.1 (0x00007ffda9974000)
        libhello.so => /root/libhello.so (0x00007fce3498b000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fce3459a000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fce34d8f000)
```

## 1.5 makefile

### 1.5.1 makefile中常见参数

---

addprefix

```makefile
result = $(addprefix demo, .c .cpp)
```

等效于

```makefile
result = demo.c demo.cpp
```

---

addsuffix 

```makefile
result = $(addsuffix .c, demo1 demo2)
```

等效于

```makefile
result = demo1.c demo2.c
```

---

abspath

```makefile
result = $(abspath demo.c)
```

返回result为demo.c的绝对路径

---

if

```makefile
$(if <condition>, <then-part>,<else-part>)
```

condition参数是 if的表达式，如果其返回的为非空字符串，那么这个表达式就相当于返回真，于是，then-part会被计算，返回计算结果字符串；否则else- part会被计算，返回计算结果字符串。else-part可以省略，则表示condition为空时返回为空。

---

wildcard

```makefile
$(wildcard <pattern1 pattern2 ...>)
```

展开pattern中的通配符

```makefile
result = $(wildcard src/*)
```

result输出src目录下的文件列表

---

filter

```makefile
$(filter <pattern1 pattern2 ...>, <text>)
```

以pattern模式过滤text字符串中的单词，保留符合模式pattern的单词

```makefile
result = $(filter %.c %.cpp, $(wildcard src/*))
```

result输出src目录下所有后缀是.c和.cpp的文件序列

---

dir

从文件名序列names中取出目录部分。目录部分是指最后一个反斜杠之前的部分

```makefile
result = $(dir build/src/demo.c src/demo.c demo.c)
```

等效于

```makefile
result = build/src/ src/ ./
```

---

call

```bash
$(call <expression>,<param1>,<param2>,...)
```

call可以让param1，param2替换expression中的$1，$2

```makefile
define COMPILE
	@echo -e "$(CYAN)" "CC:      $1"
	@mkdir -p $(dir $2)
	@$(CC) -MD -fPIC -c $(CFLAGS) $(abspath $1) -o $2
endef

@$(call COMPILE,demo.c,src)
```

等效于

```makefile
	@echo -e "$(CYAN)" "CC:      demo.c"
	@mkdir -p $(dir src)
	@$(CC) -MD -fPIC -c $(CFLAGS) $(abspath demo.c) -o src
```

---

@

```makefile
echo 123
```

在makefile中直接写echo 123的话，编译时打印如下:

```bash
echo 123
123
```

如果想在make时隐藏echo 123则可以

```makefile
@echo 123
```

这时候再make打印如下:

```bash
123
```

## 1.6 编程宏

定义弱符号

```c
__attribute__((weak))
```

# 2 LINUX内核编程

## 2.1 内核模块

### 2.1.1 内核模块的编译

编译内核模块Makefile模板如下:

```makefile
# Makefile 4.0
obj-m := []#编译文件
CURRENT_PATH := $(shell pwd)

#LINUX_KERNEL := $(shell uname -r)
#LINUX_KERNEL_PATH := /usr/src/linux-headers-$(LINUX_KERNEL)
LINUX_KERNEL_PATH := []#内核绝对路径

EXTRA_CFLAGS += -Wfatal-errors  # 遇到错误停止编译

export CROSS_COMPILE = []#交叉编译工具链 
export ARCH = []#交叉编译架构

all:
        make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) modules

help:
        make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) help

clean:
        make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) clean
```

### 2.1.2 内核模块中常用的宏

#### 2.1.2.1 动态获取内核模块中的变量

需要在插入模块时动态填入参数，则可以用到头文件<linux/moduleparam.h>中的宏

---
&rArr;module_param

最常用的是module_param,它的定义如下:

```c
module_param(name, type, perm);
```

@name - 内核模块文件中需要接受动态变量名

@type - name变量的类型，可以接受的类型有:

```c
byte, short, ushort, int, uint, long, ulong, charp, bool, invbool
```

填入的type通过的方式去找到内核中事先定义好的对应类型的结构体来匹配数据类型

```c
param_ops_##type
```

@perm - 变量在sysfs下生成节点文件的权限，权限值不能包含让普通用户也有写权限，否则编译报错，因此它一般为0644、0444、0600、0400。如果填入0则不会在sysfs下生成变量节点文件

这些生成的变量的节点文件位置如下:

```bash
/sys/module/[module_name]/parameters
```
---
&rArr;module_param_named

我们将/sys/module/[module_name]/parameters/demo_param和insmod demo.ko demo_param=123称为变量的外部名称，将在模块内部定义的用来接收外部传入值的变量称为内部变量。用module_param宏时它们外部变量和内部变量的名字相同都是name中填入的名字。在少数情况下我们想要这两者不同，则可以使用module_param_named，它的定义如下:

```c
module_param_named(name, variable, type, perm);
```

它的type和perm参数和module_param没有区别，区别在于外部变量的名字填入name，內部变量的名字填入variable。

我们看一看module_param的原始定义就能很清楚看出区别

```c
#define module_param(name, type, perm)				\
	module_param_named(name, name, type, perm)
```
---
&rArr;module_param_array

如果要同时获取多个变量则需要使用module_param_array，它的定义如下:

```c
module_param_array(name, type, nump, perm);
```

对比module_param来看type，perm的用法是一样的，区别在于name中填入的变量必须是一个数组，用来储存多个变量，nump则是可以获取insmod时传入参数的数量。

下面举个简单的例子:

```c
static int demo_param[8];
static int num;
module_param_array(demo_param, int, &num, 0400);
```

```bash
$ insmod demo_module demo_param=123,456
```

此时demo_param[0]被赋值123，demo_param[1]被赋值456，num被赋值2

若不需要获取传入的数量nump可以传入NULL，如下:

```c
static int demo_param[8];
module_param_array(demo_param, int, NULL, 0400);
```

同时要注意字符串数组中的字符串似乎不能包含逗号，否则一个字符串会被解析成两个。

---
&rArr;module_param_array_named

同样有区别内部变量和外部变量的宏module_param_array_named，它的定义如下:

```c
module_param_array_named(name, array, type, nump, perm);
```

我们也只需看module_param_array的原始定义即可理解module_param_array_named

```c
#define module_param_array(name, type, nump, perm)		\
	module_param_array_named(name, name, type, nump, perm)
```
---
&rArr;MODULE_PARM_DESC

对于非内核模块的使用者而言，他们往往拿不到内核模块的源代码，他们在插入模块带参数的模块时要正确添加参数则会产生一些困难，MODULE_PARM_DESC这个宏可以对模块中的参数进行注释，它的定义如下:

```c
MODULE_PARM_DESC(_parm, desc)
```

@_parm - 要注释的变量名

@desc - 以字符串的形式填写注释内容

使用modinfo就可以查看到这些注释

```bash
$ modinfo demo_module.ko
```

[内核模块动态变量demo](#内核模块动态变量demo)



## 2.2 内核启动参数

### 2.2.1 修改内核启动参数

```bash
$ make menuconfig
```

搜索CONFIG_CMDLINE，修改搜索结果可以修改启动参数

在启动参数下方有个Kernel command line type的项，它有三个选项

```config
( ) Use bootloader kernel arguments if available
( ) Extend bootloader kernel arguments
(X) Always use the default kernel command string
```

选择第一种模式Use bootloader kernel arguments if available后，若bootloader(如uboot、uefi)中传入了启动参数则使用bootloader中的启动参数启动内核

选择第二种模式Extend bootloader kernel arguments后，则对bootloader的启动参数进行拓展，作为最终的启动参数

选择第三种模式Always use the default kernel command string则内核会一直使用CONFIG_CMDLINE中填写的内核启动参数

### 2.2.2 添加自定义内核启动参数

在头文件<linux/init.h>中定义了宏__setup，它的定义如下:

```c
__setup(str, fn)
```

使用方法我们可以参考init/do_mounts.c，它里面包含了关于启动系统时文件系统挂载阶段相关的启动参数。如我们常见的“rw“、”ro“、”root=“，”rootdelay=“都在这个文件里面定义。

```c
static int __init readonly(char *str)
{
        if (*str)
                return 0;
        root_mountflags |= MS_RDONLY;
        return 1;
}
static int __init readwrite(char *str)
{
        if (*str)
                return 0;
        root_mountflags &= ~MS_RDONLY;
        return 1;
}
__setup("ro", readonly);
__setup("rw", readwrite);

static int __init root_dev_setup(char *line)
{
        strlcpy(saved_root_name, line, sizeof(saved_root_name));
        return 1;
}
__setup("root=", root_dev_setup);

static int __init root_delay_setup(char *str)
{
        root_delay = simple_strtoul(str, NULL, 0);
        return 1;
}
__setup("rootdelay=", root_delay_setup);
```

## 2.3 设备树

### 2.3.1 内核中获取设备树常用api

以下函数所使用的头文件是<linux/of.h>

---
&rArr;通过compatible属性查找指定节点

```c
struct device_node of_find_compatible_node(struct device_node *from,const char *type, const char *compat);
```

@from - 指向开始路径的节点，如果为NULL，则从根节点开始

@type - device_type设备类型，可以为NULL

@compat - 指向节点的compatible属性的值（字符串）的首地址

成功：得到节点的首地址；失败：NULL

---
&rArr;通过of_device_id结构体查找指定节点

```c
struct device_node *of_find_matching_node(struct device_node *from,const struct of_device_id *matches);
```

@from - 指向开始路径的节点，如果为NULL，则从根节点开始

@matches - 指向设备ID表，注意ID表必须以NULL结束

例如:

```C
static const struct of_device_id geth_of_match[] = {
	{.compatible = "allwinner,sunxi-gmac",},
	{},
};
```

成功：得到节点的首地址；失败：NULL 

---
&rArr;通过路径查找指定节点 

```C
struct device_node *of_find_node_by_path(const char *path);
```

@path - 带全路径的节点名，也可以是节点的别名 

成功：得到节点的首地址；失败：NULL 

---
&rArr;提取指定属性的值

```c
struct property *of_find_property(const struct device_node *np, const char *name, int *lenp);
```

 @np - 设备节点指针

@name - 属性名称 

@lenp - 属性值的字节数

成功：属性值的首地址；失败：NULL

---
&rArr;得到属性值中数据的数量 

```c
int of_property_count_elems_of_size(const struct device_node *np,const char *propname, int elem_size);
```

@np - 设备节点指针

@propname  - 属性名称

@elem_size - 每个数据的单位（字节数） 

成功：属性值的数据个数；失败：负数，绝对值是错误码

---
&rArr;得到属性值中指定标号的32位数据值

```c
int of_property_read_u32_index(const struct device_node *np, const char *propname, u32 index, u32 *out_value);
```

@np - 设备节点指针

@propname  - 属性名称 

@index  - 属性值中指定数据的标号

@out_value - 输出参数，得到指定数据的值 

成功：0；失败：负数，绝对值是错误码

---
&rArr;提取属性值中的字符串

```c
int of_property_read_string(struct device_node np, const char *propname, const char *out_string);
```

 @np - 设备节点指针

 @propname  - 属性名称 

@out_string - 输出参数，指向字符串（属性值） 

成功：0；失败：负数，绝对值是错误码 

---
&rArr;提取属性值中的多字符串

```c
inline int of_property_read_string_index(const struct device_node *np, const char *propname, int index, const char **output);
```

 @np - 设备节点指针

 @propname  - 属性名称 

@index - 提取字符串的位置

@out_string - 输出参数，指向字符串（属性值） 

成功：0；失败：负数，绝对值是错误码 

多个字符串在设备树中填写方式如下:

```dts
	pins = "PH2","PH3","PH4","PH7";
```

---
&rArr;提取默认属性“#address-cells”的值

```c
int of_n_addr_cells(struct device_node *np);
```

 @np - 设备节点指针 

成功：地址长度的数量；失败：负数，绝对值是错误码

---
&rArr;提取I/O口地址 

```c
__be32 *of_get_address(struct device_node *np, int index, u64 *size, unsigned int *flags);
```

 @np - 设备节点指针 

@index - 地址的标号

@size - 输出参数，I/O口地址的长度

 @flags - 输出参数，类型（IORESOURCE_IO、IORESOURCE_MEM） 

成功：I/O口地址的首地址；失败：NULL

---
&rArr;提取I/O口地址转换成物理地址

```c
u64 of_translate_address(struct device_node *np, const __be32 *in_addr);
```

@np - 设备节点指针

@in_addr - 设备树提取的I/O地址

成功：物理地址；失败：OF_BAD_ADDR

---
&rArr;提取I/O口地址并映射成虚拟地址

```c
void __iomem *of_iomap(struct device_node *np, int index);
```

@np - 设备节点指针

@index - I/O地址的标号

成功：映射好虚拟地址；失败：NULL

---
&rArr;提取I/O口地址并申请I/O资源及映射成虚拟地址

```c
void __iomem *of_io_request_and_map(struct device_node *np, int index, const char *name)
```

@np - 设备节点指针

@index - I/O地址的标号

@name - 设备名，申请I/O地址时使用

成功：映射好虚拟地址；失败：NULL

---
&rArr;从设备树中提取资源resource（I/O地址）

```c
int of_address_to_resource(struct device_node *np, int index, struct resource *r);
```

@np - 设备节点指针

@index - I/O地址资源的标号

@r - 输出参数，指向资源resource（I/O地址）

成功：0；失败：负数，绝对值是错误码

---
&rArr;从设备树中提取gpio口

需要包含头文件<linux/of_gpio.h>

```c
int of_get_named_gpio(struct device_node *np, const char *propname, int index);
```

@np - 设备节点指针

@propname - 属性名

 @index - gpio口引脚标号

成功：得到GPIO口编号；失败：负数，绝对值是错误码

---
&rArr;从设备树中提取中断的数量 

```c
int of_irq_count(struct device_node *dev);
```

@np - 设备节点指针

成功：大于等于0，实际中断数量，0则表示没有中断

---
&rArr;从设备树中提取中断号

```c
int of_irq_get(struct device_node *dev, int index);
```

@np - 设备节点指针

@index - 要提取的中断号的标号

成功：中断号；失败：负数，其绝对值是错误码 

---
&rArr;从设备树中提取MAC地址

```c
void *of_get_mac_address(struct device_node *np);
```

@np - 设备节点指针

@成功：MAC（6字节）的首地址；失败：NULL

---
&rArr;从设备树中提取clk

```c
struct clk *of_clk_get(struct device_node *np, int index)
```

@np - 设备节点指针

@index - 要提取的clk的标号

成功：clk结构体；失败：负数，其绝对值是错误码 

---
## 2.4 ACPI

从系统中获取acpi

```bash
sudo cat /sys/firmware/acpi/tables/DSDT > DSDT.aml
sudo apt-get install iasl
iasl -d DSDT.aml
```

如此做就可以生成能看懂的DSDT.dsl

## 2.5 内核线程

```c
#include <linux/sched.h>     //wake_up_process()
#include <linux/kthread.h>   //kthread_create()、kthread_run()

static struct task_struct *demo_task;
static int demo_function(void *arg)
{
    while(1)
    {
        
    }
    return 0;
}

#if 1
demo_task = kthread_create(demo_function, NULL, "demo_function");
wake_up_process(test_task);
#else
demo_task = kthread_run(demo_function, NULL, "demo_function");
#endif
kthread_stop(demo_task);
```

## 2.6 内核时钟

内核clk框架我们以allwinner-t3为示例介绍说明，其他芯片大同小异

在设备树中有关于初始化clk的设备树节点

```dts
compatible = "allwinner,clk-init";
device_type = "clocks";
#address-cells = <2>;
#size-cells = <2>;
ranges;
reg = <0x0 0x01c20000 0x0 0x0324>;  /*cpux space*/
```

在内核中有CLK_OF_DECLARE(sunxi_clocks_init, "allwinner,clk-init", of_sunxi_clocks_init);

在of_sunxi_clocks_init函数中初始化了clk_ops

```c
static const struct clk_ops clk_factors_ops = {
	.enable = sunxi_clk_fators_enable,
	.disable = sunxi_clk_fators_disable,
	.is_enabled = sunxi_clk_fators_is_enabled,

	.recalc_rate = sunxi_clk_factors_recalc_rate,
	.round_rate = sunxi_clk_factors_round_rate,
	.set_rate = sunxi_clk_factors_set_rate,
};
```

一般在使用时我们会在驱动中使用如下方式描述驱动时钟:

```dts
spi0: spi@01c05000 {
	...
	clocks = <&clk_pll_periph0>, <&clk_spi0>;
	...
}
```

clk_pll_periph0定义如下:

```dts
clk_pll_periph0: pll_periph0 {
        #clock-cells = <0>;
        compatible = "allwinner,pll-clock";
        assigned-clock-rates = <600000000>;
        lock-mode = "new";
        clock-output-names = "pll_periph0";
};
```

clk_spi0定义如下:

```dts
clk_spi0: spi0 {
        #clock-cells = <0>;
        compatible = "allwinner,periph-clock";
        clock-output-names = "spi0";
};
```

通过上面两个设备树，系统在初始化时就会运行of_sunxi_pll_clk_setup和of_sunxi_periph_clk_setup

```c
CLK_OF_DECLARE(sunxi_pll_clk, "allwinner,pll-clock", of_sunxi_pll_clk_setup);

CLK_OF_DECLARE(sunxi_periph_clk, "allwinner,periph-clock", of_sunxi_periph_clk_setup);
```

of_sunxi_pll_clk_setup函数中通过 clock-output-names中的字符串"pll_periph0"去匹配一个结构体

```c
{"pll_cpu",     ....}
{"pll_ddr0",    ....}
{"pll_periph0", ....}
{"pll_periph1", ....}
{"pll_gpu",     ....}
{"pll_video0",  ....}
{"pll_video1",  ....}
{"pll_ve",      ....}
{"pll_de",      ....}
{"pll_audio",   ....}
{"pll_sata",    ....}
{"pll_mipi",    ....}
{"pll_ddr1",    ....}
```

这些结构体中省略的参数为初始化时钟源的参数，of_sunxi_pll_clk_setup最终会注册一个clk结构体，这个结构体可以根据”clk_pll_periph0"字符串找到

of_sunxi_periph_clk_setup函数和of_sunxi_pll_clk_setup函数同理

```c
.................
{"twi0",    ....}
{"twi1",    ....}
{"twi2",    ....}
{"twi3",    ....}
{"twi4",    ....}
{"spi0",    ....}
{"spi1",    ....}
{"spi2",    ....}
{"spi3",    ....}
{"ps20",    ....}
{"ps21",    ....}
{"ir0",     ....}
{"ir1",     ....}
{"emac",    ....}
{"gmac",    ....}
.................
```

这些结构体中省略的参数为初始化spi0的时钟配置，of_sunxi_periph_clk_setup最终注册一个clk结构体，这个结构体可以根据”clk_spi0"字符串找到

那么驱动中如何使用对时钟进行配置呢？我们同样用spi0作为例子

```dts
spi0: spi@01c05000 {
	...
	clocks = <&clk_pll_periph0>, <&clk_spi0>;
	...
}
```

我们通过手册得知spi0时钟是这样来的: 

```
OSC24M
PLL_PERIPH0(1X)    --->  spi0 module clock --->  spi0内部分频器
PLL_PERIPH1(1X)
```

代码如下:

```c
#include <linux/clk.h>

struct clk *pclk;  /* PLL clock */
struct clk *mclk;  /* spi module clock */

long rate;

pclk = of_clk_get(of_node, 0);
mclk = of_clk_get(of_node, 1);

clk_set_parent(mclk, pclk);

rate = clk_round_rate(mclk, mod_clk);

clk_set_rate(mclk, rate);

clk_prepare_enable(mclk);
```

通过of_clk_get我们从设备树中通过"clk_pll_periph0"和"clk_spi0"字符串分别获取在上述时钟初始化过程中注册的clk结构体分别保存在pclk和mclk中

clk_set_parent的功能是设置spi0 module clock的时钟源，在上面提到的结构体中spi2时钟源配置了它的默认时钟源为PLL_PERIPH1，我们可以在clk_set_parent函数中再次手动设置时钟源，例如此处通过设备树将其配置为了PLL_PERIPH0

```c
{"spi0", ....,  PLL_PERIPH1, ....}
{"spi1",    ....}
{"spi2",    ....}
{"spi3",    ....}
```

clk_round_rate函数可以获取当前时钟频率和限制最大频率值中的较小值，例如在设备树中有clock-frequency节点，我们使用of_property_read_u32就可以获取到mod_clk的值，rate是mclk和mod_clk中的较小值

```dts
spi0: spi@01c05000 {
	...
	clocks = <&clk_pll_periph0>, <&clk_spi0>;
	clock-frequency = <100000000>;
	...
}
```

clk_set_rate可以通过clk_round_rate得出的较小值重新设置速率

clk_prepare_enable可以将spi0时钟源使能

以上都属于时钟源使能，操作的是ccu寄存器，

设置好时钟源之后spi内部的时钟寄存器就可以读取到时钟寄存器读取到时钟源，用户会通过ioctl传入需要的频率，此时再通过spi的内部分频，让时钟频率最终输出。

## 2.7 内核中断

### 2.7.1 GIC
GIC(Generic Interrupt Controller)是ARM公司提供的一个通用的中断控制器,自外设的interrupt source输入信号分成两种类型,分别是私有外设中断PPI(Private Peripheral Interrupt)和共享外设中断SPI(Shared Peripheral Interrupt),还有一种通过内部产生的中断:软中断SGI(Software-generated interrupt)

#### 2.7.1.1 SPI中断
内核中申请SPI中断的函数为:

```c
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
	    const char *name, void *dev)
```

@irq - 要申请的中断号

@handler - 中断处理函数

@flags - 中断标志位

@name - 中断名称

@dev - 传入中断处理函数的参数

(在GIC中0-15号中断为SGI中断,16-31号为PPI中断，所以在申请SPI中断时,需要把手册上的中断号减去32,即第1个SPI中断为内核中的0号中断)

中断处理函数定义如下:

```c
static irqreturn_t demo_handler(int irq, void *dev_id)
```

@irq - request_irq时传递的中断号

@dev_id - request_irq时传入的参数

返回值 如下:

```c
typedef enum irqreturn irqreturn_t;
enum irqreturn {
	IRQ_NONE			= (0 << 0),
	IRQ_HANDLED			= (1 << 0),
	IRQ_WAKE_THREAD		= (1 << 1),
};
```

返回IRQ_NONE时表示中断不是来自此设备或未被处理

返回IRQ_HANDLED时表示中断正确处理

返回IRQ_WAKE_THREAD时表示请求唤醒处理程序线程

可以使用下面这个宏来做判断返回

```c
#define IRQ_RETVAL(x)	((x) ? IRQ_HANDLED : IRQ_NONE)
```

完整申请中断的流程如下:

在设备树中我们以interrupts来描述中断:

```dts
interrupts = <GIC_SPI 10 IRQ_TYPE_LEVEL_HIGH>;
```

```c
#include <linux/interrupt.h>

static irqreturn_t demo_handler(int irq, void *arg)
{
    ......
    return IRQ_HANDLED;
}

irq_num = of_irq_get(node, 0);

request_irq(irq_num, demo_handler, 0, dev_name, arg);

free_irq(irq_num, NULL);
```

#### 2.7.1.2 SGI中断
在arm处理器中，有16个SGI，硬件中断号为0-15。它通常用于多核之间的通信。SGI通常在Linux内核中用作IPI处理器间中断。在linux内核中，已经定义了以下IPI中断，所以当用户使用自定义IPI中断时，建议使用未使用的中断8-15。
内核中申请SGI中断的函数:

```c
//arch/arm/kernel/smp.c
enum ipi_msg_type {
	IPI_WAKEUP,
	IPI_TIMER,
	IPI_RESCHEDULE,
	IPI_CALL_FUNC,
	IPI_CPU_STOP,
	IPI_IRQ_WORK,
	IPI_COMPLETION,
	IPI_CPU_BACKTRACE,
};
```

```c
int set_ipi_handler(int ipinr, void *handler, char *desc)
```
@ipinr - 要申请的中断号

@handler - 中断处理函数

@desc - 中断名称

内核中触发SGI中断的函数:
```c
void gic_raise_softirq(const struct cpumask *mask, unsigned int irq)
```
@mask - 传递要控制的cpu

@irq - 要控制的中断号

内核中清除SGI中断:
```c
void clear_ipi_handler(int ipinr)
```
@ipinr - 要清除的中断号

完整申请中断的流程如下:
```c
void handler_name(void)
{

}

set_ipi_handler(irq_num, demo_handler, dev_name);

gic_raise_softirq(cpumask_of(1), irq_num);

clear_ipi_handler(irq_num);
```

## 2.8 内核熵

Linux内核采用熵来描述数据的随机性

## 2.9 内核错误打印

### 2.9.1 系统调用错误

在应用层可以用以下方式查看一个系统调用的错误类型

```c
#include <errno.h>

int ret = 0;

ret = ioctl(fd, request, arg);
if(ret < 0)
{
    perror("ioctl");
}
```

得到的报错打印如下:

```bash
ioctl: Inappropriate ioctl for device
```

不同的报错码对应的字符串再glibc中定义:

```c
  EPERM                          = 0x40000001,  /* Operation not permitted */
  ENOENT                         = 0x40000002,  /* No such file or directory */
  ESRCH                          = 0x40000003,  /* No such process */
  EINTR                          = 0x40000004,  /* Interrupted system call */
  EIO                            = 0x40000005,  /* Input/output error */
  ENXIO                          = 0x40000006,  /* No such device or address */
  E2BIG                          = 0x40000007,  /* Argument list too long */
  ENOEXEC                        = 0x40000008,  /* Exec format error */
  EBADF                          = 0x40000009,  /* Bad file descriptor */
  ECHILD                         = 0x4000000a,  /* No child processes */
  EDEADLK                        = 0x4000000b,  /* Resource deadlock avoided */
  ENOMEM                         = 0x4000000c,  /* Cannot allocate memory */
  EACCES                         = 0x4000000d,  /* Permission denied */
  EFAULT                         = 0x4000000e,  /* Bad address */
  ENOTBLK                        = 0x4000000f,  /* Block device required */
  EBUSY                          = 0x40000010,  /* Device or resource busy */
  EEXIST                         = 0x40000011,  /* File exists */
  EXDEV                          = 0x40000012,  /* Invalid cross-device link */
  ENODEV                         = 0x40000013,  /* No such device */
  ENOTDIR                        = 0x40000014,  /* Not a directory */
  EISDIR                         = 0x40000015,  /* Is a directory */
  EINVAL                         = 0x40000016,  /* Invalid argument */
  EMFILE                         = 0x40000018,  /* Too many open files */
  ENFILE                         = 0x40000017,  /* Too many open files in system */
  ENOTTY                         = 0x40000019,  /* Inappropriate ioctl for device */
  ETXTBSY                        = 0x4000001a,  /* Text file busy */
  EFBIG                          = 0x4000001b,  /* File too large */
  ENOSPC                         = 0x4000001c,  /* No space left on device */
  ESPIPE                         = 0x4000001d,  /* Illegal seek */
  EROFS                          = 0x4000001e,  /* Read-only file system */
  EMLINK                         = 0x4000001f,  /* Too many links */
  EPIPE                          = 0x40000020,  /* Broken pipe */
  EDOM                           = 0x40000021,  /* Numerical argument out of domain */
  ERANGE                         = 0x40000022,  /* Numerical result out of range */
  EAGAIN                         = 0x40000023,  /* Resource temporarily unavailable */
  EINPROGRESS                    = 0x40000024,  /* Operation now in progress */
  EALREADY                       = 0x40000025,  /* Operation already in progress */
  ENOTSOCK                       = 0x40000026,  /* Socket operation on non-socket */
  EMSGSIZE                       = 0x40000028,  /* Message too long */
  EPROTOTYPE                     = 0x40000029,  /* Protocol wrong type for socket */
  ENOPROTOOPT                    = 0x4000002a,  /* Protocol not available */
  EPROTONOSUPPORT                = 0x4000002b,  /* Protocol not supported */
  ESOCKTNOSUPPORT                = 0x4000002c,  /* Socket type not supported */
  EOPNOTSUPP                     = 0x4000002d,  /* Operation not supported */
  EPFNOSUPPORT                   = 0x4000002e,  /* Protocol family not supported */
  EAFNOSUPPORT                   = 0x4000002f,  /* Address family not supported by protocol */
  EADDRINUSE                     = 0x40000030,  /* Address already in use */
  EADDRNOTAVAIL                  = 0x40000031,  /* Cannot assign requested address */
  ENETDOWN                       = 0x40000032,  /* Network is down */
  ENETUNREACH                    = 0x40000033,  /* Network is unreachable */
  ENETRESET                      = 0x40000034,  /* Network dropped connection on reset */
  ECONNABORTED                   = 0x40000035,  /* Software caused connection abort */
  ECONNRESET                     = 0x40000036,  /* Connection reset by peer */
  ENOBUFS                        = 0x40000037,  /* No buffer space available */
  EISCONN                        = 0x40000038,  /* Transport endpoint is already connected */
  ENOTCONN                       = 0x40000039,  /* Transport endpoint is not connected */
  EDESTADDRREQ                   = 0x40000027,  /* Destination address required */
  ESHUTDOWN                      = 0x4000003a,  /* Cannot send after transport endpoint shutdown */
  ETOOMANYREFS                   = 0x4000003b,  /* Too many references: cannot splice */
  ETIMEDOUT                      = 0x4000003c,  /* Connection timed out */
  ECONNREFUSED                   = 0x4000003d,  /* Connection refused */
  ELOOP                          = 0x4000003e,  /* Too many levels of symbolic links */
  ENAMETOOLONG                   = 0x4000003f,  /* File name too long */
  EHOSTDOWN                      = 0x40000040,  /* Host is down */
  EHOSTUNREACH                   = 0x40000041,  /* No route to host */
  ENOTEMPTY                      = 0x40000042,  /* Directory not empty */
  EPROCLIM                       = 0x40000043,  /* Too many processes */
  EUSERS                         = 0x40000044,  /* Too many users */
  EDQUOT                         = 0x40000045,  /* Disk quota exceeded */
  ESTALE                         = 0x40000046,  /* Stale file handle */
  EREMOTE                        = 0x40000047,  /* Object is remote */
  EBADRPC                        = 0x40000048,  /* RPC struct is bad */
  ERPCMISMATCH                   = 0x40000049,  /* RPC version wrong */
  EPROGUNAVAIL                   = 0x4000004a,  /* RPC program not available */
  EPROGMISMATCH                  = 0x4000004b,  /* RPC program version wrong */
  EPROCUNAVAIL                   = 0x4000004c,  /* RPC bad procedure for program */
  ENOLCK                         = 0x4000004d,  /* No locks available */
  EFTYPE                         = 0x4000004f,  /* Inappropriate file type or format */
  EAUTH                          = 0x40000050,  /* Authentication error */
  ENEEDAUTH                      = 0x40000051,  /* Need authenticator */
  ENOSYS                         = 0x4000004e,  /* Function not implemented */
  ENOTSUP                        = 0x40000076,  /* Not supported */
  EILSEQ                         = 0x4000006a,  /* Invalid or incomplete multibyte or wide character */
  EBACKGROUND                    = 0x40000064,  /* Inappropriate operation for background process */
  EDIED                          = 0x40000065,  /* Translator died */
  ED                             = 0x40000066,  /* ? */
  EGREGIOUS                      = 0x40000067,  /* You really blew it this time */
  EIEIO                          = 0x40000068,  /* Computer bought the farm */
  EGRATUITOUS                    = 0x40000069,  /* Gratuitous error */
  EBADMSG                        = 0x4000006b,  /* Bad message */
  EIDRM                          = 0x4000006c,  /* Identifier removed */
  EMULTIHOP                      = 0x4000006d,  /* Multihop attempted */
  ENODATA                        = 0x4000006e,  /* No data available */
  ENOLINK                        = 0x4000006f,  /* Link has been severed */
  ENOMSG                         = 0x40000070,  /* No message of desired type */
  ENOSR                          = 0x40000071,  /* Out of streams resources */
  ENOSTR                         = 0x40000072,  /* Device not a stream */
  EOVERFLOW                      = 0x40000073,  /* Value too large for defined data type */
  EPROTO                         = 0x40000074,  /* Protocol error */
  ETIME                          = 0x40000075,  /* Timer expired */
  ECANCELED                      = 0x40000077,  /* Operation canceled */
  EOWNERDEAD                     = 0x40000078,  /* Owner died */
  ENOTRECOVERABLE                = 0x40000079,  /* State not recoverable */
```
在内核中写系统调用返回值的时候可以参考此报错表，以便使用人员进行错误分析

## 2.10 内核内存

设备树中可以配置内核中使用的内存地址位置，一般用memory节点表示，在ft2000a中默认的内存地址，起始为0x80000000，大小为2G那么设备树中的描述则如下:

```dts
        memory {
                device_type = "memory";
                reg = <0x0 0x80000000 0x0 0x80000000>;
        };
```

上述设备树的含义为从0x80000000开始0x80000000大小的这段2G的地址空间配置为内存地址

注：reg中前面的0x0和0x80000000都代表了启示地址，前面的0x0表示高位的地址，所以设备树节点中可以描述64位的地址空间

如:需要表示起始地址为0x100000000，大小为2G的地址段，可以描述为

```dts
        memory {
                device_type = "memory";
                reg = <0x1 0x00000000 0x0 0x80000000>;
        };
```

如果我们想对ft2000a内存进行拓展，拓展大小为2G，则需要在其规定的拓展内存地址0x200000000上接入2G内存，并将设备树做如下修改:

```dts
    memory {
            device_type = "memory";
            reg = <0x0 0x80000000 0x0 0x80000000
            		0x2 0x00000000 0x0 0x80000000>;
    };
```

## 2.11 内核 PROC_FS

# 3 LINUX文件系统

## 3.1 cpio文件的打包压缩、解包解压

打包压缩
```bash
$ ls
bin  dev  etc  init  lib  lib32  linuxrc  media  mnt  opt  proc  root run  sbin  sys  tmp  usr  var
$ find ./* | cpio -H newc -o > rootfs.cpio
16007 blocks
$ ls
bin  dev  etc  init  lib  lib32  linuxrc  media  mnt  opt  proc  root  rootfs.cpio  run  sbin  sys  tmp  usr  var
$ gzip rootfs.cpio
bin  dev  etc  init  lib  lib32  linuxrc  media  mnt  opt  proc  root  rootfs.cpio.gz  run  sbin  sys  tmp  usr  var
```

解包解压
```bash
$ ls
rootfs.cpio.gz
$ gunzip rootfs.cpio.gz
$ ls
rootfs.cpio
$ cpio -idmv < rootfs.cpio
......
$ ls
bin  dev  etc  init  lib  lib32  linuxrc  media  mnt  opt  proc  root  rootfs.cpio  run  sbin  sys  tmp  usr  var
```

## 3.2 制作ramdisk

```bash
$ ls
rootfs
$ ls rootfs           
bin  dev  etc  init  lib  lib32  linuxrc  media  mnt  opt  proc  root  run  sbin  sys  tmp  usr  var
$ genext2fs -b 81920 -d rootfs ramdisk.image
$ ls
rootfs ramdisk.image
$ gzip -9 ramdisk.image 
$ ls
rootfs ramdisk.image.gz
$ mkimage -A arm -T ramdisk -C gzip -n Ramdisk -d ramdisk.image.gz uramdisk.image.gz
$ ls
rootfs ramdisk.image.gz uramdisk.image.gz
```

# 4 SSH

## 4.1 ssh的使用

ssh客户端一般Linux都会自带
远程登陆:

```bash
ssh user@ip
```

一般来说ssh端口为22，所以以上命令也等效于下面这个

```bash
ssh user@ip -p 22
```

如果所要连接ssh端口不是22，则可以使用-P手动指定端口号

```bash
ssh user@ip -p port_mun
```

----

安装ssh服务端

```bash
sudo apt install openssh-server
```

重启sshd服务

```bash
service ssh restart
```

重启sshd服务后其他设备就可以对本设备使用ssh命令进行连接

-----

生成公钥和私钥

```bash
ssh-keygen
```

生成的公钥和私钥位于家目录的.ssh目录

```bash
$ ls ~/.ssh
id_rsa  id_rsa.pub
```

免密登陆

将A机器的~/.ssh/id_rsa.pub添加到B机器的~/.ssh/authorized_keys中，即可让A机器能免密登陆B机器

使用ssh连接其他机器之后，~/.ssh/known_hosts下会记录连接过的ip和其对应的公钥信息，以后每次连接都会对其进行核验，如果二者不一致则会报警告

消除警告有两种方法

例如我的192.168.1.109ip在ssh连接时提示ip和其对应的公钥信息不一致

1、

```bash
vim ~/.ssh/authorized_keys
```

手动删除其中关于109的信息保存退出即可

2、

```bash
ssh-keygen -f "~/.ssh/known_hosts" -R "192.168.1.109"
```

这个命令本质上也是删除~/.ssh/known_hosts中关于192.168.1.109的信息，只不过是把操作集成在了ssh-keygen工具中

## 4.2ssh无法登陆问题汇总

报错:

```bash
/var/empty mus  t be owned by root and not group or world-writable.
```

解决方法:

```bash
$ chown root.root /var/empty
$ mkdir -p /var/empty/sshd
$ chown -R root.root /var/empty/sshd
$ chmod 744 /var/empty/sshd
$ /etc/init.d/S50sshd restart
```

-------------



# 5 LINUX常用工具

## 5.1 git

撤销上一次commit但是保留修改 

```bash
$ git reset --soft HEAD^
```

撤销git add状态

```bash
$ git restore --staged [filename]
```

## 5.2 devmem

下载

```bash
$ git clone https://github.com/wxlinuxstore/devmem.git
```

编译

```bash
$ cd devmem
$ make
```

使用

```bash
$ ./devmem [x] 
```

读取某个内存地址的值，x为内存地址 

```bash
$ ./devmem [x] [y] [z] 
```

把某个值写入某个内存地址，x为内存地址，y为写入长度可以为w、h、b，分别对应unsigned long、unsigned short、unsigned char，z为要写入的值

## 5.3 i2ctools

下载

```bash
$ git clone https://github.com/wxlinuxstore/i2ctools.git
```

编译

```bash
$ cd i2ctools 
$ make
```

使用

```bash
$ export LD_LIBRARY_PATH=`pwd`/lib 
```

设置动态库路径

```bash
$ ./tools/i2cdetect -l
```

查看当前系统上已有的i2c设备节点

```bash
$ ./tools/i2cdetect -y -r [x]
```

查看某个节点下挂载了哪些地址的i2c设备，x为设备节点序号

```bash
$ ./tools/i2cdump -f -y [x] [y]
```

 查看某个节点下的某个设备地址的寄存器，x为设备节点序号，y为设备地址

```bash
$ ./i2cget -f -y [x] [y] [z]
```

读某个节点下的某个设备地址中的某个寄存器 ，x为设备节点序号，y为设备地址，z为寄存器地址

```bash
$ ./i2cset -f -y [x] [y] [z] [w] 
```

写某个节点下的某个设备地址的某个寄存器 ，x为设备节点序号，y为设备地址，z为寄存器地址，w为要写入的值

## 5.4 can-utils

使用

```bash
$ ip link set can0 down
```

关闭can0接口 只有在关闭状态下才能对can配置

```bash
$ ip link set can0 type can bitrate 50000  [loopback (on/off)]
```

配置can0波特率,方括号内为可选项-配置回环模式

```bash
$ ip link set can0 up
```

打开can0接口

```bash
$ cansend can0 123#DEADBEEF
```

can0发送数据123#DEADBEEF

```bash
$ candump can0 &
```

can0后台接收

```bash
$ ip -details -statistics link show can0
```

查看 can0 的比特率配置等,以及统计数据(接收/发送/出错帧等)

## 5.5 fdisk

新建分区

```bash
fdisk "磁盘节点名称"
command                                         :p			(查看当前分区情况)
command                                         :n			(选择新建分区指令)
selected partition                              :			(填写需要新建的分区号)
Last sector, +sectors or +size{K,M,G,T,P}       :			(大小根据需求填写新建分区大小如"+1G/+1M")
command                                         :w			(将修改写入磁盘)
```

删除分区

```bash
fdisk "磁盘节点名称"
command                                         :p				(查看当前分区情况)
command                                         :d				(选择删除分区指令)
selected partition                              :				(填写需要删除的分区号)
command                                         :w				(将修改写入磁盘)
```

修改分区文件系统类型

```bash
fdisk "磁盘节点名称"     
command                                         :p				(查看当前分区情况)
command                                         :t				(选择修改分区类型指令)
selected partition                              :				(填写需要修改分区类型的分区号)
Hex code                                        :				(此时输入l可以查看所支持的所有系统类型的编号)
```

## 5.6 docker

拉取docker镜像

```bash
$ docker pull arm64v8/ubuntu:22.04
```

镜像导出

```bash
$ docker save -o arm64-ubuntu.tar
```

镜像导入

```bash
$ docker load -i arm64-ubuntu.tar
```

查看镜像

```bash
$ docker images
```

容器启动，[x]为docker images中看到的镜像id

```bash
$ docker run -itd [x]
```

查看运行中的容器

```bash
$ docker ps
```

进入容器进行操作，[x]为docker ps中看到的container  id

```bash
$ docker exec -it [x] /bin/sh
```

在容器中退出

```bash
# exit
```

停止运行中的容器，[x]为docker ps中看到的container  id

```bash
$ docker stop [x]
```

删除容器，[x]为docker ps中看到的container  id

```bash
$ docker rm [x]
```

删除镜像，[x]为docker images中看到的镜像id

```bash
$ docker image rm [x]
```

## 5.7 rng-tools、haveged

查询熵值

```bash
$ cat /proc/sys/kernel/random/entropy_avail
68
```

安装rng-tools、haveged

```bash
$ apt install rng-tools haveged -y
$ systemctl enable haveged
$ systemctl enable rng-tools
$ systemctl restart rng-tools
```

buildroot文件系统中则在menuconfig中增加haveged和rng-tools的编译选项

安装完成之后再次查询熵值		

```bash
$ cat /proc/sys/kernel/random/entropy_avail
2268
```

达到2000左右则基本合格

## 5.8 static-ip

ubuntu

```bash
$ sudo vim /etc/network/interfaces
```

添加以下启动项:

```bash
auto eth0
iface eth0 inet static
address 192.168.0.4
netmask 255.255.255.0
gateway 192.168.0.1
```

重启网络服务:

```bash
$ sudo /etc/init.d/networking restart
```

如果遇到一下问题

```bash
Configuring network interfaces in background...RTNETLINK answers: File exists
```

可能是以下两个服务有冲突导致的 停用NetworkManager即可

```bash
/etc/init.d/networking
/etc/init.d/network-manager
```

如果解决不了，直接重启即可。

## 5.9 xfonts-utils

查看系统当前支持的字体列表

```bash
$ fc-list
```

为linux系统添加windows系统字体

```bash
$ cd /usr/share/fonts/ 
$ sudo mkdir new
$ sudo chmod -R 755 /usr/share/fonts/new
```

将C:\Windows\Fonts的文件拷贝到刚刚创建的new目录下

```bash
$ sudo vim /etc/fonts/fonts.conf 
```

在如下位置中添加一行:

```conf
<dir>/usr/share/fonts/new</dir>
```

------

```conf
<!-- Font directory list -->

        <dir>/usr/share/fonts</dir>
        <dir>/usr/local/share/fonts</dir>
        <dir>/usr/share/fonts/new</dir>
        <dir prefix="xdg">fonts</dir>
        <!-- the following element will be removed in the future -->
        <dir>~/.fonts</dir>

<!--
```

------

刷新当前系统字体

```bash
$ fc-cache
```

查看系统当前支持的字体列表可以看到新添加的字体

```bash
$ fc-list
```

## 5.10 xrandr

直接输入xrandr可以查看当前系统支持的分辨率

```bash
$ xrandr
Screen 0: minimum 1 x 1, current 1360 x 768, maximum 8192 x 8192
Virtual1 connected primary 1360x768+0+0 (normal left inverted right x axis y axis) 0mm x 0mm
   800x600       60.00 +  60.32  
   2560x1600     59.99  
   1920x1440     60.00  
   1856x1392     60.00  
   1792x1344     60.00  
   1920x1200     59.88  
   1600x1200     60.00  
   1680x1050     59.95  
   1400x1050     59.98  
   1280x1024     60.02  
   1440x900      59.89  
   1280x960      60.00  
   1360x768      60.02* 
   1280x800      59.81  
   1152x864      75.00  
   1280x768      59.87  
   1024x768      60.00  
   640x480       59.94  
Virtual2 disconnected (normal left inverted right x axis y axis)
Virtual3 disconnected (normal left inverted right x axis y axis)
Virtual4 disconnected (normal left inverted right x axis y axis)
Virtual5 disconnected (normal left inverted right x axis y axis)
Virtual6 disconnected (normal left inverted right x axis y axis)
Virtual7 disconnected (normal left inverted right x axis y axis)
Virtual8 disconnected (normal left inverted right x axis y axis)
```

修改分辨率为1920x1200，所修改的分辨率必须是列表中有的

```bash
$ xrandr -s 1920x1200
```

自定义分辨率（1920 1080为例）：

```bash
$ cvt 1920 1080
# 1920x1080 59.96 Hz (CVT 2.07M9) hsync: 67.16 kHz; pclk: 173.00 MHz
Modeline "1920x1080_60.00"  173.00  1920 2048 2248 2576  1080 1083 1088 1120 -hsync +vsync
```

cvt命令是进行添加参数的辅助工具，它会生成一段详细配置

我们用这个详细配置可以添加一段分辨率配置

```bash
$ xrandr --newmode "1920x1080_60.00"  173.00  1920 2048 2248 2576  1080 1083 1088 1120 -hsync +vsync
$ xrandr 
Screen 0: minimum 1 x 1, current 1360 x 768, maximum 8192 x 8192
Virtual1 connected primary 1360x768+0+0 (normal left inverted right x axis y axis) 0mm x 0mm
   800x600       60.00 +  60.32  
   2560x1600     59.99  
   1920x1440     60.00  
   1856x1392     60.00  
   1792x1344     60.00  
   1920x1200     59.88  
   1600x1200     60.00  
   1680x1050     59.95  
   1400x1050     59.98  
   1280x1024     60.02  
   1440x900      59.89  
   1280x960      60.00  
   1360x768      60.02* 
   1280x800      59.81  
   1152x864      75.00  
   1280x768      59.87  
   1024x768      60.00  
   640x480       59.94  
Virtual2 disconnected (normal left inverted right x axis y axis)
Virtual3 disconnected (normal left inverted right x axis y axis)
Virtual4 disconnected (normal left inverted right x axis y axis)
Virtual5 disconnected (normal left inverted right x axis y axis)
Virtual6 disconnected (normal left inverted right x axis y axis)
Virtual7 disconnected (normal left inverted right x axis y axis)
Virtual8 disconnected (normal left inverted right x axis y axis)
  1920x1080_60.00 (0x557) 173.000MHz -HSync +VSync
        h: width  1920 start 2048 end 2248 total 2576 skew    0 clock  67.16KHz
        v: height 1080 start 1083 end 1088 total 1120           clock  59.96Hz
```

再使用xrandr --addmode添加到某个显示器中，我这里是Virtual1

```bash
$ xrandr --addmode Virtual1 1920x1080_60.00
$ xrandr
Screen 0: minimum 1 x 1, current 1360 x 768, maximum 8192 x 8192
Virtual1 connected primary 1360x768+0+0 (normal left inverted right x axis y axis) 0mm x 0mm
   800x600       60.00 +  60.32  
   2560x1600     59.99  
   1920x1440     60.00  
   1856x1392     60.00  
   1792x1344     60.00  
   1920x1200     59.88  
   1600x1200     60.00  
   1680x1050     59.95  
   1400x1050     59.98  
   1280x1024     60.02  
   1440x900      59.89  
   1280x960      60.00  
   1360x768      60.02* 
   1280x800      59.81  
   1152x864      75.00  
   1280x768      59.87  
   1024x768      60.00  
   640x480       59.94  
   1920x1080_60.00  59.96  
Virtual2 disconnected (normal left inverted right x axis y axis)
Virtual3 disconnected (normal left inverted right x axis y axis)
Virtual4 disconnected (normal left inverted right x axis y axis)
Virtual5 disconnected (normal left inverted right x axis y axis)
Virtual6 disconnected (normal left inverted right x axis y axis)
Virtual7 disconnected (normal left inverted right x axis y axis)
Virtual8 disconnected (normal left inverted right x axis y axis)
```

这时候就可以选择此分辨率了

```bash
$ xrandr -s 1920x1080_60.00
```

## 5.11 压缩

### tar.gz

压缩：

```bash
$ tar zcf test.tar.gz test
```

解压：

```bash
$ tar -xf test.tar.gz -C [dir]
```

### tar.xz

压缩：

```bash
$ tar -cJf test.tar.xz test
```

解压：

```bash
$ tar -xJf test.tar.xz
```

### rar

压缩



解压

```bash
$ rar x test.rar
```





## 5.12 samba

安装：

```bash
$ sudo apt install samba
```

添加samba用户密码:

```bash
$ sudo smbpasswd -a xxx
```

修改配置文件:

```bash
$ sudo vi /etc/samba/smb.conf
```

在末尾添加

```config
[xxx]
   comment = xxx
   path = /home/xxx
   avlid users = xxx,root
   write list = xxx,root
   browseable = yes
   create mask = 0777
   directory mask = 0777
   writeable = yes
```

重启samba服务:

```bash
$ sudo service smbd restart
```

# 6 ISO

## 6.1查看磁盘UUID

```bash
ls -l /dev/disk/by-uuid
```

```
blkid -s UUID
```



## 6.2 efibootmgr

管理uefi启动项

查看:

```bash
efibootmgr
BootCurrent: 0005
Timeout: 5 seconds
BootOrder: 0001,0002,0003,0004,0005
Boot0001* Enter Setup
Boot0002* Update Bios
Boot0003* UEFI SM681GEE AES
Boot0004* UEFI Shell
Boot0005* UEFI aigo MiniKing 90009737AB26B655
```

删除:

```bash
efibootmgr --delete-bootnum --bootnum 5
```



```bash
2444945b-8416-8246-9fe4-06157710186e
ddebb8d2-1c14-4b2e-b9c9-4d905b926b19
```



```bash
mount -o bind /dev/ /mnt/dev/
mount -o bind /proc/ /mnt/proc/
mount -o bind /sys /mnt/sys

chroot /mnt
```



```bash
grub-install --target=arm64-efi --efi-directory=/boot/efi --bootloader-id=JARI
```

加--removable不回置顶

```
grub-install --target=arm64-efi --efi-directory=/boot/efi --bootloader-id=JARI --removable
```



```
efibootmgr --create --disk=/dev/nvme0n1 --part=1 --label="JARI" --loader='/mnt/boot/efi/EFI/JARI/grubaa64.efi'
```



```bash
efibootmgr -o 0010,0000,0011,0012,0013,0017,0018,0019,001A,001B,001C,001D,001E
```



# 7 MMU

MMU在主存中使用三层页表来存储完整的翻译信息，并且页表条目缓存在MMU中，以提供快速翻译，MMU提供的功能包括：

32位虚拟地址

36位物理地址

固定4k字节的页面大小

支持三层映射的稀疏地址空间

支持大型线性映射(4K, 256K, 16M, 4G bytes)

支持多个上下文

页面级保护

硬件错误处理



sparc MMU提供三个主要功能

它将每个正在运行的进程的虚拟地址转换为主存中的物理地址。这种映射是以4k字节的页为单位进行的，因此，例如，一个8mb的进程不需要位于主内存的连续区域中。任何虚拟页面都可以映射到任何可用的物理页面。

它提供内存保护，因此一个进程不能读写另一个进程的地址空间。对于大多数操作系统来说，允许多个进程同时安全地驻留在物理内存中是必要的。

它实现了虚拟内存。页表跟踪哪些页在主存中;如果内存引用发生在当前不驻留的页面上，MMU就会发出页面故障的信号。



MMU将CPU的虚拟地址转换为物理地址，32位的虚拟地址被转换为36位的物理地址,提供64 gb的物理地址空间，以支持大型物理内存和32位总线的内存映射,一个物理地址在逻辑上由一个4Kbyte页的偏移量和一个物理页号组成



页面总是按4k字节的边界对齐;因此，物理地址的低阶12位始终与虚拟地址的低阶12位相同，并且不需要转换。对于内存中驻留的每个有效虚拟页，都有一个对应的page Table Entry，其中包含虚拟页的物理页码。将虚拟地址转换为物理地址将虚拟页码替换为物理页码。



| Level-1 Table   | Level-2 Table   | Level-3 Table   |
| --------------- | --------------- | --------------- |
| Page Table Desc | Page Table Desc | Page Table Desc |
| 256(8bit)       | 64(6bit)        | 64(6bit)        |



MMU所需要的所有地址转换信息都驻留在主存的物理寻址数据结构中，MMU根据需要通过访问主存从这些数据结构中获取翻译。为了有效地支持稀疏寻址，虚拟地址空间的映射由多达三层的页表来完成。这些表的第一级和第二级通常(虽然不一定需要)包含指向下一级PTD)。一个第三级表条目总是一个PTE，它指向一个物理页面。当然一级或二级页表也可能是PTE。

根指针页表描述符对于每个上下文都是唯一的，可以在上下文表中找到。



虚拟地址的组成：

| Index 1 | Index 2 | Index 3 | Page Offset |
| ------- | ------- | ------- | ----------- |
| 31      | 23      | 17      | 11          |

每个index字段提供到页表相应级别的偏移量，很少需要一组完整的表，因为虚拟地址空间通常很少填充，在某些情况下，不需要完整的三层页表来获得页表条目，当线性内存的256K、16M或4g段被映射为单个Page Table Entry时，就会发生这种情况。

如果为了将虚拟地址转换为物理地址，每个CPU内存引用都需要遵循主内存中的三层页表，那么CPU内存引用将会太慢。

因此，页表条目被缓存在MMU的页面描述符缓存，简称PDC(通常称为翻译lookaside缓冲区，简称TLB)



虚拟地址进入MMU并锁存在一个内部寄存器中，然后将它与存储在PDC中的虚拟地址标记进行比较。

匹配PDC中的PTE

若匹配上则说明该PTE已经存储在MMU中，然后物理地址直接生成。

如果没有匹配上，则发生报错。

在出现错误时MMU自动接管CPU的地址和数据总线，获取Page Table描述符，直到到达所需要的PTE。

然后PTE被缓存在MMU中，翻译发生，并且原始存储器请求从锁存地址继续。

检查每个翻译的内存访问权限，如果请求权限被拒绝，则会报错。

如果产生报错，则相应的错误信息则会被记录在Fault Status Register和e Fault Address Regsiter，并产生处理器故障

MMU会同时保留几个进程地址空间的转换，这加快了进程之间的上下文切换，每个地址空间由一个“上下文”数字标识，可能被系统用来在一个虚拟缓存中维护多个进程。多上下文的管理，包括将上下文分配给流程，回收未使用的上下文和重新分配上下文，这是内存管理软件的职责



PTD(页表描述符):

页表描述符(PTD)包含页表的物理地址，并定义了上下文表(Context Table)、一级页表和二级页表。PTD的定义如下:
| PTP  | ET   |
| ---- | ---- |
| 31   |1     |

PTP:Page Table Pointer 下一级页表头的物理地址。在错误处理期间，PTP出现在物理地址总线的第35位到第6位。PTP指向的页表必须在与页表大小相等的边界上对齐

| Level | Page Table Size |
| ----- | --------------- |
| 1     | 1024            |
| 2     | 256             |
| 3     | 256             |

ET(入口类型):用来区分PTD和PTE

| ET   | Type     |
| ---- | -------- |
| 0    | Invalid  |
| 1    | PTD      |
| 2    | PTE      |
| 3    | Reserved |

“Invalid”意味着对应的虚拟地址范围目前没有映射到物理地址。

PTE(页表条目):指定了页面的物理地址和访问权限。

| PPN  | C    | M    | R    | ACC  | ET   |
| ---- | ---- | ---- | ---- | ---- | ---- |
| 31   | 8    | 6    | 5    | 4    | 1    |

PPN(Physical Page Number):

当转换完成时，PPN出现在物理地址总线的第35位到第12位上。

C(Cacheable):

如果该位为1，则该页可由指令和/或数据缓存缓存

编程注意

MMU映射的所有输入/输出(I/O)位置都应该将对应pte中的C位设为0

M(Modified)

当对页面进行写入访问时，MMU将该位设置为1（除非是通过参考MMU直通访问ASI）

R(Referenced)

当对页面进行读访问时，MMU将该位设置为1（除非是通过参考MMU直通访问ASI）

ACC（Access Permissions）

这些位表示是否允许正在尝试的事务访问该页。地址空间访问中使用的标识符确定是数据访问还是指令访问，以及用户或管理软件是否正在尝试访问。



|      |                                  |                                       |
| ---- | -------------------------------- | ------------------------------------- |
| ACC  | 用户权限(ASI = 0x8 or ASI = 0xA) | 超级用户权限 (ASI = 0x9 or ASI = 0xB) |
| 0    | Read Only                        | Read Only                             |
| 1    | Read/Write                       | Read/Write                            |
| 2    | Read/Execute                     | Read/Execute                          |
| 3    | Read/Write/Execute               | Read/Write/Execute                    |
| 4    | Execute Only                     | Execute Only                          |
| 5    | Read Only                        | Read/Write                            |
| 6    | No Access                        | Read/Execute                          |
| 7    | No Access                        | Read/Write/Execute                    |

ET(Entry Type)

该字段将PTE与PTD区分开来。



如果在Context Table或Level-1或Level-2 Page Table中发现一个PTE，则地址转换过程停止，使用该PTE。虚拟地址从PTE对应的第一个虚拟地址到PTE对应的最后一个虚拟地址，线性映射到PPN指定的物理地址。PPN指定的物理地址必须在与PTE映射的区域大小相等的边界上对齐。



例如，给定一个虚拟地址如下：

| I1   | I2   | I3   | offset |
| ---- | ---- | ---- | ------ |
| 31   | 23   | 17   | 11     |

假设适当的Level-2 Page Table的I2条目是一个PTE，使用包含值PPN的物理页码字段，对应的物理地址是按位排列的

| 0    | 0    | 0    | I3   | offset |
| ---- | ---- | ---- | ---- | ------ |
| 35   | 31   | 23   | 17   | 11     |

和

| PNN  | 0    |
| ---- | ---- |
| 35   | 11   |

注意，PPN的低阶6位必须全部为零，以满足PTE映射区域的对齐要求。

下表汇总了页表中不同级别映射的区域的大小：

按MMU、Page Table Level映射的区域大小

| Level | Mapping Size  |
| ----- | ------------- |
| 3     | 4 Kilobytes   |
| 2     | 256 Kilobytes |
| 1     | 16 Megabytes  |
| Root  | 4 Gigabytes   |

ACC字段值为6或7的Page Table Entry表示主管页。如果页面描述符缓存在匹配此类引用的缓存标记时忽略用于获取PTE的上下文编号，则转换管理器地址会更有效。

MMU应该为Level-1或Level-2 PTE使用一个PDC表项，为总线、协处理器和内核提供一个大的线性地址映射，而不需要很多转换缓存表项。



MMU Flush和Probe模型

特权加载和存储交替指令用于从MMU的页面描述符缓存(PDC)中刷新条目，并探测MMU中的条目。

在用于刷新和探测的备用地址空间中，地址如下所示组成

| VFPA | Type | reserved |
| ---- | ---- | -------- |
| 31   | 11   | 7        |

VFPA

虚拟刷新或探测地址

Type

“Type”字段指示要从PDC中刷新的对象或要探测的对象。下表描述了Type字段的编码。

| Type        | Probe Object    | Flush PDC Object(s) |
| ----------- | --------------- | ------------------- |
| 0 (page)    | Level-3 entry   |                     |
| 1 (segment) | Level-2 entry   |                     |
| 2 (region)  | Level-1 entry   |                     |
| 3 (context) | Level-0 entry   |                     |
| 4 (entire)  | Level-n entry   |                     |
| 5 − 0xF     | none (reserved) | none (reserved)     |

注意，type 5-0xF会被忽略。

reserved字段是保留的，应该由软件提供为零



刷新操作：

刷新操作将从PDC中删除匹配Type字段27隐含条件的PTE或PTD。刷新是通过对适当的地址(由VFPA字段给出)、适当的类型(由type字段给出)、适当的上下文(由context寄存器给出)和适当的地址空间标识符(ASI)(由实现决定)执行存储交替指令来完成的。由存储替代指令提供的数据将被忽略。

对象中删除Type字段指定的对象PDC。以下段落描述了flush标准。由于在特定实现中的刷新操作从PDC中删除的条目可能比Type字段指定的更多，因此“精确”刷新被定义为删除Type字段隐含的最小条目数。

如果PTE的访问代码表示管理页面(PTE. acc = 6或7)，则页、段或区域刷新(类型0-2)删除PTE。如果PTE的上下文标记等于上下文寄存器，则精确页、段或区域刷新删除用户页面(PTE. acc = 0到5)。此外，如果PTE的地址标记等于VFPA字段的相应位，则精确页、段或区域刷新将删除PTE。此外，如果level标记表明PTE是从第3级页表中获取的，那么精确的页刷新将删除PTE;精确分段冲洗删除带有2级或3级标记的PTE。

如果PTE的上下文标记等于上下文寄存器，并且PTE的访问代码表示用户页面(PTE. acc = 0-5)，则精确上下文刷新(类型3)删除PTE。不精确的上下文刷新也可能会删除管理器条目。PTE的地址和级别标记被忽略。

整个刷新(类型4)删除pte，而不管它们的地址标记、上下文标记、级别标记和ACC代码的值是多少。换句话说，整个PDC都被冲洗了。

如果PTD的上下文标记等于上下文寄存器，且级别标记对应于刷新类型，则该PTD将刷新。精确分段冲洗使用所二级标签。精确区域刷新删除具有1级或2级标记的PTD。精确上下文刷新删除具有级别0、1或2标记的PTD。一次完整的冲洗会将所有ptd从PDC中移除。

下表总结了PTE刷新匹配条件：

页表项刷新匹配条件

| VA[11:8] | Flush Type | Precise PTE Flush Match Criteria                |
| -------- | ---------- | ----------------------------------------------- |
| 0        | Page       | ((ACC≥6) or Contexts_equal) and VA[31:12]_equal |
| 1        | Segment    | ((ACC≥6) or Contexts_equal) and VA[31:18]_equal |
| 2        | Region     | ((ACC≥6) or Contexts_equal) and VA[31:24]_equal |
| 3        | Context    | (ACC≤5) and Contexts_equal                      |
| 4        | Entire     | —                                               |
| 5 − 0xF  | reserved   | —                                               |

下表总结了PTD刷新匹配条件。注意，这些标准与PTE的标准相同，只是没有访问代码检查。

页表描述符刷新匹配条件

| VA[11:8] | Flush Type | Precise PTD Flush Match Criteria   |
| -------- | ---------- | ---------------------------------- |
| 0        | Page       | Contexts_equal and VA[31:12]_equal |
| 1        | Segment    | Contexts_equal and VA[31:18]_equal |
| 2        | Region     | Contexts_equal and VA[31:24]_equal |
| 3        | Context    | Contexts_equal                     |
| 4        | Entire     | —                                  |
| 5 − 0xF  | reserved   | —                                  |



探针操作：

探针从PDC或主内存中的页表返回一个条目，或者生成一个错误。探测是通过执行具有适当地址(由VFPA字段)，类型(由type字段给出)，上下文(由上下文寄存器给出)，以及地址空间标识符(ASI)——最后一个是依赖于实现的。

在探测操作过程中可能会发生两类错误:

在探测层到达之前，会遇到ET≠1(PTD)的条目。

内存错误。不会向处理器发出内存访问异常的信号，但是会更新错误寄存器。

如果出现上述任何一个错误，探测操作将返回零值。如果探测操作成功，它将从页表中返回Type字段所隐含的相应条目。

探测操作返回的值在下表中指定，对于给定的探测类型，从左到右读取表。“0”表示返回0， “·”指示返回页表项本身，“⇒”表示检查了下一级页表项

|             |                     |         |       |      |      |      |      |      |      |      |      |      |      |      |      |      |      |
| ----------- | ------------------- | ------- | ----- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| Probe Type  | Upon a Memory Error | Level-0 | Entry |      |      |      |      |      |      |      |      |      |      |      |      |      |      |
| 0 (page)    | 0                   | 0       | 0     | 0    |      |      |      |      |      |      |      |      |      |      |      |      |      |
| 1 (segment) | 0                   | 0       | 0     | 0    |      |      |      |      |      |      |      |      |      |      |      |      |      |
| 2 (region)  | 0                   | 0       | 0     |      |      |      |      |      |      |      |      |      |      |      |      |      |      |
| 3 (context) | 0                   | .       | 0     |      |      |      |      |      |      |      |      |      |      |      |      |      |      |
| 4 (entire)  | 0                   | .       | 0     |      |      |      |      |      |      |      |      |      |      |      |      |      |      |
| 5−0XF       | 0                   |         |       |      |      |      |      |      |      |      |      |      |      |      |      |      |      |













# 附录

## 动态库测试demo

首先准备一下三个文件

hello.c：

```c
#include<stdio.h>
#include"hello.h"

int print_hello(void)
{
	printf("hello\n");
}
```

hello.h

```c
#ifndef __PRINTF_HELLO_
#define __PRINTF_HELLO_

int print_hello(void);

#endif
```

main.c

```c
#include<stdio.h>
#include"hello.h"

int main(void)
{
	print_hello();
}
```

编译动态库:

```bash
$ gcc hello.c -fPIC -shared -o libhello.so
```

编译main：

```bash
$ gcc main.c -L. -lhello -o main
```

设置程序动态库位置:

```bash
$ export LD_LIBRARY_PATH=`pwd`
```

运行:

```bash
$ ./main
```

结果:

```bash
hello
```
## 静态库测试demo

首先准备一下三个文件

hello.c：

```c
#include<stdio.h>
#include"hello.h"

int print_hello(void)
{
	printf("hello\n");
}	
```

hello.h

```c
#ifndef __PRINTF_HELLO_
#define __PRINTF_HELLO_

int print_hello(void);

#endif
```

main.c

```c
#include<stdio.h>
#include"hello.h"

int main(void)
{
	print_hello();
}
```

编译静态库

```bash
$ gcc hello.c -c -o hello.o
$ ar -rc libhello.a hello.o
```

编译main

```bash
$ gcc main.c -L. -lhello -static -o main
```

## 内核模块动态变量demo

编译测试内核模块文件和Makefile

demo_modules.c

```c
#include <linux/module.h>
#include <linux/kernel.h>

static int int_var = 0;
static char *str_var = "default";
static int int_array[6];
int narr;

module_param(int_var, int, 0644);
MODULE_PARM_DESC(int_var, "int var");

module_param(str_var, charp, 0644);
MODULE_PARM_DESC(str_var, "string var");

module_param_array(int_array, int, &narr, 0644);
MODULE_PARM_DESC(int_array, "A integer array");
 

static int __init demo_init(void)
{
       int i;
       printk("int_var %d.\n", int_var);
       printk("str_var %s.\n", str_var);

       for(i = 0; i < narr; i ++){
               printk("int_array[%d] = %d\n", i, int_array[i]);
       }
       return 0;
}

static void __exit demo_exit(void)
{
}
module_init(demo_init);
module_exit(demo_exit);
MODULE_LICENSE("GPL");
```

Makefile

```makefile
# Makefile 4.0
obj-m := demo_module.o 
CURRENT_PATH := $(shell pwd)

LINUX_KERNEL := $(shell uname -r)
LINUX_KERNEL_PATH := /usr/src/linux-headers-$(LINUX_KERNEL)

EXTRA_CFLAGS += -Wfatal-errors

all:
	make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) modules

help:
	make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) help

clean:
	make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) clean
```

编译

```bash
$ make
```

插入内核模块

```bash
$ insmod demo_module.ko int_var=456 str_var=hello int_array=100,200,300
```

查看内核打印

```bash
$ dmesg   
```

结果:

```bash
[403345.951099] int_var 456.
[403345.951103] str_var hello.
[403345.951104] int_array[0] = 100
[403345.951105] int_array[1] = 200
[403345.951105] int_array[2] = 300
```

查看sysfs文件:

```bash
$ cd /sys/module/demo_module/parameters
$ cat int_array 
$ cat int_var 
$ cat str_var
```

得到的结果如下:

```bash
$ cat int_array                        
100,200,300
$ cat int_var  
456
$ cat str_var              
hello
```

使用modinfo查看参数注释:

```bash
$ modinfo demo_module.ko
```

结果:

```bash
parm:           int_var:int var (int)
parm:           str_var:string var (charp)
parm:           int_array:A integer array (array of int)
```
