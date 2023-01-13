---
description: Written on 2022. 2. 27. 13:06
---

# 파일 시스템 생성 및 자동 마운트 설정하기

## Intro

오늘은 파일 시스템을 생성하는 일련의 과정과 부팅시에도 자동 마운트가 되도록 fstab에 등록하는 방법에 대해 알아보도록 한다.

이전에도 이런 과정을 경험한적은 있지만, 제대로 다시 정리해보자!&#x20;



## 파일시스템이란?

파일 시스템(File System)은 운영체제가 파티션이나 디스크에 데이터를 저장/읽기/찾기를 하기 위해 구성하는 체계를 의미한다. 리눅스를 다루다 보면 데이터가 파일 단위로 저장되고, 파일은 디렉터리에 속하게 된다.

이러한 파일들과 계층 구조를 관리하는 시스템을 파일 시스템이라고 생각할 수 있다.

파일 시스템은 파일 생성/수정/삭제 기능, 파일 접근 제어, 파일 백업/복구 기능, 정보의 암호화/복호화 등을 수행한다.

따라서 새롭게 하드 디스크를 추가한 하드 디스크에는 파일을 어떻게 기록하고 저장할지에 대한 정의가 되어 있지 않다.

이를 설정하기 위해서 디스크를 Formatting(=파일 시스템 생성) 해야 한다.&#x20;

이제 본격적으로 파일 시스템 생성 및 마운트를 수행해 본다.



## Hands-on

### 하드 디스크 인식 확인

나는 가상서버에 10GB의 블록 스토리지를 연결해서 진행했다. 디스크는 자동 인식 되기 때문에 다음 명령어로 장치 파일명을 확인하도록 한다. 이 같은 경우 장치명이/dev/vdb 임을 알 수 있다.\
(이때 여담으로 'fdisk -l'과 'df -T' 명령어의 차이를 짚어보자. 'fdisk'는 mount 되지 않은 디스크도 출력하지만 'df'는 mount 된 디스크만 출력한다.)

```shell-session
[root@wglee ~]# fdisk -l

Disk /dev/vda: 32.2 GB, 32212254720 bytes, 62914560 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000b0d11

   Device Boot      Start         End      Blocks   Id  System
/dev/vda1   *        2048    62914526    31456239+  83  Linux

Disk /dev/vdb: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes​
```



### 파티션 분할 및 생성

파티션(Partition)은 대용량 디스크를 하나로 다룰 경우 유휴 공간이 생기는 것을 방지하기 위해 디스크를 쪼갠 단위이다. 각각의 파티션은 별개의 디스크처럼 취급된다. 나는 /dev/vdb1, /dev/vdb2를 생성했다.&#x20;

| 옵션 | 용도                    |
| -- | --------------------- |
| p  | 현재 디스크 정보 출력          |
| d  | 파티션 삭제                |
| n  | 파티션 새롭게 생성(추가)        |
| w  | 변경된 파티션 정보 저장하고 종료    |
| q  | 변경된 파티션 정보 저장하지 않고 종료 |

```shell-session
[root@wglee ~]# fdisk /dev/vdb
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0x4b81fe4a.

Command (m for help): p

Disk /dev/vdb: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x4b81fe4a

   Device Boot      Start         End      Blocks   Id  System



Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p):
Using default response p
Partition number (1-4, default 1):
First sector (2048-20971519, default 2048):
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-20971519, default 20971519): 5368709
Partition 1 of type Linux and of size 2.6 GiB is set

Command (m for help): p

Disk /dev/vdb: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x4b81fe4a

   Device Boot      Start         End      Blocks   Id  System
/dev/vdb1            2048     5368709     2683331   83  Linux

Command (m for help): p

Disk /dev/vdb: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x4b81fe4a

   Device Boot      Start         End      Blocks   Id  System
/dev/vdb1            2048     5368709     2683331   83  Linux

Command (m for help): n
Partition type:
   p   primary (1 primary, 0 extended, 3 free)
   e   extended
Select (default p):
Using default response p
Partition number (2-4, default 2):
First sector (5368710-20971519, default 5369856):
Using default value 5369856
Last sector, +sectors or +size{K,M,G} (5369856-20971519, default 20971519):
Using default value 20971519
Partition 2 of type Linux and of size 7.5 GiB is set

Command (m for help): p

Disk /dev/vdb: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x4b81fe4a

   Device Boot      Start         End      Blocks   Id  System
/dev/vdb1            2048     5368709     2683331   83  Linux
/dev/vdb2         5369856    20971519     7800832   83  Linux


Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```

