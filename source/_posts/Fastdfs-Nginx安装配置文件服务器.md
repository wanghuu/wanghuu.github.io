---
title: Fastdfs+Nginx安装配置文件服务器
date: 2016-12-09 09:30:56
tags: [linux,Nginx]
---
最近在开发一个电商项目，需要用到图片上传功能。传统得做法是将图片存储在本地路径，但对于大型的网站来说，一般都会有单独存储图片的服务器，通过查阅相关资料，折腾了几天，我选择使用了Fastdfs+Nginx的方式来配置了一台文件存储服务器。

# Fastdfs是什么 #
> FastDFS是一个开源的轻量级分布式文件系统，它对文件进行管理，功能包括：文件存储、文件同步、文件访问（文件上传、文件下载）等，解决了大容量存储和负载均衡的问题。特别适合以文件为载体的在线服务，如相册网站、视频网站等等。

所以说使用Fastdfs不仅仅可以用来作图片存储，各种以文件为载体的服务都可以由它来提供。
> fastfds有两个角色：**跟踪服务**和**存储服务**，跟踪服务控制，调度文件以负载均衡的方式访问；存储服务包括：文件存储，文件同步，提供文件访问接口，同时以key value的方式管理文件的元数据。
> 跟踪和存储服务可以由1台或者多台服务器组成，同时可以动态的添加，删除跟踪和存储服务而不会对在线的服务产生影响，在集群中，tracker服务是对等的。

由于是第一次使用，对原理也并不是有很深刻的理解，所以，我更加养生记录一下安装和配置的过程，安装的过程还是会遇到各种各样的小问题的，是个比较繁琐考验耐心的过程。

----------

# 安装步骤 #
### 1. 准备安装包
<div align="left"><img src="/img/fastdfs/install_package.jpg"  alt="install_package" align=center /><div>
需要如图上的四个安装包，以上资源都可以在网上或者github上找到。由于fastdfs依赖libfastcommon，所以我们先安装libfastcommon，为了方便操作，首先切换到root用户。
我是使用centos6.2的系统。

### 2. 开始安装

由于是从github上直接下载的，格式为zip的压缩文件，所以使用unzip解压，如果是tar包可使用tar命令。
####  安装libfastcommon


    cd  /usr/local
    unzip libfastcommon-master.zip
    cd libfastcommon-master
    ./make.sh
    ./make.sh   install

注意安装过程中的输出信息，如果没有报错就表示libfastcommon安装成功了。 由于libfastcommon.so默认安装到了/usr/lib64/libfastcommon.so，而FastDFS主程序设置的lib目录是/usr/local/lib，所以需要设置软连接：

    ln -s /usr/lib64/libfastcommon.so /usr/local/lib/libfastcommon.so
    ln -s /usr/lib64/libfastcommon.so /usr/lib/libfastcommon.so
    ln -s /usr/lib64/libfdfsclient.so /usr/local/lib/libfdfsclient.so
    ln -s /usr/lib64/libfdfsclient.so /usr/lib/libfdfsclient.so
至此libfastcommon安装成功了，接下来安装FastDFS

####  安装fastdfs
同样，首先解压

`[root@Frank install]# unzip fastdfs-master.zip`

进入fastdfs-master目录：

    cd fastdfs-master
依次执行下面命令

    ./make.sh  
    ./make.sh install
如果没报错就表示安装成功了。

配置

修改配置文件：
配置tracker
打开tracker.conf

    vim tracker.conf
修改配置文件的这几项（根据数据情况修改）：

    base_path=/home/zq/fastdfs
    bind_addr=192.168.248.130
    启动trackerd服务：
    [plain] view plaincopy
    fdfs_trackerd tracker.conf
通过如下命令查看trackerd服务是否启动：

    netstat -tupln | grep trackerd
输出如下类似的信息表示已经启动了：
tcp0  0 192.168.248.130:22122   0.0.0.0:*   LISTEN  11309/fdfs_trackerd
也可以通过查看日志文件看下有没有出错（/home/zq/fastdfs是前面配置的路径），如果没有报错，应该trackerd服务启动了.

