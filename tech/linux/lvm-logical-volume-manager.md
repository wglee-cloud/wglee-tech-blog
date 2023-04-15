---
description: Written on 2021. 9. 1. 03:07
---

# LVM(Logical Volume Manager) 의 개념과 설정 방법

## LVM이란

LVM(Logical Volume Manager)는 리눅스의 저장 공간을 효율적이고 유연하게 관리하기 위한 커널의 한 부분이다.

&#x20;

## LVM vs. 일반 disk partitioning

LVM이 아닌 기존 방식의 경우, 하드 디스크를 파티셔닝 한 후 OS 영역에 마운트하여 read/wirte를 수행했다.\
이 경우 저장 공간의 크기가 고정되어서 증설/축소가 어렵다. 이를 보완하기 위한 방법으로 LVM을 구성할 수 있다.\
LVM은 파티션 대신에 volume이라는 단위로 저장 장치를 다룬다.\
스토리지의 확장,변경에 유연하며, 크기를 변경할 때 기존 데이터의 이전이 필요 없다.

&#x20;

## LVM 사용의 장점

* 유연한 용량 조절
* 크기 조절이 가능한 storage pool
* 편의에 따른 장치 이름 지정
* disk striping, mirror volume등을 제공

&#x20;

## LVM 관련 용어 및 구성

<figure><img src="https://blog.kakaocdn.net/dn/v2dRZ/btrdFmw1kHd/ATWeqg7alB5ZKZXea9hcxK/img.png" alt=""><figcaption></figcaption></figure>

<figure><img src="https://blog.kakaocdn.net/dn/cEzAYL/btrKTpBcia9/DR41zDF1ZNVGgXFTFhezaK/img.png" alt=""><figcaption><p>image from&#x26;nbsp;https://www.thegeekdiary.com/redhat-centos-a-beginners-guide-to-lvm-logical-volume-manager/</p></figcaption></figure>

&#x20;

### 물리적 볼륨 / PV (Physical Volume)

\- 실제 디스크 장치를 분할한 파티션된 상태를의미한다.\
\- PV는 일정한 크기의 PE들로 구성된다.\


### 물리적 확장 / PE (Physical Extent)

\- PV를 구성하는 일정한 크기의 Block.\
\- 보통 1PE는 4MB에 해당한다.\
\- PE와 LE는 1:1로 대응한다.\


### 볼륨 그룹 / VG (Volume Group)

\- PV들이 모여서 생성되는 단위이다. (모든걸 합친 거대한 지점토 덩어리의 느낌이다)\
\- 사용자는 VG를 원하는대로 쪼개서 LV로 만들게 된다.\


### 논리적 볼륨 / LV (Logical Volume)

\- 사용자가 최종적으로 사용하는 단위로, VG에서 필요한 크기로 할당받아 LV를 생성한다.

간단한 실습을 진행해 보자!

&#x20;

## LVM 구성하기

### 가상서버에 Attach한 볼륨 확인

```csharp
[centos@wglee-server ~]$ sudo fdisk -l

Disk /dev/vda: 32.2 GB, 32212254720 bytes, 62914560 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000b6061

   Device Boot      Start         End      Blocks   Id  System
/dev/vda1   *        2048    62914526    31456239+  83  Linux

Disk /dev/vdb: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/vdc: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

&#x20;

### lvm package 설치

```csharp
[centos@wglee-server ~]$ sudo yum install lvm2
```



### PV 만들기

블록 스토리지 리스트 확인

```csharp
[centos@wglee-server ~]$ lsblk 
NAME     MAJ:MIN    RM  SIZE RO TYPE MOUNTPOINT 
vda     253:0       0    30G 0 disk 
-vda1   253:1       0    30G 0 part / 
vdb     253:16      0    10G 0 disk 
vdc     253:32      0    10G 0 disk
```

#### 블록 디바이스 초기화

pvcreate : LVM에 사용될 파티션을 생성하기 위해 물리 디스크를 초기화 한다.

```csharp
# vdb와 vdc 볼륨에 대해 pv를 생성한다.
[centos@wglee-server ~]$ sudo pvcreate /dev/vdb
  Physical volume "/dev/vdb" successfully created.
[centos@wglee-server ~]$ sudo pvcreate /dev/vdc
  Physical volume "/dev/vdc" successfully created.
```

#### &#x20;생성한 PV 확인 (pvdisplay)

```csharp
[centos@wglee-server ~]$ sudo pvdisplay
"/dev/vdc" is a new physical volume of "10.00 GiB"
--- NEW Physical volume ---
PV Name               /dev/vdc
VG Name
PV Size               10.00 GiB
Allocatable           NO
PE Size               0
Total PE              0
Free PE               0
Allocated PE          0
PV UUID               oGJ5YP-EVJo-VJij-nC4o-R1d0-qQz7-QN06by

