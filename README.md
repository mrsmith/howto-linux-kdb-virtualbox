# howto-linux-kdb-virtualbox
How to use kdb with VirtualBox

## Overview
This how to guide desribes how to use KDB in-kernel debugger to debug Linux kernel running as a guest inside VirtualBox VM. The setup consists of 4 steps:

1. Setup the VM, in this case VirtualBox
2. Setup the Guest, in this case AmazonLinux 2
3. Obtain vmlinux image and kernel source
4. Attach the debuger

## 1. Setup the VM
KDB is interacting with GDB on the host using serial port. To configure serial port on the VirtialBox guest go to `Machine > Settings > Ports` and configure `Port 1` as shown below:

![VirtualBox port configuration](/images/vbox-ports.png)

In oreder to turn the pipe into a PTY device you will need socat:

```
# the pty device name that socat creates can be found in `./tty` text file
socat -d -d /tmp/alinux.ttyS0 pty &

# save the PTY device name from socat log into a file for future reference
# the log message shold look like "PTY is /dev/*"
echo /dev/ttys000 > ./tty

# the newly created PTY can be tested with minicom
# make sure to disconnect minicom before attaching GDB
minicom -D $(cat ./tty)
```

## 2. Setup the Guest
This section largely depends on the Linux distribution and the kernel configuration being used. Here is an example setup for AmazonLinux 2, which should be similar for any other RPM based distributions.

```
# load kdb module
modprobe kgdboc kgdboc=ttyS0,115200

# verify: you sould see the message "KGDB: Registered I/O driver kgdboc"
dmesg | tail
```

## 3. Obtain vmlinux image & kernel source
In AmazonLinux 2 `vmlinux` ships as a part of `kernel-debuginfo` package.

```
sudo yum -y install kernel-debuginfo-$(uname -r)
ls -l /usr/lib/debug/lib/modules/4.14.77-81.59.amzn2.x86_64/vmlinux
```

The `vmlinux` file needs to be copied to the host.

Aside from `vmlinux` debugger is also going to need kernel source code.

```
# install tools
sudo yum -y install rpmdevtools

# enable source repository `amzn2-core-source`
sudo $EDITOR /etc/yum.repos.d/amzn2-core.repo

# obtain source rpm for kernel
yumdownloader --source kernel-$(uname -r)

# unpack the source
sudo rpm -Uvh ./kernel-4.14.77-81.59.amzn2.src.rpm

# move source back to $HOME
sudo mv ~root/rpmbuild ./
sudo chown ec2-user:ec2-user -R ./rpmbuild/

# tell rpmbuild to use user's `rpmbuild` location
echo "%_topdir $HOME/rpmbuild" > ~/rpmmacros

# prep the source tree
mkdir ~/rpmbuild/BUILD
rpmbuild -bp ./rpmbuild/SPECS/kernel.spec --nodeps

# validate
ls -l ./rpmbuild/BUILD/kernel-4.14.77.amzn2/linux-4.14.77-81.59.amzn2.x86_64
```

The last directory `linux-*` will need to be copied to the host.

## 4. Attach the debugger

Before GDB can be attached the guest kernel should be interrupted so that KDB can take control:

```
# guest:
cat g | sudo tee > /proc/sysrq-trigger 
```

```
# host:
gdb -ex "dir ./linux-4.14.77-81.59.amzn2.x86_64" -ex "target remote $(cat ./tty)" ./vmlinux
```

If all goes well you should be inside a GDB debug session how.

## References
### Amazon Linux 2 and RPM
- https://aws.amazon.com/amazon-linux-2/
- https://blog.packagecloud.io/eng/2015/04/20/working-with-source-rpms/
- http://ftp.rpm.org/max-rpm/s1-rpm-anywhere-different-build-area.html

### KGDB
- https://www.kernel.org/doc/html/v4.14/dev-tools/kgdb.html
- https://people.freedesktop.org/~narmstrong/meson_drm_doc/dev-tools/gdb-kernel-debugging.html
- https://www.kernel.org/doc/htmldocs/kgdb/EnableKGDB.html
- https://opensourceforu.com/2011/03/kgdb-with-virtualbox-debug-live-kernel/
