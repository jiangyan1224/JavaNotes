### SHELL命令使用（后续不断更新）：

http://c.biancheng.net/view/1005.html

##### 1.grep：可用于在指定某些文件中查找含有某字符串(正则表达式也可)的行

grep -i -n "test" \*file*

文件名中含有file的文件中找到内容含有test字符串的行，输出行号以及对应行 -i忽略"test"字母大小区别

\*file*处也可以换成指定目录

```shell
jy@ubuntu:~/Documents$ ll
total 16
drwxr-xr-x  2 jy jy 4096 Feb 28 04:16 ./
drwxr-xr-x 16 jy jy 4096 Feb 28 04:11 ../
-rw-rw-r--  1 jy jy   31 Feb 28 04:15 testfile1
-rw-rw-r--  1 jy jy   31 Feb 28 04:16 testfile2
# |管道，把左边命令的输出作为输入给右边的命令
jy@ubuntu:~/Documents$ ll | grep -n "test"
4:-rw-rw-r--  1 jy jy   31 Feb 28 04:15 testfile1
5:-rw-rw-r--  1 jy jy   31 Feb 28 04:16 testfile2
```



##### 2.chmod：设置文件权限

```shell
jy@ubuntu:~/Documents$ ll
total 16
drwxr-xr-x  2 jy jy 4096 Feb 28 04:16 ./
drwxr-xr-x 16 jy jy 4096 Feb 28 04:11 ../
-rw-rw-r--  1 jy jy   31 Feb 28 04:15 testfile1
-rw-rw-r--  1 jy jy   31 Feb 28 04:16 testfile2
#文件权限三位一组，一共三位，分别代表u-user g-group o-other对文件的权限
#- rw- rw- r-- 全满应该是rwx，这里u g只有rw没有x权限
#将文件 file1.txt 与 file2.txt 设为该文件拥有者，与其所属同一个群体者可写入，但其他以外的人则不可写入 :
chmod ug+w,o-w file1.txt file2.txt
```



##### 3.\> 和 \>>：输出重定向

ll>a.txt 将ll执行的结果写到a.txt文件中，没有a.txt会自动创建（覆盖写）

ll>>a.txt 将ll执行的结果写到a.txt文件中，没有a.txt会自动创建（追加写）
cat 文件1>文件2 将文件1的内容覆盖到文件2 通常也可以用来复制文件
echo “内容”>>文件 将内容追加到文件中（如果是>代表覆盖写入）



##### 4.shell中定义变量：A=1 

等号两侧不能有空格，且变量默认都是字符串类型，不能直接数值运算（C=1+1，值就是"1+1"）

变量的值如果有空格，需要用双引号/单引号括起来（D="hello world shell"）



##### 5.shell运算符

$((运算式))  或  $[运算式]  或  expr + - \\* / %运算式（expr运算符间要有空格）

```shell
jy@ubuntu:~$ expr 6 - 7
-1
#esc键下的撇号，作用类似于括号，优先计算
jy@ubuntu:~$ expr 4 \* `expr 6 - 7`
-4
```

```shell
jy@ubuntu:~$ s=$[4*(6-7)]
jy@ubuntu:~$ echo $s
-4
```



##### 6.shell条件判断

[ condition ] condition前后有空格，条件只要非空即为true(这句存疑)，[]返回false

```shell
jy@ubuntu:~$ [ 23 -gt 22 ]
jy@ubuntu:~$ echo $?
0
jy@ubuntu:~$ [ 23 -lt 22 ]
jy@ubuntu:~$ echo $?
1
```

```shell
jy@ubuntu:~/Documents$ [ -e testfile1 ]
jy@ubuntu:~/Documents$ s=$?
jy@ubuntu:~/Documents$ echo $s
0
```

```shell
jy@ubuntu:~/Documents$ [ -e testfile1 ] && [ -d testfile1 ] && echo "success"
jy@ubuntu:~/Documents$ echo $?
1
jy@ubuntu:~/Documents$ [ -e testfile1 ] && [ -r testfile1 ] && echo "success"
success
jy@ubuntu:~/Documents$ echo $?
0
```



