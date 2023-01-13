---
description: Written on 2021. 8. 13. 16:06
---

# Systemd로 서비스 관리하기

## Intro

일하면서 아무래도 가장 많이 쓰는 명령어는 systemd 관련 커맨드가 아닐까 싶다.

Systemd가 어떤식으로 동작하는지, 동작중인 서비스 리스트는 어떻게 확인하는지 등을 알아보겠다!



Systemd는 Linux OS 시스템을 제어하기 위한 서비스 매니저이다.&#x20;

SysV init 과 호환 되지만, 더 나은 기능들을 제공한다. 실제로도 RHEL7 부터는 systemd 가 init 을 대체하여 쓰인다.



## **Systemd features**

* 시스템 부팅 때 systemd 서비스를 병렬로 시작시킴
* on-demand activation of daemons&#x20;
* dependency-based service control logic&#x20;



## **Systemd Unit Files Location**

Systemd 은 용도에 따라 다양한 unit으로 시스템을 관리한다.

그중에서 몇가지만 알아보도록 한다.

1. Service : 흔히 생각하는 그 시스템 서비스는 모두 Service 유닛에 해당한다.
2. Target : Target은 Service 유닛의 집합이다.&#x20;
3. Swap : Swap 디바이스 혹은 Swap 파일

unit configration 파일은 용도에 따라 다음과 같은 경로에 위치한다.

**/usr/lib/systemd/system/**

* RPM pakage로 설치하면 그에 대한 systemd 파일이 이 경로에 생성된다.&#x20;
* 예를 들어서 httpd 패키지를 설치하면 이 경로에 httpd.service 파일이 생성됨.

**/run/systemd/system/**

* 런타임에 생성된 Systemd unit files.

**/etc/systemd/system**

* systemctl enable로 생성된 파일들이 저장된다.
*   예시로 httpd를 부팅시에 자동으로 올라오도록 enable하면 해당 디렉터리 하위의 multi-user.target.wants에 httpd 서비스 파일에 대한 심볼릭 링크가 생성된다. 참고로 이때 multi-user.target은 run level3(텍스트 모드)로 서버를 사용할 때 default로 설정되는 Systemd Target이다.&#x20;

    ```shell-session
    [centos@wglee system]$ systemctl enable httpd
    Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.

    [centos@wglee-linux multi-user.target.wants]$ pwd
    /etc/systemd/system/multi-user.target.wants

    [centos@wglee-linux multi-user.target.wants]$ ls
    total 8
    drwxr-xr-x.  2 root root 4096 Aug 13 06:20 .
    drwxr-xr-x. 13 root root 4096 Oct 30  2020 ..
    lrwxrwxrwx.  1 root root   38 Oct 30  2020 auditd.service -> /usr/lib/systemd/system/auditd.service
    lrwxrwxrwx.  1 root root   39 Oct 30  2020 chronyd.service -> /usr/lib/systemd/system/chronyd.service
    lrwxrwxrwx.  1 root root   37 Oct 30  2020 crond.service -> /usr/lib/systemd/system/crond.service
    lrwxrwxrwx.  1 root root   37 Aug 13 06:20 httpd.service -> /usr/lib/systemd/system/httpd.service
    lrwxrwxrwx.  1 root root   42 Oct 30  2020 irqbalance.service -> /usr/lib/systemd/system/irqbalance.service
    lrwxrwxrwx.  1 root root   37 Oct 30  2020 kdump.service -> /usr/lib/systemd/system/kdump.service
    lrwxrwxrwx.  1 root root   41 Oct 30  2020 nfs-client.target -> /usr/lib/systemd/system/nfs-client.target
    lrwxrwxrwx.  1 root root   39 Oct 30  2020 postfix.service -> /usr/lib/systemd/system/postfix.service
    lrwxrwxrwx.  1 root root   40 Oct 30  2020 remote-fs.target -> /usr/lib/systemd/system/remote-fs.target
    lrwxrwxrwx.  1 root root   46 Oct 30  2020 rhel-configure.service -> /usr/lib/systemd/system/rhel-configure.service
    lrwxrwxrwx.  1 root root   39 Oct 30  2020 rpcbind.service -> /usr/lib/systemd/system/rpcbind.service
    lrwxrwxrwx.  1 root root   39 Oct 30  2020 rsyslog.service -> /usr/lib/systemd/system/rsyslog.service
    lrwxrwxrwx.  1 root root   36 Oct 30  2020 sshd.service -> /usr/lib/systemd/system/sshd.service
    lrwxrwxrwx.  1 root root   37 Oct 30  2020 tuned.service -> /usr/lib/systemd/system/tuned.service
    ```

&#x20;

## **서비스 리스트 출력하기**

현재 시스템에 올라간 서비스 리스트는 다음과 같이 확인한다. 나중에 오픈스택 특정 프로젝트의 데몬이 모두 잘 동작하고 있는지 확인할때 status 로 각각 확인 하는게 아니라 이 명령어로 한번에 보는 것도 좋은 방법일 것 같다.

```shell
[centos@wglee-linux system]$ systemctl list-units --type service
  UNIT                               LOAD   ACTIVE SUB     DESCRIPTION
  auditd.service                     loaded active running Security Auditing Service
  chronyd.service                    loaded active running NTP client/server
  cloud-config.service               loaded active exited  Apply the settings specified in cloud-config
  cloud-final.service                loaded failed failed  Execute cloud user/final scripts
```

만약 서비스의 enable 여부를 확인하고 싶으면 다음 명령어를 쓴다.

```shell-session
[centos@wglee-linux system]$ systemctl list-unit-files --type service
UNIT FILE                                     STATE
arp-ethers.service                            disabled
auditd.service                                enabled
auth-rpcgss-module.service                    static
autovt@.service                               enabled
blk-availability.service                      disabled
brandbot.service                              static
chrony-dnssrv@.service                        static
chrony-wait.service                           disabled
chronyd.service                               enabled
cloud-config.service                          enabled​
```

&#x20;
