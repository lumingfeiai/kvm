## 1.虚拟化介绍

virtualization

虚拟化是云计算的基础。简单的说，虚拟化使得在一台物理的服务器上可以跑多台虚拟机，虚拟机共享物理机的 CPU、内存、IO 硬件资源，但逻辑上虚拟机之间是相互隔离的。

物理机我们一般称为**宿主机（Host）**，宿主机上面的虚拟机称为**客户机（Guest）**。

那么 Host 是如何将自己的硬件资源虚拟化，并提供给 Guest 使用的呢？
这个主要是通过一个叫做 Hypervisor 的程序实现的。

根据 Hypervisor 的实现方式和所处的位置，虚拟化又分为两种：

- 全虚拟化（企业级虚拟化）
- 半虚拟化 （桌面级虚拟化）

**全虚拟化：**
Hypervisor 直接安装在物理机上，多个虚拟机在 Hypervisor 上运行。Hypervisor 实现方式一般是一个特殊定制的 Linux 系统。Xen 和 VMWare 的 ESXi 都属于这个类型

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201130161412527.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpX3FpbmdqdW4=,size_16,color_FFFFFF,t_70#pic_center)

**半虚拟化：**
物理机上首先安装常规的操作系统，比如 Redhat、Ubuntu 和 Windows。Hypervisor 作为 OS 上的一个程序模块运行，并对管理虚拟机进行管理。KVM、VirtualBox 和 VMWare Workstation 都属于这个类型

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020113016145740.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpX3FpbmdqdW4=,size_16,color_FFFFFF,t_70#pic_center)

**理论上讲：**
全虚拟化一般对硬件虚拟化功能进行了特别优化，性能上比半虚拟化要高；
半虚拟化因为基于普通的操作系统，会比较灵活，比如支持虚拟机嵌套。嵌套意味着可以在KVM虚拟机中再运行KVM。



## 2.kvm介绍

kVM 全称是 Kernel-Based Virtual Machine。也就是说 KVM 是基于 Linux 内核实现的。
KVM有一个内核模块叫 kvm.ko，只用于管理虚拟 CPU 和内存。

那 IO 的虚拟化，比如存储和网络设备则是由 Linux 内核与Qemu来实现。

作为一个 Hypervisor，KVM 本身只关注虚拟机调度和内存管理这两个方面。IO 外设的任务交给 Linux 内核和 Qemu。

大家在网上看 KVM 相关文章的时候肯定经常会看到 Libvirt 这个东西。

Libvirt 就是 KVM 的管理工具。

其实，Libvirt 除了能管理 KVM 这种 Hypervisor，还能管理 Xen，VirtualBox 等。

Libvirt 包含 3 个东西：后台 daemon 程序 libvirtd、API 库和命令行工具 virsh

- libvirtd是服务程序，接收和处理 API 请求；

- API 库使得其他人可以开发基于 Libvirt 的高级工具，比如 virt-manager，这是个图形化的 KVM 管理工具；

- virsh 是我们经常要用的 KVM 命令行工具

  

## 3. kvm部署

### 3.1 kvm安装

部署前请确保你的CPU虚拟化功能已开启。分为两种情况：

- 虚拟机要关机设置CPU虚拟化
- 物理机要在BIOS里开启CPU虚拟化

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201130161915435.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpX3FpbmdqdW4=,size_16,color_FFFFFF,t_70#pic_center)

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020113016192641.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpX3FpbmdqdW4=,size_16,color_FFFFFF,t_70#pic_center)

