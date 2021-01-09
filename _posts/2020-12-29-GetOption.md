getopt() 函数位于 unistd.h 系统头文件中，函数原型是： 

```
int getopt( int argc, char *const argv[], const char *optstring );
```

getopt使用main函数的argc和argv作为前两个参数，optsting是一个字符列表，每个字符代表一个 **单字符**选项，如果一个字符后面紧跟以冒号（：），表示该字符有一个关联值作为下一个参数；两个冒号"::"代表这个选项的参数是可选的。 getopt的返回值是argv数组中的下一个选项参数， 由optind记录argv数组的下标,如果选项参数处理完毕，函数返回-1； 如果遇到一个无法识别的选项，返回问号（？），并保存在optopt中；

如果一个选项需要一个关联值，而程序执行时没有提供，返回一个问号（？）,如果将optstring的第一个字符设为冒号（:),这种情况下，函数会返回冒号而不是问号。

选项参数处理完毕后，optind会指向argv数组尾部的其他非选项参数。 实际上，**getopt在执行过程中会重排argv数组，将非选项参数移到数组的尾部 。**

getopt() 所设置的全局变量（在unistd.h中）包括：

optarg——指向当前选项参数（如果有）的指针。

optind—— getopt() 即将处理的下一个参数 argv 指针的索引。

optopt——最后一个已知选项。

短选项‘+’表示在遇到非选项参数的时候就停止向后解析了，这样就不会重排argv数组了


https://www.freebsd.org/cgi/man.cgi?getopt_long(3)