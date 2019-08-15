---
layout: post
title:  "Run Container"
description: Experience the Power of Docker
date:   2019-08-15 10:20:36
categories: Ubuntu MySQL Tensorflow
---
도커를 Mac 또는 Window에 설치하고 난 뒤에 정상적으로 설치되었는지 명령어를 통해 확인하자.

```shell
docker version
```

output:

```shell
Client:
 Version:      1.12.6
 API version:  1.24
 Go version:   go1.6.4
 Git commit:   78d1802
 Built:        Wed Jan 11 00:23:16 2017
 OS/Arch:      darwin/amd64

Server:
 Version:      1.12.6
 API version:  1.24
 Go version:   go1.6.4
 Git commit:   78d1802
 Built:        Wed Jan 11 00:23:16 2017
 OS/Arch:      linux/amd64
```

버전 정보가 클라이언트와 서버로 나뉘어져 있따는 것을 확인할 수 있다. 도커는 하나의 실행파일이지만, 실제로 클라이언트와 서버 역할을 각각 할 수 있다. 도커 커맨드를 입력하면 도커 클라이언트가 도커 서버로 명령을 전송하고, 결과를 받아 터미널에 출력해주는 구조이다.

기본 값이 도커 서버의 소켓을 바라보고 있기 때문에 사용자는 의식하지 않고 마치 바로 명령을 내리는 것 같은 느낌을 받는다. 이러한 설계가 mac이나 windows의 터미널에서 명령을 입력했을 때 가상 서버에 설치된 도커가 동작하는 이유이다.

<center><img src="https://github.com/daeuni/daeuni.github.io/blob/master/assets/clientserver.png?raw=true"></center> 

<br>

<hr>

<br>

**"컨테이너 실행하기"**

<br>

도커 컨테이너에 여러개의 프로그램을 손쉽게 띄워보는 예제이다.

도커를 실행하는 명령어는 다음과 같다.

```shell
docker run [OPTIONS] IMAGE[:TAG|@DIGEST] [COMMAND] [ARG...]
```

다음은 자주 사용하는 옵션이다.

<img src="https://github.com/daeuni/daeuni.github.io/blob/master/assets/optiontable.PNG?raw=true">

<br>

**1) ubuntu 16.04 container**

```shell
docker run ubuntu:16.04
```

`run`명령어를 사용하면 사용할 이미지가 저장되어 있는지 확인하고 없다면 다운로드(`pull`)를 한 후 컨테이너를 생성(`create`)하고 시작(`start`) 합니다.

`ubuntu:16.04` 이미지를 다운받은 적이 없다면 이미지를 다운로드 한 후 컨테이너가 실행된다. 컨테이너는 정상적으로 실행되었찌만 뭘 하라는 명령어를 전달하지 않기 때문에 생성되자마자 종료된다. 

컨테이너는 프로세스이기 때문에 실행중인 프로세스가 없으면 컨테이너는 종료된다.

이번에는  `/bin/bash` 명령어를 입력해서 `ubuntu:16.04` 컨테이너를 실행해보자.

```shell
docker run --rm -it ubuntu:16.04 /bin/bash

# in container
$ cat /etc/issue
Ubuntu 16.04.1 LTS \n \l

$ ls
bin   dev  home  lib64  mnt  proc  run   srv  tmp  var
boot  etc  lib   media  opt  root  sbin  sys  usr
```

컨테이너 내부에 들어가기 위해 bash 쉘을 실행하고 키보드 입력을 위해 `-it` 옵션을 줍니다. 추가적으로 프로세스가 종료되면 컨테이너가 자동으로 삭제되도록 `--rm` 옵션도 추가하였다.

이번에는 바로 전에 이미지를 다운 받았기 때문에 이미지를 다운로드 하는 화면 없이 바로 실행되었다. `cat /etc/issue`와 `ls`를 실행해보면 ubuntu 리눅스인걸 알 수 있다. 뭔가 가벼운 가상머신 같은 느낌이다.

`exit`로 bash 쉘을 종료하면 컨테이너도 같이 종료된다.

도커를 이용하여 가장 기본적인 컨테이너를 순식간에 만들어보았다.

<br>

**2) MySQL 5.7 container**

두번째 실행할 컨테이너는 MySQL 서버이다. 가장 흔하게 사용하는 데이터베이스로 `-e` 옵션을 이용하여 환경변수를 설정하고 `--name` 옵션을 이용하여 컨테이너에 읽기 어려운 ID 대신 쉬운 이름을 부여할 예정이다.

[MySQL Docker hub](https://hub.docker.com/_/mysql/) 페이지에 접속하면 간단한 사용법과 환경변수에 대한 설명이 있다. 여러가지 설정값이 있는데 패스워드 없이 root계정을 만들기 위해 `MYSQL_ALLOW_EMPTY_PASSWORD` 환경변수를 설정한다. 그리고 컨테이너의 이름은 `mysql`로 할당하고 백그라운드 모드로 띄우기 위해 `-d` 옵션을 줍니다. 포트는 `3306포트`를 호스트에서 그대로 사용하려고 한다.

```shell
docker run -d -p 3306:3306 \
  -e MYSQL_ALLOW_EMPTY_PASSWORD=true \
  --name mysql \
  mysql:5.7

# MySQL test
$ mysql -h127.0.0.1 -uroot

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

mysql> quit
```

순식간에 MySQL 서버가 실행되었다. 이번 테스트는 호스트 OS에 MySQL 클라이언트가 설치되어야 한다. 

<br>

**3) tensorflow**

[tensorflow](https://www.tensorflow.org/)는 손쉽게 머신러닝을 할 수 있는 툴이다. tensorflow는 python으로 만들어져 python과 관련 패키지를 설치해야한다. 이번에 설치하는 이미지는 python과 함께 numpy, scipy, pandas, jupyter, scikit-learn, gensim, BeautifulSoup4, Tensorflow가 설치되어 있다. 뭔가 복잡해 보이지만 도커라면 손쉽게 실행해 볼 수 있다.

```shell
docker run -d -p 8888:8888 -p 6006:6006 teamlab/pydata-tensorflow:0.1
```

설치된 파일이 많아 다운로드 하는데 시간이 좀 걸린다. 컨테이너가 실행되면 웹 브라우저에서 jupyter에 접속하여 머신러닝을 시작해 보자!

<center><img src="https://github.com/daeuni/daeuni.github.io/blob/master/assets/tensorflow.PNG?raw=true"></center> 

여기까지 ubuntu, MySQL, tensorflow를 실행해보았다. 가상머신을 이용해 동일한 작업을 한다면 컴퓨터가 엄청 버벅이겠지만 컨테이너 기반의 도커를 이용하여 매우 가볍게 실행하고 있다. 

<br>

<hr>

다음으로는 내가 만든 애플리케이션을 컨테이너로 실행해서 배포하는 과정을 해보도록 하겠다.

<hr>

<br>

