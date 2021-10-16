## linux内核模块

### linux内核模块简介

​		linux内核的整体架构非常庞大，其包含的组件也非常多。linux提供了一种机制，使得编译出的内核本身并不需要包含所有功能，而在这些功能**需要被使用的时候，其对应的代码被动地加载到内核中**，这种机制被称为**模块**。

​		模块具有以下**特点**：

- 模块本身不被编译如内核映像，从而**控制了内核的大小**；

- 模块一旦被加载，它就**和内核中的其他部分完全一样**。

  为了对模块建立感性认识，先看一个最简单的内核模块“Hello World”

```c
#include <linux/init.h>
#include <linux/module.h>

static int __init hello_init(void)
{
    printk(KERN_INFO "Hello World enter\n");
    return 0;
}

static void __exit hello_exit(void)
{
    printk(KERN_INFO "Hello World exit\n");
}

module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL v2");
MODULE_DESCRIPTION("A simple Hello World Module");
MODULE_ALIAS("a simplest module");
```

​		这个最简单的内核模块只包含内核模块加载函数、卸载函数和对GPL v2许可权限的声明以及一些描述信息。编译它会产生hello.ko目标文件，通过“insmod ./hello.ko”命令可以加载它，通过“rmmod hello”命令可以卸载它，加载时输出“Hello World enter”，卸载时输出“Hello World exit”。

注：内核模块中用于输出的函数时内核空间的printk()，而不是用户空间的printf()。

​		modprobe命令比insmod命令要强大，它在加载某模块时，会同时加载该模块所依赖的其他模块。使用modprobe命令加载的模块若以“modprobe -r filename”的方式卸载，将同时卸载其他依赖的模块。

### linux内核模块程序结构

一个linux内核模块主要由如下几个部分组成：

1. 模块加载函数

   ​		当通过insmod或modprobe命令加载内核模块时，模块的加载函数会自动被内核执行，完成本模块的相关初始化工作。

2. 模块卸载函数

   ​		当通过rmmod命令卸载某模块时，模块的卸载函数会自动被内核执行，完成与模块卸载函数相反的功能。

3. 模块许可证声明

   ​		许可证声明描述内核模块的许可权限，如果不声明LICENSE，模块被加载时，将收到内核被污染的警告。

4. 模块参数（可选）

   ​		模块参数是模块被加载的时候可以传递给它的值，它本身对应模块内部的全局变量。

5. 模块导出符号（可选）

   ​		内核模块可以导出的符号（symbol，对应于函数或变量），若导出，其他模块则可以使用本模块中的变量或函数。

6. 模块作者等信息声明（可选）

### 模块加载函数

​		linux内核模块加载函数一般以`__init`标识声明，典型的模块加载函数的形式如下所示。

```c
static int __init initialization_function(void)
{
    /* 初始化代码 */
    return 0;
}
module_init(initialization_function);
```

​		模块加载函数以“module_init(函数名)”的形式被指定。它返回整型值，若初始化成功，应返回0。而在初始化失败时，应返回错误编码。总是返回相应的错误编码是种非常好的习惯，因为只有这样，用户程序才可以利用perror等方法把它们转换成有意义的错误信息字符串。

​		在linux内核中，可以使用request_module(const char *fmt, ...)函数加载内核模块，驱动开发人员通过调用下列代码,灵活地加载其他内核模块。

```c
request_module(module_name);
```

​		在linux中，所有标识为`__init`的函数如果直接编译进入内核，成为内核镜像的一部分，在连接的时候都会放在`.init.text`这个区段内。所有的`__init`函数在区段`.initcall.init`中还保存了一份函数指针，在初始化时内核会通过这些函数指针调用这些`__init`函数，并在初始化完成后，释放`init`区段（包括`.init.text`、`.initcall.init`等）的内存。

​		除了函数以外，数据也可以被定义为`__initdata`，对于只是初始化阶段需要的数据，内核在初始化完后，也可以释放它们占用的内存。

### 模块卸载函数

​		linux内核模块加载函数一般以`__exit`标识声明，典型的模块卸载函数的形式如下所示。

