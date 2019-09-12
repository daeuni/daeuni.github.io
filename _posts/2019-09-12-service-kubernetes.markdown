---
layout: post
title:  "Kubernetes: Service"
date:   2019-09-12 10:00:36
categories: pod loadbalancing dns 

---

Pod의 경우에 지정되는 Ip가 랜덤하게 지정이 되고 restart 때마다 변하기 때문에 고정된 엔드포인트로 호출이 어렵다.

또한 여러 Pod에 같은 애플리케이션을 운용할 경우 이 Pod 간의 로드밸런싱을 지원해줘야 하는데, 서비스가 이러한 역할을 한다.

서비스는 지정된 IP로 생성이 가능하고, 여러 Pod를 묶어서 로드 밸런싱이 가능하며, 고유한 DNS 이름을 가질 수 있다.



<br>

**서비스 구성**

서비스는 라벨 셀렉터 (label selector)를 이용하여, 관리하고자 하는 Pod 들을 정의할 수 있다. 

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-node-svc
spec:
  selector:
    app: hello-node
  ports:
    - port: 80
      protocol: TCP
      targetPort: 8080
  type: LoadBalancer
```



<br>

**멀티 포트 지원**

  서비스는 동시에 하나의 포트 뿐 아니라 여러개의 포트를 동시에 지원할 수 있다. 

예를 들어 웹서버의 HTTP와 HTTPS 포트가 대표적인데,  아래와 같이 ports 부분에 두개의 포트 정보를 정의해주면 된다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-node-svc
spec:
  selector:
    app: hello-node
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 8080
    - name: https
      port: 443
      protocol: TCP
      targetPort: 8082
  type: LoadBalancer
```



<br>

**로드밸런싱 알고리즘**

서비스가 Pod들에 부하를 분산할때 디폴트 알고리즘은 Pod 간에 랜덤으로 부하를 분산하도록 한다. 

만약에 특정 클라이언트가 특정 Pod로 지속적으로 연결이 되게 하려면  Session Affinity를 사용하면 되는데, 서비스의 spec 부분에 sessionAffinity: ClientIP로 주면 된다.

```yaml
apiVersion: v1
kind: Service
spec:
  sessionAffinity: ClientIP
  ...
```

웹에서   HTTP Session을 사용하는 경우와 같이 각 서버에 각 클라이언트의 상태정보가 저장되어 있는 경우에 유용하게 사용할 수 있다.

<hr>

<br>

**Service Type**

서비스는 IP 주소 할당 방식과 연동 서비스등에 따라 크게 4가지로 구별할 수 있다.

- Cluster IP
- Load Balancer
- Node IP
- External name

<br>

[1] ClusterIP

디폴트 설정으로, 서비스에 클러스터 IP (내부 IP)를 할당한다. 쿠버네티스 클러스터 내에서는 이 서비스에 접근이 가능하지만, 클러스터 외부에서는 외부 IP 를 할당  받지 못했기 때문에, 접근이 불가능하다.

[2] Load Balancer

보통 클라우드 벤더에서 제공하는 설정 방식으로, 외부 IP 를 가지고 있는 로드밸런서를 할당한다. 외부 IP를 가지고 있기  때문에, 클러스터 외부에서 접근이 가능하다. 

[3] NodePort

클러스터 IP로만 접근이 가능한것이 아니라, 모든 노드의 IP와 포트를 통해서도 접근이 가능하게 된다. 예를 들어 아래와 같이 hello-node-svc 라는 서비스를 NodePort 타입으로 선언을 하고, nodePort를 30036으로 설정하면, 아래 설정에 따라 클러스터 IP의  80포트로도 접근이 가능하지만, 모든 노드의 30036 포트로도 서비스를 접근할 수 있다. 

<br>

hello-node-svc-nodeport.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-node-svc
spec:
  selector:
    app: hello-node
  type: NodePort
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 8080
      nodePort: 30036
```

아래 그림과 같은 구조가 된다.

<center><img src = "https://github.com/daeuni/daeuni.github.io/blob/master/assets/cloudservice2.png?raw=true"></center>

이를 간단하게 테스트 해보자.

아래는 구글 클라우드에서 쿠버네티스 테스트 환경에서 노드로 사용되고 있는 3개의 VM 목록과 IP 주소이다.

현재 노드는 아래와 같이 3개의 노드가 배포되어 있고 IP 는 10.146.0.8~10이다.

내부 IP이기 때문에, VPC 내의 내부 IP를 가지고 있는 서버에서 테스트를 해야 한다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99F5A4475B27C2D60C"></center>

같은 내부 IP를 가지고 있는 envoy-ubuntu 라는 머신 (10.146.0.18)에서 각 노드의 30036 포트로 curl을 테스트해본 결과 아래와 같이 모든 노드의 IP를 통해서 서비스 접근이 가능한것을 확인할 수 있다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/992C4A495B27C2D619"></center>

<br>

[4]External Name

ExternalName은 외부 서비스를, 쿠버네티스 내부에서 호출할 때 사용한다. 

쿠버네티스 클러스터내의 Pod들은 클러스터 IP를 가지고 있기 때문에, 클러스터 IP 대역 밖의 서비스를 호출하고자 하면, NAT 설정 등 복잡한 설정이 필요하다.

특히 AWS 나 GCP와 같은 클라우드 환경을 사용할 경우 데이터베이스나, 또는 클라우드에서 제공되는 매지니드 서비스 (RDS, CloudSQL)등을 사용하고자 할 경우에는 쿠버네티스 클러스터 밖이기 때문에, 호출이 어려운 경우가 있다.

이를 쉽게 해결할 수 있는 방법이 ExternalName 타입이다.

아래와 같이 서비스를 ExternalName 타입으로 설정하고, 주소를 DNS로 my.database.example.com으로 설정해주면 

이 my-service는 들어오는 모든 요청을 my.database.example.com 으로 포워딩 해준다. (일종의 프록시와 같은 역할이다.) 

```yaml
kind: Service
apiVersion: v1
metadata:
	name: my-service
	namespace: prod
