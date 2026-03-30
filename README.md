# 🐳 Docker Compose 기반 Spring Boot + MySQL 헬스체크 실습

---

## 👥 Contributors
|       배기영       |
| :-----------------: |
| [<img width="160px" src="https://github.com/bbky323.png">](https://github.com/bbky323) |
| [@bbky323](https://github.com/bbky323) |

---

## 📌 1. 실습 목적

Spring Boot와 MySQL을 Docker Compose로 함께 실행할 때,  
단순히 컨테이너가 실행되는 것만으로는 서비스가 정상 준비되었다고 볼 수 없다.

이번 실습의 목적은 다음과 같다.

- Docker Compose로 Spring Boot와 MySQL을 함께 구성한다.
- MySQL에 `healthcheck`를 적용해 DB 준비 상태를 확인한다.
- Spring Boot가 MySQL이 준비된 뒤 실행되도록 설정한다.
- MySQL 장애 시 app 컨테이너 동작을 확인한다.
- Docker Compose의 한계를 이해하고 Kubernetes 필요성과 연결한다.

---

## 📝 2. 핵심 개념

### 2.1 running과 healthy의 차이

컨테이너가 **running** 상태여도 서비스가 바로 사용 가능한 것은 아니다.  
특히 MySQL은 실행 직후 초기화 시간이 필요하므로, Spring Boot가 너무 빨리 뜨면 DB 연결 실패가 발생할 수 있다.

### 2.2 healthcheck

**healthcheck**는 컨테이너 내부 서비스가 정상 동작 중인지 확인하는 기능이다.  
MySQL은 **mysqladmin ping** 명령으로 상태를 확인할 수 있다.

### 2.3 depends_on의 한계

**depends_on**은 기본적으로 시작 순서만 보장한다.  
실제 준비 상태까지 고려하려면 **condition: service_healthy**를 함께 사용해야 한다.

---

## 🚀 3. 실습 구성

### docker Image
#### 기존 jar파일 소스 코드

```JAVA
server.servlet.context-path=/emp server.port=8083 # JSP 파일 경로 설정
spring.mvc.view.prefix=/WEB-INF/views/ spring.mvc.view.suffix=.jsp

# 1. Database Connection
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://mysqldb:3306/fisa
serverTimezone=Asia/Seoul&useSSL=false&allowPublicKeyRetrieval=true
spring.datasource.username=user01 spring.datasource.password=user01
```
#### Dockerfile
```Dockerfile
FROM eclipse-temurin:17-jre
COPY step04_empApp-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```
- 위 두 파일을 통해 okcoco03/emp-app:latest라는 Dokcer Image 생성

### docker-compose2.yml
```YAML
version: "3.8"

services:
  mysqldb:
    image: mysql:8.0
    container_name: mysqldb // default 컨테이너명은 <폴더명>_서비스명_번호!! 잘못된 예시
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: fisa
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -u root -proot || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

  app:
    image: okcoco03/emp-app:latest
    container_name: emp-app
    ports:
      - "8083:8083"
    depends_on:
      mysqldb:
        condition: service_healthy
```

- container 이름의 경우 <폴더명>_서비스명_번호로 하는 게 기본이지만, 위 실습에서는 기존 이미지를 활용하기 위해 mysqldb로 설정함.
- 또한 Docker Compose는 기본적으로 **같은 compose 파일 안의 서비스들을 자동으로 같은 네트워크에 붙여**주기 때문에 위에선 굳이 네트워크 설정을 하지 않았음.
  - 위 실습에서는 **healthcheck, depends on, 장애 확인**이 핵심이라서 네트워크 설정 생략!
  - 마찬가지로 volume도 같은 맥락에서 설정 생략!
  - environment 같은 경우에는 이미 이미지 파일에 포함되어 있기 때문에 생략됨

---

## 📕 4. 실습 시나리오
### 4-1. 정상 실행 확인
#### 컨테이너 생성 및 상태 확인
```bash
docker compose -f docker-compose2.yml up -d // 컨테이너 생성
docker compose -f docker-compose2.yml ps // 실행중인 컨테이너 확인 
```
<img width="735" height="209" alt="image" src="https://github.com/user-attachments/assets/23d8580b-7ad0-4ca8-91f7-716039dd0e52" />

- mysqldb 상테가 healthy
- app 컨테이너가 Up 상태

#### 상세 로그 확인
```bash
docker compose -f docker-compose2.yml logs mysqldb
docker compose -f docker-compose2.yml logs app
```
<img width="736" height="95" alt="image" src="https://github.com/user-attachments/assets/2fd7257b-0093-42c2-876a-7f032abba9e4" />

<img width="726" height="74" alt="image" src="https://github.com/user-attachments/assets/312ba88b-29ce-499e-b3eb-26bb9e12d82a" />

<img width="716" height="112" alt="image" src="https://github.com/user-attachments/assets/e492c4be-b76b-45cc-b6f2-86edadd1c3ae" />

<img width="716" height="130" alt="image" src="https://github.com/user-attachments/assets/dfae7563-30ae-4813-9cdc-2f0a6ee90273" />


- mysql 로그에서 정상 기동
- app 로그에서 DB 연결 성공 후 Spring Boot 실행 완료

#### URL을 브라우저에 입력
<img width="930" height="257" alt="image" src="https://github.com/user-attachments/assets/4a1c1080-8c22-4709-83b3-5a7b4c299670" />

- DB에 데이터가 없어 아무것도 없지만 연결은 된 모습!
> **결론: 위 결과들로 보아, app과 mysql은 정상적으로 구동 중임을 확인 가능!**

### 4-2. MySQL 장애 확인
#### mysql 컨테이너 중지
```bash
docker compose -f docker-compose2.yml stop mysqldb
```
<img width="734" height="107" alt="image" src="https://github.com/user-attachments/assets/6f335e9a-b3d3-45d8-abba-b72070593bb8" />

#### 컨테이너 상태 확인
```bash
docker compose -f docker-compose2.yml ps
```
<img width="726" height="158" alt="image" src="https://github.com/user-attachments/assets/17bf427a-e05b-4f6a-b802-21ea5e99ee91" />

- mysqldb 컨테이너는 보이지 않고, app만 구동중임을 확인

#### app 로그 확인
```bash
docker compose -f docker-compose2.yml logs app
```
<img width="731" height="267" alt="image" src="https://github.com/user-attachments/assets/74080f2b-2f4d-417e-b2ac-9874269d977c" />

- mysqldb가 중지됨으로써, app에서 계속 DB 연결 실패 메세지가 발생함

> **결론: depends_on: condition: service_healthy는 초기 발생 시점에는 유효하지만, 실행 중 MySQL 장에에 대해 app을 자동 복구하지는 못한다는 점을 확인할 수 있다.**

---

## 📚 5. 검증 결과 정리
### 5-1. MySQL healthcheck 확인

MySQL 컨테이너에 healthcheck를 설정한 결과, 단순히 컨테이너가 실행 중인지가 아니라 실제로 DB가 응답 가능한 상태인지를 기준으로 상태를 확인할 수 있었다.

### 5-2. Spring Boot 실행 확인

Spring Boot 컨테이너는 MySQL이 healthy 상태가 된 이후 실행되었으며, 로그를 통해 DB 연결 이후 애플리케이션이 정상 가동된 것을 확인할 수 있었다.

### 5-3. MySQL 장애 상황 확인

실행 중이던 MySQL 컨테이너를 중지한 뒤에도 app 컨테이너는 자동 종료되지 않았다.
즉, Docker Compose의 healthcheck와 depends_on은 초기 실행 순서 제어에는 유용하지만, 런타임 장애에 대한 자동 복구까지는 처리하지 못했다.

---

## 🔎 6. 고찰
Docker Compose는 Spring Boot와 MySQL 같은 멀티 컨테이너 환경을 간단하게 구성하고,
**healthcheck**와 **depends_on**을 통해 초기 실행 순서를 제어하는 데 유용했다.

특히 MySQL이 실제로 준비된 이후 app이 실행되도록 설정함으로써, 초기 DB 연결 실패 가능성을 줄일 수 있었다.

하지만 MySQL이 실행 중 중단되었을 때 app 컨테이너를 자동으로 재시작하거나 복구하지는 못했다.
즉, Docker Compose는 개발 및 실습 환경에서는 충분히 유용하지만, 운영 환경에서 요구되는 자동 복구와 self-healing에는 한계가 있다.

이러한 한계를 보완하기 위해 이후에는 **Kubernetes**를 사용할 수 있다.

---

## 🛠️ 7. 트러블슈팅
### 문제1.
#### 상황
**docker compose up -d**를 실행했을 때, 의도한 **docker-compose2.yml**이 아니라 이전에 사용하던 **docker-compose.yml** 기준으로 컨테이너가 생성되었다.
그 결과 원하지 않은 db 컨테이너가 다시 생성되는 문제가 발생했다.

#### 원인

**Docker Compose**는 별도 파일을 지정하지 않으면 기본적으로 **docker-compose.yml** 파일을 우선적으로 사용한다.
같은 디렉토리에 **docker-compose.yml**과 **docker-compose2.yml**이 함께 존재했기 때문에, 단순 **docker compose up -d** 실행 시 기본 파일이 적용되었다.

#### 해결 방법

실행할 compose 파일을 명시하여 다음과 같이 실행하였다.
```bash
docker compose -f docker-compose2.yml up -d
```
중지할 때도 동일하게 파일을 지정하여 사용하였다.

### 문제2.
#### 상황
<img width="513" height="246" alt="image" src="https://github.com/user-attachments/assets/71d1c2b2-715c-45d6-ad5a-faa8965bb6ae" />

MySQL 컨테이너 실행 시 **bind: address already in use** 에러가 발생하며 3306 포트 바인딩에 실패했다.

#### 원인
호스트 서버의 3306 포트를 이미 다른 MySQL 서비스가 사용 중이었기 때문이다.

#### 해결 방법
기존 VM 로컬에서 돌아가던 MySQL 서비스를 **sudo systemctl stop mysql** 명령어를 통해 정지하여 3306포트를 컨테이너가 점유하도록 했다.
