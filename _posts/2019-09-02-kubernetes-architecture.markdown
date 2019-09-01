---
layout: post
title:  "Kubernetes Architecture"
description: open-source container-orchestration system
date:   2019-09-02 10:00:36

---

컨테이너는 아주 심플하고 우아하게 동작한다. run을 하면 실행되고 stop을 하면 멈춥니다.

서버-클라이언트 구조를 안다면 컨테이너를 관리하는 에이전트를 만들고 중앙에서 API를 이용하여 원격으로 관리하는 모습을 쉽게 그려볼 수 있습니다.

<center><img src="https://subicura.com/assets/article_images/2019-05-19-kubernetes-basic-1/server-agent.png"></center>

쿠버네티스 또한 중앙(Master)에 API 서버와 상태 저장소를 두고 각 서버(Node)의 에이전트(kubelet)와 통신하는 단순한 구조입니다. 하지만, 앞에서 얘기한 개념을 여러 모듈로 쪼개어 구현하고 다양한 오픈소스를 사용하기 때문에 설치가 까다롭고 언뜻 구성이 복잡해 보입니다.

<br>

**Master-Node 구조**

<br>

<center><img src="https://subicura.com/assets/article_images/2019-05-19-kubernetes-basic-1/master-node.png"></center>

쿠버네티스는 전체 클러스터를 관리하는 **마스터**와 컨테이너가 배포되는 **노드**로 구성되어 있습니다. 모든 명령은 마스터의 API 서버를 호출하고 노드는 마스터와 통신하면서 필요한 작업을 수행합니다. 특정 노드의 컨테이너에 명령하거나 로그를 조회할 때도 노드에 직접 명령하는 게 아니라 마스터에 명령을 내리고 마스터가 노드에 접속하여 대신 결과를 응답합니다.

- Master

  마스터 서버는 다양한 모듈이 확장성을 고려하여 기능별로 쪼개져 있는 것이 특징입니다. 관리자만 접속할 수 있도록 보안 설정을 해야 하고 마스터 서버가 죽으면 클러스터를 관리할 수 없기 때문에 보통 3대를 구성하여 안정성을 높입니다. AWS EKS 같은 경우 마스터를 AWS에서 자체 관리하여 안정성을 높였고(마스터에 접속 불가) 개발 환경이나 소규모 환경에선 마스터와 노드를 분리하지 않고 같은 서버에 구성하기도 합니다.

- Node

  노드 서버는 마스터 서버와 통신하면서 필요한 Pod을 생성하고 네트워크와 볼륨을 설정합니다. 실제 컨테이너들이 생성되는 곳으로 수백, 수천대로 확장할 수 있습니다. 각각의 서버에 라벨을 붙여 사용목적(GPU 특화, SSD 서버 등)을 정의할 수 있습니다.

- Kubectl

  API 서버는 json 또는 protobuf 형식을 이용한 http 통신을 지원합니다. 이 방식을 그대로 쓰면 불편하므로 보통 `kubectl`이라는 <u>명령행 도구</u>를 사용합니다. 앞으로 엄청나게 많이 사용할 예정입니다. 어떻게 읽어야 할지 난감한데 공식적으로 큐브컨트롤(cube control)이라고 읽지만 큐브씨티엘, 쿱컨트롤, 쿱씨티엘등도 많이 쓰입니다.

<br>

**Master 구성 요소**

<center><img src="https://subicura.com/assets/article_images/2019-05-19-kubernetes-basic-1/kubernetes-master.png"></center>

- API 서버 kube-apiserver

  API 서버는 모든 요청을 처리하는 마스터의 핵심 모듈입니다. kubectl의 요청뿐 아니라 내부 모듈의 요청도 처리하며 권한을 체크하여 요청을 거부할 수 있습니다. 실제로 하는 일은 원하는 상태를 key-value 저장소에 저장하고 저장된 상태를 조회하는 매우 단순한 작업입니다. Pod을 노드에 할당하고 상태를 체크하는 일은 다른 모듈로 분리되어 있습니다. 노드에서 실행 중인 컨테이너의 로그를 보여주고 명령을 보내는 등 디버거 역할도 수행합니다.

