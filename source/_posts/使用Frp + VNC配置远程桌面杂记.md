---
title: 【内网穿透】使用Frp + VNC配置远程桌面杂记
date: 2023-1-4
tags: [网络]
cover: https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/16631.webp
mathjax: true
---

# 一. VNC部分

参考链接[Jetson nano(Ubuntu18.04) 网线连接电脑，实现VNC远程桌面_PiQiuNi的博客-CSDN博客_jetson nano连接电脑](https://blog.csdn.net/PiQiuNi/article/details/123798298)

本文通过网线连接jetson nano(Ubuntu18.04) 与windows电脑，实现了网络共享及VNC远程桌面访问
配置Jetson nano (此过程需要连接屏幕及外设)
以下内容来自系统自带文档 “README-vnc.txt”

安装VNC

```
sudo apt update
sudo apt install vino
```

开启VNC服务
实现开机自动启动

```
mkdir -p ~/.config/autostart
cp /usr/share/applications/vino-server.desktop ~/.config/autostart
```

配置VNC服务

```
gsettings set org.gnome.Vino prompt-enabled false
gsettings set org.gnome.Vino require-encryption false
```

设置连接密码

```
gsettings set org.gnome.Vino authentication-methods "['vnc']"
gsettings set org.gnome.Vino vnc-password $(echo -n '你的密码'|base64)
```

设置无屏幕启动分辨率

```
sudo vim /etc/X11/xorg.conf 
```

在文件末尾插入以下内容（可自行设置分辨率）：

```
Section "Screen"
   Identifier    "Default Screen"
   Monitor       "Configured Monitor"
   Device        "Tegra0"
   SubSection "Display"
       Depth    24
       Virtual 1280 800 # Modify the resolution by editing these values
   EndSubSection
EndSection
```



重启jetsonnano

```
sudo reboot
```

# 二. frp部分
参考链接[(1条消息) VNC+frp实现远程访问Ubuntu和树莓派_圆滚熊的博客-CSDN博客](https://blog.csdn.net/y459541195/article/details/102522290)

### 搭建方式一：有公网服务器情况下

### 1.准备

1. **公网服务器一台**（腾讯云，阿里云，华为云等服务器均可）
2. **内网穿透工具 frp** （免费开源）
3. **远程控制软件 RealVNC**

### 2.服务器上部署（服务端）

首先了解一下frp是什么？
frp是一个可用于内部网穿透的高级反向代理应用程序，支持tcp，udp协议，为http和https应用协议提供了额外的能力，并且尝试性支持了点对点穿透。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012162031891.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70#pic_center)

- **第一步**：**下载frp到公网服务器**

登录公网服务器下载frp，frp下载地址：https://github.com/fatedier/frp/releases
找到对应版本下载
（注:可以输入arch，查看cpu架构，云服务和Ubuntu16.0.4都是x86_64处理器架构，所以下载amd64的包）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012163441511.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70#pic_center)
这里下载的是当前更新的最新版frp_0.29.0_linux_amd64.tar.gz

```
wget https://github.com/fatedier/frp/releases/download/v0.29.0/frp_0.29.0_linux_amd64.tar.gz
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012164116147.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
可能下载有点慢，等待一会儿

- **第二步**：**配置frp**

下载完之后进行解压：

```
tar -xzvf frp_0.29.0_linux_amd64.tar.gz
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/201910141600021.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
解压之后，找到frps.ini这个文件，并打开：

```
cd frp_0.29.0_linux_amd64
sudo nano frps.ini
```

参考frps_full.ini，添加一下内容：

```
[common]
bind_port = 7000  

token = 12345678  

dashboard_port = 7500 
dashboard_user = admin 
dashboard_pwd = admin

max_pool_count = 5
log_file = ./frps.log  
log_level = info
log_max_days = 3
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019101416223818.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
保存，退出。
把云服务器的7000,7500等相应的端口放行。我这里的后台管理端口是7600，不是上面的 7500。
在当前目录下运行frps:

```
nohup ./frps -c ./frps.ini &
```

查看7600端口是否在监听

```
netstat -ap | grep 7600
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014164556357.png)
测试后台能否打开：

```
#云服务器公网ip + 7600
http://x.x.x.x:7600
```

记得x.x.x.x替换为自己公网ip地址，提示输入账户密码，默认admin
界面是这样的：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014165133252.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
至此，服务端frp部署成功了

### 3. Ubuntu上部署（客户端）

- **第一步：Ubuntu打开桌面共享**

这里操作的Ubuntu系统版本是16.0.4
搜索出桌面共享，输入sharing
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014172708797.png)
点击打开“Desktop Sharing”，勾选如下配置：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014172950390.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
第3那里要设置访问密码，并记住。