##### 7.shell流程控制

###### 7.1 if控制

if  [ condition ];then

   //do sth

fi

或者

if  [ condition ]

   then

   		//do sth

fi

（if 后要有空格）

```shell
#!/bin/bash
if [ $1 -eq 1 ];then
   echo "input is 1"
elif [ $1 -eq 2 ];then
   echo "input is 2"
fi

jy@ubuntu:~/Documents$ ./ifTest.sh 1
input is 1
```

###### 7.2 case语句

case $变量名 in
"值1")

如果变量的值等于值1，则执行程序1

;;
“值2")
如果变量的值等于值2，则执行程序2

;;

…省略其他分支…
*)
如果变量的值都不是以上的值，则执行此程序
;;
esac

###### 7.3 for循环

for(( 初始值;循环控制条件;变量变化 ))

​      do

​			//do sth

​	  done

或者

for 变量 in 值1 值2 值3…
do
程序
done

```shell
#!/bin/sh
s=0
for((i=0;i<=100;i++))
   do
        s=$[$s+$i]
   done
echo $s

jy@ubuntu:~/Documents$ bash ./forTest.sh 
5050
```

```shell
#!/bin/bash

for i in $*
do
        echo "input use \$*: $i"
done
for j in $@
do
        echo "input use \$@: $j"
done
for i in "$*"
do
        echo "input use \"\$*\": $i"
done
for j in "$@"
do
        echo "input use \"\$@\": $j"
done


jy@ubuntu:~/Documents$ bash ./forTest2.sh 1 2
input use $*: 1
input use $*: 2
input use $@: 1
input use $@: 2
input use "$*": 1 2
input use "$@": 1
input use "$@": 2
```

###### 7.4while循环

##### 8 shell工具

###### 8.1cut

http://c.biancheng.net/view/989.html

grep用于提取文件中符合条件的行；而cut用于提取符合条件的列

cut [选项] 文件名

- -f 列号：提取第几列；
- -d 分隔符：按照指定分隔符分割列；默认制表符
- -c 字符范围：不依赖分隔符来区分列，而是通过字符范围（行首为 0）来进行字段提取。"n-"表示从第 n 个字符到行尾；"n-m"表示从第 n 个字符到第 m 个字符；"-m"表示从第 1 个字符到第 m 个字符；

```shell
#以:为分隔符，提取/etc/passwd的第1 3列
jy@ubuntu:~/Documents$ cut -d ":" -f 1,3 /etc/passwd
root:0
daemon:1
bin:2
sys:3
sync:4
games:5
man:6
lp:
...
```

但是有些命令输出的分隔符，不是制表符而是多个空格符， cut 命令会忠实地将每个空格符当作一个分隔符，而这样数，输出的列有可能并不是想要的列（比如df、ifconfig的输出）

