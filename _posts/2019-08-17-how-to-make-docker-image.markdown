---
layout: post
title:  "Deploy Ruby Application on Ubuntu"
description: How to make docker image
date:   2019-08-17 10:20:36
categories: Ruby Ubuntu

---

도커는 이미지를 만들기 위해 컨테이너의 상태를 그대로 이미지로 저장하는 단순한 방법을 사용한다.

예를 들어, 어떤 애플리케이션을 이미지로 만든다면 리눅스만 설치된 컨테이너에 애플리케이션을 설치하고 그 상태를 그대로 이미지로 저장한다.

이제 Ruby로 만들어진 간단한 웹 애플리케이션을 Dockerizing(도커 이미지를 만듬) 해보자.

<br>

<hr>

<br>

**Sinatra 웹 애플리케이션**

<br>

[Sinatra](http://www.sinatrarb.com/)라는 가벼운 웹 프레임워크를 사용하기 위해 새로운 폴더를 만들고 `Gemfile`과 `app.rb`를 만든다.

**Gemfile**

```shell
source 'https://rubygems.org'
gem 'sinatra'
```

**app.rb**

```shell
require 'sinatra'
require 'socket'

get '/' do
  Socket.gethostname
end
```

ruby와 sinatra에 대해 전혀 모르더라도 `Gemfile`은 패키지를 관리하고 `app.rb`는 호스트명을 출력하는 웹 서버를 만들었다는 걸 예상할 수 있다.

만약, ruby가 설치되어 있지 않다면 도커가 있으면 문제가 없다! 다음 명령어를 실행해보자.

```shell
docker run --rm \
-p 4567:4567 \
-v $PWD:/usr/src/app \
-w /usr/src/app \
ruby \
bash -c "bundle install && bundle exec ruby app.rb -o 0.0.0.0"
```

호스트의 디렉토리를 루비가 설치된 컨테이너의 디렉토리에 마운트한 다음 그대로 명령어를 실행하면 로컬에 개발환경을 구축하지 않고 도커 컨테이너를 개발 환경으로 사용할 수 있다! Awesome!

서버가 정상적으로 실행되었으면 웹 브라우저에서 테스트해보자. ``http://localhost:4567``

<center><img src="https://github.com/daeuni/daeuni.github.io/blob/master/assets/local1.png?raw=true"></center> 

도커 컨테이너의 호스트명을 확인할 수 있다! 이제 도커 이미지를 만들 준비가 되었다!!!!!!!

<br>

**Ruby Application Dockerfile**


<center><img src="https://github.com/daeuni/daeuni.github.io/blob/master/assets/dockerfile.png?raw=true"></center> 

도커는 이미지를 만들기 위해 `Dockerfile`이라는 이미지 빌드용 DSLDomain Specific Language 파일을 사용합니다. 단순 텍스트 파일로 일반적으로 소스와 함께 관리한다.

고급 개발자는 바로 Dockerfile을 만들 수도 있겠지만, 일반 개발자들은 일단 리눅스 서버에서 테스트로 설치해보고 안 되면 될 때까지 최적의 과정을 Dockerfile로 작성해야 합니다. 

우리는 초보이므로 Ruby 웹 애플리케이션을 ubuntu에 배포하는 과정을 먼저 살펴보겠다.
<br>
<hr>
1. ubuntu설치
2. ruby설치
3. 소스 복사
4. Gem 패키지 설치
5. Sinatra 서버 실행
<hr>
이 과정을 그대로 쉘 스크립트로 옮겨보면 아래와 같다.

```shell
# 1. ubuntu 설치 (패키지 업데이트)
apt-get update

# 2. ruby 설치
apt-get install ruby
gem install bundler

# 3. 소스 복사
mkdir -p /usr/src/app
scp Gemfile app.rb root@ubuntu:/usr/src/app  # From host

# 4. Gem 패키지 설치
bundle install

# 5. Sinatra 서버 실행
bundle exec ruby app.rb
```

ubuntu 컨테이너를 실행하고 위 명령어를 그대로 실행하면 웹 서버를 실행할 수 있다. 리눅스에서 테스트가 끝났으니 이 과정을 Dockerfile로 만들면 된다. 아직 자세한 명령어를 배우진 않았지만 일단 만들어 보자. 핵심 명령어는 파일을 복사하는 `COPY`와 명령어를 실행하는 `RUN`이다.

```dockerfile
# 1. ubuntu 설치 (패키지 업데이트 + 만든사람 표시)
FROM       ubuntu:16.04
MAINTAINER subicura@subicura.com
RUN        apt-get -y update

# 2. ruby 설치
RUN apt-get -y install ruby
RUN gem install bundler

# 3. 소스 복사
COPY . /usr/src/app

# 4. Gem 패키지 설치 (실행 디렉토리 설정)
WORKDIR /usr/src/app
RUN     bundle install

# 5. Sinatra 서버 실행 (Listen 포트 정의)
EXPOSE 4567
CMD    bundle exec ruby app.rb -o 0.0.0.0
```

빌드 파일을 작성했고 이제 이미지를 만들 수 있다.

<br>

**Docker build**


이미지를 빌드하는 명령은 다음과 같다.

```shell
docker build [OPTIONS] PATH | URL | -
```

생성할 이미지 이름을 지정하기 위한 -t(tag)옵션만 알면 빌드가 가능하다.

Dockerfile을 만든 디렉토리로 이동하여 다음 명령어를 입력하면 app이라는 이미지가 생성된다!!

```she
docker build -t app .
```

output:

```shell
Sending build context to Docker daemon 22.02 kB
Step 1/10 : FROM ubuntu:16.04
 ---> f49eec89601e
Step 2/10 : MAINTAINER subicura@subicura.com
 ---> Running in 06f20ac1017d
 ---> fc41cd8ac52d
Removing intermediate container 06f20ac1017d
Step 3/10 : RUN apt-get -y update
 ---> Running in c35738e75a51
Get:1 http://archive.ubuntu.com/ubuntu xenial InRelease [247 kB]
Get:2 http://archive.ubuntu.com/ubuntu xenial-updates InRelease [102 kB]
Get:3 http://archive.ubuntu.com/ubuntu xenial-security InRelease [102 kB]
Get:4 http://archive.ubuntu.com/ubuntu xenial/main Sources [1103 kB]
Get:5 http://archive.ubuntu.com/ubuntu xenial/restricted Sources [5179 B]

... 생략 ...

Step 9/10 : EXPOSE 4567
 ---> Running in 9c514a1f0c8e
 ---> c5ce4376282e
Removing intermediate container 9c514a1f0c8e
Step 10/10 : CMD    bundle exec ruby app.rb -o 0.0.0.0
 ---> Running in aff9f4bb4fdf
Removing intermediate container aff9f4bb4fdf
 ---> 2ac7d879488f
Successfully built 2ac7d879488f
Successfully tagged app:latest
```

빌드 명령어를 실행하면 Dockerfile에 정의한 내용이 한 줄 한 줄 실행되는 걸 볼 수 있습니다. 실제로 명령어를 실행하기 때문에 빌드 시간이 꽤 걸리는 걸 알 수 있다. 

최종적으로 `Successfully built xxxxxxxx` 메시지가 보이면 정상적으로 이미지를 생성한 것이다.

그럼 이미지가 잘 생성되었는지 확인해보자.

```shell
docker images
```

output:

```shell
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
app                 latest              2ac7d879488f        3 minutes ago       213MB
ruby                latest              8fe6e1f7b421        2 days ago          840MB
ubuntu              16.04               5e13f8dd4c1a        3 weeks ago         120MB
```

와우~!!! 드디어 첫번쨰 도커 이미지를 생성했다! 이미지를 잘 생성했으니 잘 동작하는지 컨테이너를 실행해보자!

```shell
docker run -d -p 8080:4567 app
docker run -d -p 8081:4567 app
docker run -d -p 8082:4567 app
```

<center><img src="https://github.com/daeuni/daeuni.github.io/blob/master/assets/local2.png?raw=true"></center> 

접속 성공입니다! 호스트 네임을 출력하는 웹서버를 3개나 만들 수 있습니다!!!

이미지가 잘 만들어졌네요 :)

