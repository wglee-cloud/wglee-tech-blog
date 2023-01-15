---
description: 2022. 9. 5. 09:04
---

# Ansible Semaphore 설치 (CentOS 7)

## Intro

오늘은 Ansible Semaphore 에 대해 알아보도록 한다.\
Ansible Semaphore 는 Ansible Playbook을 UI 적으로 관리할 수 있는 인터페이스이다.\
Web UI로 쉽게 다룰 수 있어 cli 사용이 어려운 비엔지니어도 사용이 편리하다.\
또한 역할 기반으로 엑세스를 제어하여 보안을 높이고,\
각 task별로 cron을 등록해 스케줄링할 수 있다.&#x20;

* Semaphore Home : [https://ansible-semaphore.com/](https://ansible-semaphore.com/)
* Semaphore Docs : [https://docs.ansible-semaphore.com/](https://docs.ansible-semaphore.com/)



## 설치 환경

* Centos 7.6
* Ansible Semaphore 최신 버전 ( v2.8.49 )
* Ansible 2.9.27



## 설치 과정

Ansible Semphore를 package 방식으로 설치할 경우 Python, Ansible, Git 이 사전에 설치되어 있어야 한다.

## MariaDB 설치

semphore 를 설치하는 서버에 MariaDB를 설치한다.

```shell-session
[root@wglee-deploy ~]# yum install -y https://dev.mysql.com/get/mysql80-community-release-el7-7.noarch.rpm
[root@wglee-deploy ~]# yum repolist enabled | grep "mysql.*"
mysql-connectors-community/x86_64 MySQL Connectors Community                 199
mysql-tools-community/x86_64      MySQL Tools Community                       92
mysql80-community/x86_64          MySQL 8.0 Community Server                 346

[root@wglee-deploy ~]# yum install -y mysql-server
...
Installed:
  mysql-community-client.x86_64 0:8.0.30-1.el7            mysql-community-libs.x86_64 0:8.0.30-1.el7            mysql-community-libs-compat.x86_64 0:8.0.30-1.el7
  mysql-community-server.x86_64 0:8.0.30-1.el7

Dependency Installed:
  libaio.x86_64 0:0.3.109-13.el7                             mysql-community-client-plugins.x86_64 0:8.0.30-1.el7       mysql-community-common.x86_64 0:8.0.30-1.el7
  mysql-community-icu-data-files.x86_64 0:8.0.30-1.el7

Replaced:
  mariadb.x86_64 1:5.5.68-1.el7                                                      mariadb-libs.x86_64 1:5.5.68-1.el7

Complete!

[root@wglee-deploy ~]# systemctl enable mysqld && systemctl start mysqld && systemctl status mysqld
```

mariaDB의 초기 패스워드를 확인하여 root 로 접근한 다음, 보안성 높을 패스워드로 변경한다.

```shell-session
[root@wglee-deploy ~]# grep 'temporary password' /var/log/mysqld.log
2022-09-01T08:11:51.426277Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: [임시패스워드]

[root@wglee-deploy ~]# mysql -u root -p
Enter password: [임시패스워드 입력]
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 12
Server version: 8.0.30

mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '어려운패스워드';
Query OK, 0 rows affected (0.04 sec)
```



## semaphore 설치

[https://docs.ansible-semaphore.com/administration-guide/installation#package-manager](https://docs.ansible-semaphore.com/administration-guide/installation#package-manager)\
위의 공식 문서를 기반으로 설치하였다.

semaphore 패키지를 설치한다.

```shell-session
[root@wglee-deploy ~]# wget https://github.com/ansible-semaphore/semaphore/releases/download/v2.8.49/semaphore_2.8.49_linux_amd64.rpm
[root@wglee-deploy ~]# sudo yum install semaphore_2.8.49_linux_amd64.rpm
```

semaphore setup 을 수행한다.

```shell-session
[root@wglee-deploy ~]# semaphore setup

Hello! You will now be guided through a setup to:

1. Set up configuration for a MySQL/MariaDB database
2. Set up a path for your playbooks (auto-created)
3. Run database Migrations
4. Set up initial semaphore user & password

What database to use:
   1 - MySQL
   2 - BoltDB
   3 - PostgreSQL
 (default 1): [ 엔터 ]

db Hostname (default 127.0.0.1:3306): [ 엔터 ]

db User (default root): [ 엔터 ]

db Password: [변경한 MariaDB 패스워드]

db Name (default semaphore): [ 엔터 ] 

Playbook path (default /tmp/semaphore): /opt/semaphore

Web root URL (optional, see https://github.com/ansible-semaphore/semaphore/wiki/Web-root-URL): [ 엔터 ]

Enable email alerts? (yes/no) (default no): [ 엔터 ]

Enable telegram alerts? (yes/no) (default no): [ 엔터 ]

Enable LDAP authentication? (yes/no) (default no): [ 엔터 ]


Generated configuration:
 {
  ...생략...
 }

Is this correct? (yes/no) (default yes): yes

Config output directory (default /root): yes

Running: mkdir -p yes..
Configuration written to yes/config.json..
 Pinging db..
Running db Migrations..
Executing migration v0.0.0 (at 2022-09-01 08:16:45.395715702 +0000 UTC m=+58.602036709)...
Creating migrations table
...생략...
 > Username: admin
 > Email: admin@cafe24.com
WARN[0122] no rows in result set                         level=Warn
 > Your name: wglee
 > Password: [패스워드]

 You are all setup wglee!
 Re-launch this program pointing to the configuration file

./semaphore server --config yes/config.json

 To run as daemon:

nohup ./semaphore server --config yes/config.json &

 You can login with admin@cafe24.com or admin.
```

setup 을 완료하면 mariaDB에 semaphore 데이터베이스가 생성된다.

```shell-session
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| semaphore          |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql> use semaphore;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+----------------------+
| Tables_in_semaphore  |
+----------------------+
| access_key           |
| event                |
| event_backup_5784568 |
| migrations           |
| project              |
| project__environment |
| project__inventory   |
| project__repository  |
| project__schedule    |
| project__template    |
| project__user        |
| project__view        |
| session              |
| task                 |
| task__output         |
| user                 |
| user__token          |
+----------------------+
17 rows in set (0.00 sec)
```

semaphore 서버가 setup 으로 인해 생성된 config.json 을 사용하도록 한다.

```shell-session
[root@wglee-deploy ~]# semaphore server --config ~/config.json
MySQL root@127.0.0.1:3306 semaphore
Tmp Path (projects home) /opt/semaphore
Semaphore v2.8.49
Interface
Port :3000
Server is running
```



## 설치 완료

semaphore가 설치된 가상서버의 공인 IP에 semaphore가 동작중인 3000번 포트로 semaphore web에 접근할 수 있다.

<figure><img src="https://blog.kakaocdn.net/dn/bh4fdx/btrLiCtmLWv/Zik2HD8DsBY0kKvMNVd7K0/img.png" alt=""><figcaption></figcaption></figure>



