---
title: "이미지를 많이 사용하는 단일 페이지 웹 성능 최적화"
tags:
  - 프론트엔드
  - 성능
---


이미지가 많이 들어가는 소개 페이지의 LightHouse의 성능 개선기를 적어본다. 각 섹션마다 고해상도 배경 이미지가 사용되었는데, 이 이미지들은 용량이 커서 첫 로딩시 LCP에 영향이 가게 되었다.

### 방법

1. 이미지 최적화
2. LCP 개선
3. CLS 개선
4. 접근성 개선

### **1. 이미지 최적화 (webp 변환)**

png, jpg 이미지를 webp로 변환해 이미지 용량을 줄이는 것은 웹 성능 최적화에서 매우 효과적인 방법 중 하나다. webp는 구글에서 만든 이미지 포맷으로, **더 작은 용량으로 동일한 품질**을 제공하기 때문에, 네트워크 트래픽 절감 및 페이지 로딩 속도 개선에 유리하다

여기에 AVIF를 추가할 수도 있다. webp 보다 더 압축률이 좋아 리소스 크기를 감소시킬 수 있다. 하지만 지원이 되지 않는 브라우저도 있기 때문에 프로젝트의 상황에 맞게 사용해야 한다.

```html
<picture>
  <source srcset="/image.avif" type="image/avif" />
  <source srcset="/image.webp" type="image/webp" />
  <img src="/image.png" alt="..." />
</picture>
```

브라우저가 webp를 지원할 경우 <source>의 이미지를, 그렇지 않을 경우 <img> 태그에 정의된 이미지가 사용된다. 이렇게 `<picture>` 태그를 활용하면 **브라우저 호환성을 확보**하면서도 webp(혹은 avif) 최적화를 적용할 수 있다.

### 2. LCP 개선

- Lazy Loading 적용
- IntersectionObserver로 세밀 제어
- LCP 리소스 프리로드
- 뷰포트별 이미지 제공 (srcSet)

1. `img` 태그로 Lazy Loading

   img 태그에 `loading="lazy"` 속성을 설정하여 브라우저에게 이미지 리소스의 호출 시점을 맡길 수 있다.

   ```html
   <img src="/image.png" loading="lazy" alt="..." />
   ```

   브라우저가 자체적으로 판단해 **뷰포트 근처에 도달하면 미리 로드**하는 방식이므로, 사용자가 이미지를 실제로 보게 되는 시점과는 다를 수 있다.

   추가로 Image 컴포넌트를 만들어, LCP 영역의 이미지는 fetchpriority="high"로 빠르게 로드하고, 그 외 이미지는 loading="lazy"와 decoding="async"로 설정했다.

   ```tsx
   <Image isLcp src="/image.png" alt="..." /> // fetch priority 적용
   ```

   위 속성을 지정해준 후 첫 로딩시 이미지 리소스들이 줄긴했지만, 예상치 못하게 계속 로드 되는 이미지들도 있었다. 그 이유는 `img` 태그가 아닌 `배경 이미지(background-image)`였기 때문이다. `css` 로 불려오는 이미지들에 대해 원하는 시점에 요청하기 위해 `IntersectionObserver` 를 활용해 수동으로 Lazy Loading 을 해주기로 했다.

   ![image.png](/assets/posts/2025-05-07-web-lighthouse/image.png)

2. IntersectionObserver를 활용한 수동 Lazy Loading

   `img` 태그의 `loading="lazy"`는 브라우저에 로딩 타이밍을 위임하는 방식이기 때문에, 정확히 원하는 시점에 이미지 로딩을 제어하긴 어렵다. 예를 들어, 정확히 사용자 눈에 보일 때에만 이미지를 로드하고 싶다면, `IntersectionObserver`를 사용하는 방식이 더 적합하다. 이미지 로딩 로직은 다음과 같이 구성할 수 있다.

   ```tsx
   const imgRef = useRef(null)

   <img data-src={props.src} ref={imgRef} alt="..." />

   useEffect(() => {
     const callback = (entries, observer) => {
       entries.forEach((entry) => {
         if (entry.isIntersecting) {
           entry.target.src = entry.target.dataset.src;
           observer.unobserve(entry.target); // 로딩 후 더 이상 감시하지 않음
         }
       });
     };

     const options = {}; // 기본 감시 옵션
     const observer = new IntersectionObserver(callback, options);
     observer.observe(imgRef.current);

     return () => {
       observer.disconnect(); // 컴포넌트 언마운트 시 Observer 해제
     };
   }, []);
   ```

   이때 data-src 속성에 이미지 경로를 넣어두고, 화면에 노출되는 시점에 src 속성을 동적으로 할당하는 방식이다.


   **IntersectionObserver 세밀 제어: rootMargin 옵션**

   만약 화면에 딱 보이는 순간에 이미지를 로드하게 되면, 사용자에게 로딩되는 과정 자체가 보일 수도 있다. 이를 개선하려면, 이미지가 미리 로딩되도록 rootMargin 옵션을 활용해 여유 범위를 줄 수 있다.

   ```tsx
   const options = {
     rootMargin: "1000px 0px 500px 0px", // 위·아래 여유 공간을 둠
   };
   const observer = new IntersectionObserver(callback, options);
   ```

   rootMargin을 활용하면 뷰포트 기준 감지 범위를 확장할 수 있어, 사용자가 스크롤로 도달하기 전에 미리 이미지를 로딩할 수 있다. 예를 들어 위 코드는 상단 1000px, 하단 500px 범위 안에 요소가 들어오면 로딩을 시작한다는 의미다.

   이후 별도의 Custom Hook을 만들어 적용하였다.

   ```tsx
   const { ref, backgroundImage } = useLazyBackground(imageUrl);
   ```

   ![IntersectionOberserver lazy image loading 적용 후 2개만 불러오게 바꿨다](/assets/posts/2025-05-07-web-lighthouse/image%202.png)

   IntersectionOberserver lazy image loading 적용 후 2개만 불러오게 바꿨다

   적용 후의 모습이다. 4개의 이미지를 한 번에 불러오던 전의 상황에서 현재는 2개만 불러올 수 있게 제어했다. 작업을 함으로써 첫 요청 리소스도 1.5MB 감소하였다.

   ![image.png](/assets/posts/2025-05-07-web-lighthouse/image%203.png)

   이제까지는 lighthouse의 기기를 PC로 해놓았을 경우였다. 모바일에서도 예상한대로 리소스를 불러오는 지 확인할 필요가 있다. 아래는 모바일 환경에서 첫 렌더링시 불러오는 리스트 목록이다.

   ![image.png](/assets/posts/2025-05-07-web-lighthouse/image%204.png)

   모바일에서도 예측 가능한 리소스만 불러올 수 있게 추가 작업을 해줘야 했다. 위는 첫 로딩에 필요하지 않은 리소스를 표시해놨다. 표식이 된 리소스들은 첫 로딩시에 불러와지지 않게 수정할 것이다. `intersectionObserver` 로 뷰에 근접했을 때 불러올 수 있도록 했다.

   ![image.png](/assets/posts/2025-05-07-web-lighthouse/image%205.png)

    위는 예측 가능한 리소스만 불러올 수 있도록 정리된 모습이다.

