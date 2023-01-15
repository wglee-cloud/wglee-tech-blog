---
description: 2021. 8. 7. 22:04
---

# Ethernet의 역사

## Intro

Ethernet이라는걸 많이 들어보기는 했는데 정확하게 설명하지는 못하는 나를 위해 한번 알아보도록 한다.

<figure><img src="https://blog.kakaocdn.net/dn/qAbMu/btrb1rlJcFa/afwwpakAOporUlzdIXGTYk/img.png" alt=""><figcaption></figcaption></figure>

## **Ethernet**

이더넷은 네트워크를 구성하는 방식 중 가장 보편적으로 사용되고 있는 방식이다. 즉 LAN, MAN, WAN 등을 구성하는 기술인데, 사실은 Ethernet 말고도 Token Ring, FDDI 와 같은 다른 방식도 존재한다.&#x20;

사실 난 Token Ring이나 FDDI 구조는 잘 알지 못하기 때문에, 내가 지금까지 공부했고 앞으로 공부할 LAN 네트워킹 방식이 모두 Ethernet 타입이라고 생각하면 될 것 같다.&#x20;

고유한 MAC(MediaAccess Control) 주소를 기반으로 데이터를 주고 받으며, 허브, 스위치, 리피터 등의 장비를 사용하여 장치를 연결한다.&#x20;



## **CSMA/CD**

CSMA/CD(Carrier-sense multiple access with collision detection) 은 한국어로 하면 '반송파 감지 다중 접속 및 충돌 감지' 방식이다.&#x20;

Ethernet 네트워킹 방식은 CSMA/CD 기반으로 통신한다. Ethernet이 처음 등장 했을때 기본적인 LAN 구성방식은 Bus 형이였다.&#x20;

이는 하나의 회선으로 모든 client가 통신하기 때문에 여러 client가 데이터를 송신하면 충돌이 일어나는 문제가 있었다.

더군다나 이더넷에서 사용하는 통신 회선은 baseband 이기 때문에 한번에 하나의 신호만 송출할 수 있다.&#x20;

이러한 문제를 해결하기 위해 도입된 것이 CSMA/CD 방식이다.

다른 디바이스가 신호를 송출하고 있는지 확인하여 하나의 회선에서 충돌이 일어나지 않도록 한다. (collision detection)

<figure><img src="https://blog.kakaocdn.net/dn/AvAGK/btrb24jf5QK/Z1MhcHvNCeXDxiFwCorSj0/img.png" alt=""><figcaption><p>CSMA/CD</p></figcaption></figure>



## **Ethernet 케이블**

Ethernet에서 사용하는 네트워크 케이블에는 여러개가 있는데 몇가지만 알아보도록 한다.&#x20;

* 10base5 (thinknet) : 최대 500 미터&#x20;
* 10base2(thinnet) : 최대 185 미터
* 10base-T : 10Mbps의 전송속도로 절연된 Twisted 케이블을 사용한다.&#x20;

맨 앞에 붙는 숫자는 전송속도를 의미한다. 즉, 위의 경우에는 10Mbps의 전송 속도로 데이터를 전달한다고 볼 수 있다.

과거에는 10Mbps로 시작했으나 기술의 발전에 따라 100Mbps, 1000Mbps(1Gbps)까지 지원된다.

집에 와서 우리집에서 쓰이는 랜선은 어떤 규격인가 확인해 봤는데 CAT 5 UTP였다.&#x20;

이는 100Mbps전송속도의 UTP 케이블을 의미한다.



이때 base는 baseband를 의미하는데, 이는 요즘에 흔하게 들리는 broadband와 상반되는 개념이다.

이는 물리 케이블에서 한번에 다룰 수 있는 시그널의 개수와 연관이 있다.

baseband는 한번에 오직 하나의 시그널 만을 전달할 수 있으며, broadband는 한번에 여러개의 시그널을 처리한다.



LAN 구성을 더 잘 이해하기 위해 Ethernet에 대해 대략적으로 알아보게 된거 같다.

다음에는 LAN 구성부터 해서 스위치, 라우터 등에 트래픽이 어떻게 흐르는지 확인해 보도록 하겠다.&#x20;

