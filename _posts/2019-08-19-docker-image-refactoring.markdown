---
layout: post
title:  "Docker Image Refactoring"
description: How to optimize docker image
date:   2019-08-19 10:20:36
categories: Refactoring DockerHub 
---

앞서 만든 이미지를 최적화시키기 위해서 리팩토링 작업을 진행해보자.

<br>

**Base Image**

<br>

앞에서 만든 Ruby 애플리케이션 이미지는 `ubuntu`를 베이스로 만들었지만 사실 훨씬 간단한 `ruby` 베이스 이미지가 존재한다. 기존에 ruby를 설치했던 명령어는 ruby 이미지를 사용하는 것으로 간단하게 생략할 수 있다.

```shell
# before
FROM ubuntu:16.04
MAINTAINER subicura@subicura.com
RUN apt-get -y update
RUN apt-get -y install ruby
RUN gem install bundler

# after
FROM ruby:2.3
MAINTAINER subicura@subicura.com
```

ruby외에도 nodejs, python, java, go등 다양한 베이스 이미지가 이미 존재한다. 세부적인 설정이 필요하지 않다면 그대로 사용하는게 간편하다.

<br>

**Build Cache**

<br>

앞에서 빌드한 이미지를 다시 빌드해보자. 한번 빌드한 이미지를 다시 빌드하면 굉장히 빠르게 완료되는 걸 알 수 있다. 이미지를 빌드하는 과정에서 각 단계를 이미지 레이어로 저장하고 다음 빌드에서 캐시로 사용하기 때문이다.

```shell
Sending build context to Docker daemon 13.31 kB
Step 1/10 : FROM ubuntu:16.04
 ---> f49eec89601e
Step 2/10 : MAINTAINER subicura@subicura.com
 ---> Using cache
 ---> fc41cd8ac52d
Step 3/10 : RUN apt-get -y update
 ---> Using cache
 ---> 61d45ce11dc6
 ....
```

도커는 빌드할 때 Dockerfile의 명령어가 수정되었거나 추가하는 파일이 변경 되었을 때 캐시가 깨지고 그 이후 작업은 새로 이미지를 만들게 된다. ruby gem 패키지를 설치하는 과정은 꽤 많은 시간이 소요되기 때문에 최대한 캐시를 이용하여 빌드 시간을 줄여야 한다.

기존 소스에서 소스파일이 수정되면 캐시가 깨지는 부분은 다음과 같다.

```shell
COPY . /usr/src/app    # <- 소스파일이 변경되면 캐시가 깨짐
WORKDIR /usr/src/app
RUN bundle install     # 패키지를 추가하지 않았는데 또 install하게 됨 ㅠㅠ
```

복사하는 파일이 이전과 다르면 캐시를 사용하지 않고 그 이후 명령어는 다시 실행된다. ruby gem 패키지를 관리하는 파일은 Gemfile이다. 

Gemfile은 잘 수정되지 않으므로 다음과 같이 순서를 바꿀 수 있다.

```shell
COPY Gemfile* /usr/src/app/ # Gemfile을 먼저 복사함
WORKDIR /usr/src/app
RUN bundle install          # 패키지 인스톨
COPY . /usr/src/app         # <- 소스가 바꼈을 때 캐시가 깨지는 시점 ^0^
```

gem 설치 하는 부분을 소스 복사 이전으로 옮겼다. 이제 소스가 수정되더라도 매번 gem을 설치하지 않아 더욱 빠르게 빌드할 수 있다!!!

요즘 언어들은 대부분 패키지 매니저를 사용하므로 비슷한 전략으로 작성하면 된다.

<br>

**Command Optimzation**

<br>

이미지를 빌드할 때 불필요한 로그는 무시하는게 좋고 패키지 설치시 문서 파일도 생성할 필요가 없다.

따라서 명령어의 몇가지 옵션을 추가하여 명령어를 최적화하자.

```shell
# before
RUN apt-get -y update

# after
RUN apt-get -y -qq update
```

`-qq` 옵션으로 로그를 출력하지 않게 했다. 각종 리눅스 명령어는 보통 `quite`옵션이 있으니 적절하게 적용하면 된다.

```sh
# before
RUN bundle install

# after
RUN bundle install --no-rdoc --no-ri
```

`--no-doc`과 `--no-ri` 옵션으로 필요 없는 문서를 생성하지 않아 이미지 용량도 줄이고 빌드 속도도 더 빠르게 했다!

<br>

**Write Code Pretty**

<br>

명령어는 비슷한 것끼리 묶어 주는 게 보기도 좋고 레이어 수를 줄이는데 도움이 된다. 도커 이미지는 스토리지 엔진에 따라 레이어의 개수가 127개로 제한되어 있는 경우도 있어 너무 많은 명령어는 좋지 않다.

따라서 명령어를 &&를 사용하여 줄여보자.

