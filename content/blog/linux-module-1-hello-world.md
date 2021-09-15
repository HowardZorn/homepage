+++
title = "Linux内核模块编写之1: Hello World及Makefile"
date = 2020-03-28T17:51:34+08:00
draft = true
toc = true
[[copyright]]
  owner = "贺若舟"
  date = "2021"
  license = "cc-by-nd-4.0"
+++

## 内核模块
内核模块不是用户程序，运行方式上有很多不同之处。编译过程上也是有显著不同。

我们平时所用的头文件大多在`/usr/include`，这个目录是用来存放各种在用户态下运行的库的C/C++头文件。而Linux内核模块编译时需要使用Linux内核的源代码(主要是头文件)，运行时也是在内核态中，不允许使用各种用户态库。`/usr/include`下的各类头文件也就无法被我们利用了。

所以我们得找到Linux内核的头文件所在地。当然了，在这一步之前需要自行安装对应的软件包(在Arch Linux是`linux-headers`)。如果在你使用的系统是Arch Linux，可以通过`pacman -Fl linux-headers`查看Linux内核的头文件所在的位置。在Arch Linux下通常是`/usr/lib/modules/$(uname -r)/build/`(随内核版本而变)或者`/usr/src/linux`(“固定的”，其实是`linux-headers`软件包所带的一个软链接)。
```
/usr/src $ ll
total 0
lrwxrwxrwx 1 root root 35 Mar 18 16:40 linux -> ../lib/modules/5.5.10-arch1-1/build
```
你可以在这个目录找到一个Makefile，之后会用上。

找到了这些头文件，就可以开始着手准备Makefile了。之所以选择GNU Make作为构建系统，是因为Linus只给我们这个选项。之前找到的那个Makefile就是关键。

现在直接贴上我写好的Makefile，以后都可以在这个基础上修改：
```Makefile
# 你的模块的名字
NAME = ModuleName
# 内核的源码头文件目录
# 也可以是 /usr/lib/modules/$(shell uname -r)/build/
KDIR := /usr/src/linux/

# 最后生成的*.ko的名字，与前面的NAME对应
obj-m := $(NAME).o

# 源文件的名字.o
$(NAME)-objs := YourSrcFiles0.o YourSrcFiles1.o

# 当前目录，给之后的M赋值
PWD := $(shell pwd)

# 进入内核源码头文件目录，调用该目录下的Makefile
# 设置M=PWD，M是内核源码头文件目录下的Makefile的一个变量
# 用来指示生成的模块存放的目录
build:
    $(MAKE) -C $(KDIR) M=$(PWD) modules
# 这样clean比较方便
clean: 
    $(MAKE) -C $(KDIR) M=$(PWD) clean
# 安装模块，需要权限
insmod:
    sudo insmod $(NAME).ko
# 卸载模块，需要权限
rmmod:
    sudo rmmod $(NAME)
```
这样我们就能用`make`命令来编译一个模块了。

## Hello, World!

这种简单的工程只需要单个文件就好了。代码如下：

```c hello.c
#include <linux/module.h> // 所有模块都需要
#include <linux/kernel.h> // printk和KERN_INFO等等
#include <linux/init.h>   // __init、__exit的定义

// 声明模块是GPL开源
// 有些符号只提供给声明了GPL的模块
MODULE_LICENSE("GPL"); 
// 作者信息
MODULE_AUTHOR("Fw[a]rd");
// 模块描述
MODULE_DESCRIPTION("A test module");
// 版本号
MODULE_VERSION("1:1.0");
// 以上都可以通过modinfo查看

// 加载模块调用的函数，需要用__init宏
static int __init hello_init(void)
{
    // 用户态的printf()不能使用，所以得用printk()
    // 采用这样的格式"Hello: %s"
    // 是因为这样可以在dmesg下可以变成比较漂亮的格式
    printk(KERN_INFO "Hello: Hello, World!\n");
    // 0是加载成功的标志
    return 0;
}

// 卸载模块调用的函数，需要用__exit宏
static void __exit hello_exit(void)
{   
    printk(KERN_INFO "Hello: Goodbye, crazy world!\n");
}

// 定义模块的加载卸载函数
module_init(hello_init);
module_exit(hello_exit);

```
然后就要编译这个模块了。我们设置Makefile的`NAME=Hello`，假设你的源文件名为hello.c，那么`$(NAME)-objs := hello.o`，接下来输入make就可以编译了，成功之后生成Hello.ko。

输入modinfo Hello.ko，可以看到我们生成的模块的信息：
```
$ modinfo Hello.ko
filename:       /home/********/Hello.ko
description:    A test module
author:         Fw[a]rd
license:        GPL
srcversion:     B581BFF80AEE7DA22B645F5
depends:        
retpoline:      Y
name:           Hello
vermagic:       5.5.9-arch1-2 SMP preempt mod_unload 
```
用`make insmod`挂载之，然后再用`make rmmod`卸载。之后用`dmesg | Hello:`查看我们写入的日志。
```
$ make insmod 
sudo insmod Hello.ko
[sudo] password for ****: 
$ make rmmod 
sudo rmmod Hello
$ dmesg | grep Hello:
[ 3637.190841] Hello: Hello, World!
[ 3642.641519] Hello: Goodbye, crazy world!
```
成功写入。以上就是编写一个Hello, World内核模块的全过程。
