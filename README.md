# Frontend CI/CD: S3 + CloudFront + GitHub Actions

정적 파일을 S3 에 호스팅하고 CloudFront 로 배포한 뒤, GitHub Actions 로 자동화하는 환경 구축 가이드.

## 아키텍처

```
[Developer] ── git push (main) ──▶ [GitHub Actions]
                                       │
                         ┌─────────────┴────────────┐
                         │ 1. OIDC → IAM Role assume │
                         │ 2. aws s3 sync            │
                         │ 3. cloudfront invalidate  │
                         └─────────────┬────────────┘
                                       ▼
[End User] ──HTTPS──▶ [CloudFront] ──(OAC)──▶ [S3 (private)]
```

- **S3**: 원본 저장. 퍼블릭 접근 완전 차단 (OAC 로만 읽힘).
- **CloudFront**: HTTPS 종단 + 글로벌 캐시.
- **OAC (Origin Access Control)**: 최신 표준. 구식 OAI 대체.
- **OIDC**: 장기 access key 없이 GitHub Actions 가 IAM Role assume.

## 사전 조건

- AWS 계정 (Admin 권한)
- AWS CLI 로컬 설정 완료 (`aws configure`)
- GitHub 리포지토리 + 정적 파일 (`index.html` 포함)
- 리전: `ap-northeast-2` (서울)

---

## Step 1. S3 버킷 생성

버킷명은 글로벌 유니크.

```bash
export BUCKET=hoya-frontend-prac-2026   # ← 본인 값으로 변경
export REGION=ap-northeast-2

aws s3api create-bucket \
  --bucket $BUCKET \
  --region $REGION \
  --create-bucket-configuration LocationConstraint=$REGION
```

퍼블릭 접근 차단 (기본값이지만 명시):

```bash
aws s3api put-public-access-block \
  --bucket $BUCKET \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

> ⚠️ S3 **정적 웹사이트 호스팅은 활성화하지 않습니다**. CloudFront + OAC 패턴에서는 불필요하며 보안 표면을 늘립니다.

## Step 2. 초기 정적 파일 업로드

```bash
aws s3 sync . s3://$BUCKET \
  --exclude ".git/*" \
  --exclude ".github/*" \
  --exclude "README.md"
```

이 시점에는 CloudFront 가 없어 외부 접근 불가가 정상입니다.

## Step 3. CloudFront 배포 생성 (콘솔)

**CloudFront → Create distribution**:

| 항목 | 값 |
|---|---|
| Origin domain | S3 버킷 선택 |
| Origin access | **Origin access control settings (recommended)** |
| OAC | **Create new OAC** (기본 설정 OK) |
| Viewer protocol policy | **Redirect HTTP to HTTPS** |
| Allowed HTTP methods | GET, HEAD |
| Default root object | `index.html` |
| WAF | Do not enable (실습 기준) |
| Price class | 선호에 따라 (실습은 NA + EU 만으로 충분) |

생성 후 상단의 **"Copy policy"** 버튼 → 클립보드에 S3 버킷 정책 JSON 복사됨.

## Step 4. S3 버킷 정책 적용

복사한 정책을 `bucket-policy.json` 으로 저장 후:

```bash
aws s3api put-bucket-policy \
  --bucket $BUCKET \
  --policy file://bucket-policy.json
```

정책 형태:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipal",
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::BUCKET/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT_ID:distribution/DIST_ID"
      }
    }
  }]
}
```

## Step 5. 동작 확인

CloudFront 상태가 **Deployed** (5–10 분) 되면 배포 도메인으로 접근:

```
https://dXXXXXXXXX.cloudfront.net
```

`index.html` 이 뜨면 수동 배포 단계 완료. 여기서부터 자동화.

---

## Step 6. GitHub OIDC Provider 등록 (계정당 1회)

기존 등록 여부 확인:
```bash
aws iam list-open-id-connect-providers
```

없으면 등록:
```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

## Step 7. IAM Role 생성

**`trust-policy.json`** — 특정 리포·브랜치만 assume 가능:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:OWNER/REPO:ref:refs/heads/main"
      }
    }
  }]
}
```

**`permissions-policy.json`** — 최소 권한:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::BUCKET_NAME"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:DeleteObject", "s3:GetObject"],
      "Resource": "arn:aws:s3:::BUCKET_NAME/*"
    },
    {
      "Effect": "Allow",
      "Action": "cloudfront:CreateInvalidation",
      "Resource": "arn:aws:cloudfront::ACCOUNT_ID:distribution/DIST_ID"
    }
  ]
}
```

생성:
```bash
aws iam create-role \
  --role-name github-actions-frontend-deploy \
  --assume-role-policy-document file://trust-policy.json

