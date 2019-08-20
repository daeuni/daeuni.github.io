---
layout: post
title:  "Non-disruptive Deployment of Web Services Using Docker"
description: Blue-Green Deployment Using Docker
date:   2019-08-20 10:20:36
categories: nginx php mysql redis 
---

쇼핑이 목적인 웹사이트를 운영하는 관리자라면, 하루에도 여러번 소스를 업데이트 하고 운영서버로 배포하는 일이 많을 것이다.

배포는 단순히 로컬의 소스를 운영 서버로 복사하는 것이다. FTP로 파일을 복사하는 방식은 가장 기본이면서 그럭저럭 잘 동작한다. 배포 중에 서비스가 잠깐 멈추는 문제가 있다면 새벽에 배포하면 되고 굳이 다른 개발할일도 많은데 배포에 신경을 써야 하는 생각을 할 수 있다.

하지만, 오히려 배포가 탄탄해지면 서비스 개발에 집중할 수 있다. 또한 요즘 세상은 하루에 몇번이라도 배포를 자주, 더 빨리 하는 것이 서비스의 경쟁력이 되는 세상이다. 서비스가 빠르게 발전하고 있고 서버를 확장하려면 미리 미리 자동화를 하는 것을 준비해야 한다.

쇼핑 웹사이트처럼 운영서버는 크게 웹(+API), 마이크로서비스(배송조회, 우편번호검색), 영상, 채팅, 모니터링으로 구성되어 있다. 여기서는 웹 서비스 배포에 대해서 이야기 하려고 한다.

웹 서비스는 웹서버 1대(nginx + php)와 디비서버 1대(mysql + redis)로 간단하게 구성되어 있다. 서버 규모에 맞게 최대한 단순하게 배포 프로세스를 구성하려고 했지만, 추후 서비스가 잘 되었을때를 고려하여 확장 가능성도 고려하였다.

앞에서 도커의 이점에 대해 정리했지만 한번 더 짚고 넘어가는게 좋을 것 같다는 생각이 든다.

<br>

**도커를 이용한 배포**

<br>

도커를 이용한 배포가 갖는 특징이다.

**확장성**

- 이미지만 만들어 놓으면 컨테이너는 그냥 띄우기만 하면됨
- 다른 서버로 서비스를 옮기거나 새로운 서버에 서비스를 하나 더 띄우는건 `docker run` 명령어 하나로 끝
- 개발서버 띄우기도 편하고 테스트서버 띄우기도 간편

**표준성**

- 도커를 사용하지 않는 경우 ruby, nodejs, go, php로 만든 서비스들의 배포 방식은 제각각 다름
- 컨테이너라는 표준으로 서버를 배포하므로 모든 서비스들의 배포과정이 동일해짐

**이미지**

