---
title: '디자인 시스템 왜 필요할까?'
tags: 'frontend' 
---

이전 회사에서 `ant-design` 을 기반으로한 디자인 시스템을 만들었다. 프로젝트 개발과 병행해 공통 컴포넌트로 두면 좋겠다 하는 것은 추가로 같이 개발했다.

이번에는 `ant-design` 과 같은 UI 라이브러리를 사용하지 않고, 기존 프로젝트에서 재사용할 수 있는 부분들을 컴포넌트로 분리하는 작업에 집중했다.

### 왜 이런 작업이 필요했을까?

자사 소개 페이지의 디자인이 한 번 바뀌면 다른 소개페이지의 디자인도 대공사로 변경해주는 일이 있었다.
또 우리 회사에는 익스텐션 월렛과 앱 월렛이 있는데 디자인이 톤은 일관되어있는데, 컴포넌트 디자인이 약간씩 달랐다.

그래서 일관성을 위해 내가 맡고 있는 익스텐션 월렛의 디자인 시스템을 구축한 뒤, 앱 월렛에도 적용한다면 어떨까 생각했다.
더 나아가, 회사에 한 번 디자인 시스템이 구축된다면,
디자인 시스템의 필요성을 가지고 소개페이지를 개발할 때나 다른 프로젝트를 만들어갈 때 디자인 시스템을 고려한 프로덕트가 나오지 않을까 하는 생각이 들었다.

### 디자인 시스템 도입 방향

- 파운데이션 및 토큰(color, typography, spacing, radius 등) 정의
- 토큰 기반 공통 컴포넌트 개발
- 스토리북으로 컴포넌트 문서화
- NPM 배포로 다른 프로젝트에서도 바로 사용 가능하게

### 구현 방법

1️⃣ **디자인 시스템 전용 리포지토리 생성**

2️⃣ **폴더 구조**

```
├── package
│   ├── ui
│   │   ├── Button
│   │   │   ├── Button.stories.tsx // 스토리북
│   │   │   ├── Button.style.ts // 스타일
│   │   │   ├── Button.tsx // 컴포넌트
│   │   │   ├── Button.type.ts // 타입
│   │   │   └── index.ts
│   │   ├── Checkbox
│   │   │   ├── Checkbox.stories.tsx
│   │   │   ├── Checkbox.style.ts
│   │   │   ├── Checkbox.tsx
│   │   │   ├── Checkbox.type.ts
│   │   │   └── index.ts
│   └── app
│   │   ├── NFTItem
│   │   │   ├── NFTItem.stories.tsx
│   │   │   ├── NFTItem.style.ts
│   │   │   ├── NFTItem.tsx
│   │   │   ├── NFTItem.type.ts
│   │   │   └── index.ts
│   └── tokens
│       ├── colors.ts
│       ├── iconography.tsx
│       ├── index.ts
│       └── typography.ts

```

**2️⃣ 디자인 토큰 정의**

- colors 정의

```
export const colors = {
  base: { red: '#FF668A', white: '#FFFFFF', black: '#000000' },
  purple: { 100: '#6710FF', 200: '#8D7AFE' },
  gray: { 100: '#303030', 200: '#9899B2' },
};

```

- typography 정의

```
export const typographyToken = {
  fontFamily: "'Pretendard Variable', Pretendard, ...",
  fontSize: {
    title1: '30px',
    title2: '24px',
    // ...
  },
  fontWeight: {
    regular: 400,
    medium: 500,
    // ...
  },
  lineHeight: {
    title1: '30px',
    title2: '32px',
    // ...
  },
  letterSpacing: {
    tight: '-0.02em',
    normal: '0',
  },
};

export const typography = {
  title1: {
    fontFamily: typographyToken.fontFamily,
    fontWeight: typographyToken.fontWeight.bold,
    fontSize: typographyToken.fontSize.title1,
    // ...
  },
  body1: {
    fontFamily: typographyToken.fontFamily,
    fontWeight: typographyToken.fontWeight.regular,
    fontSize: typographyToken.fontSize.body1,
    // ...
  },
  // ...
};

type Typography = keyof typeof typography;

export type { Typography };

```

3️⃣ **컴포넌트 리팩토링 및 개발**

