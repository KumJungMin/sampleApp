# CDN 도입 전후 성능 비교


## 1. 서론
- 본 문서는 웹 애플리케이션의 사용자 경험을 향상시키고 전반적인 성능을 최적화하기 위해 CDN(Content Delivery Network)을 도입한 후의 성능 변화를 분석하고 평가하는 것을 목표로 합니다.
- 내용은, CDN 도입 전(AWS S3 직접 접근)과 도입 후(AWS CloudFront 경유)의 성능 지표를 네트워크 패널 데이터를 기반으로 비교하여 작성하었습니다.


<br/>

## 2. CDN 도입 후 성능 개선 상세 보고


### 2.1. 주요 성능 지표 비교
| 지표                    | CDN 도입 전 (S3) | CDN 도입 후 (CloudFront) | 개선폭      |
| --------------------- | ------------- | --------------------- | -------- |
| **DOMContentLoaded**  | 1.15 s        | **0.227 s**           | ▲ 80.2 % |
| **Load 완료**           | 1.56 s        | **0.534 s**           | ▲ 65.8 % |
| **TTFB (HTML 304)**   | 540 ms        | **10 ms**             | ▲ 98.1 % |
| 작은 JS 번들 (0.3 KB)     | 300 ms        | **11 ms**             | ▲ 96.3 % |
| `favicon.ico` (26 KB) | 220 ms        | **40 ms**             | ▲ 81.8 % |
| `plugin.js` (5.5 MB)  | 249 ms        | **298 ms**            | ▼ 19.7 % |
| `page.js` (2.7 MB)    | 145 ms        | **262 ms**            | ▼ 80.7 % |


<br/>

| CDN 도입 전 (S3)| CDN 도입 후 (CloudFront)|
|---|---|
|<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbSeRVm%2FbtsOhaGvtw7%2FOvOrIoBjjXizjKqhf7MyeK%2Fimg.png" />|<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbYclBC%2FbtsOgOxb4xj%2FO95bvMMVwaqUhNGS5IZIW0%2Fimg.png" />|
|<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbwzdJP%2FbtsOhI3X0Kd%2FK6BOHDMatMpCDdD2G2Xd8k%2Fimg.png" />|<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FAAWCT%2FbtsOgOYgecb%2Fxg8ZDXswmi9pbSAdT83wVk%2Fimg.png" />|

<br/>

### 2.2. 성능 지표 비교 결과
#### 전반적인 페이지 로딩 속도 향상: ✅
- DOMContentLoaded 시간은 1.15초에서 227ms로 약 80.2% 단축되었으며, 전체 페이지 Load 시간은 1.56초에서 534ms로 약 65.8% 단축되었습니다. 이는 사용자가 페이지의 주요 콘텐츠를 훨씬 빠르게 인식하고 상호작용할 수 있게 되었음을 의미합니다.

#### HTML 문서 및 작은 정적 에셋의 극적인 로딩 시간 단축: ✅
- HTML 문서 자체의 로딩 시간(304 응답 기준)이 540ms에서 10ms로 98% 이상 단축된 것은 CDN의 핵심 이점인 지리적 근접성을 통한 지연 시간 감소를 명확히 보여줍니다. 이는 사용자가 페이지에 접속했을 때 가장 먼저 느끼는 반응 속도 개선에 크게 기여합니다.
- 304 응답을 받는 여러 작은 JS 파일들의 평균 로딩 시간 또한 약 300ms에서 11ms로 96% 이상 단축되었습니다. 이는 CDN 엣지 서버에서의 효율적인 캐시 검증 및 빠른 응답 덕분입니다.

#### 대용량 파일 로딩 시간 분석 (plugin.js, page.js): 🚨
- plugin.js (5.5MB)와 page.js (2.7MB) 같은 대용량 파일의 경우, 제공된 테스트 결과에서는 CDN 도입 후 로딩 시간이 오히려 소폭 증가한 것으로 나타났습니다 (plugin.js: 249ms -> 298ms, page.js: 145ms -> 262ms)


<br/>

