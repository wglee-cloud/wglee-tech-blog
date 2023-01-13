---
description: 2022. 6. 3. 23:17
---

# anacrontab 사용하여 job 스케줄링 하기

## Intro

리눅스 시스템에서 특정 시간 및 일정 주기로 스크립트를 동작 시키거나 간단한 커맨드를 적용해야 하는 경우가 있다.\
이럴 때 보통은 crontab을 사용해 왔는데, anacrontab 이라는 것을 새롭게 알게 되어 정리해 보고자 한다.

crontab 은 지정한 시간에만 job을 수행을 시도한다. 즉, 만약 해당 시간에 서버 전원이 꺼져 있어서 수행을 못하면 skip 한다.\
하지만anacrontab 으로 설정한 job은 서버가 job 실행이 가능한 상태가 되면 다시 수행 된다.&#x20;



## anacrontab 설정하기

anacrontab 은 /etc/anacrontab 파일에서 설정 한다.

* **Period in days**\
  ****job을 반복할 일별 주기를 의미한다. @daily 로 하면 1로 설정하는 것과 동일한데, 이는 매일 실행한다는 의미이다.\
  @weekly로 하면 7일에 해당하며, 주 단위로 스케줄링하게 된다.
* **Delay in minutes**\
  ****crond 데몬이 해당 job을 시작하기 전에 대기할 시간을 의미한다.\
  만약 0 으로 설정하면 delay 없이 바로 수행한다.\
  만약 10으로 설정하면, 서버가 스크립트 실행 가능한 상태가 된 후 10분 대기하였다가 실패 했던 job을 실행시킨다.
* **Job identifier**\
  ****log 메세지에서 job을 식별할 고유한 이름이다.
* **Command**\
  ****실행시킬 명령어에 해당한다.

아래는 anacrontab 파일을 처음 열면 설정되어 있는 내용이다.

```bash
# /etc/anacrontab: configuration file for anacron

# See anacron(8) and anacrontab(5) for details.

SHELL=/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
# the maximal random delay added to the base delay of the jobs
RANDOM_DELAY=45
# the jobs will be started during the following hours only
START_HOURS_RANGE=3-22

#period in days   delay in minutes   job-identifier   command
1       5       cron.daily              nice run-parts /etc/cron.daily
7       25      cron.weekly             nice run-parts /etc/cron.weekly
@monthly 45     cron.monthly            nice run-parts /etc/cron.monthly
~
```



위 내용에서 **run-parts** 명령어가 무엇인지 좀 궁금했다.\
run-parts 명령어는 지정한 이렉터리 안에 있는 실행 권한이 있는 스크립트를 모두 실행시킨다.\
특정 디렉터리 안의 run-parts 로 실행 가능한 파일 리스트를 조회하기 위해서는 다음과 같이 하면 된다.

```shell-session
# run-parts --test [directory]

[root@server-a ~]# run-parts --test /etc/cron.hourly/
/etc/cron.hourly/0anacron
```



이제 다시 위에서 확인한 anacrontab 설정을 보면 이제는 이해가 된다.\
/etc/cron.daily, /etc/cron.weekly, /etc/cron.monthly 디렉터리 하위의 파일을 해당 주기로 실행 하도록 run-parts 명령어가anacrontab 파일에 설정되어 있다.\
/etc/cron.hourly 디렉터리의 경우에는 /etc/cron.d/0hourly 파일에 설정되어 있다.

```shell-session
[root@server-a ~]# cat /etc/cron.d/0hourly
# Run the hourly jobs
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
01 * * * * root run-parts /etc/cron.hourly
```



## 참고 문서&#x20;

{% embed url="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/deployment_guide/ch-automating_system_tasks" %}
