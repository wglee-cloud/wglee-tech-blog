---
description: 2022. 9. 25. 17:13
---

# KDUMP - vmcore 분석 방법

## Intro

**kdump**는 Kernel Crash된 시스템의 장애 원인을 분석하기 위한 core dump를 생성하는 역할을 한다.\
kdump은 커널 패닉이 발생하면 kexec 시스템 콜을 사용해서 본래 부팅하던 커널이 아닌, capture kernel 로 부팅한다.\
capture kernel로 부팅하면 crash 된 커널의 메모리 덤프(ex. vmcore)를 떠서 파일로 남기게 된다.\
사용자는 서버 재부팅 후 vmcore 파일로 시스템 장애 원인을 분석할 수 있다.

vmcore의 커널과 다른 버전의 OS에서 디버깅 하는 법을 테스트 하기 위해 일부러 상이한 버전에서 진행하였다.\
고객사의 vmcore을 받아 분석해야 하는 경우 꼭 같은 버전에서 분석하라는 법이 없기 때문에 알아두면 좋다.



<mark style="color:blue;">**TIP**</mark>\
sosreport는 용량이 크기 때문에 가끔 vmcore 파일이 같이 안 받아질 수 있다.\
고객사로부터 sosreport를 먼저 받은 다음, vmcore가 없으면 vmcore만 다시 요청 하는 것이 순서상 좋다.



## kdump 분석을 위한 패키지 설치

core dump 를 분석하기 위해 crash 유틸리티를 설치한다.

```shell-session
[root@wglee ~]# yum install crash
```

debuginfo-install은 특정 패키지를 디버깅 하기 위한 rpm 설치를 하는 명령어이다. vmcore 파일을 디버깅 하기 위해서는 vmcore 과 동일한 커널 버전의 vmlinux 실행 파일이 필요하다.\
vmcore 파일로 받은 system의 OS 버전을 확인한다.

```shell-session
[root@wglee-vm ~]# crash --osrelease /home/vmcore-test/vmcore
3.10.0-1160.el7.x86_64
```

분석을 하는 나의 시스템은 rocky 8.4 이기 때문에 커널 버전이 일치하지 않는다.

```shell-session
[root@wglee ~]# uname -r
4.18.0-305.3.1.el8_4.x86_64
```

동일한 커널 버전의 kernel-debuginfo-common, kernel-debuginfo 패키지를 설치한다.\
kernel-debuginfo-common 없이 kernel-debuginfo를 먼저 설치하려고 하면 에러가 발생한다.\
kernel-debuginfo 패키지들은 아래 repo 에서 찾을 수 있다.\
[http://debuginfo.centos.org/7/x86\_64/](http://debuginfo.centos.org/7/x86\_64/)

<figure><img src="https://blog.kakaocdn.net/dn/EDlDh/btrMVdTqGz9/1eza3tLKnh4XBhVct3jk2K/img.png" alt=""><figcaption></figcaption></figure>

```shell-session
[root@wglee ~]# wget http://debuginfo.centos.org/7/x86_64/kernel-debuginfo-common-x86_64-3.10.0-1160.el7.x86_64.rpm
[root@wglee ~]# wget http://debuginfo.centos.org/7/x86_64/kernel-debuginfo-3.10.0-1160.el7.x86_64.rpm

[root@wglee ~]#  yum install kernel-debuginfo-common-x86_64-3.10.0-1160.el7.x86_64.rpm
[root@wglee ~]#  yum install kernel-debuginfo-3.10.0-1160.el7.x86_64.rpm

...
# 번외 : kernel-debuginfo-common 설치하지 않으면 에러 발생
[root@ubuntu ~]# yum install kernel-debuginfo-3.10.0-1160.el7.x86_64.rpm
Last metadata expiration check: 0:27:21 ago on Sun 25 Sep 2022 05:24:16 PM UTC.
Error:
 Problem: conflicting requests
  - nothing provides kernel-debuginfo-common-x86_64 = 3.10.0-1160.el7 needed by kernel-debuginfo-3.10.0-1160.el7.x86_64
(try to add '--skip-broken' to skip uninstallable packages or '--nobest' to use not only best candidate packages)
```

이렇게 하면 vmcore와 동일한 커널 버전의 vmlinux 가 생성되었다.

```shell-session
[root@wglee 3.10.0-1160.el7.x86_64]# pwd
/usr/lib/debug/usr/lib/modules/3.10.0-1160.el7.x86_64
[root@wglee 3.10.0-1160.el7.x86_64]# ls
kernel  vdso  vmlinux
```



## vmcore 덤프 파일 분석

crash utility 를 사용하여 vmcore 파일을 분석한다.

```shell-session
[root@wglee ~]# crash /usr/lib/debug/usr/lib/modules/3.10.0-1160.el7.x86_64/vmlinux /home/vmcore-test/vmcore
```

<figure><img src="https://blog.kakaocdn.net/dn/3tfaf/btrMVdsl6qX/qkL7nQcZLMbfPvo7fXZIU1/img.png" alt=""><figcaption></figcaption></figure>

## Crash interface Commands

Crash 명령어로 interactive 환경에 도입하면 몇가지 명령어로 vmcore를 효율적으로 분석할 수 있다.\
그 중 몇가지는 아래와 같다.

**bt**\
명령어로 backtrace 를 확인한다.\
각각의 hash (#)로 시작하는 라인이 crash 직전에 호출된 system call에 해당한다. 프로그램의 실행을 위해 커널에서 동작 중이던 커널의 function call 을 확인할 수 있다.\
exception RIP 문자열로 어떤 함수를 호출 할 때 문제가 발생했는지 파악할 수 있다.

**ps -t**\
프로세스 별로 runtime을 확인 한다.

**ps -p \[PID]**\
특정 프로세스의 정보를 확인 한다.

**files \[PID]**\
해당 프로세스로 open 중인 파일 리스트를 확인 한다.

**rd \[메모리 주소]**\
메모리 번지수에 담긴 내용을 16진수(hexadecimal)로 확인할 수 있다.



## 참고 문서

{% embed url="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/installing-kdumpmanaging-monitoring-and-updating-the-kernel" %}

{% embed url="https://docs.oracle.com/en/operating-systems/oracle-linux/6/admin/ol_ssc_crash.html" %}

{% embed url="https://mapoo.net/os/oslinux/vmcore-analyze/https://www.dedoimedo.com/computers/crash-analyze.html#mozTocId782257" %}

\
tainted? :&#x20;

{% embed url="https://support.hpe.com/hpesc/public/docDisplay?docId=emr_na-c02905321" %}

{% embed url="https://www.kernel.org/doc/html/v5.8/admin-guide/tainted-kernels.html" %}

{% embed url="https://crash-utility.github.io/crash_whitepaper.html" %}