<br>

**Build 분석**


도커는 Dockerfile을 가지고 어떻게 빌드를 하는 것인지! build 로그를 보면서 분석해볼 수 있다.

```shell
Sending build context to Docker daemon  5.12 kB   <-- (1)
Step 1/10 : FROM ubuntu:16.04                     <-- (2)
 ---> f49eec89601e                                <-- (3)
Step 2/10 : MAINTAINER subicura@subicura.com      <-- (4)
 ---> Running in f4de0c750abb                     <-- (5)
 ---> 4a400609ff73                                <-- (6)
Removing intermediate container f4de0c750abb      <-- (7)
Step 3/10 : RUN apt-get -y update                 <-- (8)
...
...
Successfully built 20369cef9829                   <-- (9)
```

```
(1) Sending build context to Docker daemon  5.12 kB
```

빌드 명령어를 실행한 디렉토리의 파일들을 빌드컨텍스트build context라고 하고 이 파일들을 도커 서버(daemon)로 전송한다. 도커는 서버-클라이언트 구조이므로 도커 서버가 작업하려면 미리 파일을 전송해야 한다.

```
(2) Step 1/10 : FROM ubuntu:16.04
```

Dockerfile을 한 줄 한 줄 수행한다. 첫 번째로 FROM 명령어를 수행한다. ubuntu:16.04 이미지를 다운받는 작업이다.

