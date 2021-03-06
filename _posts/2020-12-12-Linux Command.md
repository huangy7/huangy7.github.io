---
layout:     post
title:      Linux 命令
subtitle:   Linux 命令
date:       2020-12-12
author:     HY
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - Linux
---

# ls
用于显示指定工作目录下之内容（列出目前工作目录所含之文件及子目录)
```
ls -lht
#-l 列出更详细资料
#-h human即更人性化显示，主要看文件大小
#-t 按时间顺序排列
#-a 显示隐藏文件
```
# cd
用于切换当前工作目录
```bash
cd ~ #跳到用户目录
cd .. #跳到上一级目录
cd /usr/local/bin #跳到指定目录
```

# cat
用于查看文件
```bash
#查看文件
[root@huangy helloworld]# cat helloworld.c 
#include <stdio.h>

int main(int argc, char** argv)
{
	printf("HelloWorld");
	return 0;
}
#可以通过管道对输出的文件进一步处理
#输出grep匹配HelloWorld的行
# = grep HelloWorld helloworld.c
[root@huangy helloworld]# cat helloworld.c | grep HelloWorld
	printf("HelloWorld");

#awk以"为分隔符，输出第二个元素
[root@huangy helloworld]# cat helloworld.c | grep HelloWorld  | awk -F '"' '{print $2}'
HelloWorld
```

# more
类似于cat，它可以一页一页读，空格键翻页。q键退出

# grep
匹配关键词，可以和上面cat结合管道使用，也可以单独使用
```bash
#输出匹配HelloWorld的行
[root@huangy helloworld]# grep HelloWorld helloworld.c 
	printf("HelloWorld");
#同时输出匹配行上下两行
[root@huangy helloworld]# grep -n2 HelloWorld helloworld.c 
3-int main(int argc, char** argv)
4-{
5:	printf("HelloWorld");
6-	return 0;
7-}
#输出匹配行上面两行就 -B2  (即Before)
#输出匹配行下面两行就 -A2  (即After)
# -i 忽略大小写
# -w 全词匹配
# -v 反向匹配

# 也可以用来搜索匹配到关键词的文件
[root@huangy ~]# grep -r HelloWorld /root/
/root/helloworld/helloworld.c:	printf("HelloWorld");
```

# mkdir
用于创建文件夹
```bash
#在工作目录下的 runoob2 目录中，建立一个名为 test 的子目录。
#若 runoob2 目录原本不存在，则建立一个。（注：本例若不加 -p 参数，且原本 runoob2 目录不存在，则产生错误。）
mkdir -p runoob2/test
```

# [vi 或者 vim](https://www.runoob.com/linux/linux-vim.html)
文本编辑器，vi就够用了，有vim可以用vim，没有也可以装个
```
#打开文件
vim helloworld.c 
# 此时进入命令模式，按i进入输入模式
# 在输入模式按esc回到命令模式
# 在命令模式按:可输入命令
# :/Hello  查找关键字Hello所在位置
# :wq  保存并退出
```
# rm 
删除文件或文件夹
```bash
rm test.txt # 删除前会询问是否要删除
rm -f test.txt # 直接强制删除，不询问
rm -rf /root/test # 强制删除整个目录
```

# mv
移动文件或文件夹
```bash
# 将b.txt移动到helloworld目录里
mv b.txt /root/helloworld/
# 将 /usr/runoob 下的所有文件和目录移到当前目录下
$ mv /usr/runoob/*  .
```

# cp
复制文件或文件夹
```bash
# 将b.txt复制到helloworld目录里
cp b.txt /root/helloworld/
# 将 /usr/runoob 下的所有文件和目录复制当前目录下
$ cp -rf /usr/runoob/*  .
```

# find
查找文件
```bash
# 在当前目录下查找helloworld.c文件目录
[root@huangy ~]# find . -name helloworld.c
./helloworld/helloworld.c
# 也可以模糊查找，*表示
[root@huangy ~]# find . -name *lo*.c
./helloworld/helloworld.c
```

# chattr
chattr +i 锁定关键文件

chattr -i 解锁

```
[root@localhost ~]# chattr +i /etc/passwd /etc/shadow /etc/group /etc/gshadow /etc/inittab
[root@localhost ~]# rm -f /etc/passwd
rm: cannot remove '/etc/passwd': Operation not permitted
[root@localhost ~]# useradd test1
useradd: cannot open /etc/passwd
```

# tar 
用于打包解包，压缩解压

```bash
#打包压缩
tar -zcvf /tmp/etc.tar.gz /etc 
tar -jcvf /tmp/etc.tar.bz2 /etc
tar -zcf /tmp/strongswan_backup.tar.gz -C /usr/local strongswan
#解压解包
tar -zxf etc.tar.gz
tar -zxf /tmp/strongswan_backup.tar.gz -C /usr/local
#-z gzip
#-j bzip2
#-c create一个压缩包
#-x 解开一个压缩包
#-f 使用档名
#-v 打印详细过程
```


# scp
secure copy 用于 Linux 之间复制文件和目录(带加密)。

1. 将本地文件复制到目标机器
scp -P 22(指定端口，默认22可不指定) 文件名 用户名@目标机器IP:目标机器路径       回车后输入密码
```
scp -P 22 ./test.php zyl@192.168.10.11:/tmp/
```

2. 将目标机器的文件复制到本地
scp 用户名@目标机器IP:目标机器文件名 本地路径       回车后输入密码
```
scp zyl@192.168.10.11:/home/zyl/temp/test.php   ./temp
```

# [netstat](https://linux.cn/article-2434-1.html)
用于列出系统上所有的网络套接字连接情况
```
netstat -nlupta | grep 8899
#-a 列出所有当前的连接
#-n 禁用反向域名解析，加快查询速度
#-l 只列出监听中的连接 listen
#-p 查看进程信息
#-u udp
#-t tcp
```
# curl
curl用于模拟HTTP请求
```
curl http://www.baidu.com
```

# netcat
用于发送udp测试数据
```bash
echo "Hello World\!" | nc -4u 192.168.31.65 2605
echo "Hello World\!" | nc -p 9527 -4u 192.168.31.65 9527
```
