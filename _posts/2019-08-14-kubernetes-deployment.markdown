---
layout: post
title:  "Kubernetes: Deployment"
date:   2019-09-30 10:00:36

---

애플리케이션을 배포 하는 방법에 대해서 알아보자. 일반적으로 애플리케이션을 배포하는 방법은 블루/그린, 카날리 배포, 롤링 업데이트도 여러가지 방법이 있다. 그 중에서 몇가지 패턴에 대해서 알아보도록 하자.

<br>

**롤링 업데이트**

롤링 업데이트는 가장 많이 사용되는 배포 방식 중의 하나이다. 

새 버전을 배포하면서 새 버전 인스턴스를 하나씩 늘려나가고, 기존 버전을 하나씩 줄여나가는 방식이다. 이 경우 기존 버전과 새 버전이 동시에 존재할 수 있는 단점은 있지만, 시스템을 무장애로 업데이트할 수 있다는 장점이 있다. 

롤링 업데이트를 쿠버네티스의 Replication Controller (RC)를 이용하는 방법을 보면 다음과 같다. 

아래와 같이 RC가 v1 버전의 Pod들을 관리하고 있다고 보자. 아래와 같이 3개의 Pod가 서비스 되고 있다고 하자.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/996976385B3717AB14"></center>

v2를 배포하기 위해서, v2 Pod를 컨트롤한 RC를 만들고 replica의 수를 1로해서 v2 Pod를 하나 생성한다.. 그리고 RC v1에서, replica의 수를 3 → 2로 줄여서 v1 Pod의 수를 2개로 조정한다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/990BD74F5B3717AB27"></center>

같은 방식으로 v1 Pod의 수를 1로 조절하고, v2 Pod의 수를 2로 늘린다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/997FE0445B3717AB15"></center>

마지막으로 Pod v1을 0으로 줄이고, RC v1을 삭제한다. 그리고 Pod v2을 하나 더 늘린다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/9980C0405B3717AB1F"></center>

  앞에서 봤듯이, 롤링 업데이트를 하려면 <u>RC를 두 개</u>를 만들어야 하고, RC의 replica 수를 단계적으로 조절해줘야 한다. 또한 배포가 잘못되었을때 이를 롤백하려면 이 순서를 꺼꾸로 다시 해야 한다.

<br>

**Deployment**

여러가지 배포 방식을 RC를 이용해서 구현할 수 있지만, 운영이 복잡해지는 단점이 있다. 그래서 쿠버네티스에서는 일반적으로 RC를 이용해서 배포하지 않고 Deployment라는 개념을 사용한다. 

앞에서 봤듯이 롤링 업데이트 등을 할때, RC를 두개를 만들어야 하고, 하나씩 Pod의 수를 수동으로 조정해야 하기 때문에 이를 자동화해서 추상화한 개념이 Deployment이다.

Deployment는 기본적으로 RC를 생성하고, 이를 관리하는 역할을 한다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/995B25405B3717AB10"></center>

v1 버전을 배포했다가 v2를 배포하는 시나리오를 구현해보자. 먼저 node.js로 v1과 v2 애플리케이션을 구현해보자.

아래는 version 1 애플리케이션인 server.js 이다. 응답으로 “Hello World! I’m Server version 1” 을 리턴한다. 

```javascript
var os = require('os');
var http = require('http');

var handleRequest = function(request, response) {

  response.writeHead(200);
  response.end("Hello World! I'm Server version 1 .  "+os.hostname() +" \n");

  //log
  console.log("["+
		Date(Date.now()).toLocaleString()+
		"] "+os.hostname());
}

var www = http.createServer(handleRequest);
www.listen(8080);
```

Version 2 애플리케이션도 내용은 갔다. 응답 문자열을 아래와 같이 version 1에서 version 2로만 변경하였다.

```javascript
response.end("Hello World! I'm Server version 2. "+os.hostname() +" \n");
```

이 두 server.js를 도커 이미지로 만든 다음에, gcr.io/terrycho-sandbox/deployment:v1 , gcr.io/terrycho-sandbox/deployment:v2 라는 이름으로 컨테이너 레지스트리에 등록한다.

다음 Pod를 컨트롤할 RC를 정의해야 하는데, Deployment를 사용하면 RC를 별도로 정의하지 않고, Deployment가 자동으로 RC를 생성하도록 한다.

아래는 hello-deployment.yaml 로 Deployment를 정의한 내용이다.  RC 정의와 크게 다르지 않고, selector 부분을 matchLabels를 사용하였다.

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: hello-deployment
spec:
  replicas: 3
  minReadySeconds: 5
  selector:
    matchLabels:
      app: hello-deployment
  template:
    metadata:
      name: hello-deployment-pod
      labels:
        app: hello-deployment
    spec:
      containers:
      - name: hello-deployment
        image: gcr.io/terrycho-sandbox/deployment:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
