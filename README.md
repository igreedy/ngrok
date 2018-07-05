---
title: 搭建ngrok远程桌面的环境
tags: 
 - ngrok
categories: go
date: 2018-06-29 17:29:21
---

[TOC]
#### 场景使用
>在家里用笔记本laptop可以远程访问公司的内部网的工作电脑computer。其效果就是服务器server开启ngrok服务端，公司工作电脑computer开启ngrok.bat脚本客户端，然后家里笔记本laptop用mstsc远程桌面。输入my.domain.com:50123（这个域名需要自己购买）。然后输入工作电脑computer的用户和密码。这样远程登录成功。这样即使你那工作电脑computer是公司内网的电脑，也没有关系。这样如果公司有急事要处理就省得去公司办公了。直接远程桌面处理即可。
#### 服务器server环境准备
##### 服务器server的centos环境
```bash
yum -y install zlib-devel openssl-devel perl hg cpio expat-devel gettext-devel curl curl-devel perl-ExtUtils-MakeMaker hg wget gcc gcc-c++ git
```
##### 服务器server的go语言环境
go使用版本，可以用go verison查看，是 go version go1.8.3 linux/amd64
下载方式可以使用
```bash
// 删除 关于golang的依赖包
rpm -qa|grep golang|xargs rpm -e
// 下载安装包
wget https://storage.googleapis.com/golang/go1.8.3.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.8.3.linux-amd64.tar.gz
vim /etc/profile
//末尾添加''' '''里面的内容：
'''
#go lang
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin
'''
source /etc/profile
//检测是否安装成功go
go version
```
##### 工作电脑computer的环境准备

![](https://raw.githubusercontent.com/igreedy/igreedy.github.io/master/images/go/0.png)



##### 域名的准备
>远程桌面环境搭建需要一个域名，这个可在[godaddy](www.godaddy.coom)自行购买。
我买的域名是domain.com。然后ngrok环境搭建用的域名是**my.domain.com**。
下面代码有关这个配置，需要你自己修改成你购买的域名。当然在你购买的域名，需要
在DNS管理页面上添加一个记录。

![](https://raw.githubusercontent.com/igreedy/igreedy.github.io/master/images/go/1.png)



##### 端口的准备
>远程桌面环境搭建需要公司服务器server提供两个端口。这个需要在 **/etc/sysconfig/iptables** 里配置。
例如一个是远程桌面需要访问的端口，这里我用的是50123(这个自己定)，另外一个是工作电脑computer的
ngrok.bat脚本客户端与公司服务器server交互的端口4443(这个是默认的)。

```bash
vim /etc/sysconfig/iptables
// 添加''' '''里面的内容
'''
-A INPUT -m state --state NEW -m tcp -p tcp --dport 50123 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 4443 -j ACCEPT
'''
/etc/init.d/iptables reload
/etc/init.d/iptables restart
// 可用下面的命令来查看开放的端口
iptables -nL
```

#### 服务器server上安装ngrok
##### 下载ngrok安装包
[完整的ngrok下载包](https://github.com/igreedy/ngrok/raw/master/ngrok.tar.gz)

>如果下载官方的安装包，在之后执行make release-server 可能会报错。
因为不能翻墙什么缘故。不能在ngrok/src目录下下载github.com 和gopkg.in 里面的数据。
所以我把下好的数据添加到src目录下。这样执行 make release-server。
就不会因为网络缘故而报错。[下载](https://github.com/igreedy/ngrok/raw/master/ngrok.tar.gz)
也可以用下面方式来安装。

```bash
cd /usr/local
wget https://github.com/igreedy/ngrok/raw/master/ngrok.tar.gz
```
##### 生成证书
```bash
cd /ngrok
mkdir cert
cd cert
NGROK_DOMAIN="my.domain.com"
openssl genrsa -out rootCA.key 2048
openssl req -x509 -new -nodes -key rootCA.key -subj "/CN=$NGROK_DOMAIN" -days 5000 -out rootCA.pem
openssl genrsa -out device.key 2048
openssl req -new -key device.key -subj "/CN=$NGROK_DOMAIN" -out device.csr
openssl x509 -req -in device.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out device.crt -days 5000
```
##### 覆盖原本证书
```bash
yes|cp rootCA.pem /usr/local/ngrok/assets/client/tls/ngrokroot.crt
yes|cp device.crt /usr/local/ngrok/assets/server/tls/snakeoil.crt
yes|cp device.key /usr/local/ngrok/assets/server/tls/snakeoil.key
```
##### 查看不同系统下的不同配置信息
```
#Linux 平台 32 位系统：GOOS=linux GOARCH=386
#Linux 平台 64 位系统：GOOS=linux GOARCH=amd64
#Windows 平台 32 位系统：GOOS=windows GOARCH=386
#Windows 平台 64 位系统：GOOS=windows GOARCH=amd64
#MAC 平台 32 位系统：GOOS=darwin GOARCH=386
#MAC 平台 64 位系统：GOOS=darwin GOARCH=amd64
#ARM 平台：GOOS=linux GOARCH=arm
```
##### 编译生成 Linux 服务端
```bash
// 会在 ngrok/bin/ 目录下生成 go-bindata 和 ngrokd 这个文件
make release-server
// 上面的代码也可以用下面的代码代替
GOOS=linux GOARCH=amd64 make release-server
```

##### 编译生成 window 客户端
```bash
// 下面代码全执行后，会在 ngrok/bin/windows_amd64/ 目录下生成 ngrok.exe 这个文件
cd /usr/local/go/src
GOOS=windows GOARCH=amd64 ./make.bash
cd /usr/local/ngrok/
GOOS=windows GOARCH=amd64 make release-client
```
#### 运行与使用

##### 服务器server运行ngrok
```bash
cd /usr/local/ngrok
nohup ./bin/ngrokd -tlsKey="assets/server/tls/snakeoil.key" -tlsCrt="assets/server/tls/snakeoil.crt" -domain="my.domain.com"  -httpAddr=":7788" &
```
##### 工作电脑computer配置
>将服务器server的/usr/local/ngrok/bin/windows_amd64/ngrok.exe的 ngrok.exe 文件拷贝到工作电脑computer的E盘文件夹ngrok目录下
然后在这个e:\ngrok\的目录下 新建一个 ngrok.cfg 和 ngrok.bat 两个文件。
用notepad++编辑 ngrok.cfg文件

```
server_addr: "my.domain.com:4443"
trust_host_root_certs: false
tunnels:
  mstsc:
    remote_port: 50123
    proto:
      tcp: "127.0.0.1:3389"
```
> 用notepad++编辑 ngrok.bat 文件

```
ngrok.exe -config=ngrok.cfg start mstsc
```
>然后直接双击  ngrok.bat 文件。就会弹出如下图片

![](https://raw.githubusercontent.com/igreedy/igreedy.github.io/master/images/go/2.png)


>这时候你就可以用别的电脑打开mstsc 远程桌面来尝试连接了。

![](https://raw.githubusercontent.com/igreedy/igreedy.github.io/master/images/go/3.png)
