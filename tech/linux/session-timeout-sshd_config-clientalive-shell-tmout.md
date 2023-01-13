---
description: 2022. 9. 24. 22:01
---

# \[Session Timeout] sshd\_config의 ClientAlive\* 와 Shell 변수 TMOUT 의 차이

## Intro

Linux 세션 타임아웃을 설정하기 위해 sshd\_config 의 ClientAliveInterval, ClientAliveCountMax를 설정했으나 나의 의도처럼 동작하지 않았다.\
나의 목표는 터미널에서 60초 동안 아무런 명령어를 치지 않았을 때 세션이 끊기는 것이었다.\
sshd\_config 의 ClientAlive\* 지시자 들은 쓰임새가 다른 것인데 그것을 login session timeout으로 오해해서 생긴 문제였다.\
나와 같은 오해를 하는 케이스가 있을 수 있을 것 같아 정리해 본다.



## sshd\_config - ClientAliveInterval, ClientAliveCoundMax

서버는 client system이 살아있는지 확인하기 위해 packet을 보내고, 응답을 받으면 세션을 지속하는 방법으로 동작한다.

**ClientAliveInterval** 로 client 로 응답을 받기 까지 세션을 유지 (대기) 할 시간을 설정한다. 단위는 second 이다.\
**ClientAliveCountMax** 는 client 가 응답이 없어도 접속을 유지하는 횟수이다.

따라서, 총 세션 시간은 ClientAliveInterval X ClientAliveCountMax 가 된다.

```shell-session
[root@wglee-server ~]# cat /etc/ssh/sshd_config | grep Client
ClientAliveInterval 60
ClientAliveCountMax 1
```

**다만!!**\
위에서 설정한 ClientAlive\* 지시자들을 설정함으로써 ssh 접속한 client가 5초 동안 터미널에 아무런 명령어를 치지 않았다고 세션이 끊기는 것은 아니다.\
sshd\_config 에서 설정한 ClientAlive\* 는 말 그대로 sshd 에 대한 설정이다.\
client 가 shell 에 로그인한 상태이면 기본적으로 client는 서버의 요청에 응답 패킷을 보내기 때문에 서버는 client 가 살아 있다고 생각한다.\
전체 client 가 죽는 상황( ex. 예기치 못하게 종료되거나 네트워크가 끊김)이면 더 이상 서버에게 세션 응답 packet을 보낼 수 없는데 ClientAlive\* 지시자는 이런 상황을 감지하는 것이다.\
그래서 일정 시간 동안 로그인한 shell과 "상호작용" 여부에 따라 세션을 종료하려면 **TMOUT** shell 환경변수를 설정해야 한다.



## TMOUT environment variable

TMOUT은 사용자 계정의 login session 을 유지하는 시간을 설정하는 shell 환경 변수이다. 지정 시간 동안 계정의 activity가 없으면 login shell에서 자동 로그아웃을 한다. ( 단위 : seconds )\
특히 root 권한으로 로그인하여 작업 하지 않을 때도 방치해 두면 악의적인 사용이나 공격에 노출될 위험이 있다.\
때문에 일정 시간 이후에는 자동으로 로그아웃 되도록 할 필요가 있다.&#x20;

```shell-session
[root@wglee-server ~]# tail -n 2 /etc/profile
export HISTTIMEFORMAT="[%Y/%m/%d %H:%M] "
export TMOUT=60

[root@wglee-vm ~]# history timed out waiting for input: auto-logout
[centos@wglee-vm ~]$

[wglee3@wglee-vm ~]$ timed out waiting for input: auto-logout
[root@wglee-vm ~]#
```



## 참고

{% embed url="https://www.geeksforgeeks.org/auto-logout-in-linux-shell-using-tmout-shell-variable/" %}

{% embed url="https://superuser.com/questions/1283597/ssh-disable-port-forward-remotely-instead-of-killing-process-on-the-server/1283733#1283733" %}

{% embed url="https://unix.stackexchange.com/questions/585862/ssh-clientalivecountmax-setting-does-not-seem-to-work-and-to-disconnect-the-user" %}

{% embed url="https://www.redhat.com/sysadmin/eight-ways-secure-ssh" %}

