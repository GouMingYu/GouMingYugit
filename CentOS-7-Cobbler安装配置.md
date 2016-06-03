
# CentOS 7 Cobbler安装配置 #

## 1. Cobbler介绍 ##

Cobbler是一个快速网络安装linux的服务，而且在经过调整也可以支持网络安装windows。该工具使用python开发，小巧轻便（才15k行python代码），使用简单的命令即可完成PXE网络安装环境的配置，同时还可以管理DHCP、DNS、TFTP、RSYNC以及yum仓库、构造系统ISO镜像。  
Cobbler支持命令行管理，web界面管理，还提供了API接口，可以方便二次开发使用。  
Cobbler客户端Koan支持虚拟机安装和操作系统重新安装，使重装系统更便捷。


## 1.2 系统环境准备 ##

	[root@linux-node2 ~]# cat /etc/redhat-release
	CentOS Linux release 7.2.1511 (Core)
	[root@linux-node2 ~]# uname -r               
	3.10.0-327.el7.x86_64
	[root@linux-node2 ~]# getenforce 
	Disabled
	systemctl stop firewalld 
	[root@linux-node2 ~]# hostname
	linux-node2.oldboyedu.com

安装repo源

	wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo

>
**注意：**  
网卡使用的是NAT模式，因为我们会搭建DHCP服务器，在同一局域网多个DHCP服务会有冲突，并且导致实践失败。


## 2. Cobbler安装配置 ##
### 2.1 安装Cobbler ###

**下载所需服务：**  
**cobbler**   服务包  
**cobbler-web**  cobbler的web服务包  
**dhcp**  dhcp服务  
**tftp**  tftp服务  
**pykickstart**  cobbler检查kickstart语法错误  
**http**  web服务  
**xinetd**  网络守护进程服务  

	yum install cobbler cobbler-web dhcp tftp-server pykickstart httpd xinetd -y

**Cobbler文件说明：**
>
- /etc/cobbler                   # 配置文件目录
- /etc/cobbler/settings          # cobbler主配置文件
- /etc/cobbler/dhcp.template     # DHCP服务的配置模板
- /etc/cobbler/tftpd.template    # tftp服务的配置模板
- /etc/cobbler/rsync.template    # rsync服务的配置模板
- /etc/cobbler/iso               # iso模板配置文件目录
- /etc/cobbler/pxe               # pxe模板文件目录
- /etc/cobbler/power             # 电源的配置文件目录
- /etc/cobbler/users.conf        # Web服务授权配置文件
- /etc/cobbler/users.digest      # web访问的用户名密码配置文件
- /etc/cobbler/dnsmasq.template  # DNS服务的配置模板
- /etc/cobbler/modules.conf      # Cobbler模块配置文件
- /var/lib/cobbler               # Cobbler数据目录
- /var/lib/cobbler/config        # 配置文件
- /var/lib/cobbler/kickstarts    # 默认存放kickstart文件
- /var/lib/cobbler/loaders       # 存放的各种引导程序
- /var/www/cobbler               # 系统安装镜像目录
- /var/www/cobbler/ks_mirror     # 导入的系统镜像列表
- /var/www/cobbler/images        # 导入的系统镜像启动文件
- /var/www/cobbler/repo_mirror   # yum源存储目录
- /var/log/cobbler               # 日志目录
- /var/log/cobbler/install.log   # 客户端系统安装日志
- /var/log/cobbler/cobbler.log   # cobbler日志

### 2.2 配置Cobbler ###
#### 2.2.1 检测Cobbler ####

>cobbler的运行依赖于dhcp、tftp、rsync及dns服务，其中dhcp可由dhcpd（isc）提供，也可由dnsmasq提供；tftp可由tftp-server程序包提供，也可由cobbler功能提供，rsync有rsync程序包提供，dns可由bind提供，也可由dnsmasq提供。  
cobbler可自行管理这些服务中的部分甚至是全部，但需要配置文件/etc/cobbler/settings中的“manange_dhcp”、“manager_tftpd”、“manager_rsync”、“manager_dns”分别来进行定义，另外，由于各种服务都有着不同的实现方式，如若需要进行自定义，需要通过修改/etc/cobbler/modules.conf配置文件中各服务的模块参数的值来实现。  

