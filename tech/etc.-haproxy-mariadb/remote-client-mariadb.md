# Remote client 에서 MariaDB 원격 접속 방법

## Intro

MariaDB에 원격 서버로부터의 접근을 허용하고, 사용자 계정에 따른 권한 부여를 하여 보안을 강화하도록 한다.

<figure><img src="https://blog.kakaocdn.net/dn/bPYwH9/btrLr1Nqu6X/lcrz8VVdKFX0wXlKBS86ak/img.png" alt=""><figcaption></figcaption></figure>

MariaDB client에 해당하는 WEB 서버에 MariaDB-Client 패키지를 설치하고,\
DB 서버에 Mariadb-Server 패키지를 설치한다.\
MariaDB 패키지를 설치하기 위해서는 아래 링크에서 yum repo를 복사하여 각 서버에 추가하여야 한다.\
[https://mariadb.org/download/?t=repo-config](https://mariadb.org/download/?t=repo-config)

```
## web 서버
[root@wglee-web ~]# yum search all MariaDB-client
[root@wglee-web ~]# yum install MariaDB-client

## db 서버
[root@wglee-db ~]# yum install mariadb-server
```



## MariaDB-Server setup

db 서버에서 아래의 명령어로 root 패스워드를 초기화 한다.\
해당 명령어는 shell script 기반으로 동작하며, 다음 항목들을 초기 설정할 수 있다.\
\-> root password 초기화\
\-> 원격지에서 root로 접근 못하도록 제한\
\-> MariaDB 설치시에 테스트 위해 생성된 anonymous 사용자 계정 제거\
\-> anonymous 사용자가 접근할 수 있는 test 데이터베이스 제거

```
[root@wglee-db ~]#  mysql_secure_installation
```

그 다음, root password 입력 없이 mysql에 접속할 수 있도록 my.cnf 에 설정한다.\
그리고 모든 대역에서 들어오는 connection을 처리하도록 0.0.0.0으로 바인딩 한다.\
( my.cnf는 mariaDB의 공식 설정 파일이다.)

```
[root@wglee-db ~]# cat /etc/my.cnf | grep -v '^$\|^#'
[mysqld]
...
# 추가
bind-address = 0.0.0.0
...
# 추가
[client]
user=root
password="[패스워드]"
```



## MariaDB-Server - Create Database, User

원격 서버에서 접근할 유저와, 해당 유저에게 접근 허용을 할 임의의 데이터베이스를 생성한다.

```
MariaDB [(none)]> CREATE DATABASE wgleeDB;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> CREATE USER  'wglee'@'localhost' IDENTIFIED BY '[패스워드]';
Query OK, 0 rows affected (0.00 sec)
```



## MariaDB-Server - Grant privileges to user

원격에서 접근할 사용자 wglee에 대한 권한을 설정한다.\
접근 허용할 데이터베이스, 원격지 주소 등을 허용 범위에 따라 다음과 같이 설정할 수 있다.

```
## No1. wglee 계정이 wgleeDB 데이터베이스에 원격지IP에서 접근 가능하도록 허용
MariaDB [(none)]> GRANT ALL ON wgleeDB.* to 'wglee'@'공인IP' IDENTIFIED BY '[패스워드]' WITH GRANT OPTIO
N;
Query OK, 0 rows affected 

## No2. wglee 계정이 wgleeDB 데이터베이스에 모든 원격지 IP에서 접근 가능하도록 허용
MariaDB [(none)]> GRANT ALL ON wgleeDB.* to 'wglee'@'%' IDENTIFIED BY '[패스워드]' WITH GRANT OPTION;

## No3. wglee 계정이 모든 데이터베이스에 .0/24 대역의 원격지 IP에서 접근 가능하도록 허용
MariaDB [(none)]> GRANT ALL ON *.* to 'wglee'@'공인IP.%' IDENTIFIED BY '[패스워드]' WITH GRANT OPTION;
```

마지막에는 flush privileges를 해서 변경 사항을 반영한다.

```
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> EXIT;
```



## MariaDB-Client - Test Connection

client인 web 서버에서 wglee 계정으로 접속을 테스트 한다.\
이때 client와 server 사이에 방화벽이 허용되어 있어야 한다.\
(지금은 mariadb-server가 3306으로 돌고 있기 때문에 server에서 inbound 3306 로 허용해줌)\
`-h` 옵션으로 wglee-db 서버의 공인 아이피를 host로 지정한다.\
앞서 설정한 wglee 계정의 패스워드를 입력하면 원격지의 MariaDB에 접근한 것을 볼 수 있다.

```
[root@wglee-web ~]# mysql -u wglee -h DB서버공인IP -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 10
Server version: 5.5.68-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| wgleeDB            |
+--------------------+
2 rows in set (0.001 sec)

MariaDB [(none)]> use wgleeDB
Database changed
```

[https://webdock.io/en/docs/how-guides/database-guides/how-enable-remote-access-your-mariadbmysql-database](https://webdock.io/en/docs/how-guides/database-guides/how-enable-remote-access-your-mariadbmysql-database)
