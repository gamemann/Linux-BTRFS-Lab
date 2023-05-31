# Linux BTRFS Lab
This is just a small repository to store my results on testing Linux [BTRFS's](https://archive.kernel.org/oldwiki/btrfs.wiki.kernel.org/) out-of-band deduplication [feature](https://btrfs.readthedocs.io/en/latest/Deduplication.html). BTRFS is an advanced Linux file system that comes with many neat features!

## Lab Specs
* Created on my '[SpyKids](https://github.com/gamemann/Home-Lab#three-spykids)' home server running Ubuntu 22.04.
* Virtual machine created with KVM/QEMU running Ubuntu 23.04.
* Two virtual cores.
* 4 GBs of RAM.
* 1 x 200 GBs SSD (virtio driver).

Output from `lshw -short` on VM.

```bash
christian@sk-btrfstest01:~$ sudo lshw -short
H/W path           Device      Class          Description
=========================================================
                               system         Standard PC (Q35 + ICH9, 2009)
/0                             bus            Motherboard
/0/0                           memory         96KiB BIOS
/0/400                         processor      Intel(R) Core(TM) i7-8700K CPU @ 3.70GHz
/0/401                         processor      Intel(R) Core(TM) i7-8700K CPU @ 3.70GHz
/0/1000                        memory         4GiB System Memory
/0/1000/0                      memory         4GiB DIMM RAM
/0/100                         bridge         82G33/G31/P35/P31 Express DRAM Controller
/0/100/1           /dev/fb0    display        QXL paravirtual graphic card
/0/100/2                       bridge         QEMU PCIe Root port
/0/100/2/0                     network        Virtio network device
/0/100/2/0/0       enp1s0      network        Ethernet interface
/0/100/2.1                     bridge         QEMU PCIe Root port
/0/100/2.1/0                   bus            QEMU XHCI Host Controller
/0/100/2.1/0/0     usb1        bus            xHCI Host Controller
/0/100/2.1/0/0/1   input4      input          QEMU QEMU USB Tablet
/0/100/2.1/0/1     usb2        bus            xHCI Host Controller
/0/100/2.2                     bridge         QEMU PCIe Root port
/0/100/2.2/0                   communication  Virtio console
/0/100/2.2/0/0                 generic        Virtual I/O device
/0/100/2.3                     bridge         QEMU PCIe Root port
/0/100/2.3/0                   storage        Virtio block device
/0/100/2.3/0/0     /dev/vda    disk           214GB Virtual I/O device
/0/100/2.3/0/0/1   /dev/vda1   volume         1023KiB BIOS Boot partition
/0/100/2.3/0/0/2   /dev/vda2   volume         199GiB EFI partition
/0/100/2.4                     bridge         QEMU PCIe Root port
/0/100/2.4/0                   generic        Virtio memory balloon
/0/100/2.4/0/0                 generic        Virtual I/O device
/0/100/2.5                     bridge         QEMU PCIe Root port
/0/100/2.5/0                   generic        Virtio RNG
/0/100/2.5/0/0                 generic        Virtual I/O device
/0/100/2.6                     bridge         QEMU PCIe Root port
/0/100/2.7                     bridge         QEMU PCIe Root port
/0/100/3                       bridge         QEMU PCIe Root port
/0/100/3.1                     bridge         QEMU PCIe Root port
/0/100/3.2                     bridge         QEMU PCIe Root port
/0/100/3.3                     bridge         QEMU PCIe Root port
/0/100/3.4                     bridge         QEMU PCIe Root port
/0/100/3.5                     bridge         QEMU PCIe Root port
/0/100/1b          card0       multimedia     82801I (ICH9 Family) HD Audio Controller
/0/100/1f                      bridge         82801IB (ICH9) LPC Interface Controller
/0/100/1f/0                    communication  PnP device PNP0501
/0/100/1f/1                    input          PnP device PNP0303
/0/100/1f/2                    input          PnP device PNP0f13
/0/100/1f/3                    system         PnP device PNP0b00
/0/100/1f/4                    system         PnP device PNP0c01
/0/100/1f.2        scsi0       storage        82801IR/IO/IH (ICH9R/DO/DH) 6 port SATA Controller [AHCI mode]
/0/100/1f.2/0.0.0  /dev/cdrom  disk           QEMU DVD-ROM
/0/100/1f.3                    bus            82801I (ICH9 Family) SMBus Controller
/1                 input0      input          Power Button
/2                 input1      input          AT Translated Set 2 keyboard
/3                 input3      input          ImExPS/2 Generic Explorer Mouse
/4                 input6      input          spice vdagent tablet
```

Output from `uname -r` on VM.

```bash
christian@sk-btrfstest01:~$ sudo uname -r
6.2.0-20-generic
```

Output from `cat /etc/*-release` on VM.

```bash
christian@sk-btrfstest01:~$ cat /etc/*-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=23.04
DISTRIB_CODENAME=lunar
DISTRIB_DESCRIPTION="Ubuntu 23.04"
PRETTY_NAME="Ubuntu 23.04"
NAME="Ubuntu"
VERSION_ID="23.04"
VERSION="23.04 (Lunar Lobster)"
VERSION_CODENAME=lunar
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=lunar
LOGO=ubuntu-logo
```

## Partition Table When Installing Ubuntu 23.04
Here is a screenshot of the VM's partition table/configuration during the Ubuntu installation.

!([Ubuntu Install](./images/ubuntu_install.png))

For simplicity, I created a single partition for the entire hard drive mounted at `/` (root) which uses BTRFS.

## Installing Deduplication Tools
On Ubuntu 23.04, you will want to install deduplication tools using `apt`. You may run the following command to do so.

```bash
sudo apt install -y duperemove
```

## Starting Disk Space
Here's the output from `df -h` on the VM with a vanilla installation of Ubuntu 23.04.

```bash
christian@sk-btrfstest01:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           390M  1.7M  389M   1% /run
/dev/vda2       200G  6.5G  192G   4% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M  8.0K  5.0M   1% /run/lock
tmpfs           390M  180K  390M   1% /run/user/1000
```

## Creating Dummy Files
In this lab, we will create a directory with two files. One file will be 15 GBs and the other will be 10 GBs. We can use the `dd` Linux tool to create these files that are padded with 0's.

```bash
# Create directory.
mkdir test1

# Change directory.
cd test1

# Create 15 GBs file.
dd if=/dev/zero of=filedum1 bs=1G count=15

# Create 10 GBs file.
dd if=/dev/zero of=filedum2 bs=1G count=10
```

**Note** - There are faster commands to create dummy files other than `dd`. However, `dd` is the most commonly used which is why I chose to use the command in this lab.

Here is the output from the commands above.

```bash
christian@sk-btrfstest01:~$ # Create directory.
mkdir test1

# Change directory.
cd test1

# Create 15 GBs file.
dd if=/dev/zero of=filedum1 bs=1G count=15

# Create 10 GBs file.
dd if=/dev/zero of=filedum2 bs=1G count=10
15+0 records in
15+0 records out
16106127360 bytes (16 GB, 15 GiB) copied, 25.2724 s, 637 MB/s
10+0 records in
10+0 records out
10737418240 bytes (11 GB, 10 GiB) copied, 23.9933 s, 448 MB/s
```

Now if we execute `ls -lah .`, you can see the size of each file in the test directory.

```bash
christian@sk-btrfstest01:~/test1$ ls -lah .
total 25G
drwxrwxr-x 1 christian christian  32 May 31 18:51 .
drwxr-x--- 1 christian christian 308 May 31 18:51 ..
-rw-rw-r-- 1 christian christian 15G May 31 18:51 filedum1
-rw-rw-r-- 1 christian christian 10G May 31 18:52 filedum2
```

Now let's check the output from `df -h` again.

```bash
christian@sk-btrfstest01:~/test1$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           390M  1.7M  389M   1% /run
/dev/vda2       200G   32G  167G  16% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M  8.0K  5.0M   1% /run/lock
tmpfs           390M  172K  390M   1% /run/user/1000
```

## Testing Duplication
Now, we could copy the directory we just created, but I noticed this automatically handles deduplication due to what the `cp` command does. Therefore, you won't see what it looks like without deduplication.

However, if you want a quick example, you may execute the following.

```bash
# Go up a directory.
cd ..

# Copy test1 to test2.
cp -r test1/ test2/
```

If you execute `du -sh *`, you'll see both directories are 25 GBs in size.

```bash
christian@sk-btrfstest01:~$ du -sh *
...
25G     test1
25G     test2
...
```

If we run `df -h` again, you'll see we have the same size as before (32 GBs used).

```bash
christian@sk-btrfstest01:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           390M  1.7M  389M   1% /run
/dev/vda2       200G   32G  167G  16% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M  8.0K  5.0M   1% /run/lock
tmpfs           390M  172K  390M   1% /run/user/1000
```

This is the deduplication feature working!

### Testing Without Copying Directory
Let's try testing without copying the directory which handles deduplication automatically. We will use the same commands as we used the first time to generate large files. However, we'll rename `test1` to `test2` like the following.

```bash
# Create directory.
mkdir test2

# Change directory.
cd test2

# Create 15 GBs file.
dd if=/dev/zero of=filedum1 bs=1G count=15

# Create 10 GBs file.
dd if=/dev/zero of=filedum2 bs=1G count=10
```

Here's the output from above.

```bash
christian@sk-btrfstest01:~$ # Create directory.
mkdir test2

# Change directory.
cd test2

# Create 15 GBs file.
dd if=/dev/zero of=filedum1 bs=1G count=15

# Create 10 GBs file.
dd if=/dev/zero of=filedum2 bs=1G count=10

15+0 records in
15+0 records out
16106127360 bytes (16 GB, 15 GiB) copied, 26.6588 s, 604 MB/s
10+0 records in
10+0 records out
10737418240 bytes (11 GB, 10 GiB) copied, 19.1798 s, 560 MB/s
```

Now if we run `df -h`, you can see we're using 57 GBs instead of 32 GBs.

```bash
christian@sk-btrfstest01:~/test2$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           390M  1.7M  389M   1% /run
/dev/vda2       200G   57G  142G  29% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M  8.0K  5.0M   1% /run/lock
tmpfs           390M  172K  390M   1% /run/user/1000
```

To my understanding, since BTRFS operates in out-of-band mode **only**, it will not handle deduplication automatically on each write. Therefore, we will have to run the command `duperemove` with a couple of parameters.

Let's run this command with the `-dr` flags!

```bash
# Go up one directory.
cd ..

# Run deduplication command.
sudo duperemove -dr .
```

This ends up consuming a bit of CPU since it is handling deduplication.

**To Be Continued**

## Credits
* [Christian Deacon](https://github.com/gamemann)