\
위와 같이 파티셔닝을 한 결과는 다음과 같이 확인할 수 있다.

```shell-session
[root@wglee ~]# fdisk -l

Disk /dev/vda: 32.2 GB, 32212254720 bytes, 62914560 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000b0d11

   Device Boot      Start         End      Blocks   Id  System
/dev/vda1   *        2048    62914526    31456239+  83  Linux

Disk /dev/vdb: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x4b81fe4a

   Device Boot      Start         End      Blocks   Id  System
/dev/vdb1            2048     5368709     2683331   83  Linux
/dev/vdb2         5369856    20971519     7800832   83  Linux​
```



### 파일 시스템 생성&#x20;

이는 포맷(Format)이라고 부르기도 한다. 원하는 파일 시스템으로 Formatting 할 수 있는데, 주로 쓰이는 파일 시스템 종류는 다음과 같다.&#x20;

| 파일 시스템 | 특징                                                |
| ------ | ------------------------------------------------- |
| ext3   | ext2의 확장판으로, 리눅스의 대표적인 저널링 파일 시스템. ACL 접근 제어를 제공. |
| ext4   | ext2, ext3과 호환성이 있다. 대용량 파일을 지원한다.                |
| XFS    | SGI에서 개발한 저널링 파일 시스템.                             |
| NFS    | Network File System, 네트워크 상의파일을 공유할 때 쓰이는 파일 시스템. |

{% hint style="info" %}
**Journaling** \
****저널링은 데이터를 디스크에 쓰기 전에 로그(Log)에 데이터를 남기는 기술이다. 복구가 필요한 상황에서 로그를 읽어 fsck 보다 빠르고 안정적으로 복구할 수 있도록 한다. \
저널링이 도입되지 않은 파일 시스템의 경우, fsck 명령어로 슈퍼 블록, 비트맵, 아이노드 등 파일 시스템 전체를 검사하여 복구해야 하는데 이는 시간이 많이 걸린다.\
저널링 파일 시스템은 로그를 읽으면 되기 때문에 속도가 더 빠르다는 장점이 있다.
{% endhint %}

파일 시스템을 생성하는 명령어에는 mkfs, mke2fs가 있다. 두 명령어 모두 -t 옵션으로 파일 시스템 유형을 지정 가능하나, mke2fs 경우에는 xfs를 지원하지 않는다. (ext2, ext3, ext4)

```shell-session
[root@wglee ~]# mke2fs -t ext3 /dev/vdb1
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
168000 inodes, 670832 blocks
33541 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=687865856
21 block groups
32768 blocks per group, 32768 fragments per group
8000 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done​
```



### Mount

이렇게 생성한 파일 시스템을 사용하기 위해서는 mount 작업을 하여 특정 디렉터리와 연결해야 한다. \
다만 이렇게 mount 한 것은 영구적이지 않다. mount를 한 후 df로 보면 각 파일 시스템이 잘 마운트 되어 있다. 이제 확장한 디스크 영역에 파일을 생성, 삭제할 수 있다. \
(나는 테스트 용으로 아까 ext3, xfs 로 각각 포맷을 했는데 두 파일 시스템 사이에 데이터 교환 등이 원활하게 이뤄지는지 테스트 해보고 싶었다.)&#x20;

```shell-session
[root@wglee ~]# mount -t ext3 /dev/vdb1 /etc/mount1

[root@wglee ~]# mkfs.xfs /dev/vdb2
meta-data=/dev/vdb2              isize=512    agcount=4, agsize=487552 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=1950208, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

[root@wglee ~]# mount /dev/vdb2 /etc/mount2

[root@wglee ~]# df -hT
Filesystem     Type      Size  Used Avail Use% Mounted on
devtmpfs       devtmpfs  473M     0  473M   0% /dev
tmpfs          tmpfs     496M     0  496M   0% /dev/shm
tmpfs          tmpfs     496M   13M  483M   3% /run
tmpfs          tmpfs     496M     0  496M   0% /sys/fs/cgroup
/dev/vda1      xfs        30G  842M   30G   3% /
tmpfs          tmpfs     100M     0  100M   0% /run/user/1000
/dev/vdb1      ext3      2.5G  4.0M  2.4G   1% /etc/mount1
/dev/vdb2      xfs       7.5G   33M  7.4G   1% /etc/mount2​
```



### 부팅시 자동 마운트 설정

