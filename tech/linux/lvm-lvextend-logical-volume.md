---
description: 2022. 8. 28. 12:39
---

# \[LVM] lvextend 로 logical volume 확장하기

## Intro

LVM의 LV(logical volume)을 이용할 때의 장점 중 하나로, 시스템 중단 없이 파일 시스템을 확장할 수 있다.\
VG(Volume Group)의 리소스로 LV를 유연하게 확장 시킨 다음에 LV에 마운트 된 파일 시스템에 바로 확장된 용량을 적용할 수 있다.



## lvextend examples

```
1vextend -1 50 : lv의 크기를 정확히 50개 의 LE로 조정. 
lvextend -l +50 : 현 lv에 50개의 LE를 추가
lvextend -L 100M : 현 lv의 총 용량을 딱 100M로 resize. 
lvextend -L +100M : lv에 100MiB 를 추가
lvextend -l +50%FREE : vg에 FREE로 남아있는 영역의 50% 만큼 확장한다.
```



## FileSystem 확장

위의 명령어로는 lv 만 확장한 것이다.&#x20;

실제로 확장된 용량을 사용하려면 파일시스템에 변경 사항을 적용해야 한다.&#x20;

파일시스템의 umount 없이 확장된 lv 용량을 적용 가능할 수 있다.  다만 파일 시스템 타입에 따라 명령어가 다르다.&#x20;

### XFS

```shell-session
# xfs_growfs [마운트포인트]
```

다음과 같이 동작을 확인할 수 있다.\
( 사실 xfs\_growfs 를 따로 해 주기보다는, lvextend에 -r 옵션을 줘서 파일 시스템에 바로 반영할 수도 있다. )

```shell-session
[root@server-1-lab ~]# lvextend -L250M /dev/examvg/exam_lv
  Rounding size to boundary between physical extents: 252.00 MiB.
  Size of logical volume examvg/exam_lv changed from 92.00 MiB (23 extents) to 252.00 MiB (63 extents).
  Logical volume examvg/exam_lv successfully resized.
```

lsblk 로 봤을 때 exam\_lv 용량이 252M로 증설 되었다. 하지만 아직 /data 파일시스템의 크기는 변하지 않았다.

```shell-session
[root@server-1-lab ~]# lsblk
NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                  8:0    0    2G  0 disk
|-sda1               8:1    0   96M  0 part
| `-examvg-exam_lv 253:0    0  252M  0 lvm  /data
|-sda2               8:2    0  128M  0 part
|-sda3               8:3    0  100M  0 part
|-sda4               8:4    0    1K  0 part
|-sda5               8:5    0  256M  0 part
| `-testvg-test_lv 253:1    0  200M  0 lvm  /exports/test
|-sda6               8:6    0  200M  0 part
| `-examvg-exam_lv 253:0    0  252M  0 lvm  /data
`-sda7               8:7    0  120M  0 part [SWAP]
vda                252:0    0   30G  0 disk
`-vda1             252:1    0   30G  0 part /

[root@server-1-lab ~]# df -h /data
Filesystem                  Size  Used Avail Use% Mounted on
/dev/mapper/examvg-exam_lv   89M  4.9M   84M   6% /data
```

xfs\_growfs \[ 마운트포인트 ] 명령어를 실행한 후 /data 영역의 용량이 249M로 확장되었다.

```shell-session
[root@server-1-lab ~]# xfs_growfs /data
meta-data=/dev/mapper/examvg-exam_lv isize=512    agcount=4, agsize=5888 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=23552, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=855, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 23552 to 64512

[root@server-1-lab ~]# df -hT /data
Filesystem                 Type  Size  Used Avail Use% Mounted on
/dev/mapper/examvg-exam_lv xfs   249M  5.2M  244M   3% /data
```

### ext4

ext4 파일 시스템인 경우 resize2fs 명령어를 사용한다.

```shell-session
# resize2fs [Logical Volume 경로]
```



