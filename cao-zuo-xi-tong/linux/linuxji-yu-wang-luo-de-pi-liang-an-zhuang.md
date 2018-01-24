## 准备：

DHCP服务器

TFTP服务器

FTP服务器

Kickstart文件

拥有PXE ROM芯片，支持网络启动的客户端，即要安装系统的裸机

Linux镜像文件

这里用一台主机同时提供DHCP,TFTP,FTP三种服务，kickstart也放在这台服务器上。

## 原理

远程客户端计算机启动，由于BIOS设置了网卡启动，所以网卡PXE ROM中的程序被调入内存执行。首先，客户端在网络中寻找DHCP服务器，然后请求一个IP地址；同时DHCP服务器联系到TFTP服务器为此客户端发送一个bootstrap\(引导程序\)。客户端收到bootstrap\(文件pxelinux.0\)后执行，bootstrap会请求TFTP传送bootstrap的配置文件\(pxelinux.cfg\)。收到后读配置文件。根据配置文件内容和客户情况，客户端请求TFTP传送内核映象文件\(vmlinuz\)和根文件系统文件\(initrd.img\)。最后启动内核。这就是一个完整的pxe构建过程。然而要使网卡启动后再继续网络安装系统，则最后还需要FTP服务将系统所需安装文件放置FTP相应目录中进行传输安装。

## 安装

\#yum –disablerepo=\_ --enablerepo=c5-mediainstall dchp\_ tftp\* ftp\* system-config-kickstart\*

## 配置服务

安装完DHCP后，其配置文件为空。我们可以根据其文档中的样本修改

\[root@pxe ~\]\# cat/usr/share/doc/dhcp-3.0.5/dhcpd.conf.sample

/etc/dhcpd.conf

然后修改其内容

\[root@pxe ~\]\# vi /etc/dhcpd.conf  

ddns-update-style interim;  

ignore client-updates;  

allow booting; \#新添加  

allow bootp; \#新添加  

subnet 192.168.128.0 netmask 255.255.255.0{  

\# --- default gateway  

option routers                 192.168.128.1;  

option subnet-mask             255.255.255.0;  

option nis-domain              "domain.org";  

option domain-name             "domain.org";  

\#      option domain-name-servers     192.168.128.1;  \#注释此行，以加快启动  



option time-offset             -18000; \# Eastern Standard Time  

\#      option ntp-servers             192.168.1.1;  

\#      option netbios-name-servers    192.168.1.1;  

\# --- Selects point-to-point node \(defaultis hybrid\). Don't change this unless  

\# -- you understand Netbios very well  

\#      option netbios-node-type 2;  



range dynamic-bootp 192.168.128.150 192.168.128.200;  

filename "/pxelinux.0"; \#指定启动文件  

next-server 192.168.128.111; \#指定服务器IP  

default-lease-time 21600;  

max-lease-time 43200;  



\# we want the nameserver to appear at a fixed address  

host ns {  

next-server marvin.redhat.com;  

hardware ethernet12:34:56:78:AB:CD;  

fixed-address 207.175.42.254;  

}  

}  

修改TFTP的配置



* \[root@pxe ~\]\# cat /etc/xinetd.d/tftp  
* \# default: off  
* \# description: The tftp server serves filesusing the trivial file transfer   
* \#      protocol.  The tftp protocol isoften used to boot diskless   
* \#      workstations, download configuration files to network-aware printers,   
* \#      and to start the installation process for some operating systems.  
* service tftp  
* {  
* socket\_type             = dgram  
* protocol                = udp  
* wait                    = yes  
* user                    = root  
* server                  = /usr/sbin/in.tftpd  
* server\_args             = -s/tftpboot  
* disable                 = no    \#将yes改为no  
* per\_source              = 11  
* cps                     = 100 2  
* flags                   = IPv4  

}  

FTP使用匿名登录，使用默认目录/var/ftp，配置不做修改，只需将linux镜像文件放在/var/ftp下。