配置client并测试上传
打开client.conf:
vim client.conf
修改配置文件的这几项（根据数据情况修改）：
base_path=/home/zq/fastdfs
tracker_server=192.168.248.130:22122
上传测试：
fdfs_upload_file client.conf client.conf.sample  
将会上传client.conf.sample文件，如果看到类型下面的信息，那么恭喜你，配置成功了：
group1/M00/00/00/wKjHglYshRGAfbrTAAAFtTzeg5c.sample
#### 安装和配置nginx插件
配置fastdfs-nginx-module
##### 1.  解压fastdfs-nginx-module_v1.16.tar.gz
tar -zxvf fastdfs-nginx-module_v1.16.tar.gz  
##### 2. 修改config文件
cd fastdfs-nginx-module/src/
vim config  
修改配置，找到下面这行

    CORE_INCS="$CORE_INCS /usr/local/include/fastdfs /usr/local/include/fastcommon/"
改成

    CORE_INCS="$CORE_INCS /usr/include/fastdfs /usr/include/fastcommon/"
这个是很重要的，不然在nginx编译的时候会报错的，我看网上很多在安装nginx的fastdfs的插件报错，都是这个原因，而不是版本不匹配。
##### 3.修改mod_fastdfs.conf，先复制一份到/etc/fdfs目录下
    cp mod_fastdfs.conf /etc/fdfs/
    cd /etc/fdfs/
    gedit mod_fastdfs.conf
修改如下几项：

    tracker_server=192.168.199.130:22122
    store_path0=/home/zq/fastdfs
    base_path=/home/zq/fastdfs
    url_have_group_name = true（配置多个tracker时，应该将此项设置为true）
##### 4.建立文件服务器的软连接，并做一些需要的操作
建立软连接（配置文件中storage存放数据的路径):

    ln -s /home/zq/fastdfs/data /home/zq/fastdfs/data/M00
    将fastdfs-5.05配置目录下的2个文件复制到/etc/fdfs目录下：
    cp /usr/local/fastdfs-5.05/conf/http.conf .
    cp /usr/local/fastdfs-5.05/conf/mime.types .

#### 安装和配置nginx：


##### 1.安装之前的依赖
PCRE(Perl Compatible Regular Expressions)是一个Perl库，包括 perl 兼容的正则表达式库。nginx的http模块使用pcre来解析正则表达式，所以需要在linux上安装pcre库。

    yum install -y pcre pcre-deve
zlib库提供了很多种压缩和解压缩的方式，nginx使用zlib对http包的内容进行gzip，所以需要在linux上安装zlib库。

    yum install -y zlib zlib-devel
OpenSSL 是一个强大的安全套接字层密码库，囊括主要的密码算法、常用的密钥和证书封装管理功能及SSL协议，并提供丰富的应用程序供测试或其它目的使用。nginx不仅支持http协议，还支持https（即在ssl协议上传输http），所以需要在linux安装openssl库。

    yum install -y openssl openssl-devel

##### 2. 安装nginx-1.8.1.tar.gz
    tar -zxvf nginx-1.8.1.tar.gz

    tar -zxvf nginx-1.8.1.tar.gz
    cd nginx-1.8.1
    configure
    ./configure --help查询详细参数

参数设置如下：

    ./configure \
    --prefix=/usr/local/nginx \
    --pid-path=/var/run/nginx/nginx.pid \
    --lock-path=/var/lock/nginx.lock \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --with-http_gzip_static_module \
    --http-client-body-temp-path=/var/temp/nginx/client \
    --http-proxy-temp-path=/var/temp/nginx/proxy \
    --http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
    --http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
    --http-scgi-temp-path=/var/temp/nginx/scgi

注意：上边将临时文件目录指定为/var/temp/nginx，需要在/var下创建temp及nginx目录

然后依次执行下面的命令：

    make  
    make install

##### 3.配置nginx

打开nginx.conf配置文件
cd /usr/local/nginx/conf
gedit nginx.conf
在server节点加入下面的配置

    location /group1/M00{
    root /usrdata/fastdfs/data;
    ngx_fastdfs_module;
    }


##### 4.启动nginx
/usr/local/nginx/sbin/nginx
用如下命令查看nginx是否启动了：

    netstat -tupln | grep nginx
输出如下（nginx默认监听的是80端口）：
tcp0  0 0.0.0.0:80  0.0.0.0:*   LISTEN  9342/nginx

##### 5.测试配置的nginx
打开浏览器输入:http://127.0.0.1 能看到下面的界面表示安装成功了！

<div align="left"><img src="/img/fastdfs/nginx_success.jpg"  alt="install_package" align=center /><div>