- **第二步：配置frpc**

下载frp_0.29.0_linux_amd64.tar.gz，解压后打开frpc.ini

```
tar -xzvf frp_0.29.0_linux_amd64.tar.gz
cd frp_0.29.0_linux_amd64
sudo nano frpc.ini
```

添加一下内容：

```
[common]
server_addr = x.x.x.x  
server_port = 7000   
token = 12345678	  

[ubuntu-ssh]         
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 8085  

[ubuntu-vnc]        
type = tcp
local_ip = 127.0.0.1
local_port = 5900 
remote_port = 5910   
```

x.x.x.x要替换为自己的公网ip
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014190107700.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
在当前目录下运行frpc.ini文件：

```
nohup ./frpc -c ./frpc.ini &
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014191906161.png)
查看一下后台，看看是否在线
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014192105720.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
说明部署成功了。

- **第三步：加入开机启动**

打开Ubuntu的/etc/rc.local开机启动文件：

```
sudo nano /etc/rc.local
```

添加自己的文件启动路径：

```
nohup  x/x/frpc -c x/x/frpc.ini &
```

x/x/需要替换掉，如果不知道frpc路径，可以

```
pwd
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014193046189.png)
这里的路径如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191018101655805.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
添加完之后，保存退出。

### 4.树莓派上部署（客户端）

这里用的是树莓派3b+
下载的是arm文件：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014194009297.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
下载后解压，同样是打开frpc.ini文件：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014194619451.png)

```
sudo nano frpc.ini
```

和上面Ubuntu配置frpc一样，只是改了端口和名称，箭头所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014194942389.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
记得云服务器上开放相应端口段，如：5900-6000，22, 80等。

**加入开机启动：**
刚开始尝试了直接在 rc.local里启动frpc失败。
参考这篇博文：https://blog.csdn.net/zmy12007/article/details/84642081，树莓派开机后让frpc.ini延迟启动即可

```
sudo nano startfrpc.sh
```

添加一下内容：

```
#/bin/bash
cd /home/pi/frp_0.29.0_linux_arm
echo "start frpc from shell" >> ./log.txt
sleep 15s
nohup ./frpc -c ./frpc.ini &
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014201514709.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
保存退出
添加文件权限

```
sudo chmod +x startfrpc.sh
```

接着打开rc.local文件

```
sudo nano /etc/rc.local
```

添加如下内容：

```
echo "start rc.local" > /home/pi/rc.log

nohup /bin/bash /home/pi/startfrpc.sh &
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014202213180.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
保存， 退出
重启树莓派：

```
sudo reboot
```

查看一下后台，看看是否在线：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014202945648.png)
至此，树莓派部署成功了。

### 5.远程连接

这里用win10系统连接Ubuntu和树莓派

**5.1 远程连接Ubuntu**

**vnc连接：**
到官网下载vnc:https://www.realvnc.com/en/connect/download/viewer/相对应版本的vnc viewer
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014204024583.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
下载后安装，打开VNC Viewer软件
输入 **公网ip + 端口号**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014190925160.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019101420513725.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
这个密码是Ubuntu桌面共享时配置的密码，输入密码后确定，这样就完成了远程了连接
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014205824265.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
**ssh连接：**

```
ssh -oPort=8085 ubuntu@x.x.x.x
```

端口8085，ubuntu 为用户名 ，x.x.x.x为公网ip
此时，出现了错误：
ssh_exchange_identification: Connection closed by remote host
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014213640719.png)
查看ssh是否安装：