```c
static void __exit cleanup_function(void)
{
    /* 释放代码 */
}
module_exit(cleanup_function);
```

​		模块卸载函数在模块卸载的时候执行，而不返回任何值，且必须以`“module_exit(函数名)”`的形式被指定。通常来说，模块卸载函数要完成与模块加载函数相反的功能。

​		用`__exit`来修饰模块卸载函数，可以告诉内核如果相关的模块被直接编译进内核（即`built-in`），则`cleanup_function`函数会被省略，直接不链进最后的镜像。既然模块被内置了，就不可能卸载它了，卸载函数也就没有存在的必要了。除了函数以外，只是退出阶段采用的数据也可以用`__exitdata`。

### 模块参数

​		可以使用`module_param(参数名，参数类型，参数读/写权限)`为模块定义一个参数。

例如下列代码定义了1个整型参数和1个字符指针参数：

```c
static char *book_name = "dissecting Linux Device Driver";
module_param(book_name, charp, S_IRUGO);

static int book_num = 400;
module_param(book_num, int, S_IRUGO);
```

​		在装载内核模块时，用户可以向模块传递参数，形式为`insmode(或modprobe) 模块名 参数名 = 参数值`，如果不传递，参数将使用模块内定义的缺省值。如果模块被内置，就无法insmod了，但是bootloader可以通过bootargs里设置“模块名.参数名 = 值”的形式给该内置的模块传递参数。

​		参数类型可以是`byte short ushort int uint long ulong charp(字符指针) bool或invbool(布尔的反)`。除此之外，模块也可以拥有参数数组，形式为“`module_param_array(数组名，数组类型，数组长，参数读/写权限)`”。运行insmod或modprobe命令时，应使用逗号分隔输入的数组元素。

### 导出符号

​		linux的`"/proc/kallsyms"`文件对应着内核符号表，它记录了符号以及符号所在的内存地址。模块可以使用如下宏导出符号到内核符号表中：

```c
EXPORT_SYMBOL(符号名);
EXPORT_SYMBOL_GPL(符号名);
```

​		导出的符号可以被其他模块使用，只需要使用前声明一下即可。`EXPORT_SYMBOL_GPL()`只适用于包含GPL许可权的模块。

```c
#include <linux/init.h>
#include <linux/module.h>

int add_integar(int a, int b)
{
    return a + b;
}

int sub_integar(int a, int b)
{
    return a - b;
}

EXPORT_SYMBOL_GPL(add_integar);
EXPORT_SYMBOL_GPL(sub_integar);
MODULE_LICENSE("GPL v2");
```

### 模块声明与描述

​		在linux内核模块中，可以用`MODULE_AUTHOR MODULE_DESCRIPTION MODULE_VERSION MODULE_DEVICE_TABLE MODULE_ALIAS`分别声明模块的作者、描述、版本、设备表和别名。

​		对于USB、PCI等设备驱动，通常会创建一个`MODULE_DEVICE_TABLE `，以表明该驱动模块所支持的设备，代码如下：



### 模块的使用计数

​			在linux2.6以后的内核提供了模块计数管理接口`try_module_get(&module)`和`module_put(&module)`，从而取代linux2.4内核中的模块使用计数管理宏。模块的使用计数一般不必由模块自身管理，而且模块计数管理还考虑了SMP与PREEMPT机制的影响。

```c
int try_module_get(struct module *module);
```

​		该函数用于增加模块使用计数；若返回为0，表示调用失败，希望使用的模块没有被加载或正在被卸载中。

```c
void module_put(struct module *module);
```

​		该函数用于减少模块使用计数。

​		linux2.6以后的内核为不同类型的设备定义了`struct module *owner`域，用来指向管理此设备的模块。当开始使用某个设备时，内核使用`try_module_get(dev->owner)`去增加管理此设备的`owner`模块的使用计数；当不再使用此设备时，内核使用`module_put(dev->owner)`减少对管理此设备的管理模块的使用计数。

### 使用模块“绕开”GPL

