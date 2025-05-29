# 항해 5기 9주차. 네트워크/인프라 관점의 성능 최적화

네트워크 기반 프론트엔드 성능 최적화 실습 프로젝트

## 다이어그램
![image](https://github.com/user-attachments/assets/f77b200e-4d5c-4871-b4ff-ed45257c7e32)

## 주요 링크

- S3 버킷 웹사이트 엔드포인트: http://clarion.kr.s3-website.ap-northeast-2.amazonaws.com/
- CloudFrount 배포 도메인 이름: https://d1pw52sct3vs7x.cloudfront.net/
- Route53으로 도메인 연결 : https://clarion.kr/

## 주요 개념

- GitHub Actions과 CI/CD 도구
```yaml
name: Deploy Next.js to S3 and invalidate CloudFront # 워크플로우의 이름 지정

on: # 워크플로우를 트리거할 이벤트 정의
  push: # git에 push가 발생했을 때 실행
    branches: # 아랫줄의 적힌 브랜치 명에 push될 때 실행
      - main
  workflow_dispatch: # 수동 실행을 가능하게 함 (GitHub UI에서 실행 가능)

jobs: # 실행할 작업들 정의
  deploy: # 작업 이름. 유니크 하기만 하면 자유롭게 변경가능. 여러 작업을 작성할 수 있음
    runs-on: ubuntu-latest # GitHub가 제공하는 최신 우분투 환경에서 작업을 실행
    
    steps: # 작업 순서
    - name: Checkout repository # 작업 이름
      uses: actions/checkout@v4 # GitHub Actions에서 가장 많이 쓰이는 액션 중 하나. workflow가 실행중인 러너에 현재 저장소를 clone 해줌

    - name: Install dependencies
      run: npm ci # `package-lock.json` 기준으로 패키지를 설치하여 일관성 보장

    - name: Build
      run: npm run build # 빌드 명령 실행

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1 # GitHub에서 제공하는 AWS 인증 액션 사용
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}

    - name: Deploy to S3
      run: |
        aws s3 sync out/ s3://${{ secrets.S3_BUCKET_NAME }} --delete # out 폴더의 파일을 S3 버킷과 동기화, 불필요한 파일은 삭제

    - name: Invalidate CloudFront cache
      run: |
        aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*" # 전체 캐시 경로 무효화
```

## 성능비교

![image](https://github.com/user-attachments/assets/bb21b902-5f02-476c-8b58-7a45ad6e0985)

### 🔍 성능 비교 요약

| 항목                      | S3 직접 접근 (좌측) | CloudFront 접근 (우측) | 비교 결과            |
| ----------------------- | ------------- | ------------------ | ---------------- |
| **문서 전송 시간**            | 23 ms         | 6 ms               | -17ms (73.9%) |
| **DOMContentLoaded 시간** | 84 ms         | 62 ms              | -22ms (26.2%) |
| **로드 완료 시간**            | 159 ms        | 110 ms             | -49ms (30.8%) |
| **전송량 (Transferred)**   | 491 kB        | 207 kB             | -284kb (57.8%)  |
| **리소스 크기 (Resources)**  | 485 kB        | 485 kB             | 동일               |
| **요청 수**                | 17건           | 17건                | 동일               |


### 📈 세부 분석
1. 초기 로딩 시간
- CloudFront: 6ms (HTML 문서 전송), DOMContentLoaded: 62ms, Load: 110ms
- S3: 23ms (HTML 문서 전송), DOMContentLoaded: 84ms, Load: 159ms
- ➡️ CloudFront는 전반적으로 렌더링까지 약 30~50ms 빠르게 완료됨.

2. 데이터 전송량
- S3: 전송된 양이 491 kB
- CloudFront: 전송된 양이 207 kB
- ➡️ 동일한 리소스 크기(485 kB)임에도 불구하고 CloudFront는 압축 또는 캐싱 효과로 전송량이 절반 수준

3. 캐싱 및 압축 효과
- CloudFront는 HTTP 헤더에서 gzip, brotli 등 압축이 활성화되어 전달되는 경우가 많고,
- 에지 로케이션 캐시를 통해 거리 및 응답 시간 개선 가능
- ➡️ 클라이언트와의 물리적 거리 및 처리 속도가 향상됨.

### 📝 결론
| 항목     | CloudFront 사용 효과            |
| ------ | --------------------------- |
| 속도 향상  | 초기 로딩 속도와 렌더링 타임 개선         |
| 용량 절감  | 전송량 최소화 → 모바일 환경, 대역폭 비용 절감 |
| 사용자 경험 | 빠른 로딩으로 인한 UX 개선            |