```bash
1. 关闭防火墙与SELINUX
[root@localhost ~]#  systemctl stop firewalld
[root@localhost ~]# setenforce 0
setenforce: SELinux is disabled
[root@localhost ~]# reboot

2. 配置网络源

[root@localhost ~]#  wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo

[root@localhost ~]# sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo
[root@localhost ~]# yum install -y https://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
[root@localhost ~]# sed -i 's|^#baseurl=https://download.fedoraproject.org/pub|baseurl=https://mirrors.aliyun.com|' /etc/yum.repos.d/epel*
[root@localhost ~]# sed -i 's|^metalink|#metalink|' /etc/yum.repos.d/epel*
[root@localhost ~]# cd /etc/yum.repos.d/
[root@localhost yum.repos.d]# ls
CentOS-Base.repo   epel-playground.repo  epel-testing-modular.repo  redhat.repo
epel-modular.repo  epel.repo             epel-testing.repo
[root@localhost ~]# yum -y install  vim wget net-tools unzip zip gcc gcc-c++

3. /验证CPU是否支持KVM；如果结果中有vmx（Intel）或svm(AMD)字样，就说明CPU的支持的
[root@localhost ~]# egrep -o 'vmx|svm' /proc/cpuinfo
vmx
vmx
vmx           有几个vmx就说明是几核
vmx
vmx
vmx
vmx
vmx


4. kvm安装
[root@localhost ~]#  yum -y install qemu-kvm qemu-kvm-tools qemu-img virt-manager libvirt libvirt-python libvirt-client virt-install virt-viewer bridge-utils libguestfs-tools            若有些包报错，则安装下面的包

[root@localhost ~]# wget http://mirror.centos.org/centos/7/updates/x86_64/Packages/qemu-kvm-tools-1.5.3-175.el7_9.1.x86_64.rpm
[root@localhost ~]# wget http://mirror.centos.org/centos/7/os/x86_64/Packages/libvirt-python-4.5.0-1.el7.x86_64.rpm
[root@localhost ~]# wget http://mirror.centos.org/centos/7/os/x86_64/Packages/bridge-utils-1.5-9.el7.x86_64.rpm
[root@localhost ~]# rpm -ivh  --nodeps qemu-kvm-tools-1.5.3-175.el7_9.1.x86_64.rpm
[root@localhost ~]# yum -y localinstall bridge-utils-1.5-9.el7.x86_64.rpm 
[root@localhost ~]#  yum -y localinstall libvirt-python-4.5.0-1.el7.x86_64.rpm 


5. 因为虚拟机中网络，我们一般都是和公司的其他服务器是同一个网段，所以我们需要把 \
KVM服务器的网卡配置成桥接模式。这样的话KVM的虚拟机就可以通过该桥接网卡和公司内部 \
其他服务器处于同一网段
[root@localhost ~]# cd /etc/sysconfig/network-scripts/
[root@localhost network-scripts]# ls
ifcfg-ens160
[root@localhost network-scripts]# cp ifcfg-ens160 ifcfg-br0
[root@localhost network-scripts]# ls
ifcfg-br0  ifcfg-ens160
[root@localhost network-scripts]# vim ifcfg-br0 

TYPE="Bridge"
BOOTPROTO="static"
NAME="br0"
DEVICE="br0"
ONBOOT="yes"
NM_CONTROLLED=no                  在8的版本里可以不用写这条命令
IPADDR=192.168.50.156                   虚拟机IP
NETMASK=255.255.255.0
GATEWAY=192.168.50.2
DNS1=114.114.114.114
[root@localhost network-scripts]# vim ifcfg-ens160 

TYPE="Ethernet"
BOOTPROTO="static"
BRIDGE=br0
NAME="ens160"
DEVICE="ens160"
ONBOOT="yes"

[root@localhost ~]# systemctl restart NetworkManager
[root@localhost ~]# ifdown ens160;ifup ens160
Error: '/etc/sysconfig/network-scripts/ifcfg-ens160' is not an active connection.
Error: no active connection provided.
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/3)
[root@localhost ~]# ifdown br0;ifup br0
Connection 'br0' successfully deactivated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/2)
Connection successfully activated (master waiting for slaves) (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/4)
[root@localhost ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br0 state UP group default qlen 1000
    link/ether 00:0c:29:72:93:e6 brd ff:ff:ff:ff:ff:ff
4: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:0c:29:72:93:e6 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.156/24 brd 192.168.50.255 scope global noprefixroute br0
       valid_lft forever preferred_lft forever
    inet6 fe80::bc35:faff:fe61:57d6/64 scope link 
       valid_lft forever preferred_lft forever


6. 启动服务
[root@localhost ~]# systemctl enable --now libvirtd
[root@localhost ~]# systemctl status libvirtd
● libvirtd.service - Virtualization daemon
   Loaded: loaded (/usr/lib/systemd/system/libvirtd.servi>
   Active: active (running) since Mon 2020-11-30 16:49:26>
     Docs: man:libvirtd(8)
           https://libvirt.org
 Main PID: 6015 (libvirtd)
    Tasks: 19 (limit: 32768)
   Memory: 82.0M
   CGroup: /system.slice/libvirtd.service
           ├─6015 /usr/sbin/libvirtd
           ├─6088 /usr/sbin/dnsmasq --conf-file=/var/lib/>
           └─6089 /usr/sbin/dnsmasq --conf-file=/var/lib/>

Nov 30 16:49:26 localhost.localdomain dnsmasq[6079]: list>
Nov 30 16:49:26 localhost.localdomain dnsmasq[6088]: star>
Nov 30 16:49:26 localhost.localdomain dnsmasq[6088]: comp>
Nov 30 16:49:26 localhost.localdomain dnsmasq-dhcp[6088]:>
Nov 30 16:49:26 localhost.localdomain dnsmasq-dhcp[6088]:>
Nov 30 16:49:26 localhost.localdomain dnsmasq[6088]: read>
Nov 30 16:49:26 localhost.localdomain dnsmasq[6088]: usin>
Nov 30 16:49:26 localhost.localdomain dnsmasq[6088]: read>
Nov 30 16:49:26 localhost.localdomain dnsmasq[6088]: read>
Nov 30 16:49:26 localhost.localdomain dnsmasq-dhcp[6088]:>
lines 1-23/23 (END)
[root@localhost ~]# ss -antl
State  Recv-Q Send-Q Local Address:Port   Peer Address:Port                                                   
LISTEN 0      128          0.0.0.0:5355        0.0.0.0:*                                                      
LISTEN 0      128          0.0.0.0:111         0.0.0.0:*                                                      
LISTEN 0      32     192.168.122.1:53          0.0.0.0:*                                                      
LISTEN 0      128          0.0.0.0:22          0.0.0.0:*                                                      
LISTEN 0      128             [::]:5355           [::]:*                                                      
LISTEN 0      128             [::]:111            [::]:*                                                      
LISTEN 0      128             [::]:22             [::]:

7. 验证安装结果
[root@localhost ~]# lsmod|grep kvm
kvm_intel             294912  0
kvm                   786432  1 kvm_intel
irqbypass              16384  1 kvm


8. 测试并验证安装结果
[root@localhost ~]# virsh -c qemu:///system list
 Id    Name                           State
----------------------------------------------------

[root@localhost ~]#  virsh --version
4.5.0
[root@localhost ~]# virt-install --version
2.2.1


9. 查看网桥信息
[root@localhost ~]# nmcli dev
DEVICE      TYPE      STATE      CONNECTION 
br0         bridge    connected  br0        
virbr0      bridge    connected  virbr0     
ens160      ethernet  connected  ens160     
lo          loopback  unmanaged  --         
virbr0-nic  tun       unmanaged  --         
[root@localhost ~]#  nmcli con
NAME    UUID                                  TYPE      DEVICE 
br0     d2d68553-f97e-7549-7a26-b34a26f29318  bridge    br0    
virbr0  4a11f219-20a6-46e7-b5fe-2c4f760e64f5  bridge    virbr0 
ens160  ea74cf24-c2a2-ecee-3747-a2d76d46f93b  ethernet  ens160 


```

