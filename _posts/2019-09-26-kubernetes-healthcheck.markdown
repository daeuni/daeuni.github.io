---
layout: post
title:  "Kubernetes: Health Check"
date:   2019-09-26 10:00:36

---

쿠버네티스는 각 컨테이너의 상태를 주기적으로 체크해서, 문제가 있는 컨테이너를 자동으로 재시작하거나 문제가 있는 컨테이너를 서비스에서 제외할 수 있다. 

이러한 기능을 Health Check라고 하는데 크게 두 가지 방법이 있다.

컨테이너가 살아 있는지 아닌지를 체크하는 방법이 Liveness probe, 그리고 컨테이너가 서비스가 가능한 상태인지를 체크하는 방법을 Readiness probe 라고 한다

<br>

**Probe Types**

Liveness probe와 readiness probe는 컨테이너가 정상적인지 아닌지를 체크하는 방법으로, 다음과 같이 3가지 방식을 제공한다.

- Command probe
- HTTP probe
- TCP probe

그럼 각각에 대해서 살펴보자.



**[1] Command probe**

  Command probe는 컨테이너의 상태 체크를 쉘 명령을 수행하고 나서, 그 결과를 가지고 컨테이너의 정상여부를 체크한다. 

쉘 명령어를 수행한 후, 결과값이 0 이면 성공, 0이 아니면 실패로 간주한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-pod
spec:
  containers:
  - name: liveness
    image: gcr.io/terrycho-sandbox/liveness:v1
    imagePullPolicy: Always
    ports:
    - containerPort: 8080
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
```

Readiness probe 또는 liveness probe 부분에 exec: 으로 정의하고, command: 아래에 실행하고자 하는 쉘 명령어에 대한 인자를 기술한다.

이 쉘명령이 성공적으로 실행되서 0을 리턴하면, probe를 정상으로 판단한다. 

<br>

**[2] HTTP probe**

가장 많이 사용하는 probe 방식으로 HTTP GET을 이용하여, 컨테이너의 상태를 체크한다. 

지정된 URL로 HTTP GET 요청을 보내서 리턴되는 HTTP 응답 코드가 200~300 사이면 probe를 정상으로 판단하고, 그 이외의 값일 경우에는 비정상으로 판단한다. 

아래는 HTTP probe를 이용한 readiness probe를 정의한 예제이다. 

```yaml
metadata:
  name: readiness-rc
spec:
  replicas: 2
  selector:
    app: readiness
  template:
    metadata:
      name: readiness-pod
      labels:
        app: readiness
    spec:
      containers:
      - name: readiness
        image: gcr.io/terrycho-sandbox/readiness:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /readiness
            port: 8080
