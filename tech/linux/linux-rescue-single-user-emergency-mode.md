---
description: 2022. 5. 15. 10:10
---

# Linux 시스템 복구 방법 (Rescue, Single-user, Emergency mode)

## Intro

root 패스워드를 분실 하여 시스템에 접속할 방법이 없거나, /etc/fstab 을 잘못 설정하여 파일 시스템 오류로 정상 부팅 되지 않는 겨우가 있다.\
이럴 때는 각 상황에 맞는 방법으로 시스템 복구를 수행하는데, rescue mode, single-user mode, emergency mode 등의 차이를 잘 몰라 한번 알아보고자 한다.



## 1. Rescue Mode

Rescue Mode는 리눅스 설치 CD나 USB를 이용해서 복구하는 방법이다.\
설치 모드에서 linux rescue 명령어를 이용하여 접근할 수 있다고 한다.\
/etc/fstab의 파일 시스템의 설정 등이 잘못 되어 부팅이 안되는 경우 Rescue mode에서 해당 설정 파일을 수정하고 다시 부팅 할 수 있다.



## 2. Single User Mode

설치 CD나 USB 없이 시도할 수 있다는 장점이 있다.\
Single-user mode로 부팅을 하면 시스템은 runlevel 1(Single Mode)으로 부팅된다.\
부팅 과정에서 파일 시스템을 mount 하려고 시도하기 때문에, 파일 시스템 문제로 부팅이 안되는 상황이라면 Single user mode는 도움이 안될 것이다. (이럴때는 emergency mode 와 같은 다른 방법을 사용해야 한다)



## 3. Emergency Mode

시스템이 부팅될 수 있는 가장 최소한의 수준으로 올라오는 방법으로, 네트워크도 활성화 하지 않는다.\
root 파일 시스템만 read-only 권한으로 마운트 한다.\
root 영역을 제외한 파일 시스템은 자동 마운트 되지 않기 때문에 설정을 확인한 후에 수동으로 마운트 시도하고 데이터에 접근할 수 있다.\
Single-user mode와 달리 init file들이 로드 되지 않은 상태이기 때문에 init 설정에 문제가 있는 상황에서도 부팅이 가능하다.



## 참고 문서

{% embed url="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/deployment_guide/ch-system_recovery" %}
