---
description: 2021. 2. 21. 23:16
---

# Ceph 기본 동작 원리

## Overview

본 글은 [Dynamic Data Placement with Red Hat Ceph Storage](https://www.youtube.com/watch?v=8j1aqsUEPLY\&feature=youtu.be)을 보고 이론적으로 정리한 것으로 추가적인 공부가 필요하다. 이 글의 모든 사진은 글의 이해를 돕기 위해 위의 youtube 영상에서 캡처한 것이다.

<div align="left">

<figure><img src="https://blog.kakaocdn.net/dn/HpGhQ/btqYbJeVN2p/lqpNyA1zZ5plbcWcD0Lx60/img.png" alt=""><figcaption></figcaption></figure>

</div>

## Contents

### _**Basic Architecture**_

<figure><img src="https://blog.kakaocdn.net/dn/ByPjF/btqXY1n71rP/XpmkkHmqXJKOAVhwmMymmK/img.png" alt=""><figcaption></figcaption></figure>

### RADOS (reliable, autonomous, distributed object store)

* ceph의 기반이 되는 cluster로, 실질적으로 데이터가 저장되는 곳이다.
* osd, monitor, manager 등이 올라간다.



### LIBRADOS

* application이 rados에 직접적으로 접속할 수 있도록 허용한다.
* HTTP overhead 방지
* rados cluster와 socket 방식으로 통신.



### RGW

* object storage를 위한 gateway로, S3 및 swift와 호환된다.
* 아래의 그림과 같이, S3 요청이 들어오면 radosgateway는 librados에게 전달한다.

<div align="left">

<figure><img src="https://blog.kakaocdn.net/dn/cRhQAJ/btqX6fk9Bw5/0Lg9mkahyKUCYTTHMlIJE0/img.png" alt=""><figcaption></figcaption></figure>

</div>

### RBD (Rados Block Device)

데이터 분실을 최소화하며 분산된 block storage 구현

<div align="left">

<figure><img src="https://blog.kakaocdn.net/dn/RZU2p/btqYbIG5zId/LPnQDnrGPMkcz0REvQDQOK/img.png" alt=""><figcaption></figcaption></figure>

</div>

위의 사진의 경우, openstack으로 구축된 클라우드 환경에서 vm에서 block storage에 access요청이 일어나면 hypervisor에서 해당 request를 librbd로 전달한다.&#x20;

Openstack 뿐만 아니라 다른 오픈소스들과도 함께 쓸 수 있다.



### CEPHFS

* ceph file storage 제공



## _**OSD (Object Storage Daemon)**_

<div align="left">

<figure><img src="https://blog.kakaocdn.net/dn/QdgNY/btqXXVaPoUi/lg4X9u0xD7tDHdPhoUmwy0/img.png" alt=""><figcaption></figcaption></figure>

</div>

1. 하나의 osd는 하나의 disk에 올라간다.
2. Cluster 내에 10s\~10000s 개의 osd를 올릴 수 있으나 최소 100개는 올리는 것이 성능상 좋다.
3. 저장소에 object 저장을 수행한다.
4. peer for replication, recovery, / rebalancing



## _**Monitor**_

1. cluster와 각종 map의 상태를 유지하는 역할을 한다.
2. 분산 시스템인 만큼 osd의 상태(정상 동작 중인지, 어떤 disk와 matching 중이고 disk 상태는 어떤지) 등에 대해 알아야 한다.
3. monitor는 위의 사항들을 체크하여 요청을 어떻게 보낼지 판단한다.
4. 홀수의 monitor를 두어 쿼럼을 하도록 한다.
5. 데이터 자체를 다루지는 않음.



## _**Manager**_

1. luminous 버전부터 등장한 데몬
2. external monitoring과 management를 위해 추가적인 인터페이스를 제공한다.
3. manager framework interface를 통해 다양한 모듈과의 호환성을 높일 수 있다.
4.  모듈 예시\


    <div align="left">

    <figure><img src="https://blog.kakaocdn.net/dn/tF3dN/btqX320GdcB/ik9aEzwkh55nhYbAsZosDK/img.png" alt=""><figcaption></figcaption></figure>

    </div>

## _**Object Placement with crush**_

### CRUSH : Controlled Replication Under Scalable Hashing

client측에서 해당 object를 이용하기 위해서는 어떤 osd에 접근할지 알아야 한다.

이에 대한 정보를 제공하는 것이 crush 알고리즘이다.

접근 방식 (object storage, block storage, ceph file system등)에 상관없이 crush 알고리즘을 사용하는 것은 동일하다.

### PGs

100개가 넘는 osd에 걸쳐 복제되어 있는 수많은 object를 관리하는것은 힘들다. 따라서 모든 object를 개별적으로 관리하는 대신에, 이를 PG(placement group)으로 손쉽게 다루도록 한다.

object가 저장된 osd를 탐색하기 위해서는 pool을 placement group이라는 sub device로 나누게 된다.

* object name hash % number of pgs in the pool
* 이때의 pool : cluster를 논리적으로 나눈 파티션

<div align="left">

<figure><img src="https://blog.kakaocdn.net/dn/dhYAOg/btqYbImMjaP/84fe4AE0yfpEY7b9Etw5aK/img.png" alt=""><figcaption></figcaption></figure>

</div>

{% hint style="info" %}
A Placement Group (PG) is a logical collection of objects that are replicated on OSDs to provide reliability in a storage system. Depending on the replication level of a Ceph pool, each PG is replicated and distributed on more than one OSD of a Ceph cluster. You can consider a PG as a logical container holding multiple objects, such that this logical container is mapped to multiple OSDs
{% endhint %}



### 데이터 저장 방식

데이터를 안전하게 저장하기 위한 방식으로 두가지가 있다.

<figure><img src="https://blog.kakaocdn.net/dn/E9v8f/btqX0qgJ7rD/KS5fTrykKOPQdNb8eL2970/img.png" alt=""><figcaption></figcaption></figure>

### replicated 방식

* default : 해당 pool에서 각각의 pg를 replicated 3하는 것.

### erasure coded 방식

* 저장 공간을 절약할 수 있다.
* 하지만, 하나의 osd를 잃으면 다른 osd들을 모두 읽어서 데이터를 복구시켜야 한다.
  * cluster recovery가 더 어려워짐.





#### 참고자료 <a href="#ceph" id="ceph"></a>

[Ceph CRUSH Map, Bucket 알고리즘](https://ssup2.github.io/theory\_analysis/Ceph\_CRUSH\_Map\_Bucket\_Algorithm/)

[Ceph](https://www.jacobbaek.com/594?category=624425)

[Dynamic Data Placement with Red Hat Ceph Storage](https://www.youtube.com/watch?v=8j1aqsUEPLY\&feature=youtu.be)

<figure><img src="https://scrap.kakaocdn.net/dn/gZFBv/hyJmnbznsL/f35baJkdXAD5KDpo1NE8o0/img.jpg?width=1280&#x26;height=720&#x26;face=82_24_128_74" alt=""><figcaption></figcaption></figure>
