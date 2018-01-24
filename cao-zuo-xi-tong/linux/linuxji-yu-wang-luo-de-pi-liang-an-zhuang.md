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



**\[plain\]**

[view plain](http://blog.csdn.net/limb99/article/details/7481878#)

[copy](http://blog.csdn.net/limb99/article/details/7481878#)

1. \#yum –disablerepo=\\* --enablerepo=c5-mediainstall dchp\* tftp\* ftp\* system-config-kickstart\*  

  




## 配置服务

安装完DHCP后，其配置文件为空。我们可以根据其文档中的样本修改



**\[plain\]**

[view plain](http://blog.csdn.net/limb99/article/details/7481878#)

[copy](http://blog.csdn.net/limb99/article/details/7481878#)

1. \[root@pxe ~\]\# cat/usr/share/doc/dhcp-3.0.5/dhcpd.conf.sample 
   &gt;
   &gt;
    /etc/dhcpd.conf  

  




然后修改其内容



**\[plain\]**

[view plain](http://blog.csdn.net/limb99/article/details/7481878#)

[copy](http://blog.csdn.net/limb99/article/details/7481878#)

1. \[root@pxe ~\]\# vi /etc/dhcpd.conf  
2. ddns-update-style interim;  
3. ignore client-updates;  
4. allow booting; \#新添加  
5. allow bootp; \#新添加  
6. subnet 192.168.128.0 netmask 255.255.255.0{  
7. 
8. \# --- default gateway  
9.        option routers                 192.168.128.1;  
10.        option subnet-mask             255.255.255.0;  
11. 
12.        option nis-domain              "domain.org";  
13.        option domain-name             "domain.org";  
14. \#      option domain-name-servers     192.168.128.1;  \#注释此行，以加快启动  
15. 
16.        option time-offset             -18000; \# Eastern Standard Time  
17. \#      option ntp-servers             192.168.1.1;  
18. \#      option netbios-name-servers    192.168.1.1;  
19. \# --- Selects point-to-point node \(defaultis hybrid\). Don't change this unless  
20. \# -- you understand Netbios very well  
21. \#      option netbios-node-type 2;  
22. 
23.        range dynamic-bootp 192.168.128.150 192.168.128.200;  
24.        filename "/pxelinux.0"; \#指定启动文件  
25.        next-server 192.168.128.111; \#指定服务器IP  
26.        default-lease-time 21600;  
27.        max-lease-time 43200;  
28. 
29.        \# we want the nameserver to appear at a fixed address  
30.        host ns {  
31.                 next-server marvin.redhat.com;  
32.                 hardware ethernet12:34:56:78:AB:CD;  
33.                 fixed-address 207.175.42.254;  
34.        }  
35. }  

  


  




修改TFTP的配置



**\[plain\]**

[view plain](http://blog.csdn.net/limb99/article/details/7481878#)

[copy](http://blog.csdn.net/limb99/article/details/7481878#)

1. \[root@pxe ~\]\# cat /etc/xinetd.d/tftp  
2. \# default: off  
3. \# description: The tftp server serves filesusing the trivial file transfer \  
4. \#      protocol.  The tftp protocol isoften used to boot diskless \  
5. \#      workstations, download configuration files to network-aware printers, \  
6. \#      and to start the installation process for some operating systems.  
7. service tftp  
8. {  
9.        socket\_type             = dgram  
10.        protocol                = udp  
11.        wait                    = yes  
12.        user                    = root  
13.        server                  = /usr/sbin/in.tftpd  
14.        server\_args             = -s/tftpboot  
15.        disable                 = no    \#将yes改为no  
16.        per\_source              = 11  
17.        cps                     = 100 2  
18.        flags                   = IPv4  
19. }  

  


  




FTP使用匿名登录，使用默认目录/var/ftp，配置不做修改，只需将linux镜像文件放在/var/ftp下。



**\[plain\]**

[view plain](http://blog.csdn.net/limb99/article/details/7481878#)

[copy](http://blog.csdn.net/limb99/article/details/7481878#)

1. \[root@pxe ~\]\# ls /var/ftp/cdrom/  
2. CentOS images    RELEASE-NOTES-cs       RELEASE-NOTES-de.html  RELEASE-NOTES-en\_US       RELEASE-NOTES-es.html  RELEASE-NOTES-ja       RELEASE-NOTES-nl.html     RELEASE-NOTES-ro       RPM-GPG-KEY-beta  
3. EULA   isolinux RELEASE-NOTES-cs.html RELEASE-NOTES-en      RELEASE-NOTES-en\_US.html RELEASE-NOTES-fr      RELEASE-NOTES-ja.html RELEASE-NOTES-pt\_BR      RELEASE-NOTES-ro.html RPM-GPG-KEY-CentOS-5  
4. GPL    NOTES     RELEASE-NOTES-de       RELEASE-NOTES-en.html  RELEASE-NOTES-es          RELEASE-NOTES-fr.html  RELEASE-NOTES-nl       RELEASE-NOTES-pt\_BR.html  repodata               TRANS.TBL  
5. 
  




在TFTP服务安装完后，会创建一个/tftpboot目录，这个目录便是我们用来放置bootstrap引导程序\(pxelinux.0\)，bootstrap配置文件\(default\)，内核映像\(vmlinuz\)和文件系统文件\(initrd.img\)的。这里我们需要先创建一个pxelinux.cfg目录，并将default文件放在其下。



**\[plain\]**

[view plain](http://blog.csdn.net/limb99/article/details/7481878#)

[copy](http://blog.csdn.net/limb99/article/details/7481878#)

1. \[root@pxe ~\]\# ll /tftpboot/  
2. 总计 9828  
3. -rw-r--r-- 1 root root 8056614 04-18 00:57initrd.img  
4. -rw-r--r-- 1 root root   13148 04-19 20:22 pxelinux.0  
5. drwxr-xr-x 2 root root    4096 04-20 18:40 pxelinux.cfg  
6. -rw-r--r-- 1 root root 1953660 04-19 20:21vmlinuz  
7. \[root@pxe ~\]\# ll /tftpboot/pxelinux.cfg/  
8. 总计 8  
9. -rwxr-xr-x 1 root root 396 04-19 22:03default  

  


  




上述四个文件的来源：



**\[plain\]**

[view plain](http://blog.csdn.net/limb99/article/details/7481878#)

[copy](http://blog.csdn.net/limb99/article/details/7481878#)

1. \[root@pxe ~\]\# cp/usr/lib/syslinux/pxelinux.0 /tftpboot/  
2. \[root@pxe ~\]\# cd /var/ftp/cdrom/isolinux/  
3. \[root@pxe isolinux\]\# pwd  
4. /var/ftp/cdrom/isolinux  
5. \[root@pxe isolinux\]\# cp initrd.imgisolinux.cfg vmlinuz /tftpboot/  

  




这里的isolinux.cfg就是我们的default文件，我们需要更名并放在pxelinux.cfg目录下



**\[plain\]**

[view plain](http://blog.csdn.net/limb99/article/details/7481878#)

[copy](http://blog.csdn.net/limb99/article/details/7481878#)

1. \[root@pxe tftpboot\]\# mkdir pxelinux.cfg/  
2. \[root@pxe tftpboot\]\# mv isolinux.cfgpxelinux.cfg/  
3. \[root@pxe tftpboot\]\# cd pxelinux.cfg/  
4. \[root@pxe pxelinux.cfg\]\# mv isolinux.cfgdefault  
5. 
  




default文件我们还需要一些设置



**\[plain\]**

[view plain](http://blog.csdn.net/limb99/article/details/7481878#)

[copy](http://blog.csdn.net/limb99/article/details/7481878#)

1. \[root@pxe pxelinux.cfg\]\# vi default  
2. default linux  
3. prompt 1  
4. timeout 600  
5. display boot.msg  
6. F1 boot.msg  
7. F2 options.msg  
8. F3 general.msg  
9. F4 param.msg  
10. F5 rescue.msg  
11. label linux  
12.  kernel vmlinuz  
13.   \# ks=ftp://192.168.128.111/ks.cfg指向的是ftp服务器/var/ftp下的文件，这个文件就是linux安装的应答文件，即我们前面说的Kickstart文件  
14.  append ks=ftp://192.168.128.111/ks.cfg initrd=initrd.img  
15. label text  
16.  kernel vmlinuz  
17.  append initrd=initrd.img text  
18. label ks  
19.  kernel vmlinuz  
20.  append ks initrd=initrd.img  
21. label local  
22.  localboot 1  
23. label memtest86  
24.  kernel memtest  
25.  append -  

  




kickstart文件的制作可以参考[Kickstart的配置和使用](http://blog.csdn.net/limb99/article/details/7481726)

最后要将ks.cfg文件放在/var/ftp下



**\[plain\]**

[view plain](http://blog.csdn.net/limb99/article/details/7481878#)

[copy](http://blog.csdn.net/limb99/article/details/7481878#)

1. \[root@pxe ftp\]\# ll  
2. 总计 24  
3. drwxr-xr-x 7 root root 4096 04-19 23:04cdrom  
4. -rw-r--r-- 1 root root 1399 04-20 20:41ks.cfg  
5. drwxr-xr-x 2 root root 4096 04-20 20:43 pub  

  




启动服务

启动DHCP服务



**\[plain\]**

[view plain](http://blog.csdn.net/limb99/article/details/7481878#)

[copy](http://blog.csdn.net/limb99/article/details/7481878#)

1. \[root@pxe ftp\]\# service dhcpd start  



启动TFTP服务



**\[plain\]**

[view plain](http://blog.csdn.net/limb99/article/details/7481878#)

[copy](http://blog.csdn.net/limb99/article/details/7481878#)

1. \[root@pxe ftp\]\# service xinetd start  

启动FTP服务





**\[plain\]**

[view plain](http://blog.csdn.net/limb99/article/details/7481878#)

[copy](http://blog.csdn.net/limb99/article/details/7481878#)

1. \[root@pxe ftp\]\# service vsftpd start  

  


  


至此，服务器配置完成。

