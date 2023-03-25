---
description: 2022. 11. 14. 09:40
---

# \[K8S] Service (NodePort / ClusterIP / LoadBalancer)

## Service ?

Worker Node위에 생성된 pod는 기본적으로는 외부에서 접근이 불가능하다.\
Service를 이용하여 외부 사용자의 pod 접근 및 pod <-> pod 의 효율적인 내부접근을 구성할 수 있다.\
Service의 기능은 다음과 같다.

* Cloud platform의 로드밸런서 or 클러스터에 속한 worker node의 port를 통해 pod를 외부에 노출
* 여러 pod에 쉽게 접근할 수 있는 고유한 도메인 이름 부여
* 여러 pod에 접근할 때 요청을 분산하는 LB 기능 수행

Service에는 3가지 종류가 있다.\
NodePort / ClusterPort / LoadBalancer 타입이다. 하나씩 알아보도록 하자!

![](https://blog.kakaocdn.net/dn/c08oXF/btrQ0HDkzBX/Gt9fO4kUnvSNZ0ktwgAju1/img.png)





## NodePort Type

Worker Node의 특정 port를 개방하여 외부에서 pod에 접근할 수 있도록 하는 방법이다.\
만약 Worker Node가 증설되면서 pod가 새로운 node에 스케줄링 되어도 기본적으로 클러스터의 모든 노드에 동일하게 port 를 개방하기 때문에 별도의 추가적인 설정 없이 접근이 가능하다.\
Node port는 definition 파일에서 지정할 수 있지만 따로 정하지 않을 경우 30000\~32767 사이에서 랜덤하게 부여된다.\
만약 원하는 NodePort를 명시하려면 ports: 섹션에 `nodePort: 30000` 이런식으로 추가하면 된다.

![](https://blog.kakaocdn.net/dn/dh3uFM/btrQ65Je7Fc/e4zeSi3w9BaktTDTRcDBf0/img.png)

### NodePort Service 생성하기

* **spec.selector** : 해당 서비스가 적용될 pod의 라벨을 지정
* **spec.ports.port** : 서비스는 k8s cluster 내에서 접근 가능한 고유한 cluster ip를 부여 받는다 . 이때 부여받은 IP에 접근할 때 사용할 포트.
* **spec.ports.targetPort** : 대상이 되는 pod들이 내부적으로 사용하는 port. 즉, pod template의 containerPort와 같아야 한다.
* **spec.type** : 해당 서비스의 타입 지정

```yaml
apiVersion: v1
kind: Service
metadata:
name: service-nodeport
spec:
ports:
- name: web-port
  port: 8080
  targetPort: 80
selector:
app: webserver
type: NodePort
```



NodePort 타입의 서비스가 생성되었다.\
이때 8080:30642/TCP ( 여기서 8080은 서비스의 port )에서 30642는 모든 node에서 해당 서비스로 동일하게 접근할 수 있는 NodePort를 의미한다. 어떠한 노드로 들어오든지 내부IP or 외부IP의 30642 포트로 접근하면 동일한 서비스에 연결된다.

```
root@master001:~/script/service# kubectl get service -o wide
NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE   SELECTOR
kubernetes          ClusterIP   10.96.0.1      <none>        443/TCP          40m   <none>
service-nodeport    NodePort    10.103.32.57   <none>        8080:30642/TCP   63s   app=webserver
```



node internal ip의 30642 포트로 접근했을때 service-nodeport 서비스를 타고 3개의 웹 pod에 랜덤하게 접근하는 것을 확인할 수 있다.

```
root@master001:~/script/service# curl 10.10.11.10:30642 --silent | grep Hello
        <p>Hello,  hostname-deployment-7dfd748479-9n9v2</p>     </blockquote>
root@master001:~/script/service# curl 10.10.11.10:30642 --silent | grep Hello
        <p>Hello,  hostname-deployment-7dfd748479-tx99m</p>     </blockquote>
root@master001:~/script/service# curl 10.10.11.20:30642 --silent | grep Hello
        <p>Hello,  hostname-deployment-7dfd748479-9n9v2</p>     </blockquote>
root@master001:~/script/service# curl 10.10.11.20:30642 --silent | grep Hello
        <p>Hello,  hostname-deployment-7dfd748479-j299p</p>     </blockquote>
```



이번에는 cluster node의 external ip로 접근해 보도록 한다.\
현재 cluster의 externel network 정보는 다음과 같다.

```
[root@wglee ~]# virsh net-dumpxml study-external
<network>
  <name>study-external</name>
  <uuid>664d3619-a64f-4388-a74b-e833e96b03b2</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr2' stp='on' delay='0'/>
  <mac address='52:54:00:01:d3:a9'/>
  <dns enable='no'/>
  <ip family='ipv4' address='172.16.110.1' prefix='24'>
    <dhcp>
      <range start='172.16.110.2' end='172.16.110.254'/>
      <host mac='52:54:00:76:9D:11' name='study-master001' ip='172.16.110.10'/>
      <host mac='52:54:00:36:8E:1D' name='study-worker001' ip='172.16.110.20'/>
      <host mac='52:54:00:32:56:44' name='study-deploy' ip='172.16.110.100'/>
      <host mac='52:54:00:02:C8:FB' name='study-worker002' ip='172.16.110.21'/>
    </dhcp>
  </ip>
</network>
```



node의 ExternalIP:NodePort 로 curl 을 해서 외부에서도 접속되는 것을 볼 수 있다.\
몇 번 계속 하다보면 요청이 랜덤하게 분산 되고 있는걸 볼 수 있다.\
이처럼 서비스는 별도의 설정을 하지 않아도 자동으로 요청을 로드밸런싱한다.

```
[root@wglee ~]# curl 172.16.110.20:30642 --silent | grep Hello
        <p>Hello,  hostname-deployment-7dfd748479-j299p</p>     </blockquote>
[root@wglee ~]# curl 172.16.110.20:30642 --silent | grep Hello
        <p>Hello,  hostname-deployment-7dfd748479-tx99m</p>     </blockquote>
[root@wglee ~]# curl 172.16.110.20:30642 --silent | grep Hello
        <p>Hello,  hostname-deployment-7dfd748479-j299p</p>     </blockquote>
[root@wglee ~]# curl 172.16.110.21:30642 --silent | grep Hello
        <p>Hello,  hostname-deployment-7dfd748479-tx99m</p>     </blockquote>
```





## ClusterIP Type

ClusterIP는 쿠버네티스 내부의 pod끼리 접근할 때 사용된다. (이는 pod를 외부에 노출하는 방법은 아니다. )\
아래와 같은 상황을 가정해 보자.\
frontend / backend application 이 동작하는 pod가 각각 동작하고 있다.\
frontend pod와 backend pod는 서로 통신해야 하는데 각 pod ip를 목적지로 해서 통신하는 것은 경우의 수가 너무 많을 뿐더러, pod가 재생성 될 경우 pod ip 또한 변동이 있을 수 있어 좋은 방법이 아니다.\
이때 Service를 ClusterIP 타입으로 배포함으로써 하나의 Service object를 거처 서로 통신하도록 할 수 있다.

<figure><img src="https://blog.kakaocdn.net/dn/5dI0S/btrQ1Kzj0QU/l2snQkOtG8VYdwacJ5ZpH0/img.png" alt=""><figcaption></figcaption></figure>

### ClusterIP Service 생성하기

* **spec.selector** : 해당 서비스가 적용될 pod의 라벨을 지정
* **spec.ports.port** : 서비스는 k8s cluster 내에서 접근 가능한 고유한 cluster ip를 부여 받는다 . 이때 부여받은 IP에 접근할 때 사용할 포트.
* **spec.ports.targetPort** : 대상이 되는 pod들이 내부적으로 사용하는 port. 즉, pod template의 containerPort와 같아야 한다.
* **spec.type** : 해당 서비스의 타입 지정

```yaml
apiVersion: v1
kind: Service
metadata:
name: clusterip_service
spec:
ports:
- name: web-port
  port: 8080
  targetPort: 80
selector:
app: webserver
type: ClusterIP
```



생성한 definition 파일로 ClusterIP 타입의 서비스를 생성한다.

```
root@master001:~/script/service# kubectl get service -o wide
NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE     SELECTOR
clusterip-service   ClusterIP   10.106.34.37   <none>        8080/TCP   8m42s   app=webserver
kubernetes          ClusterIP   10.96.0.1      <none>        443/TCP    9m23s   <none>
```



서비스를 통해 pod에 어떻게 접근 되는지 보기 위해 curl을 날려 본다.\
clusterIP의 8080(10.106.34.37:8080)을 통해 pod에 80으로 떠 있는 웹서버에 접근한다.\
NodeIP 타입과 마찬가지로 계속 요청을 날리다 보면 3개의 pod에 요청이 랜덤하게 분산 되고 있다.

```
root@master001:~/script/service# curl 10.106.34.37:8080 --silent | grep hostname
        <p>Hello,  hostname-deployment-7dfd748479-tx99m</p>     </blockquote>
root@master001:~/script/service# curl 10.106.34.37:8080 --silent | grep hostname
        <p>Hello,  hostname-deployment-7dfd748479-j299p</p>     </blockquote>
root@master001:~/script/service# curl 10.106.34.37:8080 --silent | grep hostname
        <p>Hello,  hostname-deployment-7dfd748479-tx99m</p>     </blockquote>
root@master001:~/script/service# curl 10.106.34.37:8080 --silent | grep hostname
        <p>Hello,  hostname-deployment-7dfd748479-9n9v2</p>     </blockquote>
root@master001:~/script/service# curl 10.106.34.37:8080 --silent | grep hostname
        <p>Hello,  hostname-deployment-7dfd748479-j299p</p>     </blockquote>
```



Service의 label selector와 Pod의 label이 매칭되어 Service가 모니터링할 Pod들이 정해지면 자동으로 endpoint 오브젝트가 생성된다.\
endpoint 오브젝트에는 POD\_IP:SERVICE\_PORT 정보가 등록된다.

```
root@master001:~/script/service# kubectl get endpoints
NAME                ENDPOINTS                                            AGE
clusterip-service   172.30.254.10:80,172.30.254.17:80,172.30.65.153:80   24m
kubernetes          172.16.110.10:6443                                   24m
```





## LoadBalancer Type

서비스 생성과 동시에 로드 밸런서를 생성해 pod와 연결한다.\
AWS, GCP와 같은 클라우드 플랫폼에서 적용 가능하다. 만약 온프레미스 서버에서 Load Balancer 타입을 사용할 경우, MetalLB 같은 환경을 구축해야 한다고 한다.





#### **참고 문서**

* 시작하세요 도커/쿠버네티스
* udemy Certified Kubernetes Administrator (CKA) with Practice Tests 강의
