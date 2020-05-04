# 使用VirtualBox在Oracle Linux 7上安装Oracle Database 19c RAC

**最新更新为2019-11-20，发布于2020-5-3**

本文介绍了使用VirtualBox（6.0.10）在Linux（Oracle Linux 7.7 64位）上安装Oracle Database 19c RAC，而没有其他共享磁盘设备的情况。

## 介绍

阻碍人们建立测试RAC环境的最大障碍之一是对共享存储的需求。在生产环境中，共享存储通常由SAN或高端NAS设备提供，但是当您要做的只是获得一些安装和使用RAC的经验时，这两种选择都非常昂贵。一种更便宜的选择是使用FireWire磁盘盒，以允许两台计算机访问相同的磁盘，但这仍然要花钱，并且需要两台服务器。第三种选择是使用虚拟化来伪造共享存储。

使用VirtualBox，您可以在一台服务器上运行多个虚拟机（VM），从而使您可以在一台计算机上运行两个RAC节点。此外，它还允许您设置共享虚拟磁盘，从而克服了昂贵的共享存储的障碍。

![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/19c_arch_Diagram.png "Introduction")


在开始进行此安装之前，请考虑以下几点。
- 完成的系统包括主机操作系统，三个来宾操作系统，两套Oracle Grid Infrastructure（集群软件+ ASM）和两个数据库实例，它们都在同一台计算机上。可以想象，这需要大量的磁盘空间，CPU和内存。
- 从最后一点开始，每个RAC节点VM至少需要6G的RAM，但是您会看到我为每个使用了10G，但它仍然很慢。不要以为您可以在小型PC或笔记本电脑上运行它。你不会的
- 此过程提供了基本的安装，以使RAC正常工作。 Grid Infrastructure安装或ASM安装中没有冗余。要添加此文件，只需创建两倍数量的共享磁盘，然后在提供时选择“普通”冗余选项。当然，这将占用更多的磁盘空间。
- 这不是（也不应视为）生产就绪系统的说明。只是为了让您了解安装RAC所需的条件并为您提供一个可以进行试验的系统。
- 需要DNS以支持扫描侦听器。在以前的版本中，我建议在主机服务器上运行DNS，但这比较容易。
- 本文使用Oracle Linux和Oracle 19c的64位版本。
- 在服务器上进行此安装时，我将虚拟磁盘拆分为不同的物理磁盘。这不是必需的，但是会使运行更快。
- 这可能需要三个多小时才能完成。如果存在严重的内存或磁盘速度限制，则可能更长一些。

## 下载软件

下载以下软件。
- [Oracle Linux 7](http://edelivery.oracle.com/linux) (使用最新版本，例如7.7)
- [VirtualBox (6.0.10)](http://www.virtualbox.org/wiki/Downloads)
- [Oracle 19c (19.3) 软件 (64 bit)](http://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html)

本文已针对19c版本进行了更新，但自12.2.0.1起安装基本上没有变化。 19c特有的任何变化都将被注意。

根据您的VirtualBox和Oracle Linux版本，屏幕快照的外观可能会略有不同。

## VirtualBox安装

首先，安装VirtualBox软件。在RHEL及其克隆上，您可以使用以下类型的命令作为root用户来执行此操作。

```console
# rpm -Uvh VirtualBox*.rpm
```

程序包名称将根据所使用的主机分发版本而有所不同。完成后，从菜单启动VirtualBox。

## VirtualBox网络设置
我们需要确保配置了仅主机的网络，并检查/修改了该网络的IP范围。这将是我们RAC安装的公共网络。

- 从菜单中启动VirtualBox。
- 选择“工具”选项。
- 在弹出菜单中单击“网络”。
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113524@2x.jpg "VirtualBox Network Setup")
- 点击屏幕右侧尺寸上的“创建”按钮。将创建一个名为“vboxnet0”的网络。
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113536@2x.jpg "VirtualBox Network Setup")
- 如果您要为公共地址使用其他子网，则可以在此处更改网络详细信息。只要确保您选择的子网与网络上的任何实际子网都不匹配即可。我决定使用默认值，对我来说是“192.168.56.X”。
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113559@2x.jpg "VirtualBox Network Setup")

## 虚拟机设置

现在，我们必须定义两个虚拟RAC节点。 我们可以通过定义一个VM，然后在安装它时将其克隆来节省时间。

启动VirtualBox并单击工具栏上的“新建”按钮。输入名称“ol7-19c-rac1”，操作系统“Linux”和版本“Oracle（64位）”，然后单击“继续”按钮。
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-112613@2x.jpg "Virtual Machine Setup")
输入“ 4096”作为基本内存大小，然后单击“继续”按钮。
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-112838@2x.jpg "Virtual Machine Setup")
通过单击“创建”按钮，接受默认选项以创建新的虚拟硬盘。
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-112911@2x.jpg "Virtual Machine Setup")
通过单击“继续”按钮接受默认的硬盘驱动器文件类型。
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-112924@2x.jpg "Virtual Machine Setup")
通过单击“继续”按钮接受“动态分配”选项。
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-112939@2x.jpg "Virtual Machine Setup")
接受默认位置并将大小设置为“50G”，然后单击“创建”按钮。如果可以将虚拟磁盘分散到不同的物理磁盘上，则可以提高性能。
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113049@2x.jpg "Virtual Machine Setup")
“ol7-19c-rac1” VM将出现在左侧窗格中。向下滚动右侧的详细信息，然后单击“网络”链接。
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113536@2x.jpg "Virtual Machine Setup")
确保启用了“适配器1”，将其设置为“NAT”，然后单击“适配器2”选项卡。
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113617@2x.jpg "Virtual Machine Setup")
确保启用了“适配器2”，将其设置为“仅主机适配器”，然后单击“适配器3”选项卡。
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113639@2x.jpg "Virtual Machine Setup")
确保启用了“适配器3”，将其设置为“内部网络”，然后单击“系统”部分。
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113708@2x.jpg "Virtual Machine Setup")
将“硬盘”移至启动顺序的顶部，然后取消选中“软盘”选项，然后单击“确定”按钮。
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113747@2x.jpg "Virtual Machine Setup")
现在已配置了虚拟机，因此我们可以开始客户机操作系统安装。

## 虚拟机操作系统安装
在突出显示新VM的情况下，单击工具栏上的“开始”按钮。在“选择启动磁盘”屏幕上，选择相关的Oracle Linux ISO映像，然后单击“启动”按钮。
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113838@2x.jpg "Guest Operating System Installation")
出现的控制台窗口将包含Oracle Linux引导屏幕。
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113956@2x.jpg "Guest Operating System Installation")

![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-114118@2x.jpg "Guest Operating System Installation")
像安装基本服务器一样，继续安装Oracle Linux 7。可以在此处找到安装的一般图片指南。更具体地说，应该是至少安装4G +交换分区，禁用防火墙，将SELinux设置为宽松并安装以下软件包组的服务器安装：
- 带有GUI的服务器
- 硬件监控实用程序
- 大型系统性能
- 网络文件系统客户端
- 性能工具
- 兼容性库
- 开发工具

为了与本文的其余部分保持一致，应在安装过程中设置以下信息。
- 主机名：ol7-19c-rac1.localdomain
- enp0s3 (eth0): DHCP（自动连接）
- enp0s8 (eth1): IP=192.168.56.101, Subnet=255.255.255.0, Gateway=192.168.56.1, DNS=192.168.56.1, Search=localdomain（自动连接）
- enp0s9 (eth2): IP=192.168.1.101, Subnet=255.255.255.0, Gateway=<blank>, DNS=<blank>, Search=<blank>（自动连接）

