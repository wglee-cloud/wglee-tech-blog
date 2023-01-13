---
description: 2022. 8. 13. 19:49
---

# SNAT / DNAT 개념과 iptables 에 NAT 룰 적용해보기

## Intro

SNAT / DNAT 개념을 한번 정리하고 직접 iptables 로 설정해보자\~!



## NAT?

SNAT / DNAT에 공통적으로 NAT 라는 단어가 포함되어 있다.\
NAT (Network Address Translation)는 IP 패킷 헤더의 ip 주소를 변경하는 것을 의미한다.\
사설IP와 공인IP를 매핑하여 사설 네트워크에 있는 어플리케이션이 외부 인터넷과 통신할 수 있도록 많이 사용한다.

NAT는 IP를 참조하는 L3 계층에 해당한다.\
다음과 같이 상황과 용도에 맞게 NAT 설정을 할 수 있으나 나는 iptables에 적용하는 것을 테스트 해 보겠다!

\
**(네트워크 물리 장비)** - router 에 NAT 설정\
**(linux)** - iptables 로 NAT 설정



## NAT의 목적

1. **공인 IP 절약**\
   ****IPv4 공인IP는 한정되어 있다. 때문에 회사/집/학교 등에 있는 모든 PC에 공인 IP를 부여하면 가격도 비싸고 빠르게 고갈될 위험이 높다.\
   회사 내부 PC 들에는 사설 IP를 부여하고, 인터넷 접근 시 하나의 특정 공인 IP로 나가도록 NAT 설정하면 공인 IP를 절약할 수 있다.
2. **보안**\
   ****외부 인터넷에서 해킹등의 공격 받는 것을 방지하기 위해 보안이 필요한 단말기들을 사설로 운영하여 인터넷에서 직접적으로 접근하지 못하도록 한다.



## iptables NAT 설정해보기

테스트 환경은 다음과 같다.\
3개의 가상서버 중 wglee-nat 서버에만 공인 아이피가 있다.\
wglee-nat 서버의 공인아이피로 wglee-server-001,002 웹서버의 index.html 에 접근하도록 nat 설정한다.

<figure><img src="https://blog.kakaocdn.net/dn/P9jkA/btrJItpUmt6/RzFEgiG27IivymzlL6Yw6K/img.png" alt=""><figcaption></figcaption></figure>

먼저, iptables에는 nat 테이블이라는 것이 있다.\
패킷의 header에 있는 source ip, dest ip를 변경할 수 있게 한다.\
nat 테이블에는 다음과 같은 체인이 있다.

* _**PREROUTING**_** ** : DNAT (Destination NAT). header의 dest ip 를 변경한다. 외부 -> 내부
* _**POSTROUTING**_** ** : SNAT (Source NAT) : header의 source ip 를 변경. 내부 -> 외부



#### 설정 1 ) nat서버의 8080 포트 <-> wglee-server-001 8080 포트

```shell
# PREROUTING (DNAT) : 8080 포트로 들어오는 패킷의 dest ip를 192.168.1.6:8080로 변경한다. 
iptables -A PREROUTING -t nat -j DNAT -p tcp --dport 8080 --to-destination 192.168.1.6:8080

# POSTROUTING (SNAT) : 192.168.1.6의 8080 포트를 목적지로 오는 패킷의 source ip를 192.168.1.23로 변경한다. 
iptables -A POSTROUTING -t nat -j SNAT  -p tcp -d 192.168.1.6 --to-source 192.168.1.23 --dport 8080
```



#### 설정 2) nat서버의 80포트 <-> wglee-server-002 8081 포트

```shell
# PREROUTING (DNAT) : 80 포트로 들어오는 패킷의 dest ip를 192.168.1.28:8081로 변경한다. 
iptables -A PREROUTING -t nat -j DNAT -p tcp --dport 80 --to-destination 192.168.1.28:8081

# POSTROUTING (SNAT) : 192.168.1.28의 8081를 목적지로 오는 패킷의 source ip를 192.168.1.23로 변경한다. 
iptables -A POSTROUTING -t nat -j SNAT  -p tcp -d 192.168.1.28 --to-source 192.168.1.23 --dport 8081
```

이제 wglee-nat 서버의 공인 아이피로 각 웹서버에 접근할 수 있다.

