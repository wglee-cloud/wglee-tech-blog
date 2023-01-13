---
description: 2022. 8. 28. 12:39
---

# 로컬 yum 레포지터리 만들기! (reposync, yumdownloader, createrepo)

## Intro

오늘은 createrepo 명령어로 local yum repo를 한번 생성해 보겠다.\
원격 서버에서도 로컬 yum 레포지터리에 http url로 접근할 수 있도록 설정해 본다.



## 로컬 repository 생성

repo로 사용할 디렉터리를 생성한다.

```shell-session
[root@server-1-lab ~]# mkdir /wgleeyumrepo
```

repo 디렉터리에 레포지터리에서 관리할 rpm 파일들을 위치 시킨다.

\
나는 두가지 방법으로 rpm 파일을 다운 받았다.\
1번은 개인 학습용으로 해 본 것으로, 만약 실제로 특정 mirror에서 다수의 패키지를 다운받아서 로컬에서 관리하고자 함이라면\
2번에 해당하는 reposync 방법이 제일 활용도 높을 것이다.\
**(1)** 14-.2.0 버전의 ceph 패키지를 mirror에서 wget으로 다운\
~~(2) `yumdownloader --resolve [패키지명]` 으로 dependency 걸린 패키지들까지 resolve 해서 다운~~\
~~yumdownloader 명령어는 rpm 파일을 설치하지 않고 로컬에 내려받는 역할만 한다.~~\
**(2)** `reposync` 명령어를 사용하여 원격지 mirror의 패키지를 로컬에 다운로드



(1) 번의 방법은 다음과 같다.

```shell-session
[root@server-1-lab wgleeyumrepo]# wget https://mirror.kakao.com/centos/7.9.2009/storage/x86_64/ceph-nautilus/Packages/c/ceph-14.2.0-1.el7.x86_64.rpm
```



(2) 번에 해당하는 방법은 다음과 같다.\
`reposync` 는 yum repo를 원격 디렉터리에 sync 할 수 있는 명령어이다. \
쓰인 옵션의 의미도 알아보자.\
**-g** : --gpgcheck. gpgcheck fail 한 패키지는 삭제한다.\
**-l** : --plugins. yum 플러그인을 활성화 한다.\
**-d** : --delete. 로컬에 있는 패키지 중, 원격 repo에 없는 것이라면 삭제한다.\
**-m** : --downloadcomps. comps.xml 도 다운로드 한다.\
**--repoid** : 질의할 repo의 ID를 지정한다. 나는 wglee-docker.repo 파일로 등록한 wglee-docker-repo로 지정했다.\
**--download-metadata** : 모든 metadata를 다운로드 한다.\
**--download\_path** : 패키지를 다운로드할 디렉터리를 지정한다. 기본값은 현 디렉터리.



reposync 로 받은 패키지 중에서도 특정 버전만 local repo로 관리하려면 awk 등을 이용해서 한번 더 정제하면 된다.

```shell-session
[root@server-1-lab ~]# cat /etc/yum.repos.d/wglee-docker.repo
[wglee-docker-repo]
name=wglee docker repo
baseurl=https://download.docker.com/linux/centos/7/x86_64/stable/
enabled=1
gpgcheck=0
gpgkey=https://download.docker.com/linux/centos/gpg

[root@server-1-lab ~]# reposync -g -l -d -m --repoid=wglee-docker-repo --download-metadata --download_path=/wgleeyumrepo
```



/wgleeyumrepo 디렉터리에 대해 createrepo 명령어를 실행한다.\
createrepo 명령어는 해당 경로에 위치한 rpm 패키지에 대해 metadata를 생성한다.\
이 metadata는 repodata라고 불리며, 패키지 별로 위치 및 의존성 정보 등을 담는다.

```shell-session
[root@server-1-lab wgleeyumrepo]# createrepo .
Spawning worker 0 with 51 pkgs
Workers Finished
Saving Primary metadata
Saving file lists metadata
Saving other metadata
Generating sqlite DBs
Sqlite DBs complete

[root@server-1-lab wgleeyumrepo]# createrepo wglee-docker-repo/
```



