---
title: 记一次 php 并发写入 顺序混乱的问题排查(上)
date: 2021-04-23 08:37:06
tags: 
  - PHP
  - Linux
  - IO
categories: PHP
---

> 记一次 php  file_put_contents 并发写入  顺序混乱的问题排查


### 起因

事情是这样的，本来用 spark 跑采集的客户端日志，结果报错如下:

```bash
WARN [main] DataSource: Found duplicate column(s) in the data schema and the partition schema:  `report_time`;
Exception in thread "main" org.apache.spark.sql.AnalysisException: Found duplicate column(s) in the data schema: `report_time`;
```

#### 尝试搜索解决

通过 搜索 大多数说法是日志中可能存在相同字段但是不同大小写的问题，只需要在代码中添加以下配置即可解决

```scala
conf.set("spark.sql.caseSensitive", "true")
```

但是添加完之后，依然报相同错误，经过一番推测 可能是源日志 有问题，于是写了 以下脚本来校验每行 `json` 是否完整

file: check.php

```php
<?php
$file = "report.log";
$fd = fopen($file,"r");
$i=0;
while(!feof($fd)){
    $row = fgets($fd,65535);
    $arr = json_decode($row,true);  // 如果有不完整的数据 或者解析 json 失败的数据 就打印
    if(json_last_error()){
        echo "==============start={$i}=============\n";
        echo $row.PHP_EOL;
        echo "==============end==={$i}===========\n";
    }
    $i++;
}
fclose($fd);
```



得到输出如下:

```bash
....
==============start=586487=============
{...   省略若干无用信息
"app_version":"1.0.ice_model":"V2049A",
 ...   省略若干无用信息
}

==============end===586487===========
....
```

可以看到 存在 这样的数据，明显是 并发写入时  json 被截断

```bash
"app_version":"1.0.ice_model":"V2049A",
```

一看不合理啊 客户端日志上报 是用 `file_put_centents` 写入的， 已经添加了  `FILE_APPEND` 参数，此参数 对应 系统调用的  `write` `O_APPEND` 参数，记得在 `APUE` 中 写到 `write`  加上  `O_APPEND` 参数 就能保证写入数据是原子的，不会互相覆盖，但是目前现象是 写入的数据乱序了。

### 复现问题

猜测可能是 并发写入时导致的，因此 通过 PHP `多进程 `并发写同一文件 测试来复现

#### 使用多进程复现

file:  pcntl_chunk.php

```php
$out = "out.log";
$content = "record\n";

for ( $i = 1; $i <= 200; $i++ ) {
    $pid = pcntl_fork();
    if ( 0 == $pid ) {
        for ($c = 1; $c <= 10; $c++ ) {
            file_put_contents($out, $content, FILE_APPEND);
        }
        exit;
    }
    sleep(3);
    exit;
}
```



```bash
$ wc -l out.log
2000 out.log
```

正常 应该输出 2000 行，多次测试 均无异常，但是线上收集客户端写入的日志里确实存在被截断的情况

干脆直接 通过  `strace` 命令，追踪 收集上报日志的  php-fpm 进程

假设  1234  为某个  php-fpm 进程 id

```bash
$ strace  -p   1234
```

可以看到以下输出

```bash
...  省略若干无用信息

       
open("/data/collect/zyj/2021/04/21", O_WRONLY|O_CREAT|O_APPEND, 0666) = 5
fstat(5, {st_mode=S_IFREG|0644, st_size=4360482272, ...}) = 0
lseek(5, 0, SEEK_CUR)                   = 0
lseek(5, 0, SEEK_CUR)                   = 0
write(5, "{\"type\":\"up\",\"time\":\"2021-0"..., 8192) = 8192   # 可以看到这里  写入的日志 超过 8192 被分批写入
write(5, "755e18a\",\"ip\":\"127.0.0.0\",\""..., 7263) = 7263    # 这里写入余下的数据
close(5)                                = 0
write(4, "\1\6\0\1\0T\4\0X-Powered-By: PHP/7.2.27"..., 112) = 112
shutdown(4, SHUT_WR)                    = 0
recvfrom(4, "\1\5\0\1\0\0\0\0", 8, 0, NULL, NULL) = 8
recvfrom(4, "", 8, 0, NULL, NULL)       = 0
close(4)

...  省略若干无用信息
```

