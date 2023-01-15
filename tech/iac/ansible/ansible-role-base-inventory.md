---
description: 2022. 9. 5. 09:06
---

# ( 메모 ) Ansible role base 구조 및 inventory 그룹핑 방법

<figure><img src="https://blog.kakaocdn.net/dn/cGFcRs/btrLh9yqehA/fxKQopBH6mUnUtc7QkYx6k/img.png" alt=""><figcaption></figcaption></figure>

## Intro

이번 게시글은 ansible playbook 구조에 대해 러프하게 보려고 한다.\
목적은 세부적으로 모든 것을 테스트 하는 것은 아니고, 기존에 playbook을 유지보수 하면서 어렴풋이 알았던 것을 "아\~ 이게 이런 용도 였구나!" 라고 한번 짚어보기 위함이다.

( 체계적이지 못한 내용이라 다른 분들께 도움이 못 될 것 같습니다.. )



## Sample Directory Layout

아래는 ansible 공식문서에서 찾을 수 있는 Directory Layout 이다.\
이중에서 오늘 새롭게 알게 된 것은 handler 이다.\
task에 쓰인 모듈의 changed 값에 따라 trigger 되는 함수 개념이라고 한다.\
예를 들어서 특정 서비스의 status 확인을 했는데 active 상태가 아니면 `notify:`를 사용해 handler를 호출 할 수 있다.

```yaml
production                # inventory file for production servers
staging                   # inventory file for staging environment

group_vars/
   group1.yml             # here we assign variables to particular groups
   group2.yml
host_vars/
   hostname1.yml          # here we assign variables to particular systems
   hostname2.yml

library/                  # if any custom modules, put them here (optional)
module_utils/             # if any custom module_utils to support modules, put them here (optional)
filter_plugins/           # if any custom filter plugins, put them here (optional)

site.yml                  # master playbook
webservers.yml            # playbook for webserver tier
dbservers.yml             # playbook for dbserver tier

roles/
    common/               # this hierarchy represents a "role"
        tasks/            #
            main.yml      #  <-- tasks file can include smaller files if warranted
        handlers/         #
            main.yml      #  <-- handlers file
        templates/        #  <-- files for use with the template resource
            ntp.conf.j2   #  <------- templates end in .j2
        files/            #
            bar.txt       #  <-- files for use with the copy resource
            foo.sh        #  <-- script files for use with the script resource
        vars/             #
            main.yml      #  <-- variables associated with this role
        defaults/         #
            main.yml      #  <-- default lower priority variables for this role
        meta/             #
            main.yml      #  <-- role dependencies
        library/          # roles can also include custom modules
        module_utils/     # roles can also include custom module_utils
        lookup_plugins/   # or other types of plugins, like lookup in this case

    webtier/              # same kind of structure as "common" was above, done for the webtier role
    monitoring/           # ""
    fooapp/               # ""
```



## Inventory - Host Grouping&#x20;

static inventory의 경우 아래와 같이 `children`을 섹션 뒤쪽에 붙여서 기존의 그룹을 재편성할 수 있다.

```yaml
# file: inventory.ini

[controller]
controller-01
controller-02

[compute]
compute-01
compute-02

[new-compute]
compute-03
compute-04

[db]
mariadb-01
mariadb-02


### Grouping 
[compute-all:children]
compute
new-compute

[controller-compute:children]
controller
compute
```



ansible-playbook 돌릴 때 특정 그룹을 지정하려면 다음과 같이 하면 된다.\
`-l`은 `--limit`을 의미한다.

```
root@deploy:/etc/ansible# ansible -i inventory.ini -m ping all -l controller
root@deploy:/etc/ansible# ansible -i inventory.ini -m ping all -l compute-all
```



{% embed url="https://docs.ansible.com/ansible/2.8/user_guide/playbooks_best_practices.html" %}