您可以随意更改IP地址以适合您的网络，但是请记住在本文的其余部分中始终与这些调整保持一致。同样，在本文中，我将网络适配器称为enp0s3，enp0s8和enp0s9。在以前的Linux版本中，它们分别是eth0，eth1和eth2。

## Oracle安装先决条件
执行自动设置或手动设置以完成基本先决条件。 所有安装都需要附加设置。

### 自动设置
如果计划使用“oracle-rdbms-server-12cR1-preinstall”软件包执行所有必备安装程序，请发出以下命令。
```console
# yum install -y oracle-database-preinstall-19c
```

早期版本的Oracle Linux需要按照http://public-yum.oracle.com上的说明手动设置Yum存储库。
 
也可能值得进行完整的更新，但是严格来讲这不是必需的。
 ```console
 # yum update -y
 ```

### 手动设置
如果尚未使用“oracle-database-preinstall-19c”软件包执行所有先决条件，则需要手动执行以下安装任务。

将以下行添加到“/etc/sysctl.conf”文件中，或添加到名为“/etc/sysctl.d/98-oracle.conf”的文件中。

```console
fs.file-max = 6815744
kernel.sem = 250 32000 100 128
kernel.shmmni = 4096
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104
kernel.panic_on_oops = 1
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2
fs.aio-max-nr = 1048576
net.ipv4.ip_local_port_range = 9000 65500
```
运行以下命令来更改当前内核参数。
```console
/sbin/sysctl -p
# or
/sbin/sysctl -p /etc/sysctl.d/98-oracle.conf
```

将以下行添加到名为“/etc/security/limits.d/oracle-database-server-19c-preinstall.conf”的文件中。
```console
oracle   soft   nofile    1024
oracle   hard   nofile    65536
oracle   soft   nproc    16384
oracle   hard   nproc    16384
oracle   soft   stack    10240
oracle   hard   stack    32768
oracle   hard   memlock    134217728
oracle   soft   memlock    134217728
```

除了基本的OS安装外，还必须以root用户身份登录时安装以下软件包。 这包括某些软件包的64位和32位版本。
```console
# From Public Yum or ULN
yum install -y bc    
yum install -y binutils
yum install -y compat-libcap1
yum install -y compat-libstdc++
yum install -y elfutils-libelf
yum install -y elfutils-libelf-devel
yum install -y fontconfig-devel
yum install -y glibc
yum install -y glibc-devel
yum install -y ksh
yum install -y libaio
yum install -y libaio-devel
yum install -y libdtrace-ctf-devel
yum install -y libXrender
yum install -y libXrender-devel
yum install -y libX11
yum install -y libXau
yum install -y libXi
yum install -y libXtst
yum install -y libgcc
yum install -y libstdc++
yum install -y libstdc++-devel
yum install -y libxcb
yum install -y make
yum install -y net-tools # Clusterware
yum install -y nfs-utils # ACFS
yum install -y python # ACFS
yum install -y python-configshell # ACFS
yum install -y python-rtslib # ACFS
yum install -y python-six # ACFS
yum install -y targetcli # ACFS
yum install -y smartmontools
yum install -y sysstat
```

创建新的组和用户。

```console
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54323 oper
#groupadd -g 54324 backupdba
#groupadd -g 54325 dgdba
#groupadd -g 54326 kmdba
#groupadd -g 54327 asmdba
#groupadd -g 54328 asmoper
#groupadd -g 54329 asmadmin

useradd -u 54321 -g oinstall -G dba,oper oracle
```

您可以定义其他组，并将其分配给“oracle”用户。 这样可以在安装过程中分配各个组。 对于此安装，我只使用了“dba”组。

```console
groupadd -g 54324 backupdba
groupadd -g 54325 dgdba
groupadd -g 54326 kmdba
groupadd -g 54327 asmdba
groupadd -g 54328 asmoper
groupadd -g 54329 asmadmin
groupadd -g 54330 racdba

useradd -u 54321 -g oinstall -G dba,oper,backupdba,dgdba,kmdba,asmdba,asmoper,asmadmin,racdba oracle
```

### 其他设置
无论您是手动还是自动设置，都必须执行以下步骤。
以root用户身份登录到“ol7-19c-rac1”虚拟机时，执行以下步骤。
设置“oracle”用户的密码。
```console
passwd oracle
```

除了本地主机地址外，“/etc/hosts”文件可以保留为空白，但是我更喜欢将这些地址放入以供参考。
```console
127.0.0.1       localhost.localdomain   localhost
# Public
192.168.56.101   ol7-19c-rac1
192.168.56.102   ol7-19c-rac2
# Private
192.168.1.101   ol7-19c-rac1-priv
192.168.1.102   ol7-19c-rac2-priv
# Virtual
192.168.56.103   ol7-19c-rac1-vip
192.168.56.104   ol7-19c-rac2-vip
# SCAN
#192.168.56.105   ol7-19c-scan
```

