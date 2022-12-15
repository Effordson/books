---
创建时间: 222022-12-14 00:12:41
tags:
- Linux
- FTP
- obsidian
- picgo
---



# 1 搭建ftp服务器

## 1.1 安装vsftpd软件
切换到root用户
安装vsftpd：
```
apt install vsftpd

# vsftpd 是very secure FTP daemon的缩写
```

## 1.2 启动查看
查看版本：
```
root@starpc:~# dpkg -s vsftpd
Package: vsftpd
Status: install ok installed
Priority: extra
Section: net
Installed-Size: 328
Maintainer: Ubuntu Developers <ubuntu-devel-discuss@lists.ubuntu.com>
Architecture: amd64
Version: 3.0.3-3ubuntu2
Replaces: ftp-server
Provides: ftp-server
```

查看状态：service vsftpd status | systemctl status vsftpd.service -- 安装后自动启动
查看进程：ps -ef |grep vsftpd
查看端口：netstat -ntlp |grep vsftpd

默认配置：
```
root@starpc:/etc# grep -Ev "^$|^#" vsftpd.conf 
listen=NO
listen_ipv6=YES
anonymous_enable=NO
local_enable=YES
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
rsa_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
rsa_private_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
ssl_enable=NO
```

预设用户：writer 读写，reader 只读，创建用户：
```
useradd -m reader
passwd reader
```

## 1.3 登录ftp服务
默认是没有权限查看的：
```
[root@m01 ~]# lftp -u writer 192.168.5.203 
Password: 
lftp writer@192.168.5.203:~> ls       
ls: Login failed: 530 Login incorrect.  
```
此错误需要注释默认/etc/pam.d/vsftpd配置最后一行：
```
# auth  required        pam_shells.so
```

错误：
```
[root@m01 ~]# lftp -u writer 192.168.5.203 
Password: 
lftp writer@192.168.5.203:~> ls       
ls: Login failed: 500 OOPS: vsftpd: refusing to run with writable root inside chroot()

解决方法1：开启写入权限
# 全局启用写入功能
write_enable=YES
# 添加家目录写入权限
allow_writeable_chroot=YES

解决方法2：关闭写入权限
家目录去掉写入权限 chmod u-w /home/xx
```


**注意：** vsftpd 使用的用户名和密码就是linux系统里面的用户和密码

ftp常用命令：lftp writer@192.168.5.203:~> help

本地目录切换：lcd
查看本地目录：!ls
获取文件：`get [OPTS] <rfile> [-o <lfile>]`
上传文件：`put [OPTS] <lfile> [-o <rfile>]`
获取多个文件：`mget [OPTS] <files>`
下载文件夹到本地：`mirror [OPTS] [remote [local]]`
创建目录：`mkdir [-p] <dirs>`
删除目录：`rmdir [-f] <dirs>`
修改属性：`chmod [OPTS] mode file...`
查看文件内容：`cat [-b] <files>`


## 1.4 配置文件
默认有3个配置文件：
/etc/ftpusers：ftp黑名单，如果想让一个用户登录不了ftp服务，可以将用户加入黑名单
/etc/vsftpd.conf：服务配置文件
/etc/pam.d/vsftpd：权限认证配置文件

开启所有用户家目录上传下载文件(命令行操作有效)：
```
# 全局启用写入功能
write_enable=YES
# 添加家目录写入权限
allow_writeable_chroot=YES
# 设置UMASK
local_umask=022
# 只允许在家目录中，不能跳出家目录
chroot_local_user=YES
```


黑名单或白名单：**userlist_deny=NO** 白名单，**userlist_deny=YES** 黑名单，相关参数
```
#对本地用户限制在自己的家目录里
chroot_local_user=YES  
#启用限制名单
chroot_list_enable=YES
# 具体的名单路径，这个名单的用户不受限制，可以随意切换目录
chroot_list_file=/etc/vsftpd.chroot_list

#这个文件没有，需要自己新建。
root@starpc:/etc# cat /etc/vsftpd.chroot_list
writer
```


## 1.5 用户可读可写
注意：注释默认/etc/pam.d/vsftpd配置最后一行

/etc/vsftpd.conf配置：
```
# 默认配置
listen=NO
listen_ipv6=YES
anonymous_enable=NO
local_enable=YES
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
rsa_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
rsa_private_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
ssl_enable=NO

# 全局启用写入功能
write_enable=YES
# 添加家目录写入权限
allow_writeable_chroot=YES
# 设置UMASK
local_umask=022
# 只允许在家目录中，不能跳出
chroot_local_user=YES
# 开启跳出家目录功能，哪些用户可以跳出，由chroot_list文件配置
chroot_list_enable=YES
chroot_list_file=/etc/vsftpd.chroot_list
```

## 1.6 匿名用户可读
注意：注释配置文件/etc/pam.d/vsftpd最后一行

修改上面的/etc/vsftpd.conf配置：
```
anonymous_enable=YES
anon_umask=022
# 匿名默认发布和访问目录
anon_root=/var/ftp
```

