---
description: 2022. 7. 11. 21:56
---

# HAProxy Configuration

## Intro

HAproxy 설정 및 동작 방식을 알아본다.\
HAProxy는 TCP/HTTP 트래픽을 소프트웨어적으로 로드밸런싱 할 수 있는 오픈소스이다.

<figure><img src="https://blog.kakaocdn.net/dn/bBlN24/btrJBl0EwSy/SnEKlYQiXVFL3Icb67nF0K/img.png" alt=""><figcaption></figcaption></figure>

나는 오픈스택 HA 구조를 구축하면서 vip 에 대한 요청을 여러 백엔드 서버들로 분산하도록 하기 위해 HAProxy를 사용했다.\
한번은 오픈스택의 각 유저가 mariadb의 DB에 정상적으로 접근하지 못한다는 connection aborted 에러가 mysql.err에 발생했다.

```
root@wglee-controller-001:/var/log/mysql# tail -f error.log
2022-07-05 22:09:34 30861 [Warning] Aborted connection 30861 to db: 'placement' user: 'placement' host: 'wglee-controller-001' (Got an error reading communication packets)
2022-07-05 22:10:07 30862 [Warning] Aborted connection 30862 to db: 'neutron' user: 'neutron' host: 'wglee-controller-001' (Got an error reading communication packets)
2022-07-05 22:11:09 30863 [Warning] Aborted connection 30863 to db: 'neutron' user: 'neutron' host: 'wglee-controller-001' (Got an error reading communication packets)
2022-07-05 22:12:10 30864 [Warning] Aborted connection 30864 to db: 'keystone' user: 'keystone' host: 'wglee-controller-001' (Got an error reading communication packets)
```

도대체 뭔가 하면서 mariadb의 timout 수치만 열심히 튜닝했는데\
결국 haproxy에서 vip로 받은 요청을 backend server로 넘길 때의 타임아웃에 걸리는 것이 이슈로 보였다.^^...

그 후로 HAProxy의 설정 파일 구성와 옵션을 잘 알아야겠다는 생각이 들어서 한번 정리해 본다.



## HAProxy Configuration

Haproxy 설정 파일은 크게 다음 섹션들로 이루어져 있다.

```
global
    # global settings here

defaults
    # defaults here

frontend
    # a frontend that accepts requests from clients

backend

    # servers that fulfill the requests
```



### 1. global

프로세스 전반적으로 적용되는 보안/성능 튜닝 설정

```
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon
```

**maxconn** : HAProxy가 받아들일 커넥션의 최대치. 로드밸런서의 메모리 부족을 방지한다.\
**log** : 로그를 남길 경로를 지정한다. local0으로 설정해서 syslog 기능을 이용할 수 있다.\
**user / group** : 따로 지정하지 않으면 haproxy는 root 권한으로 동작하기 때문에 미리 계정을 생성하고 지정하도록 한다.



### 2. defaults

설정 파일에서 중복을 제거하기 위해 사용한다.\
default 세팅은 frontend와 backend 섹션에 적용된다. (frontend, backend 에서 오버라이드도 가능)

```
defaults
    timeout connect 10s
    timeout client 30s
    timeout server 30s
    log global
    mode http
    option httplog
    maxconn 3000
```

**timeout \[항목]** : 각 항목에 대한 timout 을 설정한다. "s" 를 명시하면 초단위 설정을 하게 되며, 아무것도 붙이지 않을 경우 기본적으로 milliseconds 로 설정된다.\
**timeout connect** : HAProxy가 백엔드 서버에 TCP 연결이 established 될때까지 대기하는 시간\
**timeout client** : client ↔ haproxy frontend 사이의 연결에 대해 대기하는 시간이다. 이 옵션은 TCP 체크에 한해서 사용 가능하다.\
**timeout server** : haproxy backend에서 server 로 요청을 보내고 대기하는 타임아웃 시간이다.\
**mode** : HAProxy가 어떤 protocol에 대해 Proxy할지를 정의. TCP / HTTP\
**log global** : frontend 에서 global 섹션에\
**maxconn** : 각 frontend에서 수용할 커넥션



### 3. frontend

client가 connect할 ip와 port 등의 설정한다.

```
frontend wglee-openstack-vip
bind 20.20.0.5:80 ssl crt /etc/wglee.pem
mode            http
option          httpclose
option          forwardfor
option          accept-invalid-http-request
reqadd          X-Forwarded-Proto:\ https
default_backend object_storage
```