```

liveness 또는 readinessProbe  항목 아래에 httpGet이라는 이름으로 정의하고, path에  HTTP GET을 보낼 URL을 그리고, port에는 HTTP GET을 보낼 port 를 지정한다.

일반적인 HTTP 서비스를 보내는 port와 HTTP readiness를 서비스 하는 포트를 분리할 수 있는데, HTTP GET 포트가 외부에 노출될 경우에는 DDos 공격등을 받을 수 있는 가능성이 있기 때문에, 필요하다면 서비스 포트와 probe 포트를 분리해서 구성할 수 있다. 

<br>

**[3] TCP probe**

  마지막으로 TCP probe는 지정된 포트에 TCP 연결을 시도하여, 연결이 성공하면, 컨테이너가 정상인것으로 판단한다. 다음은 tcp probe를 적용한 liveness probe의 예제이다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-pod-tcp
spec:
  containers:
  - name: liveness
    image: gcr.io/terrycho-sandbox/liveness:v1
    imagePullPolicy: Always
    ports:
    - containerPort: 8080
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

Tcp probe는 간단하게 livenessProbe나 readinessProbe 아래 tcpSocket이라는 항목으로 정의하고 그 아래 port 항목에 tcp port를 지정하면 된다. 

이 포트로 TCP 연결을 시도하고, 이 연결이 성공하면 컨테이너가 정상인것으로 실패하면 비정상으로 판단한다. 

실제로 Liveness Probe와 Readiness Probe를 예제를 통해서 조금 더 상세하게 살펴보도록 하자. 

<br>

**Liveness Probe**

Liveness probe는 컨테이너의 상태를 주기적으로 체크해서, 응답이 없으면 컨테이너를 자동으로 재시작해준다. 컨테이너가 정상적으로 기동중인지를 체크하는 기능이다.

Liveness probe는 Pod의 상태를 체크하다가, Pod의 상태가 비정상인 경우 kubelet을 통해서 재 시작한다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99621F3D5B2FA1AE1D"></center>

이해를 돕기 위해서 예제를 하나 살펴보자.

node.js 애플리케이션을 기동하는 컨테이너를 만들어서 배포 하도록 한다. node.js는 앞에서 사용한 애플리케이션과 동일한 server.js  애플리케이션을 사용한다.

Health Check를 하는 방법은 여러가지가 있지만, 컨터이너에서 “cat /tmp/healthy” 명령어를 실행해서 성공하면 컨테이너를 정상으로 판단하고 실패하면 비정상으로 판단하도록 하겠다. 

이를 위해서 컨테이너 생성시에 /tmp/ 디렉토리에 healthy 파일을 복사해 놓도록 한다.

heatlhy 파일의 내용은 아래와 같다.

```shell
i'm healthy
```

파일만 존재하면 되기 때문에 내용은 크게 중요하지 않다.

Dockerfile을 다음과 같이 작성하자.

```dockerfile
FROM node:carbon
EXPOSE 8080
COPY server.js .
COPY healthy /tmp/
CMD node server.js > log.out
```

앞서 작성한 healthy 파일을 /tmp 디렉토리에 복사하였다. 



이제 pod를 정의해보자 다음은 liveness-pod.yaml 파일이다. 여기에 cat /tmp/healthy 명령을 이용하여 컨테이너의 상태를 체크하도록 하였다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-pod
spec:
  containers:
  - name: liveness
    image: gcr.io/terrycho-sandbox/liveness:v1
    imagePullPolicy: Always
    ports:
    - containerPort: 8080
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

컨테이너가 기동 된후 initialDelaySecond에 설정된 값 만큼 대기를 했다가 periodSecond 에 정해진 주기 단위로 컨테이너의 헬스 체크를 한다. 

initialDelaySecond를 주는 이유는, 컨테이너가 기동 되면서 애플리케이션이 기동되는데, 설정 정보나 각종 초기화 작업이 필요하기 때문이다.

컨테이너가 기동되자 마자 헬스 체크를 하게 되면, 서비스할 준비가 되지 않았기 때문에 헬스 체크에 실패할 수 있기 때문에, 준비 기간을 주는 것이다. 준비 시간이 끝나면, periodSecond에 정의된 주기에 따라 헬스 체크를 진행하게 된다.

<br>

헬스 체크 방식은 여러가지가 있는데, 
<u>(1) HTTP 를 이용하는 방식 (2) TCP를 이용하는 방식 (3) 쉘 명령어를 이용하는 방식</u> 3가지가 있다. 

이 예제에서는 쉘 명령을 이용하는 방식을 사용하였다. 

“cat /tmp/healty” 라는 명령을 사용하였고, 이 명령 실행이 성공하면 이 컨테이너를 정상이라고 판단하고, 만약 이 명령이 실패하면 컨테이너가 비정상이라고 판단한다. 

앞서 작성한 Dockerfile을 이용해서 컨테이너를 생성한 후, 이 컨테이너를 리파지토리에 등록하자.

다음 앞에서 작성한 liveness-prod.yaml 파일을 이용하여 Pod를 생성해보자.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99AD7E505B2FA1AE1C"></center>

다음 테스트를 위해 /tmp/healthy 파일을 인위적으로 삭제해보자.

파일을 삭제하면 위의 그림과 같이 cat /tmp/healthy 는 exit code 1 을 내면서 에러로 종료된다.

수 초 후에, 해당 컨테이너가 재시작되는데, kubectl get pod 명령을 이용하여 pod의 상태를 확인해보면 다음과 같다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/995D373E5B2FA1AE1B"></center>

  위의 그림과 같이 중간에 “Killing container with id docker://liveness:Container failed liveness probe.. Container will be killed and recreated.” 메세지가 나오면서 liveness probe 체크가 실패하고, 컨테이너를 재 시작하는 것을 확인할 수 있다.

<br>

**Readiness probe**

컨테이너의 상태 체크중에 liveness의 경우에는 컨테이너가 비정상적으로 작동이 불가능한 경우도 있지만 Configuration을 로딩하거나, 많은 데이타를 로딩하거나, 외부 서비스를 호출하는 경우에는 일시적으로 서비스가 불가능한 상태가 될 수 있다. 

이런 경우에는 컨테이너를 재시작한다 하더라도 정상적으로 서비스가 불가능할 수 있다. 이런 경우에는 컨테이너를 일시적으로 서비스가 불가능한 상태로 마킹해주면 되는데, 이러한 기능은 쿠버네티스의 서비스와 함께 사용하면 유용하게 이용할 수 있다. 

예를 들어 쿠버네티스 서비스에서 아래와 같이 3개의 Pod를 로드밸런싱으로 서비스를 하고 있을때, Readiness probe 를 이용해서 서비스 가능 여부를 주기적으로 체크한다고 하자. 

이 경우 하나의 Pod가 서비스가 불가능한 상태가 되었을때, 즉 Readiness Probe에 대해서 응답이 없거나 실패 응답을 보냈을때는 해당 Pod를 사용 불가능한 상태로 체크하고 서비스 목록에서 제외한다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/991653345B2FA1AE11"></center>

Liveness probe와 Readiness probe의 차이점은 Liveness probe는 컨테이너의 상태가 비정상이라고 판단하면, 해당 Pod를 재시작하는데 반해 

Readiness probe는 컨테이너가 비정상일 경우에는 해당 Pod를 사용할 수 없음으로 표시하고, 서비스등에서 제외한다.

간단한 예제를 보자. 아래 server.js 코드는 /readiness 를 호출하면 파일 시스템내에  /tmp/healthy라는 파일이 있으면 HTTP 응답코드 200 정상을 리턴하고, 파일이 없으면 HTTP 응답코드 500 비정상을 리턴하는 코드이다. server.js 파일을 보자.

```javascript
var os = require('os');
var fs = require('fs');

