---
title: 'Vite 빌드 후 결과물 자동 압축하기: 날짜·시간 기반 폴더명으로 관리하기'
tags: ''
---

서비스를 운영하다 보면, 빌드 후 생성된 결과물을 전달하거나 관리할 때 헷갈리거나 반복적인 작업이 생기곤 합니다. 이번 글에서는 Vite 빌드 과정에서 날짜·시간 기반의 폴더명을 자동으로 생성하고, zip 압축까지 자동화하는 과정을 정리해보겠습니다.

<br/>

### **왜 이런 작업이 필요했을까?**

기존에는 빌드 후 생성된 **결과물 폴더(dist)명**을 개발자가 직접 바꾸고, zip으로 압축해서 전달해왔습니다.

예를 들어, `wallet-1.0.18-250410-11.zip` 같은 형식으로 만들어야 했는데, 이걸 매번 수동으로 하다 보니 번거롭고 실수할 여지도 있었습니다. 이 과정을 자동화하면

- 날짜와 시간을 일일이 입력할 필요 없이 자동 처리 (반복 작업 해방)
- 파일명 규칙이 일관돼 QA 팀이나 다른 팀에서도 혼란 없이 관리 가능
  
<br/>

### **해결 방안**

- 빌드 결과물 폴더명을 자동으로 생성
- 날짜·시간 기반 폴더명 포맷 → `wallet-1.0.18-250410-11`
    - `1.0.18`(버전)- `250410` (YYMMDD) - `11` (HH)
- 빌드 완료 후, 해당 폴더를 zip으로 압축

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

<br/>

### **사용 라이브러리**

- [adm-zip](https://www.npmjs.com/package/adm-zip): zip 압축 생성을 도와주는 Node.js 라이브러리

<br/>

### **효과**

- 결과물 관리 표준화
- 릴리즈 전달 속도 개선
- 운영 실수 최소화

<br/>

### **정리**

빌드 후 자동으로 폴더명을 구분하고, zip으로 묶어내는 작은 개선이지만, 팀의 수작업을 줄여주었습니다. 앞으로도 빌드와 관련된 불필요한 작업은 꾸준히 자동화할 방법을 찾아봐야겠다는 생각이 듭니다.
