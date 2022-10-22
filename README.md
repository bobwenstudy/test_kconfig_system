# 从零到一搭建Kconfig配置系统



## 背景说明

最早接触到`Kconfig`是在`zephyr`项目中，之后陆续知道`linux`和`RT-Thread`等项目都是用`Kconfig`来管理编译的，而自己也陆续有大型项目开发需要，了解过后对其使用愈发感兴趣起来。

在实际项目开发中，通常会有需要去使能/关闭一些代码模块或者修改一些配置参数。

项目代码都放在GitHub上，[bobwenstudy/test_kconfig_system (github.com)](https://github.com/bobwenstudy/test_kconfig_system)

### 代码管理

使能或者关闭代码，可以通过`#define+#ifdef`就可以实现这一目的。

```c
#define CONFIG_TEST_ENABLE

#ifdef CONFIG_TEST_ENABLE
	... ... ...
#endif
```

调整配置参数，直接通过`#define`就可以实现这一目的。

```c
#define CONFIG_TEST_SHOW_STRING "Test 123"
#define CONFIG_TEST_SHOW_INT (123)
```

但是通过这个方式有个问题，就是不直观，如果就几个参数还好，但是当参数越来越多，并之前存在先后关系时，管理难度会呈指数上升。

如下面的例子，`CONFIG_TEST_SUB_1_ENABLE`的开启前提是`CONFIG_TEST_TOP_ENABLE`开启，当`CONFIG_TEST_SUB_0_ENABLE`和`CONFIG_TEST_SUB_1_ENABLE`都开启的情况下`CONFIG_TEST_SHOW_INT = 123`，否则`CONFIG_TEST_SHOW_INT = 456`。

这些宏之间的关系都用代码来描述，需要开发人员熟悉所有代码行为，才能很好的配置这些功能。

```c
#define CONFIG_TEST_TOP_ENABLE
#define CONFIG_TEST_SUB_0_ENABLE

#ifdef CONFIG_TEST_TOP_ENABLE
#define CONFIG_TEST_SUB_1_ENABLE
#endif

#if defined(CONFIG_TEST_SUB_0_ENABLE) && defined(CONFIG_TEST_SUB_1_ENABLE)
#define CONFIG_TEST_SHOW_SUB_INT (123)
#else
#define CONFIG_TEST_SHOW_SUB_INT (456)
#endif
```



### 预编译管理

除了在代码中配置外，也可以通过-D预编译来管理，然后编译全局都有这些配置参数了，对应代码中的宏定义。

```c
gcc xxx -DCONFIG_TEST_ENABLE -DCONFIG_TEST_SHOW_INIT=123

#define CONFIG_TEST_ENABLE
#define CONFIG_TEST_SHOW_INIT (123)
```



### 问题

如果是小型项目通过上述两种方式管理都还好，当项目越来越大时，所需配置的参数越来越多时，希望有一个工具专门来管理这些配置参数，并且能够可视化这些编译配置。

而Kconfig就是这样一个通用的工具来解决这一问题。



## 环境搭建

`Kconfig`是一个配置描述文件，而其配置界面程序需要通过GUI来完成，不然看到的只是一些文本文件，并且这个文件并不能被c所识别。

这里推荐用python的[kconfiglib · PyPI](https://pypi.org/project/kconfiglib/)库来使用。

### Kconfiglib安装

直接python安装即可。由于要使用menuconfig，在windows环境下还需要安装`windows-curses`。

```python
python -m pip install windows-curses
python -m pip install kconfiglib
```



安装完成后，会在python路径下生成`menuconfig.exe`执行程序。输入`menuconfig -h`，出现下面的信息说明已经安装好了。

![image-20221021154103835](https://markdown-1306347444.cos.ap-shanghai.myqcloud.com/img/image-20221021154103835.png)



### c头文件生成脚本-kconfig.py

使用`Kconfiglib`生成的是`.config`文件，而c代码要使用，必须要提供头文件。其实生成功能主要还是依靠`kconfiglib`中提供的`class Kconfig`中的`write_autoconf`方法，但是也需要一个脚本来调用这个库。

源码所在路径为：[Kconfiglib/kconfiglib.py at master · ulfalizer/Kconfiglib (github.com)](https://github.com/ulfalizer/Kconfiglib/blob/master/kconfiglib.py)

![image-20221021174305501](https://markdown-1306347444.cos.ap-shanghai.myqcloud.com/img/image-20221021174305501.png)

[kconfig 实例1: 基于 python 的 kconfig 系统 - 简书 (jianshu.com)](https://www.jianshu.com/p/25237ab0bf66)这个里面提供了一个简单的实现版本，但是有点太简单了，不符合我们实际项目的需要。

![image-20221021173111119](https://markdown-1306347444.cos.ap-shanghai.myqcloud.com/img/image-20221021173111119.png)

这里还是推荐用zephyr项目的版本：[zephyr/kconfig.py at main · zephyrproject-rtos/zephyr (github.com)](https://github.com/zephyrproject-rtos/zephyr/blob/main/scripts/kconfig/kconfig.py)。这个使用需要输入如下参数：

- kconfig_file，顶层的Kconfig配置文件路径
- config_out，输出.config文件路径
- header_out，输出c头文件路径
- kconfig_list_out，日志文件
- configs_in，1个或多个输入.config文件

![image-20221021173232502](https://markdown-1306347444.cos.ap-shanghai.myqcloud.com/img/image-20221021173232502.png)



### GCC安装

这里调试用的是windows环境，直接自行下载msys2+mingw即可。

https://blog.csdn.net/qq_31985307/article/details/114235846







## Kconfig基本概念&语法

### Kconfig简介

`Kconfig`语言定义了一套完整的规则来表述配置项及配置项间的关系。

之前安装的`Kconfiglib`只是用来显示Kconfig的，实际工程中通常会包含`Kconfig`和`.config`文件。

- **Kconfig**，是配置项的描述文件，支持设置配置项及其默认值，依赖关系等等，该文件还会继续依赖各个模块的**Kconfig**文件。
- **.config**，产品配置文件，提供配置项及在产品中这些配置项的设置值，可能和**Kconfig**配置项的默认取值不一致，属于产品对配置项的定制。这些配置文件在可以在**makefile**文件中使用。
- **autoconfig.h**，生成的C语言头文件，提供配置项的宏定义版，在C语言程序中使用。

### Kconfig语法

由于本文重点是讲Kconfig环境的搭建，所以语法就不展开。

linux的原版可以看这个：[Kconfig Language — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/kbuild/kconfig-language.html#introduction)

zephyr项目的介绍：[Configuration System (Kconfig) — Zephyr Project Documentation](https://docs.zephyrproject.org/latest/build/kconfig/index.html)

中文的一些说明：

[Kconfig 语法 - fluidog - 博客园 (cnblogs.com)](https://www.cnblogs.com/fluidog/p/15176680.html)

[kconfig语法整理 - 简书 (jianshu.com)](https://www.jianshu.com/p/aba588d380c2)



## Kconfig实战

这里围绕实际使用的几个场景来进行说明

### 直接配置版本

#### 准备

需要提供main.c，Makefile，Kconfig。下面分别进行描述：

![image-20221021175825950](https://markdown-1306347444.cos.ap-shanghai.myqcloud.com/img/image-20221021175825950.png)

##### Kconfig

这里我们将最开始背景说明中的宏定义配置改成Kconfig写法。

```python
mainmenu "Kconfig Demo"

menu "Test Params setting"
config TEST_ENABLE
    bool "Enable test work"
    default n
    help
        Will print debug information if enable.

config TEST_SHOW_STRING
    string "The show string info"
    default "Test 123"

config TEST_SHOW_INT
    int "The show int info"
	range 0 255
    default 123


config TEST_TOP_ENABLE
	bool "Test Top Func"
    default n
    help
        Function Test Top

config TEST_SUB_0_ENABLE
	bool "Test Sub 0 Func"
    default n
    help
        Function Test Sub 0

config TEST_SUB_1_ENABLE
	bool "Test Sub 1 Func"
    default n
    depends on TEST_TOP_ENABLE
    help
        Function Test Sub 1

config TEST_SHOW_SUB_INT
    int
    default 456 if TEST_SUB_0_ENABLE && TEST_SUB_1_ENABLE
    default 123


endmenu
```



##### Makefile

相比于传统的makefile，加入了`autoconfig.h`、`.config`和`menuconfig`编译目标。

**menuconfig**，用于显示menuconfig页面，让用户通过GUI选择当前配置参数。

**.config**，如果用户第一次没有`.config`文件时，调用`menuconfig`来配置`Kconfig`并生成`.config`。

**autoconfig.h**，如果`.config`有更新就需要执行，实际是调用`kconfig.py`脚本来生成`autoconfig.h`头文件。

```makefile
all: main.o
	gcc main.o -o main
main.o: main.c autoconfig.h
	gcc main.c -c -o main.o
clean:
	del main.o main.exe

autoconfig.h:.config
	python ../scripts/kconfig.py Kconfig .config autoconfig.h log.txt .config
.config:
	menuconfig
menuconfig:
	menuconfig
```



##### main.c

比较简单，一个是引用生成的autoconfig.h头文件，然后再根据不同的配置打印当前的信息

```c
#include <stdio.h>
#include "autoconfig.h"
int main()
{
    printf("hello, world\n");
#ifdef CONFIG_TEST_ENABLE
    printf("CONFIG_TEST_ENABLE\n");
#endif
	printf("CONFIG_TEST_SHOW_STRING: %s\n", CONFIG_TEST_SHOW_STRING);
	printf("CONFIG_TEST_SHOW_INT: %d\n", CONFIG_TEST_SHOW_INT);
#ifdef CONFIG_TEST_TOP_ENABLE
    printf("CONFIG_TEST_TOP_ENABLE\n");
#endif
#ifdef CONFIG_TEST_SUB_0_ENABLE
    printf("CONFIG_TEST_SUB_0_ENABLE\n");
#endif
#ifdef CONFIG_TEST_SUB_1_ENABLE
    printf("CONFIG_TEST_SUB_1_ENABLE\n");
#endif
	printf("CONFIG_TEST_SHOW_SUB_INT: %d\n", CONFIG_TEST_SHOW_SUB_INT);
    return 0;
}
```



#### 首次编译

直接键入`make all`，由于初始环境没有.config文件，会调用menuconfig进行配置生成.config文件。

如下图配置完成后输入Q，会提示保存.config文件，直接保存即可。

![image-20221021171934216](https://markdown-1306347444.cos.ap-shanghai.myqcloud.com/img/image-20221021171934216.png)



有`.config`文件后，就会执行**python**脚本生成`autoconfig.h`文件，最后调用`gcc`生成`main.exe`。

![image-20221021180323871](https://markdown-1306347444.cos.ap-shanghai.myqcloud.com/img/image-20221021180323871.png)



执行编译之后，生成的`autoconfig.h`文件信息如下。如上图所示，`main.exe`的执行结果和锁配置的`autoconfig.h`参数相同。

```c
#define CONFIG_TEST_ENABLE 1
#define CONFIG_TEST_SHOW_STRING "Test 567"
#define CONFIG_TEST_SHOW_INT 123
#define CONFIG_TEST_TOP_ENABLE 1
#define CONFIG_TEST_SHOW_SUB_INT 123
```



生成的`.config`文件如下，可以看到默认值通过`#`注释了，其他就是我们在`menuconfig`所指定的值。

```python

#
# Test Params setting
#
CONFIG_TEST_ENABLE=y
CONFIG_TEST_SHOW_STRING="Test 567"
CONFIG_TEST_SHOW_INT=123
CONFIG_TEST_TOP_ENABLE=y
# CONFIG_TEST_SUB_0_ENABLE is not set
# CONFIG_TEST_SUB_1_ENABLE is not set
CONFIG_TEST_SHOW_SUB_INT=123
# end of Test Params setting

```



如上图所示的执行完程序之后，生成了很多中间文件，和目标文件。真正有用的是`.config`和`autoconfig.h`文件。

![image-20221021200109412](https://markdown-1306347444.cos.ap-shanghai.myqcloud.com/img/image-20221021200109412.png)



#### 调整参数

第一次编译完毕以后，之后我们想修改参数可以通过输入`make menuconfig`进配置页面，**需要注意的是，这时候打开时，不再是默认值，而是我们之前调整后的值**。

按照如下配置后。

![image-20221021201439203](https://markdown-1306347444.cos.ap-shanghai.myqcloud.com/img/image-20221021201439203.png)



修改后，再键入`make all`，由于`autoconfig.h`所依赖的`.config`变化了，会触发再次运行**python**脚本生成`autoconfig.h`，再生成`main.exe`。

![image-20221021202232194](https://markdown-1306347444.cos.ap-shanghai.myqcloud.com/img/image-20221021202232194.png)



生成的`autoconfig.h`文件如下所示，从上图的执行结果可以看出行为一致。

```c
#define CONFIG_TEST_ENABLE 1
#define CONFIG_TEST_SHOW_STRING "Test 567"
#define CONFIG_TEST_SHOW_INT 78
#define CONFIG_TEST_TOP_ENABLE 1
#define CONFIG_TEST_SUB_0_ENABLE 1
#define CONFIG_TEST_SUB_1_ENABLE 1
#define CONFIG_TEST_SHOW_SUB_INT 456
```

生成的`.config`文件如下所示，和我们GUI配置的参数一致。

```python

#
# Test Params setting
#
CONFIG_TEST_ENABLE=y
CONFIG_TEST_SHOW_STRING="Test 567"
CONFIG_TEST_SHOW_INT=78
CONFIG_TEST_TOP_ENABLE=y
CONFIG_TEST_SUB_0_ENABLE=y
CONFIG_TEST_SUB_1_ENABLE=y
CONFIG_TEST_SHOW_SUB_INT=456
# end of Test Params setting
```



#### 总结

上述的方案算是一个最常用的方式了，发布的时候只提供`Kconfig`，`.config`为默认值生成的，用户需要改的时候自己通过`make menuconfig`来修改，以便生成不同的应用程序。

这个方式已经可以满足绝大数应用场景的需要了。





### Zephyr提出的持久化版本/多应用版本

虽然上述版本已经满足大多数场景需要了，但是对于像Zephyr这种项目，项目非常大，有多个驱动，并且其会提供很多例程，这些例程都是预先配置好参数给客户直接编译下载的。

不同例程之间所使用的参数各不相同，而且还想让客户可以通过GUI调整某个例程的参数信息，这时候如何管理呢。

这对应芯片厂来讲是一个很普遍的需求，芯片厂所提供的SDK一般包含多个例程，不同例程公用驱动库，只是所使用参数不同。

Zephyr原文：[Configuration System (Kconfig) — Zephyr Project Documentation](https://docs.zephyrproject.org/latest/build/kconfig/index.html)

好的Zephyr Kconfig使用思路文档：

- [Zephyr Devicetree 与 Kconfig 配置指南 — PAN1080 DK Documentation (panchip.com)](https://docs.panchip.com/pan1080dk-doc/0.5.0/04_dev_guides/zephyr_configuration_guidance.html#)
- [Zephyr-系统配置(Kconfig)_只想.静静的博客-CSDN博客_zephyr如何配置](https://blog.csdn.net/u011638175/article/details/121339581)

Zephyr项目有点大，他的一些理念可以学习，但本文会给大家准备一个精简版的例程，借鉴其思路来实现类似其工作效果（学习zephyr可以参考下）。



#### 准备

假定有2个应用，每个应用有一个main函数。Makefile通过APP参数来指定不同的编译目标，每个应用有其配置参数。项目的目录结构如下图所示。

需要提供main.c，Makefile，Kconfig。下面分别进行描述：

























