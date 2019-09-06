---
layout: post
title:  "Deploying Services Using Replication Controllers"
date:   2019-09-06 10:00:36
Categories: node.js RC 
---

node.js로 간단한 웹서버를 만들어 도커로 패키징을 진행한다.

이후에  Pod과 ReplicationController를 생성하여 서비스를 배포한다.

<br>

**도커 파일 만들기**

실습을 진행하기 위해서 로컬 환경에 도커와, node.js 가 설치되어 있어야 한다. 이 두 부분은 생략하도록 한다.

여기서 사용한 실습 환경은 node.js carbon 버전 (8.11.3), 도커 맥용 18.05.0-ce, build f150324 을 사용하였다.

**[1] node.js Application 준비**

server.js라는 이름으로 아래 코드를 작성한다.

```javascript
var os = require('os');
var http = require('http');

var handleRequest = function(request, response) {
  response.writeHead(200);
  response.end("Hello World! I'm "+os.hostname());

  //log
  console.log("["+
                Date(Date.now()).toLocaleString()+
                "] "+os.hostname());
}

var www = http.createServer(handleRequest);
www.listen(8080);
```

이 코드는 8080 포트로 웹서버를 띄워서 접속하면 “Hello World!” 문자열과 함께, 서버의 호스트명을 출력해준다. 그리고 stdout에 로그로, 시간과 서버의 호스트명을 출력해준다.

코드 작성이 끝났으면, 서버를 실행해보자

```shell
%node server.js
```

브라우저에 접속하면 다음과 같은 결과를 얻을 수 있다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99C0E34B5B25C3AE33"></center>

이후 콘솔화면에서는 아래와 같이 시간과 호스트명이 로그로 함께 출력된다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99CE61385B25C3AE20"></center>

<br>

**[2] 도커로 패키징하기**

그러면 이 node.js 애플리케이션을 도커 컨테이너로 패키징 해보자

Dockerfile 이라는 파일을 만들고 아래 코드를 작성한다.

```dockerfile
FROM node:carbon
EXPOSE 8080
COPY server.js .
CMD node server.js > log.out
```

이 코드는 node.js carborn (8.11.3) 컨테이너 이미지를 베이스로 한후에,  앞서 작성한 server.js 코드를 복사한후에, node server.js > log.out 명령어를 실행하도록 하는 컨테이너를 만드는 설정파일이다.

설정 파일이 준비되었으면,  도커 컨테이너 파일을 만들어보자

```shell
% docker build -t gcr.io/ann-sandbox/hello-node:v1 .
```

docker build  명령은 컨테이너를 만드는 명령이고, -t는 빌드될 이미지에 대한 태그를 정하는 명령이다.

