---
layout:     post
title:      Linux中__init、__devinit等内核优化宏
subtitle:   Linux中__init、__devinit等内核优化宏
date:       2020-12-23
author:     HY
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - Linux - C
---

# Linux中__init、__devinit等内核优化宏【转】

转自：https://blog.csdn.net/qingkongyeyue/article/details/72935439
https://blog.csdn.net/qingkongyeyue/article/details/72935439

内核使用了大量不同的宏来标记具有不同作用的函数和数据结构。如宏__init 、__devinit 等。这些宏在include/linux/init.h 头文件中定义。编译器通过这些宏可以把代码优化放到合适的内存位置，以减少内存占用和提高内核效率。

下面是一些常用的宏：

·  `__init` ，标记内核启动时使用的初始化代码，内核启动完成后不再需要。以此标记的代码位于.init.text 内存区域。它的宏定义是这样的：

```C
· #define _ _init _ _attribute_ _ ((_ _section_ _ (".text.init")))
```

·  `__exit` ，标记退出代码，对于非模块无效。如果编译稳定的话，exit函数将永远不会被调用。

```C
#ifdef MODULE
#define __exit __attribute__ ((__section__(".exit.text")))
#else
#define __exit __attribute_used__ __attribute__((__section__(".exit.text")))
#endif
```



·  `__initdata`，标记内核启动时使用的初始化数据结构，内核启动完成后不再需要。以此标记的代码位于.init.data 内存区域。

·  `__devinit`，标记设备初始化使用的代码。

·  `__devinitdata`，标记初始化设备数据结构的函数。

·  `__devexit`，标记移除设备时使用的代码。

·  `xxx_initcall`，一系列的初始化代码，按降序优先级排列。

初始化代码的特点是：在系统启动运行，且一旦运行后马上退出内存，不再占用内存。

1. 所有标识为`__init`的函数，在链接的时候，都放在.init.text这个区域中。在这个区域中，函数的摆放顺序是和链接顺序有关的，是不确定的。
2. 所有的`__init`函数在区域.initcall.init中还保存了一份函数指针。在初始化时，内核会通过这些函数指针调 用这些`__init`函数，并在整个初始化完成后，释放整个init区域 (包括.init.text, .initcall.init...)

> 注：这些函数在内核初始化过程中的调用顺序只和这里的函数指针顺序有关，和1中所述的这些函数代码本身在.init.text区域中的顺序无关。

对于驱动程序模块来说，这些优化标记使用的情况如下：

·  通过module_init() 和module_exit() 函数调用的函数就需要使用`__init`和 `__exit`宏来标记。

·  pci_driver 数据结构不需标记。

·  probe() 和remove() 函数应该使用 `__devinit` 和 `__devexit`标记，且只能标记probe() 和remove()

·  如果remove() 使用 `__devexit` 标记，则在pci_driver 结构中要用 `__devexit_p(remove)`来引用remove() 函数。

·  如果你不确定需不需要添加优化宏则不要添加。