- 분산 데이터 저장소 etcd

  RAFT 알고리즘을 이용한 key-value 저장소입니다. 여러 개로 분산하여 복제할 수 있기 때문에 안정성이 높고 속도도 빠른 편입니다. 단순히 값을 저장하고 읽는 기능뿐 아니라 watch 기능이 있어 어떤 상태가 변경되면 바로 체크하여 로직을 실행할 수 있습니다. 

  클러스터의 모든 설정, 상태 데이터는 여기 저장되고 나머지 모듈은 stateless하게 동작하기 때문에 etcd만 잘 백업해두면 언제든지 클러스터를 복구할 수 있습니다. etcd는 오직 API 서버와 통신하고 다른 모듈은 API 서버를 거쳐 etcd 데이터에 접근합니다. [k3s](https://k3s.io/) 같은 초경량 쿠버네티스 배포판에서는 etcd대신 sqlite를 사용하기도 합니다.

- 스케줄러, 컨트롤러

  API 서버는 요청을 받으면 etcd 저장소와 통신할 뿐, 실제로 상태를 바꾸는 건 스케줄러와 컨트롤러 입니다. 현재 상태를 모니터링하다가 원하는 상태와 다르면 각자 맡은 작업을 수행하고 상태를 갱신합니다.

  - 스케줄러 kube-scheduler

    스케줄러는 할당되지 않은 Pod을 여러 가지 조건(필요한 자원, 라벨)에 따라 적절한 노드 서버에 할당해주는 모듈입니다.

  - 큐브 컨트롤러 kube-controller-manager

    큐브 컨트롤러는 다양한 역할을 하는 아주 바쁜 모듈입니다. 쿠버네티스에 있는 거의 모든 오브젝트의 상태를 관리합니다. 오브젝트별로 철저하게 분업화되어 Deployment는 ReplicaSet을 생성하고 ReplicaSet은 Pod을 생성하고 Pod은 스케줄러가 관리하는 식입니다.

  - 클라우드 컨트롤러 cloud-controller-manager

    클라우드 컨트롤러는 AWS, GCE, Azure 등 클라우드에 특화된 모듈입니다. 노드를 추가/삭제하고 로드 밸런서를 연결하거나 볼륨을 붙일 수 있습니다. 각 클라우드 업체에서 인터페이스에 맞춰 구현하면 되기 때문에 확장성이 좋고 많은 곳에서 자체 모듈을 만들어 제공하고 있습니다.

<br>

**Node 구성 요소**

<center><img src="https://subicura.com/assets/article_images/2019-05-19-kubernetes-basic-1/kubernetes-node.png"></center>

- 큐블릿 kubelet

  노드에 할당된 Pod의 생명주기를 관리합니다. Pod을 생성하고 Pod 안의 컨테이너에 이상이 없는지 확인하면서 주기적으로 마스터에 상태를 전달합니다. API 서버의 요청을 받아 컨테이너의 로그를 전달하거나 특정 명령을 대신 수행하기도 합니다.

- 프록시 kube-proxy

  큐블릿이 Pod을 관리한다면 프록시는 Pod으로 연결되는 네트워크를 관리합니다. TCP, UDP, SCTP 스트림을 포워딩하고 여러 개의 Pod을 라운드로빈 형태로 묶어 서비스를 제공할 수 있습니다. 초기에는 kube-proxy 자체가 프록시 서버로 동작하면서 실제 요청을 프록시 서버가 받고 각 Pod에 전달해 주었는데 시간이 지나면서 iptables를 설정하는 방식으로 변경되었습니다. iptables에 등록된 규칙이 많아지면 느려지는 문제 때문에 최근 IPVS를 지원하기 시작했습니다.

- 추상화

  컨테이너는 도커고 도커가 컨테이너라고 생각해도 무리가 없는 상황이지만 쿠버네티스는 CRI(Container runtime interface)를 구현한 다양한 컨테이너 런타임을 지원합니다. [containerd](https://containerd.io/)(사실상 도커..), [rkt](https://coreos.com/rkt/), [CRI-O](https://cri-o.io/) 등이 있습니다.

  CRI 외에 CNI(네트워크), CSI(스토리지)를 지원하여 인터페이스만 구현한다면 쉽게 확장하여 사용할 수 있습니다.

<br>

**하나의 Pod이 생성되는 과정**

위에서 이야기한 조각을 하나하나 모아서 전체적인 흐름을 살펴보겠습니다. 관리자가 애플리케이션을 배포하기 위해 ReplicaSet을 생성하면 다음과 같은 과정을 거쳐 Pod을 생성합니다.

<center><img src="https://subicura.com/assets/article_images/2019-05-19-kubernetes-basic-1/create-replicaset.png"></center>

흐름을 보면 각 모듈은 서로 통신하지 않고 오직 API Server와 통신하는 것을 알 수 있습니다. API Server를 통해 etcd에 저장된 상태를 체크하고 현재 상태와 원하는 상태가 다르면 필요한 작업을 수행합니다. 각 모듈이 하는 일을 보면 다음과 같습니다.

**kubectl**

-  ReplicaSet 명세를 yml파일로 정의하고 kubectl 도구를 이용하여 API Server에 명령을 전달
-  API Server는 새로운 ReplicaSet Object를 etcd에 저장

**Kube Controller**

-  Kube Controller에 포함된 ReplicaSet Controller가 ReplicaSet을 감시하다가 ReplicaSet에 정의된 Label Selector 조건을 만족하는 Pod이 존재하는지 체크
-  해당하는 Label의 Pod이 없으면 ReplicaSet의 Pod 템플릿을 보고 새로운 Pod(no assign)을 생성. 생성은 역시 API Server에 전달하고 API Server는 etcd에 저장

**Scheduler**

-  Scheduler는 할당되지 않은(no assign) Pod이 있는지 체크
-  할당되지 않은 Pod이 있으면 조건에 맞는 Node를 찾아 해당 Pod을 할당

**Kubelet**

-  Kubelet은 자신의 Node에 할당되었지만 아직 생성되지 않은 Pod이 있는지 체크
-  생성되지 않은 Pod이 있으면 명세를 보고 Pod을 생성
-  Pod의 상태를 주기적으로 API Server에 전달

<br>

이제 복잡했던 그림이 이해되시나요? 위의 예제는 ReplicaSet에 대해 다뤘지만 모든 노드에 Pod을 배포하는 DaemonSet도 동일한 방식으로 동작합니다. DaemonSet controller와 Scheduler가 전체 노드에 대해 Pod을 할당하면 kubelet이 자기 노드에 할당된 Pod을 생성하는 식입니다.

각각의 모듈이 각자 담당한 상태를 체크하고 독립적으로 동작하는 것을 보면서 참 잘 만든 마이크로서비스 구조라는 생각이 듭니다.

<hr>

<br>

이번 글에서 쿠버네티스의 특징과 기본 개념, 개념을 구현한 아키텍처에 대해 알아보았습니다. 쿠버네티스는 아키텍처와 설치가 반, 설정 파일 작성이 나머지 반입니다. 그러니까 벌써 1/4 정도 했네요. 조금 복잡한 것이 사실이지만 예전에 어떤 걸 배울지 선택조차 어렵고 배워도 금방 옛날 지식이 되는 것보단 상황이 낫습니다. 이제 쿠버네티스만 배우면 되니까요.

쿠버네티스와 관련된 생태계는 빠르게 변하고 발전하고 있습니다. Cloud controller는 원래 Kube controller에 포함되어 있었는데 최근 분리되었고 Node의 모니터링을 위해 cAdvisor라는 것을 기본으로 사용했지만 선택 가능하게 제거되었습니다. 사용법이 바뀌고 계속해서 새로운 기능이 추가되지만 기본 적인 구성과 핵심 아키텍처는 크게 바뀌지 않습니다. 기본 적인 동작 원리를 이해하고 있다면 새로운 버전이 나오고 새로운 프레임워크가 등장해도 쉽게 이해하고 확장하여 사용할 수 있습니다.

<br>

<hr>
<br>