월렛 익스텐션에서 공통으로 쓸 수 있는 컴포넌트를 식별해서 사용할 수 있는 컴포넌트들은 일부 리팩토링을 한 후 새로 만든 리포지토리에 추가해주었다.
아래와 같은 내용들을 신경썼던 것 같다.

- 코드 컨벤션
- 순환 참조 제거
- 프로젝트에 종속되어 있는 코드의 경우 분리 (훅/유틸과 같은 함수는 분리하여 디자인 시스템 컴포넌트 내부에서 사용할 수 있도록)

그 외 필요한 컴포넌트들은 직접 개발하여 리포지토리에 추가했다.

4️⃣ 디자인 토큰 기반 컴포넌트 개발

- `colors token` 사용 예

```tsx
const ButtonVariants = {
  primary: {
    text: colors.base.white,
    bg: colors.purple[100],
    hoverBg: colors.purple[200],
  },
  secondary: {
    text: colors.purple[100],
    bg: colors.base.white,
    hoverBg: colors.purple[200],
  },
  outlined: {
    text: colors.purple[100],
    bg: colors.base.white,
    hoverBg: colors.purple[200],
  },
  gray: {
    text: colors.gray[100],
    bg: colors.gray[400],
    hoverBg: colors.purple[200],
  },
};

```

- 컴포넌트 사용 예

```tsx
<Button variant="primary">Click!</Button>

```

- `typography token` 사용 예

```tsx
const ButtonSizes = {
  lg: { height: '50px', ...typography.button1 },
  md: { height: '40px', ...typography.button2 },
};

```

- 컴포넌트 사용 예

```tsx
<Button size={'lg'} variant={'primary'}>Click!</Button>

```

**5️⃣ Storybook 문서 구축**

- 스토리북 구조
  - 토큰 및 파운데이션: Colors / Typography / Spacing / Radius
  - 공통 컴포넌트: UI Component
  - 도메인 컴포넌트: App Component
  - 페이지: 컴포넌트들의 (UI Component + App Component) 조합

    ![image.png](/assets/posts/2025-04-20-design-system/image.png)

- 크로마틱으로 배포하여 스토리북 공유

```json
"scripts": {
  "chromatic": "chromatic --project-token=$CHROMATIC_PROJECT_TOKEN"
}

```

**6️⃣ NPM 배포**

```json
{
  "name": "sit-ui", // 배포할 패키지 이름
  "version": "0.1.13", // 현재 패키지 버전
  "main": "dist/index.cjs.js", // CJS 환경에서 사용할 진입점
  "module": "dist/index.esm.js", // ESM 환경에서 사용할 진입점
  "types": "dist/package/index.d.ts", // 타입 정의 파일 위치
  "files": [
    "dist",
    "!**/*.stories.*"
  ], // 배포할 때 포함할 폴더 or 파일, 스토리북은 제거할 것임

```

```bash
npm login
npm publish

```

- 사용 예

```tsx
import { Button, Dropdown } from 'sit-ui';

<Dropdown placement={'topRight'} menu={{ items, onClick: () => console.log('clicked!') }}>
    <Button variant={'outlined'} size={'sm'}>Click!</Button>
</Dropdown>

```

**7️⃣앞으로 컴포넌트를 추가해 나갈 때 주의할 점**

- index import 금지 (순환참조 방지)

아래처럼 컴포넌트를 export 를 해놓은 경우 외부에서만 사용하도록 해야한다.

```tsx
export { Button } from './Button';
export { Text } from './Text';

```

내부에서 임포트할 때에 주의할 점이다.

❌ 잘못된 예

```tsx
import { Text } from '../';

```

✅ 올바른 예

```tsx
import { Text } from '../Text';

```

버튼 스토리북이 본인의 버튼 컴포넌트를 참조하려면 아래처럼,

```tsx
import { Button } from '../'; // ❌
import { Button } from './';  // ✅

```

순환 참조는 로컬에서는 문제 없지만 빌드에서 다음처럼 순회하며 터진다.

> Button.stories.tsx → index.ts → Button.tsx → index.ts → Text.tsx → index.ts → …



### 마무리

이번 디자인 시스템 구축은 작은 프로젝트를 넘어 회사 전반의 일관성을 높이고 중복 작업을 줄이는 데 큰 의미가 있다고 생각한다.
아직 디자이너와 나 둘이서 진행 중이라 갈 길이 멀지만, 자사의 아이덴티티를 한층 또렷하게 만들어줄 것이라 기대한다.