由于必须使用DNS解析SCAN地址，因此已将其从主机文件中注释掉，因此它可以在与公用IP相同的子网上的3个地址之间循环漫游。 可以使用[BIND](https://oracle-base.com/articles/linux/dns-configuration-for-scan)或[Dnsmasq](https://oracle-base.com/articles/linux/dnsmasq-for-simple-dns-configurations)，这要简单得多。 如果使用的是Dnsmasq，则将RAC特定的条目放在主机的“/etc/host”文件中，并取消注释SCAN条目，然后重新启动Dnsmasq。
确保“/etc/resolv.conf”文件包含指向正确名称服务器的名称服务器条目。 另外，如果同时存在“域”和“搜索”条目，请注释掉其中之一。 对于此安装，我的“/etc/resolv.conf”看起来像这样。
```console
#domain localdomain
search localdomain
nameserver 192.168.56.1
```

由于存在NAT接口，对“resolv.conf”的更改将被网络管理器覆盖。 因此，现在应该在启动时禁用此接口。 如果需要从VM访问Internet，则可以手动启用它。 编辑“/etc/sysconfig/network-scripts/ifcfg-eth0”文件，进行以下更改。 这将在下一次重新启动后生效。
```console
ONBOOT=no
```

无需立即重新启动。 您可以只运行以下命令。
```console
# ifdown enp0s3
```

此时，第一个节点的联网应类似于以下内容。 请注意，enp0s3没有关联的IP地址，因为它已被禁用。
```console
# ifconfig
enp0s3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 08:00:27:f6:88:78  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp0s8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.56.101  netmask 255.255.255.0  broadcast 192.168.56.255
        inet6 fe80::cf8d:317d:534:17d9  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:82:06:32  txqueuelen 1000  (Ethernet)
        RX packets 574  bytes 54444 (53.1 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 547  bytes 71219 (69.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp0s9: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.101  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::9a9a:f249:61d1:5447  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:2e:2c:cf  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 29  bytes 4250 (4.1 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 0  (Local Loopback)
        RX packets 68  bytes 5780 (5.6 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 68  bytes 5780 (5.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
        ether 52:54:00:4a:12:2f  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

#
```

有了此设置并配置了DNS之后，就可以将SCAN地址解析为所有三个IP地址。
```console
# nslookup ol7-19c-scan
Server:		192.168.56.1
Address:	192.168.56.1#53

Name:	ol7-19c-scan.localdomain
Address: 192.168.56.105
#
```

如下所述修改“/etc/security/limits.d/90-nproc.conf”文件。 请参阅[MOS文档[ID 1487773.1]](https://support.oracle.com/epmos/faces/DocContentDisplay?id=1487773.1)
```console
# Change this
*          soft    nproc    1024

# To this
* - nproc 16384
```

通过编辑“/etc/selinux/config”文件，将SELinux的设置更改为宽松，确保SELINUX标志设置如下。
```console
SELINUX=permissive
```

如果启用了Linux防火墙，则需要禁用或配置它，如[这里](https://oracle-base.com/articles/linux/oracle-linux-6-installation#firewall)或[这里](https://oracle-base.com/articles/linux/linux-firewall#installation)所示。 以下是禁用防火墙的示例。
```console
# systemctl stop firewalld
# systemctl disable firewalld
```

请配置NTP，或确保未配置NTP，以便Oracle集群时间同步服务（ctssd）可以同步RAC节点的时间。

确保启用了NTP（在OL7/RHEL7上为Chrony）。
```console
# systemctl enable chronyd
# systemctl restart chronyd
# chronyc -a 'burst 4/4'
# chronyc -a makestep
```

创建将在其中安装Oracle软件的目录。
```console
mkdir -p /u01/app/19.0.0/grid
mkdir -p /u01/app/oracle/product/19.0.0/db_1
chown -R oracle:oinstall /u01
chmod -R 775 /u01/
```

以“oracle”用户身份登录，并将以下行添加到“/home/oracle/.bash_profile”文件的末尾。
```console
# Oracle Settings
export TMP=/tmp
export TMPDIR=$TMP

export ORACLE_HOSTNAME=ol7-19c-rac1
export ORACLE_UNQNAME=CDBRAC
export ORACLE_BASE=/u01/app/oracle
export GRID_HOME=/u01/app/19.0.0/grid
export DB_HOME=$ORACLE_BASE/product/19.0.0/db_1
export ORACLE_HOME=$DB_HOME
export ORACLE_SID=cdbrac1
export ORACLE_TERM=xterm
export BASE_PATH=/usr/sbin:$PATH
export PATH=$ORACLE_HOME/bin:$BASE_PATH

export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib

alias grid_env='. /home/oracle/grid_env'
alias db_env='. /home/oracle/db_env'
```

创建具有以下内容的名为“/home/oracle/grid_env”的文件。
```console
export ORACLE_SID=+ASM1
export ORACLE_HOME=$GRID_HOME
export PATH=$ORACLE_HOME/bin:$BASE_PATH

export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
```

创建具有以下内容的名为“/home/oracle/db_env”的文件。
```console
export ORACLE_SID=cdbrac1
export ORACLE_HOME=$DB_HOME
export PATH=$ORACLE_HOME/bin:$BASE_PATH

export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
```

一旦运行了“/home/oracle/.bash_profile”，您就可以在环境之间进行如下切换。
```console
$ grid_env
$ echo $ORACLE_HOME
/u01/app/19.0.0/grid
$ db_env
$ echo $ORACLE_HOME
/u01/app/oracle/product/19.0.0/db_1
$
```

我们进行了很多更改，因此此时值得重新启动VM，以确保所有更改都已生效。
```console
# shutdown -r now
```

## 安装访客附加

单击VM屏幕顶部的“设备>安装来宾添加项”菜单选项。 如果您选择了自动运行，请选择它。 如果没有，请运行以下命令。
```console
cd /run/media/VBOXADDITIONS*
sh ./VBoxLinuxAdditions.run
```

将“oracle”用户添加到“vboxsf”组中，以便它可以访问共享驱动器。
```console
# usermod -G vboxsf,dba oracle
# id oracle
uid=54321(oracle) gid=54321(oinstall) groups=54321(oinstall),54322(dba),54323(vboxsf)
#
```
在虚拟机上创建一个共享文件夹（设备>共享文件夹），指向主机上解压缩Oracle软件的目录。 在单击“确定”按钮之前，请检查“自动安装”和“永久”选项。

需要重新启动VM，才能正确使用来宾添加。 下一部分需要关闭，因此此时无需其他重新启动。 VM重新启动后，“oracle”用户将可以访问名为“/run//media/sf_19.0.0”的共享文件夹。
## 创建共享磁盘

使用以下命令关闭“ol7-19c-rac1”虚拟机。
```console
# shutdown -h now
```
On the host server, create 4 sharable virtual disks and associate them as virtual media using the following commands. You can pick a different location, but make sure they are outside the existing VM directory.
```console
$ mkdir -p /u04/VirtualBox/ol7-19c-rac
$ cd /u04/VirtualBox/ol7-19c-rac
$
$ # Create the disks and associate them with VirtualBox as virtual media.
$ VBoxManage createhd --filename asm1.vdi --size 10240 --format VDI --variant Fixed
$ VBoxManage createhd --filename asm2.vdi --size 10240 --format VDI --variant Fixed
$ VBoxManage createhd --filename asm3.vdi --size 10240 --format VDI --variant Fixed
$ VBoxManage createhd --filename asm4.vdi --size 10240 --format VDI --variant Fixed
$
$ # Connect them to the VM.
$ VBoxManage storageattach ol7-19c-rac1 --storagectl "SATA" --port 1 --device 0 --type hdd \
    --medium asm1.vdi --mtype shareable
$ VBoxManage storageattach ol7-19c-rac1 --storagectl "SATA" --port 2 --device 0 --type hdd \
    --medium asm2.vdi --mtype shareable
$ VBoxManage storageattach ol7-19c-rac1 --storagectl "SATA" --port 3 --device 0 --type hdd \
    --medium asm3.vdi --mtype shareable
$ VBoxManage storageattach ol7-19c-rac1 --storagectl "SATA" --port 4 --device 0 --type hdd \
    --medium asm4.vdi --mtype shareable
$
$ # Make shareable.
$ VBoxManage modifyhd asm1.vdi --type shareable
$ VBoxManage modifyhd asm2.vdi --type shareable
$ VBoxManage modifyhd asm3.vdi --type shareable
$ VBoxManage modifyhd asm4.vdi --type shareable

```
If you are using a Windows host, you will have to modify the paths, but the process is the same.
```console
C:
mkdir C:\VirtualBox\ol7-19c-rac
cd C:\VirtualBox\ol7-19c-rac

"c:\Program Files\Oracle\VirtualBox\VBoxManage" createhd --filename asm1.vdi --size 10240 --format VDI --variant Fixed
"c:\Program Files\Oracle\VirtualBox\VBoxManage" createhd --filename asm2.vdi --size 10240 --format VDI --variant Fixed
"c:\Program Files\Oracle\VirtualBox\VBoxManage" createhd --filename asm3.vdi --size 10240 --format VDI --variant Fixed
"c:\Program Files\Oracle\VirtualBox\VBoxManage" createhd --filename asm4.vdi --size 10240 --format VDI --variant Fixed

"c:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach ol7-19c-rac1 --storagectl "SATA" --port 1 --device 0 --type hdd --medium asm1.vdi --mtype shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach ol7-19c-rac1 --storagectl "SATA" --port 2 --device 0 --type hdd --medium asm2.vdi --mtype shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach ol7-19c-rac1 --storagectl "SATA" --port 3 --device 0 --type hdd --medium asm3.vdi --mtype shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach ol7-19c-rac1 --storagectl "SATA" --port 4 --device 0 --type hdd --medium asm4.vdi --mtype shareable

"c:\Program Files\Oracle\VirtualBox\VBoxManage" modifyhd asm1.vdi --type shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" modifyhd asm2.vdi --type shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" modifyhd asm3.vdi --type shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" modifyhd asm4.vdi --type shareable
```
Start the "ol7-19c-rac1" virtual machine by clicking the "Start" button on the toolbar. When the server has started, log in as the root user so you can configure the shared disks. The current disks can be seen by issuing the following commands.
```console
# cd /dev
# ls sd*
sda  sda1  sda2  sdb  sdc  sdd  sde
#
```
Use the "fdisk" command to partition the disks sdb to sde. The following output shows the expected fdisk output for the sdb disk.
```console
# fdisk /dev/sdb
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0x14a4629c.

Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1): 
First sector (2048-41943039, default 2048): 
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-41943039, default 41943039): 
Using default value 41943039
Partition 1 of type Linux and of size 20 GiB is set

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
#
```
In each case, the sequence of answers is "n", "p", "1", "Return", "Return" and "w".
Once all the disks are partitioned, the results can be seen by repeating the previous "ls" command.
```console
# cd /dev
# ls sd*
sda  sda1  sda2  sdb  sdb1  sdc  sdc1  sdd  sdd1  sde  sde1
#
```
Configure your UDEV rules, as shown [here](https://oracle-base.com/articles/linux/udev-scsi-rules-configuration-in-oracle-linux).
Add the following to the "/etc/scsi_id.config" file to configure SCSI devices as trusted. Create the file if it doesn't already exist.
```console
options=-g
```
The SCSI ID of my disks are displayed below.
```console
# /usr/lib/udev/scsi_id -g -u -d /dev/sdb1
1ATA_VBOX_HARDDISK_VB189c7a69-689f61b0
# /usr/lib/udev/scsi_id -g -u -d /dev/sdc1
1ATA_VBOX_HARDDISK_VBc4ae174e-fc756d12
# /usr/lib/udev/scsi_id -g -u -d /dev/sdd1
1ATA_VBOX_HARDDISK_VBa4e03079-ae751cbd
# /usr/lib/udev/scsi_id -g -u -d /dev/sde1
1ATA_VBOX_HARDDISK_VBf00747dc-10252f06
#
```
Using these values, edit the "/etc/udev/rules.d/99-oracle-asmdevices.rules" file adding the following 4 entries. All parameters for a single entry must be on the same line.
```console
KERNEL=="sd?1", SUBSYSTEM=="block", PROGRAM=="/usr/lib/udev/scsi_id -g -u -d /dev/$parent", RESULT=="1ATA_VBOX_HARDDISK_VB189c7a69-689f61b0", SYMLINK+="oracleasm/asm-disk1", OWNER="oracle", GROUP="dba", MODE="0660"
KERNEL=="sd?1", SUBSYSTEM=="block", PROGRAM=="/usr/lib/udev/scsi_id -g -u -d /dev/$parent", RESULT=="1ATA_VBOX_HARDDISK_VBc4ae174e-fc756d12", SYMLINK+="oracleasm/asm-disk2", OWNER="oracle", GROUP="dba", MODE="0660"
KERNEL=="sd?1", SUBSYSTEM=="block", PROGRAM=="/usr/lib/udev/scsi_id -g -u -d /dev/$parent", RESULT=="1ATA_VBOX_HARDDISK_VBa4e03079-ae751cbd", SYMLINK+="oracleasm/asm-disk3", OWNER="oracle", GROUP="dba", MODE="0660"
KERNEL=="sd?1", SUBSYSTEM=="block", PROGRAM=="/usr/lib/udev/scsi_id -g -u -d /dev/$parent", RESULT=="1ATA_VBOX_HARDDISK_VBf00747dc-10252f06", SYMLINK+="oracleasm/asm-disk4", OWNER="oracle", GROUP="dba", MODE="0660"
```
Load updated block device partition tables.
```console
# /sbin/partprobe /dev/sdb1
# /sbin/partprobe /dev/sdc1
# /sbin/partprobe /dev/sdd1
# /sbin/partprobe /dev/sde1
```
Test the rules are working as expected.
```console
# /sbin/udevadm test /block/sdb/sdb1
```
Reload the UDEV rules and start UDEV.
```console
# /sbin/udevadm control --reload-rules
```
The disks should now be visible and have the correct ownership using the following command. If they are not visible, your UDEV configuration is incorrect and must be fixed before you proceed.
```console
# ls -al /dev/oracleasm/*
lrwxrwxrwx. 1 root root 7 Mar  6 17:41 /dev/oracleasm/asm-disk1 -> ../sdb1
lrwxrwxrwx. 1 root root 7 Mar  6 17:41 /dev/oracleasm/asm-disk2 -> ../sdc1
lrwxrwxrwx. 1 root root 7 Mar  6 17:41 /dev/oracleasm/asm-disk3 -> ../sdd1
lrwxrwxrwx. 1 root root 7 Mar  6 17:41 /dev/oracleasm/asm-disk4 -> ../sde1
#
```
The symbolic links are owned by root, but the devices they point to now have the correct ownership.
```console
# ls -al /dev/sd*1
brw-rw----. 1 root   disk 8,  1 Apr 25 14:11 /dev/sda1
brw-rw----. 1 oracle dba  8, 17 Apr 25 14:11 /dev/sdb1
brw-rw----. 1 oracle dba  8, 33 Apr 25 14:11 /dev/sdc1
brw-rw----. 1 oracle dba  8, 49 Apr 25 14:11 /dev/sdd1
brw-rw----. 1 oracle dba  8, 65 Apr 25 14:11 /dev/sde1
#
```
The shared disks are now configured for the grid infrastructure.

## Clone the Virtual Machine
Later versions of VirtualBox allow you to clone VMs, but these also attempt to clone the shared disks, which is not what we want. Instead we must manually clone the VM.
Shut down the "ol7-19c-rac1" virtual machine using the following command.
```console
# shutdown -h now
```
Manually clone the "ol7-19c-rac1.vdi" disk using the following commands on the host server.
```console
$ mkdir -p /u03/VirtualBox/ol7-19c-rac2
$ VBoxManage clonehd /u01/VirtualBox/ol7-19c-rac1/ol7-19c-rac1.vdi /u03/VirtualBox/ol7-19c-rac2/ol7-19c-rac2.vdi

Rem Windows
mkdir "C:\VirtualBox\ol7-19c-rac2"
"c:\Program Files\Oracle\VirtualBox\VBoxManage" clonehd "C:\VirtualBox\ol7-19c-rac1\ol7-19c-rac1.vdi" "C:\VirtualBox\ol7-19c-rac2\ol7-19c-rac2.vdi"
```
Create the "ol7-19c-rac2" virtual machine in VirtualBox in the same way as you did for "ol7-19c-rac1", with the exception of using an existing "ol7-19c-rac2.vdi" virtual hard drive.

Remember to add the three network adaptor as you did on the "ol7-19c-rac1" VM. When the VM is created, attach the shared disks to this VM.
```console
$ # Linux : Switch to the shared storage location and attach them.
$ cd /u04/VirtualBox/ol7-19c-rac
$
$ VBoxManage storageattach ol7-19c-rac2 --storagectl "SATA" --port 1 --device 0 --type hdd --medium asm1.vdi --mtype shareable
$ VBoxManage storageattach ol7-19c-rac2 --storagectl "SATA" --port 2 --device 0 --type hdd --medium asm2.vdi --mtype shareable
$ VBoxManage storageattach ol7-19c-rac2 --storagectl "SATA" --port 3 --device 0 --type hdd --medium asm3.vdi --mtype shareable
$ VBoxManage storageattach ol7-19c-rac2 --storagectl "SATA" --port 4 --device 0 --type hdd --medium asm4.vdi --mtype shareable


Rem Windows : Switch to the shared storage location and attach them.
C:
cd C:\VirtualBox\ol7-19c-rac

"c:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach ol7-19c-rac2 --storagectl "SATA" --port 1 --device 0 --type hdd --medium asm1.vdi --mtype shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach ol7-19c-rac2 --storagectl "SATA" --port 2 --device 0 --type hdd --medium asm2.vdi --mtype shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach ol7-19c-rac2 --storagectl "SATA" --port 3 --device 0 --type hdd --medium asm3.vdi --mtype shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach ol7-19c-rac2 --storagectl "SATA" --port 4 --device 0 --type hdd --medium asm4.vdi --mtype shareable
```
Start the "ol7-19c-rac2" virtual machine by clicking the "Start" button on the toolbar. Ignore any network errors during the startup.

Log in to the "ol7-19c-rac2" virtual machine as the "root" user so we can reconfigure the network settings to match the following.
- hostname: ol7-19c-rac2.localdomain
- enp0s3 (eth0): DHCP (*Not* Connect Automatically)
- enp0s8 (eth1): IP=192.168.56.102, Subnet=255.255.255.0, Gateway=192.168.56.1, DNS=192.168.56.1, Search=localdomain (Connect Automatically)
- enp0s9 (eth2): IP=192.168.1.102, Subnet=255.255.255.0, Gateway=<blank>, DNS=<blank>, Search=<blank> (Connect Automatically)

Amend the hostname in the "/etc/hostname" file.
```console
ol7-19c-rac2.localdomain
```
Unlike previous Linux versions, we shouldn't have to edit the MAC address associated with the network adapters, but we will have to alter their IP addresses.

Edit the "/etc/sysconfig/network-scripts/ifcfg-enp0s8" (eth1), amending only the IPADDR settings as follows and deleting the UUID entry.
```console
IPADDR=192.168.56.102 
```
Edit the "/etc/sysconfig/network-scripts/ifcfg-enp0s9" (eth2), amending only the IPADDR settings as follows and deleting the UUID entry.
```console
IPADDR=192.168.1.102 
```
Restart the virtual machines.
```console
# shutdown -r now
```
At this point, the networking for the second node should look something like the following. Notice that eth0 has no associated IP address because it is disabled.
```console
# ifconfig
enp0s3 : flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 08:00:27:dc:7c:74  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp0s8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.56.102  netmask 255.255.255.0  broadcast 192.168.56.255
        inet6 fe80::a00:27ff:fed9:c89a  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:d9:c8:9a  txqueuelen 1000  (Ethernet)
        RX packets 197  bytes 19460 (19.0 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 178  bytes 27171 (26.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp0s9: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.102  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::a00:27ff:feb4:6bf  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:b4:06:bf  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 30  bytes 4112 (4.0 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 0  (Local Loopback)
        RX packets 4  bytes 420 (420.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 4  bytes 420 (420.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

#
```
Edit the "/home/oracle/.bash_profile" file on the "ol7-19c-rac2" node to correct the ORACLE_SID and ORACLE_HOSTNAME values.
```console
export ORACLE_SID=cdbrac2
export ORACLE_HOSTNAME=ol7-19c-rac2.localdomain
```
Also, amend the ORACLE_SID setting in the "/home/oracle/db_env" and "/home/oracle/grid_env" files.

Restart the "ol7-19c-rac2" virtual machine and start the "ol7-19c-rac1" virtual machine. When both nodes have started, check they can both ping all the public and private IP addresses using the following commands.
```console
ping -c 3 ol7-19c-rac1
ping -c 3 ol7-19c-rac1-priv
ping -c 3 ol7-19c-rac2
ping -c 3 ol7-19c-rac2-priv
```
 Check the SCAN address is still being resolved properly on both nodes.
```console
# nslookup ol7-19c-scan
Server:		192.168.56.1
Address:	192.168.56.1#53

Name:	ol7-19c-scan.localdomain
Address: 192.168.56.105

#
```
At this point the virtual IP addresses defined in the "/etc/hosts" file will not work, so don't bother testing them.
Check the UDEV rules are working on both machines.
```console
# ls -al /dev/oracleasm/*
lrwxrwxrwx. 1 root root 7 Sep 18 08:19 /dev/oracleasm/asm-disk1 -> ../sdb1
lrwxrwxrwx. 1 root root 7 Sep 18 08:19 /dev/oracleasm/asm-disk2 -> ../sdc1
lrwxrwxrwx. 1 root root 7 Sep 18 08:19 /dev/oracleasm/asm-disk3 -> ../sdd1
lrwxrwxrwx. 1 root root 7 Sep 18 08:19 /dev/oracleasm/asm-disk4 -> ../sde1
#
```
Prior to 11gR2 we would probably use the "runcluvfy.sh" utility in the clusterware root directory to check the prerequisites have been met. If you are intending to configure SSH connectivity using the installer this check should be omitted as it will always fail. If you want to [setup SSH connectivity manually](https://oracle-base.com/articles/linux/user-equivalence-configuration-on-linux), then once it is done you can run the "runcluvfy.sh" with the following command.
```console
/mountpoint/clusterware/runcluvfy.sh stage -pre crsinst -n ol7-19c-rac1,ol7-19c-rac2 -verbose
```
If you get any failures be sure to correct them before proceeding.
The virtual machine setup is now complete.
Before moving forward you should probably shut down your VMs and take snapshots of them. If any failures happen beyond this point it is probably better to switch back to those snapshots, clean up the shared drives and start the grid installation again. An alternative to [cleaning up the shared disks](https://oracle-base.com/articles/rac/clean-up-a-failed-grid-infrastructure-installation#asm_disks) is to back them up now using zip and just replace them in the event of a failure.
```console
$ # Linux
$ cd /u04/VirtualBox/ol7-19c-rac
$ zip PreGrid.zip *.vdi

Rem Windows
C:
cd C:\VirtualBox\ol7-19c-rac
zip PreGrid.zip *.vdi
```

##  Install the Grid Infrastructure

Make sure both virtual machines are started. The GI is now an image installation, so perform the following on the first node as the "oracle" user.
```console
export SOFTWARE_LOCATION=/media/sf_19.0.0/
cd /u01/app/19.0.0/grid
unzip -q $SOFTWARE_LOCATION/linuxx64_1900_grid_home.zip
```
or unzip the grid software to target directory on the first node. Don’t do this on the second node.
```console
unzip -d /u01/app/19.0.0/grid V982068-01.zip
```
Install the following package from the grid home as the "root" user on all nodes.
```console
su -
# Local node.
cd /u01/app/19.0.0/grid/cv/rpm
rpm -Uvh cvuqdisk*

# Remote node.
scp ./cvuqdisk* root@ol7-19c-rac2:/tmp
ssh root@ol7-19c-rac2 rpm -Uvh /tmp/cvuqdisk*
exit
```
If you were planning on using the AFD Driver (the new ASMLib) you would configure the shared disks using the asmcmd command as shown below. We are using UDEV, so this is not necessary.
```console
# !!!! I did not do this! !!!!
su -

# Set environment.
export ORACLE_HOME=/u01/app/19.0.0/grid
export ORACLE_BASE=/tmp

# Mark disks.
$ORACLE_HOME/bin/asmcmd afd_label DISK1 /dev/oracleasm/asm-disk1 --init
$ORACLE_HOME/bin/asmcmd afd_label DISK2 /dev/oracleasm/asm-disk2 --init
$ORACLE_HOME/bin/asmcmd afd_label DISK3 /dev/oracleasm/asm-disk3 --init
$ORACLE_HOME/bin/asmcmd afd_label DISK4 /dev/oracleasm/asm-disk4 --init

# Test Disks.
$ORACLE_HOME//bin/asmcmd afd_lslbl /dev/oracleasm/asm-disk1
$ORACLE_HOME//bin/asmcmd afd_lslbl /dev/oracleasm/asm-disk2
$ORACLE_HOME//bin/asmcmd afd_lslbl /dev/oracleasm/asm-disk3
$ORACLE_HOME//bin/asmcmd afd_lslbl /dev/oracleasm/asm-disk4

# unset environment.
unset ORACLE_BASE

exit
```
Configure the Grid Infrastructure by running the following as the "oracle" user.

I could have run the configuration in silent mode using this edited response file (grid_config.rsp) with the following command.
```console
cd /u01/app/19.0.0/grid
./gridSetup.sh -silent -responseFile /tmp/grid_config.rsp
```
Instead, here's the interactive configuration.
```console
cd /u01/app/19.0.0/grid
./gridSetup.sh
```
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-184019@2x.jpg "Install the Grid Infrastructure")
Select the "Configure Oracle Grid Infrastructure for a New Cluster" option, then click the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-184049@2x.jpg "Install the Grid Infrastructure")
Accept the "Configure an Oracle Standalone Cluster" option by clicking the "Next" button.

Enter the cluster name "ol7-19c-cluster", SCAN name "ol7-19c-scan" and SCAN port "1521", then click the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-184214@2x.jpg "Install the Grid Infrastructure")
On the "Cluster Node Information" screen, click the "Add" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-184312@2x.jpg "Install the Grid Infrastructure")
Click the "SSH Connectivity..." button and enter the password for the "oracle" user. Click the "Setup" button to configure SSH connectivity, and the "Test" button to test it once it is complete. Once the test is complete, click the "Next" button.

Check the public and private networks are specified correctly. Make sure enp0s9 are used for “ASM & Private”, If the NAT interface is displayed, remember to mark it as "Do Not Use". Click the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-184917@2x.jpg "Install the Grid Infrastructure")
Accept the "Use Oracle Flex ASM for storage" option by clicking the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-184858@2x.jpg "Install the Grid Infrastructure")
Select the "No" option, as we don't want to create a separate disk group for the GIMR in this case. Click the "Next" button.

