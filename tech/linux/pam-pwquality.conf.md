---
description: 2022. 9. 24. 22:00
---

# PAM 개념 및 패스워드 복잡성 설정 방법 (pwquality.conf)

## Intro

PAM (Pluggable Authentication Modules) 은 console/network ssh 등으로 시스템에 로그인 할 때 사용자 인증에 사용되는 모듈이다.\
PAM 은 다양한 application 들이 각자의 상황에 맞는 인증을 하도록 모듈화 할 수 있다.

<figure><img src="../../.gitbook/assets/image (3).png" alt=""><figcaption><p>image from <a href="https://www.redhat.com/sysadmin/pluggable-authentication-modules-pam">https://www.redhat.com/sysadmin/pluggable-authentication-modules-pam</a></p></figcaption></figure>



pam 은 root 를 포함하여 모든 계정에 공통적으로 적용되기 때문에 매우 조심스럽게 설정해야 한다.\
pam 을 수정해야 하는 경우 **root 세션을 추가로 연결해 두어** 문제 발생시 재설정을 할 수 있도록 대비하는 것이 좋다.

<mark style="color:red;">**FYI**</mark>\
해당 게시글은 PAM 에 대해 알아보기 위해 작성된 것으로,\
RHEL8 부터는 pam 파일을 직접적으로 수정하는 것은 권장 되지 않고 있다.\
기존의 authconfig 명령어가 authselect 로 대체되었으며, 되도록 authselect 로 PAM을 제어하는 것이 좋다.&#x20;



## pam 구성

### /etc/pam.d

/etc/pam.d 하위에는 각 application들에 대한 PAM 설정 파일들이 존재하며, 이는 libpam.so 라이브러리를 호출해서 동작한다. libpam은 PAM 모듈 로딩을 제어하는 PAM API 라이브러리이다.

그 중에서도 여러 application 에서 공통적으로 쓰이는 모듈은 system-auth, password-auth 에 설정되어 있다.

_**system-auth**_ : 시스템 계정들에 대한 local authentication 의 설정을 담당한다.\
_**password-auth**_ : ssh, vsftpd 같이 원격에서 접속하는 계정에 대한 인증을 담당한다.

system-auth 와 password-auth 파일은 cron, sshd와 같은 pam 설정 파일에서 용도에 맞게 include 되어 사용 된다.

