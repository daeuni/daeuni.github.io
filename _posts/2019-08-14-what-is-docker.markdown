---
layout: post
title:  "What is Docker?"
date:   2019-08-14 14:27:36
---

도커(Docker)는 컨테이너형 가상화 기술을 구현하기 위한 상주 애플리케이션(dockered라는 데몬)과 이 애플리케이션을 조작하기 위한 명령행 도구(CLI)로 구성되는 프로덕트다.  애플리케이션 배포에 특화돼 있기 때문에 애플리케이션 개발 및 운영을 컨테이너 중심으로 할 수 있다.

<center><img src="https://github.com/daeuni/daeuni.github.io/blob/master/assets/dockerImage.png?raw=true"></center> 
<br>
<hr>
<br>
- **도커의 장점**

  1) 도커는 조작이 간편하고 경량 컨테이너라는 장점 때문에 도커는 로컬 머신의 개발 환경 구축에 널리 사용된다.

  2) 개발 환경 구축뿐만 아니라 개발 후 운영 환경에 대한 배포나 애플리케이션 플랫폼으로 가능할 수 있다.

  3) 테스트 환경뿐만 아니라 운영 환경에서도 컨테이너를 사용할 수 있다.

  4) 이식성이 뛰어나다.

  5) 컨테이너 간의 연동 및 클라우드 플랫폼 지원 등 여러 면에서 장점이 있다.    


<br>
- **도커가 적합하지 않은 경우**

  도커 컨테이너의 내부는 리눅스 계열 운영 체제와 같은 구성을 취하는 것이 대부분인데, 컨테이너는 운영 체제의 동작을 완전히 재연하지는 못한다.

  조금 더 엄밀한 리눅스 계열 운영체제의 동작이 요구되는 가상 환경을 구축해야 한다면 기존 방법대로 VMWare나 VirtualBox같은 가상화 소프트웨어를 사용하는 것이 낫다.

  단지, 도커는 애플리케이션을 배포하는 목적에 특화된 박스라고 생각하는 편이 좋다.    


<br>
<hr>
<br>
**[1] 컨테이너형 가상화 기술**
<br>
<br>
   도커는 컨테이너형 가상화 기술(운영 체제 수준 가상화: Operating-system-level virtualization)을 사용한다.

   컨테이너형 가상화를 사용하면 가상화 소프트웨어 없이도 운영체제의 리소스를 격리해 가상 운영 체제로 만들 수 있다. 이 가상 운영 체제를 <u>컨테이너</u>라고 한다.

   이와 달리, 운영 체제 위에서 가상화 소프트웨어를 사용해 하드웨어를 에뮬레이션하는 방법으로 게스트 운영 체제를 만드는 방식을 호스트 운영체제 가상화(VMWare Player, Oracle VisualBox)라고 한다. 컨테이너 가상화와 비교하면 구조적으로 오버헤드가 더 크다.

   "<u>컨테이너형 가상화 기술을 사용해서 컨테이너를 쉽게 만들고 쉽게 버릴 수 있다는 점이 도커의 주요 특징 중 하나다.</u>"


<center><img src = "https://github.com/daeuni/daeuni.github.io/blob/master/assets/container.png?raw=true">
</center>
  


<br>
**[2] 도커 스타일 체험하기**
<br>
<br>
   설명만으로는 애플리케이션과 실행 환경을 함께 배포하는 도커 스타일의 배포 방식이 잘 이해되지 않을 수도 있다. 

   실제로 도커를 사용해 애플리케이션을 배포하는 코드를 살펴모자. 애플리케이션이 포함된 도커 이미지를 어떻게 만들고, 컨테이너를 어떻게 실행하는지를 간단한 예를 들어 이해해보자.

   1) helloworld.sh

   ```shell
   #!/bin/sh
   
   echo "Hello, World!"
   ```

   이제 이 스크립트를 도커 컨테이너에 담아 보자. DockerFile이나 애플리케이션 실행 파일을 사용해서 도커 컨테이너의 우너형이 될 <u>이미지</u>를 만드는 과정을 "도커 이미지 빌드"라고 한다.

   2) Dockerfile
   
   shell script와 같은 폴더에 작성한다.

   ```dockerfile
   FROM ubuntu:16.04
   
   COPY helloworld /usr/local/bin
   RUN chmod +x /usr/local/bin/helloworld
   
   CMD ["helloworld"]
   ```


<hr>
<img src = "https://github.com/daeuni/daeuni.github.io/blob/master/tagtable.JPG?raw=true">
<br>
<hr>
<br>

   이제 Dockerfile을 사용해 이미지를 빌드하고 실행하자. Dockerfile이 있는 폴더에서 docker image build명령을 실행한다  

   ```dockerfile
   $docker image build -t helloworld:latest .
   Sending build context to Docker daemon 97.5MB
   ```

   빌드가 끝난 다음 docker container run 명령으로 도커 컨테이너를 실행하는 것이 기본 사용법이다.  

   ```dockerfile
   $docker container run helloworld:latest
   Hello, World!
   ```
  
   이런 방식으로 도커 이미지에 애플리케이션에 필요한 파일을 운영체제와 함께 담아 컨테이너 형태로 실행하는 것이 기본적인 스타일이며 본 예제는 셸 스크립트를 우분투 운영체제와 함께 컨테이너로 실행한 것이다.


<br>
**[3] 실용적인 예제**
<br><br>
   앞의 helloworld 예제는 echo명령을 실행하고 종료된다.

   실제 개발에서 도커 컨테이너로 배포되는 대상은 주로 웹 애플리케이션이나 API 애플리케이션처럼 <u>항상 가동되는 것</u>들이다.

   예를들어 Node.js 웹 애플리케이션을 배포 대상으로 하는 상황에서 애플리케이션은 계속 실행된 상태로 남아있다. 애플리케이션이 의존하는 Node.js 버전의 기반 이미지를 이용하되, npm으로 모듈을 추가설치하거나 애플리케이션 빌드를 컨테이너 안에서 수행해 이미지를 만든다.

   완성된 이미지는 도커가 실행되는 환경이라면 어떤 환경에서도 실행할 수 있으며 호스트 운영체제에 Node.js나 npm을 설치할 필요가 전혀 없다 !!
<br>
<br>
<hr>
실제 사용 예와는 조금 차이가 있지만, 지금까지의 설명으로 그럭저럭 도커를 어떻게 사용하는지 감을 잡고 더 실용적이 도커 사용법은 다음 포스트에 계속 이어나갈 예정이다.
<hr>
<br>
<br>