Set the redundancy to "External", click the "Change Discovery Path" button and set the path to "/dev/oracleasm/*". Return to the main screen and select all 4 disks. Uncheck the "Configure Oracle ASM Filter Driver" option, then click the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-185004@2x.jpg "Install the Grid Infrastructure")
Enter the credentials and click the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-185024@2x.jpg "Install the Grid Infrastructure")
Accept the default IPMI option by clicking the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-185035@2x.jpg "Install the Grid Infrastructure")
Don't register with EM. Click the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-185045@2x.jpg "Install the Grid Infrastructure")
We are using a single user and group manage both ASM add the database, so set the groups to "dba" and click the "Next" button. Accept the warnings on the subsequent dialog by clicking the "Yes" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-185132@2x.jpg "Install the Grid Infrastructure")
Enter the Oracle Base location "/u01/app/oracle" and click the "Next" button. We have already pre-created directories for the later database installation, so ignore the subsequent warning about the Oracle Base not being empty by clicking the "Yes" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-185158@2x.jpg "Install the Grid Infrastructure")
Accept the default inventory directory by clicking the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-185518@2x.jpg "Install the Grid Infrastructure")
Wait while the prerequisite checks complete. If you have any issues use the "Fix & Check Again" button. Once possible fixes are complete, check the "Ignore All" checkbox and click the "Next" button. It is likely the "Physical Memory" and "Network Time Protocol (NTP)" tests will fail for this type of installation. This is OK.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-185737@2x.jpg "Install the Grid Infrastructure")
By check “Ignore All” to proceed the installation.

