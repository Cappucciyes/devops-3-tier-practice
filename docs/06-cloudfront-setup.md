# 실습 6. CloudFront 연결

## 목표

- CloudFront Distribution 생성
- S3 (Frontend) + ALB (Backend API)를 Origin으로
- OAC로 S3 보호
- Behavior로 경로 기반 라우팅
- HTTPS 접속 테스트

## Step 1. CloudFront Distribution 생성

CloudFront 콘솔 → Create distribution

<img width="1587" height="1125" alt="image" src="https://github.com/user-attachments/assets/ce8e82ba-9286-4900-bf94-dfe657d89d2f" />

Free-Tier 말고 Pay-as-you-go 선택

<img width="2000" height="957" alt="image" src="https://github.com/user-attachments/assets/b3de5f1d-60a3-4041-bed7-7331c24390c8" />

### Origin 1: S3 (Frontend)

**Origin domain**
- 드롭다운에서 S3 버킷 선택
- `guestbook-frontend-hojun121.s3.ap-northeast-2.amazonaws.com`

<img width="1672" height="1125" alt="image" src="https://github.com/user-attachments/assets/49355e71-d7eb-4bb7-bade-b262c29c8857" />

**WAF 미사용**
<img width="2000" height="455" alt="image" src="https://github.com/user-attachments/assets/38639d74-18cf-4d83-9780-4bede10b98b2" />

### CloudFront 생성 (Create distribution)
- 배포까지 약 5~15분 소요.
<img width="2000" height="997" alt="image" src="https://github.com/user-attachments/assets/09ff38a6-6118-4a80-ab39-99b6615ddddf" />


### (중요) Default root object ⭐
- 생성 이후 General 탭에서 Default root object 이름 변경
- `index.html` 입력 (앞에 `/` 없이)
- **필수.** 비워두면 루트(`/`) 접근 시 CloudFront가 객체를 못 찾아 S3가 AccessDenied 반환 → 방명록 화면이 안 뜸
- S3 정적 호스팅을 끄고 OAC로만 접근하는 구조이므로, "루트 → index.html" 매핑은 S3가 아니라 **CloudFront의 이 설정**이 책임진다

<img width="2000" height="1650" alt="image" src="https://github.com/user-attachments/assets/77d4ff73-a31f-49d7-8acd-09b1ef59ed05" />


## Step 2. S3 버킷 정책 업데이트 (OAC 적용)

CloudFront가 알려주는 버킷 정책 복사 → S3 버킷 정책에 붙여넣기.

Distribution 상세 → Origins 탭 → S3 origin 선택 → Edit → 하단에 "Copy policy" 버튼

또는 수동:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::guestbook-frontend-hojun121/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EXXXXXXXXXXX"
        }
      }
    }
  ]
}
```

## Step 3. Origin 2 추가: ALB (Backend API)

Distribution 상세 → Origins 탭 → Create origin

**Origin domain**
- ALB DNS 입력: `guestbook-alb-xxxxx.ap-northeast-2.elb.amazonaws.com`

**Protocol**
- **HTTP only** (ALB가 HTTP만 열려있음)
- Port: 80

**Name**
- `alb-origin`

Create origin.

## Step 4. Behavior 추가: /api/* → ALB

Distribution 상세 → Behaviors 탭 → Create behavior

**Settings**
- Path pattern: `/api/*`
- Origin: `alb-origin`

**Cache behavior**
- Viewer protocol policy: **Redirect HTTP to HTTPS**
- Allowed HTTP methods: **GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE** ⭐ (API이므로)

**Cache key and origin requests**
- Cache policy: **CachingDisabled** ⭐ (API는 캐싱 안 함)
- Origin request policy: **AllViewer**

Create behavior.

## Step 5. Behavior 우선순위 확인

Distribution → Behaviors 탭에서 순서 확인:

| Precedence | Path pattern | Origin |
|---|---|---|
| 0 | `/api/*` | alb-origin |
| 1 | Default (*) | S3 |

`/api/*`가 위에 있어야 함. 아니면 Move up.

## Step 6. 배포 완료 대기

Distribution 상태가 `Deploying` → `Enabled`로 변경 대기 (5~15분).

## Step 7. 접속 테스트

Distribution의 **Distribution domain name** 복사:
```
https://dxxxxxxxxxxxxx.cloudfront.net
```

브라우저로 접속:

### 확인 사항

1. **HTTPS로 자동 리다이렉트**
   - `http://dxxx.cloudfront.net` 입력 → HTTPS로 변경됨

2. **Frontend 정상 로드**
   - 루트(`/`)로 접근 시 방명록 화면 표시 (Default root object 동작 확인)

3. **API 호출 동작**
   - `/api/messages` 호출 성공
   - 메시지 작성/조회/삭제 정상

4. **로드밸런싱 확인**
   - 새로고침 시 서버 ID 변경

5. **S3 직접 접근 차단**
   - `https://guestbook-frontend-hojun121.s3.ap-northeast-2.amazonaws.com/index.html`
   - 403 Forbidden 확인 → OAC 동작

### curl 테스트

```bash
# HTTPS로 API 호출
curl https://dxxx.cloudfront.net/api/health

# Frontend
curl https://dxxx.cloudfront.net/

# 여러 번 호출해서 로드밸런싱 확인
for i in {1..10}; do
  curl -s https://dxxx.cloudfront.net/api/health | grep -o '"server":"[^"]*"'
done
```

## Step 8. Invalidation (캐시 무효화)

Frontend 파일 수정 후 바로 반영 안 될 때:

Distribution → Invalidations 탭 → Create invalidation

- Object paths: `/*` (또는 특정 파일 `/index.html`)

월 1,000건까지 무료.

## 체크리스트

- [ ] CloudFront Distribution 생성
- [ ] Default root object를 `index.html`로 설정
- [ ] OAC로 S3 보호
- [ ] S3 Block Public Access 유지 확인 (OAC만 접근)
- [ ] Origin 2개: S3 + ALB
- [ ] Behavior: `/api/*` → ALB (캐싱 비활성)
- [ ] Behavior: Default → S3 (캐싱 활성)
- [ ] HTTPS 자동 리다이렉트
- [ ] CloudFront URL로 전체 기능 동작
- [ ] S3 직접 접근 403 확인

## 최종 아키텍처 확인

```
사용자 (HTTPS)
  ↓
CloudFront (dxxx.cloudfront.net)
  ↓
  ├─ Default (*) ─→ S3 (Frontend, OAC 보호)
  │
  └─ /api/* ──→ ALB (pub-elb)
                 ↓
                Backend EC2 × 2 (pub-svc)
                 ↓
                RDS MySQL (pri-db)
```

축하합니다! 완전한 3-Tier 웹 아키텍처가 완성되었습니다.
