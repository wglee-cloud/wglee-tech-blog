# NetworkManager - nmcli 명령어로 리눅스 네트워크 설정하기

## Intro

nmcli 명령어는 NetworkManager 를 cli 로 제어하고, 현재 네트워크 디바이스(인터페이스) 및 커넥션의 상태를 알려주는 명령어이다.\
NetworkManager는 /etc/sysconfig/network-scripts 디렉터리 하위의 인터페이스 설정 파일을 참조하여 네트워크를 구성하는 데몬이다.\
nmcli con mon 명령어로 수정하는 내용은 해당 디렉터리 하위의 ifcfg-(interface\_name) 파일에 반영이 된다.



## 네트워크 인터페이스 상태 확인 하기

nmcli 명령어로 네트워크 인터페이스의 상태를 같이 확인 할 수 있다.

```shell-session
[root@server-a ~]# nmcli dev status
DEVICE  TYPE      STATE      CONNECTION
eth0    ethernet  connected  System eth0
lo      loopback  unmanaged  --
```



## nmcli 명령어로 connection add / modify

nmcli con add 명령어로 새로운 connection을 생성한다.

```shell-session
# nmcli con add con-name "test-con" ifname eth0 type ethernet ipv4.method manual ipv4.address 192.168.1.10/24 ipv4.gateway 192.168.1.1
```

nmcli con mod 명령어로 이미 있는 connection을 수정할 수 있다.

```shell-session
# nmcli con mod "test-con" ipv4.address 192.168.1.15/24
```

새롭게 등록한 test-con 을 활성화 시킨다.

```shell-session
# nmcli con up "test-com"
```



## nmcli 명령어와 ifcfg-\* 파일 구문 비교

cli 와 /etc/sysconfig/network-scripts/ifcfg-\* 파일 구문을 비교하면 다음과 같다.

|                                                   | nmcli con mod                   | ifcfg-\* 옵션                    |
| ------------------------------------------------- | ------------------------------- | ------------------------------ |
| ipv4 아이피 정적으로 설정                                  | ipv4.method manual              | BOOTPROTO=none                 |
| ipv4 아이피 dhcp로 자동 할당                              | ipv4.method auto                | BOOTPROTO=dhcp                 |
| ipv4 아이피 할당                                       | ipv4.addresses 192.0.2.1/24     | IPADDR=192.168.0.10 PREFIX=24  |
| default gateway 설정                                | ipv4.gateway 192.0.2.254        | GATEWAY=192.168.0.1            |
| 해당 네임서버 이용하도록 /etc/resolv.conf 수정                 | ipv4.dns 8.8.8.8                | DNS1=8.8.8.8                   |
| 부팅할 때 자동으로 connection 활성화                         | connection.autoconnect yes      | ONBOOT=yes                     |
| 본 connection 의 이름                                 | connection.id eth0              | NAME=eth0                      |
| <p>본 connection이 연결된 network interface 지정<br></p> | connection.interface-name eth0  | DEVICE=eth0                    |

만약 ifcfg 설정 파일을 수정했을 경우, NetworkManager 데몬이 변경 사항을 반영하도록 reload를 해야 한다.

```shell-session
# nmcli con reload
```



## nmcli command examples

{% embed url="https://www.golinuxcloud.com/nmcli-command-examples-cheatsheet-centos-rhel/" %}
