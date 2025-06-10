---
layout: post
title: File backup solution with file container. Efficent for network transfers and file corruption restoration. Testable.
---

# Catalyst

Filesystem corruption on a large 7zip file, the **file container**, prompted a (hopefully) improved solution.

Documenting this for
1. My information when I forget in a year (or less?)
1. In case some portion of this can help someone else.
1. To reference in comments to some of the questions I used with the goal of #2

# File Container

Moving away from a zip format (.zip, .7z, .rar, .tar, .tgz) to a Virtual Disk image.

VMDK selected, **specifically for** the disk type referred to in the Virtual Disk Format specification as [twoGbMaxExtentFlat](https://web.archive.org/web/20191016053703/https://www.vmware.com/app/vmdk/?src=vmdk). The [VMDK Virtual Disk Types](https://vdc-download.vmware.com/vmwb-repository/dcr-public/2bba164b-4115-4279-9c99-40f4c14319ad/03a845fc-5345-45de-9a27-31e868d6e751/doc/vddkDataStruct.5.3.html) describes this as:
>**Split disk into 2GB files**  
>The first VMDK file is small and points to a sequence of other files, all of which have an f before the sequence number, meaning flat. The number of files depends on the requested size.

Virtual Disks images and zip formats are both easily accessible on Linux and Windows. A Virtual Disk image provides several advantages over a zip file
- the ability to mount (test) it,
- access and interact with it using general filesystem transfer tools
    - robocopy in Windows and rsync in Linux
    - btrfs/ZFS [send & receive](https://docs.oracle.com/cd/E18752_01/html/819-5461/gbchx.html)
- [btrfs snapshot](https://btrfs.readthedocs.io/en/latest/btrfs-subvolume.html#subvolume-and-snapshot) / [ZFS snapshot](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-snapshot.8.html#zfs-snapshot-8) can be used to provide finer grained point in time recovery while reducing space requirements
- access files without having to _extract_ them
- utilize various encryption technologies

while it still allows for
- compression with filesystem compression
- cascading various encryption technologies

The zip formats generally support splitting the file into smaller chunks, which was [another important consideration](#split-file). That's why the VMDK [twoGbMaxExtentFlat](https://web.archive.org/web/20191016053703/https://www.vmware.com/app/vmdk/?src=vmdk) was chosen as oppossed to the other various Virtual Disk Image formats VHDX, VDI, etc.

## Create

**Many** options to create and mount Virtual Disk images on all Operating Systems. In this example utilized a [Docker image of Ubuntu noble 24.04](https://hub.docker.com/_/ubuntu/tags?name=noble) using [QEMU](https://qemu-project.gitlab.io/qemu/about/index.html). This didn't work as well as I'd hoped for providing isolation because I had to run the container in `--privileged` mode. See [Docker references](#docker) below for detailed explanation.

Utilizing this package:
```Shell
apt-get install qemu-utils
```

Then creating a VMDK with
command:
```Shell
qemu-img create -f vmdk -o subformat=twoGbMaxExtentFlat Demo_6G.vmdk 6G
```
output:
```ShellSession
Formatting 'Demo_6G.vmdk', fmt=vmdk size=6442450944 compat6=off hwversion=undefined subformat=twoGbMaxExtentFlats
```

command:
```Shell
ll -h Demo_6G*
```
output:
```ShellSession
-rw-r--r-- 1 root root 2.0G Jun 10 16:17 Demo_6G-f001.vmdk
-rw-r--r-- 1 root root 2.0G Jun 10 16:17 Demo_6G-f002.vmdk
-rw-r--r-- 1 root root 2.0G Jun 10 16:17 Demo_6G-f003.vmdk
-rw-r--r-- 1 root root  430 Jun 10 16:17 Demo_6G.vmdk
```

Because multiple files are created, and depending on the size, it could be a lot of files, VMDK created in a subdirectory.

command:
```Shell
parentdir="/mnt/vmdk" && vmdkname="Demo_6G" && size="6" && sizeformat="G"
if [ ! -d ${parentdir}/${vmdkname} ]; then mkdir --verbose ${parentdir}/${vmdkname}; fi && \
qemu-img create -f vmdk -o subformat=twoGbMaxExtentFlat ${parentdir}/${vmdkname}/${vmdkname}.vmdk ${size}${sizeformat} && \
extents128=$( expr "$size" '*' 8 ) && echo "Requires $extents128 extents @ 128M"
```

output:
```ShellSession
mkdir: created directory '/mnt/vmdk/Demo_6G'
Formatting '/mnt/vmdk/Demo_6G/Demo_6G.vmdk', fmt=vmdk size=6442450944 compat6=off hwversion=undefined subformat=twoGbMaxExtentFlat
Requires 48 extents @ 128M
```

The echo with the number of extents will be addressed in [Undocumented VMDK Splitting](#undocumented-vmdk-splitting). It can be ignored otherwise.

If I didn't use [QEMU](https://qemu-project.gitlab.io/qemu/about/index.html) on a Linux machine, I'd likely have used [VBoxManage createmedium](https://www.virtualbox.org/manual/ch08.html#vboxmanage-createmedium) on a Windows 11 machine.

>--format=VDI | VMDK | VHD  
>Specifies the file format of the output file. Valid formats are VDI, VMDK, and VHD. The default format is VDI.
>
>--variant=Standard,Fixed,Split2G,Stream,ESX,Formatted,RawDisk  
>Specifies the file format variant for the target medium, which is a comma-separated list of variants. Following are the valid values:  
>Split2G indicates that the disk image is split into 2GB segments. This value is valid for VMDK disk images only.


### Flat

I **VERY unsucessfuly TRIED** utilizing Sparse files initally. Sparse files would have obvious advantages of not having to pre-allocate space. However, through a lot of _frustrating_ trials and tribulations, I gave up and switched to Flat. Performance was just unacceptable.
I didn't get to the level of watching the block IO requests, but I tried watching the size increase during a transfer to a sparse extent/disk.
```Shell
for i in $(seq -f "%03g" 1 100); do ls -ltr | tail -1; sleep .5; ls -ltr | tail -1; done
```

It **appeared** to be increasing by 64k. The constant size increase requests resulted in truly horrific performance.

Disk Images and the chosen filesystem can pretty easily be expanded as necessary for space. The extra effort managing disk size utilizing a Flat disk type became a necessary requirement for me from a performance standpoint.

## Mounting

Utilizing these packages:
```Shell
apt-get install kmod
apt-get install qemu-utils
apt-get install nbd-client
```

Must once run after start of Operating System
```Shell
modprobe nbd
```


```Shell
#Connect IF not already connected.
nbd0_mount=`nbd-client -check /dev/nbd0`; if [ ${#nbd0_mount} -eq 0 ]; then qemu-nbd --connect=/dev/nbd0 ${parentdir}/${vmdkname}/${vmdkname}.vmdk; else echo "ERROR - ALREADY CONNECTED"; ps -ef | grep -E "qemu|/dev/nbd" | grep -v "grep"; fi
```
No output will be produced if it connected.

Can be verified with `nbd-client -check /dev/nbd0` output `1110701`
or `ps -ef | grep -E "qemu|/dev/nbd" | grep -v "grep"` output: `root        7714       1  0 16:39 ?        00:00:00 qemu-nbd --connect=/dev/nbd0 /mnt/vmdk/Demo_6G/Demo_6G.vmdk`

## Utilizing 

There are a great many ways to utilize at this point. The drive may be formatted with any number of file systems, encrypted or not encrypted.

If you were doing this on a Windows machine, you could format it with NTFS with BitLocker enabled and then robocopy files to it.

An ideal scenario would be in a Linux environment if you have a CoW filesystem such as ZFS or btrfs, this could be a send target for those. Allowing you to send incrimental updates, reducing the updates to the extents, thereby reducing the changes necessary to transfer over the network.

# Split File

The ability to split the container into smaller chunks was a key requirement for several reasons
- As mentioned in the [Catalyst](#catalyst), file corruption on a large file requires remediation on the entire LARGE file. 
- My experience has generally been that small files, especially when multithreaded, are much faster and better to transfer over a WAN and/or to Cloud storage.
    - Similar to the file corruption, if there is an issue with tranferring, especially if transfer resuming isn't supported, the small files provide a way to resume.
    - Few file transfer programs feature multithreading of a single file
- In resource constrained sitautions, many small disks can be used

# Encryption

[Data-at-rest encryption](https://wiki.archlinux.org/title/Data-at-rest_encryption) was an important consideration. A feature I found critical in the [Comparison table](https://wiki.archlinux.org/title/Data-at-rest_encryption#Comparison_table):

>Support for (manually) resizing the encrypted block device in-place
>| [LUKS](https://gitlab.com/cryptsetup/cryptsetup/) | [VeraCrypt](https://veracrypt.io/en/Home.html) | [ZFS](https://openzfs.github.io/openzfs-docs/Project%20and%20Community/index.html#project-and-community) |
>| --- | --- | --- | 
>| Yes | No | Yes |

This table is on the ArchLinux Wiki, so it doesn't address Windows. But this advantage would apply to BitLocker over VeraCrypt as well.

# Undocumented VMDK Splitting

The 2GB chunks that [twoGbMaxExtentFlat](https://web.archive.org/web/20191016053703/https://www.vmware.com/app/vmdk/?src=vmdk) provides is an advantage. For the reasons listed in [Split File](#split-file), I'd much prefer to have 50 count files of 2GB size than a single 100GB file. 
However, 2GB is still large. I thought 128 - 512 MB would be more in the range I'd prefer.

The disk format type is `twoGbMax` **Max** not **Must be**. So I hypothesized that the extents could be any size desired 2Gb or below. Given that it's not a requirement for the disk size to be divisible by 2 Gb, it has to support at least ONE extent smaller than 2 Gb (the last one).

Eventually I found the disk format specifications, but to initially test the feasability without committing the time to better understand, created the size I desired (128M) plus 2G.
>2G = 1024 x 2 = 2048. 2048 + 128 = 2176.

command:
```Shell
qemu-img create -f vmdk -o subformat=twoGbMaxExtentFlat 2_125G.vmdk 2176M
```

output:
```ShellSession
Formatting '2_125G.vmdk', fmt=vmdk size=2281701376 compat6=off hwversion=undefined subformat=twoGbMaxExtentFlat
```

command:
```Shell
ll -h  2_125G*
```

output:
```ShellSession
-rw-r--r-- 1 root root 2.0G Jun 10 17:14 2_125G-f001.vmdk
-rw-r--r-- 1 root root 128M Jun 10 17:14 2_125G-f002.vmdk
-rw-r--r-- 1 root root  388 Jun 10 17:14 2_125G.vmdk
```

The last extent, `2_125G-f002.vmdk` is the desired size. So to test if I could use those.
>2176 / 128 = **9**

So 9 extents are necessary. I copied `2_125G-f002.vmdk` to `2_125G-f001.vmdk` through `2_125G-f009.vmdk`. Circling back to the [Create](#create) section, this is why there is an echo to to display the number of 128M extents.

Then I had to edit the Descriptor File `2_125G.vmdk`.

Changing
```Shell
# Extent description
RW 4194304 FLAT "2_125G-f001.vmdk" 0
RW 262144 FLAT "2_125G-f002.vmdk" 0
```
to
```Shell
# Extent description
RW 262144 FLAT "2_125G-f001.vmdk" 0
RW 262144 FLAT "2_125G-f002.vmdk" 0
RW 262144 FLAT "2_125G-f003.vmdk" 0
RW 262144 FLAT "2_125G-f004.vmdk" 0
RW 262144 FLAT "2_125G-f005.vmdk" 0
RW 262144 FLAT "2_125G-f006.vmdk" 0
RW 262144 FLAT "2_125G-f007.vmdk" 0
RW 262144 FLAT "2_125G-f008.vmdk" 0
RW 262144 FLAT "2_125G-f009.vmdk" 0
```

Then I followed the [Mounting](#mounting) process, and it worked!

I decided to script this out. This script requires that the disk size MUST be divisible by 128M. The smaller extent size that was chosen. As has been demonstrated, this is NOT a hard requirement. But scripting wasn't added to handle outside of this use case. Setting the disk to the size necessary rounded up to the nearest 128M is acceptable.
Initially the `2_125G-f002.vmdk` was copied to `128M_extent_template.vmdk` for use. Later through the process, I learned that `dd` with `/dev/zero` could be used to create the flat extents.
If I were doing this on Windows, I'd probably have stuck with the template file. Also, if I was using Sparse extents, I'd have stuck with the template.

## vmdk_extent_generate.sh
```Shell
startnumber=1
totalnumber=9
vmdkname=2_125G
filenameprefix=${vmdkname}-f
for i in $(seq -f "%03g" $startnumber $totalnumber); do
    dd if=/dev/zero of=${filenameprefix}${i}.vmdk bs=8M count=16
done

sed --quiet --expression='0,/# Extent description/p' ${vmdkname}.vmdk > ${vmdkname}.new
for i in $(seq -f "%03g" $startnumber $totalnumber); do
    echo "RW 262144 FLAT \"${filenameprefix}${i}.vmdk\" 0" >> ${vmdkname}.new
done

echo "" >> ${vmdkname}.new
sed --quiet --expression='/# The Disk Data Base/,$p' ${vmdkname}.vmdk >> ${vmdkname}.new

mv --verbose ./${vmdkname}.new ./${vmdkname}.vmdk
```
Executing this script

output:
```ShellSession
root@lxc:/mnt/vmdk/2_125G# ../scripts/vmdk_extent_generate.sh
16+0 records in
16+0 records out
134217728 bytes (134 MB, 128 MiB) copied, 0.088087 s, 1.5 GB/s
16+0 records in
16+0 records out
134217728 bytes (134 MB, 128 MiB) copied, 0.0509471 s, 2.6 GB/s
16+0 records in
16+0 records out
134217728 bytes (134 MB, 128 MiB) copied, 0.0492777 s, 2.7 GB/s
16+0 records in
16+0 records out
134217728 bytes (134 MB, 128 MiB) copied, 0.04119 s, 3.3 GB/s
16+0 records in
16+0 records out
134217728 bytes (134 MB, 128 MiB) copied, 0.0426714 s, 3.1 GB/s
16+0 records in
16+0 records out
134217728 bytes (134 MB, 128 MiB) copied, 0.0595835 s, 2.3 GB/s
16+0 records in
16+0 records out
134217728 bytes (134 MB, 128 MiB) copied, 0.0725661 s, 1.8 GB/s
16+0 records in
16+0 records out
134217728 bytes (134 MB, 128 MiB) copied, 0.0598657 s, 2.2 GB/s
16+0 records in
16+0 records out
134217728 bytes (134 MB, 128 MiB) copied, 0.0625653 s, 2.1 GB/s
renamed './2_125G.new' -> './2_125G.vmdk'
```

Preventing the rename (mv) at the end of the script (what I did during development) allows to see the changes being made to the file.

```
root@lxc:/mnt/vmdk/2_125G# diff --side-by-side --report-identical-files 2_125G.vmdk 2_125G.new
# Disk DescriptorFile                                           # Disk DescriptorFile
version=1                                                       version=1
CID=dc0e0f14                                                    CID=dc0e0f14
parentCID=ffffffff                                              parentCID=ffffffff
createType="twoGbMaxExtentFlat"                                 createType="twoGbMaxExtentFlat"

# Extent description                                            # Extent description
RW 4194304 FLAT "2_125G-f001.vmdk" 0                          | RW 262144 FLAT "2_125G-f001.vmdk" 0
RW 262144 FLAT "2_125G-f002.vmdk" 0                             RW 262144 FLAT "2_125G-f002.vmdk" 0
                                                              > RW 262144 FLAT "2_125G-f003.vmdk" 0
                                                              > RW 262144 FLAT "2_125G-f004.vmdk" 0
                                                              > RW 262144 FLAT "2_125G-f005.vmdk" 0
                                                              > RW 262144 FLAT "2_125G-f006.vmdk" 0
                                                              > RW 262144 FLAT "2_125G-f007.vmdk" 0
                                                              > RW 262144 FLAT "2_125G-f008.vmdk" 0
                                                              > RW 262144 FLAT "2_125G-f009.vmdk" 0

# The Disk Data Base                                            # The Disk Data Base
#DDB                                                            #DDB

ddb.virtualHWVersion = "4"                                      ddb.virtualHWVersion = "4"
ddb.geometry.cylinders = "4421"                                 ddb.geometry.cylinders = "4421"
ddb.geometry.heads = "16"                                       ddb.geometry.heads = "16"
ddb.geometry.sectors = "63"                                     ddb.geometry.sectors = "63"
ddb.adapterType = "ide"                                         ddb.adapterType = "ide"
ddb.toolsVersion = "2147483647"                                 ddb.toolsVersion = "2147483647"
```


# Important References

These are SOME of the resources I used to learn and blindly meander through this process. 

## VeraCrypt
- [Create an encrypted container that is split over many files #1047](https://github.com/veracrypt/VeraCrypt/issues/1047)
    - [This is already quite trivial to do by assigning the containers as loop devices, then building a linear raid out of those loop devices](https://github.com/veracrypt/VeraCrypt/issues/1047#issuecomment-1553157538)
## VMDK
- [VMware Virtual Disks - Virtual Disk Format 1.1](https://web.archive.org/web/20191016053703/https://www.vmware.com/app/vmdk/?src=vmdk)
- [Virtual Disk Types](https://vdc-download.vmware.com/vmwb-repository/dcr-public/2bba164b-4115-4279-9c99-40f4c14319ad/03a845fc-5345-45de-9a27-31e868d6e751/doc/vddkDataStruct.5.3.html)
- [VMDK Handbook - Basics](https://sanbarrow.com/vmdk-basics.html)
## Qemu
- [Mounting VMDK disk image](https://stackoverflow.com/a/49000377)
## Docker
- [Runtime privilege and Linux capabilities](https://docs.docker.com/engine/containers/run/#runtime-privilege-and-linux-capabilities)
- [Linux user namespace on all containers](https://docs.docker.com/security/for-admins/hardened-desktop/enhanced-container-isolation/features-benefits/#privileged-containers-are-also-secured)
- [Docker loading kernel modules](https://stackoverflow.com/a/33017933)
- [nbd-client and nbd-server in docker container: "Couldn't resolve the nbd netlink family"](https://unix.stackexchange.com/a/685020)
## ZFS
- [Is it possible to split stream created by "zfs send" between multiple harddrives?](https://www.reddit.com/r/zfs/comments/oeq17q/is_it_possible_to_split_stream_created_by_zfs/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button)
- [Trying to zfs send my whole filesystem to AWS/S3 but running into maximum 5TB single file size issue. How do I work around this?](https://www.reddit.com/r/zfs/comments/8zxjdw/trying_to_zfs_send_my_whole_filesystem_to_awss3/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button)
- [Best compression for ZFS send/recv](https://serverfault.com/a/696201)
- [OpenZFS / Performance and Tuning / Workload Tuning / ](https://openzfs.github.io/openzfs-docs/Performance%20and%20Tuning/Workload%20Tuning.html#virtual-machines)
    - [16K volblocksize should be recommended in the ZFS documentation? #14771](https://github.com/openzfs/zfs/issues/14771#issuecomment-1515492430)