>**注意：**  
检查Cobbler必须在Cobbler和web服务都启动的情况下方可。

	systemctl start httpd
	systemctl start cobblerd
	cobbler check


**按提示一一解决即可**

>The following are potential configuration items that you may want to fix:

>1 : The 'server' field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolvable hostname or IP for the boot server as reachable by all machines that will use it.

>2 : For PXE to be functional, the 'next_server' field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boot server on the PXE network.

>3 : change 'disable' to 'no' in /etc/xinetd.d/tftp

>4 : some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.

>5 : enable and start rsyncd.service with systemctl

>6 : debmirror package is not installed, it will be required to manage debian deployments and repositories

>7 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to 'cobbler' and should be changed, try: "openssl passwd -1 -salt 'random-phrase-here' 'your-password-here'" to generate new one

>8 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them

>Restart cobblerd and then run 'cobbler sync' to apply changes.



1 . 修改/etc/cobbler/settings文件中server地址为Cobbler服务器IP

	server: 192.168.56.12


2 . 修改/etc/cobbler/settings文件中netx_server地址为PXE服务器IP
	
	next_server: 192.168.56.12
	

3 . 修改/etc/xinetd.d/tftp文件中disable的值为no
	
	disable                 = no
	

4 . 下载网络引导装载程序，否则，需要安装syslinux程序包，而后复制/usr/share/syslinux/{pxelinux.0,memu.c32}等文件至/var/lib/cobbler/loaders/目录中；

	cobbler get-loaders
	cd /var/lib/cobbler/loaders/  # 查看下载的内容
	ls
	  COPYING.elilo     COPYING.yaboot  pxelinux.0 
	  grub-x86_64.efi menu.c32    README  
	  COPYING.syslinux  elilo-ia64.efi  
           

5 . 启动rsync服务

	systemctl enable rsyncd.service


6 . 安装debmirror包，如没有debian环境可以忽略，然后根据错误进行解决,一般错误如下。
注释/etc/dedmirror.conf文件中的  @dists=”sid”;  @arches=”i386”;

	yum install debmirror


7 . 生成密码并替换setting文件中的默认密码

	openssl passwd -1 -salt 'cobbler' 'cobbler'
	vim /etc/cobbler/settings
	default_password_crypted: "$1$cobbler$M6SE55xZodWc9.vAKLJs6."


8 .	下载电源管理功能（可选）

	yum -y install cman fence-agents


**重启Cobbler即可**

	systemctl restart cobblerd                   
	cobbler check


### 2.2.2 配置DHCP服务 ###
1.使用Cobbler管理DHCP服务 

	sed -ir 's#manage_dhcp: 0#manage_dhcp: 1#g' /etc/cobbler/settings


2.修改Cobbler的DHCP模版

>注意：  
不要直接修改dhcp本身的配置文件，因为cobbler会覆盖。

	vim /etc/cobbler/dhcp.template
	subnet 192.168.56.0 netmask 255.255.255.0 {  #网段
	     option routers             192.168.56.2;#网关
	     option domain-name-servers 192.168.56.2;#DNS
	     option subnet-mask         255.255.255.0;
	     range dynamic-bootp        192.168.56.100 192.168.56.254;
	     default-lease-time         21600;
	     max-lease-time             43200;
	     next-server                $next_server;


### 2.2.3 同步Cobbler ###

	systemctl restart xinetd 
	systemctl restart cobblerd 
	cobbler sync 

### 2.2.4 加入开机自启动 ###
**方法1**  
>加入系统开启自启动

	systemctl enable cobblerd.service  
	systemctl enable xinetd.service   
	systemctl enable httpd.service  
	systemctl enable rsyncd.service
	systemctl enable dhcpd.service

**方法2**
>编写自定义启动脚本，加入开机自启动

	vim /etc/init.d/cobbler.sh
    
	#!/bin/bash  
	# chkconfig: 345 80 90  
	# description:cobbler  
	case $1 in
	  start)
	    systemctl start httpd 
	    systemctl start xinetd
	    systemctl start dhcpd
	    systemctl start cobblerd
	    ;;
      stop)
        systemctl stop httpd 
        systemctl stop xinetd 
        systemctl stop dhcpd 
        systemctl stop cobblerd 
    	;;
      restart)
    	systemctl restart httpd 
    	systemctl restart xinetd 
    	systemctl restart dhcpd 
    	systemctl restart cobblerd 
    	;;
      status)
    	systemctl status httpd 
    	systemctl status xinetd 
    	systemctl status dhcpd 
    	systemctl status cobblerd 
    	;;
      sync)
    	cobbler sync
    	;;
      *)
     	 echo "Input error,please in put 'start|stop|restart|status|sync'!"
     	 exit 2
     	 ;;
	esac


