---
description: Written on 2022. 3. 1. 14:52
---

# Swap Memory 개념 및 swap partition, file 생성 방법

## SWAP Memory?

1. Swap 공간은 RAM이 모두 찼을 때 RAM에서 잘 사용하지 않는 page들을 옮겨 두는 disk 공간이다.
2. Hard Drive (디스크)에 위치하기 때문에 RAM 보다 느리다.
3. Swap 공간은 Swap 파티션(권장), Swap 파일, Swap 파티션 + Swap 파일의 방법으로 구성할 수 있다.

**하지만 요즘은 RAM의 공급과 사양이 예전에 비해 나아 지면서 굳이 성능을 하락시키는 swap을 사용하지 않는 추세이다.**&#x20;

{% hint style="info" %}
**Swap Partition과 Swap File의 차이점?**\
****\
******A swap partition** is just what its name implies—a standard disk partition that is designated as swap space by the mkswap command.

**A swap file** can be used if there is no free disk space in which to create a new swap partition or space in a volume groupwhere a logical volume can be created for swap space. This is just a regular file that is created and preallocated to a specified size.Then the mkswap command is run to configure it as swap space. I don’t recommend using a file for swap space unless absolutely necessary.



즉, swap file은 더 이상 Swap space로 할당할 수 있는 디스크 공간이 없을 때 사용하는 것으로, 일반적인 file을 만든 다음에 mkswap 명령어로 swap 공간으로 지정한다. 꼭 필요하지 않는 이상 권장하지 않음.
{% endhint %}

####

## RAM 크기에 따라 권장되는 Swap 공간

권장 swap 공간은 OS 설치 중에 자동으로 설정 된다.

하지만 hibernation 모드를 사용하는 경우에는 사용자가 swap 공간을 수동 편집해야 한다.\
_hibernation?_ : 시스템 전원을 끄기 전에 시스템 메모리에 있는 모든 내용을 Hard disk 와 같은 비활성 메모리에 기록하는 것.

Redhat 문서에서 제공하는 권장 사양은 다음과 같다.

| 시스템의 RAM 용량    | 권장 Swap space | Hibernation 을 사용할 때의 권장 Swap space |
| -------------- | ------------- | ---------------------------------- |
| ⩽ 2 GB         | RAM의 두 배      | RAM의 3배                            |
| > 2 GB – 8 GB  | RAM과 동일하게     | RAM의 2배                            |
| > 8 GB – 64 GB | 최소 4GB        | RAM의 1.5배                          |
| > 64 GB        | 최소 4GB        | Hibernation 권장하지 않음                |



## Swap Partition 생성하기

나의 경우에는 [이전에 생성한 LVM 파일 시스템](lvm-logical-volume-manager.md)을 사용하여 swap 파티션을 생성했다. mkswap 명령어를 이용해여 /dev/vg1/wglv\_1 영역을 swap 공간으로 지정한다.

