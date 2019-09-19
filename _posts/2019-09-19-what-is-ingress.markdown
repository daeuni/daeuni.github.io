---
layout: post
title:  "Kubernetes: Ingress"
date:   2019-09-19 10:00:36

---

쿠버네티스의 서비스는 L4 레이어로 TCP 단에서 Pod들을 밸런싱한다. 쿠버네티스에서 HTTP(S)기반의 L7 로드밸런싱 기능을 제공하는 컴포넌트를 Ingress라고 한다.

서비스의 경우에는 TLS (SSL)이나, VirtualHost와 같이 여러 호스트명을 사용하거나 호스트명에 대한 라우팅이 불가능하고, URL Path에 따른 서비스간 라우팅이 불가능하다.

또한 마이크로 서비스 아키텍쳐 (MSA)의 경우에는 쿠버네티스의 서비스 하나가 MSA의 서비스로 표현되는 경우가 많고 서비스는 하나의 URL로 대표 되는 경우가 많다. (/users, /products, …) 

그래서 MSA 서비스간의 라우팅을 하기 위해서는 API 게이트웨이를 넣는 경우가 많다. 이 경우에는 API 게이트웨이에 대한 관리 포인트가 생기기 때문에 URL 기반의 라우팅 정도라면, API 게이트웨이처럼 무거운 아키텍쳐 컴포넌트가 아니라, L7 로드밸런서 정도로 위의 기능을 모두 제공이 가능하다.

쿠버네티스에서 HTTP(S)기반의 L7 로드밸런싱 기능을 제공하는 컴포넌트를 Ingress라고 한다.

개념을 도식화 해보면 아래와 같은데, Ingress 가 서비스 앞에서 L7 로드밸런서 역할을 하고, URL에 따라서 라우팅을 하게 된다.

<center><img src = "https://github.com/daeuni/daeuni.github.io/blob/master/assets/ingress.png?raw=true"></center>

Ingress 가 서비스 앞에 붙어서, URL이 /users와 /products 인것을 각각 다른 서비스로 라우팅 해주는 구조가 된다.

Ingress 은 여러가지 구현체가 존재한다.