仔细看上面注释，存在系统调用 `write` 文件时 长度大于 `8192` 也就是 8k  分批写入的情况，因此推测 刚刚复现的脚本中写入的数据长度不够，所以没有成功复现
我们加大 写入内容的长度，再次用 `strace` 观察 在系统调用 `write` 时是否是分批写入

file:chunk.php

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

写入数据 长度为 30935 字节

```bash
# php chunk.php
30935
```

用 strace 执行可以看到如下输出

```bash
strace php chunk.php
```

```bash
... ... 省去无用信息

lstat("/root/test/out.log", {st_mode=S_IFREG|0644, st_size=12712298, ...}) = 0
open("/root/test/out.log", O_WRONLY|O_CREAT|O_APPEND, 0666) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=12712298, ...}) = 0
lseek(3, 0, SEEK_CUR)                   = 0
lseek(3, 0, SEEK_CUR)                   = 0
write(3, "{\"type\":\"report\",\"time\":16191055"..., 8192) = 8192
write(3, ":359},{\"abc551\":207},{\"abc552\":5"..., 8192) = 8192
write(3, "225},{\"abc1091\":544},{\"abc1092\":"..., 8192) = 8192
write(3, "266},{\"abc1603\":422},{\"abc1604\":"..., 6359) = 6359
close(3)                                = 0
close(2)                                = 0
close(1)                                = 0
close(0)                                = 0
... ... 省去无用信息
```

可以看到  写入数据 分了五次  8192*3 + 6359 = 30935 刚好是我们写入的长度。

### 再次复现

用 多进程 加大写入数据长度 并发 写入 out.log 文件

file : pcntl_fork.php

```php
$data = [
    "type"=>"report",
    "time"=>time(),
    "data"=>[],
];

for($i=0;$i<2000;$i++){
    $data["data"][] = ["abc".$i=>rand(100,999)];
}

$content = json_encode($data).PHP_EOL;

for ( $i = 0; $i < 20; $i++ ) {
    $pid = pcntl_fork();
    if ( 0 == $pid ) {
        for ($c = 0; $c < 10; $c++) {
            $out = "out.log";
            file_put_contents($out, $content, FILE_APPEND);
        }
        exit;
    }
}
```

执行  

```bash
php  pcntl_fork.php
```

用  check.php  检查 输出 out.log 输出 发现有以下内容

```bash
==============start=94=============
abc293":823},{"abc294":539},{"abc29
...
==============end===94===========
```

由此问题复现，可以得出结论，file_put_contents 时 如果只加 FILE_APPEND 参数，那么 写入长度 不大于 8k 时是原子写入，如果大于 8k 就不是原子，并发写入存在乱序的可能， 此时如果要杜绝这种情况 需要添加 LOCK_EX 参数。

一般情况 到这里就结束了，但是本着 刨根问底的思想，我们要一探究竟，为什么？ PHP 什么会出现这种情况，是内部 file_put_contents 实现的原因吗？  此时我们要到源码一探究竟

### 查看源码 过程

#### 1. 下载源码

```bash
wget https://github.com/php/php-src/archive/refs/tags/php-7.4.18RC1.tar.gz

tar xf php-7.4.18RC1.tar.gz

cd php-7.4.18RC1
```

#### 2. 先找到 file_put_contents 定义

file : ext/standard/file.c

```c
PHP_FUNCTION(file_put_contents)
{
	php_stream *stream;
	

.....  省去无关代码

		case IS_STRING:
			if (Z_STRLEN_P(data)) {
                // ======================= ！！！ 主要看这里  !!! =================
                // ==========  file_put_contents 中 写入主要是 php_stream_write ==
				numbytes = php_stream_write(stream, Z_STRVAL_P(data), Z_STRLEN_P(data));
                // ======================= ！！！ 主要看这里  !!! =================
				if (numbytes != Z_STRLEN_P(data)) {
					php_error_docref(NULL, E_WARNING, "Only %zd of %zd bytes written, possibly out of free disk space", numbytes, Z_STRLEN_P(data));
					numbytes = -1;
				}
			}
			break;

......  省去无关代码 
	php_stream_close(stream);

	if (numbytes < 0) {
		RETURN_FALSE;
	}

```