### 3.2 kvm web管理界面安装

- kvm 的 web 管理界面是由 webvirtmgr 程序提供的。

```bash
1. 安装依赖包
[root@localhost ~]#  yum -y install git python-pip libvirt-python libxml2-python python-websockify supervisor nginx python2-devel              若有包报错，则安装下面的包

[root@localhost ~]#  wget http://mirror.centos.org/centos/7/os/x86_64/Packages/libxml2-python-2.9.1-6.el7.5.x86_64.rpm
[root@localhost ~]#  wget https://download-ib01.fedoraproject.org/pub/epel/7/x86_64/Packages/p/python-websockify-0.6.0-2.el7.noarch.rpm
[root@localhost ~]# yum -y install python2-devel python2-pip git libvirt-python supervisor nginx 
[root@localhost ~]#  rpm -ivh --nodeps libxml2-python-2.9.1-6.el7.5.x86_64.rpm
[root@localhost ~]# rpm -ivh --nodeps python-websockify-0.6.0-2.el7.noarch.rpm


3. 升级pip
[root@localhost ~]#  pip2 install --upgrade pip
WARNING: Running pip install with root privileges is generally not a good idea. Try `pip3 install --user` instead.
Collecting pip
  Downloading https://files.pythonhosted.org/packages/cb/28/91f26bd088ce8e22169032100d4260614fc3da435025ff389ef1d396a433/pip-20.2.4-py2.py3-none-any.whl (1.5MB)
    0% |▏                               | 10kB 4.3MB/s et
    1% |▍                               | 20kB 674kB/s et
    2% |▋                               | 30kB 112kB/s et
    ......
Installing collected packages: pip
Successfully installed pip-20.2.4


4. 从github上下载webvirtmgr代码
[root@localhost ~]# cd /usr/local/src/
[root@localhost src]# ls
[root@localhost src]#  git clone git://github.com/retspen/webvirtmgr.git
[root@localhost src]# ls
webvirtmgr



5. 安装webvirtmgr
[root@localhost src]# cd webvirtmgr/
[root@localhost webvirtmgr]# pip install -r requirements.txt
Collecting django==1.5.5
  Downloading Django-1.5.5.tar.gz (8.1 MB)
......


6. 检查sqlite3是否安装
[root@localhost webvirtmgr]# python2
Python 2.7.17(default, Dec  5 2019, 15:45:45) 
[GCC 8.3.1 20191121 (Red Hat 8.3.1-5)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import aqlite3                             没有这个东西就会报错，就需要再次装
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ModuleNotFoundError: No module named 'aqlite3'
>>> import sqlite3                                      有这个东西就不会报错
>>> exit()
[root@localhost webvirtmgr]# 


7. 初始化帐号信息
[root@localhost webvirtmgr]# python2 manage.py syncdb
WARNING:root:No local_settings file found.
Creating tables ...
Creating table auth_permission
Creating table auth_group_permissions
Creating table auth_group
Creating table auth_user_groups
Creating table auth_user_user_permissions
Creating table auth_user
Creating table django_content_type
Creating table django_session
Creating table django_site
Creating table servers_compute
Creating table instance_instance
Creating table create_flavor

You just installed Django's auth system, which means you don't have any superusers defined.
Would you like to create one now? (yes/no): yes                          问你是否创建超级管理员帐号
Username (leave blank to use 'root'):    admin             指定超级管理员帐号用户名，默认留空为root
Email address:      1@2.com         设置超级管理员邮箱
Password:                     设置超级管理员密码
Password (again):                再次输入超级管理员密码
Superuser created successfully.
Installing custom SQL ...
Installing indexes ...
Installed 6 object(s) from 1 fixture(s)


7. 拷贝web网页至指定目录
[root@localhost webvirtmgr]#  mkdir /var/www
mkdir: cannot create directory ‘/var/www’: File exists
[root@localhost webvirtmgr]# ls /var/www
cgi-bin  html
[root@localhost webvirtmgr]# cp -r /usr/local/src/webvirtmgr /var/www/
[root@localhost webvirtmgr]#  chown -R nginx.nginx /var/www/webvirtmgr/



8. 生成密钥
[root@localhost ~]#  ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:dzGb7KLIT7HCOnJGVF1p1uAg2M+R8EPX5RV8dMvcc1Y root@localhost.localdomain
The key's randomart image is:
+---[RSA 3072]----+
|     ooooo+= .ooE|
|    . o++++ o.oo*|
|     . oo+. o .==|
|    .   o. . = .o|
|   .    S . =    |
|    ..   + o     |
|   .  o o . .    |
|  . +o + . .     |
|   +..o.o        |
+----[SHA256]-----+
由于这里webvirtmgr和kvm服务部署在同一台机器，所以这里本地信任。如果kvm部署在其他机器，那么这个是它的ip
[root@localhost ~]#  ssh-copy-id 192.168.50.156
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host '192.168.50.156 (192.168.50.156)' can't be established.
ECDSA key fingerprint is SHA256:sQqUajvZSfuHD9T+PkdNt5QwjMJ5RNCws+LTyZAcD54.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@192.168.50.156's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh '192.168.50.156'"
and check to make sure that only the key(s) you wanted were added.



9. 配置端口转发
[root@localhost ~]# ssh 192.168.50.156 -L localhost:8000:localhost:8000 -L localhost:6080:localhost:60
Last login: Mon Nov 30 16:27:34 2020 from 192.168.50.1
[root@localhost ~]# ss -antl
State          Recv-Q         Send-Q                   Local Address:Port                   Peer Address:Port         
LISTEN         0              128                          127.0.0.1:6080                        0.0.0.0:*            
LISTEN         0              128                          127.0.0.1:8000                        0.0.0.0:*            
LISTEN         0              128                            0.0.0.0:5355                        0.0.0.0:*            
LISTEN         0              128                            0.0.0.0:111                         0.0.0.0:*            
LISTEN         0              32                       192.168.122.1:53                          0.0.0.0:*            
LISTEN         0              128                            0.0.0.0:22                          0.0.0.0:*            
LISTEN         0              128                              [::1]:6080                           [::]:*            
LISTEN         0              128                              [::1]:8000                           [::]:*            
LISTEN         0              128                               [::]:5355                           [::]:*            
LISTEN         0              128                               [::]:111                            [::]:*            
LISTEN         0              128                               [::]:22                             [::]:*            


 10. 配置nginx
[root@localhost ~]# cat > /etc/nginx/nginx.conf <<'EOF'
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80;
        server_name  localhost;

        include /etc/nginx/default.d/*.conf;

        location / {
            root html;
            index index.html index.htm;
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
}

[root@localhost ~]# cat > webvirtmgr.conf <<'EOF'

server {
    listen 80 default_server;

    server_name $hostname;
    #access_log /var/log/nginx/webvirtmgr_access_log;

    location /static/ {
        root /var/www/webvirtmgr/webvirtmgr;
        expires max;
    }

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-for $proxy_add_x_forwarded_for;
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Forwarded-Proto $remote_addr;
        proxy_connect_timeout 600;
        proxy_read_timeout 600;
        proxy_send_timeout 600;
        client_max_body_size 1024M;
    }
}



11. 确保bind绑定的是本机的8000端口
[root@localhost ~]# vim /var/www/webvirtmgr/conf/gunicorn.conf.py
......
bind = '0.0.0.0:8000'
backlog = 2048
......



12.  重启nginx
[root@localhost ~]# systemctl enable --now nginx
Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /usr/lib/systemd/system/nginx.service.
[root@localhost ~]# ss -antl
State    Recv-Q   Send-Q     Local Address:Port                    Peer Address:Port                
LISTEN   0        128            127.0.0.1:6080                         0.0.0.0:*                   
LISTEN   0        128            127.0.0.1:8000                         0.0.0.0:*                   
LISTEN   0        128              0.0.0.0:5355                         0.0.0.0:*                   
LISTEN   0        128              0.0.0.0:111                          0.0.0.0:*                   
LISTEN   0        128              0.0.0.0:80                           0.0.0.0:*                   
LISTEN   0        32         192.168.122.1:53                           0.0.0.0:*                   
LISTEN   0        128              0.0.0.0:22                           0.0.0.0:*                   
LISTEN   0        128                [::1]:6080                            [::]:*                   
LISTEN   0        128                [::1]:8000                            [::]:*                   
LISTEN   0        128                 [::]:5355                            [::]:*                   
LISTEN   0        128                 [::]:111                             [::]:*                   
LISTEN   0        128                 [::]:22                              [::]:*                   


13. 设置supervisor
[root@localhost ~]# cat >> /etc/supervisord.conf <<EOF
......                           文件最后面添加以下内容
[program:webvirtmgr]
command=/usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.conf.py
directory=/var/www/webvirtmgr
autostart=true
autorestart=true
logfile=/var/log/supervisor/webvirtmgr.log
log_stderr=true
user=nginx

[program:webvirtmgr-console]
command=/usr/bin/python2 /var/www/webvirtmgr/console/webvirtmgr-console
directory=/var/www/webvirtmgr
autostart=true
autorestart=true
stdout_logfile=/var/log/supervisor/webvirtmgr-console.log
redirect_stderr=true
user=nginx



14. 启动supervisor并设置开机自启
[root@localhost ~]# systemctl enable --now supervisord
Created symlink /etc/systemd/system/multi-user.target.wants/supervisord.service → /usr/lib/systemd/system/supervisord.service.
[root@localhost ~]# systemctl status supervisord
● supervisord.service - Process Monitoring and Control Daemon
   Loaded: loaded (/usr/lib/systemd/system/supervisord.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2020-11-30 21:01:37 CST; 58s ago
  Process: 10082 ExecStart=/usr/bin/supervisord -c /etc/supervisord.conf (code=exited, status=0/SUCCESS)
 Main PID: 10085 (supervisord)
    Tasks: 19 (limit: 50621)
   Memory: 255.8M
   CGroup: /system.slice/supervisord.service
           ├─10085 /usr/bin/python3.6 /usr/bin/supervisord -c /etc/supervisord.conf
           ├─10086 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
           ├─10091 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
           ├─10092 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
           ├─10093 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
           ├─10094 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
           ├─10095 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
           ├─10096 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
           ├─10097 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
           ├─10098 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
           ├─10099 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
           ├─10100 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
           ├─10101 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
           ├─10102 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
           ├─10104 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
           ├─10105 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
           ├─10106 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
           ├─10107 /usr/bin/python2 /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.c>
lines 1-26




15. 配置nginx用户
[root@localhost ~]# su - nginx -s /bin/bash
[nginx@localhost ~]$ id
uid=989(nginx) gid=988(nginx) groups=988(nginx)
[nginx@localhost ~]$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/var/lib/nginx/.ssh/id_rsa): 
Created directory '/var/lib/nginx/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /var/lib/nginx/.ssh/id_rsa.
Your public key has been saved in /var/lib/nginx/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:Q8tfrvMNQEskohXenMPf2f1tkDeg7iXDWZQ6h3mi0kU nginx@localhost.localdomain
The key's randomart image is:
+---[RSA 3072]----+
|      +.. .      |
|     + = +    .  |
|    . . B oE +   |
|       o *.o*o.o |
|        S +O+++.o|
|        .o=+B  o+|
|       . o.*o.  +|
|        . o.+o . |
|          .+. .  |
+----[SHA256]-----+
[nginx@localhost ~]$ touch ~/.ssh/config
[nginx@localhost ~]$ echo -e "StrictHostKeyChecking=no\nUserKnownHostsFile=/dev/null" >> ~/.ssh/config
[nginx@localhost ~]$ cat ~/.ssh/config 
StrictHostKeyChecking=no
UserKnownHostsFile=/dev/null
[nginx@localhost ~]$ chmod 0600 ~/.ssh/config
[nginx@localhost ~]$ ssh-copy-id root@192.168.50.156
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/var/lib/nginx/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
Warning: Permanently added '192.168.50.156' (ECDSA) to the list of known hosts.
root@192.168.50.156's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@192.168.50.156'"
and check to make sure that only the key(s) you wanted were added.
[nginx@localhost ~]$ exit
logout



16. 创建一个配置文件并写入东西
[root@localhost ~]#  vim /etc/polkit-1/localauthority/50-local.d/50-libvirt-remote-access.pkla
[Remote libvirt SSH access]
Identity=unix-user:root
Action=org.libvirt.unix.manage
ResultAny=yes
ResultInactive=yes
ResultActive=yes



17. 修改属主
[root@localhost ~]# chown -R root.root /etc/polkit-1/localauthority/50-local.d/50-libvirt-remote-access.pkla



18. 重启服务，关闭防火墙
[root@localhost ~]# systemctl restart nginx
[root@localhost ~]# systemctl restart libvirtd
[root@localhost ~]# systemctl stop firewalld



```

