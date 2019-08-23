---
layout: post
title:  "Distributed server management with Docker Swarm"
description: Quick and Easy management
date:   2019-08-23 10:20:36
categories: swarm cluster manager worker
---

많은 서비스를 효율적으로 관리하기 위해 서버마다 역할을 부여한다. 예를들어, 1번 서버는 CI, 2번 서버는 모니터링, 3번 서버는 웹, 4번 서버는 디비, 5번 서버는 큐 이런 식으로 서비스의 종류에 따라 서버를 구분한다. 이런 방식은 효율적이고 정리가 잘된 것처럼 보이지만 서버가 늘어나고 컨테이너가 추가될 때마다 비효율적이다.

서버가 역할에 따라 나뉘어 있었기 때문에 특정 서버에 컨테이너가 몰리는 상황이 발생했고 역할이 모호한 애플리케이션의 경우는 어디에 설치할지 고민이 된다. 또한 리소스가 여유 있는 서버로 컨테이너를 옮기고 싶어도 다른 컨테이너와 의존성이 있어 쉽게 못 옮기는 경우도 있고 IP, Port와 같은 설정 파일을 수정하는 것도 문제다. 컨테이너는 굉장히 유연하여 확장이나 이동이 쉬운데 그런 장점을 이용하지 못하는 경우가 많다.

도커덕분에 애플리케이션 설치 자체가 굉장히 편해져 이런 작업은 사소해 보일 수 있지만, 인간의 욕심은 끝이 없고 더욱 격렬하게 아무것도 안 하고 싶기 때문에 컨테이너를 실행할 서버를 고르는 것부터 정보를 관리하는 것까지 모든 과정을 자동화하면 좋겠다는 생각을 한다.

