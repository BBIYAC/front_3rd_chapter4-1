## Git Actions 배포 워크플로우

Next.js 애플리케이션을 Amazon S3에 배포하고 CloudFront 캐시를 무효화하는 GitHub Actions 워크플로우를 설명합니다. 워크플로우는 **main 브랜치로 푸시**되거나 Actions 페이지에서  **수동으로 실행**될 때 트리거됩니다.

![image](https://github.com/user-attachments/assets/3b7ff5ea-08c0-43e2-a0a3-b145aee071f4)

1. 코드 베이스를 가져오기 위한 **레파지토리 체크아웃**
2. `npm ci` 를 통한 **의존성 설치**
3. **Next.js 빌드** 를 통한 정적 파일 생성 `npm run build`
4. S3 및 CloudFront 작업을 위한 **AWS 자격 증명 설정**
5. 생성된 빌드 파일을 **S3에 배포** 하여 동기화
6. 최신 파일 제공을 위한 **CloudFront 캐시 무효화**

## 배포 링크

- [S3 버킷 웹사이트](http://hanghae-plus-frondend-3rd.s3-website.ap-northeast-2.amazonaws.com/)
- [CloudFrount 배포 URL](https://d3qgmoc1zjnt2o.cloudfront.net)
  
## 주요 개념

### 1. GitHub Actions과 CI/CD 도구

GitHub Actions는 GitHub에서 제공하는 **자동화 도구**입니다.

- 본 레파지토리에서는 **CI/CD** 워크플로우를 설정하는 데 사용됩니다.
- **CI(Continuous Integration)**
  - 코드 변경 사항을 지속적으로 통합하고 자동으로 빌드 및 테스트하여 문제를 조기에 발견할 수 있도록 합니다.
  - 워크플로우에서는 `npm ci` 명령어를 통해 의존성을 설치하고 `npm run build`로 애플리케이션을 빌드합니다.
- **CD(Continuous Deployment)**
  - 검증된 코드를 자동으로 프로덕션 환경에 배포하여 소프트웨어 릴리스 과정을 간소화합니다.
  - 워크플로우에서는 빌드 산출물을 S3 버킷에 업로드하고, CloudFront 캐시를 무효화하여 새로운 콘텐츠가 사용자에게 즉시 제공되도록 합니다.

### 2. S3와 스토리지

**Amazon S3(Simple Storage Service)** 는 AWS에서 제공하는 **객체 스토리지 서비스**입니다.

- 정적 웹 콘텐츠(HTML, CSS, JS)를 저장할 수 있는 안정적이고 확장 가능한 데이터 저장소를 제공합니다.
- `aws s3 sync` 명령어를 사용해 빌드된 정적 파일들을 S3 버킷에 업로드합니다.
- `--delete` 플래그를 사용해 S3 버킷의 불필요한 파일을 제거하여 최신 상태를 유지합니다.

### 3. CloudFront와 CDN

**Amazon CloudFront** 는 **글로벌 콘텐츠 전송 네트워크(CDN) 서비스**입니다.

- 사용자의 지리적 위치에 따라 가까운 엣지 서버에서 콘텐츠를 제공하여 응답 속도를 크게 향상시킵니다.
- 캐시 무효화를 통해 최신 웹 콘텐츠를 **사용자에게 빠르고 안전하게 전달**할 수 있도록 지원합니다.

### 4. 캐시 무효화(Cache Invalidation)

캐시 무효화는 CDN에 저장된 **오래된 캐시 데이터를 제거하고 새로운 데이터를 적용하는 과정**을 의미합니다.

- 배포 후 사용자들이 즉시 최신 콘텐츠에 접근할 수 있도록 `aws cloudfront create-invalidation` 명령어를 실행하여 CloudFront의 캐시를 무효화합니다.
- YAML에서 특정 경로만 지정하여 캐시 무효화 처리가 가능하며, 본 워크플로우에서는 `--paths "/*"` 옵션을 통해 CloudFront의 모든 캐시를 무효화합니다.

### 5. Repository secret과 환경변수

GitHub Repository Secret은 **코드에 민감한 데이터가 노출되지 않고 안전하게 저장하고 관리**하기 위한 GitHub 기능입니다.

- `환경변수(Environment Variables)`: 워크플로우 실행 시 필요한 설정값(API 키, 비밀번호 등)을 저장하고 참조할 수 있도록 지원합니다.
- GitHub Actions에서 `secrets`를 활용하여 보안성을 유지하면서 배포 워크플로우를 구성할 수 있습니다.
```yaml
- name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}
```

## CDN 도입을 통한 성능 최적화

<img width="912" alt="image" src="https://github.com/user-attachments/assets/372de55a-4e4a-4b2a-ad5d-ee776efa2d66">

### 1. 콘텐츠 크기 비교

- S3 요청: 콘텐츠 크기 변화 없음 (transferred 용량 유지)
- CloudFront 요청: 콘텐츠 크기 `506kB` → `266kB`로 감소
- 효과: CloudFront의 캐싱으로 서버로의 리소스 요청 횟수 감소

### 2. 응답 속도 비교

- S3 요청: Load 시간 `590ms`, Finish 시간 `573ms`
- CloudFront 요청: Load 시간 `336ms`, Finish 시간 `360ms`
- 효과: CloudFront 사용 시 응답 속도가 Load 시간 `43%` 단축, Finish 시간 `37%` 단축

### 3. 상세 성능 지표

| **지표**          | **S3 요청** | **CloudFront 요청** | **개선율**         |
|--------------------|-------------|----------------------|--------------------|
| 콘텐츠 크기       | 506kB       | 266kB               | **47% 감소**       |
| Load 시간         | 590ms       | 336ms               | **43% 단축**       |
| Finish 시간       | 573ms       | 360ms               | **37% 단축**       |

### 4. 결론

정적 콘텐츠(JS, CSS, 이미지, 동영상 등) 을 다수 사용하는 경우 자주 요청되는 파일이 캐싱되어 네트워크 요청과 서버 부하 감소 효과를 기대할 수 있습니다. 또한 글로벌 사용자를 대상으로 가까운 엣지 서버에서 콘텐츠를 제공해 지연을 줄이고 사용자 경험을 향상시킬 수 있습니다.
