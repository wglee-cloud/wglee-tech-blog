# stratis 방식으로 파일 시스템 구현해 보기

## Intro

최근에 stratis 라는 스토리지 구성 방식이 있다는 것을 처음 알게 되었다.\
stratis는 현 시점(2022.05)에서 Tech preview 단계에 있는 feature로, 아직 안정화 되지는 않았다고 한다.\
아직 실서비스에 적용하는 것은 위험성이 있는 기능이지만 장단점을 정리하고 한번 실습해 보고자 한다.



## Stratis File System?

Stratis는 새로운 로컬 스토리지 관리 솔루션으로, pool 이라는 개념을 이용하여 스토리지를 구성한다.\
기존에 알던 xfs, ext4 같은 경우는 물리 디스크 디바이스를 바로 파일 시스템으로 구성했었다.\
하지만 Stratis는 pool 이라는 개념을 사용하여 사용자가 요청하는 파일 시스템을 pool에서 thin provisoning한다.\
즉, 파일 시스템을 10GB로 생성한다고 해서 실제로 10GB할당 되는 것이 아니다.\
실제로 사용 중인 데이터가 3GB라면 pool 에서는 3GB만 해당 파일 시스템에 할당한다. 실사용량에 따라 가변적인 할당이 가능해서 리소스의 낭비를 최소화 하여 효율적으로 사용할 수 있다.

<figure><img src="https://blog.kakaocdn.net/dn/b0had2/btrCKX06JOk/tpyIvppXR9k1k0NqKCwZRK/img.png" alt=""><figcaption></figcaption></figure>

위의 사진처럼 디스크 디바이스를 pool 단위로 맵핑하고, 파일 시스템은 pool 에서 프로비저닝 된다.



## Hands-on

지금 내 가상서버의 disk는 다음과 같다.

```shell-session
[root@wglee-server ~]# fdisk -l
Disk /dev/vda: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xccddae53

Device     Boot Start      End  Sectors Size Id Type
/dev/vda1  *     2048 20971486 20969439  10G 83 Linux


Disk /dev/vdb: 28 GiB, 30000000000 bytes, 58593750 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/vdc: 2.8 GiB, 3000000512 bytes, 5859376 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/vdd: 2.8 GiB, 3000000512 bytes, 5859376 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```



### 패키지 설치

```shell-session
[root@wglee-server ~]# yum install stratis-cli stratisd
[root@wglee-server ~]# systemctl enable --now stratisd
```



### Pool 생성하기

wglee-pool 이라는 이름의 pool을 생성한다. 블록 디바이스 /dev/vdb 를 사용한다.

```shell-session
[root@wglee-server ~]# stratis pool create wglee-pool /dev/vdb
```

pool 리스트는 다음과 같이 확인할 수 있다.

```shell-session
[root@wglee-server ~]# stratis pool list
Name                            Total Physical   Properties                                   UUID
wglee-pool   27.94 GiB / 37.64 MiB / 27.90 GiB      ~Ca,~Cr   e00efad7-b7b0-4381-aaa0-0a2dad298243
```

만약 블록 디바이스를 해당 pool에 추가하고 싶으면 add-data 명령어를 사용한다.

```shell-session
[root@wglee-server ~]# stratis pool add-data wglee-pool /dev/vdc
[root@wglee-server ~]# stratis pool add-data wglee-pool /dev/vdd

[root@wglee-server ~]# stratis pool list
Name                            Total Physical   Properties                                   UUID
wglee-pool   33.53 GiB / 47.24 MiB / 33.48 GiB      ~Ca,~Cr   e00efad7-b7b0-4381-aaa0-0a2dad298243
```

pool에 연결된 블록 디바이스 정보는 다음과 같이 확인할 수 있다.

```shell-session
[root@wglee-server data]# stratis blockdev
Pool Name    Device Node   Physical Size   Tier
wglee-pool   /dev/vdb          27.94 GiB   Data
wglee-pool   /dev/vdc           2.79 GiB   Data
wglee-pool   /dev/vdd           2.79 GiB   Data
```



