# 프론트엔드 배포 파이프라인

## 개요

<table>
  <tr>
    <td style="width: 50%;">
      <img src="https://github.com/user-attachments/assets/7d4ad27e-9407-49e3-b609-a158ff3501b4" width="100%">
    </td>
    <td style="vertical-align: top; padding-left: 20px;">
      <h3>배포 파이프라인 개요</h3>
      <p>GitHub Actions에 워크플로우를 작성해 </br>다음과 같이 배포가 진행되도록 구성했습니다.</p>
      <ol>
        <li><code>actions/checkout</code>으로 코드 가져오기</li>
        <li><code>npm ci</code> 명령어로 의존성 설치</li>
        <li><code>npm run build</code> 명령어로 Next.js 정적 파일 빌드</li>
        <li>AWS 자격 증명 구성 (S3/CloudFront)</li>
        <li><code>aws s3 sync</code>로 S3에 빌드 파일 업로드</li>
        <li><code>aws cloudfront create-invalidation</code>로 캐시 무효화</li>
      </ol>
    </td>
  </tr>
</table>

## 주요 링크

- **S3 버킷 웹사이트 엔드포인트**: http://carrot333bucket.s3-website.eu-north-1.amazonaws.com
- **CloudFront 배포 도메인 이름**: https://dlf0oywdwkhb5.cloudfront.net

## 주요 개념

> **GitHub Actions + CI/CD**:
>- 정의: GitHub 저장소에 코드가 push되면 자동으로 실행되는 워크플로우
>- 기능: 테스트, 빌드, 배포 등 모든 작업 자동화
>- 장점: 수동 배포 오류 방지, 반복 작업 제거, 빠른 피드백
  
> **S3 (정적 스토리지)**:
>- 정의: AWS의 객체 저장 서비스
>- 용도: 정적 웹사이트(HTML, CSS, JS, 이미지 등) 호스팅
>- 특징: 고가용성, 내구성 99.999999999%, 확장성

> **CloudFront (CDN)**:
>- 정의: AWS의 글로벌 콘텐츠 전송 네트워크
>- 기능: 사용자와 가까운 위치(엣지 로케이션)에서 빠르게 콘텐츠 제공
>- 장점: 응답 속도 개선, 트래픽 분산, DDoS 방어

> **캐시 무효화 (Invalidation)**:
>- 정의: CloudFront에 저장된 캐시를 강제로 갱신하는 작업
>- 필요성: 빌드 후 변경된 파일이 사용자에게 즉시 반영되도록 하기 위함
>- 명령어 예시: `aws cloudfront create-invalidation --paths "/*"`

> **Secrets / 환경변수 설정**:
>- 정의: 민감한 정보(AWS 키 등)를 GitHub 저장소 내에 안전하게 보관하는 방식
>- 용도: 워크플로우 실행 시 환경 변수로 주입되어 보안 유지
>- 설정 예시: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `DISTRIBUTION_ID`



## CDN 도입 전후 성능 개선 비교

| 항목                               | **CDN 도입 전** (S3 직접 접속) | **CDN 도입 후** (CloudFront 경유) |
| -------------------------------- | ----------------------- | ---------------------------- |
| 초기 페이지 로드 시간 (`document`)        | **1.08초**               | **45ms**                     |
| 주요 JS 파일 응답 속도 (`main-app-*.js`) | **약 1.3\~1.4초**         | **약 37ms**                   |
| 전체 요청 수                          | 18건 이상                  | 18건 이상 (동일)                  |
| 평균 이미지 및 폰트 응답 속도                | 약 **500\~600ms**        | **30\~40ms**                 |
| `x-cache` 헤더                     | ❌ 없음 (S3 직접 요청)         | ✅ **Hit from CloudFront**    |
| 사용자 체감 속도                        | 눈에 띄는 지연 있음             | **즉시 반응** 수준                 |

<p float="left">
  <img src="https://github.com/user-attachments/assets/10aa3c36-eb29-4d4b-a184-d20e2d4be46e" width="45%" />
  <img src="https://github.com/user-attachments/assets/cd3699f5-b432-4aeb-877e-1fcb7a4f9427" width="45%" />
</p>


## 🌐 CDN 캐시 구조 및 엣지 로케이션 설명

<details>
<summary><strong>📌 엣지 로케이션이란?</strong></summary>

**CloudFront 엣지 로케이션(Edge Location)**은 전 세계에 분산된 CDN 캐시 서버입니다.  
사용자의 요청을 가장 가까운 위치에서 처리하여 응답 속도를 크게 개선합니다.

</details>

<details>
<summary><strong>✈️ 사용자 요청 흐름</strong></summary>

### 1. 최초 요청 (엣지에 캐시 없음)
- 사용자가 CloudFront URL로 웹사이트에 접속  
- 가장 가까운 엣지 로케이션으로 요청이 전송됨  
- 캐시에 없을 경우, S3 원본에서 데이터를 받아와 사용자에게 전달  
- 동시에 엣지 로케이션에 콘텐츠가 캐싱됨  

### 2. 후속 요청 (캐시 존재)
- 동일한 지역의 다른 사용자가 같은 콘텐츠를 요청  
- 엣지 로케이션에서 바로 응답 (S3에 재요청하지 않음)  
- 응답 속도: 수백 ms → 수십 ms 수준  

</details>

<details>
<summary><strong>📍 내 상황에서의 실제 적용</strong></summary>

- ✅ 한국 사용자 기준: CloudFront는 **서울 또는 도쿄 리전의 엣지 서버** 사용  
- 평균 응답 속도 **500ms 이상 → 30~40ms** 수준으로 개선됨  
- 이미지, JS, CSS 리소스 모두 엣지에서 제공됨  
- `x-cache: Hit from CloudFront` 확인됨 → 캐시가 정상 작동 중

  
<p float="left">
<img width="48%" src="https://github.com/user-attachments/assets/a5a47a37-2e22-4d80-a79c-ecb3d00db9b3"  width="45%"  />
<img width="48%" src="https://github.com/user-attachments/assets/33bf7b8a-bdd1-4fdd-b5d7-7e7f9193a213"  width="45%"  />
</p>



</details>

<details>
<summary><strong>🧠 추가 고려사항</strong></summary>

| 항목 | 설명 |
|------|------|
| **TTL (Time to Live)** | 캐시 유지 시간 설정 가능 (기본 24시간 등) |
| **Invalidation** | 변경된 파일을 즉시 반영하려면 캐시 무효화 필요 |
| **동적 콘텐츠** | 로그인 등 개인화된 콘텐츠는 캐싱 대상에서 제외하거나 TTL을 짧게 설정 |

</details>

<details>
<summary><strong>📈 기대 효과 요약</strong></summary>

| 항목 | 효과 |
|------|------|
| **응답 속도 개선** | 사용자와 가까운 위치에서 제공 → 로딩 체감 속도 대폭 상승 |
| **S3 부하 감소** | 반복 요청을 엣지에서 처리 → 원본 비용 및 트래픽 절감 |
| **UX 향상** | 지연 없는 응답 → 사용자 이탈률 감소, SEO 개선 |

</details>

</br>


**결론**: 
> 초기 로드 시간이 1.08초 → 45ms로 20배 이상 개선
> JS/폰트/이미지 등 정적 리소스 응답 속도가 평균 500ms 이상 → 30~40ms 수준으로 감소
> CloudFront 캐시가 잘 동작하며 (Hit from CloudFront), 사용자는 S3 원본이 아닌 엣지 서버에서 응답을 빠르게 받음 </br>
> 네트워크 요청 수는 같지만 성능과 반응성에서 극명한 차이가 발생함