그냥 mount 명령어로만 마운트를 한 경우 가상서버가 재부팅되면 기존 설정이 날아간다. 때문에 부팅시 자동 마운트를 설정하려면 /etc/fstab에 설정을 해줘야 한다.\
/etc/fstab은 파일 시스템에 대한 다양한정보를 담고 있다. mount, umount, fsck 등이 해당 파일을 참조해서 동작한다.&#x20;

```shell-session
[root@wglee ~]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Fri Oct 30 14:22:27 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=9cff3d69-3769-4ad9-8460-9c54050583f9 /                       xfs     defaults        0 0
```

&#x20;필드 구성은 다음과 같다. (총6개)

| 필드   | 내용                                                                                                                                                                               |
| ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 첫번째  | <p>장치명을 기록하는 필드로, /dev/vdb와 같은 디바이스 명이나 볼륨 라벨, UUID를 적을 수 있다.<br>주의할 점은 디바이스명은 항상 일관되지 않아 장치를 고유하게 식별하기에 적합하지 않다.<br>그래서 고유한 UUID로 마운트 하는 것이 좋다. (장치의 UUID는 "blkid" 명령어로 확인)</p> |
| 두번째  | Mount Point 정보. OS에서 해당 디바이스를 마운트할 위치를 의미한다.                                                                                                                                     |
| 세번째  | 파일 시스템의 유형                                                                                                                                                                       |
| 네번째  | <p>마운트될 때의 옵션으로, defaults, auto, usrquota, grpqouta 등이 있다. <br>defaults로 설정할 경우 rw(read-write) 권한이 부여된다.</p>                                                                     |
| 다섯번째 | dump 명령으로 백업할 주기를 결정. 0이면 백업하지 않으며, 1이면 하루, 2이면 이틀에 한번 수행한다.                                                                                                                     |
| 여섯번째 | <p>부팅시 파일 시스템을 점검하는 fsck 명령으로 체크할 순서를 지정한다. <br>0이면 아예 검사하지 않고, 1이번 첫번째, 2이면 두번째인 식이다. </p>                                                                                      |

이제 fstab에 등록할 장치의 UUID를 먼저 확인한다.

```shell-session
[root@wglee ~]# blkid
/dev/vda1: UUID="9cff3d69-3769-4ad9-8460-9c54050583f9" TYPE="xfs"
/dev/vdb1: UUID="f1b9d1d2-87b0-426e-ad49-22e355763d53" TYPE="ext3"
/dev/vdb2: UUID="4388fdc4-c6e4-4521-bde1-cc2bffd7576a" TYPE="xfs"
```

fstab에 /dev/vdb1과 /dev/vdb2의 UUID를 사용하여 마운트할 위치를 지정한다. 디바이스명은 향후 작업에 따라 변동될 수 있기 때문에 고유ID인 UUID를 사용하는 것이 좋다.

```shell-session
[root@wglee /etc/mount1]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Fri Oct 30 14:22:27 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=9cff3d69-3769-4ad9-8460-9c54050583f9 /                       xfs     defaults        0 0
UUID=f1b9d1d2-87b0-426e-ad49-22e355763d53 /etc/mount1             ext3    defaults        0 0
UUID=4388fdc4-c6e4-4521-bde1-cc2bffd7576a /etc/mount2             xfs     defaults        0 0
```

### 재부팅

/etc/fstab에 등록했었기 때문에 재부팅 한 후에도 새롭게 생성한 파일 시스템이 마운트 되어 있는 것을 볼 수 있다.

```shell-session
[root@wglee ~]# reboot

[root@wglee ~]# df -Th
Filesystem     Type      Size  Used Avail Use% Mounted on
devtmpfs       devtmpfs  473M     0  473M   0% /dev
tmpfs          tmpfs     496M     0  496M   0% /dev/shm
tmpfs          tmpfs     496M   13M  483M   3% /run
tmpfs          tmpfs     496M     0  496M   0% /sys/fs/cgroup
/dev/vda1      xfs        30G  1.1G   29G   4% /
/dev/vdb2      xfs       7.5G   33M  7.4G   1% /etc/mount2
/dev/vdb1      ext3      2.5G  4.0M  2.4G   1% /etc/mount1
tmpfs          tmpfs     100M     0  100M   0% /run/user/1000
```

****

## **참고문서**

{% embed url="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/5/html/installation_guide/ch-partitions-x86#fig-partitions-other-formatted-d-x86" %}

