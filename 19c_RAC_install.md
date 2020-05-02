# Oracle Database 19c RAC On Oracle Linux 7 Using VirtualBox

**Last update 2019-11-20, published on 2020-5-2**

This article describes the installation of Oracle Database 19c RAC on Linux (Oracle Linux 7.7 64-bit) using VirtualBox (6.0.10) with no additional shared disk devices.


## Introduction

One of the biggest obstacles preventing people from setting up test RAC environments is the requirement for shared storage. In a production environment, shared storage is often provided by a SAN or high-end NAS device, but both of these options are very expensive when all you want to do is get some experience installing and using RAC. A cheaper alternative is to use a FireWire disk enclosure to allow two machines to access the same disk(s), but that still costs money and requires two servers. A third option is to use virtualization to fake the shared storage.

Using VirtualBox you can run multiple Virtual Machines (VMs) on a single server, allowing you to run both RAC nodes on a single machine. In addition, it allows you to set up shared virtual disks, overcoming the obstacle of expensive shared storage.

Before you launch into this installation, here are a few things to consider.
- The finished system includes the host operating system, three guest operating systems, two sets of Oracle Grid Infrastructure (Clusterware + ASM) and two Database instances all on a single machine. As you can imagine, this requires a significant amount of disk space, CPU and memory.
- Following on from the last point, the RAC node VMs will each need at least 6G of RAM, but you will see I used 10G for each, and it was still slow. Don't assume you will be able to run this on a small PC or laptop. You won't.
- This procedure provides a bare bones installation to get the RAC working. There is no redundancy in the Grid Infrastructure installation or the ASM installation. To add this, simply create double the amount of shared disks and select the "Normal" redundancy option when it is offered. Of course, this will take more disk space.
- This is not, and should not be considered, instructions for a production-ready system. It's simply to allow you to see what is required to install RAC and give you a system to experiment with.
- The DNS is required to support the scan listener. In previous releases I suggested running the DNS on the host server, but this is easier.
- This article uses the 64-bit versions of Oracle Linux and Oracle 19c.
- When doing this installation on my server, I split the virtual disks on to different physical disks. This is not necessary, but makes things run a bit faster.
- This will probably take over three hours to complete. Maybe a lot longer if you have severe memory or disk speed limitations.

## Download Software

Download the following software.
- [Oracle Linux 7](http://edelivery.oracle.com/linux) (Use the latest spin eg. 7.7)
- [VirtualBox (6.0.10)](http://www.virtualbox.org/wiki/Downloads)
- [Oracle 19c (19.3) Software (64 bit)](http://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html)

This article has been updated for the 19c release, but the installation is essentially unchanged since 12.2.0.1. Any variations specific for 19c will be noted.

Depending on your version of VirtualBox and Oracle Linux, there may be some slight variation in how the screen shots look.

## VirtualBox Installation

First, install the VirtualBox software. On RHEL and its clones you do this with the following type of command as the root user.

```console
# rpm -Uvh VirtualBox*.rpm
```

The package name will vary depending on the host distribution you are using. Once complete, VirtualBox is started from the menu.

## VirtualBox Network Setup
We need to make sure a host-only network is configured and check/modify the IP range for that network. This will be the public network for our RAC installation.

- Start VirtualBox from the menu.
- Select the "Tools" option.
- Click "Network" in the pop out menu.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113524@2x.jpg "VirtualBox Network Setup")
- Click the "Create" button on the right size of the screen. A network called "vboxnet0" will be created.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113536@2x.jpg "VirtualBox Network Setup")
- If you want to use a different subnet for your public addresses you can change the network details here. Just make sure the subnet you choose doesn't match any real subnets on your network. I've decided to stick with the default, which for me is "192.168.56.X".
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113559@2x.jpg "VirtualBox Network Setup")

## Virtual Machine Setup

Now we must define the two virtual RAC nodes. We can save time by defining one VM, then cloning it when it is installed.