### 3.3 kvm web界面管理

- 通过ip地址在浏览器上访问kvm，例如我这里就是：192.168.50.156

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020120122395657.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpX3FpbmdqdW4=,size_16,color_FFFFFF,t_70#pic_center)

#### 3.3.1 kvm连接管理

**创建SSH连接：**

![image-20240612174550041](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612174550041.png)

![image-20240612174613157](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612174613157.png)

![image-20240612174704047](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612174704047.png)

#### 3.3.2 kvm存储管理

**创建存储：**

![image-20240612174740637](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612174740637.png)

- 创建目录

```bash
[root@localhost ~]# ls /var/lib/libvirt/images/
[root@localhost ~]# df -h
Filesystem             Size  Used Avail Use% Mounted on
devtmpfs               963M     0  963M   0% /dev
tmpfs                  981M     0  981M   0% /dev/shm
tmpfs                  981M  8.9M  972M   1% /run
tmpfs                  981M     0  981M   0% /sys/fs/cgroup
/dev/mapper/rhel-root   17G  2.6G   15G  15% /
/dev/nvme0n1p1        1014M  161M  854M  16% /boot
tmpfs                  196M     0  196M   0% /run/user/0
[root@localhost ~]# mkdir /virtual_host

```

![image-20240612174833680](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612174833680.png)