**bind** : 바인딩하여 listen할 아이피와 포트를 지정한다.\
**ssl, crt 옵션** : HAProxy가 SSL/TLS 처리를 하도록 한다.\
**use\_backend, default\_backend** : 해당 frontend 로 들어온 요청을 처리해달라고 보낼 backend.\
&#x20;  (우선순위 : use\_backend -> (use\_backend가 실패하면) default\_backend)



### 4. backend

요청을 처리할 backend 서버의 그룹\
하위 real server 정보와 각 옵션을 넣는다.

```
backend web_servers
    balance roundrobin
    cookie SERVERUSED insert indirect nocache
    option httpchk HEAD /
    default-server check maxconn 20
    server server1 10.0.1.3:80 cookie server1
    server server2 10.0.1.4:80 cookie server2
```

**balance** : 로드밸런싱할 알고리즘을 지정한다. (roundrobin, leastconn)\
**option httpchk** : HAProxy가 백엔드 서버에 대해 Layer 7 헬스 체크를 하도록 한다. 응답이 없는 서버에는 요청 포워딩하지 않음.\
**default-server** : 이후에 따라오는 server 들에 공통으로 적용되는 디폴트 세팅\
\-> inter : health check 의 interval을 의미한다.\
\-> rise : backend server가 동작 중이라고 여기기 위해 성공적으로 수행되어야 하는 health check의 횟수 ( 기본은 2 )\
\-> fall : backend server가 죽었다고 여기기 위해 실패해야 하는 health check의 횟수. ( 기본은 3 )\
\-> check : backend server에 대해 health check를 활성화 한다.\
server : 벡엔드로 사용할 서버 등록



### 5. listen

listen을 사용하면 frontend와 backend 의 기능을 한번에 사용한다고 보면 됨

```
listen mariadb
  bind wglee-openstack-vip:3306
  mode tcp
  balance leastconn
  timeout client  10800s
  timeout server  10800s
  default-server port 9200 inter 2s downinter 5s rise 3 fall 2 slowstart 60s
  server wglee-controller-001 20.20.0.20:3306 check
  server wglee-controller-002 20.20.0.21:3306 check backup
  server wglee-controller-003 20.20.0.22:3306 check backup
```

위에서 mariadb 에 connection aborted가 발생했다고 했는데 위와 같이 timout client, timeout server를 늘려주고 나서 해결 되었다.\
**port** : port 파라미터를 사용하면 health check 용으로 다른 port를 사용할 수 있다.\
때로 어플리케이션이 동작하는 포트보다 다른 port 를 사용해서 health check를 해야 하는 경우가 있다. 이 경우에 사용한다.\
예를 들어서 mariadb는 3306으로 동작하지만 health check를 3306 포트로 하는 것은 적합하지 않다. ( curl 했을 때 반환되는 HTTP\_CODE가 없음 ) 그래서 대안으로 clustercheck가 xinetd로 동작하는 9200 를 체크한다.&#x20;

```
root@wglee-controller-001:~# curl -o /dev/null -s -w %{http_code} 20.20.0.20:9200 ; echo
200
root@wglee-controller-001:~# curl -o /dev/null -s -w %{http_code} 20.20.0.20:3306 ; echo
000
```

<figure><img src="https://blog.kakaocdn.net/dn/nti1s/btr3XW862ME/JkQZRY8Dp5kwdnu3ydyrv1/img.png" alt=""><figcaption></figcaption></figure>

내 haproxy 설정 기준으로 frontend / backend / listen 에 적용된 옵션을 각각 설명하다 보니까 공통으로 사용할 수 있는 옵션도 여기저기 적어버렸다.\
상황에 맞게 확인해서 잘 사용해야겠다.





**참고링크**

* [https://cbonte.github.io/haproxy-dconv/2.0/configuration.html#port](https://cbonte.github.io/haproxy-dconv/2.0/configuration.html#port)
* [https://www.haproxy.com/blog/exploring-the-haproxy-stats-page/](https://www.haproxy.com/blog/exploring-the-haproxy-stats-page/)
* [https://www.haproxy.com/blog/the-four-essential-sections-of-an-haproxy-configuration/#maxconn-1](https://www.haproxy.com/blog/the-four-essential-sections-of-an-haproxy-configuration/#maxconn-1)
* [http://docs.haproxy.org/2.6/configuration.html#5.2-port](http://docs.haproxy.org/2.6/configuration.html#5.2-port)
