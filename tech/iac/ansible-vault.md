---
description: 2022. 9. 5. 09:06
---

# Ansible Vault로 민감한 데이터 암호화하기

## Intro

Ansible Playbook에 비밀번호나 api key 같은 보안상 민감한 내용이 사용되어야 할 때가 있다.\
평문으로 넘길 경우 보안상 취약하기 때문에 암호화하는 것이 필요하다.\
이때 Ansible Vault로 변수의 암호화를 수행하고, playbook 에서 이를 참조하여 보안을 강화할 수 있다.

<figure><img src="https://blog.kakaocdn.net/dn/DmbcC/btrLlFiCR5Q/GklD6PVs855gViSrLpZqG0/img.png" alt=""><figcaption></figcaption></figure>

## 배경

다음과 같이 dbadm 계정을 생성하고 패스워드를 지정하려고 한다.\
이때 계정에 대한 패스워드를 아래와 같이 평문으로 넘기는 것은 보안상 취약하다.

```yaml
[root@wglee-deploy training-wglee-playbook]# cat roles/deploy-db/tasks/main.yml
---
- name: Create user
  user:
    name: dbadm
    password: "dbatest"
    comment: DB Administrator
    uid: 3999
    group: 3999
```

현재 플레이북의 전체 구조는 아래와 같다.

```shell-session
[root@wglee-deploy training-wglee-playbook]# tree
.
├── inventory
│   └── inventory.ini
├── roles
│   ├── deploy-db
│   │   └── tasks
│   │       └── main.yml
└── roles.yml
```



## Vault 암호화된 변수 파일 생성하기

이제 deploy-db role에서 암호화된 password 변수를 사용하도록 할 것이다.\
inventory에서 분류한 db 그룹에 대한 변수 디렉터리인 ./group\_vars/db 디렉터리를 생성한다.\
(변수 파일은 우선순위/사용 대상에 따라 다양한 위치에 생성할 수 있으므로 상황에 맞게 조정한다.)

`ansible-vault create` 명령어로 암호화 파일을 새롭게 생성했다.\
만약 기존의 파일을 암호화 하고 싶다면 `ansible-vault encrypt [파일이름]` 명령어를 사용한다.

```shell-session
[root@wglee-deploy training-wglee-playbook]# ansible-vault create group_vars/db/vault.yml
New Vault password:
Confirm New Vault password:
```

위와 같이 vault 패스워드를 입력하면 편집할 수 있는 파일이 열린다.\
변수 파일과 동일하게 key:value 값을 입력하고 qw! 로 저장하면 자동으로 암호화된다.

```shell-session
password: dbatest
```

생성된 vault.yml을 한면 cat으로 본다. 암호화 된 것을 확인할 수 있다.

```shell-session
[root@wglee-deploy training-wglee-playbook]# cat group_vars/db/vault.yml
$ANSIBLE_VAULT;1.1;AES256
...생략 ...
26262393938373365656663373437316463
```



## 암호화 변수 파일 참조하기

이제 내가 사용하는 main playbook인 site.yml 에서 deploy-db role에서 해당 변수 파일을 참조하도록 추가했다.

```yaml
[root@wglee-deploy training-wglee-playbook]# cat site.yml
---
- name: deploy DB
  hosts: db
  remote_user: root
  become: true
  vars_files:
    - ./group_vars/db/vault.yml
  roles:
    - deploy-db
  tags: deploy-db
```



## Playbook 에서 vault 변수 사용하기

변수 파일에서 지정한 key 값으로 암호화된 변수를 가져온다. 이때 `password_hash('sha512')` 부분을 넣어야 playbook 에서 해당 변수가 암호화 된 것을 제대로 인식한다.

```yaml
[root@wglee-deploy training-wglee-playbook]# cat roles/deploy-db/tasks/main.yml
---
- name: Create user
  user:
    name: dbadm
    password: "{{ password | password_hash('sha512') }}"
    comment: DB Administrator
    uid: 3999
    group: 3999
```



## 암호화된 변수 사용하는 playbook 실행하기

해당 playbook은 vault 변수를 사용하기 때문에 패스워드 인증이 필요하다.\
`--ask-vault-pass` 옵션을 사용해서 처음에 `ansible-vault create` 할 때 지정했던 패스워드를 입력한다.\
만약 각각 다른 패스워드로 설정된 다수의 vault 파일을 참조할 경우, `--vault-id` 옵션을 사용하여 인증한다.

```
ansible-playbook -i inventory/inventory.ini roles.yml -t deploy-db --ask-vault-pass
```

vault 패스워드 인증하는 과정이 없으면 플레이북은 다음과 같이 vault secret을 찾지 못한다.

```
[root@wglee-deploy training-wglee-playbook]# ansible-playbook -i inventory/inventory.ini site.yml -t deploy-db
ERROR! Attempting to decrypt but no vault secrets found
```



## 배포된 내용 검증

target 서버에서 dbatest 패스워드로 user switching 이 가능한지 테스트 해 본다.\
정상적으로 되었다.

```shell-session
[centos@wglee-semaphore-db ~]$ su dbadm
Password:
bash-4.2$
```

/etc/shadow 파일에도 패스워드가 encrypt 되어서 저장 되었다.\
만약 vault 과정 없이 playbook 에서 평문으로 지정했으면 /etc/shadow에도 그냥 dbatest로 들어간다.

```shell-session
[root@wglee-semaphore-db ~]# tail -n 1 /etc/shadow
dbadm:$6$c8/PvnrctiVC.l.t$v8RvbX28kvcBHrhO5X3IuilEsCRnnqfBQuXTcxU0EJkSbuPPojjtjMDKlZ5tm4n0hFzd7GXTrHbKZkIrkKMqw0:19239:0:99999:7:::
```



## 추가

본 블로그에서는 `--vault-id`를 이용한 vault multi password 사용에 대해서는 기술하지 않았다.\
vault 파일 생성할 때 vault-id를 지정해서 구분하는 방법인 듯 하다.\
필요시 참고 ) [https://nayoungs.tistory.com/entry/Ansible-Ansible-Vault](https://nayoungs.tistory.com/entry/Ansible-Ansible-Vault)



## 참고 문서

{% embed url="https://docs.ansible.com/ansible/latest/user_guide/vault.html" %}

{% embed url="https://watch-n-learn.tistory.com/83?category=841096" %}