구글 클라우드의 경우에는 [글로벌 로드 밸런서](https://github.com/kubernetes/ingress-gce/blob/master/README.md) 를 Ingress로 사용이 가능하며, 오픈소스 구현체로는 [nginx](https://github.com/kubernetes/ingress-nginx/blob/master/README.md)  기반의 ingress 구현체가 있다.  상용 제품으로는 [F5 BIG IP Controller](http://clouddocs.f5.com/products/connectors/k8s-bigip-ctlr/v1.5/) 가 현재 사용이 가능하고, 재미있는 제품으로는 오픈소스 API 게이트웨이 솔루션인 [Kong](https://konghq.com/blog/kubernetes-ingress-controller-for-kong/)이 Ingress 컨트롤러의 기능을 지원한다.

각 구현체마다 설정 방법이 다소 차이가 있으며, 특히 Ingress 기능은 베타 상태이기 때문에, 향후 변경이 있을 수 있음을 감안하여 사용하자.



<br>

**URL Path 기반의 라우팅**

이 글에서는 구글 클라우드 플랫폼의 로드밸런서를 Ingress로 사용하는 것을 예를 들어 설명한다.

위의 그림과 같이 users 와 products 서비스 두개를 구현하여 배포하고, 이를 ingress를 이용하여 URI가  /users/* 와 /products/* 를 각각의 서비스로 라우팅 하는 방법을 구현해보도록 하겠다. 

node.js와 users와 products 서비스를 구현한다.

서비스는 앞에서 계속 사용해왔던 간단한 HelloWorld 서비스를 약간 변형해서 사용하였다.

아래는 users 서비스의 server.js 코드로 “Hello World! I’m User server ..”를 HTTP 응답으로 출력하도록 하였다.  Products 서비스는 User server를 product 서버로 문자열만 변경하였다. 

```javascript
var os = require('os');
var http = require('http');
var handleRequest = function(request, response) {
  response.writeHead(200);
  response.end("Hello World! I'm User server "+os.hostname() +" \n");

  //log
  console.log("["+
		Date(Date.now()).toLocaleString()+
		"] "+os.hostname());
}

var www = http.createServer(handleRequest);
www.listen(8080);
```

다음으로 서비스를 배포해야 하는데, Ingress를 사용하려면 서비스는 Load Balancer 타입이 아니라, NodePort 타입으로 배포해야 한다.  다음은 <u>user 서비스</u>를 nodeport 서비스로 배포하는 yaml 스크립트이다. users-svc-nodepart.yaml 파일로 만든다. (Pod를 컨트롤하는 Deployment 스크립트는 생략하였다.)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: users-node-svc
spec:
  selector:
    app: users
  type: NodePort
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 8080
```

같은 방식으로, <u>product 서비스</u>도 아래와같이 NodePart로 배포한다. product-svc-nodepart.yaml 파일로 만든다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: products-node-svc
spec:
  selector:
    app: products
  type: NodePort
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 8080
```

이때 별도로 nodeport를 지정해주지 않았는데, 자동으로 쿠버네티스 클러스터가 nodeport를 지정해준다.

아래와 같이 products-node-svc와 users-node-svc가 각각 배포된것을 확인할 수 있고, ClusterIP의 포트는 80, NodePort는 각각 31442, 32220으로 배포된것을 확인할 수 있다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99300E3A5B2D16942A"></center>

 다음 Ingress를 생성해보자. 다음은 hello-ingress 라는 이름으로 위에서 만든 두개의 서비스를 라우팅해주는 서비스를 생성하기 위한 yaml  파일이다. 파일 이름은 hello-ingress.yaml로 한다.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hello-ingress
spec:
  rules:
  - http:
      paths:
      - path: /users/*
        backend:
          serviceName: users-node-svc
          servicePort: 80
      - path: /products/*
        backend:
          serviceName: products-node-svc
          servicePort: 80
```

spec의 rules.http.paths 부분에, 라우팅할 path와 서비스를 정의해준다.

User 서비스는 /users/* URI인 경우 라우팅하게 하고, 앞에서 만든 users-node-svc로 라우팅하도록 한다. 이때 servicePort는 ClusterIP의 service port를 지정한다. (Google Cloud HTTP Load balancer를 이용하는 Ingress의 경우에는  실질적으로는 nodeport로 통신을 하지만 별도로 지정하지 않고 ingress가 자동으로 해당 서비스의 nodeport를 찾아서 맵핑이 된다.  )
(참고 : https://kubernetes.io/docs/concepts/services-networking/ingress/ Lines 12-14: A backend is a service:port combination as described in the [services doc](https://kubernetes.io/docs/concepts/services-networking/service/). Ingress traffic is typically sent directly to the endpoints matching a backend.)

```shell
%kubectl create -f hello-ingress.yaml
```

을 실행하면 ingress가 생성이 되고 kubectl get ing 명령어를 이용하면 생성된 ingress를 확인할 수 있다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99E3314F5B2D16930F"></center>

Ingress 가 생성된 후, 실제로 사용이 가능하기까지는 약 1~2분의 시간이 소요된다. 

물리적으로 HTTP 로드밸런서를 생성하고, 이 로드밸런서가 서비스가 배포되어 있는 노드에 대한 HealthCheck를 완료하고 문제가 없으면 서비스를 제공하는데, HealthCheck 주기가 1분이기 때문에, 1~2분 정도를 기다려 주는게 좋다. 그전까지는 404 에러나 500 에러가 날것이다.

준비가 끝난후, curl 명령을 이용해서 ingress의 URL에 /users/ 와 /products를 각각 호출해보면 각각, users를 서비스 하는 서버와, products를 서비스 하는 서버로 라우팅이 되서 각각 다른 메세지가 출력되는 것을 확인할 수 있다.

<br>

**Google Cloud Console Ingress 확인**

그러면 내부적으로 클라우드 내에서 Ingress를 위한 인프라가 어떻게 생성되었는지 확인해보자.

구글 클라우드 콘솔에서 Network services > Load balancing 메뉴로 들어가보자. 아래와 같이 HTTP 로드밸런서가 생성이 된 것을 확인할 수 있다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99E5FE4D5B2D16930E"></center>

이름을 보면 k8s는 쿠버네티스용 로드밸런서임을 뜻하고, 중간에 default는 네임 스페이스를 의미한다. 그리고 ingress의 이름인 hello-ingress로 생성이 되어 있다. 

로드밸런서를 클릭해서 디테일을 들어가 보면 아래와 같은 정보를 확인할 수 있다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/9938A5425B2D16932A"></center>

3개의 백엔드 (인스턴스 그룹)이 맵핑되었으며, /users/*용, /products/*용 그리고, 디폴트용이 생성되었다.

모든 트래픽이 쿠버네티스 클러스터 노드로 동일하게 들어가기 때문에, Instance group의 이름을 보면 모두 동일한것을 확인할 수 있다. 단, 중간에 Named Port 부분을 보면 포트가 다른것을 볼 수 있는데, 31442, 32220 포트를 사용하고 있고, 앞에서 users, produtcs 서비스를 nodeport로 생성하였을때, 자동으로 할당된 nodeport이다. 

개념적으로 다음과 같은 구조가 된다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99D9683F5B2D169315"></center>

(편의상 디폴트 백앤드의 라우팅은 표현에서 제외하였다.)

Ingress에 접속되는 서비스를 LoadBalancer나 ClusterIP타입이 아닌 NodePort 타입을 사용하는 이유는, Ingress로 사용되는 구글 클라우드 로드밸런서에서, 각 서비스에 대한 Hearbeat 체크를 하기 위해서인데, Ingress로 배포된 구글 클라우드 로드밸런서는 각 노드에 대해서 nodeport로 Heartbeat 체크를 해서 문제 있는 노드를 로드밸런서에서 자동으로 제거나 복구가되었을때는 자동으로 추가한다. 

<br>

**Ingress Static IP 지정하기**

서비스와 마찬가지로 Ingress 역시 Static IP를 지정할 수 있다. 

서비스와 마찬가지로, static IP를 gcloud 명령을 이용해서 생성한다. 이때 IP를 regional로 생성할 수 도 있지만, ingress의 경우에는 global IP를 사용할 수 있다. --global 옵션을 주면되는데, global IP의 경우에는 regional IP와는 다르게 구글 클라우드의 망 가속 기능을 이용하기 때문에, 구글 클라우드의 100+ 의 Pop (Point of Presence)를 이용하여 가속이 된다.

조금 더 깊게 설명을 하면, 일반적으로 한국에서 미국으로 트래픽을 보낼 경우 한국 → 인터넷 → 미국 식으로 트래픽이 가는데 반해 global IP를 이용하면, 한국에서 가장 가까운 Pop (일본)으로 접속되고, Pop으로 부터는 구글 클라우드의 전용 네트워크를 이용해서 구글 데이타 센터까지 연결 (한국 → 인터넷 → 일본 Pop → 미국 ) 이 되기 때문에 일반 인터넷으로 연결하는 것 대비에서 빠른 성능을 낼 수 있다.

아래와 같이 gcloud 명령을 이용하여, global IP를 생성한다.

```shell
$ gcloud compute addresses create hello-ingress-ip --global
```

  구글 클라우드 콘솔에서, 정적 IP를 확인해보면 아래와 같이 hello-ingress-ip 와 같이 IP가 생성되어 등록되어 있는 것을 확인할 수 있다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/990F3B385B2D169329"></center>

Static IP를 이용해서 hello-ingress-staticip 이름으로 ingress를 만들어보자

다음과 같이 hello-ingress-staticip.yaml 파일을 생성한다. 

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hello-ingress-staticip
  annotations:
    kubernetes.io/ingress.global-static-ip-name: "hello-ingress-ip"
spec:
  rules:
  - http:
      paths:
      - path: /users/*
        backend:
          serviceName: users-node-svc
          servicePort: 80
      - path: /products/*
        backend:
          serviceName: products-node-svc
          servicePort: 80
```

  이 파일을 이용하여, ingress를 생성한 후에, ingress ip를 확인하고 curl 을 이용해서 결과를 확인하면 다음과 같다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99B03B425B2D169331"></center>

<br>

**Ingress with TLS**

이번에는 Ingress 로드밸런서를 HTTP가 아닌 HTTPS로 생성해보자.

[1] SSL 인증서 생성

SSL을 사용하기 위해서는 SSL 인증서를 생성해야 한다. openssl (https://www.openssl.org/)툴을 이용하여 인증서를 생성해보도록 한다.



인증서 생성에 사용할 키를 생성한다.

```shell
%openssl genrsa -out hello-ingress.key 2048
```

명령으로 키를 생성하면 hello-ingress.key라는 이름으로 Private Key 파일이 생성된다. 

다음 SSL 인증서를 생성하기 위해서, 인증서 신청서를 생성한다.인증서 신청서 생성시에는 앞에서 생성한 Private Key를 사용한다. 

다음 명령어를 실행해서 인증서 신청서 생성을 한다. 

```shell
%openssl req -new -key hello-ingress.key -out hello-ingress.csr
```

이때 인증서 내용에 들어갈 국가, 회사 정보, 연락처등을 아래와 같이 입력한다. 

인증서 신청서가 hello-ingress.csr 파일로 생성이 되었다. 그러면 이 신청서를 이용하여, SSL 인증서를 생성하자. 테스트이기 때문에 공인 인증 기관에 신청하지 않고, 간단하게 사설 인증서를 생성하도록 하겠다.

다음 명령어를 이용하여 hello-ingress.crt라는 이름으로 SSL 인증서를 생성한다.  

```shell
%openssl x509 -req -day 265 -in hello-ingress.csr -signkey hello-ingress.key -out hello-ingress.crt
```

<br>

[2] 설정하기

SSL 인증서 생성이 완료되었으면, 이 인증서를 이용하여 SSL을 지원하는 ingress를 생성해본다.

SSL 인증을 위해서는 앞서 생성한 인증서와 Private Key 파일이 필요한데, Ingress는 이 파일을 쿠버네티스의 secret 을 이용하여 읽어들인다.

Private Key와 SSL 인증서를 저장할 secret를 생성해보자 앞에서 생성한 hello-ingress.key와 hello-ingress.crt 파일이 ./ssl_cert 디렉토리에 있다고 하자.

다음과 같이 kubectl create secret tls 명령을 이용해서 hello-ingress-secret 이란 이름의 secret을 생성한다. 

```shell
%kubectl create secret tls hello-ingress-serect --key ./ssl_cert/hello-ingress.key --cert ./ssl_cert/hello-ingress.crt 
```

명령을 이용하여 secret을 생성하면, key 이라는 이름으로 hello-ingress.key 파일이 바이너리 형태로 secret에 저장되고 마찬가지로 cert라는 이름으로 hello-ingress.crt 가 저장된다. 

생성된 secret을 확인하기 위해서

```shell
%kubectl describe secret hello-ingress-secret 
```

명령을 실행해보면 아래와 같이 tls.key 와 tls.crt 항목이 각각 생성된것을 확인할 수 있다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/999EFE375B2D16931D"></center>

다음 SSL을 지원하는 ingress를 생성해야 한다.

앞에서 생성한 HTTP ingress와 설정이 다르지 않으나 spec 부분에 tls라는 항목에 SSL 인증서와 Private Key를 저장한 secret 이름을 secretName이라는 항목으로 넘겨줘야 한다. 파일 이름은 hello-ingress-tls.yaml로 한다.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hello-ingress-tls
spec:
  tls:
  - secretName: hello-ingress-secret
  rules:
  - http:
      paths:
      - path: /users/*
        backend:
          serviceName: users-node-svc
          servicePort: 80
      - path: /products/*
        backend:
          serviceName: products-node-svc
          servicePort: 80
```

이 파일을 이용해서 TLS ingress를 생성한 후에, IP를 조회해보자

아래와 같이 35.241.6.159 IP에 hello-ingress-tls 이름으로 ingress가 된것을 확인할 수 있고 포트는 HTTP 포트인 80 포트 이외에, HTTPS포트인 443 포트를 사용하는 것을 볼 수 있다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99EBDD505B2D169328"></center>

다음 HTTPS로 테스트를 해보면 접속이 되는 것을 확인할 수 있다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99FB1B465B2D16932C"></center>



<hr>

<br>

이번 글에서는 L7에서 호스트명에 대한 라우팅을 하기 위한 Ingress에 대해 알아보았다.

다음 포스트는 각 컨테이너의 상태를 체크하는 기능인 HealthCheck에 대해 알아보자.

<br>

<hr>
<br>