## 3. Cobbler的命令行管理 ##
### 3.1 查看命令帮助 ###
	[root@linux-node1 ~]# cobbler
	usage
	=====
	cobbler <distro|profile|system|repo|image|mgmtclass|package|file> ... 
        [add|edit|copy|getks*|list|remove|rename|report] [options|--help]
	cobbler <aclsetup|buildiso|import|list|replicate|report|reposync|sync|validateks|version|signature|get-loaders|hardlink> [options|--help]  

	[root@linux-node1 ~]# cobbler import --help
	Usage: cobbler [options]

	Options:
	  -h, --help            show this help message and exit
	  --arch=ARCH           OS architecture being imported
	  --breed=BREED         the breed being imported
	  --os-version=OS_VERSION
                        the version being imported
	  --path=PATH           local path or rsync location
	  --name=NAME           name, ex 'RHEL-5'
	  --available-as=AVAILABLE_AS
                        tree is here, don't mirror
	  --kickstart=KICKSTART_FILE
                        assign this kickstart file
	  --rsync-flags=RSYNC_FLAGS
                        pass additional flags to rsync

	cobbler check    核对当前设置是否有问题
	cobbler list     列出所有的cobbler元素
	cobbler report   列出元素的详细信息
	cobbler sync     同步配置到数据目录,更改配置最好都要执行下
	cobbler reposync 同步yum仓库
	cobbler distro   查看导入的发行版系统信息
	cobbler system   查看添加的系统信息
	cobbler profile  查看配置信息


### 3.2 导入镜像 ###

cobbler变得可用的第一步为定义distro，其可以通过为其指定外部的安装引导内核及ramdisk文件的方式实现。  
如果已经有完成的安装树（如os的安装镜像）则推荐使用improt导入的方式进行。

	mount /dev/cdrom /mnt/   # 挂载镜像光盘
	cobbler import --path=/mnt/ --name=CentOS-7.1-x86_64-distro --arch=x86_64
	cobbler distro list  	 # 列出所有的distro
	cobbler profile list	 # 列出所有的prefile（导入distro自动生成）


>**参数说明：**  
--path    # 指定镜像路径  
--name    # 指定安装源名称  
--arch    # 指定安装源位数，目前支持的选项有:x86│x86_64│ia64  

安装源的唯一标示就是根据name参数来定义，本例导入成功后，安装源的唯一标示就是：CentOS-7.1-distro-x86_64。 如果重复，系统会提示导入失败   
镜像存放目录，cobbler会将镜像中的所有安装文件拷贝到本地一份，放在/var/www/cobbler/ks_mirror下的CentOS-7.1-x86_64-distro-x86_64目录下。因此/var/www/cobbler目录必须具有足够容纳安装文件的空间。

	[root@linux-node1 ~]# cd /var/www/cobbler/ks_mirror/
	[root@linux-node1 ks_mirror]# ls
	CentOS-6.7-x86_64  CentOS-7.1-x86_64  config
	[root@linux-node1 ks_mirror]# ls CentOS-7.1-x86_64/
	CentOS_BuildTag  EULA  images    LiveOS    repodata              RPM-GPG-KEY-CentOS-Testing-7
	EFI              GPL   isolinux  Packages  RPM-GPG-KEY-CentOS-7  TRANS.TBL