이러한 작업을  [오케스트레이션](https://en.wikipedia.org/wiki/Orchestration_(computing))Orchestration이라고 한다.

<br>

**Server Orchestration**

<br>

<center><img src="https://subicura.com/assets/article_images/2017-02-25-container-orchestration-with-docker-swarm/container-orchestration.png"></center> 

서버 오케스트레이션은 여러대의 서버와 여러개의 서비스를 편리하게 관리해주는 작업이라고 할 수 있다. 실제로는 스케줄링scheduling, 클러스터링clustering, 서비스 디스커버리service discovery, 로깅logging, 모니터링monitoring 같은 일을 한다.

각 용어를 살펴보자.

**스케줄링**

: 컨테이너를 적당한 서버에 배포해 주는 작업. 툴에 따라서 지원하는 전략이 조금씩 다른데 여러 대의 서버 중 가장 할일 없는 서버에 배포하거나 그냥 차례대로 배포 또는 아예 랜덤하게 배포할 수도 있음. 컨테이너 개수를 여러 개로 늘리면 적당히 나눠서 배포하고 서버가 죽으면 실행 중이던 컨테이너를 다른 서버에 띄워주기도 함.

**클러스터링**

: 여러 개의 서버를 하나의 서버처럼 사용할 수 있음. 클러스터에 새로운 서버를 추가할 수도 있고 제거할 수도 있음. 작게는 몇 개 안 되는 서버부터 많게는 수천 대의 서버를 하나의 클러스터로 만들 수 있다. 여기저기 흩어져 있는 컨테이너도 가상 네트워크를 이용하여 마치 같은 서버에 있는 것처럼 쉽게 통신할 수 있음.

**서비스 디스커버리**

: 서비스를 찾아주는 기능. 클러스터 환경에서 컨테이너는 어느 서버에 생성될지 알 수 없고 다른 서버로 이동할 수도 있다. 따라서 컨테이너와 통신을 하기 위해서 어느 서버에서 실행중인지 알아야 하고 컨테이너가 생성되고 중지될 때 어딘가에 IP와 Port같은 정보를 업데이트해줘야 함. 키-벨류 스토리지에 정보를 저장할 수도 있고 내부 DNS 서버를 이용할 수도 있음.

**로깅, 모니터링**

: 여러 대의 서버를 관리하는 경우 로그와 서버 상태를 한곳에서 관리하는게 편합니다. 툴에서 직접 지원하는 경우도 있고 따로 프로그램을 설치해야 하는 경우도 있음. [ELK](https://www.elastic.co/kr/webinars/introduction-elk-stack)와 [prometheus](https://prometheus.io/)등 다양한 툴이 있음.
<br>
하지만, 이렇게 좋은 게 있는지 알면서도 여러 가지 이유로 도입하기 어려운 게 현실이다.

가장 큰 이유는 설치와 관리가 어렵다는 건데 보통 오케스트레이션 툴들은 수백-수천-수만 대의 엄청나게 많은 서버를 관리하기 위한 목적으로 만들어졌습니다. 기본적으로 설치와 관리가 꽤 어려운 편이고 일부 회사들이 아니면 도입하는 비용이 수작업으로 관리하는 비용보다 훨씬 크다. 툴에 대해 공부도 많이 해야 하고 뭔가 장애가 생기면 전체 서버에 문제가 생길 수 있어서 도입이 조심스러울 수밖에 없다.

이런 상황에서 등장한 **도커 스웜**은 오케스트레이션 툴은 관리가 어렵고 사용하기 복잡하다는 편견을 완전히 바꿔놓았다. **구축 비용이 거의 들지 않고 관리 또한 쉬우며 다양한 기능을 쉽게 제공하고 가볍게 사용**할 수 있다.

<hr>

<br>

**Container Orchestration Tool**

<br>

본격적인 스웜 설명에 앞서 다른 컨테이너 오케스트레이션 툴에 대해 살펴보겠다. 다양한 툴이 다양한 특징을 가지고 만들어졌으니 가볍게 비교해보면 좋을 것 같다.

<center><img src="https://subicura.com/assets/article_images/2017-02-25-container-orchestration-with-docker-swarm/orchestration-tools.png"></center> 

**fleet by CoreOS**

​	설치 및 관리 ★★★★★ / 사용법 ★★★★ / 기능 ★

- CoreOS(Container OS)에 기본으로 내장되어 있는 [systemd](https://www.freedesktop.org/wiki/Software/systemd/)의 cluster 버전
- 스케줄링 외에 특별한 기능이 없어 전체 서버에 무언가(agent류)를 bootstrap 하는 용도에 적당

<br>

**kubernetes by Cloud Native Computing Foundation (with google)**

​	설치 및 관리 ★★ / 사용법 ★★★ / 기능 ★★★★★

- 15년 동안 쌓인 구글의 노하우로 만든 “Production-Grade Container Orchestration”으로 기능이 매우 매우 훌륭함
- 자체적으로 구축하려면 설치와 관리가 까다롭지만, [Google Cloud](https://cloud.google.com/), [Azure](https://azure.microsoft.com/), [Tectonic](https://coreos.com/tectonic/), [Openshift](https://www.openshift.com/)같은 플랫폼을 이용하면 조금 편해짐
- 구성 및 개념에 대한 이해가 필요하고 새로운 툴과 사용법을 익히는데 시간이 좀 걸림

<br>

**EC2 Container Service (ECS) by AWS**

​	설치 및 관리 ★★★★★ / 사용법 ★★★★ / 기능 ★★

- AWS를 사용 중이라면 쉽게 사용할 수 있고 개념도 간단함
- 기능이 매우 심플하지만 사실 대부분 시스템에서 복잡한 기능은 필요하지 않으므로 단순한 구성이라면 이 이상 편한툴은 없음

<br>

**Nomad by HashiCorp**

​	설치 및 관리 ★★★ / 사용법 ★★★★ / 기능 ★★★★

- [Vagrant](https://www.vagrantup.com/), [Consul](https://www.consul.io/), [Terraform](https://www.terraform.io/), [Vault](https://www.vaultproject.io/)등을 만든 DevOps 장인 [HashiCorp](https://www.hashicorp.com/)의 제품

- 같은 회사의 Consul과 Vault를 연동해서 사용할 수 있고 기능은 다양하고 좋지만 반대로 공부할 게 많고 관리가 복잡할 수 있다는 뜻

- 도커외에 rkt, java 애플리케이션을 지원하고 커맨드라인 명령어는 직관적이고 매우 이쁘게 동작함

  <br>

### 결론

- 적당한 규모(수십 대 이내)에서는 `swarm`
- 큰 규모의 클러스터에서는 `kubernetes`
- AWS에서 단순하게 사용한다면 `ECS`
- HashiCorp팬이라면 `Nomad`

<br>

**Docker Swarm**

<br>

<center><img src="https://subicura.com/assets/article_images/2017-02-25-container-orchestration-with-docker-swarm/swarmnado.gif"></center> 

스웜은 도커와 별도로 개발되었지만 도커 1.12 버전부터 스웜 모드Swarm mode라는 이름으로 합쳐졌다. 도커의 모든게 내장되어 다른 툴을 설치할 필요가 없고 도커 명령어와 compose를 그대로 사용할 수 있어 다른 툴에 비해 압도적으로 쉽고 편리하다.

기능이 단순하고 필요한 것만 구현되어 있어 세부적인 컨트롤은 어렵다. 꽤 오랫동안 개발되었고(2015년 1.0버전) 1,000개 노드에 50,000개 컨테이너도 문제없이 테스트하여 안전성이 검증되었다.

<hr>
<br>

그럼 이제 도커 스웜에서 사용하는 용어를 알아보자.

**스웜swarm**

: 직역하면 떼, 군중이라는 뜻을 가지고 있다. 도커 1.12버전에서 스웜이 스웜 모드로 바뀌었지만 그냥 스웜이라고도 하는듯 한다. 스웜 클러스터 자체도 스웜이라고 한다. (스웜을 만들다. 스웜에 가입하다. = 클러스터를 만들다. 클러스터에 가입하다)

**노드node**

: 스웜 클러스터에 속한 도커 서버의 단위이다. 보통 한 서버에 하나의 도커데몬만 실행하므로 서버가 곧 노드라고 이해하면 된다. 매니저 노드와 워커 노드가 있다.

**매니저노드manager node**

: 스웜 클러스터 상태를 관리하는 노드. 매니저 노드는 곧 워커노드가 될 수 있고 스웜 명령어는 매니저 노드에서만 실행됨.

**워커노드worker node**

: 매니저 노드의 명령을 받아 컨테이너를 생성하고 상태를 체크하는 노드.

**서비스service**

: 기본적인 배포 단위. 하나의 서비스는 하나의 이미지를 기반으로 생성하고 동일한 컨테이너를 한개 이상 실행할 수 있다.

**테스크task**

: 컨테이너 배포 단위다. 하나의 서비스는 여러개의 테스크를 실행할 수 있고 각각의 테스크가 컨테이너를 관리한다.

<br>

<hr>

<br>

Docker 1.13을 기준으로 어떤 기능을 제공하는지 하나하나 살펴보자.

**스케줄링 scheduling**

: 서비스를 만들면 컨테이너를 워커노드에 배포한다. 현재는 균등하게 배포spread하는 방식만 지원하며 추후 다른 배포 전략이 추가될 예정이다. 노드에 라벨을 지정하여 특정 노드에만 배포할 수 있고 모든 서버에 한 대씩 배포하는 기능(Global)도 제공한다. 서비스별로 CPU, Memory 사용량을 미리 설정할 수 있다.

**고가용성 High Available**

: [Raft 알고리즘](https://raft.github.io/)을 이용하여 여러 개의 매니저노드를 운영할 수 있다. 3대를 사용하면 1대가 죽어도 클러스터는 정상적으로 동작하며 매니저 노드를 지정하는 방법은 간단하므로 쉽게 관리할 수 있다.

**멀티 호스트 네트워크 Multi Host Network**

: Overlay network로 불리는 SDN(Software defined networks)를 지원하여 여러 노드에 분산된 컨테이너를 하나의 네트워크로 묶을수 있다. 컨테이너마다 독립된 IP가 생기고 서로 다른 노드에 있어도 할당된 IP로 통신할 수 있다. (호스트 IP를 몰라도 된다!)

**서비스 디스커버리 Service Discovery**

: 서비스 디스커버리를 위한 자체 DNS 서버를 가지고 있다. 컨테이너를 생성하면 서비스명과 동일한 도메인을 등록하고 컨테이너가 멈추면 도메인을 제거한다. 멀티 호스트 네트워크를 사용하면 여러 노드에 분산된 컨테이너가 같은 네트워크로 묶이므로 서비스 이름으로 바로 접근할 수 있다. Consul이나 etcd, zookeeper와 같은 외부 서비스를 사용하지 않고 어떠한 추가 작업도 필요 없다. 스웜이 알아서 다 해준다.

**순차적 업데이트 Rolling Update**

: 서비스를 새로운 이미지로 업데이트하는 경우 하나 하나 차례대로 업데이트한다. 동시에 업데이트하는 작업의 개수와 업데이트 간격 시간을 조정할 수 있다.

**상태 체크 Health Check**

: 서비스가 정상적으로 실행되었는지 확인하기 위해 컨테이너 실행 여부 뿐 특정 쉘 스크립크가 정상으로 실행됐는지 여부를 추가로 체크할 수 있다. 컨테이너 실행 여부로만 체크할 경우 아직 서버가 실행 되지 않아 접속시 오류가 날 수 있는 미묘한 타이밍을 잡는데 효과적이다.

**비밀값 저장 Secret Management**

: 비밀번호를 스웜 어딘가에 생성하고 컨테이너에서 읽을 수 있다. 비밀 값을 관리하기 위한 외부 서비스를 설치하지 않고 쉽게 사용할 수 있다.

**로깅 Logging**

같은 노드에서 실행 중인 컨테이너뿐 아니라 다른 노드에서 실행 중인 서비스의 로그를 한곳에서 볼 수 있다. 특정 서비스의 로그를 보려면 어느 노드에서 실행 중인지 알필요도 없고 일일이 접속하지 않아도 된다.

**모니터링 Monitoring**

: 리소스 모니터링 기능은 제공하지 않습니다. 3rd party 서비스([prometheus](https://prometheus.io/), [grafana](http://grafana.org/))를 사용해야 합니다. 설치는 쉽지만 귀찮다.

**크론, 반복작업 Cron**

: 크론, 반복작업도 알아서 구현해야 한다. 여러 가지 방식으로 해결할 수 있겠지만 직접 제공하지 않아 아쉽다.

<br>

<hr>

<br>

**Swarm Excercise**

<br>

본격적으로 스웜 클러스터를 만들고 여러 가지 서비스를 생성해보면서 도커 스웜에 빠져보겠다. 전체적인 내용은 다음과 같다.

1. 준비
2. 스웜 클러스터 만들기
3. 기본 웹 애플리케이션 서비스
4. 방문자 수 체크 애플리케이션 (with redis)
5. 비밀키 조회 애플리케이션
6. Traefik 리버스 프록시 서버
7. 스웜 서버 모니터링

<br>

**준비**

[Vagrant](https://www.vagrantup.com/)를 이용하여 3개의 [CoreOS(Container Linux)](https://coreos.com/os/docs/latest)가상머신을 만든다. 각각 1 CPU와 1G Memory를 할당하고 매니저 노드 하나와 워커 노드 2개를 설정하여 사용힌다.

<center><img src="https://subicura.com/assets/article_images/2017-02-25-container-orchestration-with-docker-swarm/create-virtual-machine.png"></center> 

- core-01 (manager) / 172.17.8.101 / 1 cpu / 1G memory
- core-02 (worker) / 172.17.8.102 / 1 cpu / 1G memory
- core-03 (worker) / 172.17.8.103 / 1 cpu / 1G memory

> Vagrant는 VirtualBox, VMWare, Parallels같은 가상머신을 쉽게 만들고 관리해주는 툴이다. CoreOS는 도커가 내장된 가벼운 OS입니다.

<br>

가상머신을 만드는 방법은 다음과 같다.

1. virtual box 설치 [다운로드](https://www.virtualbox.org/)
2. Vagrant 설치 [다운로드](https://www.vagrantup.com/downloads.html)
3. [vagrant-hostsupdater](https://github.com/cogitatio/vagrant-hostsupdater) plugin 설치 `vagrant plugin install vagrant-hostsupdater`
4. 데모 파일 다운로드 [GitHub - docker swarm demo](https://github.com/subicura/docker-swarm-demo)
5. 가상머신 생성 데모 디렉토리에서 `vagrant up` 명령어 실행 -> 자동으로 생성!

가상머신이 정상적으로 생성되었는지 확인하려면 `vagrant status`, 가상머신을 삭제하려면 `vagrant destroy` 를 실행한다. 가상머신에 접속하는 명령어는 `vagrant ssh [가상머신이름]`이다.

vagrant status를 입력해서 아래의 화면이 나오면 성공이다.

```shell
Current machine states:

core-01                   running (virtualbox)
core-02                   running (virtualbox)
core-03                   running (virtualbox)
```

<br>

**스웜 클러스터 만들기**

가상머신이 정상적으로 생성되었으면 스웜 클러스터를 만들어보자. 스웜 클러스터는 manager 노드를 먼저 만들고 manager 노드가 생성한 토큰을 가지고 worker 노드에서 매니저 노드로 접속하면 된다.

먼저, `core-01`서버를 manager 노드로 설정한다. `swarm init`명령어를 사용한다.

```shell
# vagrant ssh core-01
docker swarm init --advertise-addr 172.17.8.101
```

output:

```shell
Swarm initialized: current node (gjm2fxwcx29ha4rfzkw3zskvt) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-1la23f7y6y8joqxz27hac4j79yffyixcp15tjtlrnei14wwd1t-5tlmx4q9fm46drjzzd95g1l4q \
    172.17.8.101:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

순식간에 매니저 노드가 생성되었다. `swarm init`명령어를 실행하면 친절하게 워커 노드에서 실행할 명령어를 알려준다. 명령어를 그대로 복사하여 2, 3번 서버에서 각각 실행하자.

```shell
# vagrant ssh core-02
docker swarm join \
    --token SWMTKN-1-1la23f7y6y8joqxz27hac4j79yffyixcp15tjtlrnei14wwd1t-5tlmx4q9fm46drjzzd95g1l4q \
    172.17.8.101:2377
```

output:

```shell
This node joined a swarm as a worker.
```

워커 노드가 정상적으로 만들어졌는지 매니저 노드에서 확인하자.

```
docker node ls
```

output:

```shell
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
gjm2fxwcx29ha4rfzkw3zskvt *  core-01   Ready   Active        Leader
mu6l92w2dzjmsbucvar1bl9mg    core-03   Ready   Active
q5pa26ik11g9surws03quvrsk    core-02   Ready   Active
```

명령어 3줄로 클러스터가 생성되었다. 도커 외에 아무것도 설치하지 않았고 어떤 agent도 실행하지 않았다. 이렇게 쉽게 생성되는 클러스터 보셨나요??? ><

- manager 노드가 관리하는 정보는  `/var/lib/docker/swarm`에 저장됨.
<br>
<hr>

클러스터가 생성되었으니 간단한 웹 애플리케이션 서비스를 생성해보자.

<hr>
<br>

**기본 웹 애플리케이션**

서비스를 생성하려면 `service create`명령어를 이용한다. `run`명령어와 아주 유사하다.

```
docker service create --name whoami \
  -p 4567:4567 \
  subicura/whoami:1
```

output:

```
beepr6q5kdm724vdvgmb2awxg
```

`subicura/whoami:1` 이미지는 <u>서버의 hostname을 출력하는 단순한 웹 애플리케이션</u>이다. 서비스의 이름을 `whoami` 로 지정했고 4567 포트를 오픈하였다.

서비스가 잘 생성되었는지 `service ls`명령어로 확인해보겠다.

```
docker service ls
```

output:

```
ID            NAME    MODE        REPLICAS  IMAGE                               
beepr6q5kdm7  whoami  replicated  0/1       subicura/whoami:1 
```

`whoami`라는 서비스가 보인다. 아직 컨테이너가 생성되지 않아 REPLICAS 상태가 `0/1`이다. 좀 더 상세한 상태를 확인하기 위해 `service ps`명령어를 입력해보자.

```
docker service ps whoami
```

output:

```
ID            NAME      IMAGE              NODE     DESIRED STATE  CURRENT STATE            ERROR  PORTS
wnt5m97ci36t  whoami.1  subicura/whoami:1  core-01  Running        Preparing 7 seconds ago
```

서비스의 상태와 어떤 노드에서 실행 중인지를 상세하게 확인할 수 있다. 현재 상태가 `Preparing`인걸 보니 이미지를 다운 받는 중인 것 같다. 조금만 기다리고 컨테이너가 실행되면 상태가 `Running`으로 바뀐다.

정상적으로 서비스가 실행되었는지 HTTP 요청으로 테스트해보자.

```
curl core-01:4567

#output
cfb152786b87
```

와우! 첫번째 서비스를 성공적으로 생성하였다.

`core-01`에서 테스트를 했는데 다른 서버도 테스트해볼까?

```
curl core-02:4567
curl core-03:4567

#output
cfb152786b87
cfb152786b87
```

분명 <u>컨테이너는 `core-01`노드에서 실행중인데 모든 노드에서 정상적으로 응답</u>하고 있다!!!!!!!!!!

어떻게 된일일까????

<br>

<hr>

다음 포스트에서는 이 질문에 대한 대답을 계속 이어나갈 것이다.

<hr>

<br>