Start VirtualBox and click the "New" button on the toolbar. Enter the name "ol7-19c-rac1", OS "Linux" and Version "Oracle (64 bit)", then click the "Continue" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-112613@2x.jpg "Virtual Machine Setup")
Enter "4096" as the base memory size, then click the "Continue" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-112838@2x.jpg "Virtual Machine Setup")
Accept the default option to create a new virtual hard disk by clicking the "Create" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-112911@2x.jpg "Virtual Machine Setup")
Acccept the default hard drive file type by clicking the "Continue" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-112924@2x.jpg "Virtual Machine Setup")
Acccept the "Dynamically allocated" option by clicking the "Continue" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-112939@2x.jpg "Virtual Machine Setup")
Accept the default location and set the size to "50G", then click the "Create" button. If you can spread the virtual disks onto different physical disks, that will improve performance.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113049@2x.jpg "Virtual Machine Setup")
The "ol7-19c-rac1" VM will appear on the left hand pane. Scroll down the details on the right and click on the "Network" link.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113536@2x.jpg "Virtual Machine Setup")
Make sure "Adapter 1" is enabled, set to "NAT", then click on the "Adapter 2" tab.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113617@2x.jpg "Virtual Machine Setup")
Make sure "Adapter 2" is enabled, set to "Host-only Adapter", then click on the "Adapter 3" tab.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113639@2x.jpg "Virtual Machine Setup")
Make sure "Adapter 3" is enabled, set to "Internal Network", then click on the "System" section.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113708@2x.jpg "Virtual Machine Setup")
Move "Hard Disk" to the top of the boot order and uncheck the "Floppy" option, then click the "OK" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113747@2x.jpg "Virtual Machine Setup")
The virtual machine is now configured so we can start the guest operating system installation.

## Guest Operating System Installation
With the new VM highlighted, click the "Start" button on the toolbar. On the "Select start-up disk" screen, choose the relevant Oracle Linux ISO image and click the "Start" button.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113838@2x.jpg "Guest Operating System Installation")
The resulting console window will contain the Oracle Linux boot screen.
![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-113956@2x.jpg "Guest Operating System Installation")

