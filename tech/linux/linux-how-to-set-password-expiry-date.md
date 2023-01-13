---
description: 2022. 9. 25. 17:15
---

# \[Linux] How to set password expiry date

## Intro

리눅스 시스템 관리시에 보안을 위해 사용자 패스워드를 정기적으로 재설정 하는 경우가 많다.\
계정의 패스워드의 만료일 뿐만 아니라 만료일 경고, 패스워드 최소 사용일 등도 설정 가능하다.\
/etc/login.defs 파일 수정 및 chage 명령어를 사용해서 이를 수행할 수 있다.



임의의 패스워드 사용 정책에 따라 아래와 같이 설정하도록 하겠다.\
\-- 패스워드 만료일 30\
\-- 패스워드 최소 사용일 1\
\-- 패스워드 만료 경고일 7



## /etc/login.defs

/etc/login.defs 파일은 계정 생성 시에 참조 되는 설정 파일이다.\
home directory 사용 여부, 기본 umask, shadow password 설정 등을 할 수 있다.\
다만 /etc/login.defs 파일에 설정한 내용은 수정 사항 반영 후 생성한 계정에만 반영된다.\
기존에 생성한 계정에는 영향을 미치지 않으므로 이 경우 chage 명령어로 설정해야 한다.

```shell-session
[root@wglee-vm ~]# cat /etc/login.defs | grep -v '^#\|^$' | grep PASS
PASS_MAX_DAYS   30
PASS_MIN_DAYS   1
PASS_MIN_LEN    5
PASS_WARN_AGE   7
```

**PASS\_MIN\_DAYS** : 패스워드 변경일로부터 최소 며칠이 경과해야 다른 패스워드로 변경이 가능한지 지정한다. 즉, 최소 사용일.\
**PASS\_MAX\_DAYS** : 패스워드 변경일로부터 변경 없이 사용할 수 있는 최대 일수를 지정한다.\
**PASS\_WARN\_AGE** : 패스워드 만료 며칠 전부터 사용자에게 경고를 할 것인지 지정한다.



## chage 명령어

`chage` 명령어를 사용해 패스워드 사용 기간을 설정할 수 있다.\
계정 생성 직후 처음 확인한 패스워드 수명(aging)은 다음과 같다.

```shell-session
[root@wglee-vm ~]# chage -l cafetestuser
Last password change                                    : Sep 13, 2022
Password expires                                        : never
Password inactive                                       : never
Account expires                                         : never
Minimum number of days between password change          : 0
Maximum number of days between password change          : 99999
Number of days of warning before password expires       : 7
```

chage 의 각 옵션을 사용해 아래와 같이 변경한다.

```shell-session
[root@wglee-vm ~]# chage -m 1 -M 30 -W 7 cafetestuser
[root@wglee-vm ~]# chage -l cafetestuser
Last password change                                    : Sep 14, 2022
Password expires                                        : Oct 14, 2022
Password inactive                                       : never
Account expires                                         : never
Minimum number of days between password change          : 1
Maximum number of days between password change          : 30
Number of days of warning before password expires       : 7
```

chage 옵션은 다음과 같다.\
**-m** : min days\
**-M** : max days\
**-W** : warn days\
**-I** : (대문자 i) inactive days\
**-d** : last chage date\
**-l** : (소문자 L) details of aging policy of an account\
**-E** : set the exact expiry date (YYYY-MM-DD 형식)



### **chage 명령어 활용 예시**

`chage` 명령어는 아래와 같이 쓰일 수도 있다.

```shell-session
@ 다음 로그인에서 바로 패스워드 재설정을 하도록 설정
# chage -d 0 cafetestuser

@ 2022-09-28 에 패스워드가 만료 되도록 설정
# chage -E 2022-09-28 cafetestuser

@ date 명령어로 특정 날짜를 계산하여 chage 명령어의 인자로 사용
# date +%Y-%m-%d
2022-09-25
# date -d +90days +%Y-%m-%d
2022-12-24
# chage -E $(date -d +90days +%Y-%m-%d) cafetestuser
```

### 참고

추가로, 계정 사용자의 퇴사 혹은 그 외 사유로 계정의 login 을 제한하기 위해서 usermod 명령어를 사용할 수 있다.

```shell-session
## Lock login
[root@wglee-vm ~]# usermod -L wglee05
[wglee3@wglee-vm root]$ su wglee05
Password:
su: Authentication failure

## Unlock login
[root@wglee-vm ~]# usermod -U wglee05
[wglee3@wglee-vm root]$ su wglee05
Password:
[wglee05@wglee-vm root]$
```



## 참고 문서

{% embed url="https://www.redhat.com/sysadmin/password-changes-chage-command" %}

{% embed url="https://www.redhat.com/sysadmin/password-expiration-date-linux" %}

{% embed url="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/4/html/system_administration_guide/s2-redhat-config-users-passwd-aging" %}

{% embed url="https://www.linuxquestions.org/questions/linux-newbie-8/problem-with-changing-etc-login-defs-4175541708/" %}
