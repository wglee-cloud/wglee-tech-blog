---
description: 2022. 9. 18. 21:05
---

# 헷갈리는 데이터의 단위 정리 ( KB, KiB / MB, MiB )

## Intro

데이터의 단위를 보다 보면 KB, MB, GB.. 가 있는 반면 KiB, MiB.. 같은 단위도 볼 수 있다.\
두 데이터 단위의 차이는 2진수로 데이터를 처리하는 컴퓨터의 특성 때문인 것으로 알고 있다.\
KB 는 10진수로 데이터를 계산하는 반면, KiB는 2의 10승 단위로 데이터를 계산하는 것이다.\
머릿속에서 파편화 되어 있는 단위들을 한번 정확하게 정리해 보도록 한다.



## SI 단위

일상생활에서 익숙하게 접하는 데이터 단위에 해당한다.\
10진수로 데이터를 계산하여서 값 비교가 좀 더 편리하고 인간 친화적이다.

* 1KB(kilo-bytes, 킬로바이트) = 1,000B (byte. 바이트)
* 1MB(mega-bytes) = 1,000KB
* 1GB (giga-bytes) = 1,000MB
* 1TB (tera-bytes) = 1,000 GB



## IEC 단위

IEC(International Electrotechnical Commission 국제 전기기술 위원회) 에서 지정한 단위에 해당한다.\
IEC 단위를 사용하는 이유는 SE 단위가 인간 친화적인 것에 반해, 컴퓨터는 데이터를 2진수로 처리하기 때문에 컴퓨터에서 실제로 인식하는 용량과는 오차범위가 있기 때문이다.

{% hint style="info" %}
디지털 형식의 신호는 0(False) 과 1(True)로 구성된다.\
이때 0과 1에 해당 하는 데이터가 bit 이다.\
8 bit가 모이면 1 Byte가 된다.\
1개의 bit는 0과 1 두 가지의 상태를 나타낼 수 있기 때문에, 1byte는 2의 8승인 256 종류의 정보를 나타낼 수 있다.\
1byte로는 영어권 알파벳 1개의 문자를 나타낼 수 있고, 한글의 경우에는 한 문자당 2byte가 필요하다.
{% endhint %}

* 1Byte = 8bit
* 1KiB(kibi-bytes) = 1,024 Byte
* 1MiB(mebi-bytes) = 1,024 KiB ( = 1048KB )
* 1GiB(gibi-bytes) = 1,024 MiB ( = 1073MB )
* 1TiB(tebi-bytes) = 1,024 GiB ( = 1099GB )



## **정리**

위의 내용을 표로 정리하면 아래와 같다.

| **2진수**   |      |                 | **10진수**  |     |                  |
| --------- | ---- | --------------- | --------- | --- | ---------------- |
| 단위명       | 표기   | 값               | 단위명       | 표기  | 값                |
| kibibyte  | KiB  | 2^10 (1024byte) | kilobyte  | KB  | 10^3 (1000byte)  |
| mebibyte  | MiB  | 2^20 (1024^2)   | megabyte  | MB  | 10^6 (1000^2)    |
| gibibyte  | GiB  | 2^30 (1024^3)   | gigabyte  | GB  | 10^9 (1000^3)    |
| tebibyte  | TiB  | 2^40 (1024^4)   | terabyte  | TB  | 10^12 (1000^4)   |



## 단위의 혼용과 오차

따라서 예를 들어서 제조업체 기준으로 1TB 짜리 저장장치를 구매 했다면,\
컴퓨터는 1TB에 해당하는 1,000,000,000,000 byte를 2진수로 계산하여 1TB가 아닌 931.322 GiB 로 인식한다.\
SI/ IEC 단위에 따른 오차 가이드 라인은 아래 IBM 문서에서 확인할 수 있다.

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption><p>image from https://www.ibm.com/docs/en/storage-insights?topic=overview-units-measurement-storage-data</p></figcaption></figure>



## 참고 문서

{% embed url="https://www.ibm.com/docs/en/storage-insights?topic=overview-units-measurement-storage-data" %}

{% embed url="https://semiconductor.samsung.com/kr/support/tools-resources/dictionary/bits-and-bytes-units-of-data/https://seungwubaek.github.io/computing/computer/data-storage/units_of_data_size/" %}

