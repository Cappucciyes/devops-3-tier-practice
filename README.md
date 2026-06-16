# Backend CI/CD 실습 (backend-cicd 브랜치)

이 브랜치는 **백엔드 EC2 계층을 Auto Scaling Group + CodeDeploy(Blue/Green) 기반 CI/CD로 직접 구성**하는 실습용입니다.
네트워크 / RDS / ALB / S3 / CloudFront 같은 나머지 인프라는 이미 CloudFormation으로 떠 있는 상태에서, **EC2 계층만** 콘솔에서 손으로 만들어 봅니다.

> **GitHub Actions → S3 → CodeDeploy(Blue/Green) → Auto Scaling Group** 파이프라인을 완성하는 것이 목표입니다.

---

## 사전 조건

- **강사가 `cloudformation` 브랜치 템플릿으로 스택을 이미 배포**해 둔 상태 (`CREATE_COMPLETE`)
- 리전: **서울(ap-northeast-2)**
- 본인이 이 저장소(`devops-3-tier-practice`)에 push 권한이 있고, `backend-cicd` 브랜치를 사용

---

## Step 1 — 브랜치 체크아웃 & 구조 확인

```bash
git fetch origin
git checkout backend-cicd
ls -A
```

이 브랜치의 구조는 다음과 같습니다. (자세한 설명은 Step 3)

```
backend-cicd/
├── 3tier-app-cloudformation-no-ec2.yaml   # 정적 EC2를 뺀 인프라 템플릿 (Step 2에서 사용)
├── README.md                              # 이 가이드
├── .github/
│   └── workflows/
│       └── backend-deploy.yml             # GitHub Actions 파이프라인
└── backend/                               # 배포 대상 백엔드 애플리케이션
    ├── server.js                          # Express 앱 (방명록 API)
    ├── package.json / package-lock.json
    ├── init.sql                           # DB 스키마 (참고용, 실제 초기화는 CF가 수행)
    ├── .env                               # 환경변수 placeholder (실제 값은 인스턴스가 생성)
    ├── appspec.yml                        # CodeDeploy 배포 명세
    └── scripts/                           # CodeDeploy 생명주기 훅 스크립트
        ├── before_install.sh
        ├── after_install.sh
        ├── application_start.sh
        ├── application_stop.sh
        └── validate_service.sh
```

---

## Step 2 — CloudFormation 업데이트 배포 (정적 EC2 제거)

강사가 배포한 스택에는 **정적 EC2 2대(`backend-a`, `backend-c`)** 가 떠 있습니다.
CI/CD 실습을 위해, 이 EC2를 **`3tier-app-cloudformation-no-ec2.yaml` 로 스택을 업데이트**해서 제거합니다.

이 템플릿은 원본에서 다음을 뺀 버전입니다.
- 정적 EC2 `BackendA` / `BackendC`
- EC2용 IAM `Ec2Role` / `Ec2Profile`
- TargetGroup에 하드코딩돼 있던 정적 타겟(`Targets`)

> 즉 **"EC2 계층만 비워서, 그 자리를 학생이 ASG+CodeDeploy로 채우게"** 만든 템플릿입니다.

### 콘솔 절차

1. **CloudFormation → 기존 스택 선택 → Update**
2. *Prepare template* = **Replace existing template**
3. *Template source* = **Upload a template file** → `3tier-app-cloudformation-no-ec2.yaml` 첨부 → **Next**
4. 파라미터는 기존 값 그대로 → **Next** → **Next**
5. IAM 동의 체크박스 ☑ → **Submit**
6. `UPDATE_COMPLETE` 확인 → **EC2 콘솔에서 `backend-a`/`backend-c`가 사라졌는지 확인**

> ⚠️ **이 시점부터 `/api/*` 요청은 503입니다 (정상).** TargetGroup이 비었기 때문이며, Step 7(ASG) + Step 10(첫 배포) 이후 healthy로 복구됩니다.

### 이후 단계에서 쓸 값 — 스택 **Outputs 탭**에서 확인

| Output 키 | 용도 |
|---|---|
| `RdsEndpoint` | Launch Template `.env`의 `DB_HOST` |
| `VpcId` | ASG / Launch Template 생성 |
| `BackendSubnets` | ASG에 지정할 서브넷 (`pub-svc-a,pub-svc-c`) |
| `BackendSgId` | Launch Template 보안그룹 (`backend-sg`) |
| `TargetGroupName` | ASG / CodeDeploy에 연결할 ALB 타겟그룹 |
| `CloudFrontURL` | 최종 검증 접속 주소 |

---

## Step 3 — 전체 청사진

### 3-1. 아키텍처

