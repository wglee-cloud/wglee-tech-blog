---
description: 2022. 9. 5. 09:06
---

# Ansible - target 서버 정상적으로 올라왔는지 확인하기

## Intro

Ansible Playbook에서 target 서버를 reboot했을 때, 해당 서버가 정상적으로 올라온 상태에서 나머지 task를 실행해야 할 것이다.\
올라온 것을 검증하지 않고 나머지 task를 수행하면 에러가 발생할 수 있다.\
본 게시글에서는 target서버 reboot 후 정상적으로 올라왔는지 확인하는 방법 2가지를 설명한다.\
첫번째는 wait\_for 모듈을 사용하는 것이고, 두번째는 reboot 모듈의 test\_command 파라미터를 사용하는 것이다.

_**직접 사용해보니 reboot 모듈의 test\_command 로 확인하는 것이 task를 줄일 수 있고 훨씬 간편하다.**_&#x20;

<figure><img src="https://blog.kakaocdn.net/dn/bidYro/btrLiqfzMrP/oB4POdCOuzHuc5F4SjB1pk/img.png" alt=""><figcaption></figcaption></figure>

## 1. wait\_for / connection

wailt\_for 모듈을 사용하면 특정 포트 상태를 확인하여 어플리케이션이 avaliable 한지 확인하는데 유용하다.\
또한 `connection` 지시자를 사용해 master 서버에서 target 서버로 connection 테스트를 할 수도 있다.\
특정 문자열이 파일에서 확인 될 때까지 대기하는데 쓰이기도 한다.

아래 playbook에서 첫번째 task는 웹서버를 reboot 하고 10초 대기한다.\
두번째 task 에서 _port 22번이 open 상태인지 최대 60초 까지 대기한다._

### **wait\_for 모듈**

확인할 port 상태는 `state` 값으로 명시한다.\
state 값에 따라 확인하는 포트 상태는 다음과 같다. ( 대상이 파일일 경우 약간 다른데 그건 공식 문서 참고.. )\
\-> started : 포트가 open 상태인지 ( default )\
\-> stopped : 포트가 closed 상태인지\
\-> drained : 포트에 active connection 없는지

`search_regex` : 파일에 해당 문자열 포함 여부를 확인하는데 쓰이기도 하지만 socket 통신 여부를 확인할 수도 있다.\
22번 포트로 OpenSSH이 사용되고 있는지 확인한다.

### **connection plugin**

`connection: local`은 master 서버에서 target 서버로의 connection 을 수행한다.\
이는 원격지의 포트에 대해 nc나 telnet 과 비슷한 동작을 한다고 한다.\
참고로 local 이외에도 connection 에 대한 다양한 플러그인이 존재한다.

```yaml
[root@wglee-deploy training-wglee-playbook]# cat roles/manage-case2/tasks/main.yml
---
- name: reboot webserver
  reboot:
    msg: "Reboot initiated by Ansible"
    post_reboot_delay: 10


- name: Confirm if server is rebooted
  wait_for:
    port: 22
    host: '{{ inventory_hostname }}'
    search_regex: OpenSSH
#    delay: 10   ## post_reboot_delay에서 10초 지정해서 wait_for에서는 주석 처리함 
    timeout: 60
  connection: local
```



## 2. Reboot의 test\_command 파라미터

`test_command` : reboot된 원격 서버에 특정 명령어를 수행한다. 시스템이 다음 task를 수행할 수 있는 상태인지 즉, 정상적으로 올라왔는지 파악할 수 있다.\
test\_command의 반환값을 받기까지 대기하는 "reboot\_timeout" 파라미터도 함께 설정한다.\
`reboot_timeout`은 원격 서버 reboot, test\_command 에 대해 각각 적용된다.

```yaml
[root@wglee-deploy training-wglee-playbook]# cat roles/manage-case2/tasks/main.yml
---
- name: reboot webserver
  reboot:
    msg: "Reboot initiated by Ansible"
    post_reboot_delay: 10
    reboot_timeout: 300
    test_command: "whoami"
```



## 참고 문서

{% embed url="https://docs.ansible.com/ansible/latest/collections/ansible/builtin/wait_for_module.htmlhttps://docs.ansible.com/ansible/2.5/plugins/connection.html" %}

{% embed url="https://www.middlewareinventory.com/blog/ansible_wait_for_reboot_to_complete/#The_Ansible_Playbook_with_wait_for_module" %}

{% embed url="https://docs.ansible.com/ansible/latest/collections/ansible/builtin/reboot_module.html#parameter-test_command" %}

