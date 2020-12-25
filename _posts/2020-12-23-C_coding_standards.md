---
layout:     post
title:      C语言编程规范
subtitle:   C语言编程规范
date:       2020-12-23
author:     HY
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - Linux - C
---
# 缩进
"switch"和其下级的"case"放在同一列，而不是双重缩进"case"标签
```C
    switch (suffix) {
    case 'G':
    case 'g':
        mem <<= 30;
        break;
    case 'M':
    case 'm':
        mem <<= 20;
        break;
    case 'K':
    case 'k':
        mem <<= 10;
        /* fall through */
    default:
        break; 
    }
```

不要将多个语句放在同一行
# 换行
行的长度限制为80列，这是强烈推荐的设置。
多于80列的语句将被分为合适的数块。子句应该永远比主句短，且比主句更靠右侧
对有较长参数表的函数声明同样有效
长字符串同样也被打断为短字符串
```
void fun(int a, int b, int c)
{
    if (condition)
        printk(KERN_WARNING "Warning this is a long printk with "
                        "3 parameters a: %u b: %u "
                        "c: %u \n", a, b, c);
    else
        next_statement;
}
```

# 大括号和空格
将左大括号放置在行末尾。而右大括号放置在行首
只有一行语句时不用添加多余的大括号

# 命名
- 全局变量：需要一个描述型名字，偏向命名为 g_users
- 局部变量；简洁了当，如果在循环中需要一些随机数字，你大可以命名其为 "i" 。只要不会产生歧义，命名为 "loop_counter" 毫无意义。同样的，"tmp" 可以是任何类型的临时变量。


# typedefs
不要傻呼呼地使用像"vps_t"这样的变量类型.

使用typedef来重定义已有结构体和指针本身就是个错误. 当你在源代码中见到这样的定义:
```
vps_t a;
```
天知道a到底是个什么东西!

如果你看到这样的定义:
```
struct virtual_container *a; 
```
你完全可以一目然: 哦, a是一个指向...的指针.

多数人都觉得typedefs可以提高可阅读性, 但真理往往掌握在少数人手中. Typedefs只有在如下情况下有用:

1. 需要被封装起来的对象(你本来就打算隐藏起类型信息)
如: "pte_t"这种类型.  封装出这样的类型本来就只打算让特定的"访问函数"才能访问.
请注意: 封装以及"访问函数"本来就不是什么好东西.  The reason we have them for things like pte_t etc. is that there 
really is absolutely _zero_ portably accessible information there.

2. 定长的整数类型. 这样可以在避免在某些情况下, 搞不清楚到底用的是int还是long, 把你自己搞晕!
u8/u16/u32都是完美的typdefs, 尽管更应该它们归结至规则(d)下.
再次重申: 这样定义必须要有合理的理由. 如果某个变量本来就是unsigned long类型, 你硬要它定义成这样,你就是SB了:
```
typedef unsigned long myflags_t; 
```
如果你有明确的理由, 在某种情形下变量是unsigned int类型, 而在另外的情形下又要变身成为unsigned long类型, 那就尽管去typedef.

3. 当你需要使用kernel的sparse工具做变量类型检查时,  你也可以typedef一个类型.

4. 对于特殊情况下的某些c99标准的新类型
你的大脑和眼睛只需要很短的时间就可以习惯像'uint32_t'这样的新类型，虽然有的人反对这宗用法。
因此，虽然对于你的新代码来说，linux独有的'u8/u16/u32/u64'类型并不是强制的，但是他们也是与标准类型等价的。
当编辑现有代码时，如果其中已经使用了某一种类型名规范，你应该遵循原样，使用与之相同的类型名。

5. 用户空间中的类型安全
对于某些结构，显然我们不能使用c99标准的类型，不能使用上述的‘u32’，因此咱干脆在结构中使用'_u32'或者类似的类型好了。

# 函数
1. 局部变量个数：5-10个
2. 简短


# 集中一处退出函数
当函数有很多个出口，使用goto把这些出口集中到一处是很方便的，特别是函数中有许多重复的清理工作的时候。

理由是：

-无条件跳转易于理解

-可以减少嵌套

-可以避免那种忘记更新某一个出口点的问题

-算是帮助编译器做了代码优化
```
int fun(int a) 
{ 
    int result = 0; 
    char *buffer = kmalloc(SIZE); 

    if (buffer == NULL) 
        return -ENOMEM; 

    if (condition1) { 
        while (loop1) { 
            ... 
        } 
        result = 1; 
        goto out; 
    } 
    ... 
out: 
    kfree(buffer); 
    return result; 
} 
```

# 注释
写在函数前面说明函数是干啥的。函数里头不要有注释
linux当中的注释是c89格式（"/* ... */"）的，而不是c99中新近添加的"// ..."

多于一行（多行）的注释应当准从以下格式：
```
/* 
 * This is the preferred style for multi-line 
 * comments in the Linux kernel source code. 
 * Please use it consistently. 
 * 
 * Description:  A column of asterisks on the left side, 
 * with beginning and ending almost-blank lines. 
 */ 
```

该格式对于注释标识符（常量，变量，函数等）同样适用。换句话说，你最好不要再一行里面同时声明很多个标识符（无论是用逗号还是分号隔开都是不推荐的），一行一个就可以了。这样你就可以在每一行对每一个标识符进行解释。

# 宏，枚举和RTL
定义常量的宏名字和枚举变量是大写的
定义一些相关联常量用枚举
全部为大写的宏名称是很好的，但有些和函数相似的宏名称也可以定义为小写的
通常，把内联函数被定义为宏是比较好的
多个语句的宏应该被定义在一个do-while循环体里：
```
#define macrofun(a, b, c)             \
     do {                             \
         if (a == 5)                  \
             do_this(b, c);           \
     } while (0)
```
应该避免的情况：
1. 会改变控制流的宏
```
#define FOO(x)                    \
     do {                         \
         if (blah(x) < 0)         \
             return -EBUGGERED;   \
     } while(0)
```
It is a _very_ bad idea.  It looks like a function call but exits the "calling"
function; don't break the internal parsers of those who will read the code.
它看起来像一个函数调用，但是它可能会直接使得调用者退出。搞得读代码的人会莫名其妙，为啥在这个地方退出了
2. 宏依赖奇怪名称的本地变量
```
#define FOO(val) bar(index, val)
```

# 函数返回值或名称
如果函数是一个动作或者一个命令，则成功返回0，失败返回一个整型的错误代码，例如，“add work”是一个命令， 因此add_work()函数返回0表示成功，或者返回-EBUSY表示失败。
如果函数名类似断言，那么它应该返回一个表示是否成功的布尔值。例如，“PCI device present”是一个断言，那么pci_dev_present()函数返回1表示找到匹配的设备，或者返回0表示没找到。

# 日志调试
不要使用简写，使用 "do not" 或 "don't" 而非 "dont"。保持信息简洁明了。