```
사용자 → CloudFront (HTTPS)
   ├ Default(*) → S3 (프론트, OAC 보호)
   └ /api/*    → ALB(:80) → TargetGroup → [ASG 백엔드 EC2(:8080)] → RDS MySQL(:3306)
                                              ▲
                                     이 부분을 학생이 만든다
```

### 3-2. CI/CD 흐름 (Blue/Green)

```
git push (backend-cicd)
   │
   ▼
GitHub Actions (backend-deploy.yml)
   │  1) backend/ 를 zip 으로 패키징 (appspec.yml 이 최상단)
   │  2) S3 아티팩트 버킷에 업로드
   │  3) CodeDeploy 배포 생성(create-deployment)
   ▼
CodeDeploy (Blue/Green)
   │  · 현재 ASG(Blue)를 복제해 새 ASG(Green) 생성
   │  · Green 인스턴스에 번들 배포 → 훅 실행 → /api/health 검증
   │  · 검증 통과 시 ALB 트래픽을 Green 으로 전환
   │  · Blue(구버전) 인스턴스 종료
   ▼
무중단 배포 완료
```

**Blue/Green 핵심**: 새 버전을 **별도 인스턴스(Green)** 에 먼저 띄우고 헬스체크가 통과해야 트래픽을 넘깁니다. 실패하면 기존(Blue)이 그대로 서비스 → 무중단/안전 롤백.

### 3-3. 폴더 구조 & 역할

- **`3tier-app-cloudformation-no-ec2.yaml`** — Step 2에서 스택 업데이트에 쓰는, EC2를 뺀 인프라 템플릿.
- **`backend/`** — 실제로 배포되는 애플리케이션. GitHub Actions가 **이 디렉토리만** zip으로 묶어 CodeDeploy에 넘깁니다.
- **`.github/workflows/backend-deploy.yml`** — CI/CD 파이프라인 정의.

### 3-4. 핵심 파일 간단 설명

**`backend/appspec.yml`** — CodeDeploy에게 *"번들을 어디에 풀고(`/home/ubuntu/backend`), 어떤 순서로 훅을 실행할지"* 알려주는 명세서입니다. 훅 실행 순서:

```
ApplicationStop → BeforeInstall → (파일 복사) → AfterInstall → ApplicationStart → ValidateService
```

**`backend/scripts/`** — 위 각 훅에서 실행되는 스크립트입니다.

| 스크립트 | 훅 | 하는 일 |
|---|---|---|
| `application_stop.sh` | ApplicationStop | 기존 pm2 프로세스 종료 (없으면 무시) |
| `before_install.sh` | BeforeInstall | 배포 디렉토리 준비 |
| `after_install.sh` | AfterInstall | `.env` 존재 확인 후 `npm ci`(프로덕션 의존성 설치) |
| `application_start.sh` | ApplicationStart | `pm2 start/reload` 로 서버 기동 |
| `validate_service.sh` | ValidateService | `/api/health` 헬스체크 (앱 정상 기동 확인) |

**`.github/workflows/backend-deploy.yml`** — `backend-cicd` 브랜치에 push되면 실행됩니다. OIDC로 AWS 인증 → `backend/`를 zip 패키징(appspec.yml이 zip 최상단에 오도록) → S3 업로드 → `aws deploy create-deployment` 호출 → 배포 완료까지 대기.

### 3-5. 학생이 콘솔에서 만들 것 (체크리스트)

- [ ] **S3** 아티팩트 버킷 (Step 4)
- [ ] **IAM** 3종: EC2 인스턴스 롤 / CodeDeploy 서비스 롤 / GitHub OIDC 롤 (Step 5)
- [ ] **Launch Template** (Step 6)
- [ ] **Auto Scaling Group** (Step 7)
- [ ] **CodeDeploy** Application + Deployment Group (Step 8)
- [ ] **GitHub** repo Variables (Step 9)

---

## Step 4 — S3 아티팩트 버킷

CodeDeploy 번들(zip)을 보관할 버킷입니다.

1. **S3 → Create bucket**
2. 이름: 전역 고유 (예: `guestbook-artifacts-<본인계정ID>`)
3. 리전: **ap-northeast-2**
4. *Block all public access* **유지(체크됨)** → **Create**
5. 버킷 이름을 메모 → 나중에 GitHub 변수 `ARTIFACT_BUCKET`

---

## Step 5 — IAM 3종

### 5-1. EC2 인스턴스 롤 (백엔드 인스턴스용)

1. **IAM → Roles → Create role**
2. Trusted entity: **AWS service → EC2**
3. 정책 2개 연결:
   - `AmazonSSMManagedInstanceCore` (SSM 접속)
   - `AmazonEC2RoleforAWSCodeDeploy` (CodeDeploy agent가 S3에서 번들 다운로드)
4. 이름: `backend-ec2-role` → 생성 (동명의 인스턴스 프로파일이 자동 생성됨)

