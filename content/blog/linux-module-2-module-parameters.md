+++
title = "Linux内核模块编写之2: 内核模块的参数"
date = 2020-09-17T17:51:34+08:00
draft = true
categories = ["Linux", "C"]
toc = true
[[copyright]]
  owner = "贺若舟"
  date = "2021"
  license = "cc-by-nd-4.0"
+++

## 用户程序参数

我们都知道，很多用户程序是可以在控制台接受参数的。就像下面这样：

```
$ program --opt=arg1 -o arg2 ...
```

在C/C++中，通过main函数的参数来接受外部输入的参数：
```c
int main(int argc, char* argv[]) {
    // getopt() ...
}
```

我们是否能将参数传给内核模块呢？答案很显然，是的，但是又有点不一样。下面来介绍怎么用。

## 模块参数

模块参数必须通过名称指定。下面以**万恶的**高通的某个折腾了我半年的WiFi网卡驱动为例子：

```
 $ modinfo ath10k_pci
filename:       /lib/modules/5.8.7-arch1-1/kernel/drivers/net/wireless/ath/ath10k/ath10k_pci.ko.xz
（中略）
license:        Dual BSD/GPL
description:    Driver support for Qualcomm Atheros 802.11ac WLAN PCIe/AHB devices
author:         Qualcomm Atheros
srcversion:     EAC646D85B74443C90A8D46
depends:        ath10k_core
retpoline:      Y
intree:         Y
name:           ath10k_pci
vermagic:       5.8.7-arch1-1 SMP preempt mod_unload 
（中略）
parm:           irq_mode:0: auto, 1: legacy, 2: msi (default: 0) (uint)
parm:           reset_mode:0: auto, 1: warm only (default: 0) (uint)
```
看到最后的`parm`了吗？随后跟着的就是参数的名称。我们通常在加载模块的时候指定参数。语法为：

```
 # insmod ath10k irq_mode=0 reset_mode=0
```

如果模块允许，你还可以在加载模块之后读取和修改参数。这需要使用sysfs，读写`/sys/module/${模块名}/parmeters`下的文件。可以看到ath10k的参数任何用户都可以读，但只有root用户可以写。（我不演示写啊，现在正用这个模块上网呢）

```
 $ ll /sys/module/ath10k_pci/parameters         
total 0
-rw-r--r-- 1 root root 4.0K Sep 17 17:55 irq_mode
-rw-r--r-- 1 root root 4.0K Sep 17 17:55 reset_mode
 $ cat /sys/module/ath10k_pci/parameters/irq_mode 
0
 $ cat /sys/module/ath10k_pci/parameters/reset_mode
0
```

具体怎么在模块中定义参数呢？我们不是在模块的init函数上加上参数。我们将用到module_param系列宏来定义可以传入的参数：

```c
module_param(name, type, perm)
module_param_array(name, type, nump, perm)
module_param_string(name, string, len, perm)
module_param_cb(name, ops, arg, perm)
// 还有不少...
```
这系列宏函数都在`<linux/moduleparam.h>`下面找的到。使用的时候也需要include一下这个头文件。这个系列的宏函数有一些通用的参数，`name`填入参数名称，同时也是储存这个参数的变量名；`type`填入这个模块参数的类型标识符；`perm`是个表示UNIX权限的`int`，表示这个参数在sysfs(`/sys/module/${模块名}/parmeters`)下的访问控制权限，可以用八进制表示法表示（`0777`），也可以用`S_I*`宏组合表示(`S_IRWXU|S_IRWXG|S_IRWXO`)。

这些`S_I*`宏具体的定义如下：

```c
#define S_IRWXU 00700
#define S_IRUSR 00400
#define S_IWUSR 00200
#define S_IXUSR 00100

#define S_IRWXG 00070
#define S_IRGRP 00040
#define S_IWGRP 00020
#define S_IXGRP 00010

#define S_IRWXO 00007
#define S_IROTH 00004
#define S_IWOTH 00002
#define S_IXOTH 00001
```
有些特殊的`module_param`宏要额外说明。
- 对于`module_param_string`这个宏函数，有些参数的用法有点不一样，`name`填入参数名称，但可以不是变量名，`string`填入字符串的变量名；`len`是来标识包括`'\0'`在内的最长字符串长度的`int`。
- `module_param_array`里的`nump`需要填入一个`int*`，是用来记录传入的数组长度的。

怎么定义模块参数的默认值？因为要传入变量名，所以一般在初始化存储模块参数的变量的时候定义。

接下来就用个例子来示范一下怎么用这些玩意儿。

