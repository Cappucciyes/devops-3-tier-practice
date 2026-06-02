# 실습 5. Frontend S3 배포

## 목표

- S3 버킷 생성
- Frontend 파일 업로드
- (테스트/공개는 다음 CloudFront 실습에서 진행)

## Step 1. S3 버킷 생성

S3 콘솔 → Create bucket

**일반 구성**
- 버킷 이름: `guestbook-frontend-<본인식별자>` (글로벌 유일!)
  - 예: `guestbook-frontend-hojun121`
- 리전: 아시아 태평양 (서울) ap-northeast-2

**객체 소유권**
- ACLs disabled (권장)

**퍼블릭 액세스 차단**
- 모두 **체크 유지** (권장)
- 버킷을 직접 공개하지 않고, 다음 실습에서 CloudFront(OAC)로만 접근시킬 예정

**나머지**
- 기본값 유지

버킷 생성.

## Step 2. Frontend 파일 업로드

S3 버킷 상세 → Upload → Add files

업로드할 파일:
- `index.html`
- `app.js`
- `style.css`

Upload 클릭 후 업로드 완료 확인.

## 체크리스트

- [ ] app.js의 API_BASE가 ALB DNS로 설정
- [ ] S3 버킷 이름이 글로벌 유일
- [ ] 퍼블릭 액세스 차단 유지 (CloudFront 접근 예정)
- [ ] index.html, app.js, style.css 업로드 완료