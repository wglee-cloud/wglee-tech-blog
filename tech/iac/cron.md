# cron 스케줄링

### Intro

지난 게시글에서 Ansible Semaphore 설치한 것에 이어, task 를 등록해 보도록 한다.\
Semaphore를 통해 task에 cron 을 걸어서 스케줄링을 할 수도 있다.

<figure><img src="https://blog.kakaocdn.net/dn/szRtD/btrLufqr8bO/IpNAoo1TThMypRnKudgv50/img.png" alt=""><figcaption></figcaption></figure>

#### **프로젝트 생성**

프로젝트 생성 후에 대시보드를 확인 할 수 있다.

#### **Environment 생성**

나는 따로 외부에서 변수 지정할 것은 없어서 빈 파일을 생성했다.\


<figure><img src="https://blog.kakaocdn.net/dn/con2if/btrLukyuTrZ/4DnMTO5k3lS7uusVCBdlk0/img.png" alt=""><figcaption></figcaption></figure>

#### **SSH Key 생성**

target에 task를 돌릴 때 사용할 ssh key 등록\
semaphore가 동작하는 master 서버에서 target 서버에 접속하여 플레이북을 실행할 때 사용할 ssh 개인키를 등록한다.\


<figure><img src="https://blog.kakaocdn.net/dn/HDH5h/btrLou93KJR/22oVHhr0IIqt7p62BdsVkk/img.png" alt=""><figcaption></figcaption></figure>

#### **Repository 생성**

github 주소를 url 칸에 입력한다. 나는 SSH 타입으로 git clone을 할 것이다.\
git clone을 할 때 사용할 key 도 지정한다.

만약 repo가 private이라면, 아래와 같이 github의 SSH keys 메뉴에 master 서버의 pub 키를 등록해야 한다.

<figure><img src="https://blog.kakaocdn.net/dn/cWrBHt/btrLk446dax/X7tpTvf6HvKteyPEP1G2uK/img.png" alt=""><figcaption></figcaption></figure>

master서버에 github에 대한 SSH fingerprint를 등록한다. 이 부분에서 github와 connection이 잘 되는지도 확인할 수 있다.

```
[root@wglee-deploy semaphore]# ssh -T git@github.com
The authenticity of host 'github.com (20.200.245.247)' can't be established.
ECDSA key fingerprint is SHA256:p2QAMXNIC1TJYWeIOttrVc98/R1BUFWu3/LiyKgUfQM.
ECDSA key fingerprint is MD5:7b:99:81:1e:4c:91:a5:0d:5a:2e:2e:80:13:3f:24:ca.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'github.com,20.200.245.247' (ECDSA) to the list of known hosts.
Hi WG19! You've successfully authenticated, but GitHub does not provide shell access.
```

#### **Inventory 생성**

host 서버들 정보를 담은 inventory를 생성한다.\


<figure><img src="https://blog.kakaocdn.net/dn/dPyYX2/btrLqWLVqy8/vtB0dyRAjlAulzxvEXNob0/img.png" alt=""><figcaption></figcaption></figure>

#### **Vault Password에 대한 Key 생성**

vault를 사용하는 playbook를 돌릴 것이기 때문에, vault password를 key로 생성한다.\
vault 파일을 생성할 때 입력했던 암호를 Password 부분에 그대로 넣으면 된다.

<figure><img src="https://blog.kakaocdn.net/dn/bRLC3N/btrLkadngNQ/O0tLZA6DsfDEkJzjvGBc71/img.png" alt=""><figcaption></figcaption></figure>

#### **Template 생성**

나는 다음과 같이 4개의 template을 등록하였다.\
메인 task 파일이 site.yml이며, 각각의 tag를 외부 변수로 지정하여 semaphore 상에서 task가 돌때 override 되도록 했다.

**deploy-web**\


<figure><img src="https://blog.kakaocdn.net/dn/uE6qq/btrLu9DYFlL/6LT3cAthgQhSMi87rFFbsk/img.png" alt=""><figcaption></figcaption></figure>

**deploy-db**\


<figure><img src="https://blog.kakaocdn.net/dn/bWr2xQ/btrLqWZ7MTC/Bti3gGoG0y8JlelK98oMeK/img.png" alt=""><figcaption></figcaption></figure>

**manage-case1**\
다음과 같이 task에 대한 cron을 설정할 수도 있다.\
예시의 경우 매일 9시에 manage-case1 task가 동작한다.\
"Cron Condition Repository"를 지정했기 때문에 cronjob이 수행될 때 wglee-training-playbook git repo로 부터 자동으로 clone을 한다.\


<figure><img src="https://blog.kakaocdn.net/dn/AKN9V/btrLkwm1yF9/noBJXZGKx2G12PiWRceKak/img.png" alt=""><figcaption></figcaption></figure>

(참고)\
당연한 건데 미처 생각하지 못했던 부분이라 추가로 기록한다.\
cron을 돌리는 semaphore 서버에서 timezone을 Asia/Seoul 로 해야 의도한 시간에 task가 실행될 것이다.

```
[root@wglee-deploy ~]# timedatectl set-timezone Asia/Seoul
[root@wglee-deploy ~]# timedatectl
      Local time: Tue 2022-09-06 10:03:59 KST
  Universal time: Tue 2022-09-06 01:03:59 UTC
        RTC time: Tue 2022-09-06 01:03:59
       Time zone: Asia/Seoul (KST, +0900)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
      DST active: n/a
```

**manage-case2**\


<figure><img src="https://blog.kakaocdn.net/dn/Gn58m/btrLvDSjtTh/QN1pcyQe3YJslzGz1qWKm1/img.png" alt=""><figcaption></figcaption></figure>

#### **Run Task**

4개의 task가 모두 등록 되었다.\
우측의 "RUN" 버튼을 클릭하면 UI 상에서 playbook 실행 로그를 볼 수 있다.\