```c hello_param.c
#include <linux/module.h>
#include <linux/moduleparam.h> // module_param系列宏
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/stat.h>
#include <linux/types.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Fw[a]rd");

static short int myshort = 1;
static int myint = 430;
static long int mylong = 9999;
static char mystring[255] = {'b', 'l', 'a', 'h', '\0', };
static int myintArray[255] = { -1, -1, };
static int arr_argc = 2;

module_param(myshort, short, S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP);
MODULE_PARM_DESC(myshort, "A short integer");
module_param(myint, int, S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH);
MODULE_PARM_DESC(myint, "An integer");
module_param(mylong, long, S_IRUSR);
MODULE_PARM_DESC(mylong, "A long integer");
module_param_string(myString, mystring, 255, 0644);
MODULE_PARM_DESC(myString, "A character string");
module_param_array(myintArray, int, &arr_argc, 0000);
MODULE_PARM_DESC(myintArray, "An array of integers");

static int __init hello_param_init(void)
{
	int i;
	printk(KERN_INFO "hello_param: Hello, world\n");
	printk(KERN_INFO "hello_param: myshort is a short integer: %hd\n", myshort);
	printk(KERN_INFO "hello_param: myint is an integer: %d\n", myint);
	printk(KERN_INFO "hello_param: mylong is a long integer: %ld\n", mylong);
	printk(KERN_INFO "hello_param: myString is a string: %s\n", mystring);
	for (i = 0; i < arr_argc; i++)
	{
		printk(KERN_INFO "hello_param: myintArray[%d] = %d\n", i, myintArray[i]);
	}
	printk(KERN_INFO "hello_param: got %d arguments for myintArray.\n", arr_argc);
	return 0;
}

static void __exit hello_param_exit(void)
{
	int i;
	printk(KERN_INFO "hello_param: Hello, world\n");
	printk(KERN_INFO "hello_param: myshort is a short integer: %hd\n", myshort);
	printk(KERN_INFO "hello_param: myint is an integer: %d\n", myint);
	printk(KERN_INFO "hello_param: mylong is a long integer: %ld\n", mylong);
	printk(KERN_INFO "hello_param: myString is a string: %s\n", mystring);
	for (i = 0; i < arr_argc; i++)
	{
		printk(KERN_INFO "hello_param: myintArray[%d] = %d\n", i, myintArray[i]);
	}
	printk(KERN_INFO "hello_param: got %d arguments for myintArray.\n", arr_argc);
	printk(KERN_INFO "hello_param: Goodbye, world.\n");
}

module_init(hello_param_init);
module_exit(hello_param_exit);
```
这个例子改编自[The Linux Kernel Module Programming Guide](https://tldp.org/LDP/lkmpg/2.6/html/index.html)，虽然是v2.6时代的东西，但是功能仍然基本正常。

编译完了用`modinfo`观察下，发现已经记录到`parm`信息中了。

```
 $ modinfo ./HelloParam.ko
filename:       ./HelloParam.ko
author:         Fw[a]rd
license:        GPL
srcversion:     8CD5ED754583B00842CFF4F
depends:        
retpoline:      Y
name:           HelloParam
vermagic:       5.8.7-arch1-1 SMP preempt mod_unload 
parm:           myshort:A short integer (short)
parm:           myint:An integer (int)
parm:           mylong:A long integer (long)
parm:           myString:A character string (string)
parm:           myintArray:An array of integers (array of int)
```

加载之：

```
 # insmod ./HelloParam.ko myint=820 myString="Hello,Parameters!" myintArray=1,2,3,4,5
 # dmesg | tail -n 11
[ 7475.646004] hello_param: Hello, world
[ 7475.646008] hello_param: myshort is a short integer: 1
[ 7475.646010] hello_param: myint is an integer: 820
[ 7475.646012] hello_param: mylong is a long integer: 9999
[ 7475.646014] hello_param: myString is a string: Hello,Parameters!
[ 7475.646016] hello_param: myintArray[0] = 1
[ 7475.646018] hello_param: myintArray[1] = 2
[ 7475.646019] hello_param: myintArray[2] = 3
[ 7475.646021] hello_param: myintArray[3] = 4
[ 7475.646023] hello_param: myintArray[4] = 5
[ 7475.646024] hello_param: got 5 arguments for myintArray.
```

看来没毛病。接下来修改下参数：

```
 # ls -l /sys/module/HelloParam/parameters/
total 0
-rw-r--r-- 1 root root 4096 Sep 17 18:51 myint
-r-------- 1 root root 4096 Sep 17 18:51 mylong
-rw-rw---- 1 root root 4096 Sep 17 18:51 myshort
-rw-r--r-- 1 root root 4096 Sep 17 18:51 myString
 # #可以看到没有显示权限为0000的myintArray
 # echo 233 > /sys/module/HelloParam/parameters/myshort
 # echo 233 > /sys/module/HelloParam/parameters/myint
 # echo 233 > /sys/module/HelloParam/parameters/mylong
zsh: permission denied: /sys/module/HelloParam/parameters/mylong
 # #只能读
 # echo "Bye! See you later nerd." > /sys/module/HelloParam/parameters/myString 
 # rmmod HelloParam
 # dmesg | tail -n 11
[ 8438.018906] hello_param: myint is an integer: 233
[ 8438.018908] hello_param: mylong is a long integer: 9999
[ 8438.018910] hello_param: myString is a string: Bye! See you later nerd.

[ 8438.018912] hello_param: myintArray[0] = 1
[ 8438.018913] hello_param: myintArray[1] = 2
[ 8438.018915] hello_param: myintArray[2] = 3
[ 8438.018916] hello_param: myintArray[3] = 4
[ 8438.018918] hello_param: myintArray[4] = 5
[ 8438.018919] hello_param: got 5 arguments for myintArray.
[ 8438.018920] hello_param: Goodbye, world.
```

已经够清楚了，俺不整了。

## `module_param_cb`
`module_param_cb`的`ops`需要传入一个`struct kernel_param_ops`，`arg`是一个`struct kernel_param*`
```c
struct kernel_param_ops {
    int (*set)(const char *val, const struct kernel_param *kp);
    int (*get)(char *buffer, const struct kernel_param *kp);
    void (*free)(void *arg);
};
```

此部分正在施工…
