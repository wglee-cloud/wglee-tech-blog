---
description: Written on 2021. 3. 2. 01:22
---

# iptables 다뤄보기

## Overview

linux 시스템에서 network traffic을 관리하기 위한 방법으로 iptables과 firewalld가 있다.

이 둘의 차이에 대해 알아보고, iptables로 패킷 관리하는 방법에 대해 더 자세히 보자!



## firewalld vs. iptables

1. iptables rule은 입력하는 순간 바로 적용되며, 별도의 데몬을 재시작 할 필요 없다.
2. /etc/sysconfig/iptables-config에 설정파일 존재
3. iptables의 경우 변경이 있을 때마다 기존의 rule들을 flush한 다음 /etc/sysconfig/iptables에서 변경사항이 반영된 새로운 rule들을 읽어온다.
4. 반면, firewalld는 새로운 rule을 추가하기만 하기 때문에 시스템 동작 중에 connection이 끊어질 위험이 없다.&#x20;

&#x20;

## iptables 이모저모

### Line number 출력

rule을 삭제할 때 line number를 명확하게 출력하면 삭제할 규칙의 위치를 확인하기 편하다.&#x20;

```csharp
[centos@test ~]$ sudo iptables -nvL --line-numbers
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1      555 48654 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
2        0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0
3        0     0 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0
4        6   340 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            state NEW tcp dpt:22
5       35  1424 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        1    40 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited

Chain OUTPUT (policy ACCEPT 384 packets, 70487 bytes)
num   pkts bytes target     prot opt in     out     source               destination
```



### iptables rule의 순서 고려하기

iptables rule은 순서(order)의 영향을 받는다. iptables의 rule은 위에서 아래로 반영이 된다.\
예를 들어 drop all 규칙이 중간에 있으면 그 기점으로 모든 패킷이 drop 되기 때문에 그 아래에 있는 rule들은 적용되지 않는다.

순서를 고치기 위해서는 기존의 rule을 파일로 export 해서 수정한 다음 다시  restore 해서 적용한다. &#x20;

```csharp
[centos@test ~]$ sudo iptables-save > ~/iptables.txt
[centos@test ~]$ ls
iptables.txt
[centos@test ~]$ sudo iptables-restore < ~/iptables.txt
```



### INPUT Chain&#x20;

해당 linux system 으로 들어오는 패킷을 허용하는 rule이다. 기본 포맷은 다음과 같다.

```csharp
[centos@test ~]$ iptables -I INPUT <other options>
```

*   \-A 옵션 (append) 해당 룰을 지정한 INPUT chain의 마지막 rule으로 append한다.\
    보통은 Input chain의 가장 마지막에 모든 패킷을 drop 시키는 rule을 추가하고는 한다.\


    ```csharp
    [centos@test ~]$ sudo iptables -A INPUT -j DROP

    [centos@test ~]$ sudo iptables -nvL --line-numbers
    Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
    num   pkts bytes target     prot opt in     out     source               destination
    1     1232  165K ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
    2        0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0
    3        0     0 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0
    4      105  5524 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            state NEW tcp dpt:22
    5      294 15533 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited
    6        0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0
    ```



*   \-I 옵션 (insert) 지정한 chain의 몇번째에 insert 할지 정할 수 있다. 별도의 line number를 지정하지 않았을 경우 default로는 제일 상단에 rule을 추가한다.\
    예를 들어 ssh 접속을 위해 22번 input packet을 허용하는 rule은 다음과 같이 추가할 수 있다. \


    ```csharp
    [centos@test ~]$ sudo iptables -I INPUT 2 -s 192.168.0.30/24 -p tcp --dport 22 -j ACCEPT

    [centos@test ~]$ sudo iptables -nvL --line-numbers
    Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
    num   pkts bytes target     prot opt in     out     source               destination
    1     2344  300K ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
    2        0     0 ACCEPT     tcp  --  *      *       192.168.0.0/24       0.0.0.0/0            tcp dpt:22
    ```



*   \-D (delete) 지정한 line에 있는 rule을 삭제한다.\


    ```csharp
    [centos@test ~]$ sudo iptables -D INPUT 2

    [centos@test ~]$ sudo iptables -nvL --line-numbers
    Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
    num   pkts bytes target     prot opt in     out     source               destination
    1     2778  357K ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
    2        0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0
    ```



### OUTPUT Chain / FORWORD Chain

OUTPUT chain은 해당 서버에서 바깥으로 나가는 outgoing packet에 대한 rule에 대한 것이고,\
FORWORD Chain은 해당 리눅스 시스템을 통과하는 패킷에 대한 것이다.\
insert, append, delete 등에 대한 명령어 규칙은 INPUT chain 하면서 알아본 것과 동일하다.

```csharp
[centos@test ~]$ iptables -I OUTPUT <other options>
[centos@test ~]$ iptables -I FORWORD <other options>
```



### iptables 영구 적용

설정한 iptables가 reboot 후에도 지속되기를 원한다면, iptables 룰을 `/etc/sysconfig/iptables` 경로  저장하여  수정한 뒤 다시 import 해 준다.&#x20;

```shell
[root@test ~]# sudo iptables-save > /etc/sysconfig/iptables
.. 룰 추가
[root@test ~]# sudo iptables-restore < /etc/sysconfig/iptables
```

#### &#x20;

#### 참고 문서

* [Sysadmin tools: How to use iptables](https://www.redhat.com/sysadmin/iptables)
* [5.13. SETTING AND CONTROLLING IP SETS USING IPTABLES](https://access.redhat.com/documentation/en-us/red\_hat\_enterprise\_linux/7/html/security\_guide/sec-setting\_and\_controlling\_ip\_sets\_using\_iptables)
* [Sysadmin tools: How to use iptables](https://www.redhat.com/sysadmin/iptables)

&#x20;

