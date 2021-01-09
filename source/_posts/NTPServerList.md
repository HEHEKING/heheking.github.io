---
title: 常用 NTP 服务器列表
date: 2020/11/22 20:32:51
categories:
- 教程
tags:
- ntp
- linux
---
NTP Server List
<!-- more -->
## 1. NTP Server List

```
210.72.145.44 (国家授时中心服务器IP地址)  
133.100.11.8 日本 福冈大学  
time-a.nist.gov 129.6.15.28 NIST, Gaithersburg, Maryland   
time-b.nist.gov 129.6.15.29 NIST, Gaithersburg, Maryland   
time-a.timefreq.bldrdoc.gov 132.163.4.101 NIST, Boulder, Colorado   
time-b.timefreq.bldrdoc.gov 132.163.4.102 NIST, Boulder, Colorado   
time-c.timefreq.bldrdoc.gov 132.163.4.103 NIST, Boulder, Colorado   
utcnist.colorado.edu 128.138.140.44 University of Colorado, Boulder   
time.nist.gov 192.43.244.18 NCAR, Boulder, Colorado   
time-nw.nist.gov 131.107.1.10 Microsoft, Redmond, Washington   
nist1.symmetricom.com 69.25.96.13 Symmetricom, San Jose, California   
nist1-dc.glassey.com 216.200.93.8 Abovenet, Virginia   
nist1-ny.glassey.com 208.184.49.9 Abovenet, New York City   
nist1-sj.glassey.com 207.126.98.204 Abovenet, San Jose, California   
nist1.aol-ca.truetime.com 207.200.81.113 TrueTime, AOL facility, Sunnyvale, California   
nist1.aol-va.truetime.com 64.236.96.53 TrueTime, AOL facility, Virginia  
————————————————————————————————————-----------------------------------------------------
ntp.sjtu.edu.cn 202.120.2.101 (上海交通大学网络中心NTP服务器地址）  
s1a.time.edu.cn 北京邮电大学  
s1b.time.edu.cn 清华大学  
s1c.time.edu.cn 北京大学  
s1d.time.edu.cn 东南大学  
s1e.time.edu.cn 清华大学  
s2a.time.edu.cn 清华大学  
s2b.time.edu.cn 清华大学  
s2c.time.edu.cn 北京邮电大学  
s2d.time.edu.cn 西南地区网络中心  
s2e.time.edu.cn 西北地区网络中心  
s2f.time.edu.cn 东北地区网络中心  
s2g.time.edu.cn 华东南地区网络中心  
s2h.time.edu.cn 四川大学网络管理中心  
s2j.time.edu.cn 大连理工大学网络中心  
s2k.time.edu.cn CERNET桂林主节点  
s2m.time.edu.cn 北京大学
```

转自：https://www.cnblogs.com/kaishirenshi/p/10475911.html

## 2. ntpdate

### 安装 ntp 

```
yum install ntp -y
```

### 配置本地服务器，修改 ntp.conf 

```shell
# ntp.conf的位置
vi /etc/ntp.conf

#改为如下内容：
————————————————————————————
server 127.127.1.0	# local clock
fudge 127.127.1.0 stratum 10	# stratum 设置为其他值亦可，范围为 0 ~ 15
————————————————————————————
# 这使得别的主机可以通过 ntpdate master 来同步 master 节点的时间

# 在客户端写入定时脚本
crontab -e
*/30 * * * * /usr/sbin/ntpdate master
```

关于 ntp.conf 更详细的配置，请查询其他相关资料，这里只是简单介绍。

### 使用 ntpdate 同步时间服务器

```shell
ntpdate -u 210.72.145.44
ntpdate -u ntp.ntsc.ac.cn
# 有时可以这样：
ntpdate ntp.ntsc.ac.cn
# 可能发生的错误，或许是防火墙造成的？
若不加上-u参数， 会出现以下提示：no server suitable for synchronization found
```

