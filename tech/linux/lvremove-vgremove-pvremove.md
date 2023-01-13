---
description: 2022. 8. 27. 19:05
---

# lvm 구축한 것 삭제하기 (lvremove, vgremove, pvremove)

## Intro

disk 파티셔닝을 해서 lvm 으로 사용하고 있는데, 모든 설정을 원복하고 처음부터 다시 해보려고 한다.\
이전에는 lvm 구축(pv, lv, vg 생성) 위주로 해봤다면 이번에는 삭제하는 과정을 테스트 해 본다.\
이전 게시글 : [2021.09.01 - \[✨ Linux\] - LVM(Logical Volume Manager) 의 개념과 설정 방법](lvm-logical-volume-manager.md)

<figure><img src="https://blog.kakaocdn.net/dn/HCvKS/btrKIziXmbO/PkSNVRoGoqPgfJ9rRE08rk/img.png" alt=""><figcaption></figcaption></figure>

## **AS-IS**

지금은 다음과 같이 설정되어 있다.\
/dev/sda 디바이스는 /dev/sda1 \~ /dev/sda3 으로 파티셔닝 되어있고, /dev/sda2, /dev/sda3이 test\_lv라는 logical volume 으로 사용되고 있다.

```shell-session
[root@server-1-lab ~]# fdisk -l /dev/sda

Disk /dev/sda: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 4194304 bytes
Disk label type: dos
Disk identifier: 0x000d0a4b

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1            8192      204799       98304   8e  Linux LVM
/dev/sda2          204800      458751      126976   83  Linux
/dev/sda3          458752      655359       98304   83  Linux
/dev/sda4          655360     4194303     1769472    5  Extended
/dev/sda5          663552     1187839      262144   83  Linux
/dev/sda6         1196032     4194303     1499136   83  Linux

Disk /dev/mapper/testvg-test_lv: 209 MB, 209715200 bytes, 409600 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 4194304 bytes

[root@server-1-lab ~]# lsblk
NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                  8:0    0    2G  0 disk
|-sda1               8:1    0   96M  0 part
| `-examvg-exam_lv 253:0    0   92M  0 lvm  /data
|-sda2               8:2    0  124M  0 part
| `-testvg-test_lv 253:1    0  200M  0 lvm  /exports/test
`-sda3               8:3    0   96M  0 part
  `-testvg-test_lv 253:1    0  200M  0 lvm  /exports/test
vda                252:0    0   30G  0 disk
`-vda1             252:1    0   30G  0 part /

[root@server-1-lab ~]# df -h
Filesystem                  Size  Used Avail Use% Mounted on
devtmpfs                    470M     0  470M   0% /dev
tmpfs                       496M     0  496M   0% /dev/shm
tmpfs                       496M   13M  483M   3% /run
tmpfs                       496M     0  496M   0% /sys/fs/cgroup
/dev/vda1                    30G  2.0G   29G   7% /
192.168.1.21:/               30G  1.3G   29G   5% /root/nfslist
192.168.1.21:/test           30G  1.3G   29G   5% /mnt/test
/dev/mapper/testvg-test_lv  190M  1.6M  175M   1% /exports/test
/dev/mapper/examvg-exam_lv   89M  4.9M   84M   6% /data
tmpfs                       100M     0  100M   0% /run/user/0
```



## **TO-BE**

/dev/sda2, /dev/sda3 파티션을 삭제하고 각각 100MB, 128MB로 다시 생성한다.



## Hands-on

### **umount**

원복할 disk 파티션을 사용하는 test\_lv에 마운트 된 것을 해제한다.\
fstab이 설정된 경우 fstab 도 수정 한다.

```shell-session
[root@server-1-lab ~]# umount /exports/test/
```

### **Logical Volume을 Inactive 설정하기**

test\_lv를 inactive 상태로 변경한다. lvscan 으로 logical volume 의 활성화 여부를 볼 수 있다.

```shell-session
[root@server-1-lab ~]# lvchange -an /dev/testvg/test_lv
[root@server-1-lab ~]# lvscan
  inactive          '/dev/testvg/test_lv' [200.00 MiB] inherit
  ACTIVE            '/dev/examvg/exam_lv' [92.00 MiB] inherit
```

### **Logical Volume 제거**

```shell-session
 [root@server-1-lab ~]# lvremove /dev/testvg/test_lv
  Logical volume "test_lv" successfully removed

  [root@server-1-lab ~]# lsblk
NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                  8:0    0    2G  0 disk
|-sda1               8:1    0   96M  0 part
| `-examvg-exam_lv 253:0    0   92M  0 lvm  /data
|-sda2               8:2    0  124M  0 part
|-sda3               8:3    0   96M  0 part
|-sda4               8:4    0    1K  0 part
|-sda5               8:5    0  256M  0 part
`-sda6               8:6    0  1.4G  0 part
vda                252:0    0   30G  0 disk
`-vda1             252:1    0   30G  0 part /
```

lv를 삭제하고 나서 바로 pv를 삭제하려고 하면 당연히 에러가 난다.\
일단 lv를 삭제하고 쓰지 않고 있기 때문에 PFree값이 모두 PSize와 동일하다.

```shell-session
 [root@server-1-lab ~]# pvs
  PV         VG     Fmt  Attr PSize   PFree
  /dev/sda1  examvg lvm2 a--   92.00m      0
  /dev/sda2  testvg lvm2 a--  120.00m 120.00m
  /dev/sda3  testvg lvm2 a--   90.00m  90.00m

[root@server-1-lab ~]# pvremove /dev/sda3
  PV /dev/sda3 is used by VG testvg so please use vgreduce first.
  (If you are certain you need pvremove, then confirm by using --force twice.)
  /dev/sda3: physical volume label not removed.
```

### **Volume Group 삭제**

vgreduce 명령어를 사용하여 vg에서 pv를 제거할 수 있다.\
이때 PV가 하나밖에 없는 상태에서 마지막 PV를 제거하려고 하면 vg에 대한 metadata를 저장할 공간이 없어서 실패한다.\
vgreduce 명령어는 말 그대로 "reduce" 하는 것이고, 어차피 testvg 자체를 지울 거라서 바로 vgremove를 했다.

```shell-session
[root@server-1-lab ~]# vgreduce testvg /dev/sda2
  Removed "/dev/sda2" from volume group "testvg"

[root@server-1-lab ~]# vgreduce testvg /dev/sda3
  Can't remove final physical volume "/dev/sda3" from volume group "testvg"

[root@server-1-lab ~]# vgremove testvg
  Volume group "testvg" successfully removed

[root@server-1-lab ~]# vgs
  VG     #PV #LV #SN Attr   VSize  VFree
  examvg   1   1   0 wz--n- 92.00m    0
```

### **Pyhical Volume 삭제**

pv 정보를 조회하면 testvg 삭제로 인해 /dev/sda2, /dev/sda3에 vg 정보가 없는 것을 볼 수 있다.

```shell-session
[root@server-1-lab ~]# pvs
  PV         VG     Fmt  Attr PSize   PFree
  /dev/sda1  examvg lvm2 a--   92.00m      0
  /dev/sda2         lvm2 ---  124.00m 124.00m
  /dev/sda3         lvm2 ---   96.00m  96.00m
```

pvremove로 pv를 삭제한다.

```shell-session
[root@server-1-lab ~]# pvremove /dev/sda2
  Labels on physical volume "/dev/sda2" successfully wiped.
[root@server-1-lab ~]# pvremove /dev/sda3
  Labels on physical volume "/dev/sda3" successfully wiped.
[root@server-1-lab ~]# pvs
  PV         VG     Fmt  Attr PSize  PFree
  /dev/sda1  examvg lvm2 a--  92.00m    0
```

### 결과

이렇게 /dev/testvg/test\_lv lvm 구축되어 있던 것을 모두 삭제하였다.

```shell-session
[root@server-1-lab ~]# pvs
  PV         VG     Fmt  Attr PSize  PFree
  /dev/sda1  examvg lvm2 a--  92.00m    0
[root@server-1-lab ~]# vgs
  VG     #PV #LV #SN Attr   VSize  VFree
  examvg   1   1   0 wz--n- 92.00m    0
[root@server-1-lab ~]# lvs
  LV      VG     Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  exam_lv examvg -wi-ao---- 92.00m

 [root@server-1-lab ~]# lsblk
NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                  8:0    0    2G  0 disk
|-sda1               8:1    0   96M  0 part
| `-examvg-exam_lv 253:0    0   92M  0 lvm  /data
|-sda2               8:2    0  124M  0 part
|-sda3               8:3    0   96M  0 part
|-sda4               8:4    0    1K  0 part
|-sda5               8:5    0  256M  0 part
`-sda6               8:6    0  1.4G  0 part
vda                252:0    0   30G  0 disk
`-vda1             252:1    0   30G  0 part /
```



## **참고 문서**

{% embed url="https://tylersguides.com/guides/remove-an-lvm-disk/" %}