Wait while the installation takes place.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-190342@2x.jpg "Install the Grid Infrastructure")
When prompted, run the configuration scripts on each node.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-190914@2x.jpg "Install the Grid Infrastructure")
The output from the "orainstRoot.sh" file should look something like that listed below.
```console
[root@ol7-19c-rac1 ~]# /u01/app/oraInventory/orainstRoot.sh 
Changing permissions of /u01/app/oraInventory.
Adding read,write permissions for group.
Removing read,write,execute permissions for world.

Changing groupname of /u01/app/oraInventory to oinstall.
The execution of the script is complete.
[root@ol7-19c-rac1 ~]#
```

The output of the "root.sh" will vary a little depending on the node it is run on. Example output can be seen here.
```console
[root@ol7-19c-rac1 ~]# /u01/app/19.0.0/grid/root.sh
Performing root user operation.

The following environment variables are set as:
    ORACLE_OWNER= oracle
    ORACLE_HOME=  /u01/app/19.0.0/grid

Enter the full pathname of the local bin directory: [/usr/local/bin]: 
   Copying dbhome to /usr/local/bin ...
   Copying oraenv to /usr/local/bin ...
   Copying coraenv to /usr/local/bin ...


Creating /etc/oratab file...
Entries will be added to the /etc/oratab file as needed by
Database Configuration Assistant when a database is created
Finished running generic part of root script.
Now product-specific root actions will be performed.
Relinking oracle with rac_on option
Using configuration parameter file: /u01/app/19.0.0/grid/crs/install/crsconfig_params
The log of current session can be found at:
  /u01/app/oracle/crsdata/ol7-19c-rac1/crsconfig/rootcrs_ol7-19c-rac1_2019-11-19_06-10-11AM.log
2019/11/19 06:10:25 CLSRSC-594: Executing installation step 1 of 19: 'SetupTFA'.
2019/11/19 06:10:25 CLSRSC-594: Executing installation step 2 of 19: 'ValidateEnv'.
2019/11/19 06:10:25 CLSRSC-363: User ignored prerequisites during installation
2019/11/19 06:10:25 CLSRSC-594: Executing installation step 3 of 19: 'CheckFirstNode'.
2019/11/19 06:10:27 CLSRSC-594: Executing installation step 4 of 19: 'GenSiteGUIDs'.
2019/11/19 06:10:29 CLSRSC-594: Executing installation step 5 of 19: 'SetupOSD'.
2019/11/19 06:10:29 CLSRSC-594: Executing installation step 6 of 19: 'CheckCRSConfig'.
2019/11/19 06:10:30 CLSRSC-594: Executing installation step 7 of 19: 'SetupLocalGPNP'.
2019/11/19 06:11:14 CLSRSC-594: Executing installation step 8 of 19: 'CreateRootCert'.
2019/11/19 06:11:22 CLSRSC-594: Executing installation step 9 of 19: 'ConfigOLR'.
2019/11/19 06:11:31 CLSRSC-4002: Successfully installed Oracle Trace File Analyzer (TFA) Collector.
2019/11/19 06:11:46 CLSRSC-594: Executing installation step 10 of 19: 'ConfigCHMOS'.
2019/11/19 06:11:46 CLSRSC-594: Executing installation step 11 of 19: 'CreateOHASD'.
2019/11/19 06:11:55 CLSRSC-594: Executing installation step 12 of 19: 'ConfigOHASD'.
2019/11/19 06:11:55 CLSRSC-330: Adding Clusterware entries to file 'oracle-ohasd.service'
2019/11/19 06:12:20 CLSRSC-594: Executing installation step 13 of 19: 'InstallAFD'.
2019/11/19 06:12:29 CLSRSC-594: Executing installation step 14 of 19: 'InstallACFS'.
2019/11/19 06:12:37 CLSRSC-594: Executing installation step 15 of 19: 'InstallKA'.
2019/11/19 06:12:45 CLSRSC-594: Executing installation step 16 of 19: 'InitConfig'.

ASM has been created and started successfully.

[DBT-30001] Disk groups created successfully. Check /u01/app/oracle/cfgtoollogs/asmca/asmca-191119AM061319.log for details.

2019/11/19 06:14:25 CLSRSC-482: Running command: '/u01/app/19.0.0/grid/bin/ocrconfig -upgrade oracle oinstall'
CRS-4256: Updating the profile
Successful addition of voting disk 82896a1404774fa1bf7404bea502e526.
Successfully replaced voting disk group with +DATA.
CRS-4256: Updating the profile
CRS-4266: Voting file(s) successfully replaced
##  STATE    File Universal Id                File Name Disk group
--  -----    -----------------                --------- ---------
 1. ONLINE   82896a1404774fa1bf7404bea502e526 (/dev/sdb1) [DATA]
Located 1 voting disk(s).
2019/11/19 06:16:33 CLSRSC-594: Executing installation step 17 of 19: 'StartCluster'.
2019/11/19 06:18:09 CLSRSC-343: Successfully started Oracle Clusterware stack
2019/11/19 06:18:09 CLSRSC-594: Executing installation step 18 of 19: 'ConfigNode'.
2019/11/19 06:22:17 CLSRSC-594: Executing installation step 19 of 19: 'PostConfig'.
2019/11/19 06:23:13 CLSRSC-325: Configure Oracle Grid Infrastructure for a Cluster ... succeeded
[root@ol7-19c-rac1 ~]# su - oracle
Last login: Tue Nov 19 06:23:11 EST 2019 on pts/1
ss[oracle@ol7-19c-rac1 ~]$ ssh ol7-19c-rac2
Last login: Tue Nov 19 06:09:27 2019 from ol7-19c-rac1
[oracle@ol7-19c-rac2 ~]$ su - 
Password: 
Last login: Tue Nov 19 06:09:30 EST 2019 on pts/1
[root@ol7-19c-rac2 ~]# /u01/app/19.0.0/grid/root.sh
Performing root user operation.

The following environment variables are set as:
    ORACLE_OWNER= oracle
    ORACLE_HOME=  /u01/app/19.0.0/grid

Enter the full pathname of the local bin directory: [/usr/local/bin]: 
   Copying dbhome to /usr/local/bin ...
   Copying oraenv to /usr/local/bin ...
   Copying coraenv to /usr/local/bin ...


Creating /etc/oratab file...
Entries will be added to the /etc/oratab file as needed by
Database Configuration Assistant when a database is created
Finished running generic part of root script.
Now product-specific root actions will be performed.
Relinking oracle with rac_on option
Using configuration parameter file: /u01/app/19.0.0/grid/crs/install/crsconfig_params
The log of current session can be found at:
  /u01/app/oracle/crsdata/ol7-19c-rac2/crsconfig/rootcrs_ol7-19c-rac2_2019-11-19_06-24-56AM.log
2019/11/19 06:25:07 CLSRSC-594: Executing installation step 1 of 19: 'SetupTFA'.
2019/11/19 06:25:07 CLSRSC-594: Executing installation step 2 of 19: 'ValidateEnv'.
2019/11/19 06:25:07 CLSRSC-363: User ignored prerequisites during installation
2019/11/19 06:25:07 CLSRSC-594: Executing installation step 3 of 19: 'CheckFirstNode'.
2019/11/19 06:25:09 CLSRSC-594: Executing installation step 4 of 19: 'GenSiteGUIDs'.
2019/11/19 06:25:10 CLSRSC-594: Executing installation step 5 of 19: 'SetupOSD'.
2019/11/19 06:25:10 CLSRSC-594: Executing installation step 6 of 19: 'CheckCRSConfig'.
2019/11/19 06:25:11 CLSRSC-594: Executing installation step 7 of 19: 'SetupLocalGPNP'.
2019/11/19 06:25:14 CLSRSC-594: Executing installation step 8 of 19: 'CreateRootCert'.
2019/11/19 06:25:14 CLSRSC-594: Executing installation step 9 of 19: 'ConfigOLR'.
2019/11/19 06:25:33 CLSRSC-594: Executing installation step 10 of 19: 'ConfigCHMOS'.
2019/11/19 06:25:34 CLSRSC-594: Executing installation step 11 of 19: 'CreateOHASD'.
2019/11/19 06:25:48 CLSRSC-594: Executing installation step 12 of 19: 'ConfigOHASD'.
2019/11/19 06:25:51 CLSRSC-330: Adding Clusterware entries to file 'oracle-ohasd.service'
2019/11/19 06:26:36 CLSRSC-4002: Successfully installed Oracle Trace File Analyzer (TFA) Collector.
2019/11/19 06:26:41 CLSRSC-594: Executing installation step 13 of 19: 'InstallAFD'.
2019/11/19 06:26:44 CLSRSC-594: Executing installation step 14 of 19: 'InstallACFS'.
2019/11/19 06:26:48 CLSRSC-594: Executing installation step 15 of 19: 'InstallKA'.
2019/11/19 06:26:52 CLSRSC-594: Executing installation step 16 of 19: 'InitConfig'.
2019/11/19 06:27:12 CLSRSC-594: Executing installation step 17 of 19: 'StartCluster'.
2019/11/19 06:28:23 CLSRSC-343: Successfully started Oracle Clusterware stack
2019/11/19 06:28:23 CLSRSC-594: Executing installation step 18 of 19: 'ConfigNode'.
2019/11/19 06:29:24 CLSRSC-594: Executing installation step 19 of 19: 'PostConfig'.
2019/11/19 06:29:56 CLSRSC-325: Configure Oracle Grid Infrastructure for a Cluster ... succeeded
[root@ol7-19c-rac2 ~]#
```