## 1.7 匿名可读可写
匿名用户相关配置：
```
# 匿名用户访问
anonymous_enable=YES
# 匿名用户umask
anon_umask=022
# 匿名默认发布目录
anon_root=/var/ftp
# 允许匿名用户上传文件
anon_upload_enable=YES
# 允许匿名用户删除重命名
anon_other_write_enable=YES
# 允许匿名用户建立、删除目录
anon_mkdir_write_enable=YES
# 允许匿名用户下载文件
anon_world_readable_only=NO
# 登陆用户数量控制
max_clients=xx
# 上传文件速率控制，设置最大上传速率为102400（默认单位为KB)即100K
anon_max_rate=xxxx
```

参考文档：
https://blog.csdn.net/weixin_44310047/article/details/117119963

## 1.8 报错代码
除上述基本信息之外，在我们访问FTP服务的过程中，系统会提示报错信息，其对应含义如下

| 报错  | 含义       |
|-----|----------|
| 550 | 服务程序本身拒绝 |
| 553 | 文件系统权限限制 |
| 500 | 文件系统权限过大 |
| 530 | 认证失败     |

## 1.9 使用windows向linux传文件
使用资源浏览器访问ftp服务器，CTRL+E打开文件管理器，输入：`ftp://192.168.5.203/`
登录账户，只能进行下载，不能上传，目录/var/ftp只有root(属主)有权限写入修改文件。或修改权限writer用户可以写入
tips：可以通过修改权限775(777会报权限问题)，需要写入的用户加入属组即可


# 2 GitHub图床-obsidian
1、下载安装,当前下载的是2.3.1版本
https://github.com/Molunerfinn/PicGo

2、创建仓库分支、token
仓库：books
分支：picgo
token(永久有效,名称：picgo-img-upload )：`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`

3、picgo GitHub图床配置
仓库名(用户名/仓库名)、分支名、Token、存储路径(img/，路径斜杠不能少)

4、obsidian安装picgo插件，默认配置即可

5、上传测试
![image.png](https://raw.githubusercontent.com/Effordson/books/picgo/img/20221215002642.png)

# 3 本地图床-FTP搭配Nginx-obsidian
## 3.1 FTP服务
1、安装FTP服务
2、修改配置文件/etc/vsftpd.conf：
```
# 默认配置
listen=NO
listen_ipv6=YES
# 匿名用户开启访问权限
anonymous_enable=YES
local_enable=YES
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
rsa_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
rsa_private_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
ssl_enable=NO

# 全局启用写入功能
write_enable=YES
# 添加家目录写入权限
allow_writeable_chroot=YES
# 设置UMASK
local_umask=022
# 只允许在家目录中，不能跳出
chroot_local_user=YES
# 开启跳出家目录功能，哪些用户可以跳出，由chroot_list文件配置
chroot_list_enable=YES
chroot_list_file=/etc/vsftpd.chroot_list

# 匿名默认发布和访问目录
anon_root=/var/ftp
```

3、注释默认配置文件/etc/pam.d/vsftpd最后一行
4、任意目录访问用户为writer，并写入文件/etc/vsftpd.chroot_list
5、创建用户writer
6、创建目录并修改权限：
```
mkdir /var/ftp
chgrp writer /var/ftp/
chmod 775 /var/ftp/
```

7、重启FTP服务：systemctl restart vsftpd.service
8、测试权限：
```
上传权限：lftp -u writer 192.168.5.203
匿名访问权限：ftp://192.168.5.203/
```

## 3.2 PicGo安装配置
1、下载安装,当前下载的是2.3.1版本
https://github.com/Molunerfinn/PicGo

2、安装插件：ftp-uploader,当前版本1.1.3

3、ftp插件配置：
网站标识：`imba97`
配置文件：`C:/tmp/ftpUploader.json`
ftpUploader.json文件内容：
```json
{
  "imba97": {
    "url": "http://192.168.5.203:8086",
    "path": "/img/{year}/{month}/{fullName}",
    "uploadPath": "/var/ftp/img/{year}/{month}/{fullName}",
    "host": "192.168.5.203",
    "port": 21,
    "username": "writer",
    "password": "123456"
  },
  "btools": {
    "url": "https://btools.cc",
    "path": "/btools_cc/{year}/{month}/{fullName}",
    "uploadPath": "/Web/btools_cc/{year}/{month}/{fullName}",
    "host": "1.2.3.4",
    "port": 21,
    "username": "ftpUser2",
    "password": "ftpPassword2"
  }
}
```

## 3.3 Nginx离线安装配置
1、下载1.8.1版本安装包：http://nginx.org/en/download.html 或者 https://github.com/nginx/nginx/

2、解压安装：
```
./configure --prefix=/opt/nginx-1.8.1 --without-http_rewrite_module --without-http_gzip_module
make && make install 
```

3、加入环境变量/etc/bash.bashrc：
```
export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH:/opt/nginx-1.8.1/sbin
```

4、启动：nginx ; 停止nginx -s stop；重启 nginx -s reload

5、修改配置文件/opt/nginx-1.8.1/conf/nginx.conf，修改http区域中server配置：
```
    server {
        listen       8087;
        server_name  localhost;

        location / {
            root /var/ftp;
            charset utf-8,gbk;
            autoindex on;
            autoindex_localtime on;
            autoindex_exact_size off;
            # root   html;
            # index  index.html index.htm;
        }
    }
```

注意：/var/ftp目录不能有index.html文件

## 3.4 obsidian插件安装配置
1、安装三方插件：Image auto upload Plugin
2、默认配置即可
3、截图粘贴即可自动上传

