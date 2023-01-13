---
description: written at 2021. 2. 19. 11:15
---

# NTP

## Overview

NTP 서버의 용도와 이를 이용한 서버 시간 동기화 방법에 대해 알아보도록 한다.

<figure><img src="../../.gitbook/assets/다운로드.png" alt=""><figcaption></figcaption></figure>

##

## NTP&#x20;

NTP (Network Time Protocol)를 사용하면 가상서버의 시간을 원하는 표준시로 동기화 할 수 있다.

서버 시간을 동기화 하는 패키지로는 chrony와 ntp가 있으며, chrony는 ntp 패키지의 단점을 보완하여 만들어진 것이다. 단, ntp 패키지는 centos8 부터는 지원되지 않는다.





## NTP Pool

NTP Pool Project는특정 NTP 서버에 대한 부하가 높아져 서비스가 불안정하게 되는 문제를 해결하기 위해 2003년에 시작되었다.

NTP Pool 정보는 [NTP Pool Project](https://www.ntppool.org/en/)에서 확인 할 수 있다.

기본 도메인은 pool.ntp.org이며, 이에 대한 sub-domain은 다음 3가지 중 하나를 따른다.

* Continent
* Economy
* Vendor

이 중 Continet와 Economy는 지리적으로 나누어진 타임존에 기반하여 설정된다.&#x20;

이렇게 나눠진 이유는 사용자의 위치에서 최대한 가까운 NTP 서버를 제공하기 위해서 이다.

```
0.asia.pool.ntp.org
1.asia.pool.ntp.org
2.asia.pool.ntp.org
3.asia.pool.ntp.org
```



Vendor에 따라 나눠진 NTP Pool의 경우 특정 제조사들의 이름을 따서 sub-domain이 지정된다.

해당 제조사들은 할당 받은 pool를 사용해 product를 NTP server와시간 동기화를 할 수 있다.

실제로 centos 가상서버에는 다음과 같은 pool이 default로 설정 되어 있다.&#x20;

```
0.centos.pool.ntp.org
1.centos.pool.ntp.org
2.centos.pool.ntp.org
3.centos.pool.ntp.org
```



ubuntu 가상서버에는 다음과 같이 설정되어 있다.

```
# Use servers from the NTP Pool Project. Approved by Ubuntu Technical Board
# on 2011-02-08 (LP: #104525). See http://www.pool.ntp.org/join.html for
# more information.
pool 0.ubuntu.pool.ntp.org iburst
pool 1.ubuntu.pool.ntp.org iburst
pool 2.ubuntu.pool.ntp.org iburst
pool 3.ubuntu.pool.ntp.org iburst
```





## Time Synchronization

그러면 이런 시간 동기화는 왜 필요한 걸까?

networking의 측면을 예시로 들어보자!

패킷 통신에 있어서 정확한 time stamp와 로그가 필수적이고 이에 대해 정확한 시간이 맞춰져 있어야 정상적인 서비스 분석과 시스템 간의 서비스 동기화가 가능하다.&#x20;



이를 위해서 리눅스 시스템에서는 user space에서 chrony, ntpd 등의 데몬을 동작시켜 NTP를 구현한다.

이러한 user space 데몬은 커널 상에도 동작하는 Software clock를 업데이트하게 된다.



기본적으로 리눅스에는 Software clock과 Hardware clock이 있다.

**Hardware Clock** : Real Time Clock이라고도 불린다. rtc, hwclock 명령어를 통해 확인 가능하다.

**System Clock** : Software적인 시계. 보통 Time Stamp Counter(TSC)를 이용한다. (TSC는 cpu 사이클을 수를 세는 cpu 레지스터)

시스템이 시작되면 system clock이 real time clock 값을 읽어 온다.&#x20;

하지만 RTC 시간은 매월 5분 정도 실제 시간과 격차가 생기게 된다. (하드웨어 장비가 온도 변화에 반응해서 시간이 틀어짐)

때문에 system clock이 지속적으로 외부의 실제 시간과 동기화가 되도록 하기 위해 NTP가 필요한 것이다.

system clock이 ntpd에 의해 동기화 되면, 커널은 RTC를 11분 간격으로 지속적으로 업데이트한다.

```shell
[centos@test ~]$ timedatectl
      Local time: Fri 2021-02-19 18:07:35 KST
  Universal time: Fri 2021-02-19 09:07:35 UTC
        RTC time: Fri 2021-02-19 09:07:35
       Time zone: Asia/Seoul (KST, +0900)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
      DST active: n/a
```



timedatectl 명령어로 local time, UTC, RTC, NTP 동기화 여부를 알 수 있다.

위의 결과에서 RTC Time이 UTC로 설정되어 있는걸 확인 할 수 있다.

RTC는 /etc/adjtime 파일을 따르며, 열어보면 실제로 UTC로 설정되어 있다.

```shell
[centos@test ~]$ cat /etc/adjtime
0.0 0 0.0
0
UTC
```





## NTP가 사용하는 통신 프로토콜

ntp는 udp를 사용하여 통신하여 시간을 받아오게 된다. 사용하는 포트는 123이다. 해커의 ip 스푸핑 공격 위험에 노출될 위험이 있어 이를 보완하기 위한 기능들 또한 존재한다.

```shell
[centos@test ~]$ netstat -ao
udp        0      0 test:ntp                0.0.0.0:*                           off (0.00/0/0)
udp        0      0 localhost:ntp           0.0.0.0:*                           off (0.00/0/0)
udp        0      0 0.0.0.0:ntp             0.0.0.0:*                           off (0.00/0/0)

[centos@test ~]$ netstat -ano
udp        0      0 192.168.1.31:123        0.0.0.0:*                           off (0.00/0/0)
udp        0      0 127.0.0.1:123           0.0.0.0:*                           off (0.00/0/0)
udp        0      0 0.0.0.0:123             0.0.0.0:*                           off (0.00/0/0)
```





## 가상머신의 시간 동기화

일반적인 서버의 시간 설정은 위와 같은 원리를 따르지만, 대상이 VM이라면 좀 더 고려해야 하는 사항들이 있다.

Virtual machine은 RTC에 접근할 수 없고, VM은 host 서버의 작업량의 영향을 받기 때문에 virtual clock또한 안정적이지 못하다. 때문에 host 에서 사용하는 가상화 app(ex. KVM)에서 제공하는 반가상화된 clock을 사용해야 한다.&#x20;

KVM의 경우 default clock source는 kvm-clock을 사용한다.

이점에 대해서는 가상화 공부하면서 좀더 봐야겠다!





## 참고자료

* [22.1. Introduction to NTP](https://access.redhat.com/documentation/en-us/red\_hat\_enterprise\_linux/6/html/deployment\_guide/ch-configuring\_ntp\_using\_ntpd#s1-Introduction\_to\_NTP)
* [2.2.2. NETWORK TIME PROTOCOL SETUP](https://access.redhat.com/documentation/en-us/red\_hat\_enterprise\_linux/6/html/deployment\_guide/sect-date\_and\_time\_configuration-command\_line\_configuration-network\_time\_protocol)
* [Taking a closer look at the NTP pool using DNS data](https://blog.apnic.net/2020/07/24/taking-a-closer-look-at-the-ntp-pool-using-dns-data/)
* [The NTP Pool for vendors](https://www.ntppool.org/en/vendors.html)[22.7. MANAGING THE TIME ON VIRTUAL MACHINES](https://access.redhat.com/documentation/en-us/red\_hat\_enterprise\_linux/6/html/deployment\_guide/s1-managing\_the\_time\_on\_virtual\_machines)
* [CHAPTER 14. KVM GUEST TIMING MANAGEMENT](https://access.redhat.com/documentation/en-us/red\_hat\_enterprise\_linux/6/html/virtualization\_host\_configuration\_and\_guest\_installation\_guide/chap-virtualization\_host\_configuration\_and\_guest\_installation\_guide-kvm\_guest\_timing\_management)