spec:
	type: ExternalName
	externalName: my.database.example.com
```

다음과 같은 구조로 서비스가 배포된다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/9958713E5B27C2D60A"></center>

<br>

**DNS가 아닌 IP를 이용하는 방식 ?**

위의 경우 DNS를 이용하였는데, DNS가 아니라 직접 IP 주소를 이용하는 방법도 있다.

서비스 ClusterIP 서비스로 생성을 한 후에, 이 때 서비스에 속해있는 Pod를 지정하지 않는다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-svc-nginx
spec:
  ports:
  - port: 80
```

다음으로, 아래와 같이 서비스의 EndPoint를 별도로 지정해주면 된다.

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: external-svc-nginx
subsets:
  - addresses:
    - ip: 35.225.75.124
    ports:
    - port: 80
```

이 때 서비스명과 서비스 EndPoints의 이름이 동일해야 한다. 위의 경우에는 external-svc-nginx로 같은 서비스명을 사용하였고 이 서비스는 35.225.75.124:80 서비스를 가르키도록 되어 있다.

그림으로 구조를 표현해보면 다음과 같다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/9902354D5B27C2D607"></center>



35.225.75.124:80 은 nginx 웹서버가 떠 있는 외부 서비스이고, 아래와 같이 간단한 문자열을 리턴하도록 되어 있다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99147D3C5B27C2D62B"></center>

이를 쿠버네티스 내부 클러스터의 Pod 에서 curl 명령을 이용해서 호출해보면 다음과 같이 외부 서비스를 호출할 수 있음을 확인할 수 있다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/992E52415B27C2D634"></center>

<br>

**Headless Service**

서비스는 접근을 위해서 Cluster IP 또는 External IP 를 지정받는다.

즉 서비스를 통해서 제공되는 기능들에 대한 엔드포인트를 쿠버네티스 서비스를 통해서 통제하는 개념이다.

마이크로 서비스 아키텍처에서는 기능 컴포넌트에 대한 엔드포인트 (IP 주소)를 찾는 기능을 서비스 디스커버리 (Service Discovery) 라고 하고, 서비스의 위치를 등록해놓는 서비스 디스커버리 솔루션을 제공한다. 

Etcd 나 hashcorp의 consul (https://www.consul.io/)과 같은 솔루션이 대표적인 사례인데, 이 경우 쿠버네티스 서비스를 통해서 마이크로 서비스 컴포넌트를 관리하는 것이 아니라, <u>서비스 디스커버리 솔루션을 이용</u>하기 때문에, 서비스에 대한 IP 주소가 필요없다.

이런 시나리오를 지원하기 위한 쿠버네티스의 서비스를 헤드리스 서비스 (Headless service) 라고 하는데, 이러한 헤드리스 서비스는 Cluster IP등의 주소를 가지지 않는다. 단 DNS이름을 가지게 되는데, 이 DNS 이름을 lookup 해보면, 서비스 (로드밸런서)의 IP 를 리턴하지 않고, 이 서비스에 연결된 Pod 들의 IP 주소들을 리턴하게 된다.



간단한 테스트를 해보면

![img](https://t1.daumcdn.net/cfile/tistory/993703505B27C2D611)



와 같이 기동중인 Pod들이 있을때, Pod의 IP를 조회해보면 다음과 같다.

![img](https://t1.daumcdn.net/cfile/tistory/99CA8B495B27C2D630)



10.20.0.25, 10.20.0.22, 10.20.0.29, 10.20.0.26 4개가 되는데, 

다음 스크립트를 이용해서 hello-node-svc-headless 라는 헤드리스 서비스를 만들어보자 

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-node-svc-headless
spec:
  clusterIP: None
  selector:
    app: hello-node
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 8080
```

아래와 같이 ClusterIP가 할당되지 않음을 확인할 수 있다.