### Stratis File System 생성하기

이제 wglee-pool 에서 파일 시스템을 생성해 본다.\
생성되는 파일 시스템은 XFS 타입이다.\
stratis filesystem list 명령어로 stratis로 생성한 파일시스템에 대한 디바이스 경로 및 UUID를 확인할 수 있다.

```shell-session
[root@wglee-server ~]# stratis filesystem create wglee-pool filesystem1

[root@wglee-server ~]# stratis filesystem list
Pool Name    Name          Used      Created             Device                                UUID
wglee-pool   filesystem1   546 MiB   May 22 2022 12:27   /dev/stratis/wglee-pool/filesystem1   238f0bac-25e1-45ea-a9a9-2f303266e0a0
```

생성한 파일 시스템을 /data 디렉터리에 마운트 한다.

```shell-session
[root@wglee-server ~]# mount /dev/stratis/wglee-pool/filesystem1 /data
```

이제 마운트까지 했으니 df 했을 때 나타날 것이다.

```shell-session
[root@wglee-server ~]# df -h
Filesystem                                                                                       Size  Used Avail Use% Mounted on
devtmpfs                                                                                         3.4G     0  3.4G   0% /dev
tmpfs                                                                                            3.4G     0  3.4G   0% /dev/shm
tmpfs                                                                                            3.4G   17M  3.4G   1% /run
tmpfs                                                                                            3.4G     0  3.4G   0% /sys/fs/cgroup
/dev/vda1                                                                                         10G  1.4G  8.6G  14% /
tmpfs                                                                                            683M     0  683M   0% /run/user/0
tmpfs                                                                                            1.0M     0  1.0M   0% /run/stratisd/keyfiles
/dev/mapper/stratis-1-e00efad7b7b04381aaa00a2dad298243-thin-fs-238f0bac25e145eaa9a92f303266e0a0  1.0T  7.2G 1017G   1% /data
```

위에서 특이한 점은 /data 에 마운트 된 stratis 파일 시스템 size가 1.0T로 나타난다는 것이다.\
이건 물리 메모리가 실제로 얼마나 할당 되었지와 관계 없이 일괄적으로 나타나는 수치이다.\
파일 시스템이 실제로 할당 받은 디스크 사이즈를 보기 위해서는 "stratis pool list" 명령어를 사용해야 한다.



### Thin Provisioning 확인하기

파일 시스템이 Thin 하게 저장공간을 할당 받는 것을 확인해 보자!\
먼저 처음에 생성한 상태에서 filesystem1의 USED 는 546MiB이다.\
이 파일시스템이 마운트 된 /data 디렉터리에 1GB 짜리 파일을 생성했다.\
그 후에 다시 확인했을 때 filesystem1의 USED size가 약 1.6GIB로 커진 것을 볼 수 있다.

```shell-session
[root@wglee-server data]# stratis filesystem list  
Pool Name Name Used Created Device UUID  
wglee-pool filesystem1 2.53 GiB May 22 2022 12:27 /dev/stratis/wglee-pool/filesystem1 238f0bac-25e1-45ea-a9a9-2f303266e0a0

[root@wglee-server data]# stratis filesystem list
Pool Name    Name          Used       Created             Device                                UUID
wglee-pool   filesystem1   1.63 GiB   May 22 2022 13:39   /dev/stratis/wglee-pool/filesystem1   a7a3c9be-b337-47fd-aa5b-54253c32f5ee
```

_**궁금한 점은,, 마운트된 디렉터리에서 파일을 지웠을 때 그만큼 filesystem의 실제 used 크기도 줄어들어야 할 것 같은데 변하지가 않는다. 왤까...?**_



### 파일 시스템 삭제

filesystem 삭제하기 위해서는 umount를 하고, destory 하면 된다.

```shell-session
[root@wglee-server /]# umount /data
[root@wglee-server /]# stratis filesystem destroy wglee-pool filesystem1
```



## 참고 문서

{% embed url="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_file_systems/setting-up-stratis-file-systems_managing-file-systems" %}

