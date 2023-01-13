---
description: Written on 2021. 2. 26. 22:23
---

# SUDO / SU / Wheel Group

## Overview

흔히 root 권한으로 변경하는 명령어로 sudo와 su를 사용하고는 한다.

하지만 둘 사이에는 차이점이 있으며, 이에 대해 알아보도록 한다.



`su`와 `sudo` 모두 현재 유저에 부여된 권한을 높이는 명령어이다.

차이점은 `su`는 변환하고자 하는 계정의 password를 요구하며, `sudo`는 현 사용자의 계정의 password를 요구한다.



`su` 는 완전히 root 계정으로 switch 하는 명령어로, 모든 리눅스 시스템에 root 권한으로 접근하게 되어 권장하지 않는다.

`sudo` 를 사용하면 해당 명령어에 대해서만 root 권한으로 실행하게 된다.&#x20;

&#x20;

## **SU  command**

### **(login) su -  /  su -l**&#x20;

su - 는 변경하고자 하는 사용자의 home directory로 이동한다. 변경할 계정을 명시하지 않은 경우 root로 변경한다.

su - 와 비슷하게 su -l 도 같은 기능을 한다.

```shell-session
[centos@wglee-server ~]$ su - testuser
Password:
Last login: Sat Feb 27 15:26:18 KST 2021 on pts/0

[testuser@wglee-server ~]$ pwd
/home/testuser
```

그냥 su를 하면 기존 shell에서 사용하던 directory를 유지한다.

```shell-session
[centos@wglee-server ~]$ su testuser
Password:
[testuser@wglee-server centos]$ pwd
/home/centos
```

&#x20;

## **SUDO command**

### **sudo**

sudo는 현 사용자 계정이 root 권한을 빌려 명령어를 수행할 수 있게 한다. sudo 자체도 "super user do" 라는 뜻이다!

su와 달리 sudo는 지금 로그인 중인 계정의 password를 요구한다.

sudo 는 오직 sudoers group에 속한 user에 대해서만 적용 가능하다.

```shell-session
[testuser@wglee-server root]$ sudo systemctl restart httpd

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for testuser:
testuser is not in the sudoers file.  This incident will be reported.
```

{% hint style="info" %}
When you want to run a command that requires root rights, Linux checks your username against the sudoers file. This happens when you type the command “sudo”. If it determines, that your username is not on the list, you cannot run the command/program logged in as that user.
{% endhint %}

****

### **sudo -i  / sudo -s**

sudo는 root권한을 잠깐 빌리는 용도이지만, -i 또는 -s 옵션을 줘서 root 유저로 아예 switch 할 수 있다.

해당 옵션도 sudoers group에 속하지 않은 계정에서 실행하면 에러를 발생시키게 된다.

&#x20;

sudo -s는 "sudo su"의 줄임말이다. 이는 "su"를 sudo로 실행시킴을 의미한다.

결국 이는 중복된 기능의 명령어를 함께 사용하는 것으로, 권장되지 않는다.

```shell-session
[centos@wglee-server root]$ sudo -i
[root@wglee-server ~]# pwd
/root

[centos@wglee-server root]$ sudo -s
[root@wglee-server ~]# pwd
/root
```

&#x20;

### sudoers group

이번에는 sudoers group에 사용자를 추가하는 방법을 알아본다.

sudoers group은 root 권한을 사용할 수 있는 user의 집합이다.

&#x20;

내가 생성한 testuser 계정은 root 권한이 없는 일반 계정이다.

```shell-session
[centos@wglee-server ~]$ sudo adduser test_user
[centos@wglee-server ~]$ groups test_user
test_user : test_user
```

&#x20;

이제 root 권한을 사용할 수 있도록 해당 계정을 sudoers group에 포함시킨다.

Debian 계열의 경우 sudo group을 사용하고, centos의 경우 wheel 그룹이다.

```shell-session
# Debian 계열
root@test: usermod -aG sudo testuser

# rhel, centos 계열
[centos@wglee-server ~]$ sudo usermod -aG wheel testuser

[testuser@wglee-server ~]$ groups testuser
testuser : testuser wheel
```

&#x20;

이제 아까는 안됐던 sudo 명령어를 사용할 수 있다.

```shell-session
[testuser@wglee-server ~]$ sudo systemctl restart httpd
```

#### &#x20;

## 참고문서

* [The Difference Between Sudo And Su Explained](https://phoenixnap.com/kb/sudo-vs-su-differences)
* [How To Use The Su Command In Linux With Examples](https://phoenixnap.com/kb/su-command-linux-examples)
* [Configuring the linux Sudoers file](https://www.linux.com/training-tutorials/configuring-linux-sudoers-file/)
* [www.quora.com/What-is-a-Sudo-group](https://www.quora.com/What-is-a-Sudo-group)
* [What are the differences between “su”, “sudo -s”, “sudo -i”, “sudo su”?](https://www.quora.com/What-are-the-differences-between-su-sudo-s-sudo-i-sudo-su)

&#x20;