<figure><img src="https://blog.kakaocdn.net/dn/bdcKRu/btrJItQX12U/5ycJ95yn9GMqwfWTVWjMt1/img.png" alt=""><figcaption></figcaption></figure>

iptables의 nat 테이블이 다음과 같이 설정되었다.

```shell-session
[root@wglee-nat ~]# iptables -nL -t nat
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:192.168.1.6:8080
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:192.168.1.28:8081

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
SNAT       tcp  --  0.0.0.0/0            192.168.1.6          tcp dpt:8080 to:192.168.1.23
SNAT       tcp  --  0.0.0.0/0            192.168.1.28         tcp dpt:8081 to:192.168.1.23
```



## tcpdump로 패킷 통신하는 것 확인해보기

nat 서버에서 tcpdump로 tcp 8080 port로 들어오는 트래픽을 확인한다.\
nat서버의 사설 IP 192.168.1.23가 server-001의 사설 192.168.1.6로 seq를 보내고\
반대로 192.168.1.6에서 192.168.1.23으로 ack를 반환하는 것을 볼 수 있다.

```shell-session
[root@wglee-nat ~]# tcpdump -i eth0 -p tcp port 8080 -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
01:07:25.480047 IP 211.63.40.134.53737 > 192.168.1.23.webcache: Flags [F.], seq 1345693547, ack 1392297251, win 511, length 0
01:07:25.480088 IP 192.168.1.23.53737 > 192.168.1.6.webcache: Flags [F.], seq 1345693547, ack 1392297251, win 511, length 0
01:07:25.480282 IP 192.168.1.6.webcache > 192.168.1.23.53737: Flags [.], ack 1, win 230, length 0
01:07:25.480304 IP 192.168.1.23.webcache > 211.63.40.134.53737: Flags [.], ack 1, win 230, length 0
01:07:25.480904 IP 211.63.40.134.53744 > 192.168.1.23.webcache: Flags [S], seq 1786556894, win 64240, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
01:07:25.480922 IP 192.168.1.23.53744 > 192.168.1.6.webcache: Flags [S], seq 1786556894, win 64240, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
01:07:25.481064 IP 192.168.1.6.webcache > 192.168.1.23.53744: Flags [S.], seq 73937948, ack 1786556895, win 28200, options [mss 1410,nop,nop,sackOK,nop,wscale 7], length 0
01:07:25.481072 IP 192.168.1.23.webcache > 211.63.40.134.53744: Flags [S.], seq 73937948, ack 1786556895, win 28200, options [mss 1410,nop,nop,sackOK,nop,wscale 7], length 0
01:07:25.482942 IP 211.63.40.134.53738 > 192.168.1.23.webcache: Flags [P.], seq 3212534178:3212534743, ack 1365155688, win 512, length 565: HTTP: GET / HTTP/1.1
01:07:25.482951 IP 192.168.1.23.53738 > 192.168.1.6.webcache: Flags [P.], seq 3212534178:3212534743, ack 1365155688, win 512, length 565: HTTP: GET / HTTP/1.1
01:07:25.483098 IP 192.168.1.6.webcache > 192.168.1.23.53738: Flags [.], ack 565, win 230, length 0
01:07:25.483105 IP 192.168.1.23.webcache > 211.63.40.134.53738: Flags [.], ack 565, win 230, length 0
01:07:25.483307 IP 192.168.1.6.webcache > 192.168.1.23.53738: Flags [P.], seq 1:180, ack 565, win 230, length 179: HTTP: HTTP/1.1 304 Not Modified
01:07:25.483312 IP 192.168.1.23.webcache > 211.63.40.134.53738: Flags [P.], seq 1:180, ack 565, win 230, length 179: HTTP: HTTP/1.1 304 Not Modified
01:07:25.483733 IP 211.63.40.134.53744 > 192.168.1.23.webcache: Flags [.], ack 1, win 512, length 0
01:07:25.483738 IP 192.168.1.23.53744 > 192.168.1.6.webcache: Flags [.], ack 1, win 512, length 0
01:07:25.538287 IP 211.63.40.134.53738 > 192.168.1.23.webcache: Flags [.], ack 180, win 511, length 0
01:07:25.538296 IP 192.168.1.23.53738 > 192.168.1.6.webcache: Flags [.], ack 180, win 511, length 0
```



## 참고 사이트

{% embed url="https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=alice_k106&logNo=221305928714" %}

{% embed url="https://zigispace.net/1121" %}