## 로컬 yum repo 등록하기

로컬에 레포지터리를 생성했으니, yum 명령어로 패키지를 관리 할 수 있도록 yum repo 등록을 한다.\
baseurl에 <mark style="background-color:blue;">`file:// + 레포 디렉터리 절대경로 (/wgleeyumrepo)`</mark> 로 설정한다.

```shell-session
[root@server-1-lab ~]# cat /etc/yum.repos.d/wglee-local.repo
[wgleeyumrepo]
name=wglee local repo
baseurl=file:///wgleeyumrepo
enabled=1
gpgcheck=1
gpgkey=https://mirror.kakao.com/centos/RPM-GPG-KEY-CentOS-7
```

yum repolist로 wgleeyumrepo가 조회되며, 패키지 수는 220개로 확인 된다.

```shell-session
[root@server-1-lab ~]# yum repolist | grep wglee
Failed to set locale, defaulting to C
wgleeyumrepo                 wglee local repo                               220
```

이제 wgleeyumrepo를 사용하여 ceph 패키지를 조회할 수 있다.

<figure><img src="https://blog.kakaocdn.net/dn/cpUAX8/btrKTTPFoB0/iLIoGm4KAtgY2OPkTfw7Y0/img.png" alt=""><figcaption></figcaption></figure>

## 원격지에서 사용 가능하도록 httpd 등록하기

로컬 repo를 생성한 서버에 접근 가능한 원격지 서버에서 wgleeyumrepo를 http로 사용할 수 있도록 설정해본다.\
나는 VirtualHost를 등록해서 포트 8080으로 wgleeyumrepo 를 사용하도록 했다.\
`Require all granted` 가 있어야 원격지에서 /repodata/repomd.xml에 접근할 수 있다.\
httpd.conf 수정 후 httpd 재시작을 한다.

```xml
<VirtualHost 0.0.0.0:8080>
    DocumentRoot /wgleeyumrepo
  <Directory "/wgleeyumrepo">
    Require all granted
  </Directory>
</VirtualHost>
```



## 원격지 서버에서 yum repo 등록

앞서 했던 방법과 동일하게 yum repo를 등록한다.\
다만 baseurl 은 <mark style="background-color:blue;">`http://로컬 repo 있는 서버 주소:포트`</mark>가 된다.

```shell-session
[root@master1 ~]# cat /etc/yum.repos.d/wgleeyumrepo.repo

[wgleeyumrepo]
name=wgleeyumrepo
baseurl=http://server1:8080
enabled=1
gpgcheck=0
```

이제 원격지인 master1 서버에서도 ceph 패키지가 조회된다.

```shell-session
[root@master1 ~]# yum info ceph
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
Available Packages
Name        : ceph
Arch        : x86_64
Epoch       : 2
Version     : 14.2.0
Release     : 1.el7
Size        : 2.5 k
Repo        : wgleeyumrepo
Summary     : User space components of the Ceph file system
URL         : http://ceph.com/
License     : LGPL-2.1 and CC-BY-SA-3.0 and GPL-2.0 and BSL-1.0 and BSD-3-Clause and MIT
Description : Ceph is a massively scalable, open-source, distributed storage system that runs
            : on commodity hardware and delivers object, block and file system storage.

[root@master1 ~]# yum list | grep wglee
Failed to set locale, defaulting to C
ceph.x86_64                            2:14.2.0-1.el7            wgleeyumrepo
containerd.io.x86_64                   1.6.8-3.1.el7             wgleeyumrepo
docker-ce.x86_64                       3:20.10.18-3.el7          wgleeyumrepo
docker-ce-cli.x86_64                   1:20.10.18-3.el7          wgleeyumrepo
docker-ce-rootless-extras.x86_64       20.10.18-3.el7            wgleeyumrepo
docker-ce-selinux.noarch               17.03.3.ce-1.el7          wgleeyumrepo
docker-compose-plugin.x86_64           2.10.2-3.el7              wgleeyumrepo
docker-scan-plugin.x86_64              0.17.0-3.el7              wgleeyumrepo
```



