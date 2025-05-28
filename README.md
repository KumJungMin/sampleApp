# Next.js 프로젝트: GitHub Actions, S3, CloudFront를 이용해 자동 배포 파이프라인

## 1. 개요

- 이 문서는 Next.js 애플리케이션을 AWS S3에 정적 웹사이트로 배포하고,
- CloudFront를 통해 전 세계 사용자에게 빠르고 안정적으로 콘텐츠를 제공하는 방법을 설명합니다.
- 또한, GitHub Actions를 활용해 전체 배포 과정을 자동화하는 CI/CD 파이프라인의 구축 방식을 정리하였습니다.

<br/>
<br/>

## 2. 아키텍처 다이어그램


<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FF6KCK%2FbtsOhKgreqK%2FgqkiwExITRk8HW23gEcgzK%2Fimg.png" />



**다이어그램 설명:**

1.  **개발자 (Developer):** 로컬 환경에서 코드를 개발하고 변경 사항을 GitHub 저장소의 `main` 브랜치로 푸시(Push)합니다.
2.  **GitHub Repository:** 코드 변경 사항을 감지하거나 수동 실행(`workflow_dispatch`)을 통해 GitHub Actions 워크플로우를 구동합니다.
3.  **GitHub Actions Runner:**
    |단계|설명|
    |---|---|
    |**Checkout:**|지정된 브랜치(예: `main`)의 최신 코드를 가져옵니다.|
    |**Install Dependencies:**|`npm ci` 명령어를 사용해 프로젝트의 의존성을 설치합니다.<br/>이 명령을 사용하면 package-lock.json에 명시된 버전으로 설치되기에,<br/>CI 환경(예: GitHub Actions)에서 안정적인 빌드를 위해 권장됩니다.|
    |**Build:**|`npm run build` 명령을 실행하여 Next.js 애플리케이션을 정적 파일로 빌드합니다. <br/>(_`out` 폴더에 결과물이 생성됨_)|
    |**Configure AWS Credentials:**|GitHub Secrets에 저장된 AWS 액세스 키와 시크릿 키를 사용해,<br/>GitHub Actions가 AWS 리소스에 접근할 수 있도록 자격 증명을 설정합니다.|
    |**Deploy to S3:**|빌드된 정적 파일들(`out/` 디렉토리)을 지정된 S3 버킷으로 동기화합니다.<br/>`--delete` 옵션을 사용하여 S3 버킷에는 있지만 빌드 결과물에는 없는 파일을 삭제하여 일관성을 유지합니다.|
    |**Invalidate CloudFront Cache:**|S3에 새로운 파일이 배포된 후,<br/>CloudFront 배포의 캐시를 무효화하여 사용자가 즉시 최신 콘텐츠를 볼 수 있도록 합니다.<br/>`/*` 경로는 모든 파일에 대한 캐시를 무효화합니다.|
4.  **AWS S3:** 빌드된 정적 웹사이트 파일(HTML, CSS, JavaScript, 이미지 등)을 저장하고 호스팅합니다.
5.  **AWS CloudFront:** S3 버킷을 오리진(Origin)으로 하는 CDN(Content Delivery Network)입니다. 전 세계 엣지 로케이션에 콘텐츠를 캐싱하여 사용자에게 낮은 지연 시간으로 콘텐츠를 제공하고, SSL/TLS 인증서 관리 및 트래픽 분산 등의 기능을 수행합니다.
6.  **사용자 (End user):** 웹 브라우저를 통해 CloudFront 배포 도메인으로 애플리케이션에 접근합니다. 가장 가까운 엣지 로케이션에서 캐시된 콘텐츠를 받거나, 캐시 미스(Cache Miss) 시 CloudFront가 S3 오리진에서 콘텐츠를 가져와 사용자에게 전달하고 캐싱합니다.

**(_주의: 상용 환경에서는 사용자가 특정 도메인(예: `www.example.com`)으로 접근 시,<br/>
AWS Route 53 (DNS 서비스)이 이 도메인 요청을 CloudFront 배포로 라우팅하는 단계가 추가됩니다._)**

<br/>
<br/>

## 3. GitHub Actions 워크플로우 설명 (`.github/workflows/deployment.yml`)

```yaml
name: Deploy Next.js to S3 and invalidate CloudFront # 워크플로우 이름

on:
  push:
    branches:
      - main  # main 브랜치에 push 이벤트 발생 시 실행
  workflow_dispatch: # GitHub Actions UI에서 수동으로 실행 가능

jobs:
  deploy: # 'deploy'라는 이름의 작업
    runs-on: ubuntu-latest # Ubuntu 최신 버전 환경에서 실행
    
    steps:
    - name: Checkout repository # 단계 1: 코드 체크아웃
      uses: actions/checkout@v4 # GitHub의 체크아웃 액션 사용

    - name: Install dependencies # 단계 2: 의존성 설치
      run: npm ci # package-lock.json을 기반으로 의존성 설치

    - name: Build # 단계 3: 프로젝트 빌드
      run: npm run build # Next.js 프로젝트 빌드 (정적 파일 생성)

    - name: Configure AWS credentials # 단계 4: AWS 자격 증명 설정
      uses: aws-actions/configure-aws-credentials@v1 # AWS 공식 자격 증명 설정 액션 사용
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }} # GitHub Secrets에서 Access Key ID 로드
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }} # GitHub Secrets에서 Secret Access Key 로드
        aws-region: ${{ secrets.AWS_REGION }} # GitHub Secrets에서 AWS 리전 로드

    - name: Deploy to S3 # 단계 5: S3에 배포
      run: |
        aws s3 sync out/ s3://${{ secrets.S3_BUCKET_NAME }} --delete 
        # 'out/' 디렉토리의 내용을 S3 버킷과 동기화. --delete 옵션으로 불필요한 파일 삭제

    - name: Invalidate CloudFront cache # 단계 6: CloudFront 캐시 무효화
      run: |
        aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
        # 지정된 CloudFront 배포 ID의 모든 경로("/*")에 대해 캐시 무효화 요청
```



