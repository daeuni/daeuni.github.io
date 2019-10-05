---
layout: post
title:  "Kubernetes: ConfigMap"
date:   2019-09-30 10:00:36

---

애플리케이션을 배포하다 보면 환경에 따라서 다른 설정값을 사용하는 경우가 있다. 

예를 들어, 데이타베이스의 IP, API를 호출하기 위한 API KEY, 개발/운영에 따른 디버그 모드, 환경 설정 파일들이 있다. 애플리케이션 이미지는 같지만, 이런 환경 변수가 차이가 나는 경우 매번 다른 컨테이너 이미지를 만드는 것은 관리상 불편할 수 밖에 없다. 

이러한 환경 변수나 설정값들을 변수로 관리해서 Pod가 생성될때 이 값을 넣어줄 수 있는데, 이러한 기능을 제공하는 것이 바로 Configmap과 Secret이다. 

아래 그림과 같이 설정 파일을 만들어놓고, Pod 를 배포할때 마다 다른 설정 정보를 반영하도록 할 수 있다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/999C6A3C5B3F3FD414"></center>

Configmap이나 Secret에 정의해놓고 이 정의해놓은 값을 Pod로 넘기는 방법은 크게 두가지가 있다.

- 정의해놓은 값을 Pod의 환경 변수 (Environment variable)로 넘기는 방법
- 정의해놓은 값을 Pod의 디스크 볼륨으로 마운트 하는 방법



<br>

**ConfigMap**

configmap은 앞서 설명한것과 같이 설정 정보를 저장해놓는 일종의 저장소 역할을 한다.

configmap은 키/밸류 형식으로 저장이된다. configmap을 생성하는 방법은 literal (문자)로 생성하는 방법과 파일로 생성하는 방법 두가지가 있다. 이 두가지 방법에 대해 알아보자.

<br>

**[1] Literal**

먼저 간단하게 문자로 생성하는 방법을 알아보자.

키를 “language”로 하고 그 값이 “java”인 configMap을 생성해보자

Kubectl create configmap [configmap 이름] --from-literal=[키]=[값] 

식으로 생성하면 된다. 

아래 명령을 이용하면, hello-cm 이라는 이름의 configMap에 키는 language, 값은 java인 configMap이 생성된다. 

```shell
% kubectl create configmap hello-cm --from-literal=language=java
```

또는 아래와 같이 YAML파일로도 configMap을 생성할 수 있다.

```yaml
hello-cm.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-cm
data:
  language: java
```

데이타 항목에 [키]:[값] 형식으로 라인을 추가하면 여러 개의 값을 하나의 configMap에 저장할 수 있다. 

configmap이 생성되었으면 이 값을 Pod에서 환경 변수로 불러서 사용해보도록 하자.

node.js로 간단한 웹 애플리케이션을 만든후에 “LANGUAGE”라는 환경 변수의 값을 읽어서 출력하도록 할것이다. 

아래와 같이 server.js node.js 애플리케이션을 만든다.

```javascript
var os = require('os');
var http = require('http');

var handleRequest = function(request, response) {
  response.writeHead(200);
  response.end(" my prefered language is "+process.env.LANGUAGE+ "\n");


  //log
  console.log("["+
		Date(Date.now()).toLocaleString()+
		"] "+os.hostname());
}
var www = http.createServer(handleRequest);
www.listen(8080);
```

이 파일을 컨테이너로 패키징한 후에 아래와 같이 Deployment를 정의한다.

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: cm-deployment
spec:
  replicas: 3
  minReadySeconds: 5
  selector:
    matchLabels:
      app: cm-literal
  template:
    metadata:
      name: cm-literal-pod
      labels:
        app: cm-literal
    spec:
      containers:
      - name: cm
        image: gcr.io/terrycho-sandbox/cm:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        env:
        - name: LANGUAGE
          valueFrom:
            configMapKeyRef:
               name: hello-cm
               key: language
```

configMap에서 데이타를 읽는 부분은 맨 아래에 env 부분인데, env 부분에 환경 변수를 정의한다.

name은 LANGUAGE라는 이름으로 정의하고 데이타는 valueFrom을 이용해서 configMap에서 읽어오도록 하였다. 

name에는 configMap의 이름인 hello-cm을, 그리고 읽어오고자 하는 데이타는 키 값이 “language”인 값을 읽어오도록 하였다. 

이렇게 하면 LANGUAGE 환경 변수에, configMap에 “language” 로 저장된 “java”라는 문자열을 읽어오게 된다.

이 스크립트를 이용하여 Deployment를 생성한 후에, 이 Deployment 앞에 Service (Load balancer)를 붙여 보자.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cm-literal-svc
spec:
  selector:
    app: cm-literal
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 8080
  type: LoadBalancer
```

  서비스가 생성이 되었으면 웹 브라우져에서 해당 Service의 URL을 접속해보자.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/995515495B3F3FD421"></center>

  위와 같이 환경 변수에서 “java”라는 문자열을 읽어와서 출력한것을 확인할 수 있다.

<br>

**[2] File**