* \[root@pxe ~\]\# ls /var/ftp/cdrom/  
* CentOS images    RELEASE-NOTES-cs       RELEASE-NOTES-de.html  RELEASE-NOTES-en\_US       RELEASE-NOTES-es.html  RELEASE-NOTES-ja       RELEASE-NOTES-nl.html     RELEASE-NOTES-ro       RPM-GPG-KEY-beta  
* EULA   isolinux RELEASE-NOTES-cs.html RELEASE-NOTES-en      RELEASE-NOTES-en\_US.html RELEASE-NOTES-fr      RELEASE-NOTES-ja.html RELEASE-NOTES-pt\_BR      RELEASE-NOTES-ro.html RPM-GPG-KEY-CentOS-5  
* GPL    NOTES     RELEASE-NOTES-de       RELEASE-NOTES-en.html  RELEASE-NOTES-es          RELEASE-NOTES-fr.html  RELEASE-NOTES-nl       RELEASE-NOTES-pt\_BR.html  repodata               TRANS.TBL  
* 在TFTP服务安装完后，会创建一个/tftpboot目录，这个目录便是我们用来放置bootstrap引导程序\(pxelinux.0\)，bootstrap配置文件\(default\)，内核映像\(vmlinuz\)和文件系统文件\(initrd.img\)的。这里我们需要先创建一个pxelinux.cfg目录，并将default文件放在其下。



* \[root@pxe ~\]\# ll /tftpboot/  
* 总计 9828  
* -rw-r--r-- 1 root root 8056614 04-18 00:57initrd.img  
* -rw-r--r-- 1 root root   13148 04-19 20:22 pxelinux.0  
* drwxr-xr-x 2 root root    4096 04-20 18:40 pxelinux.cfg  
* -rw-r--r-- 1 root root 1953660 04-19 20:21vmlinuz  
* \[root@pxe ~\]\# ll /tftpboot/pxelinux.cfg/  
* 总计 8  
* -rwxr-xr-x 1 root root 396 04-19 22:03default  

上述四个文件的来源：

* \[root@pxe ~\]\# cp/usr/lib/syslinux/pxelinux.0 /tftpboot/  
* \[root@pxe ~\]\# cd /var/ftp/cdrom/isolinux/  
* \[root@pxe isolinux\]\# pwd  
* /var/ftp/cdrom/isolinux  
* \[root@pxe isolinux\]\# cp initrd.imgisolinux.cfg vmlinuz /tftpboot/  

这里的isolinux.cfg就是我们的default文件，我们需要更名并放在pxelinux.cfg目录下



* \[root@pxe tftpboot\]\# mkdir pxelinux.cfg/  
* \[root@pxe tftpboot\]\# mv isolinux.cfgpxelinux.cfg/  
* \[root@pxe tftpboot\]\# cd pxelinux.cfg/  
* \[root@pxe pxelinux.cfg\]\# mv isolinux.cfgdefault  

default文件我们还需要一些设置

\[root@pxe pxelinux.cfg\]\# vi default  

* default linux  
* prompt 1  
* timeout 600  
* display boot.msg  
* F1 boot.msg  
* F2 options.msg  
* F3 general.msg  
* F4 param.msg  
* F5 rescue.msg  
* label linux  
* kernel vmlinuz  
* \# ks=ftp://192.168.128.111/ks.cfg指向的是ftp服务器/var/ftp下的文件，这个文件就是linux安装的应答文件，即我们前面说的Kickstart文件  
* append ks=ftp://192.168.128.111/ks.cfg initrd=initrd.img  
* label text  
* kernel vmlinuz  
* append initrd=initrd.img text  
* label ks  
* kernel vmlinuz  
* append ks initrd=initrd.img  
* label local  
* localboot 1  
* label memtest86  
* kernel memtest  
* append -  

kickstart文件的制作可以参考[Kickstart的配置和使用](http://blog.csdn.net/limb99/article/details/7481726)

最后要将ks.cfg文件放在/var/ftp下



* \[root@pxe ftp\]\# ll  
* 总计 24  
* drwxr-xr-x 7 root root 4096 04-19 23:04cdrom  
* -rw-r--r-- 1 root root 1399 04-20 20:41ks.cfg  
* drwxr-xr-x 2 root root 4096 04-20 20:43 pub  

启动服务

启动DHCP服务

1. \[root@pxe ftp\]\# service dhcpd start  

启动TFTP服务

1. \[root@pxe ftp\]\# service xinetd start  

启动FTP服务

1. \[root@pxe ftp\]\# service vsftpd start  

至此，服务器配置完成。