![image-20240612174856501](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612174856501.png)

- 关机加硬盘

![image-20240612191340797](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612191340797.png)

![image-20240612191416525](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612191416525.png)

<img src="C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612191444817.png" alt="image-20240612191444817" style="zoom:200%;" />

![image-20240612191624049](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612191624049.png)

- 分区，格式化，并挂载

```bash
[root@localhost ~]# lsblk
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sr0            11:0    1  7.9G  0 rom  
nvme0n1       259:0    0   20G  0 disk 
├─nvme0n1p1   259:1    0    1G  0 part /boot
└─nvme0n1p2   259:2    0   19G  0 part 
  ├─rhel-root 253:0    0   17G  0 lvm  /
  └─rhel-swap 253:1    0    2G  0 lvm  [SWAP]
nvme0n2       259:3    0  200G  0 disk 
[root@localhost ~]# ls /virtual_host/
[root@localhost ~]# fdisk /dev/nvme0n2

Welcome to fdisk (util-linux 2.32.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x749c27d5.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 
First sector (2048-419430399, default 2048): 
Last sector, +sectors or +size{K,M,G,T,P} (2048-419430399, default 419430399): 

Created a new partition 1 of type 'Linux' and of size 200 GiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

[root@localhost ~]# partprobe
Warning: Unable to open /dev/sr0 read-write (Read-only file system).  /dev/sr0 has been opened read-only.
[root@localhost ~]# lsblk
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sr0            11:0    1  7.9G  0 rom  
nvme0n1       259:0    0   20G  0 disk 
├─nvme0n1p1   259:1    0    1G  0 part /boot
└─nvme0n1p2   259:2    0   19G  0 part 
  ├─rhel-root 253:0    0   17G  0 lvm  /
  └─rhel-swap 253:1    0    2G  0 lvm  [SWAP]
nvme0n2       259:3    0  200G  0 disk 
└─nvme0n2p1   259:5    0  200G  0 part 
[root@localhost ~]# mkfs.xfs /dev/nvme0n2p1
meta-data=/dev/nvme0n2p1         isize=512    agcount=4, agsize=13107136 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=52428544, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=25599, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

[root@localhost ~]# blkid
/dev/nvme0n1: PTUUID="fd9e4a5a" PTTYPE="dos"
/dev/nvme0n1p1: UUID="339d86d6-edac-49ff-9abe-3aa194a9051d" TYPE="xfs" PARTUUID="fd9e4a5a-01"
/dev/nvme0n1p2: UUID="kuI3zy-HfI6-YQ7e-u6ks-3V4s-nxR9-Aebqde" TYPE="LVM2_member" PARTUUID="fd9e4a5a-02"
/dev/nvme0n2: PTUUID="749c27d5" PTTYPE="dos"
/dev/nvme0n2p1: UUID="2d8d33cd-472d-477f-8943-caf38754d252" TYPE="xfs" PARTUUID="749c27d5-01"
/dev/sr0: UUID="2020-04-04-08-21-15-00" LABEL="RHEL-8-2-0-BaseOS-x86_64" TYPE="iso9660" PTUUID="47055c33" PTTYPE="dos"
/dev/mapper/rhel-root: UUID="904f3641-9768-4b89-9268-71937eb86178" TYPE="xfs"
/dev/mapper/rhel-swap: UUID="fff63d51-b23a-4e77-a386-3ee9cedf339d" TYPE="swap"
[root@localhost ~]# vim /etc/fstab
添加此内容
UUID="2d8d33cd-472d-477f-8943-caf38754d252"  /virtual_host   xfs    defaults    0 0



[root@localhost ~]# mount -a
[root@localhost ~]# df -h
Filesystem             Size  Used Avail Use% Mounted on
devtmpfs               963M     0  963M   0% /dev
tmpfs                  981M     0  981M   0% /dev/shm
tmpfs                  981M  8.9M  972M   1% /run
tmpfs                  981M     0  981M   0% /sys/fs/cgroup
/dev/mapper/rhel-root   17G  2.6G   15G  15% /
/dev/nvme0n1p1        1014M  161M  854M  16% /boot
tmpfs                  196M     0  196M   0% /run/user/0
/dev/nvme0n2p1         200G  1.5G  199G   1% /virtual_host


```