```shell-session
# 이미 filesystem에 마운트 해 놓아서 swap space로 만들 수 없다.
[centos@wglee-server ~]$ sudo mkswap /dev/vg1/wglv_1
mkswap: error: /dev/vg1/wglv_1 is mounted; will not make swap space


# umount
[centos@wglee-server ~]$ sudo umount /dev/vg1/wglv_1


# format the new swap space
[centos@wglee-server ~]$ sudo mkswap /dev/vg1/wglv_1
mkswap: /dev/vg1/wglv_1: warning: wiping old ext4 signature.
Setting up swapspace version 1, size = 15728636 KiB
no label, UUID=a02d074d-417e-4381-b64b-57032f996d8c


# edit /etc/fstab file
[centos@wglee-server ~]$ cat vi /etc/fstab
cat: vi: No such file or directory
UUID=3ef2b806-efd7-4eef-aaa2-2584909365ff /                       xfs     defaults        0 0
#UUID=6028b3e9-4014-45fe-bac7-7e6f53659d46 /data/wglv_1            ext4    defaults,nofail 0 0
UUDI=a02d074d-417e-4381-b64b-57032f996d8c  swap                   swap    defaults        0 0
UUID=653f9039-81c5-43bd-82c7-40245e7c801e /data/wglv_2            ext4    defaults,nofail 0 0


# daemon-reload를 하여 fstab에 등록한 내용으로 mount unit 정보를 갱신한다. (데몬 재시작과는 다르다)
[centos@wglee-server ~]$ sudo systemctl daemon-reload


# 해당 LV에 대해 swap on 시킨다.
[centos@wglee-server ~]$ sudo swapon -v /dev/vg1/wglv_1
swapon /dev/vg1/wglv_1
swapon: /dev/mapper/vg1-wglv_1: found swap signature: version 1, page-size 4, same byte order
swapon: /dev/mapper/vg1-wglv_1: pagesize=4096, swapsize=16106127360, devsize=16106127360


# 다시 swap device를 검색한다. /dev/dm-0가 새롭게 생긴걸 확인 할 수 있다.
[centos@wglee-server ~]$ cat /proc/swaps
Filename                                Type            Size    Used    Priority
/dev/dm-0                               partition       15728636        0       -2


# free -h 로 확인
[centos@wglee-server ~]$ free -h
              total        used        free      shared  buff/cache   available
Mem:           991M        103M        351M         56M        535M        653M
Swap:           14G          0B         14G
--> wglv_1에 대한 14G가 swap space가 되었다!
```

이제 블록 디바이스 정보를 한번 확인하면 swap 으로 지정한 파일시스템은 \[SWAP]으로 보이고 있다.

```shell-session
[centos@wglee-server ~]$ lsblk
NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda          253:0    0   30G  0 disk
`-vda1       253:1    0   30G  0 part /
vdb          253:16   0   10G  0 disk
`-vg1-wglv_1 252:0    0   15G  0 lvm  [SWAP]
vdc          253:32   0   10G  0 disk
|-vg1-wglv_1 252:0    0   15G  0 lvm  [SWAP]
`-vg1-wglv_2 252:1    0  4.9G  0 lvm  /data/wglv_2
```

{% hint style="info" %}
**daemon-reload** \
****중간에 /etc/fstab을 수정한 다음에 daemon-reload를 하였다. restart 등과 어떤 차이가 있는지 알아보자.

" Reload systemd manager configuration. This will rerun all generators (see systemd.generator(7)), reload all unit files, and recreate the entire dependency tree. While the daemon is being reloaded, all sockets systemd listens on behalf of user configuration will stay accessible. This command should not be confused with the reload command. (daemon-reload won't reload/restart the services themselves, just makes systemd aware of the new configuration) "



즉, daemon-reload는 systemd 서비스 자체를 재구동 하는 것이 아니라 systemd가 변경된 환경 설정을 인지하도록 하는 명령어임을 알 수 있다.
{% endhint %}

####

## Swap File 생성하기

생성되는 swap file의 block size는 파일크기(MB) \* 1024 로 계산하면 된다.\
즉, 약 60MB의 파일을 생성하려고 한다면 61440로 block size를 지정하면 된다.

```shell-session
# 난 60MB의 swap file을 만들어야지! (count의 값은 block size를 계산하여 적는다)
[centos@wglee-server ~]$ sudo dd if=/dev/zero of=/swapfile bs=1024 count=61440
61440+0 records in
61440+0 records out
62914560 bytes (63 MB) copied, 2.40773 s, 26.1 MB/s


# swapfile 설정한다.
[centos@wglee-server ~]$ sudo mkswap /swapfile
Setting up swapspace version 1, size = 61436 KiB
no label, UUID=ddee20c0-5a99-4695-a8b1-7610352f209b

# 권한 변경
[centos@wglee-server ~]$ sudo chmod 0600 /swapfile


[centos@wglee-server ~]$ ls /
bin   data  etc   lib    media  opt   root  sbin  swapfile  tmp  var
boot  dev   home  lib64  mnt    proc  run   srv   sys       usr