>**提示：**  
如果有kickstart文件，也可以使用–kickstart=/path/to/kickstart_file进行导入，因此import会自动为导入的distro生成一个profile

### 3.3 指定ks.cfg文件及调整内核参数 ###

	[root@linux-node1 ks_mirror]# cd /var/lib/cobbler/kickstarts/
	[root@linux-node1 kickstarts]# ls
	CentOS-6-x86_64.cfg  esxi4-ks.cfg      legacy.ks            sample_end.ks（默认使用的ks文件）    sample_esxi5.ks  sample.seed
	CentOS-7-x86_64.cfg  esxi5-ks.cfg      pxerescue.ks         sample_esx4.ks   sample.ks
	default.ks           install_profiles  sample_autoyast.xml  sample_esxi4.ks  sample_old.seed
	[root@linux-node1 cobbler]# rz -y  上传准备好的ks文件
	rz waiting to receive.
     zmodem trl+C ȡ
	  100%       1 KB    1 KB/s 00:00:01       0 Errors
>说明：  
在第一次导入系统镜像后，Cobbler会给镜像指定一个默认的kickstart自动安装文件在/var/lib/cobbler/kickstarts下的sample_end.ks。

	[root@linux-node1 cobbler]# cobbler list
	distros:
	   CentOS-6.7-x86_64
	   CentOS-7.1-x86_64
	
	profiles:
	   CentOS-6.7-x86_64
	   CentOS-7.1-x86_64
	
	systems:
	   linux-node2.oldboyedu.com
	
	repos:
	   base
	
	images:
	
	mgmtclasses:
	
	packages:
	
	files:

**查看安装镜像文件信息**  

	[root@linux-node1 cobbler]# cobbler distro report --name=CentOS-7.1-x86_64
	Name                           : CentOS-7.1-x86_64
	Architecture                   : x86_64
	TFTP Boot Files                : {}
	Breed                          : redhat
	Comment                        : 
	Fetchable Files                : {}
	Initrd                         : /var/www/cobbler/ks_mirror/CentOS-7.1-x86_64/images/pxeboot/initrd.img
	Kernel                         : /var/www/cobbler/ks_mirror/CentOS-7.1-x86_64/images/pxeboot/vmlinuz
	Kernel Options                 : {}
	Kernel Options (Post Install)  : {}
	Kickstart Metadata             : {'tree': 'http://@@http_server@@/cblr/links/CentOS-7.1-x86_64'}
	Management Classes             : []
	OS Version                     : rhel7
	Owners                         : ['admin']
	Red Hat Management Key         : <<inherit>>
	Red Hat Management Server      : <<inherit>>
	Template Files                 : {}
	
查看所有的profile设置  

    [root@linux-node1 cobbler]# cobbler profile report  
    Name                           : CentOS-7.1-x86_64  
    TFTP Boot Files                : {}  
    Comment                        :   
    DHCP Tag                       : default  
    Distribution                   : CentOS-7.1-x86_64  
    Enable gPXE?                   : 0  
    Enable PXE Menu?               : 1  
    Fetchable Files                : {}  
    Kernel Options                 : {'biosdevname': '0', 'net.ifnames': '0'}  
    Kernel Options (Post Install)  : {}  
    Kickstart                      : /var/lib/cobbler/kickstarts/CentOS-7-  x86_64.cfg  
    Kickstart Metadata             : {}  
	Management Classes             : []  
	Management Parameters          : <<inherit>>  
	Name Servers                   : []  
	Name Servers Search Path       : []  
	Owners                         : ['admin']  
	Parent Profile                 :   
	Internal proxy                 :   
	Red Hat Management Key         : <<inherit>>  
	Red Hat Management Server      : <<inherit>>  
	Repos                          : []  
	Server Override                : <<inherit>>  
	Template Files                 : {}  
	Virt Auto Boot                 : 1  
	Virt Bridge                    : xenbr0  
	Virt CPUs                      : 1  
	Virt Disk Driver Type          : raw  
	Virt File Size(GB)             : 5  
	Virt Path                      :   
	Virt RAM (MB)                  : 512  
	Virt Type                      : kvm  
	  
	Name                           : CentOS-6.7-x86_64  
	TFTP Boot Files                : {}  
	Comment                        :   
	DHCP Tag                       : default  
	Distribution                   : CentOS-6.7-x86_64  
	Enable gPXE?                   : 0  
	Enable PXE Menu?               : 1  
	Fetchable Files                : {}  
	Kernel Options                 : {}  
	Kernel Options (Post Install)  : {}  
	Kickstart                      : /var/lib/cobbler/kickstarts/CentOS-6-  x86_64.cfg  
	Kickstart Metadata             : {}  
	Management Classes             : []  
	Management Parameters          : <<inherit>>  
	Name Servers                   : []  
	Name Servers Search Path       : []  
	Owners                         : ['admin']  
	Parent Profile                 :   
	Internal proxy                 :   
	Red Hat Management Key         : <<inherit>>  
	Red Hat Management Server      : <<inherit>>  
	Repos                          : []  
	Server Override                : <<inherit>>  
	Template Files                 : {}  
	Virt Auto Boot                 : 1  
	Virt Bridge                    : xenbr0  
	Virt CPUs                      : 1  
	Virt Disk Driver Type          : raw  
	Virt File Size(GB)             : 5  
	Virt Path                      :   
	Virt RAM (MB)                  : 512  
	Virt Type                      : kvm  

