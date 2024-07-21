---
title: "GitHub Actions 배포 자동화 (with 맞이한 에러들)"
tags: Frontend
---

## 들어가기 앞서

배럴아이스캔은 도커를 사용해 배포하고 있다. 개발이 완료되면 도커 명령어를 통해 빌드와 배포를 하는데, 배포가 끝나면 5분 가량이 지나버린다. 가끔은 이런 환경이 ‘어? 나 명령어 다 기억하네’ 하고 지나가지만, 바로 해결하지 않으면 안 되는 (봇이 우리 자원을 가져간다거나) 급박한 상황에서는 이게 참 곤욕이다. 그래서 남들이 다하는 CI/CD 자동화를 통해 푸시해놓고 다른 일도 볼 수 있는 시간을 만들어보려고 한다.

## 구성

### 배럴아이스캔 파이프라인 구성해야 될 것
배럴아이스캔은 세 가지 요소를 거쳐야 개발 완료된 배럴아이스캔 화면을 볼 수 있다.

- E2E 시나리오 **테스트**
- 도커를 통한 **빌드**
- 도커를 통한 **배포**


### 깃허브 액션이란

GitHub Actions는 GitHub 리포지토리에 CI/CD(Continuous Integration/Continuous Deployment) 기능을 추가할 수 있는 강력한 도구이다. 이를 통해 코드 변경 사항을 자동으로 테스트하고 배포할 수 있다. GitHub Actions 워크플로우는 YAML 파일 형식으로 작성되며, 여러 작업(job)과 단계(step)으로 구성된다.

### 처음 해야 될 것

액션을 사용해보고 싶은 레포지토리에 들어가 액션 탭을 통해 이 화면을 볼 수 있다. 배럴아이스캔은 도커를 통해 이미지를 생성/삭제하는 처리를 할 것이기에 Docker image를 통해 시작해본다.

