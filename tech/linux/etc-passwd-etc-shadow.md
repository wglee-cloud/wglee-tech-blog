---
description: Written on 2021. 8. 26. 00:08
---

# 사용자 계정 정보 - /etc/passwd , /etc/shadow

## Intro

리눅스는 사용자 계정 정보를 파일로 관리한다.&#x20;

사용자 계정 정보를 가지는 대표적인 파일은 /etc/passwd, /etc/shadow가 있다.

가장 기본적인 것 같지만 꼭 알아두어야 하는 두 파일에 대해 알아보도록 한다.

&#x20;

## /etc/passwd

/etc/passwd는 시스템에 로그인해서 리소스를 사용할 수 있는 사용자의 리스트를 담는다.

만약 centos 계정으로 시스템에 로그인/로그아웃 할때 /etc/passwd 파일에 근거해서 동작한다.&#x20;

계정명, UID, GID, 로그인 쉘 등에 대한 정보를 가지는데, 각 항목은 콜론(:)을 기준으로 구분된다.&#x20;

```shell-session
[centos@wglee ~]$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
games:x:12:100:games:/usr/games:/sbin/nologin
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody:x:99:99:Nobody:/:/sbin/nologin
systemd-network:x:192:192:systemd Network Management:/:/sbin/nologin
dbus:x:81:81:System message bus:/:/sbin/nologin
polkitd:x:999:998:User for polkitd:/:/sbin/nologin
rpc:x:32:32:Rpcbind Daemon:/var/lib/rpcbind:/sbin/nologin
rpcuser:x:29:29:RPC Service User:/var/lib/nfs:/sbin/nologin
nfsnobody:x:65534:65534:Anonymous NFS User:/var/lib/nfs:/sbin/nologin
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
postfix:x:89:89::/var/spool/postfix:/sbin/nologin
chrony:x:998:995::/var/lib/chrony:/sbin/nologin
centos:x:1000:1000:Cloud User:/home/centos:/bin/bash
wglee:x:1001:1001::/home/wglee2:/bin/bash
```

/etc/passwd 파일을 열면 위와 같이 보이는데, 세부 항목은 다음과 같다.&#x20;

```shell-session
# Username:패스워드:UserID:GroupID:FullName:홈디렉터리:로그인쉘
wglee:x:1001:1001::/home/wglee2:/bin/bash
```

&#x20;여기서 패스워드 필드는 x라고 쓰인 것을 볼 수 있다. 대부분의 리눅스 배포판은 패스워드를 암호화하여 /etc/shadow에 별도로 저장한다.

* **로그인 쉘**은 사용자가 시스템에 로그인 할 때 자동으로 사용하게 되는 쉘을 의미한다.
  * 레드햇 계열 OS에서는 /bin/bash를 default로 사용하며, 해당 필드가 비어있을 경우 /bin/sh가 로그인 쉘이 된다.
  * /sbin/nologin으로 설정되어 있으면해당 계정은 시스템에 로그인 할 수 없다.

&#x20;

## /etc/shadow

/etc/shadow 파일에서는 계정에 대한 패스워드를 암호화하여 관리한다.

보안상 중요한 계정 패스워드 정보를 가지기 때문에 root로만 접속 가능하다.

또한 패스워드 만기일, 계정 만기일도 설정할 수 있다.&#x20;
