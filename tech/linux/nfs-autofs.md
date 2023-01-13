---
description: 2022. 6. 4. 19:48
---

# NFS 자동 마운트 autofs 설정하기

## Intro

오늘은 NFS 시스템 자동 마운트 기능인 autofs에 대해 알아보도록 한다.

<figure><img src="https://blog.kakaocdn.net/dn/cEXfiX/btrD19D3w3K/pduxHTaELYJtBOGQKB5O91/img.png" alt=""><figcaption></figcaption></figure>

## NFS?

NFS(Network File System)은 파일 서버인 NFS 서버의 데이터를 원격에 공유하여 클라이언트 서버에서 사용할 수 있도록 하는 파일 시스템이다.\
NFS 서버에서 외부에 공개할 디렉터리 등을 지정하여 export를 하면 NFS 클라이언트는 해당 원격의 파일이 마치 로컬에 있는 것처럼 접\
근하여 수정, 삭제 할 수 있다.\
물론 세부적인 권한은 /etc/exports 파일을 통해 옵션으로 설정해야 한다.



## autofs vs 수동 마운트+/etc/fstab

NFS 클라이언트가 원격 데이터에 접근하기 위해서는 export된 디렉터리를 로컬 파일시스템에 마운트 해야 한다.\
부팅 시에도 mount 유지 하기 위해서는 /etc/fstab에 설정도 해야 한다.\
기존의 이러한 방법이라면 수동으로 mount, umount를 해야 하며, /etc/fstab 에 설정한 내용도 부팅시에만 적용된다는 특징이 있다.

하지만 autofs 를 설정하면 이러한 번거로움을 줄일 수 있다.\
autofs 는 NFS share가 사용될 때만 "자동으로" mount를 하며, 일정시간 사용되지 않으면 자동 umount를 한다.\
자동 마운트할 로컬 경로, 외부 share, 권한 지정 등은 NFS client 에서 map 파일에 설정한다.



## direct map과 indirect map

autofs 의 map 파일 구성 방법에는 두 가지가 있다.

1. direct map : 특정 NFS share를 로컬의 절대경로 mount point에 mount 하는 방식
2. indirect map : NFS 서버의 특정 경로 하위의 여러 디렉터리에 접근해야 할 경우 \*(asterisk) 을 사용하여 mount 할 수 있는 방식

***

## Hands-on

### autofs 패키지 설치

먼저 autofs 패키지를 NFS 클라이언트에 설치하고 재부팅 시에도 올라오도록 enable 한다.

```shell-session
[root@nfs-client ~]# yum install nfs
[root@nfs-client ~]# systemctl enable --now autofs
Created symlink /etc/systemd/system/multi-user.target.wants/autofs.service → /usr/lib/systemd/system/autofs.service.
```

테스트하기에 앞서, nfs-server 에서 share 할 파일들은 다음과 같이 구성되어 있다.

```shell-session
[root@nfs-server /]# tree shares/
shares/
├── documents
│   ├── docs1
│   ├── docs2
│   └── docs3
├── music
│   ├── music1
│   ├── music2
│   └── music3
└── videos
    ├── video1
    ├── video2
    └── video3

3 directories, 9 files
```

exportfs -a 하기

```shell-session
[root@nfs-server /]# cat /etc/exports | grep -v '^#'
/shares         nfsclient(rw,sync,no_root_squash)

[root@nfs-server /]# exportfs -a
```



### (방법1) direct map 설정

NFS client 에서 map 파일을 설정한다.\
NFS server의 /shares/documents share을 NFS client의 절대 경로 /mnt/docs 에 마운트 할 것이다.

```shell-session
[root@nfs-client ~]# cat /etc/auto.master.d/direct.autofs
/mnt    /etc/auto.direct

[root@nfs-client ~]# cat /etc/auto.direct
docs            -rw,sync        nfsserver:/shares/documents
```

autofs 서비스 재시작

```shell-session
[root@nfs-client ~]# systemctl restart autofs
```

이제 /mnt/docs 경로로 이동한다. 이동 후에 mount 명령어로 유저가 이용 시도를 함과 동시에 자동 마운트 된 것을 볼 수 있다.

```shell-session
[root@nfs-client docs\]# ll  
total 0  
-rw-r--r--. 1 root root 0 Jun 5 00:58 docs1  
-rw-r--r--. 1 root root 0 Jun 5 00:58 docs2  
-rw-r--r--. 1 root root 0 Jun 5 00:58 docs3  
-rw-r--r--. 1 root root 0 Jun 5 00:58 docs4

[root@nfs-client mnt\]# mount | grep docs  
nfs-server:/shares/documents on /mnt/docs type nfs4 (rw,relatime,sync,vers=4.2,rsize=1048576,wsize=1048576,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=172.25.250.11,local\_lock=none,addr=172.25.250.12)
```



### (방법2) indirect map 설정

NFS client 에서 map 파일을 설정한다.\
NFS server의 /shares share을 NFS client의 /mnt/nfs 에 마운트 할 것이다.\
NFS server의 /shares 하위에는 3개의 subdirectories 가 있는 상태이다.

로컬의 mount point는 \* 으로 한다.\
그리고 /etc/auto.indirect에서 nfsserver의 source location을 /shares/&로 설정했다.\
&는 /shares/ 하위의 모든 하위 디렉터리를 포함한다는 뜻이다.

```shell-session
[root@nfs-client music]# cat /etc/auto.master.d/indirect.autofs
/mnt/nfs                /etc/auto.indirect
[root@nfs-client music]# cat /etc/auto.indirect
*       -rw,sync        nfsserver:/shares/&
```

autofs 재시작

```shell-session
[root@nfs-client ~]# systemctl restart autofs
```

ls 로 해당 경로를 조회하려고 하면 없는 디렉터리라고 나온다.

```shell-session
[root@nfs-client ~]# cd /mnt/nfs/docs
-bash: cd: /mnt/nfs/docs: No such file or directory
```

cd 로 이동한다.

```shell-session
[root@nfs-client ~]# cd /mnt/nfs/documents
[root@nfs-client documents]# ll
total 0
-rw-r--r--. 1 root root 0 Jun  5 00:58 docs1
-rw-r--r--. 1 root root 0 Jun  5 00:58 docs2
-rw-r--r--. 1 root root 0 Jun  5 00:58 docs3

[root@nfs-client music]# df -h
Filesystem                    Size  Used Avail Use% Mounted on
devtmpfs                      3.4G     0  3.4G   0% /dev
tmpfs                         3.4G     0  3.4G   0% /dev/shm
tmpfs                         3.4G   17M  3.4G   1% /run
tmpfs                         3.4G     0  3.4G   0% /sys/fs/cgroup
/dev/vda1                      10G  1.4G  8.7G  14% /
tmpfs                         683M     0  683M   0% /run/user/0
nfs-server:/shares/documents   10G  1.4G  8.7G  14% /mnt/nfs/documents
nfs-server:/shares/music       10G  1.4G  8.7G  14% /mnt/nfs/music
```

이런식으로 직접 접근해서 사용할 때 자동 mount 되는 것을 볼 수 있다.

```shell-session
[root@nfs-client nfs]# tree /mnt/nfs/
/mnt/nfs/
├── documents
│   ├── docs1
│   ├── docs2
│   └── docs3
├── music
│   ├── music1
│   ├── music2
│   └── music3
└── videos
    ├── video1
    ├── video2
    └── video3

3 directories, 9 files
```



## 참고 문서

{% embed url="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/storage_administration_guide/nfs-autofs#doc-wrapper" %}