![Monosnap Create new workflow · barreleye-labs_barreleye 2024-07-21 14-49-19](https://github.com/user-attachments/assets/6cb8974b-09cc-489b-9666-91108ec30055)

### yaml 파일 파헤치기

Docker image의 yml 구성이다. 작성하기 전 Github Actions 워크플로우의 구조와 개념에 대해 정리한다.

![Monosnap New File at _ · barreleye-labs_barreleye 2024-07-21 14-51-23](https://github.com/user-attachments/assets/acaa0070-06bc-4852-8f27-8daf8a6869f9)

**주요 구성 요소**

- 워크플로우(Workflow): 워크플로우는 일련의 작업을 정의하는 YAML 파일이다. 작성한 파일은 .github/workflows 디렉토리에 저장된다.
- 이벤트(Event): 워크플로우를 트리거하는 조건이다. 예를들면 코드 푸시, 풀리퀘스트 생성할 경우 등 조건을 줄 수 있다.
- 작업(Job): 워크플로우 내에서 실행되는 독립적인 작업 단위이다. 여러 작업이 병렬로 실행될 수 있다. 각 작업은 특정한 런너(호스트 머신)에서 실행된다.
- 단계(Step): 작업 내에서 순차적으로 실행되는 개별 명령 또는 액션이다. 각 단계에 쉘 스크립트 명령어나 오픈소스를 개발한 사람들의 액션을 사용할 수 있다.
    - 예: run: npm install, uses: actions/checkout@v2
- 런너(Runner): 워크플로우의 작업을 실행하는 머신이다. 배럴아이스캔은 서버 환경은 ubuntu여서 이것을 넣었다.

## 테스팅 YAML 작성하기

### 스크립트

```yaml
name: E2E Test

# 워크플로우를 트리거하는 이벤트 정의
on:
  push:
    branches:
      - develop
  pull_request:
    branches:
      - develop

# 작업 정의
jobs:
  cypress-run:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2 # 액션을 사용하여 리포지토리를 체크아웃

      - name: Set up Node.js
        uses: actions/setup-node@v2 # Node.js 버전을 20.15.0으로 설정
        with:
          node-version: '20.15.0'

      - name: Install dependencies
        run: npm install

      - name: Start the application
        run: npm run dev & # 백그라운드에서 실행하여 애플리케이션을 시작
        env:
          CI: true

      - name: Wait for application to be ready
        uses: jakejarvis/wait-action@v0.1.0 # 액션을 사용하여 일정 시간 동안 대기
        with:
          time: '30' 

      - name: Run Cypress tests
        run: npm run cypress # Cypress 테스트를 실행
```

**npm run dev 뒤에 `&` 의미**

`&` 를 뒤에 붙이면 백그라운드에서 실행시키고 명령어가 실행되는 동안 터미널이 다음 명령어를 받을 수 있도록 한다. 이렇게 하면 개발 서버가 실행되는 동시에 다른 작업도 진행할 수 있게 되어, 병렬로 작업을 처리할 수 있다.


<br/>

## 위 YAML 파일을 푸시하면 깃허브에서 확인할 수 있는 것

브랜치 `develop`에서 푸시되면 이벤트가 트리거되도록 지정해놓았다. 코드 푸시 후 GitHub Actions 탭에 들어가면 자동으로 테스트 코드가 실행되고 있는 것을 확인할 수 있다.

```yaml
on:
  push:
    branches:
      - develop
```

<br/>


**Github Actions 탭에서 확인하기**

![3](https://github.com/user-attachments/assets/eaca6406-6cf6-4e98-97b9-1903653b054a)

<p align="center">
<img src="https://github.com/user-attachments/assets/3009945e-49d7-4c5d-9f0e-a7f79c405385">
</p>


## 빌드 YAML 작성하기


### 빌드 파이프라인 구성

1. **코드 체크아웃:** `actions/checkout@v2` 액션을 사용하여 리포지토리의 코드를 체크아웃
2. **Docker Hub 로그인:** GitHub Secrets을 사용하여 Docker Hub에 로그인합니다.
3. **모든 Docker 이미지 제거**: CI 서버에서 현재 존재하는 모든 Docker 이미지를 제거
4. **Docker 이미지 빌드:** 현재 디렉토리에 있는 Dockerfile을 사용하여 Docker 이미지를 빌드
5. **Docker 이미지 푸시:** 빌드된 Docker 이미지를 Docker Hub에 푸시

### 스크립트

```yaml
name: E2E Test and Build

# 워크플로우를 트리거하는 이벤트 정의
on:
  push:
    branches:
      - develop
  pull_request:
    branches:
      - develop

# 작업 정의
jobs:
  cypress-run:
    runs-on: ubuntu-latest
    
# ===생략===
build:
  runs-on: ubuntu-latest
  needs: cypress-run # cypress-run 작업이 성공적으로 완료된 후 이 작업이 실행된다
  steps:
    - name: Checkout code
      uses: actions/checkout@v2 # 액션을 사용하여 리포지토리를 체크아웃

    - name: Login to Docker Hub # Docker Hub에 로그인
      run: |
        docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_PASSWORD }}

    - name: Remove all Docker images # 모든 Docker 이미지를 제거
      run: |
        docker rmi $(docker images -qa)

    - name: Build Docker image # 현재 디렉토리에 있는 Dockerfile을 사용하여 이미지를 빌드
      run: |
        docker build -t ${{ secrets.USERNAME }}/${{ secrets.DOCKER_REPO }} .

    - name: Push Docker image to Docker Hub # Docker 이미지를 Docker Hub에 푸시
      run: |
        docker push ${{ secrets.USERNAME }}/${{ secrets.DOCKER_REPO }}
```

<br/>


### 민감한 정보 GitHub Actions Secrets 사용하기

> GitHub Actions Secrets는 워크플로우에서 민감한 정보를 안전하게 관리하고 사용할 수 있는 강력한 도구입니다. 이를 통해 보안성을 유지하면서도 자동화된 CI/CD 파이프라인을 구성할 수 있습니다.
> 

<p align="center">
<img width="601" alt="5" src="https://github.com/user-attachments/assets/8b28f468-b5d8-421d-9ee1-36f0f0e26471">
</p>


<br/>

### 꼭 액션을 통해 체크아웃 해주기


> **`actions/checkout@v2`란?** <br/><br/>
> GitHub Actions에서 actions/checkout@v2 액션은 리포지토리의 코드를 CI 서버로 쉽게 내려받고 특정 브랜치로 전환하는 과정을 자동화합니다.
>

체크아웃을 하지 않아서 시간을 많이 잡아먹었는데, 아래와 같이 빌드하는 과정 중에 Dockerfile을 찾을 수 없다는 로그가 찍혔다. (맨날 이렇게 빌드했는데… 왜 그럴까) <br/>
해결책은 GitHub에 올려둔 코드를 CI 서버에 내려받은 후, 특정 브랜치로 전환하는 과정이 필요했다. 
이 두 가지를 한 번에 해주는 `actions/checkout@v2`를 사용하면 Dockerfile을 못 찾는 문제가 해결된다.

```yaml
- name: Build Docker image
  run: |
  # 현재 디렉토리(.)에 있는 Dockerfile을 사용하여 Docker 이미지를 빌드
    docker build -t ${{ secrets.USERNAME }}/${{ secrets.DOCKER_REPO }} . 
```
<p align="center">
<img  src="https://github.com/user-attachments/assets/dec6a94f-1e4b-4b3d-81fc-39b57fef1f0c">
</p>


<br/>


### **`actions/checkout@v2` 분석하기**

YAML 파일에 **`actions/checkout@v2`** 추가한 후, 다시 Github Actions를 확인해보자. 이 액션 하나로 별도의 스크립트 작성 없이 간편하게 코드베이스를 CI 서버로 가져올 수 있다.

<p align="center">
<img width="814" alt="Monosnap ci: Update yml · barreleye-labs:barreleyescan@06e83e9 2024-07-21 18-57-33" src="https://github.com/user-attachments/assets/663a69b5-dbec-431d-9202-93021877af82">
</p>

- **Initializing the repository**: 리포지토리 초기화
- **Disabling automatic garbage collection**: Git의 자동 가비지 컬렉션 기능을 비활성화(체크아웃 성능을 최적화하기 위해 필요)
- **Setting up auth**: 리포지토리 접근 권한을 설정하여 인증을 처리
- **Fetching the repository**: 원격 리포지토리에서 데이터 fetch. 필요한 커밋, 브랜치 또는 태그 정보를 로컬로 복사
- **Determining the checkout info**: 특정 브랜치나 커밋을 선택하는 등의 체크아웃할 구체적인 정보를 결정
- **Checking out the ref**: 지정된 브랜치나 커밋을 워크스페이스로 가져온다.

<br/>


**빌드 자동화까지 완료**
<p align="center">
<img width="890" alt="Monosnap ci: Update yml · barreleye-labs:barreleyescan@06e83e9 2024-07-21 19-15-10" src="https://github.com/user-attachments/assets/23805858-8b69-448f-9e26-08fd7c0fc66c">
  <br/>
<img width="881" alt="Monosnap ci: Update yml · barreleye-labs:barreleyescan@06e83e9 2024-07-21 19-06-52" src="https://github.com/user-attachments/assets/628de367-13aa-457e-b49a-6b86a711e25c">
</p>

<br/>



**`needs` 키워드 사용하기**

> GitHub Actions의 needs 키워드는 워크플로우에서 특정 작업(job)이 다른 작업(job)의 완료를 기다리도록 설정할 때 사용됩니다. 즉, needs를 사용하여 작업 간의 의존성을 정의할 수 있습니다. 이 키워드를 통해 특정 작업이 완료된 후에만 다른 작업이 실행되도록 할 수 있습니다.
> 
> 
> ```yaml
> build:
>   runs-on: ubuntu-latest
>   needs: cypress-run # cypress-run 작업이 성공적으로 완료된 후 이 작업이 실행된다
> ```
>


<br/>


## 배포 YAML 작성하기

마지막 배포에서는 `appleboy/ssh-action@v1.0.3` 액션을 사용하여 SSH를 통해 원격 서버에 연결 후 스크립트를 작성해 배포할 것이다.

### 배포 파이프라인 구성

1. **SSH 연결**: 원격 서버에 SSH로 연결
2. **컨테이너 중지**: 실행 중인 Docker 컨테이너를 중지
3. **컨테이너 제거**: 지정된 이미지 기반의 모든 컨테이너를 제거
4. **이미지 제거**: 지정된 Docker 이미지를 제거
5. **최신 이미지 풀**: Docker Hub에서 최신 이미지를 풀 받기
6. **도커 컨테이너 실행**

### 스크립트

```yaml
name: E2E Test and Build and Deploy

on:
  push:
    branches:
      - develop
  pull_request:
    branches:
      - develop

jobs:
  cypress-run:
    runs-on: ubuntu-latest
    
  build:
    runs-on: ubuntu-latest

# ===생략===
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deployment
        uses: appleboy/ssh-action@v1.0.3 # ssh를 통해 원격 서버 연결
        with:
          host: ${{ secrets.EC2_IP_ADDRESS }} # SSH 연결할 원격 서버의 IP 주소
          username: ubuntu
          key: ${{ secrets.EC2_SSH_PRIVATE_KEY }} # SSH 연결에 사용할 개인 키
          script: | whoami # 연결이 잘 되었는지 확인 (원격 서버에서 현재 사용자 이름을 출력하는 명령어)
  
```

---

**`whoami` 통해 접속 먼저 확인하기**

`whoami` 를 통해 로그가 확인되었다면, 그 이후 스크립트를 작성하다가 에러를 직면해도 최소한 접속 문제는 아니라는 것을 파악하며 진행할 수 있다.

![7](https://github.com/user-attachments/assets/99eebfef-ee98-4279-af1a-0eb424671a12)

---

### 도커 배포시 이미지 및 컨테이너 삭제에 대한 이야기

**💡컨테이너 및 이미지를 삭제 해주지 않는다면?**

작은 용량의 서버를 사용하고 있는 사용자라면 마주할 수 있는 문제인데, (사실 나의 상황이다) 이미지를 삭제하지 않으면 용량 부족으로 `no space left on device` 에러 로그를 맞이하게 된다. 컨테이너는 삭제하지 않으면 충돌 문제로 교체되지 않는다.


**맞이할 수 있는 에러**

- 컨테이너를 삭제하지 않았을 때 마주할 수 있는 에러

```bash
Error response from daemon: conflict: # 이미 존재하는 컨테이너간의 충돌
```

- 이미지를 삭제하지 않았을 때 마주할 수 있는 에러: 용량 문제

![8](https://github.com/user-attachments/assets/7d29c03a-9c56-4885-8105-0201baa95add)

### 배포를 위한 컨테이너 / 이미지 삭제

**1. 컨테이너 삭제를 위해 사용할 명령어**

```bash
docker ps -f "filter"
```

- **docker ps**: 현재 실행 중인 컨테이너 목록 조회
- **-f "filter"**: 특정 조건으로 컨테이너 목록을 필터링

**2. 특정 컨테이너 ID 찾기**

찾고자 하는 리포지토리의 컨테이너 ID를 찾아주는 명령어다. 아래의 명령어를 변수에 담아 사용하도록한다.

```bash
docker ps -f "ancestor=${{ secrets.USERNAME }}/${{ secrets.DOCKER_REPO }}" --format "{{.ID}}"
```

**3. 컨테이너 ID 변수화 시킨 후 삭제 해주기**

컨테이너 ID의 로깅과 함께 해당하는 컨테이너를 삭제해준다. 이런 방향으로 삭제해야 하는 도커 이미지도 변수로 담아 삭제 처리 해준다. 

```bash
CONTAINER_IDS=$(sudo docker ps -f "ancestor=${{ secrets.USERNAME }}/${{ secrets.DOCKER_REPO }}" --format "{{.ID}}")
          echo "Containers ID: $CONTAINER_IDS"
          if [ -n "$CONTAINER_IDS" ]; then
          sudo docker rm -f $CONTAINER_IDS
          fi
```

### 완성된 배포 스크립트

```bash
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deployment
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_IP_ADDRESS }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_PRIVATE_KEY }}
          script: |
            sudo docker stop ${{ secrets.DOCKER_REPO }} || true
          
            CONTAINER_IDS=$(sudo docker ps -f "ancestor=${{ secrets.USERNAME }}/${{ secrets.DOCKER_REPO }}" --format "{{.ID}}")
            echo "Containers ID: $CONTAINER_IDS"
            if [ -n "$CONTAINER_IDS" ]; then
            sudo docker rm -f $CONTAINER_IDS
            fi
  
            IMAGE_ID=$(sudo docker images --format "{{.ID}}" ${{ secrets.USERNAME }}/${{ secrets.DOCKER_REPO }})
            echo "Image ID: $IMAGE_ID"
            if [ -n "$IMAGE_ID" ]; then
            sudo docker rmi -f $IMAGE_ID
            echo "Deleted image with ID: $IMAGE_ID"
            else
            echo "No image found for ${{ secrets.USERNAME }}/${{ secrets.DOCKER_REPO }}"
            fi
            
            sudo docker pull ${{ secrets.USERNAME }}/${{ secrets.DOCKER_REPO }}
  
            echo "Executing custom script: barreleyescan.sh"
            sudo ./barreleyescan.sh
```

### 모니터링을 위한 로깅추가

```yaml
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deployment
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_IP_ADDRESS }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_PRIVATE_KEY }}
          script: |
            # Stop running container
            echo "Stopping running container..."
            sudo docker stop ${{ secrets.DOCKER_REPO }} || true
            
             # Remove containers based on image
            echo "Removing containers based on image..."
            CONTAINER_IDS=$(sudo docker ps -f "ancestor=${{ secrets.USERNAME }}/${{ secrets.DOCKER_REPO }}" --format "{{.ID}}")
            echo "Containers ID: $CONTAINER_IDS"
            if [ -n "$CONTAINER_IDS" ]; then
            sudo docker rm -f $CONTAINER_IDS
            fi
  
            # Remove the Docker image
            echo "Removing Docker image..."
            IMAGE_ID=$(sudo docker images --format "{{.ID}}" ${{ secrets.USERNAME }}/${{ secrets.DOCKER_REPO }})
            echo "Image ID: $IMAGE_ID"
            if [ -n "$IMAGE_ID" ]; then
            sudo docker rmi -f $IMAGE_ID
            echo "Deleted image with ID: $IMAGE_ID"
            else
            echo "No image found for ${{ secrets.USERNAME }}/${{ secrets.DOCKER_REPO }}"
            fi
            
            # Pull the latest Docker image
            echo "Pulling the latest Docker image..."
            sudo docker pull ${{ secrets.USERNAME }}/${{ secrets.DOCKER_REPO }}
  
            # Execute the custom script
            echo "Executing custom script: barreleyescan.sh"
            sudo ./barreleyescan.sh
```

## 배럴아이스캔 CI/CD 자동화 결과물

<p align="center">
<img alt="Monosnap ci_ Update yaml · barreleye-labs_barreleyescanc5ced78 2024-07-21 20-12-16" src="https://github.com/user-attachments/assets/69e2bf1b-83d8-44a4-b465-06af9d8defd0">
</p>



## 끝으로

GitHub Actions를 통해 CI/CD 자동화를 구축하면서 몇 가지 문제를 맞닥뜨리게 되었고, 이를 통해 프로젝트의 수정이 필요함을 깨달았다.

**배포 시간 줄이기**

CI/CD 파이프라인을 통해 자동 배포를 설정한 후, 배포 시간이 예상보다 길어지는 문제를 발견했다. 이는 개발 속도를 늦추는 요인으로 작용했다. 이에 따라 배포 시간을 줄이기 위한 최적화 작업이 필요하게 되었다.

**용량 줄이기**

작은 용량의 서버를 사용하고 있는 경우, 특히 용량 부족 문제는 매우 빈번하게 발생한다. 나의 경우, 8GB의 용량을 사용하고 있었으나, 간략한 E2E 테스트 코드를 추가한 후 배포를 시도하니 용량 부족 문제에 직면하게 되었다. 결국 중간에 용량을 1GB 늘려야만 했다.

이를 통해 다음과 같은 교훈을 얻었다:

- 불필요한 Docker 이미지와 컨테이너를 주기적으로 삭제하여 용량을 관리해야 한다.
- 프론트코드 최적화가 필요한 시점이다.

이번 CI/CD 자동화 구축을 통해 배운 점은 많다. 프로젝트의 효율성과 안정성을 높이기 위해, 배포 시간과 서버 용량 관리를 포함한 여러 최적화 작업을 지속적으로 진행할 계획이다.
