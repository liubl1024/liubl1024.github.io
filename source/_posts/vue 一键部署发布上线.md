---
title: Vue 一键部署发布上线
date: 2021-01-23 20:00:26
tags:
  - VUE
  - LINUX
  - 脚本工具
  - expect
categories:
  - VUE
---


 一般来说软件开发流程规范的公司会有标准 CI/CD 流程，本文不谈 Jenkins 或是git hook 等工具，介绍两种在小项目中最快也最原始的方式一键打包上线的方式。

-  webpack-sftp-client
-  shell  expect



​    webpack-sftp-client，使用步骤如下：

1. 安装

```
npm install webpack-sftp-client -D
```



2. 在 webpack.dev.js webpack.prod.js 中引入并配置

```javascript
const WebpackSftpClient = require('webpack-sftp-client');

new WebpackSftpClient({
  port: '22',
  host: '',  // 服务器ip
  username: 'root', // 用户名
  password: '', // 密码
  path: './test/', // 本地路径
  remotePath: '',  // 远端服务器路径
  verbose: true
}),

```



然后 使用 npm run prod  或者 npm run dev 打包后就会自动上线了。



shell 的方式 就是通过编写 shell 脚本的形式，如下:

```bash
#!/bin/bash
# 服务器ip
serverIp=""
# 服务器存放目录
dstDir=""

npm run prod

zip -r dist.zip dist
scp dist.zip root@${serverIp}:${dstDir}
ssh  root@${serverIp}  "cd ${dstDir}; unzip -o dist.zip"
```

这种方式 需要提前使用 ssh-copy-id 打通 ssh 免密登陆，如果开发机是临时申请或 ip 变动频繁，那么每次都要配置免密登陆无疑是非常麻烦的事情，作为开发者如果是重复的事情肯定要自动化。

  一般来说 临时申请的机器 密码不会频繁变更，因此可以通过输入密码的方式，但是 ssh 密码又是交互式的，这里再介绍两种工具来处理 ssh 密码输入

- sshpass 
- expect

   sshpass 使用非常简单，可以通过 -p 直接输入密码 或者 -f 指定包含密码的文本

```bash

sshpass -p'passwd'  ssh root@serverIp

# 或是
sshpass -f ~/.ssh/passwd_file  ssh root@serverIp
```

  expect 可以用来处理交互式的输入，它可以通过匹配正则的方式来发送对应规则的输入，例子如下：

  ```bash
  
  /usr/bin/expect <<-EOF
  set timeout 30
  spawn  scp dist.zip  root@${serverIp}:${dstDir}
  expect  "*password:"
      send "${passwd}\n"
  expect eof
  set timeout 300
  spawn  ssh  root@${serverIp}  "cd ${dstDir}; unzip -o dist.zip"
  expect  "*password:"
     send "${passwd}\n"
  expect eof
  
  EOF
  ```

综合上面我们就可以完善一个发布上线的脚本：

```bash
#!/bin/bash

# 服务器密码
passwd=""
# 服务器ip
serverIp=""
# 服务器目标目录
dstDir=""

npm run prod

if [ $? != 0 ];then
    echo "打包失败"
    exit 1
fi
zip -r dist.zip dist

/usr/bin/expect <<-EOF
set timeout 30
spawn  scp dist.zip  root@${serverIp}:${dstDir}
expect  "*password:"
    send "${passwd}\n"
expect eof

set timeout 300
spawn  ssh  root@${serverIp}  "cd ${dstDir}; unzip -o dist.zip"
expect  "*password:"
   send "${passwd}\n"
expect eof

EOF
```