```

롤링 업데이트시 Pod 배포가 빠르게 되기 때문에, 테스트 용도로 Pod 배포를 인위적으로 느리게 해서 배포 과정을 살펴보기 편하게 하도록 하겠다. 

minReadySeconds를 5로 줬는데, Pod가 배포되고 서비스를 시작하는데 5초 정도 딜레이를 준다. 5초 단위로, Pod 가 배포 되는 과정을 살펴볼 수 있다.

다음 이 Pod들 앞에 Service를 배포해보자. 아래는 hello-deployment-service.yaml 이다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-deployment-svc
spec:
  selector:
    app: hello-deployment
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 8080
  type: LoadBalancer
```

<br>

**Deployment, Service를 배포**

YAML 파일 작성이 끝났으면, Deployment와 Service를 배포해보자.

배포가 끝난후에, kubectl get pod 로 확인해보면 아래와 같이 pod 가 3개가 올라온것을 확인할 수 있다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/9938CF3C5B3717AB0F"></center>

Service의 IP를 확인한후 curl 명령을 이용해서 요청을 보내보면 아래와 같이 version 1 서버로 요청이 가는 것을 확인할 수 있다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/994D7E505B3717AB21"></center>

<br>

**업데이트**

그러면 v2 이미지를 배포해보자. 여러가지 방법이 있는데, kubectl set image deployment 라는 명령을 이용하면 이미지를 새 이미지로 변경할수 있다. 다음 명령어를 실행해보자 

```shell
%kubectl set image deployment hello-deployment hello-deployment=[gcr.io/terrycho-sandbox/deployment:v2](http://gcr.io/terrycho-sandbox/deployment:v2)
```

명령어를 실행하면 v1 → v2로 pod를 하나씩 롤링 업데이트를 한다. 

위의 명령을 실행해놓고 kubectl get pod 명령을 실행해보면 아래 그림과 같이 pod를 하나씩 지워가면서 새로운 pod를 생성하는 것을 확인할 수 있다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/9948F5405B3717AB31"></center>

배포가 끝난후에, kubectl describe deploy hello-deployment 명령을 실행해서  deployment 내용을 살펴보자.

아래 내용을 보면 Replica Set replica set hello-deployment-68bd497896의 replica수를 3-->0 개로 줄이고, 새 버전의 RS인 replica set hello-deployment-5756bb6c8f의 replica 수를 1-->3으로 올리는 것을 확인할 수 있다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99DA144A5B3717AB0F"></center>

잘 보면 Replication Controller가 아니라 Replication Set (RS)을 사용하는 것을 볼 수 있는데, Deployment는 RC대신 RS를 사용한다.