3. LCP 리소스 프리로드 스캐너로 인지시키기

   위에서 첫 로딩시에 필요하지 않은 리소스들을 표시했었다. 그 중에 `time.webp` 는 첫 로딩시 가장 큰 이미지(LCP) 이다.

   ![image.png](/assets/posts/2025-05-07-web-lighthouse/image%206.png)

   `link` 태그를 활용해 프리스캐너에게 미리 `time.webp` 를 알려주어 번들 요청과 동시에 이미지를 다운로드 할 수 있다.

   ![image.png](/assets/posts/2025-05-07-web-lighthouse/image%207.png)

   이전에는 번들(`main.tsx` 와 같은)이 실행된 이후에 이미지 리소스들이 한꺼번에 다운로드 되었다면, 위의 경우는 번들을 다운로드함과 동시에 병렬로 LCP 점수에 측정되는 이미지를 미리 다운로드 받을 수 있다.


4. 뷰포트별 이미지 제공 (srcSet)

    이후에 작업을 하다보니 `image decode` 시간이 눈에 띄었다.

    ![image.png](/assets/posts/2025-05-07-web-lighthouse/image%208.png)

    두 이미지들은 webp 로 변환하더라도 사이즈가 컸던 이미지들이다. 디코딩이 오래 걸리는 이미지들은 확장자를 webp 에서 avif 로 변경해주었다. 추가적으로 디코딩 시간을 감소해주기 위해 뷰포트마다 사이즈를 맞춰 요청할 수 있도록 srcSet 을 사용해주었다.

    ```html
    <img
      srcset="time-small.avif 800w, time-medium.avif 900w, time-large.avif 1200w"
    />
    ```

    아래는 `srcSet` 을 지정해준 뒤의 모습이다.

    ![image.png](/assets/posts/2025-05-07-web-lighthouse/image%209.png)

    25.83ms → 8.09ms 로 이미지 디코딩 시간이 감소되고, 다른 14.30ms 디코딩 시간을 가지고 있던 이미지 또한 영역에서 사라졌다. 추가적으로 이 작업을 해줌으로써, 라이트하우스의 LCP 관련 내용도 사라졌다. `콘텐츠가 포함된 최대 페인트 요소` 의 내용이였다.

    ![image.png](/assets/posts/2025-05-07-web-lighthouse/image%2010.png)

    LCP 리소스 렌더링 지연 비율이 86% 차지하고 있었는데, 더이상 콘텐츠가 포함된 최대 페인트 요소 안내가 사라졌다.

    ![image.png](/assets/posts/2025-05-07-web-lighthouse/image%2011.png)

    이후 확인 해보니, 성능 점수가 77점 → 99점으로 변경되었다. 다른 부분들의 개선이 이루어진 이후 다시 데스크톱 라이트하우스를 확인해보았다.

    ![image.png](/assets/posts/2025-05-07-web-lighthouse/image%2012.png)

### 2. CLS

반응형 이미지에서는 `width: 100%`로 스타일을 주는 경우가 많다. 하지만 이렇게 하면 Lighthouse에서 “이미지에 width, height를 지정하라”는 경고 문구가 뜰 수 있다.
이때는 img 태그에 원본 크기(width, height)를 지정해두고, CSS에서 `width: 100%, height: auto`로 조정하면 된다.
이 방식은 반응형을 보장하면서도 이미지 영역을 미리 확보해 CLS 점수를 개선할 수 있다.

```html
<img src="..." width="300px" height="500px" style={{ width: '100%', height:
'auto' }}/>
```

![image.png](/assets/posts/2025-05-07-web-lighthouse/image%2013.png)

![image.png](/assets/posts/2025-05-07-web-lighthouse/image%2014.png)

### 3. 접근성 개선

그 외에 버튼 컴포넌트를 사용할 때에 [스크린 리더 접근성 확보](https://dequeuniversity.com/rules/axe/4.10/link-name)를 위해 `*aria-label` 을 추가했다.

![image.png](/assets/posts/2025-05-07-web-lighthouse/image%2015.png)

### 4. 결론

![image.png](/assets/posts/2025-05-07-web-lighthouse/image%2016.png)