aws iam put-role-policy \
  --role-name github-actions-frontend-deploy \
  --policy-name frontend-deploy \
  --policy-document file://permissions-policy.json
```

생성된 Role ARN 기록 (다음 Step 에서 사용):
```bash
aws iam get-role --role-name github-actions-frontend-deploy --query 'Role.Arn' --output text
```

## Step 8. GitHub Actions 워크플로우 추가

리포지토리에 `.github/workflows/deploy.yml` 추가:

```yaml
name: Deploy Frontend to S3 + CloudFront

on:
  push:
    branches: [your branch]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

env:
  AWS_REGION: ap-northeast-2
  S3_BUCKET: your-bucket-name              # ← 변경
  CLOUDFRONT_DISTRIBUTION_ID: EXXXXXXXXXXX # ← 변경
  AWS_ROLE_ARN: arn:aws:iam::123456789012:role/github-actions-frontend-deploy  # ← 변경

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Sync to S3
        run: |
          aws s3 sync . s3://${{ env.S3_BUCKET }} \
            --delete \
            --exclude ".git/*" \
            --exclude ".github/*" \
            --exclude "README.md" \
            --exclude ".gitignore"

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ env.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

## Step 9. 첫 자동 배포

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: add S3 + CloudFront deploy workflow"
git push origin main
```

GitHub 리포 → **Actions** 탭에서 실행 결과 확인.

성공하면 CloudFront URL 새로고침 (invalidation 적용까지 1–3 분).

---

## 채워야 할 값 체크리스트

| 위치 | 변수 | 예시 |
|---|---|---|
| Step 1 | `BUCKET` | `hoya-frontend-prac-2026` |
| Step 4 정책 | `BUCKET`, `ACCOUNT_ID`, `DIST_ID` | — |
| Step 7 trust | `ACCOUNT_ID`, `OWNER/REPO` | `123456789012`, `hoya/devops-3-tier-practice` |
| Step 7 perms | `BUCKET_NAME`, `ACCOUNT_ID`, `DIST_ID` | — |
| Step 8 workflow | `S3_BUCKET`, `CLOUDFRONT_DISTRIBUTION_ID`, `AWS_ROLE_ARN` | — |

## Troubleshooting

**`Error: Could not assume role with OIDC`**
- Trust policy `sub` 조건 점검. 형식: `repo:OWNER/REPO:ref:refs/heads/main`
- Organization 리포면 `OWNER` 가 org 이름

**`AccessDenied` (S3)**
- IAM 권한 정책의 Resource 가 `arn:aws:s3:::BUCKET` 과 `arn:aws:s3:::BUCKET/*` 두 개 모두 있는지 확인

**CloudFront 가 옛날 콘텐츠 반환**
- Invalidation 적용 1–3 분 대기
- 브라우저 강제 새로고침 (Cmd+Shift+R / Ctrl+F5)

**403 Forbidden (CloudFront)**
- S3 버킷 정책에 OAC 권한 누락 → Step 4 재확인
- CloudFront Default root object 가 `index.html` 인지

**`InvalidViewerCertificate` 또는 HTTPS 오류**
- 기본 CloudFront 도메인 (`*.cloudfront.net`) 은 자동 HTTPS. 커스텀 도메인 쓰려면 ACM 인증서가 **us-east-1 리전**에 있어야 함

## 비용 메모 (실습 기준)

| 항목 | 비용 |
|---|---|
| S3 저장 | GB 당 월 $0.025 (서울) |
| CloudFront 전송 | 월 1TB 무료, 이후 GB 당 약 $0.085 |
| Invalidation | 월 1000 paths 무료, 이후 path 당 $0.005 |

`/*` 도 path 1 개로 카운트됨. 매일 배포해도 1000 회 안 넘김.

## 다음 단계 (선택)

- **커스텀 도메인**: Route 53 + ACM (us-east-1) → CloudFront alias
- **SPA 라우팅**: CloudFront Error pages → 404 를 `/index.html` 200 응답으로 매핑 (React Router 등)
- **빌드 단계 추가**: 프레임워크 도입 시 workflow 에 `npm ci && npm run build` 추가, sync 대상을 `dist/` 또는 `build/` 로 변경
- **인프라 IaC 화**: S3 + CloudFront + OAC + IAM Role 까지 CloudFormation 으로 코드화