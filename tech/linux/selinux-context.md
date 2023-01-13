---
description: Written on 2021. 8. 26. 00:29
---

# SELinux 개념 및 Context 변경하기

## SELinux(Security Enhanced Linux) 란?

SELinux는 리눅스의 보안 기능으로, 파일과 같은 리눅스의 리소스에 대한 접근 권한을 제어할 수 있다.\
어떤 프로세스가 어떤 파일, 디렉터리, 포트에 접근 가능한지 세부적인 규칙을 설정할 수 있게 된다.\
\


## SELinux Modes

1. Enforcing\
   접근 제어 룰이 허용된 상태
2. Permissive\
   SELinux 의 접근 제어 룰이 적용 되지는 않고, 위반 된 사항들만 로그로 남기는 상태
3. Disables\
   SELinux가 완전히 비활성화 된 상태.

아래와 같이 재부팅 없이 설정이 가능하다. 만약 /etc/selinux/config 파일을 수정해서 지정할 경우 서버 재부팅을 해야 한다. 재부팅 시에도 해당 모드를 유지하려면 /etc/selinux/config 에서 커널 파라미터를 수정해야 한다.

```shell-session
[root@server-a ~]# setenforce
usage:  setenforce [ Enforcing | Permissive | 1 | 0 ]

[root@server-a ~]# setenforce 0  
[root@server-a ~]# getenforce  
Permissive
```

\


### SELinux labels

특정 프로세스에 어떤 SELinux 룰이 적용되어 있는지 보고 싶으면 보통 -Z 옵션을 사용하면 된다.\
ps, ls, cp, mkdir 등의 명령어에서 활용 가능하다.

```shell-session
[root@server-a ~]# ll -Z /var/www  
total 0  
drwxr-xr-x. 2 root root system_u:object_r:httpd_sys_script_exec_t:s0 6 Nov 12 04:58 cgi-bin  
drwxr-xr-x. 2 root root system_u:object_r:httpd_sys_content_t:s0 6 Nov 12 04:58 html

[root@server-a ~]# ps -ZC httpd  
LABEL PID TTY TIME CMD  
system_u:system_r:httpd_t:s0 13744 ? 00:00:00 httpd  
system_u:system_r:httpd_t:s0 13745 ? 00:00:00 httpd
```



조회되는 Label의 의미는 다음과 같다.

```shell-session
# SELinux_user:Role:Type:Level
system_u:system_r:httpd_t:s0
```

자세하게 알아보지는 않지만, 여기서 Role 부분이 해당 파일에 접근 가능한 프로세스와 연관이 있다.\
httpd\_t Role을 가진 프로세스는 httpd\_sys\_script\_exec\_t Role이 적용된 디렉터리에 접근 가능하다.\
하지만 httpd\_t Role이 없는 mariadb 프로세스의 경우 httpd\_sys\_script\_exec\_t 에 대한 권한 또한 없기 때문에 /var/www/html에 접근이 불가능 하다.\
\
\


## 디렉터리의 SELinux context 변경하기

만약 httpd 프로세스가 접근 해야 하는 /var/www/html 디렉터리의 SELinux role이 httpd에 관한 것이 아니면, 다음과 같이 403이 발생하게 된다.

```shell-session
[root@server-a html]# ls -Z
unconfined_u:object_r:default_t:s0 index.html

[root@server-a html]# curl http://localhost/index.html
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access this resource.</p>
</body></html>
```



messages 로그에서 selinux 에 의해 접근 거부 되었음을 확인할 수 있다.

```shell-session
May 13 06:23:10 server-a setroubleshoot[1447]: SELinux is preventing httpd from getattr access on the file /var/www/html/index.html. For complete SELinux messages run: sealert -l 3fb750be-d8d0-4e69-9076-a8b45d87f160
May 13 06:23:10 server-a platform-python[1447]: SELinux is preventing httpd from getattr access on the file /var/www/html/index.html.#012#012
```



audit.log를 참조하는 ausearch 명령어를 사용해서 권한 거부된 이력을 확인할 수도 있다. SELinux 관련 로그는 "AVC" 메세지로 검색하면 된다.

```shell-session
[root@server-a html]#  ausearch -m AVC  | tail -n 2
time->Fri May 13 06:27:32 2022
type=AVC msg=audit(1652423252.390:108): avc:  denied  { getattr } for  pid=1191 comm="httpd" path="/var/www/html/index.html" dev="vda1" ino=16778211 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:default_t:s0 tclass=file permissive=0
```



다음과 같이 해당 디렉터리에 대한 권한을 변경하고, 반영한다.

```shell-session
one of the arguments -a/--add -d/--delete -m/--modify -l/--list -E/--extract -D/--deleteall is required

[root@server-a html]# semanage fcontext -m -t httpd_sys_content_t '/var/www/html(/.*)?'

[root@server-a html]# restorecon -R /var/www/html/

[root@server-a html]# ls -Z
unconfined_u:object_r:httpd_sys_content_t:s0 index.html
```



이제 다시 curl 이 되는 것을 볼 수 있다 ^^

```shell-session
[root@server-a html]# curl http://localhost/index.html
hello world
```