빌드된 컨테이너 이미지는 gcr.io/terrycho-sandbox/hello-node로  태깅되는데, 이는 향후에 구글 클라우드 컨테이너 레지스트리에 올리기 위해서 태그 명을 구글 클라우드 컨테이너 레지스트리의 포맷을 따른 것이다. ( https://cloud.google.com/container-registry/docs/pushing-and-pulling) 

포맷은 [HOST_NAME]/[GOOGLE PROJECT-ID]/[IMAGE NAME] 으로,

 gcr.io/terrycho-sandbox는 도커 이미지가 저장될 리파지토리의 경로를 위의 규칙에 따라 정의한 것이다.

- gcr.io는 구글 클라우드 컨테이너 리파지토리 US 리전을 지칭하며, 
- ann-sandbox는 본인의 구글 프로젝트 ID를 나타낸다.
- 이미지명을 hello-node 로 지정하였다. 
- 마지막으로 콜론(:) 으로 구별되어 정의한 부분은 태그 부분으로, 여기서는 “v1”으로 태깅을 하였다. 

이미지는 위의 이름으로 지정하여 생성되어 로컬에 저장된다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99BF513B5B25C3AE2F"></center>

빌드를 실행하면 위와 같이 node:carbon 이미지를 읽어와서 필요한 server.js 파일을 복사하고 컨테이너 이미지를 생성한다.

컨테이너 이미지가 생성되었으면 로컬 환경에서 이미지를 기동 시켜보자

```shell
%docker run -d -p 8080:8080 gcr.io/ann-sandbox/hello-node:v1 
```

명령어로 컨테이너를 실행할 수 있다. 

- -d 옵션은 컨테이너를 실행하되, 백그라운드 모드로 실행하도록 하였다.
- -p는 포트 맵핑으로 뒤의 포트가 도커 컨테이너에서 돌고 있는 포트이고, 앞의 포트가 이를 밖으로 노출 시키는 포트이다 예를 들어 -p 9090:8080 이면 컨테이너의 8080포트를 9090으로 노출 시켜서 서비스 한다는 뜻이다. 여기서는 컨테이너 포트와 서비스로 노출 되는 포트를 동일하게 8080으로 사용하였다. 

컨테이너를 실행한 후에, docker ps 명령어를 이용하여 확인해보면 아래와 같이 hello-node:v1 이미지로 컨테이너가 기동중인것을 확인할 수 있다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/996F0D3F5B25C3AE28"></center>

다음 브라우져를 통해서 접속을 확인하기 위해서 localhost:8080으로 접속해보면 아래와 같이 Hello World 와 호스트명이 출력되는 것을 확인할 수 있다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99A93E4E5B25C3AE0F"></center>

로그가 제대로 출력되는지 확인하기 위해서 컨테이너 이미지에 쉘로 접속해보자

접속하는 방법은

```shell
% docker exec -i -t [컨테이너 ID] /bin/bash
```

를 실행하면 된다. 컨테이너 ID 는 앞의 docker ps 명령을 이용하여 기동중인 컨테이너 명을 보면 처음 부분이 컨테이너 ID이다. 

hostname 명령을 실행하여 호스트명을 확인해보면 위에 웹 브라우져에서 출력된 41a293ba79a7과 동일한것을 확인할 수 있다. 디렉토리에는 server.js 파일이 복사되어 있고, log.out 파일이 생성된것을 볼 수 있다.  

cat log.out을 이용해서 보면, 시간과 호스트명이 로그로 출력된것을 확인할 수 있다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99E91C3C5B25C3AE26"></center>

<br>

<hr>

<br>

**쿠버네티스 클러스터 준비**

gloud와 kubectl를 설치한 후에 쿠버네티스 클러스터 인증 정보를 얻기 위해 아래의 명령을 실행한다. 여기는 클러스터 이름이 ann-gke-10이다.

```shell
%gcloud container clusters get-credentials ann-gke-10
```

명령어 설정이 끝났으면 명령이 제대로 동작하는지 확인하기 위해 현재 구글 클라우드 내에 생성된 클러스터 목록을 읽어오는 명령어를 실행한다.

<center><img src = "https://github.com/daeuni/daeuni.github.io/blob/master/assets/cloudlist.PNG?raw=true"></center>

위와같이 ann-gke-10 이름으로 asia-northeast1-c zone에 쿠버네티스 1.12-8.gke.10 버전으로 클러스터가 생성되었고 노드는 총 3개가 실행중인 것을 확인할 수 있다.

<br>

**쿠버네티스 배포하기**

이제 구글 클라우드에 쿠버네티스 클러스터를 생성하였고, 사용을 하기 위한 준비가 되었다.

앞에서 만든 도커 이미지를 패키징 하여, 이 쿠버네티스 클러스터에 배포해보도록 하자.

여기서는 도커 이미지를 구글 클라우드내의 도커 컨테이너 레지스트리에 등록한 후, 이 이미지를 이용하여 ReplicationController를 통해 총 3개의 Pod를 구성하고 서비스를 만들어서 이 Pod들을 외부 IP를 이용하여 서비스를 제공할 것이다. 


<br>
**도커 컨테이너 이미지 등록하기**

먼저 앞에서 만든 도커 이미지를 구글 클라우드 컨테이너 레지스트리(Google Container Registry 이하 GCR) 에 등록해보자. 

GCR은 구글 클라우드에서 제공하는 컨테이너 이미지 저장 서비스로, 저장 뿐만 아니라, CI/CD 도구와 연동하여, 자동으로 컨테이너 이미지를 빌드하는 기능, 그리고 등록되는 컨테이너 이미지에 대해서 보안적인 문제가 있는지 보안 결함을 스캔해주는 기능과 같은 다양한 기능을 제공한다.

컨테이너 이미지를 로컬환경에서 도커 컨테이너 저장소에 저장하려면 docker push라는 명령을 사용하는데, 여기서는 GCR을 컨테이너 이미지 저장소로 사용할 것이기 때문에, GCR에 대한 인증이 필요하다.

인증은 한번만 해놓으면 되는데 

```shell
%gcloud auth configure-docker
```

명령을 이용하면, 인증 정보가 로컬 환경에 자동으로 저장된다.

인증 후에 docker push 명령어를 이용하여 이미지를 GCR에 저장한다.

```shell
%docker push gcr.io/ann-sandbox/hello-node:v1
```

명령어를 실행하면, GCR에 hello-node 이미지가 v1 태그로 저장된다.

GCR에 잘 저장되었는지 확인하려면 구글 클라우드 콘솔에 Container Registry(GCR) 메뉴에서 Images를 들어가면 이미지가 잘 등록되었는지 확인할 수 있다.

<br>

<hr>

<br>

**ReplicationController 생성**

컨테이너 이미지가 등록되었으면 이 이미지를 이용해서 Pod를 생성해보자.

Pod 생성은 Replication Controller (이하 rc)를 생성하여, rc가 Pod 생성 및 컨트롤을 하도록 한다.

다음은 rc 생성을 위한 hello-node-rc.yaml 파일이다.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: hello-node-rc
spec:
  replicas: 3
  selector:
    app: hello-node
  template:
    metadata:
      name: hello-node-pod
      labels:
        app: hello-node
    spec:
      containers:
      - name: hello-node
        image: gcr.io/ann-sandbox/hello-node:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
```

hello-node-rc 라는 이름으로 rc를 생성하는데, replica 를 3으로 하여, 총 3개의 pod를 생성하도록 한다. 

템플릿 부분에 컨테이너 스팩에 컨테이너 이름은 hello-node로 하고 이미지는 앞서 업로드한 gcr.io/terrycho-sandbox/hello-node:v1 를 이용해서 컨테이너를 만들도록 한다. 컨테이너의 포트는 8080을 오픈한다. 템플릿 부분에서 app 이라는 이름의 라벨을 생성하고 그 값을 hello-node로 지정하였다. 

이 라벨은 나중에 서비스 (service)에 의해 외부로 서비스될 pod들을 선택하는데 사용 된다. 

여기서 imagePullPolicy:Always  라고 설정한 부분이 있는데, 이는 Pod를 만들때 마다 매번 컨테이너 이미지를 확인해서 새 이미지를 사용하도록 하는 설정이다.  

컨테이너 이미지는 한번 다운로드가 되면 노드(Node) 에 저장이 되어 있게 되고, 사용이 되지 않는 이미지 중에 오래된 이미지는 Kublet이 가비지 컬렉션 (Garbage collection) 정책에 따라 이미지를 삭제하게 되는데, 문제는 노드에 이미 다운되어 있는 이미지가 있을 경우 컨테이너 생성시 노드에 이미 다운로드 되어 있는 이미지를 사용한다는 것이다. 

컨테이너 리파지토리에 같은 이름으로 이미지를 업데이트 하거나 심지어 그 이미지를 삭제하더라도 노드에 이미지가 이미 다운로드 되어 있으면 다운로드된 이미지를 사용하기 때문에, 업데이트 부분이 반영이 안된다.

이를 방지하기 위해서 imagePullPolicy:Always로 해주면 컨테이너 생성시마다 이미지 리파지토리를 검사해서 새 이미지를 가지고 오기 때문에, 업데이트된 내용을 제대로 반영할 수 있다.

이제 yaml파일을 실행시켜보자.

```shell
%kubectl create -f hello-node-rc.yaml
```

명령어를 실행해서 rc와 pod를 생성한다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99D75D355B25C3AE26"></center>

위의 그림과 같이 3개의 Pod가 생성된것을 확인할 수 있는데, Pod가 제대로 생성되었는지 확인하기 위해서 hello-node-rc-rsdzl pod에서 hello-node-rc-2phgg pod의 node.js 웹서버에 접속을 해볼 것이다.

아직 서비스를 붙이지 않았기 때문에, 이 pod들은 외부 ip를 이용해서 서비스가 불가능하다.

따라서, 쿠버네티스 클러스터 내부의 pod를 이용하여 내부 ip (private ip)간에 통신을 해보기 위해서 pod에서 pod를 호출 하는 것이다. 

kubectl describe pod  [pod 명] 명령을 이용하면, 해당 pod의 정보를 볼 수 있다. hello-node-rc-2hpgg pod의 cluster ip (내부 ip)를 확인해보면 10.20.1.27  인것을 확인할 수 있다.

kubectl exec 명령을 이용하면 쉘 명령어를 실행할 수 있는데, 다음과 같이 hello-node-rc-rsdzl pod에서 첫번째 pod인 hello-node-rc-2phgg의 ip인 10.20.1.27의 8080 포트로 curl 을 이용해 HTTP 요청을 보내보면 다음과 같이 정상적으로 응답이 오는 것을 볼 수 있다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/9953853D5B25C3AE31"></center>



**Service 등록 **

rc와 pod 생성이 끝났으면 이제 서비스를 생성해서 pod들을 외부 ip로 서비스 해보자

다음은 서비스를 정의한 hello-node-svc.yaml 파일이다. 

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

 Selector 부분에 app:hello-node 로 지정하여, pod들 중에 라벨의 키가 app이고 값이 hello-node인 pod 들만 서비스에 연결하도록 지정하였다. 다음 서비스의 포트는 80으로 지정하였고, pod의 port는 8080으로 지정하였다. 

<center><img src = "https://github.com/daeuni/daeuni.github.io/blob/master/assets/cloudservice1.png?raw=true"></center>

서비스가 배포되면 위와 같은 구조를 가진다. 

아래 명령을 통해 서비스를 생성하자.

```shell
%kubectl create -f hello-node-svc.yaml
```

다음 생성된 서비스의 외부 ip를 얻기 위해서 kubectl get svc 명령을 실행해보자

아래 그림과 같이 <u>35.200.40.161</u> IP가 할당된것을 확인할 수 있다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/991622405B25C3AE25"></center>

해당 IP로 접속을 해보면 아래와 같이 정상적으로 응답이 오는 것을 알 수 있다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99B8BA4B5B25C3AE32"></center>

<br>

**Replication Controller Test**

rc는 pod의 상태를 체크하다가 문제가 있으면 다시, pod를 기동해주는 기능을 한다. 

이를 테스트하기 위해서 강제적으로 모든 pod를 제거해보자. kubectl delete pod --all을 이용하면 모든 pod를 제거할 수 있다.

아래 그림을 보면, 모든 pod를 제거했더니 3개의 pod가 제거되고 새롭게 3개의 pod가 기동되는 것을 확인할 수 있다. RC가 제 역할을 톡톡이 잘하고 있다 ! 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99EA15495B25C3AE0B"></center>

<br>

운영중에 탄력적으로 pod의 개수를 조정할 수 있는데, kubectl scale 명령을 이용하면 된다.

kubectl scale --replicas=[pod의 수] rc/[rc 명] 식으로 사용하면 된다. 아래는 pod의 수를 4개로 재 조정한 내용이다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99A481385B25C3AE23"></center>

<br>

**자원 정리**

테스트가 끝났으면 서비스, rc,pod를 삭제해보자. 

- 서비스 삭제는 kubectl delete svc --all 명령어를 이용한다.
- rc 삭제는 kubectl delete rc --all
- pod 삭제는 kubectl delete pod --all

<br>

삭제시 주의할점은 pod를 삭제하기 전에 먼저 rc를 삭제해야 한다. 아니면, pod가 삭제된 후 rc에 의해서 다시 새로운 pod가 생성될 수 있다. 


<br>
<hr>

<br>

이번 글에서는 간단한 node.js application 을 docker image로 만들어 쿠버네티스로 배포해보았다.

조금 더 쿠버네티스를 자알 쓰기 위해 다음 포스트부터는 하나씩 쿠버네티스를 자세히 파해쳐 볼 것이다.

<br>

<hr>
<br>



