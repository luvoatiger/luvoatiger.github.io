---
layout: post
title: Docker로 WAS, APM Agent 확장하기
date: 2021-11-26
excerpt: "How to expand was and apm using docker"
tags: [Docker, APM]
comment: true
---



고객사 시스템을 모방한 데모 모니터링 시스템을 개발하고 운영 중이다. 프로젝트를 나가면서 데모 시스템의 인스턴스를 쉽게 확장할 수 있으면 실제 고객사 환경과 비슷한 규모의 부하를 일으킬 수 있다고 생각했다. 이번에 도커를 이용해서 WAS와 APM Agent를 이미지로 패키징 해 보았다.

먼저 도커를 설치한다.

```bash
계정 로그인
ssh user@HostPcIpAddress

## 도커 명령어들은 root 권한이 필요하므로, 사용자 계정에 sudo 권한 부여가 먼저 필요함

## 도커 설치 여부 확인
$ docker ## command not found가 아니라 명령어가 작동된다면, 이미 설치되어 있는 것. 깔끔하게 재설치하고 싶다면, 아래의 명령어들을 이용해서 docker 관련된 내용들을 전부 지워준다.

------------------------------------------------------------- Optional -------------------------------------------------------------
$ sudo systemctl stop docker
$ sudo systemctl stop containerd
$ sudo yum remove $(sudo yum list installed | grep docker | awk '{print $1}')
$ sudo yum remove docker-ce docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
 
$ sudo rm -rf /var/lib/docker
$ sudo docker stop <container ID> # 특정 도커 컨테이너 ID
$ sudo docker stop $(docker ps -q)  # 전체 실행 중인 도커 프로세스
$ sudo docker rm <Container ID> # 도커 컨테이너 삭제
$ sudo docker rm $(docker ps -a -q) # 전체 도커 컨테이너 삭제
------------------------------------------------------------- Optional -------------------------------------------------------------

## 도커 설치 쉘 스크립트 다운로드 및 실행
$ curl -fsSL https://get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh
$ sudo usermod -aG docker ${USER}
$ docker version ##이런 식으로 Client와 Server가 뜨면 정상. 만약 Server가 뜨지 않고, 도커 데몬 미실행 메세지 및 데몬 접근 권한 없음 문제가 발생하면 아래에 정리한 내용대로 처리
Client: Docker Engine - Community
 Version:           20.10.11
 API version:       1.41
 Go version:        go1.16.9
 Git commit:        dea9396
 Built:             Thu Nov 18 00:38:53 2021
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.11
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.16.9
  Git commit:       847da18
  Built:            Thu Nov 18 00:37:17 2021
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.4.12
  GitCommit:        7b11cfaabd73bb80907dd23182b9347b4245eb5d
 runc:
  Version:          1.0.2
  GitCommit:        v1.0.2-0-g52b36a2
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0

## 서버 데몬 관련 에러 발생 시
1. Docer Daemon Running 메세지 나올 경우
$ sudo systemctl start docker
$ sudo systemctl enable docker
2. Connection Refused가 뜨는 경우
$ sudo usermod -aG docker ${USER}

## 완료해도 재로그인 하지 않으면, 똑같이 Server Daemon 오류가 발생한다. 해당 터미널을 종료 후 재로그인 하면, 위의 Server Docker Engine이 정상적으로 뜨는 것을 알 수 있다.

## docker-compose 설치
$ sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
$ sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
$ docker-compose -version

## Error response from daemon: toomanyrequests: You have reached your pull rate limit. You may increase the limit by authenticating and upgrading: https://www.docker.com/increase-rate-limit가 뜨는 경우는 docker 정책 상 해당 IP에서 docker pull 할 수 있는 횟수를 초과해서 그렇다고 한다. docker 요금제를 바꾸거나 6시간 정도 대기를 하면 풀리는 것 같다.

```

WAS와 APM Agent를 하나의 이미지로 가상화하고 기동한다. WAS는 상용 솔루션이 아닌 오픈 소스인 Tomcat을 골랐다. 상용 솔루션을 오픈소스화한다면 개발회사에서는 이익이 없을테니 모든 기능을 제공하지는 않을 것으로 판단했다.


```bash
docker build -t tomcat-intermax
docker save -o tomcat-intermax.tar tomcat-intermax
docker load -i tomcat-intermax.tar
docker run -p 13200:8080 -it tomcat-intermax
vi /intermax/apache-tomcat-7.0.94/bin/catalina.sh
export JAVA_OPTS="$JAVA_OPTS -noverify -Djspd.wasid={에이전트ID} -javaagent:/intermax/WAS_Agent/intermax/jspd/lib/jspd.jar -Xbootclasspath/p:/intermax/WAS_Agent/intermax/jspd/build-rt/aspectjweaver-1.8.9.jar:/intermax/WAS_Agent/intermax/jspd/lib/jspd-rt.jar"
sh /intermax/apache-tomcat-7.0.94/bin/catalina.sh start
```


pstree 명령어로 확인해보니, 대략 30MB 정도의 메모리를 사용하므로, 필요 시 인스턴스를 쉽게 확장할 수 있을 것 같다.