#### 3. 查看  php_stream_write 定义

file : main/php_streams.h

```c
#define php_stream_write(stream, buf, count)    _php_stream_write(stream, (buf), (count))
```

#### 4. 查看 php_stream 实现

file: main/streams/streams.c

```c
PHPAPI ssize_t _php_stream_write(php_stream *stream, const char *buf, size_t count)
{
    ssize_t bytes;

    if (count == 0) {
        return 0;                                                                                        }

    ZEND_ASSERT(buf != NULL);
    if (stream->ops->write == NULL) {
        php_error_docref(NULL, E_NOTICE, "Stream is not writable");
        return (ssize_t) -1;
    }

    if (stream->writefilters.head) {                                                                         bytes = 		 _php_stream_write_filtered(stream, buf, count, PSFS_FLAG_NORMAL);
    } else {
          // ==========  这里 又跳到   _php_stream_write_buffer 函数
        bytes = _php_stream_write_buffer(stream, buf, count);
    }

    if (bytes) {
        stream->flags |= PHP_STREAM_FLAG_WAS_WRITTEN;
    }
                                                                                                         return bytes;
}
```

#### 5. 继续 看 _php_stream_write_buffer 函数实现

file : main/streams/streams.c

```c
/* Writes a buffer directly to a stream, using multiple of the chunk size */
static ssize_t _php_stream_write_buffer(php_stream *stream, const char *buf, size_t count)
{
    ssize_t didwrite = 0, justwrote;
                                                                                                        
...  省略无关代码
    while (count > 0) {
        size_t towrite = count;
        // ==================== !!!! 看这里  chunk_size 超过 chunk_size 会被截断 ================
        if (towrite > stream->chunk_size)
            towrite = stream->chunk_size;
        // ====================== chunk_size ===================

        justwrote = stream->ops->write(stream, buf, towrite);
        if (justwrote <= 0) {                                                                                    /* If we already successfully wrote some bytes and a write error occurred
             * later, report the successfully written bytes. */
            if (didwrite == 0) {
                return justwrote;
            }
            return didwrite;
        }

... 省略无关代码     
    }

    return didwrite;
}
```

#### 6. 查看  chunk_size 大小

在  php_stream 定义

file:   main/php_streams.h

```c
typedef struct _php_stream php_stream;
```

file:  main/php_streams.h

```c
struct _php_stream  {
    ...

    /* how much data to read when filling buffer */
    size_t chunk_size;
    
    ...   
}
```

我们 看到 chunk_size 大小 由  size_t  决定，根据 系统实现 在 64 位系统中为 8k 长度即为 8192 , 这就可以看到 file_put_contents 如果大于 8k 就不是一个系统调用了， 因此不是原子的，不加锁的情况 会乱序。

那么为什么 file_put_contents 要限制 写入 大小超过 size_t 要分批呢，是PHP 独有的还是 其它语言也有这个问题  ?

### 其它语言是否有这个问题？

我们接着以  C 语言查看是否有同样现象

file : write.c

```c
#include <stdio.h>
#include <string.h>

int main()
{
   FILE *fp = NULL;

   char buff[10000];
   int i;

   for(i=0;i<2000;i++){
       strcat(buff, "abcdef\n");
   }

   fp = fopen("c_out.log", O_CREAT|O_APPEND);
   fputs(buff, fp);
   fclose(fp);
}
```

编译执行

```bash
cc  write.c  -o write
strace write
```



```bash
open("c_out.log", O_RDWR|O_CREAT|O_TRUNC, 0666) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=0, ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f5b4d4b2000
write(3, "abcdef\nabcdef\nabcdef\nabcdef\nabcd"..., 8192) = 8192
write(3, "cdef\nabcdef\nabcdef\nabcdef\nabcdef"..., 1812) = 1812
close(3)                                = 0
munmap(0x7f5b4d4b2000, 4096)            = 0
exit_group(0)                           = ?

```

可以看到 C 语言中 `write`  超过 8k 也会拆分，看来不是 php 独有的问题，是系统的问题。

为什么系统调用 `wirte` 时 超过 8k 就会不是原子操作， 这个问题我们下篇文章继续探讨。

