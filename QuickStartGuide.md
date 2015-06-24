# Installing NITRO on Ubuntu 11.10 x86\_64 #
**Note:** Nitro is only compatible with the virtualization extensions provided on Intel CPUs.  Nitro will not run on AMD processors.
## The Example system ##
```
uname -a
# Linux lap-nes9 2.6.37-02063706-generic #201103281005 SMP Mon Mar 28 10:10:47 UTC 2011 x86_64 x86_64 x86_64 GNU/Linux
cat /etc/lsb-release 
# DISTRIB_ID=Ubuntu
# DISTRIB_RELEASE=11.10
# DISTRIB_CODENAME=oneiric
# DISTRIB_DESCRIPTION="Ubuntu 11.10"
```
Unfortunately, this is a ubuntu system... which smells bad. :-(

## Installing Necessary Packages ##
aptitude install libghc-zlib-dev libghc6-zlib-dev zlib1g zlib1g-dev libpci-dev build-essential gdb make automake

## Getting a compatible kernel ##
Nitro currentl supports Kernel 2.6.35 - 2.6.37, so we need to install a compatible kernel.
```
dpkg -i linux-headers-2.6.37-02063706-generic_2.6.37-02063706.201103281005_amd64.deb
dpkg -i linux-headers-2.6.37-02063706_2.6.37-02063706.201103281005_all.deb
dpkg -i linux-image-2.6.37-02063706-generic_2.6.37-02063706.201103281005_amd64.deb
```
As Debian would just remove this kernel upon the next update or kick it out of your boot loaders configuration, make sure to set it on hold, if running Debian or it's offspring.
```
aptitude hold linux-image-2.6.37-02063706-generic
aptitude hold linux-headers-2.6.37-02063706
aptitude hold linux-headers-2.6.37-02063706-generic
```
In the end these the packages stats should look like this:
```
dpkg --list |grep 2\.6\.37
# hi  linux-headers-2.6.37-02063706                  2.6.37-02063706.201103281005               Header files related to Linux kernel version 2.6.37
# hi  linux-headers-2.6.37-02063706-generic          2.6.37-02063706.201103281005               Linux kernel headers for version 2.6.37 on x86/x86_64
# hi  linux-image-2.6.37-02063706-generic            2.6.37-02063706.201103281005               Linux kernel image for version 2.6.37 on x86/x86_64
```

# Compiling Nitro #
(the usual way: configure/make/make install)

## Grabbing the sources ##
```
cd /usr/src/
svn checkout http://nitro-kvm.googlecode.com/svn/trunk/ nitro-kvm-read-only
```

## Building the VMM/qemu ##
```
cd /usr/src/nitro-kvm-read-only/nitro
./configure --prefix=/opt/nitro --enable-kvm --disable-xen --enable-debug
make
make install
```
## Building kmod-nitro ##
```
cd /usr/src/nitro-kvm-read-only/nitro-kmod
./configure
make
make install
```
# Running/Using Nitro #

Here is quick and dirty scriptlet for starting nitro:

```
#!/bin/bash
# itty bitty startup-script, 
# by Christian M. Fuchs, Fraunhofer AISEC, EWS
#
# This small script fires up a preexisting qemu-VM in nitro, and 
# attachs the qemu monitoring console directly to your terminal.

sudo modprobe kvm
sudo modprobe kvm-intel
sudo /opt/nitro/bin/qemu-system-x86_64 -snapshot -chardev vc,id=foo -monitor stdio -m 1024 -usbdevice tablet -vnc :0 /opt/windows-xpsp3-hd.img
```

```
touch ~/sbin/nitro_start
chmod +x ~/sbin/nitro_start
nano ~/sbin/nitro_start
```

Now, start the VM with our new script.

NOTE: at the time of the creation of this guide, kmod-nitro can NOT be unloaded from the kernel at runtime! Thus, each time you want to update it, you need to REBOOT the host domain!

# Using SCMON #

Inside the monitor we will add two simple rules to monitor the following syscalls:
> NtWriteFile() (RAX == 0x112) and
> NtOpenFile() (RAX == 0x74)

A neat list of system call numbers and their symbols can be found here:  http://j00ru.vexillium.org/ntapi/

The syntax of a syscall monitor(scmon) rule syntax (all values are DECIMAL! 274 == 0x112):
```
add_scmon_rule <condition_register> <condition_register_value> <output_register> <offset> <HEX/DREFSTR/INT/UINT/...>
```

So, we will create appropriate rules to trap these two system calls:
```
add_scmon_rule rax 274 rcx 0 hex
add_scmon_rule rax 274 rbx 0 hex
add_scmon_rule rax 274 rdx 0 hex
add_scmon_rule rax 274 rdx 0 derefstr
add_scmon_rule rax 116 any 0 hex

list_scmon_rules
#[0] if (rax == 0x112)
#[0] => rcx+0 hex
#[0] => rbx+0 hex
#[0] => rdx+0 hex
#[0] => rdx+0 derefstr
#[1] if (rax == 0x74)
#[1] => any+0 hex
```

syntax:
```
[rulenum] if (condition)
[rulenum] => <outputreg+offset> <output format>
```

now, start syscall tracing for this VM (46 == 0x2e == intrrupt gate for the syscall handler in windows based systems):
```
start_scmon 46 rax
```

Now you will get output similar to this (using a XML-like output format):
```
Mar 13 15:30:05 lap-nes9 kernel: [85122.251649] <syscall><rule><condition><rname>rax</rname><rval>0x74</rval></condition><actions><reg><rname>rax</rname><rform>HEX</rform><rval>0x74</rval></reg><reg><rname>rcx</rname><rform>HEX</rform><rval>0x7C92428F</rval></reg><reg><rname>rdx</rname><rform>HEX</rform><rval>0x7E6F8</rval></reg><reg><rname>rbx</rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>rsp</rname><rform>HEX</rform><rval>0x7E6F0</rval></reg><reg><rname>rbp</rname><rform>HEX</rform><rval>0x7E768</rval></reg><reg><rname>rsi</rname><rform>HEX</rform><rval>0x7FFDFC00</rval></reg><reg><rname>rdi</rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>r8 </rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>r9 </rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>r10</rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>r11</rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>r12</rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>r13</rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>r14</rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>r15</rname><rform>HEX</rform><rval>0x0</rval></reg></actions></rule></syscall>
Mar 13 15:30:05 lap-nes9 kernel: [85122.252463] <syscall><rule><condition><rname>rax</rname><rval>0x74</rval></condition><actions><reg><rname>rax</rname><rform>HEX</rform><rval>0x74</rval></reg><reg><rname>rcx</rname><rform>HEX</rform><rval>0x7C92428F</rval></reg><reg><rname>rdx</rname><rform>HEX</rform><rval>0x7E6F8</rval></reg><reg><rname>rbx</rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>rsp</rname><rform>HEX</rform><rval>0x7E6F0</rval></reg><reg><rname>rbp</rname><rform>HEX</rform><rval>0x7E768</rval></reg><reg><rname>rsi</rname><rform>HEX</rform><rval>0x7FFDFC00</rval></reg><reg><rname>rdi</rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>r8 </rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>r9 </rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>r10</rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>r11</rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>r12</rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>r13</rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>r14</rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>r15</rname><rform>HEX</rform><rval>0x0</rval></reg></actions></rule></syscall>
Mar 13 15:31:05 lap-nes9 kernel: [85132.252463] <syscall><rule><condition><rname>rax</rname><rval>0x112</rval></condition><actions><reg><rname>rcx</rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>rbx</rname><rform>HEX</rform><rval>0x0</rval></reg><reg><rname>rdx</rname><rform>HEX</rform><rval>0x21ECB6C</rval></reg><reg><rname>rdx</rname><rform>DEREFSTR</rform><rval></rval></reg></actions></rule></syscall>
```