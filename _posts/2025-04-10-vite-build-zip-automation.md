---
title: 'Vite 빌드 후 결과물 자동 압축하기: 날짜·시간 기반 폴더명으로 관리하기'
tags: '프론트엔드'
---


서비스를 운영하다 보면, 빌드 후 생성된 결과물을 전달하거나 관리할 때 헷갈리거나 반복적인 작업이 생기곤 한다. 이번 글에서는 Vite 빌드 과정에서 날짜·시간 기반의 폴더명을 자동으로 생성하고, zip 압축까지 자동화하는 과정을 정리하려고 한다.

<br/>

### **왜 이런 작업이 필요했을까?**

익스텐션은 보통 웹사이트 개발과는 다른 점이 있다. QA를 할 경우인데, 개발을 마친 후 작업물을 QA 해볼 수 있게 개발 서버를 도입해서 uri 를 전달하는 것이 아닌,
개발자가 빌드파일을 압축해서 QA 진행자에게 전달하고, 받은 파일을 구글 확장프로그램 사이트에 파일을 올려서 테스트 하는 것이다.

이 전달하는 과정에 빌드된 압축파일명을 `wallet-dist` 와 같이 정적이게 해놓았더니, 수 많은 압축폴더 중에 최신 것이 언제인지, 아니면 내가 찾고자 하는 파일이 어디있는지 찾는데 시간을 썼다.
그리고 QA 진행자와 소통을 하면서도 압축파일 명이 다 똑같다보니, 슬랙에 전달했던 이력을 찾아 쓰레드를 사용해 `이 압축파일에서 이슈가 있었던 건가요?`라고 태깅을 걸어야 했다..

그래서 압축파일을 전달하기 까지 아래와 같은 과정을 거쳤다.
- 빌드한 후 파일 명 바꾸기
- 그 파일 압축하기

그리고 이번에 아래처럼 자동화 할 것이다.
- 날짜와 시간을 일일이 입력할 필요 없이 자동화 (반복 작업 해방)
- 파일 압축 자동화

<br/>

### **방향**

- 빌드된 폴더명을 형식을 가진 폴더명으로 자동 생성되게 할 것이다.
- 날짜·시간 기반 폴더명 포맷 → `wallet-1.0.18-250410-11`
  - `1.0.18`(버전)- `250410` (YYMMDD) - `11` (HH)
- 빌드 완료 후, 해당 폴더를 zip으로 자동 압축

<br/>


### **사용 라이브러리**

- [adm-zip](https://www.npmjs.com/package/adm-zip): zip 압축 생성을 도와주는 Node.js 라이브러리

<br/>


### **구현 방법**

1️⃣ 날짜·시간 기반 폴더명 만들기

```tsx
export function useBuildFileName(version: string, mode: string): string {
  const now = new Date();
  const pad = (n: number) => n.toString().padStart(2, '0');
  const date = `${pad(now.getFullYear() % 100)}${pad(now.getMonth() + 1)}${pad(now.getDate())}`;
  const hour = `${pad(now.getHours())}`;
  const baseName = `dist/wallet-${version}-${date}-${hour}`;

  return mode === 'production' ? baseName : `${baseName}-dev`;
}
```

2️⃣ Vite Rollup Plugin에서 압축 처리

```tsx
const buildFileName = useBuildFileName(version, mode);

rollupOptions: {
        ...,
        plugins: [
          {
            name: 'zip-after-build',
            writeBundle() {
              const zip = new AdmZip();

              zip.addLocalFolder(buildFileName);
              zip.writeZip(`${buildFileName}.zip`);
            },
          },
        ],
      },
```

3️⃣ 전체 빌드 플로우

- `npm run build:prod` 실행
- `/dist/wallet-1.0.18-250410-11` 폴더 생성
- `/dist/wallet-1.0.18-250410-11.zip` 압축 파일 생성
  ![image.png](/assets/2025-04-10-vite-build-zip-automation/image.png)

<br/>

### **결과**

- 결과물 관리 표준화
- 파일명 규칙이 일관돼 QA 팀이나 다른 팀에서도 혼란 없이 관리 가능
- 릴리즈 전달 속도 개선


<br/>
