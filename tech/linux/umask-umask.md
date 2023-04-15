---
description: 2022. 9. 25. 16:57
---

# UMASK 개념과 UMASK 원리를 활용한 협업 디렉터리 구성하기

## Intro

파일이나 디렉터리가 생성됨과 동시에 지정된 권한을 따르도록 하기 위해 umask를 사용한다.



## UMASK?

umask 가 적용되는 원리는 아래와 같다.\
linux 에서 디렉터리는 기본권한이 777이고, 파일은 666이다.\
새로운 파일/디렉터리를 생성할 때 기본 권한에서 umask 값을 빼면 최종 권한이 결정된다.\
예를 들어서 umask가 022인 계정이 파일을 생성하면 파일의 권한은 666-022=644가 된다.

<figure><img src="https://blog.kakaocdn.net/dn/ckszDS/btrMVG8UeWM/ZWnmYJeCep8EOcWkkBQFck/img.png" alt=""><figcaption></figcaption></figure>

/etc/bashrc와 /etc/profile 에는 각각 다음과 같이 umask 를 설정하는 코드가 있다.\
UID가 199 보다 크고, 그룹명과 계정명이 같으면 umask 002로 설정하고 그 외에는 022로 설정한다.\
CentOS 7.6 에서 system 계정으로 예약된 uid는 0\~199 이다.\
일반 계정의 umask 는 002, system 계정의 umask 는 022 가 된다.

* **umask 002 (일반계정)**\
  \- Directory 기본 permission은 777-002 = 775 (rwx rwx r-x)\
  \- File 기본 permission은 666-002 = 664 (rw- rw- r--)
* **umask 022 (시스템 계정)**\
  \- Directory 기본 permission은 777-022 = 755 (rwx r-x r-x)\
  \- File 기본 permission은 666-022 = 644 (r-x r-- r–)

non-login shell 에 대한 umask 는 /etc/bashrc 에서 설정되고, login shell 에 대한 umask 는 /etc/profile 에서 설정된다.

#### <mark style="color:red;">주의!</mark>

default umask 의 변경으로 인해 시스템의 정상적인 동작에 문제가 발생할 가능성이 있다.\
umask를 너무 제한적으로 설정하면 응용 프로그램이 제대로 작동하지 않을 수 있다.\
너무 많은 권한을 부여할 경우 보안 위험이 있을 수 있기 때문에 되도록 기본 설정을 변경하지 않는 것이 좋다.



## umask 적용 시나리오

해당 소유자 그룹에 있는 계정들이 협업을 위해 /tmp/project1 하위의 파일을 공통으로 read/write할 수 있어야 하는 상황이라고 가정한다.\
특정 디렉터리 하위에 생성되는 파일들이 디렉터리의 소유자 그룹을 계승해서 생성되도록 setgid 설정을 한다.

또한 그룹에 속한 각 계정의 umask가 0002여야 한다. 이유는 생성하는 파일/디렉터리에 group write 권한이 기본으로 있어야 동일 그룹에 속한 다른 사용자도 파일을 편집할 수 있기 때문이다.&#x20;

```shell-session
[wglee01@wglee-vm ~]# id
uid=1009(wglee01) gid=1009(wglee01) groups=1009(wglee01)
[wglee01@wglee-vm root]$ umask
0002

[root@wglee-vm ~]# groupadd project1
[root@wglee-vm ~]# usermod -aG project1 wglee01
[root@wglee-vm ~]# usermod -aG project1 wglee02
[root@wglee-vm ~]# usermod -aG project1 wglee03

[root@wglee-vm ~]# id wglee01
uid=1009(wglee01) gid=1009(wglee01) groups=1009(wglee01),1012(project1)
[root@wglee-vm ~]# id wglee02
uid=1010(wglee02) gid=1010(wglee02) groups=1010(wglee02),1012(project1)
[root@wglee-vm ~]# id wglee03
uid=1011(wglee03) gid=1011(wglee03) groups=1011(wglee03),1012(project1)


# /tmp/project1 생성 및 소유 그룹 지정. 해당 디렉터리에 group write 권한 부여
[root@wglee-vm ~]# mkdir /tmp/project1
[root@wglee-vm ~]# chgrp project1 /tmp/project1/
[root@wglee-vm ~]# chmod g+s /tmp/project1/
[root@wglee-vm ~]# ls -al /tmp/project1/
total 0
drwxr-sr-x.  2 root project1   6 Sep 25 16:14 .
drwxr-xr-x. 16 root root     226 Sep 25 16:14 ..

[root@wglee-vm ~]# chmod 2770 /tmp/project1/
[root@wglee-vm ~]# echo "File01" > /tmp/project1/file01


# wglee01 계정으로 /tmp/project1에 파일 생성
[root@wglee-vm ~]# su wglee01
[wglee01@wglee-vm project1]$ touch file02
[wglee01@wglee-vm project1]$ ls
file01  file02
[wglee01@wglee-vm project1]$ ls -al
total 4
drwxrws---. 2 root    project1  34 Sep 25 16:20 .
drwxrwxrwt. 9 root    root     188 Sep 25 16:19 ..
-rw-r--r--. 1 root    project1   7 Sep 25 16:16 file01
-rw-rw-r--. 1 wglee01 project1   0 Sep 25 16:20 file02


# wglee02 계정으로 wglee01 이 생성한 file02 편집
[root@wglee-vm ~]# su wglee02
[wglee02@wglee-vm root]$ cd /tmp/project1/
[wglee02@wglee-vm project1]$ echo "edited by wglee02" > file02
[wglee02@wglee-vm project1]$ cat file02
edited by wglee02


# project1 그룹에 속하지 않은 wglee04는 권한이 없기 때문에 /tmp/project1의 파일에 접근할 수 없다
[wglee08@wglee-vm root]$ cd /tmp/project1/
bash: cd: /tmp/project1/: Permission denied
[wglee08@wglee-vm root]$ cat /tmp/project1/file01
cat: /tmp/project1/file01: Permission denied
```



## 참고 문서

{% embed url="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_basic_system_settings/assembly_managing-the-umask_configuring-basic-system-settings" %}

{% embed url="https://access.redhat.com/solutions/107683" %}