```shell
#获取PATH值中，第二个":"开始后的所有路径
jy@ubuntu:~/Documents$ echo $PATH | cut -d ":" -f 3-
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

切割ifconfig打印内容中的ip地址：

```shell
jy@ubuntu:~/Documents$ ifconfig
ens33     Link encap:Ethernet  HWaddr 00:0c:29:45:b7:07  
          inet addr:192.168.121.141  Bcast:192.168.121.255  Mask:255.255.255.0
          inet6 addr: fe80::6c30:ad5e:e3da:5ceb/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:339300 errors:0 dropped:0 overruns:0 frame:0
          TX packets:138479 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:456777161 (456.7 MB)  TX bytes:8394347 (8.3 MB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:350 errors:0 dropped:0 overruns:0 frame:0
          TX packets:350 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:29741 (29.7 KB)  TX bytes:29741 (29.7 KB)

jy@ubuntu:~/Documents$ ifconfig ens33 | grep "inet addr" | cut -d ":" -f 2 | cut -d " " -f 1
192.168.121.141
```

###### 8.2sed

http://c.biancheng.net/view/994.html

一种流编辑器，一次处理一行内容。处理时，把当前处理的行存储在临时缓冲区，在缓冲区中处理完毕之后，把缓冲区的内容送往屏幕，接着处理下一行，指导文件末尾。文件内容本身没有改变。（使用-i选项可以改变）

sed [选项] '[动作]' 文件名（这里是普通单引号）

```shell
studentTest.txt:
1ID Name PHP Linux MySQL Average
2 Liming 82 95 86 87.66
3 Sc 74 96 87 85.66
4 Gao 99 83 93 91.66

#在第2行前插入hello
jy@ubuntu:~/Documents$ sed -e '2i hello' -e '1d' studentTest.txt 
hello
2 Liming 82 95 86 87.66
3 Sc 74 96 87 85.66
4 Gao 99 83 93 91.66
#g是global的意思，在第2行以及后面所有行的前面都插入hello
jy@ubuntu:~/Documents$ sed -e '2,$i hello' -e 's/ID/id/g' studentTest.txt 
1id Name PHP Linux MySQL Average
hello
2 Liming 82 95 86 87.66
hello
3 Sc 74 96 87 85.66
hello
4 Gao 99 83 93 91.66
#不加g：
jy@ubuntu:~/Documents$ sed '3s/Sc/id/' studentTest.txt 
1ID Name PHP Linux MySQL Average
2 Liming 82 95 86 87.66
3 id 74 96 87 85.66
4 Gao 99 83 93 91.66
5 Sc 74 96 87 85.66

jy@ubuntu:~/Documents$ sed -e '2d' studentTest.txt 
1ID Name PHP Linux MySQL Average
3 Sc 74 96 87 85.66
4 Gao 99 83 93 91.66
```

###### 8.3awk

一个文本分析工具，逐行读入文件，以空格为默认分隔符对每行切片，切开的部分再分析处理。

 awk '条件1 {动作 1}   条件 2 {动作 2} …' 文件名

先不看条件：

```shell
#没有设定条件，所以文件中的所有行都符合，都进行处理
jy@ubuntu:~/Documents$ awk '{printf $2 "\t" $6 "\r\n"}' studentTest.txt 
Name	Average
Liming	87.66
Sc	85.66
Gao	91.66
Sc	85.66

#awk&cut对空格分隔符处理（尤其是像df这样不输出标准\t而是多个空格符的：）
jy@ubuntu:~/Documents$ df -h 
Filesystem      Size  Used Avail Use% Mounted on
udev            956M     0  956M   0% /dev
tmpfs           197M  8.7M  189M   5% /run
/dev/sda1        19G  5.1G   13G  29% /
tmpfs           985M  252K  985M   1% /dev/shm
tmpfs           5.0M  4.0K  5.0M   1% /run/lock
tmpfs           985M     0  985M   0% /sys/fs/cgroup
tmpfs           197M   72K  197M   1% /run/user/1000

jy@ubuntu:~/Documents$ df -h | cut -d " " -f 1,3
Filesystem 
udev 
tmpfs 
/dev/sda1 
tmpfs 
tmpfs 
tmpfs 
tmpfs 

jy@ubuntu:~/Documents$ df -h | awk '{print $1 "\t" $3 }'
Filesystem	Used
udev	0
tmpfs	8.7M
/dev/sda1	5.1G
tmpfs	252K
tmpfs	4.0K
tmpfs	0
tmpfs	72K

```

awk的条件：

BEGIN & END：

BEGIN的执行时机是在awk开始读取文件数据之前，且仅在开始读取文件之前执行一次，后续不再执行；END和BEGIN类似，区别在于一个是在开始读取数据之前，一个是在读取数据结束之后。

```shell
jy@ubuntu:~/Documents$ awk 'BEGIN{print "read file begin"} {print $2 "\t" $6} END{print "read file end"}' studentTest.txt 
read file begin
Name	Average
Liming	87.66
Sc	85.66
Gao	91.66
Sc	85.66
read file end

```

关系运算符：还是用studentTest.txt：

```shell
#找出所有不含Name的行，且第6列数据>=87的，输出对应行的第2列
jy@ubuntu:~/Documents$ cat studentTest.txt | grep -v "Name" | awk '$6>=87{print $2}'
Liming
Gao

#查询包含"sda数字"的行，并打印第一个字段和第五个字段
[root@localhost ~]# df -h | awk '/sda[0-9]/ {printf $1 '\t\ $5 "\n"}'
/dev/sda3 10%
/dev/sda1 15%
```

```shell
#打印出/etc/passwd中以root开头的第 1 7列（以:分割）
jy@ubuntu:~/Documents$ cat passwd | awk -F ":" '/^root/{print $1 " " $7}'
root /bin/bash
#将passwd文件中以:分割的第3列id增加数值1并输出
#-v用于定义变量
jy@ubuntu:~/Documents$ cat passwd | awk -F ":" -v i=1 '{print $3+$i}'
0
1
2
3
4
5
。。。
```

 awk '条件1 {动作 1}   条件 2 {动作 2} …' 文件名，对于每一行，都会对多个条件挨个判断，对于某个条件只要成立，就执行对应动作，不成立就不执行对应动作。

awk还有内置变量：FILENAME NR（已读/当前行数） NF（切割后列的个数）

awk一定是一行一行读取文件，对每一行，以某个分隔符分割为多列，然后根据条件筛选处理

```shell
#awk获取ip地址
jy@ubuntu:~/Documents$ ifconfig ens33| awk '/inet addr/{print $2}' | awk -F ":" '{print $2}'
192.168.121.142
#awk获取文件中空行行号
#^后的内容表示以该内容开头；$前面的内容表示以该内容结尾，所以这里表示空行
jy@ubuntu:~/Documents$ cat testfile1 | awk '/^$/{print NR}'
3
```

###### 8.4sort

Linux的排序命令，将文件的每一行作为一个单位，相互比较。默认情况下，是从首字符开始，一次按照ASCII码比较，升序输出。（类似于Java对字符串的默认比较）

sort [选项] 文件名

-r 反向排序

-n 以数值型进行排序

-t 指定分隔符，默认为\t

-k [n,m] 按照指定的字段范围进行排序。从第n个字段开始，到第m个字段结束（默认是到行尾）

如果k没有任何指定，则指从每行的第一个字段到行尾，进行排序

```shell
jy@ubuntu:~/Documents$ head -n 10 passwd | sort -t ":" -n -k 3,3
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin

jy@ubuntu:~/Documents$ head -n 10 passwd | sort
bin:x:2:2:bin:/bin:/usr/sbin/nologin
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
games:x:5:60:games:/usr/games:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
root:x:0:0:root:/root:/bin/bash
sync:x:4:65534:sync:/bin:/bin/sync
sys:x:3:3:sys:/dev:/usr/sbin/nologin
```

```shell
#对文件排序，并输出第一列数字值的和
jy@ubuntu:~/Documents$ sort -n testNumSort | awk '{a+=$0;print $0} END{print a}' 
1
6
7
8
9
10
23
24
67
88
90
333 #a

#找到当前目录下所有文本文件内容中含有字符串“test”的文件名
jy@ubuntu:~/Documents$ grep "test" *.txt | awk -F ":" '{print $1}'
student.txt
#找到/home/目录以及其子目录下，所有内容含有字符串"test"的文本文件的文件名
jy@ubuntu:~/Documents$ grep -r "test" /home/ 
/home/jy/.bashrc:    test -r ~/.dircolors && eval "$(dircolors -b ~/.dircolors)" || eval "$(dircolors -b)"
/home/jy/.bash_history:grep test test*
#。。。。。全是bash_history
/home/jy/Documents/testfile1:test
/home/jy/Documents/student.txt:test
#取出结果后，对第一列正则匹配“.txt”
jy@ubuntu:~/Documents$ grep -r "test" /home/ | awk -F ":" '$1 ~ /.txt/{print$1}'
/home/jy/Documents/student.txt
```

