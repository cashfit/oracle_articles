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

```shell
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