```
sudo ps -e |grep ssh
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014213912639.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
结果没有安装，需要安装一下：

```
sudo apt-get update
sudo apt-get install openssh-server
```

再次查看sudo ps -e |grep ssh是否安装
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014215448405.png)
出现了sshd，说明安装成功了。
再次尝试ssh:
![在这里插入图片描述](https://img-blog.csdnimg.cn/201910142159373.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
出现了错误“ `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`”，打开`C:\Users\Administrator\.ssh`下文件，用记事本打开known_hosts，删除选中的部分
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019101422072473.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
再试一下，输入ubuntu登录密码成功：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014220944705.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)

**5.2 远程连接树莓派**

**vnc连接：**
打开树莓派的vnc和ssh:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014222421723.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014222537813.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70#pic_center)
重启树莓派

```
reboot
```

重启后右上角会出现vnc图标，单击图标打开，选择Option选项
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014223442468.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014222918648.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
安全项，进行访问密码配置，记住密码

在win10上，打来VNC Viewer软件，输入`公网ip+端口`，再输入访问密码
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014223849455.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
远程访问成功
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014224021542.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
**ssh连接：**

```
ssh -oPort=8086 pi@x.x.x.x
```

端口8086，pi为用户名 ，x.x.x.x为公网ip
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019101422453057.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
ssh连接成功！

### 搭建方式二：无公网服务器情况下

推荐这种搭建，方便快捷。注意：这里通过win10远程访问Ubuntu和树莓派，win10安装VNC Viewer ，而Ubuntu和树莓派安装的是VNC Server。当然Ubuntu和树莓派也可以安装VNC Viewer来远程访问其它设备，根据自己需求来。

### 1.Ubuntu上部署

#### 1.1 安装

首先在ubuntu 上安装VNC Server。
可以到官网下载：https://www.realvnc.com/en/connect/download/vnc/linux/
找到相对应的版本。
或者通过命令下载安装：

```python
wget https://www.realvnc.com/download/file/vnc.files/VNC-Server-6.4.1-Linux-x64.deb
```

安装：

```python
sudo dpkg -i VNC-Server-6.4.1-Linux-x64.deb
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191109104010288.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
查看运行情况：

```bash
ps aux|grep vnc
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191109110115414.png)
想要卸载的话：

```python
sudo apt-get purge realvnc-vnc-server
```

卸载vnc viewer：

```python
sudo apt-get purge realvnc-vnc-viewer
```

#### 1.2 配置

在图形界面搜索VNC Server，输入Ubuntu管理员密码并打开
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191109104457922.png)
右上角会有个图标，右击选择`Licensing...`打开
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191109104901765.png)
弹出一个注册界面，没有VNC账号先注册一个

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191109105611659.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
输入账号密码，点击`sign in`，出现如下界面：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191109110423766.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
上图中填写的密码是你远程连接时候用到的密码，即`远程连接密码`，不能和vnc账户密码相同，填写一下，点击下一步，选择家庭订阅非商业用途（`Home subscription`）,接着点击next，起个名字，我这里是`ubuntu`，接着点击完成即可。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191109111202123.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)

#### 1.3 连接

接下来我们用win10远程连接乌班图，打开win10上的VNC Viewer
登录VNC账号，在地址栏中直接输入`ubuntu` ，输入`远程连接的密码`
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191109111841342.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
这样就连接上了，随时随地的访问远程电脑了。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191109112049518.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)

### 2.树莓派上部署

和ubuntu基本类似步骤，树莓派好的一点是已经预装的有vnc server了。
可以参考上面搭建方式一中的步骤打开树莓派的VNC
选择第一项，下一步
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191109114813820.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
设置访问名字，可以看到已经有一个存在了。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191109114845279.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
点击Apply。完成退出

在win10中的VNC Viewer 中登录账号就看到了要访问的树莓派。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191109133416413.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3k0NTk1NDExOTU=,size_16,color_FFFFFF,t_70)
如果不想那么费事，或者没有云服务器的，还是推荐第二种搭建方式，整体体验上比第一种要流畅一些。