查看指定的profile设置

	[root@linux-node1 cobbler]# cobbler profile report --name=CentOS-7.1-x86_64
	Name                           : CentOS-7.1-x86_64
	TFTP Boot Files                : {}
	Comment                        : 
	DHCP Tag                       : default
	Distribution                   : CentOS-7.1-x86_64
	Enable gPXE?                   : 0
	Enable PXE Menu?               : 1
	Fetchable Files                : {}
	Kernel Options                 : {'biosdevname': '0', 'net.ifnames': '0'}
	Kernel Options (Post Install)  : {}
	Kickstart                      : /var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg
	Kickstart Metadata             : {}
	Management Classes             : []
	Management Parameters          : <<inherit>>
	Name Servers                   : []
	Name Servers Search Path       : []
	Owners                         : ['admin']
	Parent Profile                 : 
	Internal proxy                 : 
	Red Hat Management Key         : <<inherit>>
	Red Hat Management Server      : <<inherit>>
	Repos                          : []
	Server Override                : <<inherit>>
	Template Files                 : {}
	Virt Auto Boot                 : 1
	Virt Bridge                    : xenbr0
	Virt CPUs                      : 1
	Virt Disk Driver Type          : raw
	Virt File Size(GB)             : 5
	Virt Path                      : 
	Virt RAM (MB)                  : 512
	Virt Type                      : kvm

**编辑profile，修改关联的ks文件**  
cobbler使用profile来为特定的需求类别提供所需要安装的配置，即在distro的基础上通过提供kiskstart文件来生成一个特定的系统安装配置。distro的profile可以出现在pxe的引导菜单中作为安装的选择之一。

	cobbler profile edit --name=CentOS-7.1-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS-7.1-x86_64.cfg

>说明：
默认是有kickstart文件的，所以edit，如果没有kickstart文件可以add

CentOS7系统网卡名变成eno…这种，为了运维标准化，我们需要修改为我们常用的eth0，使用下面的参数。但要注意是CentOS7才需要下面的步骤，CentOS6不需要

	cobbler profile edit --name=CentOS-7.1-distro-x86_64 --kopts='net.ifnames=0 biosdevname=0'
	
每次修改配置都要同步一下

	cobbler sync


### 重启PXE ###

	systemctl start xinetd
	systemctl start httpd


新部署机器安装yum源，并同步。建议使用内网yum源，在这里使用阿里云yum源

1 . 添加yum源

	cobbler repo add --name=base --mirror=http://mirrors.aliyun.com/centos/7/os/x86_64/Packages/ --arch=x86_64 –breed=yum 


2 . 同步yum源

	cobbler reposync 


3 . 每次修改profile都需要同步

	cobbler sync 


到这里就配置完成了。

部署操作系统


