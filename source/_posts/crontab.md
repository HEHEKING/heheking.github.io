---
title: Crontab 笔记
date: 2020/11/22 22:56:34
categories:
- [笔记]
tags:
- cron
- linux
pic:
---

一个在线查看 cron 表达式网站：

https://tool.lu/crontab/

<!-- more -->

## 一、格式

共 5 位

```
*    *    *    *    *
-    -    -    -    -
|    |    |    |    |
|    |    |    |    +----- 星期中星期几 (0 - 7) (星期天 为0)
|    |    |    +---------- 月份 (1 - 12) 
|    |    +--------------- 一个月中的第几天 (1 - 31)
|    +-------------------- 小时 (0 - 23)
+------------------------- 分钟 (0 - 59)
```

## 二、示例

周一到周五每天下午 5:00 寄一封信给 alex@domain.name：

```
0 17 * * 1-5 mail -s "hi" alex@domain.name < /tmp/maildata
```

每月每天的午夜 0 点 20 分, 2 点 20 分, 4 点 20 分....执行 echo "haha"：

```
20 0-23/2 * * * echo "haha"
```

**注意：**当程序在你所指定的时间执行后，系统会发一封邮件给当前的用户，显示该程序执行的内容，若是你不希望收到这样的邮件，请在每一行空一格之后加上 **> /dev/null 2>&1** 即可，如：

```
20 03 * * * . /etc/profile;/bin/sh /var/www/runoob/test.sh > /dev/null 2>&1 
```

## 三、使用

查看当前用户的定时任务

```
crontab -e
```

编辑该文件后，保存在目录：**/var/spool/cron/**

## 四、常见问题

脚本无法执行问题

如果我们使用 crontab 来定时执行脚本，无法执行，但是如果直接通过命令（如：./test.sh)又可以正常执行，这主要是因为无法读取环境变量的原因。

**解决方法：**

- 1、所有命令需要写成绝对路径形式，如: **/usr/local/bin/docker**。

- 2、在 shell 脚本开头使用以下代码：

  ```
  #!/bin/sh
  
  . /etc/profile
  . ~/.bash_profile
  ```

  3、在 **/etc/crontab** 中添加环境变量，在可执行命令之前添加命令 **. /etc/profile;/bin/sh**，使得环境变量生效，例如：

  ```
  20 03 * * * . /etc/profile;/bin/sh /var/www/runoob/test.sh
  ```

