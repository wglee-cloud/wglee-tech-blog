---
description: 2022. 7. 31. 14:25
---

# Network bandwidth의 개념과 Throughput 과의 차이

<figure><img src="https://blog.kakaocdn.net/dn/d4YQcr/btrJfqodENo/8wiL4zTLKMpIW4F6dPme91/img.png" alt=""><figcaption><p>Image from https://forum.huawei.com/enterprise/en/network-latency-vs-bandwidth-vs-throughput/thread/789837-861</p></figcaption></figure>

## Network Bandwidth란?

Network Bandwitdh(대역폭)은 네트워크에서 특정 시간 내에 전송 될 수 있는 데이터의 최대 용량을 의미한다.\
네트워크의 대역폭이 높을수록, 한번에 더 많은 데이터가 전송 될 수 있다.&#x20;

주의할 점은 대역폭이 네트워크의 실질적인 성능을 나타내지는 않는다는 것이다.\
Bandwidth를 'Data speed(속도)' 및 'Data throughput(처리량)'과 혼동해서는 안된다.



## Network Bandwidth 단위

대역폭은 네트워크 회선을 통해 초당 전송되는 데이터를 bits 단위로 측정한다.\
대역폭을 나타낼 때 사용하는 단위들은 다음과 같다.

```
bps ( Bits per Sec)
Mbps ( Megabits per Sec )
Gbps ( Gigabits per Sec )
```



## Bandwidth와 Throughput의 차이

Throughput은 네트워크에서 초당 실제로 처리되는 패킷의 양을 나타내는 실용적인 지표이다.\
반면 Bandwidth은 네트워크에서 잠재적으로 동시에 전송될 수 있는 데이터의 최대치를 나타낸다.

예를 들어 파이프에 물이 흐른다고 가정해서 나는 다음과 같이 이해했다.\
파이프를 100% 사용할 때 초당 흐르는 물의 양이 대역폭 값이다. 파이프 자체가 클 수록 더 많은 물이 흐를 수 있다.\
파이프의 폭 외에 외부적 요인에 의해 80%만 실질적으로 사용한다면, 이때의 값이 처리량이다.



## Bandwidth와 Speed 의 차이

Network speed는 시작 지점에서 도착 지점까지 데이터가 처리되는 '속도'의 개념이다. Network speed는 download speed, upload speed, lantency(지연시간)의 영향을 받는다.\
때문에 데이터의 양에 대한 개념인 bandwitdh와는 차이가 있다.



## 참고 자료

{% embed url="https://www.geeksforgeeks.org/introduction-to-bandwidth/" %}

{% embed url="https://www.techtarget.com/searchnetworking/definition/bandwidth" %}

{% embed url="https://www.indeed.com/career-advice/career-development/throughput-vs-bandwidth" %}