![image-20240612191722141](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612191722141.png)

- 通过远程连接软件上传ISO镜像文件至存储目录/var/lib/libvirt/images/

```bash
[root@localhost ~]# ls /virtual_host
rhel-8.2-x86_64-dvd.iso

```

- 在 web 界面查看ISO镜像是否存在

![image-20240612191831359](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612191831359.png)

- 创建系统安装镜像

![image-20240612191905631](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612191905631.png)

![image-20240612191921763](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612191921763.png)

![image-20240612191936264](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612191936264.png)

#### 3.3.3 kvm网络管理

- 添加桥接网络

![image-20240612192025386](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612192025386.png)

![image-20240612192037311](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612192037311.png)

![image-20240612192053662](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612192053662.png)

![image-20240612192104932](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612192104932.png)

#### 3.3.4 实例管理

**实例(虚拟机)创建:**

![image-20240612192134288](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612192134288.png)

![image-20240612192149396](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612192149396.png)

![image-20240612192200347](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612192200347.png)

- 虚拟机插入光盘

![image-20240612192235622](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612192235622.png)

- 启动虚拟机

![image-20240612192301926](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612192301926.png)

![image-20240612192320633](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612192320633.png)

![image-20240612192340924](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612192340924.png)

![image-20240612192410585](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612192410585.png)

- 虚拟机安装
  - 虚拟机安装步骤就是安装系统的步骤，此处就不再赘述

![image-20240612192432926](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612192432926.png)

![image-20240612192444591](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612192444591.png)

## 4. 案例

### 4.1 案例1