"/dev/vdb" is a new physical volume of "10.00 GiB"
--- NEW Physical volume ---
PV Name               /dev/vdb
VG Name
PV Size               10.00 GiB
Allocatable           NO
PE Size               0
Total PE              0
Free PE               0
Allocated PE          0
PV UUID               MHxcc6-RJSz-khM2-0V3L-g1Ia-hJb3-MF6rfJ
```

&#x20;

### VG 만들기

#### VG 생성하기 (vgcreate)

다음과 같이 각각의 PV 를 이용하여 VG 를 생성했다.

```csharp
# vgcreate [vg이름] [블록스토리지 경로]
[centos@wglee-server ~]$ sudo vgcreate vg1 /dev/vdb
  Volume group "vg1" successfully created
[centos@wglee-server ~]$ sudo vgcreate vg2 /dev/vdc
  Volume group "vg2" successfully created
```

#### VG 삭제하기 (vgremove)

각 PV를 각각의 VG로 만들기 보다 하나의 VG로 사용해 보고 싶어졌다.\
그래서 v2를 삭제하는 작업을 했다.

```csharp
[centos@wglee-server ~]$ sudo vgremove vg2
  Volume group "vg2" successfully removed
```

#### VG 확장하기 (vgextend)

vg1에 /dev/vdc 영역을 추가하여 확장하도록 한다.

```csharp
[centos@wglee-server ~]$ sudo vgextend vg1 /dev/vdc
Volume group "vg1" successfully extended
```

#### VG 조회 (vgdisplay)

VG1의 사이즈가 20GB인 것을 볼 수 있다. /etc/vdb와 /etc/vdc가 논리적으로 합쳐져서 하나의 묶음이 되었다.

```csharp
#VG size가 늘어났다.
    [centos@wglee-server ~]$ sudo vgdisplay
      --- Volume group ---
      VG Name               vg1
      System ID
      Format                lvm2
      Metadata Areas        2
      Metadata Sequence No  2
      VG Access             read/write
      VG Status             resizable
      MAX LV                0
      Cur LV                0
      Open LV               0
      Max PV                0
      Cur PV                2
      Act PV                2
      VG Size               19.99 GiB
      PE Size               4.00 MiB
      Total PE              5118
      Alloc PE / Size       0 / 0
      Free  PE / Size       5118 / 19.99 GiB
      VG UUID               It5WRf-7NyL-fjYi-2Ilc-ubF6-6HaJ-AuPnkU
```

&#x20;

### LV 만들기

#### LV 생성하기 (lvcreate)

LV는 2개 생성해 보도록 한다.\
lvcreate 명령어로 LV를 생성한다. vg1의 리소스를 wglv\_1과 wglv\_2로 논리적으로 나눈 것을 볼 수 있다.

```csharp
# lvcreate -n [LV이름] -L [LV용량] [VG이름].
[centos@wglee-server ~]$ sudo lvcreate -n wglv_1 -L 15G vg1
  Logical volume "wglv_1" created.
[centos@wglee-server ~]$ sudo lvcreate -n wglv_2 -L 4.9G vg1
  Rounding up size to full physical extent 4.90 GiB
  Logical volume "wglv_2" created.
```

#### 생성된 LV 확인

```csharp
# 생성된 LV 확인
    [centos@wglee-server ~]$ sudo lvdisplay
      --- Logical volume ---
      LV Path                /dev/vg1/wglv_1
      LV Name                wglv_1
      VG Name                vg1
      LV UUID                mKRGia-Hikh-orZ0-CFlw-xMPa-jERA-zsA0Cn
      LV Write Access        read/write
      LV Creation host, time wglee-server.novalocal, 2021-01-17 21:49:41 +0900
      LV Status              available
      # open                 0
      LV Size                15.00 GiB
      Current LE             3840
      Segments               2
      Allocation             inherit
      Read ahead sectors     auto
      - currently set to     256
      Block device           252:0

      --- Logical volume ---
      LV Path                /dev/vg1/wglv_2
      LV Name                wglv_2
      VG Name                vg1
      LV UUID                Pcy1Zz-THrt-LCCE-JYTC-zeGb-hOI3-F8gviF
      LV Write Access        read/write
      LV Creation host, time wglee-server.novalocal, 2021-01-17 21:49:51 +0900
      LV Status              available
      # open                 0
      LV Size                4.90 GiB
      Current LE             1255
      Segments               1
      Allocation             inherit
      Read ahead sectors     auto
      - currently set to     256
      Block device           252:1