- 이미지에서 컨테이너를 생성하기 때문에 반드시 이미지를 만드는 과정이 필요
- 이미지를 저장할 곳이 필요
- 빌드 서버에서 이미지를 만들면 해당 이미지를 [distribution](https://github.com/docker/distribution)에 저장하고 운영서버에서 이미지를 불러옴

**설정**

- 설정은 보통 환경변수로 제어함
- `MYSQL_PASS=password`와 같이 컨테이너를 띄울때 환경변수를 같이 지정
- 하나의 이미지가 환경변수에 따라 동적으로 설정파일을 생성하도록 만들어져야함

**공유자원**

- 컨테이너는 삭제 후 새로 만들면 모든 데이터가 초기화됨
- 업로드 파일을 외부 스토리지와 링크하여 사용하거나 S3같은 별도의 저장소가 필요
- 세션이나 캐시를 파일로 사용하고 있다면 memcached나 redis와 같은 외부로 분리

**힙**

- 왠지 최신기술을 쓰는 느낌을 갖음
- 힙한 개발자가 된 느낌을 갖음

<hr>

그럼 이제 쇼핑 웹페이지를 위한 도커 이미지를 만들어보자

<br>

**도커 이미지 만들기**

<br>

이제 본격적으로 이미지를 만들어보자.

<center><img src="https://github.com/daeuni/daeuni.github.io/blob/master/assets/hsimage.PNG?raw=true"></center> 

**D-nginx**

```dockerfile
FROM subicura/nginx:1.11.1
MAINTAINER Chungsub Kim <chungsub.kim@purpleworks.co.kr>

ADD /data/config/nginx/load_pagespeed_module.conf /usr/local/nginx/conf/conf.d/load_pagespeed_module.conf
ADD /data/config/nginx/production.conf /usr/local/nginx/conf/sites/production.conf

CMD /usr/local/sbin/nginx
```

Base Image

- [subicura/nginx](https://github.com/subicura/Dockerfiles/tree/master/nginx)
- nginx 최신버전
- ssl + http2 + stream + realip + ngx pagespeed와 같은 흔히 사용하는 모듈 컴파일

nginx pagespeed 모듈을 활성화하기 위한 설정파일 추가

php7-fpm 연동을 위한 설정파일 추가

<br>

**D-app**

```dockerfile
FROM subicura/php7-fpm:latest
MAINTAINER Chungsub Kim <chungsub.kim@purpleworks.co.kr>

# Add source & config files
ADD src /var/www/magento
ADD data/config/php/99-custom.ini /etc/php/7.0/fpm/conf.d/99-custom.ini

# Volume
VOLUME /var/www/magento
VOLUME /var/run/php

# Run
COPY data/docker/start.sh /usr/local/bin/
RUN ln -s usr/local/bin/start.sh /start.sh
CMD ["start.sh"]
```

Base Image

- [subicura/php7-fpm](https://github.com/subicura/Dockerfiles/blob/master/php7-fpm)
- php7-fpm 최신버전
- mysql, curl, mcrypt, mbstring등 php 익스텐션 설치
- v8, xdebug 선택적 사용가능

base 이미지에 소스파일과 php custom 설정파일을 추가함

`start.sh` 파일은 환경변수에 따라 데이터베이스 정보등을 설정하고 php7-fpm 프로세스를 실행함

<br>

**컨테이너 구성**

<br>

이미지를 만들었으니 사실상 90% 작업은 완료되었다. 이제 생성된 이미지를 컨테이너로 실행만 하면 된다.

<center><img src="https://github.com/daeuni/daeuni.github.io/blob/master/assets/hscontainer.png?raw=true"></center> 

컨테이너를 실행할때 [docker compose](https://github.com/docker/compose)를 이용하면 힘들게 명령어를 길게 입력하지 않아도 되고 컨테이너간 의존성도 알아서 체크하여 순서대로 실행해준다.

```yaml
version: '2'
services:
  nginx:
    image: xxx/likehs-nginx:latest
    network_mode: 'bridge'
    depends_on:
      - app
    volumes_from:
      - app
    ports:
      - 8080:80
  app:
    image: xxx/likehs-app:latest
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: user
      MYSQL_PASSWORD: password
      MYSQL_DATABASE: database
      REDIS_HOST: redis
      REDIS_PORT: 6379
      ENABLE_PHP7_FPM_V8: 1
    volumes:
      - /data/likehs/magento_files:/var/www/magento/media
```

nginx container

- `likehs-app`컨테이너에서 `/var/www/magento`디렉토리와 `/var/run/php`디렉토리가 볼륨으로 설정되어 있으므로 마치 내 컨테이너에 있는 디렉토리처럼 자동으로 마운트됨
- nginx 설정파일은 `unix:/var/run/php/php7.0-fpm.sock`여기를 바라보게 셋팅되어 있고 `/var/www/magento`디렉토리를 루트로 바라봄
- 컨테이너 내부의 `80포트`를 호스트의 `8080포트`로 연결함

<br>

app container

- 데이터베이스와 redis설정을 환경변수로 제어함
- 업로드 디렉토리를 호스트의 디렉토리로 연결하여 컨테이너를 새로 띄워도 파일을 유지할 수 있음

<br>

이제 운영서버에 접속한 후 `docker-compose up -d`를 실행하면 원격에서 이미지를 다운받고 서비스가 실행됩니다. 이제 배포를 위한 모든 준비가 끝났다.

웹 서비스를 배포하기 위해서는 coresos, fleet, apache mesos, kubernetes, docker swarm 등이 있지만 여기서는 간단하게 원격 API를 호출해보겠습니다.

<br>

**Docker 원격 API**

<br>

도커는 기본적으로 원격 API를 호출하기 쉬운 구조이다. 도커 명령어를 실행할때 의식하고 있지는 않지만 자동으로 `unix:///var/run/docker.sock` 여기를 바라보고 명령을 하고 있다. 도커 데몬을 실행할때 `-H tcp://0.0.0.0:2375` 옵션을 주게 되면 원격에서 명령을 보낼 준비가 완료된다.

- 도커 원격명령 실행
  - DOCKER_HOST 환경변수를 셋팅합니다.
  - `DOCKER_HOST=tcp://192.168.0.100:2375 docker run`
- Docker Compose 원격명령 실행
  - DOCKER_HOST 환경변수를 셋팅합니다.
  - `DOCKER_HOST=tcp://192.168.0.100:2375 docker-compose up -d`

<br>

**원격 API를 이용한 배포**

원격으로 도커 명령어를 실행하는 방법을 이용해 간단하게 스크립트를 만든다.

```shell
#!/bin/sh

TARGET_DEPLOY_TCP=tcp://192.168.0.100:2375
DOCKER_APP_NAME=likehs
DOCKER_HOST=${TARGET_DEPLOY_TCP} docker-compose -p ${DOCKER_APP_NAME} -f docker-compose.yml down
DOCKER_HOST=${TARGET_DEPLOY_TCP} docker-compose -p ${DOCKER_APP_NAME} -f docker-compose.yml pull
DOCKER_HOST=${TARGET_DEPLOY_TCP} docker-compose -p ${DOCKER_APP_NAME} -f docker-compose.yml up -d

```

기존에 실행중인 컨테이너를 멈추고(down) 최신버전을 내려 받고(pull) 실행하면(up) 끝이다.

<br>

<hr>

<br>

**Blue-Green 배포**

<br>

도커 이미지를 만들고 컨테이너를 배포하는데 성공했지만 배포할때마다 서비스가 잠시 중단(down하고 up하는 사이)되는 치명적인 단점이 있다. 이부분을 [blue-green 배포 방식](http://martinfowler.com/bliki/BlueGreenDeployment.html)을 이용하여 무중단으로 배포해보자.

**nginx load balance 기능 이용하기**

<center><img src="https://github.com/daeuni/daeuni.github.io/blob/master/assets/nginx.png?raw=true"></center> 

nginx는 무료면서 훌륭한 성능을 자랑하는 로드밸런서입니다. `80포트`로 들어온 요청을 `8080포트`, `8081포트`로 분산할 수 있고 health check를 통해 포트가 죽어있다면 살아있는 포트로 요청을 보내게 된다.

보통은 서로 다른 IP의 서버를 로드밸런스 하기 위해 사용하지만 한 IP에서 서로 다른 포트를 지정하는 것도 가능하다. 도커를 사용하지 않는다면 같은 서비스를 하나의 서버에 여러개 띄운다는 걸 상상하기 어렵지만 도커이기 때문에 쉽게 적용할 수 있다.

```shell
#!/bin/sh

TARGET_DEPLOY_TCP=tcp://192.168.0.100:2375
DOCKER_APP_NAME=likehs

EXIST_BLUE=$(DOCKER_HOST=${TARGET_DEPLOY_TCP} docker-compose -p ${DOCKER_APP_NAME}-blue -f docker-compose.blue.yml ps | grep Up)

if [ -z "$EXIST_BLUE" ]; then
    DOCKER_HOST=${TARGET_DEPLOY_TCP} docker-compose -p ${DOCKER_APP_NAME}-blue -f docker-compose.blue.yml pull
    DOCKER_HOST=${TARGET_DEPLOY_TCP} docker-compose -p ${DOCKER_APP_NAME}-blue -f docker-compose.blue.yml up -d

    sleep 10

    DOCKER_HOST=${TARGET_DEPLOY_TCP} docker-compose -p ${DOCKER_APP_NAME}-green -f docker-compose.green.yml down
else
    DOCKER_HOST=${TARGET_DEPLOY_TCP} docker-compose -p ${DOCKER_APP_NAME}-green -f docker-compose.green.yml pull
    DOCKER_HOST=${TARGET_DEPLOY_TCP} docker-compose -p ${DOCKER_APP_NAME}-green -f docker-compose.green.yml up -d

    sleep 10

    DOCKER_HOST=${TARGET_DEPLOY_TCP} docker-compose -p ${DOCKER_APP_NAME}-blue -f docker-compose.blue.yml down
fi
```

작은 규모에 맞는 아주 적절한 스크립트를 만들었다. 포트를 다르게 설정한 compose 파일을 2개 만들고 어떤 compose가 떠있는지 확인한다. 실행중이 아닌 compose를 실행하여 컨테이너를 띄운 후 다른 컨테이너를 멈춘다.

어떤 디펜던시도 필요 없고 어떤 에이전트도 없지만 확실하게 동작하는 스크립트이다.

<br>

<hr>

<br>

**자동화**

<br>

이제 무중단 배포까지 완료되었으니 배포를 자동화 해보자.

**git branch**

현재 소스는 총 3가지 타입의 브랜치로 관리하고 있다.

- 기능별 브랜치
  - `send-sms-after-order`와 같이 기능별로 브랜치를 만듬
  - master 브랜치로 머지 후 제거됨
- master
  - 메인 개발소스
  - 바로 push할 수 없음
  - 작업별 브랜치를 pull request 보내고 코드 리뷰 후 머지함
  - master 브랜치는 바로 스테이징서버로 배포
- production
  - 운영서버에서 사용중인 브랜치
  - master 브랜치가 스테이징서버로 배포되고 테스트가 끝나면 수동으로 master 브랜치를 production 브랜치로 머지
  - production 브랜치가 푸시되면 바로 운영서버로 배포

**gitlab webhook**

gitlab에는 푸시가 될때마다 이벤트를 보낼 수 있는 [webhook](https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/web_hooks/web_hooks.md)기능이 있고 jenkins는 webhook이 호출되면 자동으로 빌드를 시작하는 [plugin](https://wiki.jenkins-ci.org/display/JENKINS/Gitlab+Hook+Plugin)이 존재합니다. gitlab은 모든 푸시 이벤트마다 jenkins를 호출하게 되고 jenkins는 branch를 보고 master일 경우는 staging 배포, production일 경우는 운영서버에 배포하게 된다.

**jenkins**

jenkins에 [slack플러그인](https://wiki.jenkins-ci.org/display/JENKINS/Slack+Plugin)을 설치하면 시작, 배포 후 성공/실패 여부를 슬랙 메시지로 받을 수 있다.

jenkins는 `테스트` > `이미지 빌드` > `배포` 과정을 수행한다.

<br>

**롤백**

<br>

배포된 이미지에 문제가 심각할 경우 이전 이미지로 되돌릴 수 있어야 합니다. 도커를 이용하면 이미지에 태그를 걸 수 있어, 손쉽게 구현할 수 있다.

```shell
#!/bin/sh

DOCKER_REGISTRY_NAME=xxxx/likehs
COMMIT_HASH="$(git show-ref --head | grep -h HEAD | cut -d':' -f2 | head -n 1 | head -c 10)"
docker build --force-rm=true -f Dockerfile -t ${DOCKER_REGISTRY_NAME}-app:$COMMIT_HASH .
docker tag -f ${DOCKER_REGISTRY_NAME}-app:$COMMIT_HASH ${DOCKER_REGISTRY_NAME}-app:latest

```

jenkins에서 이미지를 빌드할때 현재 `git commit hash`로 태그로 만든다. 그리고 해당 이미지를 다시 배포하려면 해당 git commit hash를 latest로 태그한 후 다시 배포하면 된다.

<br>
<hr>

다음 포스트에는 도커 스웜을 활용한 애플리케이션 개발을 해보겠다.

<hr>

<br>