### 5-2. CodeDeploy 서비스 롤

1. **IAM → Roles → Create role**
2. Trusted entity: **AWS service → CodeDeploy → CodeDeploy** (use case)
3. 정책: `AWSCodeDeployRole` (자동 선택됨)
4. 이름: `codedeploy-service-role` → 생성

### 5-3. GitHub OIDC 공급자 + 롤

**(a) OIDC 공급자 등록**
1. **IAM → Identity providers → Add provider**
2. Provider type: **OpenID Connect**
3. Provider URL: `https://token.actions.githubusercontent.com` → **Get thumbprint**
4. Audience: `sts.amazonaws.com` → **Add provider**

**(b) 롤 생성**
1. **IAM → Roles → Create role → Web identity**
2. Identity provider: 방금 만든 GitHub OIDC, Audience: `sts.amazonaws.com`
3. 생성 후 **Trust relationship**를 아래로 편집 (`<OWNER>/<REPO>`는 본인 저장소로):

```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
    "StringLike": { "token.actions.githubusercontent.com:sub": "repo:<OWNER>/<REPO>:ref:refs/heads/backend-cicd" }
  }
}
```

4. **Permissions**에 아래 인라인 정책 추가 (`<ARTIFACT_BUCKET>` 치환):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["s3:PutObject"], "Resource": "arn:aws:s3:::<ARTIFACT_BUCKET>/*" },
    { "Effect": "Allow",
      "Action": ["codedeploy:CreateDeployment","codedeploy:GetDeployment",
                 "codedeploy:GetDeploymentConfig","codedeploy:RegisterApplicationRevision",
                 "codedeploy:GetApplicationRevision"],
      "Resource": "*" }
  ]
}
```

5. 이름: `github-actions-backend-role` → 생성 후 **롤 ARN 메모** → GitHub 변수 `AWS_BACKEND_ROLE_ARN`

---

## Step 6 — Launch Template

ASG가 인스턴스를 찍어낼 틀입니다. userdata는 **앱을 받지 않고**, node/pm2/**CodeDeploy agent** 설치 + `.env` 생성까지만 합니다. 실제 앱은 CodeDeploy가 넣습니다.

1. **EC2 → Launch Templates → Create launch template**
2. 이름: `backend-lt`
3. AMI: **Ubuntu Server 22.04 LTS** (x86_64)
4. Instance type: `t3.micro`
5. Key pair: 없음 (SSM으로 접속)
6. Network settings → **Security groups**: `backend-sg` 선택 (Step 2 Output `BackendSgId`)
   - ⚠️ 서브넷은 **여기서 지정하지 말 것** (ASG가 지정)
7. Advanced details → **IAM instance profile**: `backend-ec2-role`
8. Advanced details → **User data**에 아래 입력 (`<RDS_ENDPOINT>`를 Step 2 Output `RdsEndpoint`로 치환):

```bash
#!/bin/bash
set -xe
export DEBIAN_FRONTEND=noninteractive
apt-get update -y

# Node.js 20 + pm2
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs ruby-full wget
npm install -g pm2

# CodeDeploy agent (서울 리전)
cd /home/ubuntu
wget -q https://aws-codedeploy-ap-northeast-2.s3.ap-northeast-2.amazonaws.com/latest/install
chmod +x ./install
./install auto
systemctl enable codedeploy-agent
systemctl start codedeploy-agent