위와 같이 개개별 값을 공유할 수 도 있지만, 설정을 파일 형태로 해서 Pod에 공유하는 방법도 있다. 

예제를 보면서 이해하도록 하자. 

profile.properties라는 파일이 있고 파일 내용이 아래와 같다고 하자.

```shell
myname=terry
email=myemail@mycompany.com
address=seoul
```

파일을 이용해서 ConfigMap을 만들때는 아래와 같이 --from-file 을 이용해서 파일명을 넘겨주면 된다. 

```shell
%kubectl create configmap cm-file --from file=./properties/profile.properties
```

이렇게 파일을 이용해서 configMap을 생성하면, 아래와 같이 키는 파일명이 되고, 값은 파일 내용이 된다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99BD0E3E5B3F3FD42D"></center>

<br>

**[1] 환경변수로 값 전달하기**

생성된 configMap 내의 값을 Pod로 전달하는 방법은 앞에서 예를든 것과 같이 환경 변수로 넘길 수 있다.

아래 Deployment 예제를 보자.

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: cm-file-deployment
spec:
  replicas: 3
  minReadySeconds: 5
  selector:
    matchLabels:
      app: cm-file
  template:
    metadata:
      name: cm-file-pod
      labels:
        app: cm-file
    spec:
      containers:
      - name: cm-file
        image: gcr.io/terrycho-sandbox/cm-file:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        env:
        - name: PROFILE
          valueFrom:
            configMapKeyRef:
               name: cm-file
               key: profile.properties
```

cm-file configMap에서 키가 “profile.properties” (파일명)인 값을 읽어와서 환경 변수 PROFILE에 저장한다. 저장된 값은 파일의 내용인 아래 문자열이 된다.

```shell
myname=terry
email=myemail@mycompany.com
address=seoul
```

혼동하지 말아야 하는 점은 profile.properties 파일안에 문자열이 myname=terry 처럼 키/밸류 형식으로 되어 있다고 하더라도, myname 을 키로 해서 terry라는 값을 가지고 오는 것처럼 개개별 문자열을 키/밸류로 인식하는 것이 아니라 전체 파일 내용을 하나의 문자열로 처리한다는 점이다.

<br>

**[2] 디스크 볼륨으로 마운트하기**

configMap의 정보를 pod로 전달하는 방법은 앞에 처럼 환경 변수를 사용하는 방법도 있지만, Pod의 디스크 볼륨으로 마운트 시키는 방법도 있다. 

앞의 cm-file configMap을 /tmp/config/에 마운트 해보도록 하자. 

아래와 같이 Deployment 스크립트를 작성한다. 

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: cm-file-deployment-vol
spec:
  replicas: 3
  minReadySeconds: 5
  selector:
    matchLabels:
      app: cm-file-vol
  template:
    metadata:
      name: cm-file-vol-pod
      labels:
        app: cm-file-vol
    spec:
      containers:
      - name: cm-file-vol
        image: gcr.io/terrycho-sandbox/cm-file-volume:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        volumeMounts:
          - name: config-profile
            mountPath: /tmp/config
      volumes:
        - name: config-profile
          configMap:
            name: cm-file
```

configMap을 디스크 볼륨으로 마운트해서 사용하는 방법은 volumes 을 configMap으로 정의하면 된다. 

위의 예제에서 처럼 volume을 정의할때, configMap으로 정의하고 configMap의 이름을 cm-file로 정의하여, cm-file configMap을 선택하였다. 이 볼륨을 volumeMounts를 이용해서 /tmp/config에 마운트 되도록 하였다.

이때 중요한점은 마운트 포인트에 마운트 될때, 파일명이 configMap내의 키가 파일명이 된다. 

다음 테스트를 위해서 server.js 애플리케이션에 /tmp/config/profile.properties 파일을 읽어서 출력하도록 아래와 같이 코드를 작성한다.

```javascript
var os = require('os');
var fs = require('fs');
var http = require('http');

var handleRequest = function(request, response) {
  fs.readFile('/tmp/config/profile.properties',function(err,data){
    response.writeHead(200);
    response.end("Read configMap from file  "+data+" \n");
  });

  //log
  console.log("["+
		Date(Date.now()).toLocaleString()+
		"] "+os.hostname());
}

var www = http.createServer(handleRequest);
www.listen(8080);
```

이 server.js를 도커로 패키징해서 배포한후, service를 붙여서 테스트해보면 다음과 같은 결과를 얻을 수 있다. 

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99FB3B3B5B3F3FD413"></center>

파일 내용이 출력되는 것을 확인할 수 있다.

디스크에 마운트가 제대로 되었는지를 확인하기 위해서 Pod에 쉘로 로그인해서 확인해보자.

<center><img src = "https://t1.daumcdn.net/cfile/tistory/99293B4E5B3F3FD40E"></center>

그림과 같이 /tmp/config/profile.properties 파일이 생성된 것을 확인할 수 있다.

<hr>
<br>

이번 글에서는 환경에 따라서 설정값을 사용하기 위한 방법 중에서 ConfigMap에 대해 알아보았다.

다음 포스트는 Secret에 대해 알아보자!

<br>

<hr>
<br>



