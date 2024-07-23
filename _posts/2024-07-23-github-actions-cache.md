---
title: 'Github Actions 캐시 전략을 통한 빌드 시간 단축'
tags: Frontend
---

# Github Actions 캐시 전략을 통한 빌드 시간 단축

## **들어가기 앞서**

Github Actions을 통해 배럴아이스캔의 CI/CD 파이프라인을 구축했었다([**GitHub Actions 배포 자동화**](https://www.notion.so/Github-Actions-7e8124ecbb9a4e6fbb9790babb944995?pvs=21)). 그 이후, 눈에 띄는(?) 배포 시간 덕분에 좋은 경험을 공유하려고 한다. 배포 시간을 길게 하는 요소로는 Node.js와 Docker를 사용하는 환경에서 매번 모든 의존성을 새로 설치하고 빌드하는 과정이 있었는데, 이러한 문제를 해결하기 위해 GitHub Actions에서 제공하는 캐시 기능을 활용하여 빌드 시간을 단축하는 방법을 시도해 보았다.

## **구성**

배럴아이스캔 프로젝트에서는 다음과 같은 작업들을 거쳐야 최종 화면을 볼 수 있었다.

- E2E 시나리오 **테스트**
- 도커를 통한 **빌드**
- 도커를 통한 **배포**

이 중, 테스트와 빌드 시간을 단축하기 위해 캐시 전략을 적용해보았다.

---

### 입금(캐시)전 마지막 이미지
![0](https://github.com/user-attachments/assets/cec63479-8eeb-430a-87f4-af3cc400e176)

---

### **GitHub Actions 캐시란**

GitHub Actions는 CI/CD(Continuous Integration/Continuous Deployment) 기능을 제공하는 도구이다. 이를 통해 코드 변경 사항을 자동으로 테스트하고 배포할 수 있다. 캐시 기능을 활용하면 이전 빌드의 결과물을 저장해서 이후 빌드에서 재사용할 수 있다. 이렇게 하면 빌드 시간을 크게 줄일 수 있다. GitHub Actions 워크플로우는 YAML 파일 형식으로 작성되며, 여러 작업과 단계로 구성된다.

### **캐시 전략 적용**

기존 설정에서는 매번 모든 의존성을 새로 설치하고 빌드했기 때문에 시간이 많이 소요됐다. 이를 해결하기 위해 캐시 전략을 적용한 결과, 빌드 시간을 크게 단축할 수 있었다. 여기서는 Node.js 의존성과 Docker 빌드 과정에서 캐시를 활용하는 방법을 자세히 설명한다.

## **Node.js 의존성 캐시**

Node.js 프로젝트에서는 패키지 매니저(npm 또는 yarn)를 통해 의존성을 설치한다. 그러나 매번 의존성을 새로 설치하는 것은 상당한 시간이 소요된다. 이를 해결하기 위해 GitHub Actions의 캐시 기능을 사용하여 node_modules 디렉토리를 캐시할 수 있다. 이렇게 하면 동일한 의존성을 다시 설치할 필요가 없기 때문에 빌드 시간을 단축할 수 있다.

**설정 예시**

아래는 Node.js 의존성 캐시를 설정한 수정된 YAML 파일이다.

```yaml
cypress-run:
  runs-on: ubuntu-latest
  steps:
    - name: Set up Node.js
      uses: actions/setup-node@v2
      with:
        node-version: '20.15.0'

    - uses: actions/checkout@v3

    - name: Cache node modules
      uses: actions/cache@v3
      id: npm-cache
      with:
        path: ~/.npm
        key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-node-

    - if: steps.npm-cache.outputs.cache-hit == 'true'
      run: echo 'npm cache hit!' # 사용할 캐시가 있는 경우
    - if: steps.npm-cache.outputs.cache-hit != 'true'
      run: echo 'npm cache missed!' # 사용할 캐시가 없는 경우

    - run: npm ci

    - name: Start the application
      run: npm run dev &
      env:
        CI: true

    - name: Run Cypress tests
      run: npm run test
```

이 설정에서는 다음과 같은 작업을 수행한다

1. actions/setup-node 액션을 사용하여 지정된 버전의 Node.js를 설치한다.
2. actions/checkout 액션을 사용하여 리포지토리를 체크아웃한다.
3. actions/cache 액션을 사용하여 ~/.npm 디렉토리를 캐시한다. package-lock.json 파일의 해시 값을 키로 사용하여 의존성이 변경되지 않은 경우 캐시를 재사용한다.
4. 캐시 히트 여부를 확인하여 로그에 출력한다. 캐시가 히트된 경우 npm ci 명령어를 실행하지 않고, 미스된 경우 npm ci를 실행하여 의존성을 설치한다.
5. npm run dev 명령어를 실행하여 애플리케이션을 시작하고, CI=true 환경 변수를 설정한다.
6. npm run test 명령어를 실행하여 Cypress 테스트를 수행한다.

**캐시 로그 예시** 

아래는 캐시 히트 여부를 확인하는 로그이다. 캐시가 히트되면 `npm cache hit!` 메시지를 출력하고, 캐시가 미스되면 `npm cache missed!` 메시지를 출력한다.

- 저장된 캐시가 있을 경우
  - <img  alt="1" src="https://github.com/user-attachments/assets/e7cc5583-bf4f-41ea-8341-1acd48d00a11">

- 저장된 캐시가 없을 경우
  -  캐시가 없을 경우 `Post Run actions/cache@v3` 구간에서 새로 저장한다. 이 저장물은 Github Actions 의 Cache 메뉴에서 확인할 수 있다.
  - <img  alt="2" src="https://github.com/user-attachments/assets/44e44cc6-d0ea-478a-9d59-6a1bbe3563ab">
  
---

**GitHub Actions의 Caches 가시적으로 보기**

GitHub Actions의 Caches 메뉴를 통해 현재 사용 중인 캐시들을 확인할 수 있다. 
<img  alt="3" src="https://github.com/user-attachments/assets/fdc16dfa-20d4-4c1b-9e43-8df3dd6e0513">

아래 이미지는 실제로 캐시된 Docker 빌드 캐시와 Node.js 의존성 캐시의 예시이다. 여기서는 각 캐시의 크기와 캐시된 시간을 확인할 수 있다. 위에서 봤었던 캐시 (`Linux-node-510a8e....생략...` )는 73MB 크기로 19시간 전에 캐시되었음을 확인할 수 있다.<br/>
<img   alt="4" src="https://github.com/user-attachments/assets/d3ef84c8-87bb-4da3-9d38-718dd9a8bba6">

---

**초기 캐시 저장은 시간이 더 걸려요!**

처음 캐시를 저장하기 전에는 시간이 오히려 더 오래 걸린다. 아까 봤던 로그를 다시 확인해보자. 캐시가 되지 않아 `cache missed!` 로그가 보인 뒤 `Post Run actions/cache@v3` 구간에서 기존에는 1분 초반대에서 마무리 되던 테스트가 2분이 넘게 걸린다. 이후에 저장된 캐시가 있다면 다음 빌드에서는 대폭 줄은 빌드 시간을 확인할 수 있다.
<img   alt="5" src="https://github.com/user-attachments/assets/98abda35-ca1a-4951-a58c-d65d3ffd4fa1">

## Node.js 의존성 캐시 적용 결과

- 캐시 추가하기 전 빌드 시간: 1분 19초
  - ![6](https://github.com/user-attachments/assets/e2dff140-744b-4768-9555-64993e21728c)
    
- 캐시 추가 후 늘어난 빌드 시간: 3분 4초
  - ![7](https://github.com/user-attachments/assets/1d90b391-fb27-4d31-895c-5705220acdc5)

    
- 캐시 저장 후 줄어든 빌드 시간: 47초
  - ![8](https://github.com/user-attachments/assets/8b0565f6-aa0f-45d4-936b-19fe08068ddd)
  
    

이처럼 처음 캐시를 저장하기 전까지는 시간이 오히려 더 오래 걸린다. 하지만 캐시가 저장된 이후 빌드 시간이 1분 19초에서 47초로 줄어들었다. 이는 32초가 줄어든 것으로, 약 40.5%의 빌드 시간이 단축 되었다.

## **Docker 빌드 캐시**

Docker 이미지를 빌드하는 과정에서도 캐시를 활용하여 빌드 시간을 단축할 수 있다. Docker는 각 레이어를 캐시할 수 있으며, 이전 빌드의 레이어를 재사용하여 빌드 속도를 높일 수 있다. 이를 위해 `docker/build-push-action` 액션을 사용하여 GitHub Actions에서 Docker 빌드 캐시를 설정할 수 있다.

**설정 예시**

아래는 Docker 빌드에 캐시를 적용한 수정된 YAML 파일이다:

```yaml
- name: Build and push
  uses: docker/build-push-action@v2
  with:
    context: .
    push: true
    tags: ${{ secrets.USERNAME }}/${{ secrets.DOCKER_REPO }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

이 설정에서는 다음과 같은 작업을 수행한다

1. docker/build-push-action 액션을 사용하여 Docker 이미지를 빌드하고 푸시한다.
2. context 매개변수를 통해 Docker 컨텍스트를 설정한다. 일반적으로 현재 디렉토리를 사용한다.
3. push 매개변수를 true로 설정하여 빌드가 완료된 이미지를 Docker Hub에 푸시한다.
4. tags 매개변수를 사용하여 이미지를 태그한다. 여기서는 GitHub Secrets를 사용하여 Docker Hub 사용자 이름과 리포지토리 이름을 지정한다.
5. cache-from 매개변수를 type=gha로 설정하여 GitHub Actions의 캐시를 사용하여 이전 빌드의 레이어를 가져온다.
6. cache-to 매개변수를 type=gha,mode=max로 설정하여 빌드된 레이어를 최대한 GitHub Actions 캐시에 저장한다.

**캐시 로그 예시**

아래는 Docker 빌드 과정에서 캐시를 사용한 로그이다. `importing cache manifest from gha` 메시지를 통해 캐시를 불러오고, 각 레이어가 캐시되었음을 확인할 수 있다.
<img  alt="9" src="https://github.com/user-attachments/assets/06bf9322-5fe9-401e-97a9-d27718c193e4">



## Docker 캐시 적용 결과

캐시 전략을 적용한 후, 빌드 시간이 크게 단축됐다. 아래는 새로운 캐시 설정을 적용한 후의 빌드 결과이다.

- 캐시 추가하기 전 빌드 시간: 1분 53초
  - <img alt="10" width="400" src="https://github.com/user-attachments/assets/ccf8f7d3-2450-41fb-92bd-073f313c8ec2">

  
    
- 캐시 추가한 빌드 시간: 7초
  - ![11](https://github.com/user-attachments/assets/8db6e69c-1ee6-4e2f-aa03-7d0f054a27bb)

Docker 캐시를 적용한 후 빌드 시간이 1분 53초(113초)에서 7초로 줄어들었다. 이는 106초가 단축된 것으로, 약 93.8%의 빌드 시간 단축을 의미한다.

## 완성된 배럴아이스캔 캐시 전략

 새로운 캐시 전략을 적용한 결과 약 44.9%의 빌드 시간 단축할 수 있었다. 

- 캐시 전략 전: 5분 14초 (314초)
  - ![12](https://github.com/user-attachments/assets/c1004322-b8ec-4810-b806-66296dff388c)



- 캐시 전략 적용 후: 2분 53초 (173초)
  - <img alt="13" src="https://github.com/user-attachments/assets/9c9d3776-af90-4259-8c4e-9a4cce84b81c">



## **끝으로**

GitHub Actions를 통해 CI/CD 자동화를 구축하면서 캐시를 활용한 최적화가 얼마나 중요한지 깨닫게 됐다. 캐시를 적절히 활용함으로써 빌드 시간을 단축하고, 개발 속도를 높일 수 있었다. 

**캐시 활용의 중요성**

의존성과 빌드 아티팩트를 캐시함으로써 반복적인 작업 시간을 크게 줄일 수 있다. 개발 속도를 높이고, 더 짧은 시간 내에 변경 사항을 배포할 수 있게 된 것이 이렇게 편할 줄 몰랐다. (빌드까지의 명령어가 너무 많아서 피아노 치는 것 같았다)

이번 경험을 바탕으로, 앞으로도 캐시를 적극적으로 활용하여 CI/CD 파이프라인의 효율성을 극대화하고, 안정적이고 빠른 배포를 지속적으로 유지할 계획이다.
