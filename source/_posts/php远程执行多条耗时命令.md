---
title: PHP 远程执行多条耗时命令
date: 2021-01-27 20:00:26
tags:
  - PHP
  - SHELL
  - LINUX
  - SSH
categories:
  - PHP
---

本文介绍 PHP 用ssh 的方式远程执行任务的办法

开发中有时需要用 ssh 远程连接到其它服务器执行任务脚本，在 PHP 中虽然有 ssh 扩展 或是封装的 ssh 操作的包 phpseclib。

简单使用了下 他们，发现如果是执行短时间就能完成的命令，基本能满足需求。但是如果执行耗时较长的命令时，由于网络抖动或其他原因连接意外断开，那么此时 php 并不会捕获异常而是会陷入假死状态，通过  ps  aux | grep  ssh.php 查看如下

```bash
root       7156   0.0  0.0  4341748   7228 s005  S+   11:28下午   0:00.06 php ssh.php
```

可以发现此时状态是 S+ ，看起来是一切正常，但是手动连接要执行任务的机器可能发现任务已经终止，SSH 一般的客户端来说是可以设置 心跳时间的，但是由于 PHP 的ssh 扩展没有心跳时间等参数，导致连接如果时间长，没有发送心跳，很容易被系统认为连接已经中断而关闭连接，此时如果单纯用PHP 的话很难解决。



一般可以想到使用 nohup 的方式

```bash
nohup  ./task.sh  > /dev/null 2>&1 & 
```

通过 nohup 会拿到 tash.sh 的进程id，可以定时来检测 进程id 是否存在而判断任务是否结束，但是进程如果发生异常而退出就无法检测到

可以把标准错误输出到 err.log 中

```bash
nohup ./test.sh > /dev/null  2>/tmp/err.log & 
```

此时可以通过查看 /tmp/err.log 中的内容来判断是否正常执行，但是如果要执行一系列命令的情况，一个个处理将会非常麻烦。



一般这种情况可以通过 委托给 agent 的方式来执行，但是如果没有 agent 仅仅执行几个命令单独再实现一个agent 部署到每台执行任务的机器的成本略高。

我们都知道 Linux 命令成功或失败是有标准输出和错误输出的那么就可以利用这个特性，执行完任务无论成功或失败就给个回调地址

用 shell 脚本如下

```bash
#!/bin/bash

cmd='ls -al'
callback_url="http://abc.com"

success()
{
    curl  "$callback_url?&result=success&task_id={$task_id}";
}

error()
{
    echo $*;
    curl  -g "$callback_url?&result=failed&msg=\$*&task_id={$task_id}";
    exit -1;
}

    {$cmd} 2>/tmp/err.out

if [ "\$?" -ne 0 ] ; then
    error_info=`cat '/tmp/err.out' | tr "\\n" " " `
    error exec {$cmd} \${error_info}
fi

    success "success"
```

基于上面这个shell 脚本，就可以通过 PHP 构造一个函数来生成多条命令的脚本

```php
/**
 * @param $callbackUrl string  回调地址
 * @param $cmds array 执行命令的数组
 **/
function makeMultiCmdShell($callbackUrl,$cmds)
{
        $url = $callbackUrl;
       
        $bash_content= <<<EOF
#!/bin/bash

success()
{
    curl  "$callback_url?&result=success&task_id={$task_id}";
}

error()
{
    echo \$*;
    curl  -g "$callback_url?&result=failed&msg=\$*&task_id={$task_id}";
    exit -1;
}
EOF;

        foreach($cmds as $cmd){
            $bash_content .= <<<EOF

    {$cmd} 2>/tmp/err.out

if [ "\$?" -ne 0 ] ; then
    error_info=`cat '/tmp/err.out' | tr "\\n" " " `
    error exec {$cmd} \${error_info}
fi
EOF;

        }
        $bash_content .=<<<EOF
    success "success"
EOF;
     
        return $bash_content;
}
```



通过这个函数就可以得到一个 shell 脚本，放到远程服务器后 通过 nohup 执行，就可以不用一直轮询它的执行状态，等待回调消息即可，