```
(3) ---> f49eec89601e
```

명령어 수행 결과를 이미지로 저장한다. 여기서는 ubuntu:16.04를 사용하기로 했기 때문에 ubuntu 이미지의 ID가 표시된다.

```
(4) Step 2/10 : MAINTAINER subicura@subicura.com
```

Dockerfile의 두 번째 명령어인 MAINTAINER 명령어를 수행한다.

```
(5) ---> Running in f4de0c750abb
```

명령어를 수행하기 위해 바로 이전에 생성된 `f49eec89601e` 이미지를 기반으로 `f4de0c750abb`컨테이너를 임시로 생성하여 실행한다.

```
(6) ---> 4a400609ff73
```

명령어 수행 결과를 이미지로 저장합니다.

```
(7) Removing intermediate container f4de0c750abb
```

명령어를 수행하기 위해 임시로 만들었던 컨테이너를 제거합니다.

```
(8) Step 3/10 : RUN apt-get -y update
```

Dockerfile의 세 번째 명령어를 수행한다. 이전 단계와 마찬가지로 바로 전에 만들어진 이미지를 기반으로 임시 컨테이너를 만들어 명령어를 실행하고 그 결과 상태를 이미지로 만든다. 이 과정을 마지막 줄까지 무한 반복한다.

```
(9) Successfully built 20369cef9829
```

최종 성공한 이미지 ID를 출력한다.

<br>

결론적으로 도커 빌드는 ``임시 컨테이너 생성`` > ``명령어 수행`` > ``이미지로 저장`` > ``임시 컨테이너 삭제`` > ``새로 만든 이미지 기반 임시 컨테이너 생성`` > ``명령어 수행`` > ``이미지로 저장`` > ``임시 컨테이너 삭제`` > … 의 과정을 계속해서 반복한다고 볼 수 있습니다. 명령어를 실행할 때마다 이미지 레이어를 저장하고 다시 빌드할 때 Dockerfile이 변경되지 않았다면 기존에 저장된 이미지를 그대로 캐시처럼 사용한다.

이처럼 레이어 개념을 잘 이해하고 있으면 최적화된 이미지를 생성할 수 있다.

<br>

<hr>

어떻게 앞에서 만든 이미지를 리팩토링 할 수 있는지 다음 포스트에 이어가도록 하겠습니다.

<hr>

<br>