var http = require('http');
var handleRequest = function(request, response) {
  if(request.url == '/readiness') {
    if(fs.existsSync('/tmp/healthy')){
      // healthy
      response.writeHead(200);
      response.end("Im ready I'm  "+os.hostname() +" \n");
    }else{
      response.writeHead(500);
      response.end("Im not ready I'm  "+os.hostname() +" \n");
    }
  }else{
    response.writeHead(200);
    response.end("Hello World! I'm  "+os.hostname() +" \n");
  }

  //log
  console.log("["+
		Date(Date.now()).toLocaleString()+
		"] "+os.hostname());
}
var www = http.createServer(handleRequest);
www.listen(8080);
```

다음은 replication controller를 정의한다. readiness-rc.yaml로 파일을 만든다.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: readiness-rc
spec:
  replicas: 2
  selector:
    app: readiness
  template:
    metadata:
      name: readiness-pod
      labels:
        app: readiness
    spec:
      containers:
      - name: readiness
        image: gcr.io/terrycho-sandbox/readiness:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /readiness
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

앞의 Liveness probe와 다르게 이번에는 Command probe가 아니라 HTTP 로 체크를 하는 HTTP Probe를 적용해보자.

HTTP Probe는 HTTP GET으로 /readiness URL로 5초마다 호출을 해서 HTTP 응답 200을 받으면 해당 컨테이너를 정상으로 판단하고 200~300 범위를 벗어난 응답 코드를 받으면 비정상으로 판단하여, 서비스 불가능한 상태로 인식해서 쿠버네티스 서비스에서 제외한다.

Replication Controller로 Pod들을 생성하였으면, 이에 대한 로드 밸런서 역할을할 서비스를 배포한다. 파일 이름은 readiness-svc.yaml로 하였다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: readiness-svc
spec:
  selector:
    app: readiness
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 8080
  type: LoadBalancer
```

 서비스가 기동되고 Pod들이 정상적으로 기동된 상태에서 kubectl get pod 명령을 이용해서 현재 Pod 리스트를 출력해보면 다음과 같다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/991C073E5B2FA1AE22"></center>

2개의 Pod가 기동중인것을 확인할 수 있다.

서비스가 기동중인 상태에서 인위적으로 하나의 컨테이너를 서비스 불가 상태로 만들어보자.

앞에서만든 server.js가, 컨테이너 내의 /tmp/healthy 파일의 존재 여부를 체크하기 때문에,  /tmp/healthy 파일을 삭제하면 된다.

아래와 같이 

```shell
%kubectl exec  -it readiness-rc-5v64f -- rm /tmp/healthy 
```

명령을 이용해서 readiness-rc-5v64f pod의 /tmp/healthy 파일을 삭제해보자.

  kubectl describe pod readiness-rc-5v64f 명령을 이용해서 해당 Pod의 상태를 확인할 수 있는데, 아래 그림과 같이 HTTP probe가 500 상태 코드를 리턴받고 Readniess probe가 실패한것을 확인할 수 있다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99B388435B2FA1AE16"></center>

이상태에서 kubectl get pod로 pod 목록을 확인해보자.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/992BBC355B2FA1AE25"></center>

Readiness probe가 실패한 readiness-rc-5v64f 의 상태가 Running이기는 하지만 Ready 상태가 0/1인것으로 해당 컨테이너가 준비 상태가 아님을 확인할 수 있다.

이 Pod들에 연결된 서비스를 여러번 호출해보면 다음과 같은 결과를 얻을 수 있다.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99071D425B2FA1AE14"></center>

  모든 호출이 readiness-rc-5v64f 로 가지않고, 하나 남은 정상적인 Pod인 readiness-rc-89d89 로만 가는 것을 확인할 수 있다.  

<hr>

<br>

이번 글에서는 Health Check의 방법 두가지인 Liveness probe, Readiness probe에 대해 알아보았다.

다음 포스트는 애플리케이션을 배포(Deployment)하는 방법에 대해 알아보자.

<br>

<hr>
<br>