```shell
# before
RUN apt-get -y -qq update
RUN apt-get -y -qq install ruby

# after
RUN apt-get -y -qq update && \
    apt-get -y -qq install ruby
```

<br>

**Final**

<br>

최종 Dockerfile은 아래와 같다.

```dockerfile
FROM ruby:2.3
MAINTAINER subicura@subicura.com
COPY Gemfile* /usr/src/app/
WORKDIR /usr/src/app
RUN bundle install --no-rdoc --no-ri
COPY . /usr/src/app
EXPOSE 4567
CMD bundle exec ruby app.rb -o 0.0.0.0
```

드디어 아까보다 훨씬 최적화된 이미지가 완성되었다.

<br>

<hr>

**Image Repository**

<br>

<center><img src="https://github.com/daeuni/daeuni.github.io/blob/master/assets/dockerregistry.png?raw=true"></center>

도커는 빌드한 이미지를 서버에 배포하기 위해 직접 파일을 복사하는 방법 대신 [도커 레지스트리](https://docs.docker.com/registry/)Docker Registry라는 이미지 저장소를 사용한다. 도커 명령어를 이용하여 이미지를 레지스트리에 푸시push하고 다른 서버에서 풀pull받아 사용하는 구조다.


도커 레지스트리는 [오픈소스](https://github.com/docker/distribution)로 무료로 설치할 수 있고 설치형이 싫다면 도커(Docker Inc.)에서 서비스 중인 [도커 허브](https://hub.docker.com/)Docker Hub를 사용할 수 있다.

<br>

**Docker Hub**

<br>

도커 허브는 도커에서 제공하는 기본 이미지 저장소로 ubuntu, centos, debian등의 베이스 이미지와 ruby, golang, java, python 등의 공식 이미지가 저장되어 있다. 일반 사용자들이 만든 이미지도 50만 개가 넘게 저장되어 있고 다운로드 횟수는 80억 회를 넘는다.

회원가입만 하면 대용량의 이미지를 무료로 저장할 수 있고 다운로드 트래픽 또한 무료다. 단, 기본적으로 모든 이미지는 공개되어 누구나 접근 가능하므로 비공개로 사용하려면 유료 서비스를 이용해야 한다. (한 개는 무료)

회원가입을 한 뒤에 앞에서 만든 Ruby 웹 애플리케이션 이미지를 저장해보자.

<br>

**Login**

<br>

도커에서 도커 허브 계정을 사용하려면 로그인을 해야한다.

```shell
docker login
```

output:

```shell
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: subicura
Password:
Login Succeeded
```

ID와 패스워드를 입력하면 로그인이 되고 `~/.docker/config.json`에 인증정보가 저장되어 로그아웃하기 전까지 로그인 정보가 유지된다.

<br>

**Image Tag**

<br>

도커 이미지 이름은 다음과 같은 형태로 구성된다.

```shell
[Registry URL]/[사용자 ID]/[이미지명]:[tag]
```

Registry URL은 기본적으로 도커 허브를 바라보고 있고 사용자 ID를 지정하지 않으면 기본값(library)을 사용한다. 따라서 `ubuntu` = `library/ubuntu` = `docker.io/library/ubuntu` 는 모두 동일한 표현이다.

도커의 `tag`명령어를 이용하여 기존에 만든 이미지에 추가로 이름을 지어줄 수 있다.

```she
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
```

앞에서 만든 `app`이미지에 계정정보와 버전 정보를 추가해보겠다.

```shell
docker tag app subicura/sinatra-app:1
```

`subicura`라는 ID를 사용하고 이미지 이름을 `sinatra-app`으로 변경했다. 첫 번째 버전이므로 태그는 `1`을 사용한다. 이제 `push`명령을 이용해 도커 허브에 이미지를 전송해 보자.

```shell
docker push subicura/sinatra-app:1
```

output:

```shell
The push refers to a repository [docker.io/subicura/sinatra-app]
2adeabae7edc: Pushed
8343e5bcf528: Pushed
af3b68c8b565: Pushed
40dd6783317f: Pushed
c6ae77e29c22: Pushed
5eb5bd4c5014: Mounted from library/ubuntu
d195a7a18c70: Mounted from library/ubuntu
af605e724c5a: Mounted from library/ubuntu
59f161c3069d: Mounted from library/ubuntu
4f03495a4d7d: Mounted from library/ubuntu
1: digest: sha256:af83aca920982c1fb17f08b4aa300439470349d58d63c921f67261054a0c9467 size: 2409
```

성공적으로 이미지를 도커 허브에 푸시하였습니다. 도커 허브에 저장된 50만 개의 이미지에 새로운 이미지가 하나 추가되었다!

이제 어디서든 `subicura/sinatra-app:1`이미지를 사용할 수 있다.

<br>

<hr>

다음 포스트에는 서버 관리의 꽃! 배포(Deploy)에 대해 알아보도록 하자.

<hr>

<br>