( 서비스 이름별로 지정된 PAM 파일 명은 [https://docs.oracle.com/cd/E19683-01/816-4883/6mb2joao3/index.html#pam-31](https://docs.oracle.com/cd/E19683-01/816-4883/6mb2joao3/index.html#pam-31)에서 확인 가능하다. )

```shell-session
[root@wglee-vm ~]# cat /etc/pam.d/systemd-user  
# This file is part of systemd.  
#  
# Used by systemd --user instances.  

account  include system-auth  
session  include system-auth
```

### **/usr/lib64/security**

다양한 PAM 모듈에 대한 실행파일들이 존재한다.\
대부분의 모듈 이름으로 man 페이지를 확인할 수 있다.

<figure><img src="https://blog.kakaocdn.net/dn/MAICT/btrMU602qD7/mOSmcWgkzgvNPUsHuidwEk/img.png" alt=""><figcaption></figcaption></figure>

### **/etc/security**

특정 pam 모듈에 대한 추가적인 설정 파일이 위치해 있다.\
pam\_pwquality 모듈 같은 경우는 /etc/pam.d 에서 직접적으로 제어하는 것보다 /etc/security/pwqulity.conf 에서 훨씬 간편하게 수정할 수 있다.

<figure><img src="https://blog.kakaocdn.net/dn/dfHqDs/btrMVIlb89E/Ch7KH6TiPhOvRa4FjnPrHK/img.png" alt=""><figcaption></figcaption></figure>

### **/var/log/secure**

로그인 인증 및 세션에 대한 로그가 남는다.

<figure><img src="https://blog.kakaocdn.net/dn/Cnik0/btrMYTl08Mi/1E5auqObwIyC525uSVLvh1/img.png" alt=""><figcaption></figcaption></figure>

wglee3 계정에서 wglee05로의 패스워드 인증 실패 한 것에 대해 다음과 같이 로그가 확인된다.

<figure><img src="https://blog.kakaocdn.net/dn/cEKI4q/btrMVu1LO06/tTX5LztmIckDNltDALi6M0/img.png" alt=""><figcaption></figcaption></figure>

### pam configuration 구조

PAM config 는 <mark style="background-color:blue;">`module_interface   control_flag   module_name   module_arguments`</mark>구조를 가진다.

#### <mark style="color:blue;">module\_interface</mark>

auth, account, password, session 과 같은 module interface가 존재한다. 이는 인증에서 수행되는 각 단계를 나타낸다.\
**auth** : 계정 인증에 사용된다. 패스워드의 유효성을 요청/검사 하는 단계이다.\
**account** : access 허용 여부를 확인한다. 예를 들어 계정이 만료 되었는지, 혹은 특정 시간에만 로그인 할 수 있는 계정인지 확인한다.\
**password** : 패스워드 설정(변경) 시에 사용 된다. pwqulity 모듈의 password 인터페이스에서 패스워드 변경 시에 특정 복잡도를 만족해야만 변경 가능하도록 설정할 수 있다.\
**session** : 계정의 세션에 대한 설정을 할 수 있다.

#### <mark style="color:blue;">control\_flag</mark>

pam 모듈이 인증 결과에 대한 상태를 성공/실패로 반환하면, 이를 어떻게 처리할 것인지를 결정한다.\
**required** : 모듈의 결과가 성공일 때만 인증을 계속한다. 실패해도 해당 인터페이스에 등록된 모든 모듈이 수행될 때까지 사용자 계정에 노티를 하지 않는다.\
**requisite** : 모듈의 결과가 성공일 때만 인증을 계속한다. 단 테스트가 실패하면 사용자 계정에 메세지와 함께 즉시 알린다.\
**sufficient** : sufficient 라인을 실행하기 전에 required 에서 실패가 없었으며, sufficient 에서 인증 결과가 성공이라면 나머지 인증 과정은 실행하지 않는다.\
**optional** : 모듈의 인증 결과가 실패여도 인증에 영향을 주지 않는다.\
**include** : 다른 control flag와 달리, 모듈 결과를 다루는 flag가 아니다. 다른 config 파일의 설정을 import 하기 위해 사용한다.

#### <mark style="color:blue;">module\_name</mark>

사용할 모듈을 지정한다.

#### <mark style="color:blue;">module arguments</mark>

모듈에 설정할 각 옵션을 의미한다.



## PAM 설정 예시

### pwqaulity.conf 패스워드 복잡도 설정

```shell-session
[root@wglee-vm ~]# cat /etc/security/pwquality.conf | grep -v '^#\|^$'
minlen = 15
dcredit = -1
ocredit = -1
```

**minlen** : 패스워드의 최소 길이 설정\
**ocredit** : 포함 되어야 할 특수 문자의 개수 지정. 양수 1 이면 제한이 없는 것이고, -N과 같이 음수로 설정하면 특수문자 N개가 최소로 들어가야 한다는 뜻이다.\
**dcredit** : 포함 되어야 할 숫자의 개수 지정. 양수 1 이면 제한이 없는 것이고, -N과 같이 음수로 설정하면 숫자 N개가 최소로 들어가야 한다는 뜻이다.

이 외에도 ucredit ( 대문자 ) , lcredit ( 소문자 ) 등의 옵션을 사용할 수 있다.\
옵션 별로 자세한 설명은 pwqaulity.conf의 주석을 참고한다.

참고로 /etc/pam.d/sytemd-auth 및 password-auth 에서 pwquality 모듈은 password 인터페이스 부분에 다음과 같이 설정되어 있다.

```
password    requisite     pam_pwquality.so try_first_pass local_users_only retry=3 authtok_type=
```

pwquality.so 모듈에 의해 패스워드 복잡도 검사를 하여 실패시 BAD PASSWORD 문구가 보여진다.

```shell-session
# 15자 이상 미충족
[cafetestuser@wglee-vm ~]$ passwd
Changing password for user cafetestuser.
Changing password for cafetestuser.
(current) UNIX password:
New password:
BAD PASSWORD: The password is shorter than 14 characters

# 특수 문자 1개 이상 미충족
[cafetestuser@wglee-vm ~]$ passwd
Changing password for user cafetestuser.
Changing password for cafetestuser.
(current) UNIX password:
New password:
BAD PASSWORD: The password contains less than 1 non-alphanumeric characters

# 숫자 1개 이상 미충족
[cafetestuser@wglee-vm ~]$ passwd
Changing password for user cafetestuser.
Changing password for cafetestuser.
(current) UNIX password:
New password:
BAD PASSWORD: The password contains less than 1 digits
```



### 패스워드 재사용 금지 및 연속적인 실패시 lock

#### <mark style="color:blue;">패스워드 3회 재사용 금지</mark>

pam\_pwhistory.so 모듈을 사용하여 설정할 수 있다.

이때, sufficient 아래에 설정하면 sufficient 모듈의 성공 시 인증 과정이 패스될 가능성이 있으므로 sufficient 위쪽에 추가하도록 한다.

```shell-session
[root@wglee-vm ~]# cat /etc/pam.d/password-auth | grep password
password    requisite     pam_pwquality.so try_first_pass local_users_only retry=3 authtok_type=
password    requisite     pam_pwhistory.so debug use_authtok remember=3 enforce_for_root
password    sufficient    pam_unix.so sha512 shadow nullok try_first_pass use_authtok
password    required      pam_deny.so
```

**use\_authtok** : 패스워드 재설정 시 새로운 패스워드를 설정하도록 한다.\
**remember=N** : 지정한 수 만큼 패스워드 히스토리릉 보고 재사용하지 못하도록 한다. 기존 패스워드는 /etc/security/opasswd 에 암호화 되어 기록된다.

#### <mark style="color:blue;">3회 실패시 계정 lock</mark>

계정 잠금 임계치를 설정하는 모듈로 pam\_faillock.so 가 있다.\
RHEL8 부터는 기존에 주로 사용하던 pam\_tally2.so 가 deprecated 되었다.\
블로그의 앞부분에서 설명한 것처럼, RHEL8 부터는 authselect 명령어로 사용해서 faillock 모듈을 설정하는 것이 좋다.

```shell-session
[root@wglee-vm ~]# cat /etc/pam.d/password-auth | grep lock
auth        required      pam_tally2.so  file=/var/log/tallylog deny=5 onerr=fail unlock_time=10

[root@wglee-vm ~]# cat /etc/pam.d/system-auth | grep lock
auth        required      pam_tally2.so  file=/var/log/tallylog deny=5 onerr=fail unlock_time=10
```

**deny=5** : 5회 이상 로그인이 실패하면 계정을 lock 하도록 한다.\
**onerr=fail** : 비정상적인 결과가 발생하면 pam error 코드를 반환하도록 한다.\
**unlock\_time** : 해당 초(second) 후에 unlock 되도록 한다.



## 참고

{% embed url="https://www.redhat.com/sysadmin/pluggable-authentication-modules-pam" %}

{% embed url="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/chap-hardening_your_system_with_tools_and_services" %}

{% embed url="https://support.huaweicloud.com/intl/en-us/hss_faq/hss_01_0043.html" %}

{% embed url="https://www.linuxquestions.org/questions/linux-server-73/setting-password-complexity-not-working-as-root-4175423722/" %}

{% embed url="https://docs.oracle.com/cd/E19683-01/816-4883/pam-32/index.html" %}