![img](https://t1.daumcdn.net/cfile/tistory/998968355B27C2D629)

<br>

다음 쿠버네티스 클러스터내의 다른 Pod에서 nslookup으로 해당 서비스의 dns 이름을 조회해보면 

다음과 같이 서비스에 의해 제공되는 pod 들의 IP 주소 목록이 나오는 것을 확인할 수 있다. 

![img](https://t1.daumcdn.net/cfile/tistory/991DAB3D5B27C2D608)

<br>

**Service discovery**

그러면 생성된 서비스의 IP를 어떻게 알 수 있을까? 서비스가 생성된 후 kubectl get svc를 이용하면 생성된 서비스와 IP를 받아올 수 있지만, 이는 서비스가 생성된 후이고, 계속해서 변경되는 임시 IP이다.

**[1] DNS를 이용하는 방법**

가장 쉬운 방법으로는 DNS 이름을 사용하는 방법이 있다.

서비스는 생성되면 [서비스 명].[네임스페이스명].svc.cluster.local 이라는 DNS 명으로 쿠버네티스 내부 DNS에 등록이 된다. 

쿠버네티스 클러스터 내부에서는 이 DNS 명으로 서비스에 접근이 가능한데, 이때 DNS에서 리턴해주는 IP는 외부 IP (External IP)가 아니라 Cluster IP (내부 IP)이다.

<br>

아래 간단한 테스트를 살펴보자. hello-node-svc 가 생성이 되었는데, 클러스터내의 pod 중 하나에서 ping으로 hello-node-svc.default.svc.cluster.local 을 테스트 하니, hello-node-svc의 클러스터 IP인 10.23.241.62가 리턴되는 것을 확인할 수 있다. 

![img](https://t1.daumcdn.net/cfile/tistory/99E0E0365B27C2D605)

<br>

**[2] External IP (외부 IP)**

다른 방식으로는 외부 IP를 명시적으로 지정하는 방식이 있다. 쿠버네티스 클러스터에서는 이 외부 IP를 별도로 관리하지 않기 때문에, 이 IP는 외부에서 명시적으로 관리되어야 한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-node-svc
spec:
  selector:
    app: hello-node
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 8080
  externalIPs:
  - 80.11.12.11
```

외부 IP는 Service의 spec 부분에서 externalIPs 부분에 IP 주소를 지정해주면 된다.

<br>

**구글 클라우드의 경우**

퍼블릭 클라우드 (AWS, GCP 등)의 경우에는 이 방식 보다는 클라우드내의 로드밸런서를 붙이는 방법을 사용한다. 

구글 클라우드의 경우를 살펴보자.

서비스에 정적인 IP를 지정하기 위해서는 정적 IP를 생성해야 한다. 구글 클라우드 콘솔내의 VPC 메뉴의 External IP 메뉴에서 생성해도 되고, 아래와 같이 gcloud CLI 명령어를 이용해서 생성해도 된다. 

IP를 생성하는 명령어는 gcloud compute addresses create [IP 리소스명] --region [리전]

을 사용하면 된다. 구글 클라우드의 경우에는 특정 리전만 사용할 수 있는 리저널 IP와, 글로벌에 모두 사용할 있는 IP가 있는데, 서비스에서는 리저널 IP만 사용이 가능하다. (글로벌 IP는 후에 설명하는 Ingress에서 사용이 가능하다.)

아래와 같이 

```shell
%gcloud compute addresses create hello-node-ip-region  --region asia-northeast1
```

명령어를 이용해서 asia-northeast1 리전 (일본)에 hello-node-ip-region 이라는 이름으로 ip를 생성하였다. 생성된 IP는 describe 명령을 이용해서 확인할 수 있으며, 아래 35.200.64.17 이 배정된것을 확인할 수 있다. 

![img](https://t1.daumcdn.net/cfile/tistory/99B0403C5B27C2D60C)

이 IP는 서비스가 삭제되더라도 계속 유지되고, 다시 재 사용이 가능하다.

그러면 생성된 IP를 service에 적용해보자

다음과 같이 hello-node-svc-lb-externalip.yaml  파일을 생성하자

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-node-svc
spec:
  selector:
    app: hello-node
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 8080
  type: LoadBalancer
  loadBalancerIP: 35.200.64.17
```

타입을 LoadBalancer로 하고, loadBalancerIP 부분에 앞에서 생성한 35.200.64.17 IP를 할당한다.

다음 이 파일을 kubectl create -f hello-node-svc-lb-externalip.yaml 명령을 이용해서 생성하면, hello-node-svc 가 생성이 되고, 아래와 같이 External IP가 우리가 앞에서 지정한 35.200.64.17 이 지정된것을 확인할 수 있다. 

![img](https://t1.daumcdn.net/cfile/tistory/992C42415B27C2D613)



<hr>

<br>

이번 글에서는 서비스에 대해 살펴보았다. 서비스를 어떻게 구성해야 하는지 그리고 외부 로드밸런서와 어떻게 연결하는지 알아보았다.

다음 포스트는 서비스 앞에 있는 Ingress에 대해 알아보자.

<br>

<hr>
<br>
