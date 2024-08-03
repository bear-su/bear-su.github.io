---
title: 환경변수를 이용해 도커 및 도커 컴포즈 파일 관리하기
excerpt: 프로젝트에서 다양한 환경 설정을 위해 많은 Dockerfile을 관리해야 하는 문제를 해결하는 방법을 소개합니다. 환경별로 중복된 Dockerfile 작성의 번거로움을 줄이고, 동적 변수 주입과 Docker Compose를 활용하여 효율적인 컨테이너 환경을 구축하는 방법을 설명합니다.
categories: [ msa ]
tags: [ Docker, Dockerfile, Docker Compose, DevOps, 환경 설정 자동화, 동적 변수 주입, .env 파일, 컨테이너 관리, 소프트웨어 개발, 개발 환경 구축 ]
image:
  path: https://i.imgur.com/cM2E2fY.png
---


## 많아지는 도커파일
프로젝트를 진행하다보면 `로컬`, `개발`, `운영` 세 가지 환경을 기본으로 가지고 간다. 그렇게 하다보면 dockerfile도 세 가지가 필요하다. 

< Dockerfile-local.yml >
```dockerfile
FROM eclipse-temurin:17  
WORKDIR /app  
COPY build/libs/gateway-service.jar app.jar  
CMD ["java", "-Dspring.profiles.active=local", "-jar", "app.jar"]
```

< Dockerfile-dev.yml >
```dockerfile
FROM eclipse-temurin:17  
WORKDIR /app  
COPY build/libs/gateway-service.jar app.jar  
CMD ["java", "-Dspring.profiles.active=dev", "-jar", "app.jar"]
```

이런 방식의 도커파일에는 다음과 같은 문제가 있다.
1. `-Dspring.profiles.active`를 제외한 모든 부분이 중복된다. 따라서 특정 부분이 바뀌면 환경 파일별로 복사/붙여넣기 작업을 여러번 반복해야 한다. 
2. 환경별로 Dockerfile을 여러개 만들어야 한다.

## 동적 변수 주입을 활용한 문제 해결
Dockerfile을 실행할 때 동적으로 값을 넣어줄 수 있다. 
- 변수를 지정할 때는 `ENV` 키워드를 사용한다. 
- 변수를 사용할 때는 `${변수 이름}`을 사용한다.

```dockerfile
FROM eclipse-temurin:17  
WORKDIR /app  
COPY build/libs/gateway-service.jar app.jar  
ENV SPRING_PROFILES_ACTIVE=${SPRING_PROFILES_ACTIVE}
CMD ["java", "-Dspring.profiles.active=${SPRING_PROFILES_ACTIVE}", "-jar", "app.jar"]
```

### 값을 주입하는 방법
1. Docker 이미지 빌드: `docker build -t gateway-service`
2. Docker 컨테이너 실행 `docker run -e SPRING_PROFILES_ACTIVE=local gateway-service

## docker-compose 통합 
보통 docker를 사용하면 docker-compose도 같이 사용한다. docker-compose에서 dockerfile을 활용해서 컨테이너를 실행할 때 어떻게 변수에 값을 주입할 수 있는지 알아보자.

docker-compose에서는 `environment`에 변수와 값을 지정해 줄 수 있다.
```
services:
	gateway-service:  
	  container_name: gateway-service  
	  build:  
	    dockerfile: dockerfile
	  environment:  
	    SPRING_PROFILES_ACTIVE: local
	  ports:  
	    - '8000:8000'  
```

### docker-compose 변수 주입하기
docker-compose를 사용하면 dockerfile을 사용했을 때와 같은 문제가 발생한다. 환경 별로 여러개의 docker-compose파일을 만들어야 한다는 것. 따라서 `SPRING_PROFILES_ACTIVE` 값도 변수로 만들어 관리해보자.


#### .env 활용하기
.env 파일을 사용하여 환경 변수를 정의하고 docker-compose 파일에서 참조할 수 있다.
< docker-compose.yml>
```
services:
	gateway-service:  
	  container_name: gateway-service  
	  build:  
	    dockerfile: dockerfile
	  environment:  
	    SPRING_PROFILES_ACTIVE: ${SPRING_PROFILES_ACTIVE}
	  ports:  
	    - '8000:8000'  
```

< .env.local >
```
SPRING_PROFILES_ACTIVE=local
```

실행할 때는 `--env-file {파일명}`  키워드를 활용하면 된다.
```
docker-compose --env-file .env.local up -d 
```

#### 실행할 때 환경 변수 주입하기
```
SPRING_PROFILES_ACTIVE=local docker-compose up -d
```
