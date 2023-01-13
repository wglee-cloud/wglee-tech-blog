---
description: 2022. 7. 4. 23:49
---

# find 명령어 -exec 옵션으로 검색된 파일에 대해 명령 실행하기

## Intro

linux에서 find 명령어로 원하는 파일을 검색한 다음, -exec 옵션을 사용하여 검색한 대상에 대해 명령어를 실행할 수 있다.\
쓸 일이 종종 있는 것 같아서 예시를 한번 정리해 본다.

/home/wglee 디렉터리 하위에 "file" 로 시작하는 파일명을 찾는다.

```shell-session
[root@wglee ~]# find /home/wglee -name "file*"
/home/wglee/file1
/home/wglee/file2
/home/wglee/file3
/home/wglee/file4
/home/wglee/file5
/home/wglee/file6
```

이 파일들에 대해서 ls -al 로 조회하고 싶으면, 다음과 같이 한다.\
여기서 -name 은 파일명으로 찾겠다는 옵션이고, -size, -type, -ctime 처럼 다른 옵션들도 사용할 수 있다.

```shell-session
find 경로 -name "test" -exec 명령어 {} ;
```

여기서 {} 는 find 로 조회한 file 이름들을 {} 에 넣는다는 뜻이다.\
그리고 -exec 인자를 구분하는 지시자 (delimiter)로 `;` 혹은 `+`를 사용할 수 있다.\
만약 `;` 를 사용하면 각 라인은 자동 줄바꿈이 된다. 이 경우 escape 하기 위해 앞에 `\`를 붙여야 한다.\
`+`를 사용하면 결과물은 줄바꿈 없이 concat 되어서 반환된다.

```shell-session
[root@wglee ~]# find /home/wglee -name "file*" -exec ls -l {} \;
-rwxr-xr-x 1 root root 0 Jul  4 23:23 /home/wglee/file1
-rwxr-xr-x 1 root root 0 Jul  4 23:23 /home/wglee/file2
-rwxr-xr-x 1 root root 0 Jul  4 23:23 /home/wglee/file3
-rwxr-xr-x 1 root root 0 Jul  4 23:23 /home/wglee/file4
-rwxr-xr-x 1 root root 0 Jul  4 23:23 /home/wglee/file5
-rwxr-xr-x 1 root root 0 Jul  4 23:23 /home/wglee/file6

# 권한이 755인 파일들을 644로 설정하기
[root@wglee ~]# find /home/wglee -perm 0755 -exec chmod 0644 {} \;
```

`\;` 지시자를 사용한 경우

```shell-session
[root@wglee ~]# find /home/wglee -name "file*" -exec ls -l {} \;
-rw-r--r-- 1 root root 0 Jul  4 23:23 /home/wglee/file1
-rw-r--r-- 1 root root 0 Jul  4 23:23 /home/wglee/file2
-rw-r--r-- 1 root root 0 Jul  4 23:23 /home/wglee/file3
-rw-r--r-- 1 root root 0 Jul  4 23:23 /home/wglee/file4
-rw-r--r-- 1 root root 0 Jul  4 23:23 /home/wglee/file5
-rw-r--r-- 1 root root 0 Jul  4 23:23 /home/wglee/file6
```

`+` 지시자를 사용한 경우

```shell-session
[root@wglee ~]# find /home/wglee -name "file*" -exec echo {} +
/home/wglee/file1 /home/wglee/file2 /home/wglee/file3 /home/wglee/file4 /home/wglee/file5 /home/wglee/file6
```



## 참고 문서

{% embed url="https://www.baeldung.com/linux/find-exec-command" %}