Once the scripts have completed, return to the "Execute Configuration Scripts" screen on "ol7-19c-rac1" and click the "OK" button.

Wait for the configuration assistants to complete.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-193235@2x.jpg "Install the Grid Infrastructure")
If any of the configuration steps fail you should check the specified log to see if the error is a show-stopper or not. 

Provided you don't have any show-stoppers, it is safe to ignore the errors by clicking "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-193606@2x.jpg "Install the Grid Infrastructure")
Click the "Close" button to exit the installer.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-193620@2x.jpg "Install the Grid Infrastructure")
The grid infrastructure installation is now complete. We can check the status of the installation using the following commands.
```console
[root@ol7-19c-rac1 ~]# /u01/app/19.0.0/grid/bin/crsctl stat res -t
--------------------------------------------------------------------------------
Name           Target  State        Server                   State details       
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.LISTENER.lsnr
               ONLINE  ONLINE       ol7-19c-rac1             STABLE
               ONLINE  ONLINE       ol7-19c-rac2             STABLE
ora.chad
               ONLINE  ONLINE       ol7-19c-rac1             STABLE
               ONLINE  ONLINE       ol7-19c-rac2             STABLE
ora.net1.network
               ONLINE  ONLINE       ol7-19c-rac1             STABLE
               ONLINE  ONLINE       ol7-19c-rac2             STABLE
ora.ons
               ONLINE  ONLINE       ol7-19c-rac1             STABLE
               ONLINE  ONLINE       ol7-19c-rac2             STABLE
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.ASMNET1LSNR_ASM.lsnr(ora.asmgroup)
      1        ONLINE  ONLINE       ol7-19c-rac1             STABLE
      2        ONLINE  ONLINE       ol7-19c-rac2             STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.DATA.dg(ora.asmgroup)
      1        ONLINE  ONLINE       ol7-19c-rac1             STABLE
      2        ONLINE  ONLINE       ol7-19c-rac2             STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.LISTENER_SCAN1.lsnr
      1        ONLINE  ONLINE       ol7-19c-rac1             STABLE
ora.asm(ora.asmgroup)
      1        ONLINE  ONLINE       ol7-19c-rac1             Started,STABLE
      2        ONLINE  ONLINE       ol7-19c-rac2             Started,STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.asmnet1.asmnetwork(ora.asmgroup)
      1        ONLINE  ONLINE       ol7-19c-rac1             STABLE
      2        ONLINE  ONLINE       ol7-19c-rac2             STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.cvu
      1        ONLINE  ONLINE       ol7-19c-rac1             STABLE
ora.ol7-19c-rac1.vip
      1        ONLINE  ONLINE       ol7-19c-rac1             STABLE
ora.ol7-19c-rac2.vip
      1        ONLINE  ONLINE       ol7-19c-rac2             STABLE
ora.qosmserver
      1        ONLINE  ONLINE       ol7-19c-rac1             STABLE
ora.scan1.vip
      1        ONLINE  ONLINE       ol7-19c-rac1             STABLE
--------------------------------------------------------------------------------
[root@ol7-19c-rac1 ~]#
```

