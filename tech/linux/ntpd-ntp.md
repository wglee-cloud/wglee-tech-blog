---
description: written on 2021. 2. 20. 16:27
---

# NTPD로 NTP 설정하기

## Overview

[2021/02/19 - \[Linux\] - NTP](https://greencloud33.tistory.com/5) 에서 NTP에 대한 전반적인 개념을 살펴보았다.

이번에는 ntpd 패키지로 시간 동기화를 하는 방법을 알아보도록 한다.



## chronyd 데몬 사용 중지

ntpd를 사용하기 위해서는 chronyd가 중지되어 있어야 한다.

```
[centos@test ~]$ sudo systemctl stop chronyd
[centos@test ~]$ sudo systemctl disable chronyd
Removed symlink /etc/systemd/system/multi-user.target.wants/chronyd.service.
[centos@test ~]$ sudo systemctl status chronyd
● chronyd.service - NTP client/server
   Loaded: loaded (/usr/lib/systemd/system/chronyd.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:chronyd(8)
           man:chrony.conf(5)
```



## ntp 데몬 설치

```
[centos@test ~]$ sudo yum install ntp
[centos@test ~]$ sudo systemctl enable ntpd
```



## ntp.conf 설정

vi /etc/ntp.conf를 하여 ntp pool 등을 원하는 대로 지정할 수 있다.

세부 설정 방법은 [CHAPTER 19. CONFIGURING NTP USING NTPD](https://access.redhat.com/documentation/en-us/red\_hat\_enterprise\_linux/7/html/system\_administrators\_guide/ch-configuring\_ntp\_using\_ntpd)에서 확인 한다.

{% hint style="info" %}
이때 다수의 서버를 등록하고 싶으면 0, 1, 2 등의 숫자를 붙여 나열할 수 있다. 그러면 해당 서버들에 랜덤으로 접속하여 시간 정보를 가져오게 된다.
{% endhint %}

```
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
server 0.kr.pool.ntp.org
server 1.asia.pool.ntp.org
server 2.asia.pool.ntp.org
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
```

알고 가면 좋은 세부 옵션은 다음과 같다

* burst
  * time offset 통계치의 정확성을 올리기 위해 burst 옵션을 사용할 수 있다.
  * burst 옵션을 사용하면 해당 서버로부터 데이터를 polling(시간 정보 가져오는 것) 할때마다 8개의 패킷을 보낸다. default는 1개의 패킷을 보냄.
  * 단, public ntp 서버에는 해당 옵션을 사용하면 안된다!&#x20;
* iburst
  * ntp 서버와의 동기화 되는 시간을 최소화 하기 위해 해당 옵션을 사용한다.
  * 해당 ntp 서버가 unreachable 상태이면, 8개의 패킷을 보낸다. default는 1개의 패킷을 보냄.
  * 패킷은 2초 간격으로 보내지지만, 첫번째와 두번째 패킷 사이의 시간은 calldelay 명령어로 조정할 수 있다.
* minpoll / maxpoll
  * default poll interval을 해당 옵션으로 수정할 수 있다.
  * 실제로 적용하는 시간은 2^value 값이다.
  * 참고로 default pool interval은 다음과 같다
    * minpoll : 6 (2^6 = 64) -> 64초 마다 polling
    * maxpoll : 10 (2^10 = 1024) -> 1024초 마다 polling
  * 3 to 17의 값을 설정할 수 있고, maxpoll 값이 낮을 수록 정확성 또한 높아진다!
* prefer
  * 좀 더 우선 순위에 둘 서버를 지정한다.



## NTP 데몬 활성화 하여 RTC 업데이트 하기

서버의 system clock은 RTC를 읽어 온다. 따라서 ntpd 데몬을 활성화하여 RTC가 ntp와 항상 동기화 되도록 한다.

```
[centos@test ~]$ sudo yum start ntp
[centos@test ~]$ sudo systemctl enable ntpd
[centos@test ~]$ sudo systemctl status ntpd
● ntpd.service - Network Time Service
   Loaded: loaded (/usr/lib/systemd/system/ntpd.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2021-02-21 05:52:01 UTC; 32s ago
  Process: 1416 ExecStart=/usr/sbin/ntpd -u ntp:ntp $OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 1417 (ntpd)
   CGroup: /system.slice/ntpd.service
           └─1417 /usr/sbin/ntpd -u ntp:ntp -g
```



## 동기화 확인

```
[centos@test ~]$ ntptime
ntp_gettime() returns code 0 (OK)
  time e3dc766f.7de00cec  Sun, Feb 21 2021  6:04:31.491, (.491700707),
  maximum error 1355521 us, estimated error 77 us, TAI offset 0
ntp_adjtime() returns code 0 (OK)
  modes 0x0 (),
  offset -12.033 us, frequency -0.001 ppm, interval 1 s,
  maximum error 1355521 us, estimated error 77 us,
  status 0x2001 (PLL,NANO),
  time constant 6, precision 0.001 us, tolerance 500 ppm,
  
[centos@test ~]$ ntptime | grep status
status 0x2001 (PLL,NANO),
```

동기화가 정상적으로 이뤄졌는지는 `ntptime` 명령어의status 항목으로 확인 한다.

* 동기화 안된 경우 unsync 표시가 나타남
* hardware clock 이 system clock에 동기화 된 경우 : status 0x2001 (PLL,NANO)
* hardware clock이 system clock에 동기화 되지 않은 경우 : status 0x41 (PLL,UNSYNC)

ntpstat 명령어로도 확인할 수 있다.

```
[centos@test ~]$ ntpstat
synchronised to NTP server (ip) at stratum 3
   time correct to within 43 ms
   polling server every 64 s
```



## 상세 확인

```
[centos@test ~]$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*send.mx.cdnetwo 125.185.190.74   2 u   47   64  377    1.294    0.021   7.367
+time2.isu.net.s 209.51.161.238   3 u   52   64  377  353.503   -2.287   7.221
+ntp.hkg10.hk.le 130.133.1.10     2 u  125   64  136  181.077  -33.374   3.723
```

* remote : 동기화 중인 ntp server
  * 동기화 중이면 \* / secondary ntp 서버는 +
* refid : ntp 서버가 바라보는 ip
* st : remote stratum (GPS의 경우 stratum 0)
* t : 시간 받아보는 방식 (uniquest, multicast, broadcast) 등



## 참고자료

* [NTP Host Configuration using chronyd or ntpd](https://help.marklogic.com/Knowledgebase/Article/View/ntp-host-configuration-chrony-ntpd)
* [ntp.conf(5) - Linux man page](https://linux.die.net/man/5/ntp.conf)
* [CHAPTER 19. CONFIGURING NTP USING NTPD](https://access.redhat.com/documentation/en-us/red\_hat\_enterprise\_linux/7/html/system\_administrators\_guide/ch-configuring\_ntp\_using\_ntpd)
* [NTP Server: Free Public Internet Time Servers](https://timetoolsltd.com/information/public-ntp-server/)
* [Other fields in ntpq -p command output](https://www.thegeekdiary.com/what-is-the-refid-in-ntpq-p-output/)
* [Network time protocol (NTP): definition and functionality](https://www.ionos.com/digitalguide/server/know-how/network-time-protocol-ntp/)