```

&#x20;&#x20;

### 파일 시스템  생성 및 Mount

LV를 실제로 사용할 수 있도록 파일 시스템과 디렉터리 영역을 연결하도록 한다.\
마운트를 하기에 앞서서 lsblk 명령어로 디바이스 정보를 확인한다.

```csharp
    # 방금 생성한 LV는 타입이 lvm이다. 

    [centos@wglee-server ~]$ lsblk
    NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    vda          253:0    0   30G  0 disk
    `-vda1       253:1    0   30G  0 part /
    vdb          253:16   0   10G  0 disk
    `-vg1-wglv_1 252:0    0   15G  0 lvm
    vdc          253:32   0   10G  0 disk
    |-vg1-wglv_1 252:0    0   15G  0 lvm
    `-vg1-wglv_2 252:1    0  4.9G  0 lvm
```

&#x20;실제 LV의 위치는 /dev/vg1 디렉터리에 있다.

```csharp
    # 실제 LV의 위치는 /dev/vg1에 있다.
    [centos@wglee-server ~]$ ls /dev/vg1
    wglv_1  wglv_2

    # 참고 ) VG에 매핑된 LV은 다음 경로에서 확인된다. 
    [centos@wglee-server ~]$ ls /dev/mapper
    control  vg1-wglv_1  vg1-wglv_2
```

&#x20;File System을 생성한다. 지금은 ext2 타입으로 생성했다.&#x20;

```csharp
    [centos@wglee-server ~]$ sudo mkfs.ext4 /dev/vg1/wglv_1
    mke2fs 1.42.9 (28-Dec-2013)
    Filesystem label=
    OS type: Linux
    Block size=4096 (log=2)
    Fragment size=4096 (log=2)
    Stride=0 blocks, Stripe width=0 blocks
    983040 inodes, 3932160 blocks
    196608 blocks (5.00%) reserved for the super user
    First data block=0
    Maximum filesystem blocks=2151677952
    120 block groups
    32768 blocks per group, 32768 fragments per group
    8192 inodes per group
    Superblock backups stored on blocks:
            32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208

    Allocating group tables: done
    Writing inode tables: done
    Creating journal (32768 blocks): done
    Writing superblocks and filesystem accounting information: done 
```

LV를 마운트 할 디렉터리 생성

```csharp
    # mkdir -p 옵션 : 중간 디렉토리도 자동 생성한다.
    [centos@wglee-server ~]$ sudo mkdir -p /data/wglv_1
    [centos@wglee-server ~]$ sudo mkdir -p /data/wglv_2
```

mount point에 마운트

```csharp
    # mount [option] [device] [directory]
    [centos@wglee-server ~]$ sudo mount /dev/vg1/wglv_1 /data/wglv_1
    [centos@wglee-server ~]$ sudo mount /dev/vg1/wglv_2 /data/wglv_2
    [centos@wglee-server ~]$ lsblk
    NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    vda          253:0    0   30G  0 disk
    `-vda1       253:1    0   30G  0 part /
    vdb          253:16   0   10G  0 disk
    `-vg1-wglv_1 252:0    0   15G  0 lvm  /data/wglv_1
    vdc          253:32   0   10G  0 disk
    |-vg1-wglv_1 252:0    0   15G  0 lvm  /data/wglv_1
    `-vg1-wglv_2 252:1    0  4.9G  0 lvm  /data/wglv_2
```

&#x20;

### fstab 등록

물론 재부팅 시에도 마운트 상태를 유지하려면 fstab에 정보를 등록해야 한다.

&#x20;

## **참고 문서**

* [https://access.redhat.com/documentation/ko-kr/red\_hat\_enterprise\_linux/6/html/logical\_volume\_manager\_administration/doc\_organization](https://access.redhat.com/documentation/ko-kr/red\_hat\_enterprise\_linux/6/html/logical\_volume\_manager\_administration/doc\_organization)
* [https://tech.cloud.nongshim.co.kr/2018/11/23/lvmlogical-volume-manager-1-%EA%B0%9C%EB%85%90/](https://tech.cloud.nongshim.co.kr/2018/11/23/lvmlogical-volume-manager-1-%EA%B0%9C%EB%85%90/)
* [https://tech.cloud.nongshim.co.kr/2018/11/28/lvmlogical-volume-manager-linux-%EC%9D%B8%EC%8A%A4%ED%84%B4%EC%8A%A4%EC%97%90%EC%84%9C-%ED%99%9C%EC%9A%A9%ED%95%98%EA%B8%B01-2/](https://tech.cloud.nongshim.co.kr/2018/11/28/lvmlogical-volume-manager-linux-%EC%9D%B8%EC%8A%A4%ED%84%B4%EC%8A%A4%EC%97%90%EC%84%9C-%ED%99%9C%EC%9A%A9%ED%95%98%EA%B8%B01-2/)

