# 프론트엔드 배포 파이프라인

## 개요

![배포 파이프라인 다이어그램](./your-diagram-file.png) <!-- 본인 다이어그램 파일 경로 삽입 -->

GitHub Actions에 워크플로우를 작성해 다음과 같이 배포가 진행되도록 구성했습니다.

1. GitHub Actions에서 `actions/checkout`으로 코드 가져오기
2. `npm ci` 명령어로 의존성 설치
3. `npm run build` 명령어로 Next.js 정적 파일 빌드
4. AWS 자격 증명 (S3, CloudFront 배포 권한) 구성
5. `aws s3 sync` 명령어로 빌드된 파일을 S3에 업로드
6. `aws cloudfront create-invalidation` 명령어로 캐시 무효화 (CDN 최신화)

## 주요 링크

- **S3 버킷 웹사이트 엔드포인트**: http://carrot333bucket.s3-website.eu-north-1.amazonaws.com
- **CloudFront 배포 도메인 이름**: https://dlf0oywdwkhb5.cloudfront.net

## 주요 개념

- **GitHub Actions + CI/CD**: 저장소에 push 시 자동으로 빌드/배포
- **S3 (정적 스토리지)**: HTML, CSS, JS 정적 리소스를 호스팅
- **CloudFront (CDN)**: 지리적으로 가까운 엣지 로케이션을 통해 빠른 응답 제공
- **캐시 무효화 (Invalidation)**: 최신 변경 사항이 반영되도록 캐시 제거
- **Secrets / 환경변수 설정**: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, DISTRIBUTION_ID 등 GitHub Secrets에 설정

## CDN 도입 전후 성능 개선 비교

| 항목 | CDN 도입 전 | CDN 도입 후 |
|------|--------------|-------------|
| 초기 페이지 로드 시간 | 1.87초 | **823ms** |
| 주요 JS 파일 응답 속도 | 500ms | **120ms** |
| 전체 요청 수 | 32 | 32 (동일) |
| `x-cache` 헤더 | Miss from CloudFront | **Hit from CloudFront** |

**결론**: CloudFront CDN 도입 후 주요 정적 리소스 응답 속도가 약 3~4배 빨라졌으며, 전체 페이지 로딩 시간이 절반 이하로 감소하였습니다. 이는 사용자 경험(UX) 및 SEO에도 긍정적 영향을 미칩니다.