# /etc/fstab에 등록
[centos@wglee-server ~]$ cat /etc/fstab
UUID=3ef2b806-efd7-4eef-aaa2-2584909365ff /                       xfs     defaults        0 0
#UUID=6028b3e9-4014-45fe-bac7-7e6f53659d46 /data/wglv_1            ext4    defaults,nofail 0 0
UUDI=a02d074d-417e-4381-b64b-57032f996d8c  swap                   swap    defaults        0 0
UUID=653f9039-81c5-43bd-82c7-40245e7c801e /data/wglv_2            ext4    defaults,nofail 0 0
UUID=ddee20c0-5a99-4695-a8b1-7610352f209b swap                    swap    defaults        0 0


# fstab을 변경했기 때문에 damon-reload를 한다.
[centos@wglee-server ~]$ sudo systemctl daemon-reload


# 생성한 swapfile에 대해 swap on 시킨다.
[centos@wglee-server ~]$ sudo swapon /swapfile


# 최종 결과 확인하기
[centos@wglee-server ~]$ cat /proc/swaps
Filename                                Type            Size    Used    Priority
/dev/dm-0                               partition       15728636        0       -2
/swapfile                               file            61436   0       -3


[centos@wglee-server ~]$ free -h
              total        used        free      shared  buff/cache   available
Mem:           991M        105M        288M         56M        597M        650M
Swap:           15G          0B         15G



[centos@wglee-server ~]$ free -m
              total        used        free      shared  buff/cache   available
Mem:            991         105         288          56         597         651
Swap:         15419           0       15419
```



## Swap Space 줄이기/삭제 하기

Swap 공간 크기를 줄이거나 삭제하는 방법은 다음과 같다.

1. Swap 영역으로 사용된 LVM2 logical volume 전체를 제거
2. swap 파일 제거
3. 기존 LVM2 Logical volume에서 사용하는 swap 공간을 줄이기

이번에는 기존 LV 에서 스왑 공간을 줄이는 방법을 해본다.

```shell-session
# 해당 LV Disable swapping 처리
[centos@wglee-server ~]$ sudo swapoff -v /dev/vg1/wglv_1
swapoff /dev/vg1/wglv_1

# LV 크기 줄이기
[centos@wglee-server ~]$ sudo lvreduce /dev/vg1/wglv_1 -L -1G
  WARNING: Reducing active logical volume to 14.90 GiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce vg1/wglv_1? [y/n]: y
  Size of logical volume vg1/wglv_1 changed from 15.00 GiB (3840 extents) to 14.90 GiB (3815 extents).
  Logical volume vg1/wglv_1 successfully resized.


# format the new swap space
[centos@wglee-server ~]$ sudo mkswap /dev/vg1/wglv_1
mkswap: /dev/vg1/wglv_1: warning: wiping old swap signature.
Setting up swapspace version 1, size = 15626236 KiB
no label, UUID=cac3ef7c-b916-4a8c-ac1b-6bff33e93438


# Swap 활성화
[centos@wglee-server ~]$ sudo swapon -v /dev/vg1/wglv_1
swapon /dev/vg1/wglv_1
swapon: /dev/mapper/vg1-wglv_1: found swap signature: version 1, page-size 4, same byte order
swapon: /dev/mapper/vg1-wglv_1: pagesize=4096, swapsize=16001269760, devsize=16001269760

[centos@wglee-server ~]$ cat /proc/swaps
Filename                                Type            Size    Used    Priority
/dev/dm-0                               partition       14577660        0       -3
/swapfile                               file            61436   0       -2


[centos@wglee-server ~]$ free -h
              total        used        free      shared  buff/cache   available
Mem:           991M        104M        288M         56M        597M        651M
Swap:           13G          0B         13G
```





## 참고문서

{% embed url="https://opensource.com/article/18/9/swap-space-linux-systems" %}

{% embed url="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/storage_administration_guide/ch-swapspace" %}