web界面配置完成后可能会出现以下错误界面![在这里插入图片描述](https://img-blog.csdnimg.cn/20201202151248869.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpX3FpbmdqdW4=,size_16,color_FFFFFF,t_70#pic_center)

- web界面配置完成后可能会出现以下错误界面

```bash
[root@KVM ~]# wget https://download-ib01.fedoraproject.org/pub/epel/7/x86_64/Packages/n/novnc-0.5.1-2.el7.noarch.rpm
[root@KVM ~]# yum -y install novnc-0.5.1-2.el7.noarch.rpm
[root@KVM ~]# ll /etc/rc.local
lrwxrwxrwx 1 root root 13 Jul 21 22:57 /etc/rc.local -> rc.d/rc.local
[root@KVM ~]# ll /etc/rc.d/rc.local
-rw-r--r--. 1 root root 474 Jul 21 22:57 /etc/rc.d/rc.local
[root@KVM ~]# chmod +x /etc/rc.d/rc.local
[root@KVM ~]# vim /etc/rc.d/rc.local				#在此文件末尾追加下面的内容
...
nohup novnc_server 192.168.50.156:5920 &
[root@KVM ~]# . /etc/rc.d/rc.local
[root@KVM ~]# nohup: ignoring input and appending output to 'nohup.out'


```

- 做完以上操作后再次访问即可正常访问

![image-20240612192625550](C:\Users\涂祖函\AppData\Roaming\Typora\typora-user-images\image-20240612192625550.png)

### 4.2 案例2

- 第一次通过web访问kvm时可能会一直访问不了，一直转圈，而命令行界面一直报错(too many open files)

```bash
[root@localhost ~]# vim /etc/nginx/nginx.conf

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

worker_rlimit_nofile 655350;         添加此行

include /usr/share/nginx/modules/*.conf;




[root@localhost ~]# vim /etc/security/limits.conf
......
# End of file
* soft nofile 655350           添加这两行
* hard nofile 655350


[root@localhost ~]# systemctl restart nginx


```

## 5.基础命令

### 1.创建客户机

**运行 `virt-install` 命令**： 使用 `virt-install` 命令来创建和配置新的虚拟机。以下是一个创建虚拟机的基本示例命令：

```bash
virt-install \
--name vm3 \
--ram 2048 \
--vcpus 2 \
--disk path=/var/lib/libvirt/images/vm3.img,size=10 \
--os-type linux \
--os-variant centos7.0 \
--network type=direct,source=ens33,source_mode=private,model=virtio,mac=52:54:00:24:68:43 \
--graphics vnc \
--cdrom /CentOS-7-x86_64-Minimal-2009.iso
```

这个命令的参数解释如下：

- `--name vm_name`: 虚拟机名称。
- `--ram 2048`: 分配给虚拟机的内存（MB）。
- `--vcpus 2`: 分配给虚拟机的虚拟CPU数量。
- `--disk path=/var/lib/libvirt/images/vm_name.img,size=20`: 创建一个20GB的虚拟磁盘文件。
- `--os-type linux`: 操作系统类型。
- `--os-variant ubuntu20.04`: 操作系统版本（使用 `osinfo-query os` 可以查看支持的操作系统版本）。
- `--network network=default`: 使用默认的虚拟网络。
- `--graphics vnc`: 使用VNC作为图形显示方式。
- `--cdrom /path/to/installation.iso`: 安装介质的路径

### 2.控制命令

```bash
1.启动客户机
varsh start vm1
2.重启客户机
varsh reboot vm1
3.关闭
varsh shutdown vm1
virsh destroy vm1 #强制关闭，不建议用
4.暂停挂起
varsh suspend vm1
5.查看客户机的资源使用情况
varsh domstats vm1
6.编辑客户机配置文件
varsh edit vm1 #不适合自动化
vim /etc/libvirt/qemu/vm1.xml  #修改后要 varsh define 文件生效
7.查看所有客户机状态
virsh list --all
8.查看客户机的详细信息：
virsh dominfo vm1
9.查看所有开机自启的客户机
ls /etc/libvirt/qemu/autostart/ 
virsh list --all --autostart
10.删除客户机
virsh undefine vm2 #记得先关机
11.删除客户机和磁盘镜像文件
virsh undefine vm2 --remove-all-storage
```



### 3.快照

1.创建vm1客户机快照

```bash
virsh snapshot-create-as vm1 #快照名自动生成
virsh snapshot-create-as vm1 vm1-kz1  #快照名为vm1-kz1
```

2.给vm1客户机恢复快照

```bash
virsh shutdown vm8
virsh snapshot-revert vm1 vm1-kz1
```

3.查看vm1客户机快照

```bash
virsh snapshot-list vm1
```

4.删除vm1客户机快照

```bash
virsh snapshot-delete --snapshotname vm1-kz1 vm1
```



### 4.克隆

1.克隆vm1客户机

```bash
virt-clone -o vm1 -n vm2 --auto-clone  #新客户机vm2
```

2.克隆客户机并指定新的磁盘镜像文件路径和名称

```shell
virt-clone -o vm1 -n vm2 -f /var/lib/libvirt/images/vm2.img 
```



### 5.磁盘镜像文件

**1.查看镜像文件格式**：

```bash
qemu-img info /var/lib/libvirt/images/vm1.img
```

2.磁盘镜像文件格式

**raw**

使用文件来模拟实际的硬盘(当然也可以使用一块真实的硬盘或一个分区)。由于原生的裸格式，不支持snapshot也是很正常的。但如果你使用LVM的裸设备，那就另当别论。说到LVM还是十分的犀利的目前来LVM的snapshot、性能、可扩展性方面都还是有相当的效果的。目前来看备份也问题不大。就是在虚拟机迁移方面还是有很大的限制。但目前虚拟化的现状来看，真正需要热迁移的情况目前需求还不是是否的强烈。虽然使用LVM做虚拟机镜像的相关公开资料比较少，但目前来看牺牲一点灵活性，换取性能和便于管理还是不错的选择。

**qcow2**

现在比较主流的一种虚拟化镜像格式，经过一代的优化，目前qcow2的性能上接近raw裸格式的性能，这个也算是redhat的官方渠道了

对于qcow2的格式，几点还是比较突出的，qcow2的snapshot，可以在镜像上做N多个快照：

 •更小的存储空间

 •Copy-on-write support

 •支持多个snapshot，对历史snapshot进行管理

 •支持zlib的磁盘压缩

 •支持AES的加密

**2.格式转换把raw格式转换成qcow2格式**

```bash
qemu-img convert -f raw -O qcow2  /var/lib/libvirt/images/vm2.img /var/lib/libvirt/images/vm2_qcow2.img
```

**3.创建磁盘**

```bash
qemu-img create -f qcow2 /var/lib/libvirt/images/vm_disk.qcow2 20G
```

**4.将新创建的磁盘附加到现有的虚拟机**

```bash
virsh attach-disk <虚拟机名称> /var/lib/libvirt/images/vm_disk.qcow2 vdb --persistent
4.1 卸载磁盘
virsh detach-disk <虚拟机名称> /var/lib/libvirt/images/vm_disk.qcow2 vdb --persistent
```

**5.查看虚拟机磁盘信息**：

```bash
virsh domblklist <虚拟机名称>
```

**6.查看单个磁盘信息：**

```bash
qemu-img info /var/lib/libvirt/images/vm_disk.qcow2
```

#### **1.扩展源磁盘镜像文件的大小**

**安全扩展分区和文件系统步骤**

##### 步骤 1：备份重要数据

在进行任何磁盘操作之前，首先备份虚拟机磁盘或重要数据。你可以使用 `virsh` 命令导出虚拟机磁盘镜像：

```bash
virsh shutdown <虚拟机名称>
virsh dumpxml <虚拟机名称> > /tmp/<虚拟机名称>.xml
cp /var/lib/libvirt/images/vm4.qcow2 /var/lib/libvirt/images/vm4_backup.qcow2
```

##### 步骤 2：扩展磁盘镜像文件的大小

使用 `qemu-img resize` 命令扩展磁盘镜像文件的大小：

```
qemu-img resize /var/lib/libvirt/images/vm4.qcow2 100G
```

##### 步骤 3：调整分区和文件系统大小

使用 `virt-resize` 工具来调整分区和文件系统大小，而不丢失现有数据。

**1.安装 `libguestfs-tools`：**

```bash
yum install libguestfs-tools
```

**2.创建一个新的更大的磁盘镜像**：

```bash
qemu-img create -f qcow2 /var/lib/libvirt/images/vm4_resized.qcow2 100G
```

**3.使用 `virt-resize` 调整分区大小**

```
virt-resize --expand /dev/sda /var/lib/libvirt/images/vm4.qcow2 /var/lib/libvirt/images/vm4_resized.qcow2
```

- `--expand /dev/sda ：指定要扩展的分区（假设根分区是 `/dev/sda`，请根据实际情况调整）。
- `/var/lib/libvirt/images/vm4.qcow2`：源磁盘镜像。
- `/var/lib/libvirt/images/vm4_resized.qcow2`：目标磁盘镜像。

**4.替换旧的磁盘镜像**：

```bash
mv /var/lib/libvirt/images/vm4.qcow2 /var/lib/libvirt/images/vm_disk_old.qcow2 #备份
mv /var/lib/libvirt/images/vm4_resized.qcow2 /var/lib/libvirt/images/vm4.qcow2 #替换
```

**直接在虚拟机内部调整分区和文件系统大小（备用方法）**

执行完上述步骤2后

1.登录到虚拟机

2.使用 `fdisk` 调整分区大小

```bash
fdisk /dev/vda
1.删除现有分区（确保不要格式化分区）。
2.创建一个新的分区，使用所有可用空间。
3.写入更改并退出
```

3.使用 `resize2fs` 调整文件系统大小：

```bash
resize2fs /dev/vda1
```

4.验证文件系统大小：

```shell
df -h
```



## 6.KVM存储

在Libvirt中，存储池是用于组织和管理存储资源的抽象。存储池可以有多种类型，其中常见的包括目录存储池和逻辑卷存储池。下面是这两种存储池的详细介绍及操作步骤。

#### 1.目录存储池（Directory Storage Pool）

目录存储池是最简单的一种存储池类型。它使用文件系统中的一个目录来存放虚拟机磁盘镜像文件。

**1.创建存储目录：**

```bash
 mkdir -p /var/lib/libvirt/vmfs
```

**2.定义存储池**：两种写法

```bash
virsh pool-define-as vmdisk dir - - - - "/var/lib/libvirt/vmfs"
virsh pool-define-as vmdisk --type dir --target /var/lib/libvirt/vmfs
```

**3.构建存储池**：

```bash
virsh pool-build vmdisk
```

**4.启动存储池**：

```
virsh pool-start vmdisk
```

**5.设置存储池自动启动**

```
virsh pool-autostart vmdisk
```

**6.创建存储卷**

存储卷就是磁盘镜像文件，用卷管理命令不需要知道存储池路径,只需要存储池名字

```bash
virsh vol-create-as vmdisk vm-1.qcow2 10G --format qcow2
```

#### 2.创建逻辑卷存储池的步骤

假设我们有一个卷组`vg_data`，我们要基于此卷组创建一个名为`lvmpool`的逻辑卷存储池。这里`lvmpool`是存储池的名称，而`vg_data`是卷组的名称

**1.定义存储池**

```bash
virsh pool-define-as lvmpool logical - - - - "vg_data"
```

后续跟目录步骤一样把vmdisk换成lvmpool

#### 3.查看存储池和存储卷

**1.查看所有存储池**：

```bash
virsh pool-list --all
```

**2.查看存储池详细信息**：

```
virsh pool-info vmdisk
```

**3.查看存储池中的存储卷**

```bash
virsh vol-list vmdisk
```