At this point it is probably a good idea to shutdown both VMs and take snapshots. Remember to make a fresh zip of the ASM disks on the host machine, which you will need to restore if you revert to the post-grid snapshots.
```console
$ cd /u04/VirtualBox/ol7-19c-rac
$ zip PostGrid.zip *.vdi
```

## Install the Database Software
Make sure the "ol7-19c-rac1" and "ol7-19c-rac2" virtual machines are started, then login to "ol7-19c-rac1" as the oracle user and unzip the database software to target directory on the first node. Don’t do this on the second node.
```console
unzip -d /u01/app/oracle/product/19.0.0/db_1 V982063-01.zip
```
Then start the Oracle installer. Check that all services are up using "crsctl stat res -t", as described before.
```console
$ db_env
$ cd $ORACLE_HOME
$ ./runInstaller
```
Select the "Set Up Software Only" option, then click the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-194047@2x.jpg "Install the Database Software")
Accept the "Oracle Real Application Clusters database installation" option by clicking the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-194138@2x.jpg "Install the Database Software")
Make sure both nodes are selected, then click the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-194210@2x.jpg "Install the Database Software")
Select the "Enterprise Edition" option, then click the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-194630@2x.jpg "Install the Database Software")
Enter "/u01/app/oracle" as the Oracle base, then click the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-194639@2x.jpg "Install the Database Software")
Select the desired operating system groups, then click the "Next" button. In this case we are only using the "dba" group.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-194725@2x.jpg "Install the Database Software")
Accept the default options, and click “Next” button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-194734@2x.jpg "Install the Database Software")
Wait for the prerequisite check to complete. If there are any problems either click the "Fix & Check Again" button, or check the "Ignore All" checkbox and click the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-195031@2x.jpg "Install the Database Software")