## 3. CDN 도입 전후 성능 차이 발생 원인

> CDN 도입 전 후로 성능 차이가 발생하는 이유는 다음과 같습니다.

#### 1. 지리적 근접성 및 지연 시간(Latency) 감소:
- **CDN 도입 전:** 사용자는 지리적으로 멀리 떨어져 있을 수 있는 단일 지역의 원본 서버(예: 특정 리전의 S3 버킷)에서 모든 정적 에셋(HTML, CSS, JS, 이미지 등)을 다운로드해야 합니다. 이로 인해 사용자와 서버 간의 물리적 거리에 비례하여 데이터 요청 및 응답에 소요되는 시간(RTT, Round Trip Time)이 길어집니다.
- **CDN 도입 후:** CDN은 전 세계 여러 지역에 분산된 셔버(엣지 로케이션) 네트워크를 사용합니다. 사용자가 웹사이트에 접속하면, 가장 가까운 엣지 로케이션에서 콘텐츠를 제공받게 됩니다. 이는 데이터 전송 경로를 크게 단축시켜 첫 바이트 수신 시간(TTFB, Time To First Byte)을 포함한 전반적인 지연 시간을 줄입니다. 실제 측정 결과, HTML 문서(0.3KB, 304 응답) 로딩 시간이 540ms였던 것이 CloudFront 도입 후 10ms로 단축된 것은 이러한 지연 시간 감소 효과를 보여줍니다.

#### 2. 효율적인 캐싱(Caching) 전략:
- **CDN 도입 전:** 브라우저 캐시 외에는 서버 측 캐싱 전략이 제한적일 수 있습니다. 동일한 파일을 여러 사용자가 반복적으로 요청할 때마다 원본 서버는 해당 요청을 처리해야 합니다.
- **CDN 도입 후:** CDN 엣지 로케이션은 자주 요청되는 정적 에셋의 복사본을 캐시합니다. 특정 엣지 로케이션에 사용자가 처음 접속하여 에셋을 요청하면(Cache Miss), 엣지 서버는 원본 서버에서 에셋을 가져와 사용자에게 전달하고 동시에 캐시합니다. 이후 동일 엣지 로케이션으로 다른 사용자가 같은 에셋을 요청하면(Cache Hit), 원본 서버까지 갈 필요 없이 엣지 서버에서 즉시 빠르게 제공받을 수 있습니다. 이는 원본 서버의 부하를 줄이고, 응답 속도를 크게 향상시킵니다. 이미지에서 다수의 작은 JS 파일들이 304(Not Modified) 응답을 받을 때, S3에서는 260ms ~ 341ms가 소요된 반면 CloudFront에서는 9ms ~ 12ms로 크게 단축된 것은 엣지 캐싱 및 빠른 상태 확인의 결과입니다.

#### 3. 높은 데이터 전송 속도 및 병렬 다운로드 최적화:
- CDN은 일반적으로 대역폭이 높은 네트워크 인프라를 갖추고 있어, 대용량 파일도 빠르게 전송할 수 있습니다.
- 또한, 브라우저는 동일 도메인에 대한 동시 연결 수에 제한이 있지만, CDN을 사용하면 여러 엣지 로케이션 또는 최적화된 연결을 통해 에셋 다운로드를 더 효율적으로 병렬 처리할 수 있습니다.

<br/>

## 4. 결론 및 권장 사항

- CDN(AWS CloudFront) 도입은 전반적인 페이지 로딩 속도, 특히 HTML 문서 및 작은 정적 에셋의 응답 시간을 획기적으로 개선하여 사용자 경험 향상에 크게 기여하는 것으로 확인되었습니다.
- DOMContentLoaded 시간은 약 80%, 전체 Load 시간은 약 66% 단축되는 등 정량적인 성과가 뚜렷했습니다.
- 대용량 파일의 초기 로드 시 약간의 시간 증가가 관찰되었으나, 반복 요청 및 지리적으로 분산된 사용자를 고려할 때 장기적으로는 성능 이점이 클 것으로 예상됩니다.

