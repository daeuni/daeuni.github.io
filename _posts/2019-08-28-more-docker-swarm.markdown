---
layout: post
title:  "More about Docker Swarm"
description: Quick and Easy management
date:   2019-08-28 10:00:36
categories: ingress dns loadbalancer

---

<br>

**Ingress Network**

<br>

도커 스웜은 서비스를 외부에 쉽게 노출하기 위해 모든 노드가 [ingress](https://docs.docker.com/engine/swarm/ingress/)라는 가상 네트워크에 속해있다. 

ingress는 routing mesh라는 개념을 가지고 있는데 이 개념은 서비스가 포트를 오픈할 경우 모든 노드에 포트가 오픈되고 어떤 노드에 요청을 보내도 실행 중인 컨테이너에 자동으로 전달해주는 것이다.

<center><img src="https://subicura.com/assets/article_images/2017-02-25-container-orchestration-with-docker-swarm/ingress-network.png"></center> 

위의 예제에서는 4567 포트를 오픈했기 때문에 3개의 노드 전체에 4567 포트가 오픈되었고 어디에서 테스트를 하든 간에 core-01노드에 실행된 컨테이너로 요청이 전달된다. 컨테이너가 여러 개라면 내부 로드밸런서를 이용하여 여러 개의 컨테이너로 분산처리된다.

뭔가 어색한 개념이라고 생각할 수 있지만, 어느 서버에 컨테이너가 실행 중인지 알 필요 없이 외부에서 그 어떤 방법보다 쉽게 접근할 수 있다.

실제 운영시에는 외부에 nginx 또는 haproxy같은 로드밸런서를 두고 전체 스웜 노드를 바라보게 설정할 수 있다. [관련링크](https://docs.docker.com/engine/swarm/ingress/#/configure-an-external-load-balancer)

<br>

**Service Replication**

<br>

앞에서 생성한 웹 애플리케이션에 부하가 발생했다고 가정하고 컨테이너를 5개로 늘려보자. 

노드가 3개인데 컨테이너를 5개로 늘리면 서버 한 개에 2개의 컨테이너가 생성된다. 각각은 독립된 컨테이너이기 때문에 여러 개를 생성해도 상관없고 ingress 네트워크가 알아서 요청을 분산처리해준다.

<center><img src="https://subicura.com/assets/article_images/2017-02-25-container-orchestration-with-docker-swarm/replication.png"></center> 

`service scale`명령어를 이용하여 서비스 개수를 늘려보자.

```
docker service scale whoami=5
```

output:

```
whoami scaled to 5 
```

`serivice ps` 명령어를 이용하여 상태를 확인한다.

```
docker service ps whoami
```

output:

```
ID            NAME      IMAGE              NODE     DESIRED STATE  CURRENT STATE            ERROR  PORTS
z5mcfb0lcizi  whoami.1  subicura/whoami:1  core-01  Running        Running 11 minutes ago
nwkduapf1kvk  whoami.2  subicura/whoami:1  core-03  Running        Preparing 4 seconds ago
vhxmipcdht6c  whoami.3  subicura/whoami:1  core-03  Running        Preparing 4 seconds ago
mntns383mnhz  whoami.4  subicura/whoami:1  core-01  Running        Running 4 seconds ago
xga086cubnj3  whoami.5  subicura/whoami:1  core-02  Running        Preparing 4 seconds ago
```

처음 컨테이너를 생성하는 노드에서 이미지를 다운받는 동안 Pending인 상태가 보이고 조금 기다리면 모두 실행된다. 이제 테스트해보자.

```
curl core-01:4567
curl core-01:4567
curl core-01:4567
curl core-01:4567
curl core-01:4567
curl core-01:4567

#output
7590556fb196
7d5e26679eb8
a2c04acee702
88498091561a
e970ff9ed9c6
7590556fb196
```

5개의 컨테이너에 아주 고르게 요청이 분산되었다. 

아무런 설정 없이 ingress 네트워크가 알아서 로드 밸런서 역할을 해주고 있다. Nginx나 haproxy같은 설정이 필요 없다! 너무 편하다!!!

<br>

**Healthcheck**

<br>

<center><img src="https://subicura.com/assets/article_images/2017-02-25-container-orchestration-with-docker-swarm/healthcheck.png"></center> 

스웜은 컨테이너가 실행되면 ingress에서 해당 컨테이너로 요청을 보내기 시작하는데 웹 서버가 부팅되는 동안 문제가 있다. 계속 curl 테스트를 하다 보면 도중 에러가 발생하는 걸 확인할 수 있다. ingress는 컨테이너가 실행되면 요청을 보내기 때문에 문제가 없을거라고 생각할 수 있지만 컨테이너가 실행되고 웹 서버가 완전히 실행되기 전에 요청을 보내면 서버를 찾을 수 없다는 메시지가 나오고 이러한 문제를 해결하기 위해 좀 더 자세한 상태 체크 방식이 필요하게 되었다.

도커 1.12버전에서 `HEALTHCHECK` 명령어가 추가되었고 쉘 스크립트를 작성하여 좀 더 정확한 상태를 체크 할 수 있다.

```dockerfile
FROM ruby:2.3-alpine
MAINTAINER subicura@subicura.com
COPY Gemfile* /usr/src/app/
WORKDIR /usr/src/app
RUN bundle install
COPY . /usr/src/app
EXPOSE 4567
HEALTHCHECK --interval=10s CMD wget -qO- localhost:4567
CMD bundle exec ruby app.rb -o 0.0.0.0
```

아주 익숙한 도커파일샘플에 HEALTHCHECK 명령어를 추가했다. HEALTHCHECK 부분을 보면 10초마다 wget을 이용하여 4567 포트를 체크하는 걸 볼 수 있다. 

스크립트(wget)가 0을 리턴하면 정상이라고 판단하고 그때서야 요청을 보내게 됩니다.

처음 테스트로 생성한 서비스를 `subicura/whoami:2`이미지로 업데이트 해보자. 버전 1과 차이점은 HEALTHCHECK 유무이다. HEALTHCHECK 옵션이 없다면 업데이트하는 동안 순간적으로 에러가 발생할 수 있는데 그런 문제를 방지해 준다.

`service update`명령어를 이용하여 서비스를 업데이트하자.

```
docker service update \
--image subicura/whoami:2 \
whoami
```

output:

```
whoami scaled to 5
```

`service ps`명령어로 업데이트 상태를 확인해보자.

```
docker service ps whoami
```

output:

```
ID            NAME          IMAGE              NODE     DESIRED STATE  CURRENT STATE           ERROR  PORTS
kv2jddl0bsfw  whoami.1      subicura/whoami:2  core-01  Ready          Ready 2 seconds ago
z5mcfb0lcizi   \_ whoami.1  subicura/whoami:1  core-01  Shutdown       Running 11 seconds ago
nwkduapf1kvk  whoami.2      subicura/whoami:1  core-03  Running        Running 9 minutes ago
vhxmipcdht6c  whoami.3      subicura/whoami:1  core-03  Running        Running 9 minutes ago
mntns383mnhz  whoami.4      subicura/whoami:1  core-01  Running        Running 11 minutes ago
xga086cubnj3  whoami.5      subicura/whoami:1  core-02  Running        Running 9 minutes ago
```

하나씩 하나씩 예쁘게 업데이트되는 과정을 볼 수 있고 접속 테스트를 하면 문제 없이 업데이트되는걸 확인 할 수 있다.

<br>

<hr>

<br>

**방문자 수 체크 웹 애플리케이션**

<br>

실제 세상에서는 웹 애플리케이션이 혼자 실행되는 경우가 거의 없다. redis와 함께 웹 애플리케이션을 실행하고 서버에 접속할 때마다 카운트를 +1 하는 서비스를 만들어 보자.

먼저, 웹 애플리케이션과 redis가 통신할 수 있는 오버레이 네트워크를 만든다. 오버레이 네트워크를 사용하면 redis는 외부에 포트를 오픈하지 않아도 되고 웹 애플리케이션과 다른 노드에 있어도 같은 서버에 있는 것처럼 통신할 수 있다. [관련링크]([https://ko.wikipedia.org/wiki/%EC%98%A4%EB%B2%84%EB%A0%88%EC%9D%B4_%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC](https://ko.wikipedia.org/wiki/오버레이_네트워크))

<center><img src="https://subicura.com/assets/article_images/2017-02-25-container-orchestration-with-docker-swarm/overlay-network.png"></center> 

`network create`명령어로 오버레이 네트워크를 생성합니다.

```
docker network create --attachable \
  --driver overlay \
  backend
```

output:

```
k7ayhk5hqbxnk8zbb959txh6h  
```

잘 생성되었는지 `network ls`명령어로 확인해보겠습니다.

```
docker network ls
```

output:

```
NETWORK ID          NAME                DRIVER              SCOPE
k7ayhk5hqbxn        backend             overlay             swarm
c420312e4c9b        bridge              bridge              local
c3ef285ed639        docker_gwbridge     bridge              local
244840dc44fd        host                host                local
4biz947vsmez        ingress             overlay             swarm
f41f0c773519        none                null                local
```

backend라는 이름의 오버레이 네트워크가 생성되었습니다. 그 외에 네트워크는 기본으로 생성되는 네트워크들이다. 이제 redis를 backend네트워크에 생성해보자.

<center><img src="https://subicura.com/assets/article_images/2017-02-25-container-orchestration-with-docker-swarm/overlay-network-redis.png"></center> 

```
docker service create --name redis \
  --network=backend \
  redis
```

output

```
cjqrmjqfvj8zyr7k9skih01g4
```

`--network` 옵션으로 어느 네트워크에 속할지 정의하였다. backend 네트워크에 속한 redis 서비스는 외부 네트워크에서는 접근할 수 없고 backend 네트워크 상에서 원래 포트인 6379로 접속할 수 있다.

redis 서비스가 제대로 생성되었다면 telnet명령어로 테스트해보겠습니다.

```
docker run --rm -it \
  --network=backend \
  alpine \
  telnet redis 6379

# test
KEYS *
SET hello world
GET hello
DEL hello
QUIT
```

외부 네트워크에서는 redis 서버에 접근할 수 없으므로 접속 테스트용 컨테이너를 생성하였다. `--network`옵션으로 backend 네트워크에 접속하였고 redis 서비스의 이름이 곧 도메인명이므로 바로 접속할 수 있다.

<br>

이제 방문자 수를 체크하는 웹 애플리케이션 서비스를 생성해보자.

<center><img src="https://subicura.com/assets/article_images/2017-02-25-container-orchestration-with-docker-swarm/overlay-network-redis-counter.png"></center> 

```
docker service create --name counter \
  --network=backend \
  --replicas 3 \
  -e REDIS_HOST=redis \
  -p 4568:4567 \
  subicura/counter
```

output:

```
nlq75mhritr3hj4tibm8ka9d8
```

redis 서버와 통신해야 하므로 backend 네트워크를 지정했고 `--replicas`옵션을 이용하여 3개를 생성하도록 했다. 외부의 4568 포트로 접근할 수 있도록 포트를 오픈 했고 redis 서버의 도메인명을 환경변수로 전달했다.

이제 테스트해보자.

```
curl core-01:4568

#output
50bec9ce8874 > 1

curl core-01:4568

#output
50bec9ce8874 > 1
6705a6272a4a > 1

curl core-01:4568

#output
50bec9ce8874 > 1
6705a6272a4a > 1
92b471d0d741 > 1

curl core-01:4568

#output
50bec9ce8874 > 2
6705a6272a4a > 1
92b471d0d741 > 1

curl core-01:4568

#output
50bec9ce8874 > 2
6705a6272a4a > 2
92b471d0d741 > 1

curl core-01:4568

#output
50bec9ce8874 > 2
6705a6272a4a > 2
92b471d0d741 > 2
```

웹 애플리케이션 서버와 redis가 backend 오버레이 네트워크를 통해 연결되었고 ingress 네트워크가 3개의 웹 컨테이너에 부하를 분산하였다.

스웜은 서로 통신이 필요한 서비스를 같은 이름의 오버레이 네트워크로 묶고 내부 DNS 서버를 이용하여 접근할 수 있다. 여러 개의 네트워크를 쉽게 만들 수 있고 하나의 서비스는 여러 개의 네트워크에 속할 수 있다. 모든 것은 스웜이 알아서 하고 관리자는 네트워크와 서비스만 잘 만들면 된다.

<br>

**비밀키 조회 애플리케이션**

<br>

이번에는 1.13에 추가된 비밀키 관리 기능을 이용해보자. 

먼저 비밀키를 생성하자. 파일 또는 파이프를 이용한 stdin입력만 가능하다.

```
echo "this is my password!" | docker secret create my-password -
```

output:

```
8rsw3eu74xea4q2e8mnhuw6lu
```

`my-passsword`라는 이름의 비밀키에 `this is my password!`라는 내용을 저장하였다. 이제 이 키를 사용할 서비스를 하나 생성하겠다.

```
docker service create --name secret \
  --secret my-password \
  -p 4569:4567 \
  -e SECRET_PATH=/run/secrets/my-password \
  subicura/secret
```

output:

```
nmyk5647bv4mfneafwhpg7o4t
```

`--secret`옵션으로 어떤 키를 사용할지 지정하면 `/run/secret`디렉토리 밑에 키 이름의 파일로 마운트 된다. `subicura/secret` 이미지는 파일의 내용을 출력해주는 웹 애플리케이션으로 4569포트로 오픈하였다.

잘 동작하는지 테스트해보자.

```
curl core-01:4569
```

output:

```
this is my password!
```

아주 잘 동작한다. 도커를 이용하여 서비스를 구축하면 패스워드 정보를 어떻게 할지 고민일 때가 있는데 아주 유용하게 사용할 수 있다.

<br>

**Traefik Reverse Proxy Server**

<br>

스웜은 매니저 노드에서 모든 서비스를 관리할 수 있고 서비스가 생성된 것을 체크하면 무언가를 자동화할 수 있다.

 [Traefik](https://traefik.io/)은 이러한 아이디어를 이용해 웹 애플리케이션이 생성되면 자동으로 도메인을 만들고 내부 서비스로 연결해준다.

예를 들어 `whoami`라는 서비스를 만들면 `whoami.local.dev`도메인으로 접속할 수 있게 해주고 `counter`라는 서비스를 만들면 `counter.local.dev`도메인으로 접속할 수 있게 해준다.

테스트를 위해 도메인 정보를 `/etc/hosts`에 추가하자.

```
172.17.8.101 portainer.local.dev counter.local.dev
```

프록시로 연결하기 위해 `frontend`라는 네트워크를 생성해보자.

```
docker network create --driver overlay frontend
```

이제 `traefik`서비스를 frontend 네트워크에 연결해보자.

```
docker service create --name traefik \
  --constraint 'node.role == manager' \
  -p 80:80 \
  -p 8080:8080 \
  --mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock \
  --network frontend \
  traefik \
  --docker \
  --docker.swarmmode \
  --docker.domain=local.dev \
  --docker.watch \
  --web
```

`--constraint` 옵션을 이용하여 매니저 노드에서 실행하게 배포를 제한하였다. 스웜 서비스 정보를 얻기 위해서는 반드시 매니저 노드에서 실행되어야 한다. 그리고 `--mount` 옵션을 이용해 호스트의 도커 소켓을 마운트 하였다.

`docker run -v`옵션과 유사한데 service를 만들 때는 `--mount` 옵션을 사용한다. 80포트는 실제 웹 서비스를 제공하기 위해 사용할 포트, 8080은 관리자용 포트이다.

관리자 페이지에 접속해보자.

<center><img src="https://subicura.com/assets/article_images/2017-02-25-container-orchestration-with-docker-swarm/traefik.png"></center> 

뭔가 그럴싸한 관리자 화면이 보인다. 아직 아무것도 실행하지 않아 휑한 모습이다. [portainer](http://portainer.io/) 웹 애플리케이션을 실행하여 자동으로 연결해보자.

```
docker service create --name portainer \
  --network=frontend \
  --label traefik.port=9000 \
  --constraint 'node.role == manager' \
  --mount "type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock" \
  portainer/portainer
```

traefik과 접속할 수 있도록 frontend네트워크에 연결했고 `--label`옵션으로 내부적으로 사용하는 웹 포트를 알려주었다. traefik은 `traefik.port`라벨의 값을 읽어 해당 포트로 리버스 프록시를 연결한다. 

portainer는 스웜을 관리해주는 툴이기 때문에 매니저 노드에 실행하도록 했다.

서비스 이름을 portainer로 했기 때문에 서비스가 실행되면 traefik서비스가 자동으로 `portainer.local.dev`도메인 정보를 생성하고 portainer컨테이너의 9000번 포트로 연결한다. 서비스를 생성하고 잠시 기다리면 프록시 설정이 자동으로 완료된다.

<center><img src="https://subicura.com/assets/article_images/2017-02-25-container-orchestration-with-docker-swarm/traefik-portainer.png"></center> 

traefik 관리자 화면에 portainer가 등록된 모습이 보인다! 이제 실제로 `portainer.local.dev`에 접속해보자.

<center><img src="https://subicura.com/assets/article_images/2017-02-25-container-orchestration-with-docker-swarm/portainer-1.png"></center> 

portainer도 역시 접속 성공이다! 이제 어떤 서비스도 frontend 네트워크를 연결하고 `traefik.port` 라벨만 지정하면 자동으로 외부에 노출된다. 내친김에 아까 만든 `counter`서비스도 붙여보자.

```
docker service rm counter
docker service create --name counter \
  --network=frontend \
  --network=backend \
  --replicas 3 \
  --label traefik.port=4567 \
  -e REDIS_HOST=redis \
  subicura/counter
```

기존에 생성되어 있던 서비스를 삭제하고 새로운 옵션으로 서비스를 생성한다. frontend와 backend 두 개의 네트워크에 모두 속해있고 label을 주어 traefik에서 자동으로 인식할 수 있도록 했다. 이제 테스트해보자.

```
curl counter.local.dev

# output

50bec9ce8874 > 2                                                                
5cbd7ecbb5b8 > 1                                                                
6705a6272a4a > 2                                                                
92b471d0d741 > 2 
```



마법과 같은 일이 옵션 몇줄로 일어났습니다. 음 스웜 정말 좋다!!!!!!

도커 스웜의 핵심 내용인 노드와 서비스에 대해 알아보았고 ingress 네트워크와 자체 내장된 로드밸런서, DNS 서버를 사용해보았다.

도커 스웜은 어떠한 툴도 추가적으로 설치할 필요 없이 굉장히 간단하고 빠르게 클러스터를 구성할 수 있습니다. 기능적으로 아쉬운 부분이 조금 있지만 작은 규모에서 일반적인 작업을 하기에는 무리가 없다. (오토스케일링 같은 기능을 많은 곳에서 쓰진 않지만 삼성은 쓰니까 ... )

기존에 오케스트레이션 도입을 꺼렸던 많은 소규모 개발팀에서는 스웜이 아주 매력적인 선택이 될 것 같다.

<hr>

다음 포스트 대규모 개발팀을 위한 쿠버네티스에 대해 공부해보겠습니다.

<hr>
<br>