![19c_RAC_install](http://github.com/cashfit/oracle_articles/raw/master/19c_RAC_install/Jietu20191119-114118@2x.jpg "Guest Operating System Installation")
Continue through the Oracle Linux 7 installation as you would for a basic server. A general pictorial guide to the installation can be found here. More specifically, it should be a server installation with a minimum of 4G+ swap, firewall disabled, SELinux set to permissive and the following package groups installed:
- Server with GUI
- Hardware Monitoring Utilities
- Large Systems Performance
- Network file system client
- Performance Tools
- Compatibility Libraries
- Development Tools

To be consistent with the rest of the article, the following information should be set during the installation.
- hostname: ol7-19c-rac1.localdomain
- enp0s3 (eth0): DHCP (Connect Automatically)
- enp0s8 (eth1): IP=192.168.56.101, Subnet=255.255.255.0, Gateway=192.168.56.1, DNS=192.168.56.1, Search=localdomain (Connect Automatically)
- enp0s9 (eth2): IP=192.168.1.101, Subnet=255.255.255.0, Gateway=<blank>, DNS=<blank>, Search=<blank> (Connect Automatically)

You are free to change the IP addresses to suit your network, but remember to stay consistent with those adjustments throughout the rest of the article. Likewise, in this article I will refer to the network adapters as enp0s3, enp0s8 and enp0s9. In previous Linux versions they would have been eth0, eth1 and eth2 respectively.

## Oracle Installation Prerequisites
Perform either the Automatic Setup or the Manual Setup to complete the basic prerequisites. The Additional Setup is required for all installations.

### Automatic Setup
If you plan to use the "oracle-rdbms-server-12cR1-preinstall" package to perform all your prerequisite setup, issue the following command.
```console
# yum install -y oracle-database-preinstall-19c
```

 Earlier versions of Oracle Linux required manual setup of the Yum repository by following the instructions at http://public-yum.oracle.com.
 
 It is probably worth doing a full update as well, but this is not strictly speaking necessary.
 ```console
 # yum update -y
 ```

### Manual Setup
If you have not used the "oracle-database-preinstall-19c" package to perform all prerequisites, you will need to manually perform the following setup tasks.

Add the following lines to the "/etc/sysctl.conf" file, or in a file called "/etc/sysctl.d/98-oracle.conf".

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
Run the following command to change the current kernel parameters.
```console
/sbin/sysctl -p
# or
/sbin/sysctl -p /etc/sysctl.d/98-oracle.conf
```

Add the following lines to a file called "/etc/security/limits.d/oracle-database-server-19c-preinstall.conf" file.
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

In addition to the basic OS installation, the following packages must be installed whilst logged in as the root user. This includes the 64-bit and 32-bit versions of some packages.
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

Create the new groups and users.

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

You could define the additional groups and assign them to the "oracle" users. The would allow you to assign the individual groups during the installation. For this installation I've just used the "dba" group.

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

### Additional Setup
The following steps must be performed, whether you did the manual or automatic setup.
Perform the following steps whilst logged into the "ol7-19c-rac1" virtual machine as the root user.
Set the password for the "oracle" user.
```console
passwd oracle
```

Apart form the localhost address, the "/etc/hosts" file can be left blank, but I prefer to put the addresses in for reference.
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

The SCAN address is commented out of the hosts file because it must be resolved using a DNS, so it can round-robin between 3 addresses on the same subnet as the public IPs. The DNS can be configured on the host machine using [BIND](https://oracle-base.com/articles/linux/dns-configuration-for-scan) or [Dnsmasq](https://oracle-base.com/articles/linux/dnsmasq-for-simple-dns-configurations), which is much simpler. If you are using Dnsmasq, put the RAC-specific entries in the hosts machines "/etc/host" file, with the SCAN entries uncommented, and restart Dnsmasq.
Make sure the "/etc/resolv.conf" file includes a nameserver entry that points to the correct nameserver. Also, if the "domain" and "search" entries are both present, comment out one of them. For this installation my "/etc/resolv.conf" looked like this.
```console
#domain localdomain
search localdomain
nameserver 192.168.56.1
```

The changes to the "resolv.conf" will be overwritten by the network manager, due to the presence of the NAT interface. For this reason, this interface should now be disabled on startup. You can enable it manually if you need to access the internet from the VMs. Edit the "/etc/sysconfig/network-scripts/ifcfg-eth0" file, making the following change. This will take effect after the next restart.
```console
ONBOOT=no
```

There is no need to do the restart now. You can just run the following command.
```console
# ifdown enp0s3
```

At this point, the networking for the first node should look something like the following. Notice that eth0 has no associated IP address because it is disabled.
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

With this in place and the DNS configured the SCAN address is being resolved to all three IP addresses.
```console
# nslookup ol7-19c-scan
Server:		192.168.56.1
Address:	192.168.56.1#53

Name:	ol7-19c-scan.localdomain
Address: 192.168.56.105
#
```

Amend the "/etc/security/limits.d/90-nproc.conf" file as described below. See [MOS Note [ID 1487773.1]](https://support.oracle.com/epmos/faces/DocContentDisplay?id=1487773.1)
```console
# Change this
*          soft    nproc    1024

# To this
* - nproc 16384
```

Change the setting of SELinux to permissive by editing the "/etc/selinux/config" file, making sure the SELINUX flag is set as follows.
```console
SELINUX=permissive
```

If you have the Linux firewall enabled, you will need to disable or configure it, as shown [here](https://oracle-base.com/articles/linux/oracle-linux-6-installation#firewall) or [here](https://oracle-base.com/articles/linux/linux-firewall#installation). The following is an example of disabling the firewall.
```console
# systemctl stop firewalld
# systemctl disable firewalld
```

Either configure NTP, or make sure it is not configured so the Oracle Cluster Time Synchronization Service (ctssd) can synchronize the times of the RAC nodes. 

Make sure NTP (Chrony on OL7/RHEL7) is enabled.
```console
# systemctl enable chronyd
# systemctl restart chronyd
# chronyc -a 'burst 4/4'
# chronyc -a makestep
```

Create the directories in which the Oracle software will be installed.
```console
mkdir -p  /u01/app/19.0.0/grid
mkdir -p /u01/app/oracle/product/19.0.0/db_1
chown -R oracle:oinstall /u01
chmod -R 775 /u01/
```

Log in as the "oracle" user and add the following lines at the end of the "/home/oracle/.bash_profile" file.
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

Create a file called "/home/oracle/grid_env" with the following contents.
```console
export ORACLE_SID=+ASM1
export ORACLE_HOME=$GRID_HOME
export PATH=$ORACLE_HOME/bin:$BASE_PATH

export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
```

Create a file called "/home/oracle/db_env" with the following contents.
```console
export ORACLE_SID=cdbrac1
export ORACLE_HOME=$DB_HOME
export PATH=$ORACLE_HOME/bin:$BASE_PATH

export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
```

Once the "/home/oracle/.bash_profile" has been run, you will be able to switch between environments as follows.
```console
$ grid_env
$ echo $ORACLE_HOME
/u01/app/19.0.0/grid
$ db_env
$ echo $ORACLE_HOME
/u01/app/oracle/product/19.0.0/db_1
$
```

We've made a lot of changes, so it's worth doing a reboot of the VM at this point to make sure all the changes have taken effect.
```console
# shutdown -r now
```