# 앱 디렉토리 + .env (CodeDeploy 가 코드를 여기에 배포)
mkdir -p /home/ubuntu/backend
cat > /home/ubuntu/backend/.env <<EOF
PORT=8080
DB_HOST=<RDS_ENDPOINT>
DB_PORT=3306
DB_USER=admin
DB_PASSWORD=devops123!
DB_NAME=guestbook
EOF
chown -R ubuntu:ubuntu /home/ubuntu/backend
```

9. **Create launch template**

---

## Step 7 — Auto Scaling Group

1. **EC2 → Auto Scaling Groups → Create**
2. 이름: `backend-asg`, Launch template: `backend-lt` → Next
3. Network: VPC(`VpcId`), 서브넷 **`pub-svc-a`, `pub-svc-c`** (Output `BackendSubnets`) → Next
4. **Load balancing**: *Attach to an existing load balancer* → *Choose from your load balancer target groups* → **`tg-<스택명>`** (Output `TargetGroupName`)
5. **Health checks**: **EC2** 선택 (⚠️ 이유는 아래), grace period `300`
6. Group size: Desired `2`, Min `2`, Max `4` → 생성

> ⚠️ **Health check를 ELB가 아니라 EC2로 두는 이유**: 첫 배포 전에는 인스턴스에 앱이 없어 `/api/health`가 실패합니다. ELB health check면 ASG가 "비정상"으로 보고 인스턴스를 계속 재생성(churn)합니다. EC2 health check면 "인스턴스가 살아있음"만 보므로 첫 배포 전까지 안정적입니다. (Blue/Green 전환 자체는 CodeDeploy가 타겟그룹 헬스로 판단하므로 무관)

생성 후 인스턴스 2대가 뜨고 userdata가 실행됩니다. 이 시점엔 앱이 없어 TargetGroup에서 **unhealthy**로 보이는 게 정상입니다.

---

## Step 8 — CodeDeploy Application + Deployment Group (Blue/Green)

1. **CodeDeploy → Applications → Create application**
   - 이름: `guestbook-backend`, Compute platform: **EC2/On-premises** → 생성
2. **Create deployment group**
   - 이름: `guestbook-backend-dg`
   - Service role: `codedeploy-service-role` (Step 5-2)
   - Deployment type: **Blue/green**
   - Environment configuration: **Automatically copy Amazon EC2 Auto Scaling group** → `backend-asg`
   - Load balancer: **Enable** → Application Load Balancer → Target group: **`tg-<스택명>`**
   - Deployment settings:
     - *Traffic rerouting*: **Reroute traffic immediately**
     - *Original instances*: **Terminate the original instances ... (예: 0~5분 후)**
     - Deployment configuration: `CodeDeployDefault.AllAtOnce`
   - → **Create deployment group**
3. 이름 메모: `guestbook-backend`(=`CD_APP_NAME`), `guestbook-backend-dg`(=`CD_DG_NAME`)

---

## Step 9 — GitHub repo Variables

저장소 **Settings → Secrets and variables → Actions → Variables** 탭 → 아래 5개 등록:

| 변수명 | 값 |
|---|---|
| `AWS_REGION` | `ap-northeast-2` |
| `AWS_BACKEND_ROLE_ARN` | Step 5-3 OIDC 롤 ARN |
| `ARTIFACT_BUCKET` | Step 4 버킷 이름 |
| `CD_APP_NAME` | `guestbook-backend` |
| `CD_DG_NAME` | `guestbook-backend-dg` |

---

## Step 10 — 첫 배포

방법 A: `backend-cicd` 브랜치에 커밋 push
방법 B: **GitHub → Actions → Backend Deploy → Run workflow** (수동 트리거)

- **GitHub Actions** 탭에서 진행 확인: Package → Upload → Trigger CodeDeploy → Wait
- **CodeDeploy 콘솔**에서 Blue/Green 진행 확인: Green ASG 생성 → 배포 → 트래픽 전환 → Blue 종료
- **EC2 → Target Groups → `tg-...`** 가 **healthy** 로 바뀌는지 확인

---

## Step 11 — 검증

1. Step 2 Output **`CloudFrontURL`** 접속 → 방명록 페이지 로드
2. 메시지 작성 → 새로고침 후 유지되면 **RDS 저장 정상**
3. 새로고침 반복 시 응답의 `server` 값(호스트명)이 바뀌면 **로드밸런싱 정상**
4. (선택) CodeDeploy에서 한 번 더 배포 → 무중단으로 새 버전 반영되는지 확인

---

## 정리 / 주의

- 실습 후 **반드시 삭제**: CodeDeploy로 생성된 ASG → Launch Template → CodeDeploy App → S3 아티팩트 버킷 → 마지막에 CloudFormation 스택 삭제
- 비용 주의: NAT Gateway×2, RDS, ALB, CloudFront, EC2가 과금됩니다.
- ⚠️ CodeDeploy Blue/Green이 만든 ASG(`CodeDeploy_backend-asg_...`)는 스택 외부 리소스라 **CloudFormation 삭제만으로 안 지워질 수 있습니다.** ASG/인스턴스를 먼저 수동 정리하세요.

---

## 트러블슈팅

| 증상 | 원인 / 확인 |
|---|---|
| 배포가 `AfterInstall`에서 실패 | 인스턴스 `/home/ubuntu/backend/.env` 없음 → Launch Template userdata 확인 |
| CodeDeploy가 인스턴스를 못 찾음 | codedeploy-agent 미설치/미실행 → userdata, `systemctl status codedeploy-agent` |
| 번들 다운로드 권한 오류 | EC2 롤에 `AmazonEC2RoleforAWSCodeDeploy` 누락 |
| GitHub Actions 인증 실패 | OIDC 롤 trust의 `sub`(브랜치/repo) 불일치 |
| ASG 인스턴스가 계속 교체됨 | ASG health check가 ELB로 설정됨 → **EC2로 변경** |
| `/api/*` 가 계속 503 | TargetGroup에 healthy 타겟 없음 → 첫 배포 완료/헬스 확인 |
