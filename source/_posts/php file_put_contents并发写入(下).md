---
title: 记一次 php 并发写入 顺序混乱的问题排查(下)
date: 2021-04-25 20:37:06
tags: 
  - PHP
  - Linux
  - IO
categories: PHP
---


上次我们说到  PHP 写入文件超过 8k 就不是原子操作而是会分批写入，我们用 C 做了试验，然而我意识到 其实里面有个错误， fputs 并不是一个系统调用，它是由 glibc C 实现的库函数，我们必须用 linux 提供给 C 的 write 函数 来测试。

我们 修改 write.c 文件 使用 write 函数

file: write

```c
#include<stdio.h>
#include <fcntl.h>
void main()
{
    int sz;
    int fd = open("out.log",O_WRONLY | O_CREAT | O_APPEND);
    char buff[30000];
    int i;

    for(i=0;i<sizeof(buff);i+=5){
        strcat(buff, "abcd\n");
    }

    sz = write(fd, buff, sizeof(buff));

    printf("system call write(%d, \"buff \\n\", %d) return %d\n", fd, strlen(buff), sz);
    close(fd);
}
```



编译并执行

```bash
cc write.c -o write
strace ./write
```

我们看到，这次经过了一次 系统调用

```bash
...
open("out.log", O_WRONLY|O_CREAT|O_APPEND, 03777767645557370) = 3
write(3, "abcd\nabcd\nabcd\nabcd\nabcd\nabcd\nab"..., 30000) = 30000
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 3), ...}) = 0
...
```

所以 之前 fputs 应该是C 库函数为写入速度做的优化。

我们再次看 write 的定义

man 2 write

```
BUGS         top
       According to POSIX.1-2008/SUSv4 Section XSI 2.9.7 ("Thread
       Interactions with Regular File Operations"):

           All of the following functions shall be atomic with respect
           to each other in the effects specified in POSIX.1-2008 when
           they operate on regular files or symbolic links: ...

       Among the APIs subsequently listed are write() and writev(2).
       And among the effects that should be atomic across threads (and
       processes) are updates of the file offset.  However, on Linux
       before version 3.14, this was not the case: if two processes that
       share an open file description (see open(2)) perform a write()
       (or writev(2)) at the same time, then the I/O operations were not
       atomic with respect updating the file offset, with the result
       that the blocks of data output by the two processes might
       (incorrectly) overlap.  This problem was fixed in Linux 3.14.   <---- 看这一行
```

也就是说 linux 内核 3.14 之前 其实不是原子写入是存在bug，而之前测试的环境内核是 3.10.0

```bash
[root@liubl ~]# uname -a
Linux liubl 3.10.0-957.5.1.el7.x86_64 #1 SMP Fri Feb 1 14:54:57 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
```

通过查找 PHP 为什么做 超过 chunk_size 的优化 ，看到 php commit  中 php8 中已经取消了这个8192 分批写入的限制

https://github.com/php/php-src/commit/5cbe5a538c92d7d515b0270625e2f705a1c02b18

```
Don't use chunking for stream writes

We're currently splitting up large writes into 8K size chunks, which
adversely affects I/O performance in some cases. Splitting up writes
doesn't make a lot of sense, as we already must have a backing buffer,
so there is no memory/performance tradeoff to be made here.

This change disables the write chunking at the stream layer, but
retains the current retry loop for partial writes. In particular
network writes will typically only write part of the data for large
writes, so we need to keep the retry loop to preserve backwards
compatibility.

If issues due to this change turn up, chunking should be reintroduced
at lower levels where it is needed to avoid issues for specific streams,
rather than unnecessarily enforcing it for all streams.
```

![image-20210427224212749](https://cdn.jsdelivr.net/gh/liubl1024/blog_images/image/blog/image-20210427224212749.png)

因此我们可以安装 一个 php8  系统  内核 4.4  来测试

```
root@ubuntu:~/test# uname -r
4.4.0-62-generic
root@ubuntu:~/test# php -v
PHP 8.0.3 (cli) (built: Mar  5 2021 07:53:39) ( NTS )
Copyright (c) The PHP Group
Zend Engine v4.0.3, Copyright (c) Zend Technologies
    with Zend OPcache v8.0.3, Copyright (c), by Zend Technologies
```



可以看到 写入也是 只进行了一次系统调用

file: chunk.php

```php
<?php
$data = [
    "type"=>"report",
    "time"=>time(),
    "data"=>[],
];

for($i=0;$i<2000;$i++){
    $data["data"][] = ["abc".$i=>rand(100,999)];
}

$content = json_encode($data).PHP_EOL;
echo strlen($content).PHP_EOL;

$out = "out.log";
file_put_contents($out, $content, FILE_APPEND);
```



```
open("/root/test/out.log", O_WRONLY|O_CREAT|O_APPEND, 0666) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=314800000, ...}) = 0
lseek(3, 0, SEEK_CUR)                   = 0
lseek(3, 0, SEEK_CUR)                   = 0
write(3, "{\"type\":\"report\",\"time\":16192384"..., 30935) = 30935
close(3)                                = 0
close(2)                                = 0
close(1)
```



#### 总结

PHP 8 之前的版本 因为 file_put_contents 实现的原因，有  chunk_size 限制 所以 每次写入超过 8192，如果不加锁就存在混乱的可能，

而系统 linux  3.14 内核之前也有可能不是原子的写入过长数据也有可能混乱。