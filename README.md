# Linux BTRFS Lab
This is just a small repository to store my results on testing Linux [BTRFS's](https://archive.kernel.org/oldwiki/btrfs.wiki.kernel.org/) out-of-band deduplication [feature](https://btrfs.readthedocs.io/en/latest/Deduplication.html). BTRFS is an advanced Linux file system that comes with many neat features!

## Motives
While I'm interested in Linux file systems in general, a gaming community I help out in has a dedicated server running Linux with 512 GBs of disk space utilizing the `ext4` file system that runs multiple servers for the game [Counter-Strike: Global Offensive](https://steamdb.info/app/730/charts/). CS:GO's base installation files are around *~33 GBs* each which resulted in the dedicated server running low on disk space without many **custom** game server files. Using hard links is an option, but since they utilize [Pterodactyl](https://pterodactyl.io/)/Docker, implementing a hard-link approach would be more difficult since Pterodactyl's mount feature wouldn't work because we'd have hard links on separate file systems which is incompatible. Therefore, since they are buying a new machine soon, I wanted to look into using different Linux file systems that can utilize compression and/or deduplication to save disk space. I assumed the deduplication feature with file systems such as BTRFS would benefit a lot in this situation since the 33 GBs of base installation files for CS:GO are identical.

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

![Ubuntu Install](./images/ubuntu_install.png)

For simplicity, I created a single partition for the entire hard drive mounted at `/` (root) which uses BTRFS.

## Installing Deduplication Tools
There are multiple deduplication tools you can install or build on your system. In this lab, we'll be using [Duperemove](https://github.com/markfasheh/duperemove). For Ubuntu 23.04, you may run the following `apt` command to install Duperemove via package manager.

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

Now if we execute `ls -lh .`, you can see the size of each file in the test directory.

```bash
christian@sk-btrfstest01:~/test1$ ls -lh .
total 25G
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
Now, we could copy the directory we just created, but I noticed this automatically handles deduplication due to what the `cp` command does behind the scenes. Therefore, you won't see what it looks like without deduplication.

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

This ends up consuming a bit of CPU since it is scanning files/hashes to determine if the files need to be deduplicated. However, it didn't perform any deduplication on my end. I figured this was due to checksum/block differences. Afterall, we are creating completely separate files rather than copying. Though, the contents of the files should be the same (all zero'd bytes).

Therefore, I started reading the manual page for `duperemove` (`man duperemove`). I ended up trying other hashing algorithms which didn't make any differences. I then came across the  `---dedupe-options=[OPTIONS]` flag which is explained below.

```bash
--dedupe-options=options
    Comma separated list of options which alter how we dedupe. Prepend 'no' to an option in order to turn it off.

    [no]partial
        Duperemove can often find more dedupe by comparing portions of extents to each other. This can be a lengthy, CPU  in‐
        tensive task so it is turned off by default.

        The  code  behind  this  option is under active development and as a result the semantics of the partial argument may
        change.

    [no]same
        Defaults to off. Allow dedupe of extents within the same file.

    [no]fiemap
        Defaults to on. Duperemove uses the fiemap ioctl during the dedupe stage to optimize out already deduped  extents  as
        well as to provide an estimate of the space saved after dedupe operations are complete.

        Unfortunately,  some  versions of Btrfs exhibit extremely poor performance in fiemap as the number of references on a
        file extent goes up. If you are experiencing the dedupe phase slowing down or 'locking up' this option may give you a
        significant amount of performance back.

        Note: This does not turn off all usage of fiemap, to disable fiemap during the file scan stage, you will also want to
        use the --lookup-extents=no option.

    [no]block
        Deprecated.
```

I ended up using `--dedupe-options partial` which took **a lot** longer to run along with more CPU but performed deduplication. However, it did perform solid deduplication.

```bash
# Run deduplication command with partial option set.
sudo duperemove --dedupe-options partial -dr .
```

While this does consume a bit of CPU, you can use the following flags to limit the amount of cores/threads.

```bash
--io-threads=N
    Use N threads for I/O. This is used by the file hashing and dedupe stages. Default is automatically detected based on number
    of host cpus.

--cpu-threads=N
    Use N threads for CPU bound tasks. This is used by the duplicate extent finding stage.  Default  is  automatically  detected
    based on number of host cpus.

    Note:  Hyperthreading  can adversely affect performance of the extent finding stage. If duperemove detects an Intel CPU with
    hyperthreading it will use half the number of cores reported by the system for cpu bound tasks.
```

## Conclusion
This was a fun experiment for me because I feel my knowledge with file systems isn't enough and I'll be using the BTRFS file system more in the future along with the deduplication feature. I think it's best to have a cron job run late at night every two weeks or so to perform deduplication with partial support due to how long it takes.

There are other file systems I'm going to also experiment with such as [ZFS](https://en.wikipedia.org/wiki/ZFS) and [XFS](https://en.wikipedia.org/wiki/XFS) which I've heard great things about. With that said, having deduplication in-band sounds like a better approach since it performs deduplication if needed on any write operation. However, this could be costly to the CPU as well, so there are definitely pros to out-of-band deduplication (e.g. being able to pick what directories to perform deduplication on and what times).

One other thing I did want to note is while file systems such as BTRFS have matured a lot over the years, it is stated deduplication still poses a very small risk of data corruption. This is because the deduplication process involves sharing data blocks and any changes to the data block being shared could cause issues. BTRFS does have safeguards for this, though. So make sure to always **back up your files** if you can!

## Credits
* [Christian Deacon](https://github.com/gamemann)