그리고 하나 주목할만한점은 Pod의 이름은 [deployment 이름]-[RS의 해쉬 #]-[Pod의 해쉬#] 를 사용한다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/9986E6385B3717AB23"></center>

배포가 끝난 후에 확인해보면 위처럼 Pod의 이름이 hello-deployment-5756bb6c8f-* 으로, 새로 배포된 RS 의 이름과 동일한것을 볼 수 있다.

set image 명령은 기존 배포된 deployment의 이미지를 변경하도록 하지만, Deployment의 이미지 수정과 같이 쿠버네티스의 객체 (Object)의 정보를 수정하는 방법은 여러가지 방법이 있다. 

<br>

**객체 설정 업데이트**

간단하게 쿠버네티스 객체 정보를 수정하는 방법을 살펴보자.

<br>

**[1] kubectl edit**

edit 명령은 리소스의 설정 정보를 kubectl 이 설치되어 있는 머신의 에디터를 이용해서 에디트할 수 있다.

예를 들어 hello-deployment 라는 이름의 deployment 리소스의 설정을 에디트 하고 싶으면, 

```shell
%kubectl edit deploy hello-deployment 
```

명령을 실행하면, (맥북 기준) vi 에디터가 실행되고, hello-deployment  설정 정보를 vi 에디터로 편집할 수 있게 해준다. 에디터에서 설정을 변경한 후 저장하면, 그 객체의 설정값이 업데이트 된다. 

아래는 kubectl edit deploy hello-deployment 를 이용하여 컨테이너 이미지의 태그를 v2로 변경한 예이다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99FF353F5B3717AB08"></center>

  설정을 변경하는데 유용하게 사용할 수 있지만, 특정 리소스의 설정 정보를 에디터에서 보여주기 때문에, 상세한 설정 (yaml)을 보는데 유용하게 이용할 수 있다.

<br>

**[1] kubectl patch**

patch 명령은 파일 업데이트 없이 기존 리소스의 설정 정보 중 여러개의 필드를 수정하고자 하는데 이용한다.

```shell
%kubectl patch [리소스 종류] [리소스명] --patch ‘[YAML이나 JSON 포맷으로 변경하고자 하는 설정]’
```

을 넣어주면 된다. 예를 들어 아래 명령은 deployment 중에 hello-deployment 리소스에 대해서 image 명을  image: gcr.io/terrycho-sandbox/deployment:v2 로 변경한 예이다.

```shell
%kubectl patch deployment hello-deployment --patch 'spec:\n template:\n  spec:\n containers:\n - name: hello-deployment\n image: gcr.io/terrycho-sandbox/deployment:v2’
```

특히 [YAML] 설정은 한줄에 써야 하는 만큼 띄어쓰기에 조심하는 것이 좋다.



patch나 edit의 경우 쉽게 설정을 업데이트할 수 있는 장점은 있지만, 업데이트에 대한 히스토리를 추적하기 어려운 만큼 

가급적이면, 새로운 파일을 생성하고,파일을 replace를 통해서 적용하는 것이 파일 단위로 변경 내용을 추적할 수 있기 때문에 이 방법을 권장한다.

<br>

**롤백**

Deployment은 롤링 업데이트이외에, 롤백도 손쉽게 지원한다.

먼저 배포된 버전을 체크해보려면 kubectl rollout history deployment/[Deployment 이름]을 실행하면 기존에 배포된 버전을 확인할 수 있다. 디폴트로는 2개의 버전을 유지하게 되어 있다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99F8D74C5B3717AA03"></center>

1버전으로 롤백을 하려면 kubectl rollout undo deployment [ deployment 명 ] --to-revision=[롤백할 버전명] 을 하면, 그 버전으로 롤백이 된다. 

명령을 실행하면 아래와 같이 이전 버전으로 롤백이 되는 것을 확인할 수 있다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/992634335B3717AB10"></center>

롤백등을 위해서 Deployment를  RS를 여러 버전을 유지하고 있다. 유지하는 버전의 수는  Deployment에서 설정이 가능하다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/992CDB3D5B3717AA10"></center>

<br>

**Version(Tag)**

배포 이야기가 나왔기 때문에, 버전/태그에 대해서 언급하도록 하겠다.

쿠버네티스에서 사용되는 컨테이너 이미지 명은 [gcr.io/terrycho-sandbox/deployment:v2](http://gcr.io/terrycho-sandbox/deployment:v2) 와 같이 [컨테이너 이미지명]:[태그] 의 조합으로 구성되는데, 태그는 일반적으로 버전을 사용한다.

주의할 점은 컨테이너를 새로 빌드할때마다 버전을 올리도록 해야, 나중에 구분이 가능하기 때문에 컨테이너를 빌드할때 마다 다른 태그를 사용하는 것을 습관화할 필요가 있다.

새롭게 빌드한 컨테이너 이미지를 같은 태그를 사용하는 것은 향후에 어떤 내용이 변경이 되었는지를 추적하기 어렵기 때문에 좋은 방법은 아니다.

- "latest" 태그

   쿠버네티스에서 사용되는 도커 컨테이너 이미지 중에 latest 라는 특별한 의미를 가지는 태그가 있는데, docker pull [gcr.io/terrycho-sandbox/deployment](http://gcr.io/terrycho-sandbox/deployment:v2) 과 같은 명령으로 컨테이너 이미지에 태그명을 정의하지 않으면 디폴트로 latest라고 태그된 이미지를 가지고 오도록 되어 있다. 

  보통 최신 이미지에 이 latest 태그를 추가한다. (하나의 컨테이너는 두 개 이상의 이미지를 가질 수 있다.)

  이렇게 하면 latest 태그를 이용하여 항상 최신 이미지를 읽어올 수 있는 장점이 있을 수 있으나, 불러온 컨테이너 이미지의 버전을 정확히 알기가 어렵기 때문에 권장하는 방법이 아니다. 

  쿠버네티스에서 컨테이너 이미지를 가지고 올때는 가급적이면 latest 태그를 사용하는 것 보다는 명시적으로 배포하고자 하는 버전을 태그로 정의하는 것을 권장한다. 



<hr>

<br>

이번 글에서는 Deployment에 대해 알아보았다.

다음 포스트는 환경에 따라서 다른 설정값을 사용하는 경우에 어떻게 쉽게 환경 변수와 변수들을 관리 할 수 있는지 ConfigMap과 Secret에 대해 알아보자.

<br>

<hr>
<br>