![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-195100@2x.jpg "Install the Database Software")
If you are happy with the summary information, click the "Install" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-195114@2x.jpg "Install the Database Software")
Wait while the installation takes place.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-195943@2x.jpg "Install the Database Software")
When prompted, run the configuration script on each node. When the scripts have been run on each node, click the "OK" button.

Click the "Close" button to exit the installer.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-200437@2x.jpg "Install the Database Software")
Shutdown both VMs and take snapshots. Remember to make a fresh zip of the ASM disks on the host machine, which you will need to restore if you revert to the post-db snapshots.
```console
$ cd /u04/VirtualBox/ol7-19c-rac
$ zip PostDB.zip *.vdi
```

## Create a Database
Make sure the "ol7-19c-rac1" and "ol7-19c-rac2" virtual machines are started, then login to "ol7-19c-rac1" as the oracle user and start the Database Creation Asistant (DBCA).
```console
$ db_env
$ dbca
```
Select the "Create Database" option and click the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-200513@2x.jpg "Create a Database")
Select the "Advanced Mode" option. Click the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-200545@2x.jpg "Create a Database")
Check the "Custom Database" option and click the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-200603@2x.jpg "Create a Database")
Make sure both nodes are selected, then click the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-200615@2x.jpg "Create a Database")
Enter the container database name (cdbrac), pluggable database name (pdb) and administrator password. Click the "Next" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-200740@2x.jpg "Create a Database")
Accept the default values, and click “Next” button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-200818@2x.jpg "Create a Database")
Accept the default values, and click “Next” button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-200859@2x.jpg "Create a Database")
Accept the default values, and click “Next” button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-200942@2x.jpg "Create a Database")
Accept the default values, and click “Next” button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-201022@2x.jpg "Create a Database")

![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-201044@2x.jpg "Create a Database")
Deselect the CVU and EM options, and click “Next” button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-201119@2x.jpg "Create a Database")
Enter dba password, and click “Next” button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-201137@2x.jpg "Create a Database")

![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-201149@2x.jpg "Create a Database")

![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-201216@2x.jpg "Create a Database")

![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-201309@2x.jpg "Create a Database")
Select “Ignore All”, and click “Next” button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-201424@2x.jpg "Create a Database")
If you are happy with the summary information, click the "Finish" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-201435@2x.jpg "Create a Database")
Wait while the database creation takes place.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-202201@2x.jpg "Create a Database")
If you want to modify passwords, click the "Password Management" button. When finished, click the "Close" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-233237@2x.jpg "Create a Database")
The RAC database creation is now complete.

## Check the Status of the RAC
There are several ways to check the status of the RAC. The srvctl utility shows the current configuration and status of the RAC database.
```console
[oracle@ol7-19c-rac1 cdbrac]$ grid_env
[oracle@ol7-19c-rac1 cdbrac]$ srvctl config database -d cdbrac
Database unique name: cdbrac
Database name: cdbrac
Oracle home: /u01/app/oracle/product/19.0.0/db_1
Oracle user: oracle
Spfile: +DATA/CDBRAC/PARAMETERFILE/spfile.275.1024741213
Password file: +DATA/CDBRAC/PASSWORD/pwdcdbrac.258.1024730137
Domain: 
Start options: open
Stop options: immediate
Database role: PRIMARY
Management policy: AUTOMATIC
Server pools: 
Disk Groups: DATA
Mount point paths: 
Services: 
Type: RAC
Start concurrency: 
Stop concurrency: 
OSDBA group: dba
OSOPER group: 
Database instances: cdbrac1,cdbrac2
Configured nodes: ol7-19c-rac1,ol7-19c-rac2
CSS critical: no
CPU count: 0
Memory target: 0
Maximum memory: 0
Default network number for database services: 
Database is administrator managed
[oracle@ol7-19c-rac1 cdbrac]$ srvctl status database -d cdbrac
Instance cdbrac1 is running on node ol7-19c-rac1
Instance cdbrac2 is running on node ol7-19c-rac2
[oracle@ol7-19c-rac1 cdbrac]$
```
The V$ACTIVE_INSTANCES view can also display the current status of the instances.
```console
[oracle@ol7-19c-rac1 cdbrac]$ sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Tue Nov 19 10:34:25 2019
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> SELECT inst_name FROM v$active_instances;

INST_NAME
--------------------------------------------------------------------------------
ol7-19c-rac1:cdbrac1
ol7-19c-rac2:cdbrac2

SQL>
```

## Reference
For more information see:
- [Grid Infrastructure Installation and Upgrade Guide for Linux](https://docs.oracle.com/en/database/oracle/oracle-database/19/cwlin/index.html)  
- [Database Installation Guide for Linux](https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/index.html)
- [Oracle Database 12c Release 2 (12.2) RAC On Oracle Linux 7 Using VirtualBox](https://oracle-base.com/articles/12c/oracle-db-12cr2-rac-installation-on-oracle-linux-7-using-virtualbox)

### End
