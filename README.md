# 백엔드 CI/CD 구축 실습 (Blue/Green)

이 실습에서는 백엔드 서버를 **Auto Scaling Group + CodeDeploy(Blue/Green)** 기반의 자동 배포 파이프라인으로 직접 구성해 봅니다.

네트워크 · RDS · ALB · S3 · CloudFront 등 기본 인프라는 CloudFormation으로 이미 올라가 있습니다. 여기서는 그 위에 **백엔드 배포 자동화**를 얹어, 코드를 push하면 무중단으로 새 버전이 배포되는 흐름을 완성하는 것이 목표입니다.

```
git push  →  GitHub Actions  →  S3  →  CodeDeploy(Blue/Green)  →  Auto Scaling Group
```

> 진행 환경: AWS 리전 **서울(ap-northeast-2)**, 콘솔 기준.

---

## 1. 시작하기 — 브랜치 받기

먼저 작업 브랜치를 받습니다.

```bash
git fetch origin
git checkout backend-cicd
ls -A
```

폴더 구조는 다음과 같습니다.

```
backend-cicd/
├── 3tier-app-cloudformation-no-ec2.yaml   # 인프라 템플릿 (2단계에서 사용)
├── README.md                              # 이 문서
├── .github/
│   └── workflows/
│       └── backend-deploy.yml             # GitHub Actions 파이프라인
└── backend/                               # 배포되는 백엔드 애플리케이션
    ├── server.js                          # Express 방명록 API
    ├── package.json / package-lock.json
    ├── init.sql                           # DB 스키마 (참고용)
    ├── .env                               # 환경변수 예시 (실제 값은 서버가 직접 생성)
    ├── appspec.yml                        # CodeDeploy 배포 명세
    └── scripts/                           # 배포 단계별 실행 스크립트
        ├── before_install.sh
        ├── after_install.sh
        ├── application_start.sh
        ├── application_stop.sh
        └── validate_service.sh
```

---

## 2. 인프라에서 정적 EC2 걷어내기

현재 인프라에는 백엔드용 EC2 2대(`backend-a`, `backend-c`)가 고정으로 떠 있습니다. 이건 부팅할 때 코드를 한 번 받아오는 단순한 구조라, CI/CD로 전환하려면 먼저 이 EC2를 걷어내야 합니다.

이를 위해 **`3tier-app-cloudformation-no-ec2.yaml`** 로 기존 스택을 업데이트합니다. 이 템플릿은 정적 EC2와 그 IAM 역할, 그리고 타겟 그룹에 고정돼 있던 대상을 빼둔 버전입니다. 비워진 EC2 자리는 이후 단계에서 Auto Scaling Group과 CodeDeploy로 채웁니다.

### 진행

1. **CloudFormation → 기존 스택 선택 → Update**
2. *Prepare template* → **Replace existing template**
3. *Template source* → **Upload a template file** → `3tier-app-cloudformation-no-ec2.yaml` 선택 → **Next**
4. 파라미터는 그대로 두고 **Next → Next**
5. IAM 리소스 생성 동의 체크박스 ☑ → **Submit**
6. `UPDATE_COMPLETE` 가 되면, EC2 콘솔에서 `backend-a` · `backend-c` 가 사라졌는지 확인합니다.

> 이 시점에는 사이트의 `/api/*` 요청이 잠시 503으로 응답합니다. 타겟 그룹이 비어 있기 때문이며, 정상적인 상태입니다. 7단계(ASG)와 10단계(첫 배포)를 마치면 다시 정상으로 돌아옵니다.

---

## 3. Blueprint Overview

본격적으로 만들기 전에, 무엇을 만드는지 한 번 짚고 갑니다.

### 아키텍처

```
사용자 → CloudFront (HTTPS)
   ├ 일반 요청(*) → S3 (정적 프론트엔드)
   └ /api/*       → ALB(:80) → 타겟 그룹 → 백엔드 EC2(:8080) → RDS MySQL(:3306)
                                              ▲
                                  이번 실습에서 만드는 부분
```

### 배포가 일어나는 순서 (Blue/Green)

```
코드 push (backend-cicd 브랜치)
   │
   ▼
GitHub Actions
   │  ① backend/ 폴더를 zip으로 묶음
   │  ② S3 버킷에 업로드
   │  ③ CodeDeploy에 배포 요청
   ▼
CodeDeploy (Blue/Green)
   │  · 지금 돌고 있는 그룹(Blue)을 복제해 새 그룹(Green)을 만든다
   │  · Green에 새 코드를 올리고 헬스체크(/api/health)로 정상 확인
   │  · 정상이면 트래픽을 Green으로 넘긴다
   │  · 기존 Blue는 종료한다
   ▼
무중단 배포 완료
```

핵심은 **새 버전을 별도 서버(Green)에 먼저 띄워 검증한 뒤 트래픽을 넘긴다**는 점입니다. 검증에 실패하면 기존 서버(Blue)가 그대로 서비스를 이어가므로 안전합니다.

### CodeDeploy 파이프라인을 구성하는 파일들

**`backend/appspec.yml`** — CodeDeploy에게 "코드를 어디에 풀고(`/home/ubuntu/backend`), 어떤 순서로 스크립트를 실행할지" 알려주는 명세서입니다. 실행 순서는 다음과 같습니다.

```
ApplicationStop → BeforeInstall → (파일 복사) → AfterInstall → ApplicationStart → ValidateService
```

**`backend/scripts/`** — 위 각 단계에서 실행되는 스크립트입니다.

| 스크립트 | 단계 | 하는 일 |
|---|---|---|
| `application_stop.sh` | ApplicationStop | 돌고 있던 서버 종료 (없으면 그냥 넘어감) |
| `before_install.sh` | BeforeInstall | 배포할 폴더 준비 |
| `after_install.sh` | AfterInstall | `.env` 확인 후 의존성 설치(`npm ci`) |
| `application_start.sh` | ApplicationStart | pm2로 서버 기동 |
| `validate_service.sh` | ValidateService | `/api/health` 로 정상 기동 확인 |

**`.github/workflows/backend-deploy.yml`** — `backend-cicd` 브랜치에 push하면 자동으로 도는 파이프라인입니다. AWS 인증(OIDC) → `backend/` 압축 → S3 업로드 → CodeDeploy 배포 요청 → 완료 대기 순으로 동작합니다.

### 앞으로 만들 것

다음 순서로 하나씩 만들어 갑니다.

- [ ] **S3** 배포 아티팩트 버킷 (4단계)
- [ ] **IAM** 역할 3개 (5단계)
- [ ] **Launch Template** (6단계)
- [ ] **Auto Scaling Group** (7단계)
- [ ] **CodeDeploy** 애플리케이션 + 배포 그룹 (8단계)
- [ ] **GitHub** 저장소 변수 (9단계)

---

## 4. S3 아티팩트 버킷 만들기

CodeDeploy가 가져갈 배포 파일(zip)을 보관할 버킷입니다.

1. **S3 → Create bucket**
2. 이름: 전 세계에서 겹치지 않게 (예: `guestbook-artifacts-<본인계정ID>`)
3. 리전: **ap-northeast-2**
4. *Block all public access* 는 **체크된 채로 그대로** → **Create**

만든 버킷 이름은 메모해 둡니다. 나중에 GitHub 변수 `ARTIFACT_BUCKET` 에 넣습니다.

---

## 5. IAM 역할 3개 만들기

### 5-1. 백엔드 서버용 역할

서버(EC2)에 붙는 역할입니다.

1. **IAM → Roles → Create role**
2. Trusted entity: **AWS service → EC2**
3. 정책 2개 연결
   - `AmazonSSMManagedInstanceCore` — SSM으로 서버 접속
   - `AmazonEC2RoleforAWSCodeDeploy` — CodeDeploy가 S3에서 배포 파일을 받을 수 있게 함
4. 이름: `backend-ec2-role` → 생성

### 5-2. CodeDeploy용 역할

CodeDeploy가 ASG와 로드밸런서를 다룰 수 있게 해주는 역할입니다.

1. **IAM → Roles → Create role**
2. Trusted entity: **AWS service → CodeDeploy → CodeDeploy**
3. 정책 `AWSCodeDeployRole` 이 자동으로 붙습니다.
4. 이름: `codedeploy-service-role` → 생성

### 5-3. GitHub Actions용 역할

GitHub Actions가 비밀 키 없이 AWS에 접근하도록 OIDC 방식으로 연결합니다.

**먼저 OIDC 공급자를 등록합니다.**

1. **IAM → Identity providers → Add provider**
2. Provider type: **OpenID Connect**
3. Provider URL: `https://token.actions.githubusercontent.com` → **Get thumbprint**
4. Audience: `sts.amazonaws.com` → **Add provider**

**이어서 역할을 만듭니다.**

1. **IAM → Roles → Create role → Web identity**
2. Identity provider: 방금 만든 GitHub OIDC, Audience: `sts.amazonaws.com`
3. 생성 후 **Trust relationship** 을 아래처럼 편집합니다. (`<OWNER>/<REPO>` 는 본인 저장소로)

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

4. **Permissions** 에 아래 정책을 인라인으로 추가합니다. (`<ARTIFACT_BUCKET>` 치환)

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

5. 이름: `github-actions-backend-role` → 생성 후 **역할 ARN** 을 메모합니다. (GitHub 변수 `AWS_BACKEND_ROLE_ARN`)

---

## 6. Launch Template 만들기

Auto Scaling Group이 서버를 찍어낼 때 쓰는 틀입니다. 부팅 시 Node.js·pm2·CodeDeploy 에이전트를 설치하고 `.env` 파일만 만들어 둡니다. 실제 애플리케이션 코드는 CodeDeploy가 배포하므로 여기서 받지 않습니다.

1. **EC2 → Launch Templates → Create launch template**
2. 이름: `backend-lt`
3. AMI: **Ubuntu Server 22.04 LTS** (x86_64)
4. Instance type: `t3.micro`
5. Key pair: 없음 (SSM으로 접속)
6. Network settings → **Security groups**: `backend-sg` 선택 (Outputs의 `BackendSgId`)
   - 서브넷은 여기서 지정하지 않습니다. ASG가 정합니다.
7. Advanced details → **IAM instance profile**: `backend-ec2-role`
8. Advanced details → **User data** 에 아래 내용을 입력합니다. (`<RDS_ENDPOINT>` 는 Outputs의 `RdsEndpoint` 값으로 치환)

```bash
#!/bin/bash
set -xe
export DEBIAN_FRONTEND=noninteractive
apt-get update -y

# Node.js 20 + pm2
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs ruby-full wget
npm install -g pm2

# CodeDeploy 에이전트 (서울 리전)
cd /home/ubuntu
wget -q https://aws-codedeploy-ap-northeast-2.s3.ap-northeast-2.amazonaws.com/latest/install
chmod +x ./install
./install auto
systemctl enable codedeploy-agent
systemctl start codedeploy-agent

# 앱 폴더 + .env (CodeDeploy가 여기로 코드를 배포)
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

## 7. Auto Scaling Group 만들기

1. **EC2 → Auto Scaling Groups → Create**
2. 이름: `backend-asg`, Launch template: `backend-lt` → Next
3. Network: VPC(`VpcId`), 서브넷 **`pub-svc-a`, `pub-svc-c`** (`BackendSubnets`) → Next
4. **Load balancing**: *Attach to an existing load balancer* → *Choose from your load balancer target groups* → **`tg-...`** (`TargetGroupName`)
5. **Health checks**: **EC2** 선택, grace period `300`
6. Group size: Desired `2`, Min `2`, Max `4` → 생성

> 헬스체크를 ELB가 아닌 **EC2** 로 두는 이유가 있습니다. 첫 배포 전에는 서버에 아직 앱이 없어 `/api/health` 가 실패하는데, ELB 헬스체크로 두면 ASG가 이를 "고장"으로 보고 서버를 계속 새로 띄웠다 지웠다 반복합니다. EC2 헬스체크는 "서버가 켜져 있는지"만 보므로, 첫 배포 전까지 안정적입니다. (Blue/Green 전환 자체는 CodeDeploy가 타겟 그룹 상태로 판단하므로 영향받지 않습니다.)

서버 2대가 뜨고 위 User data 가 실행됩니다. 아직 앱이 없어서 타겟 그룹에는 unhealthy로 보이는데, 첫 배포 전까지는 정상입니다.

---

## 8. CodeDeploy 만들기 (Blue/Green)

1. **CodeDeploy → Applications → Create application**
   - 이름: `guestbook-backend`, Compute platform: **EC2/On-premises** → 생성
2. **Create deployment group**
   - 이름: `guestbook-backend-dg`
   - Service role: `codedeploy-service-role` (5-2)
   - Deployment type: **Blue/green**
   - Environment configuration: **Automatically copy Amazon EC2 Auto Scaling group** → `backend-asg`
   - Load balancer: **Enable** → Application Load Balancer → Target group: **`tg-...`**
   - Deployment settings
     - *Traffic rerouting*: **Reroute traffic immediately**
     - *Original instances*: 기존 서버를 일정 시간 뒤 종료 (예: 0~5분)
     - Deployment configuration: `CodeDeployDefault.AllAtOnce`
   - **Create deployment group**

애플리케이션 이름(`guestbook-backend`)과 배포 그룹 이름(`guestbook-backend-dg`)을 메모해 둡니다.

---

## 9. GitHub 저장소 변수 등록

저장소 **Settings → Secrets and variables → Actions → Variables** 탭에서 아래 5개를 등록합니다.

| 변수명 | 값 |
|---|---|
| `AWS_REGION` | `ap-northeast-2` |
| `AWS_BACKEND_ROLE_ARN` | 5-3에서 만든 역할 ARN |
| `ARTIFACT_BUCKET` | 4단계 버킷 이름 |
| `CD_APP_NAME` | `guestbook-backend` |
| `CD_DG_NAME` | `guestbook-backend-dg` |

---

## 10. 첫 배포

두 가지 방법 중 하나로 파이프라인을 실행합니다.

- `backend-cicd` 브랜치에 커밋을 push, 또는
- **GitHub → Actions → Backend Deploy → Run workflow** (수동 실행)

진행 상황은 이렇게 따라가며 확인합니다.

- **GitHub Actions** 탭: 압축 → 업로드 → 배포 요청 → 완료 대기
- **CodeDeploy 콘솔**: Green 그룹 생성 → 배포 → 트래픽 전환 → Blue 종료
- **EC2 → Target Groups**: `tg-...` 가 **healthy** 로 바뀌는지 확인

---

## 11. 확인하기

1. Outputs의 **`CloudFrontURL`** 로 접속하면 방명록 페이지가 뜹니다.
2. 메시지를 남기고 새로고침했을 때 그대로 보이면 DB 저장이 정상입니다.
3. 새로고침을 반복했을 때 응답의 `server` 값(서버 이름)이 번갈아 바뀌면 로드밸런싱이 잘 동작하는 것입니다.
4. **직접 바꿔서 다시 배포해 보기**
   `backend/server.js` 의 `/api/health` 응답에 버전 표시 한 줄을 추가합니다.

   ```js
   app.get('/api/health', (req, res) => {
     res.status(200).json({
       status: 'ok',
       version: 'v2',        // ← 이 줄 추가
       server: SERVER_ID,
       ip: SERVER_IP,
       timestamp: new Date().toISOString(),
     });
   });
   ```

   저장 후 `backend-cicd` 브랜치에 push하면 파이프라인이 다시 돕니다. 배포가 끝난 뒤 브라우저에서 `https://<CloudFrontURL>/api/health` 에 접속하면 응답에 `version` 이 새로 나타납니다.
   - 배포 전: `{"status":"ok","server":...}`
   - 배포 후: `{"status":"ok","version":"v2","server":...}`

   이 `version` 필드가 새로 보이면 바꾼 코드가 CI/CD로 배포된 것입니다. Blue/Green으로 전환되는 동안에도 사이트는 끊김 없이 계속 동작합니다.

---

## 실습 후 정리

비용이 계속 나가지 않도록, 실습이 끝나면 만든 리소스를 아래 순서대로 삭제합니다. (NAT Gateway 2개, RDS, ALB, CloudFront, EC2가 모두 과금 대상입니다.)

콘솔에서 직접 만든 리소스를 먼저 지우고, **CloudFormation 스택을 가장 마지막에** 삭제하는 것이 핵심입니다. 순서가 꼬이면 스택 삭제가 실패하거나 리소스가 남을 수 있습니다.

1. **Auto Scaling Group 삭제**
   EC2 → Auto Scaling Groups 에서 `backend-asg` 와, CodeDeploy가 배포 과정에서 만든 `CodeDeploy_backend-asg_...` 를 **모두** 삭제합니다. ASG를 지우면 그 안의 서버(EC2)도 함께 종료됩니다.

2. **Launch Template 삭제**
   EC2 → Launch Templates 에서 `backend-lt` 를 삭제합니다.

3. **CodeDeploy 삭제**
   CodeDeploy → Applications 에서 `guestbook-backend` 를 삭제합니다. 배포 그룹 `guestbook-backend-dg` 도 함께 사라집니다.

4. **S3 아티팩트 버킷 삭제**
   S3 에서 `guestbook-artifacts-...` 버킷을 **Empty(비우기)** 한 뒤 **Delete** 합니다. 버킷은 비어 있어야 삭제됩니다.

5. **IAM 정리**
   IAM → Roles 에서 `backend-ec2-role`, `codedeploy-service-role`, `github-actions-backend-role` 을 삭제하고, IAM → Identity providers 에서 GitHub OIDC 공급자를 삭제합니다.

6. **(선택) GitHub 변수 삭제**
   저장소 Settings → Secrets and variables → Actions → Variables 에서 등록했던 5개 변수를 삭제합니다.

7. **CloudFormation 스택 삭제**
   마지막으로 스택을 **Delete** 하면 VPC · NAT · RDS · ALB · 프론트엔드 S3 · CloudFront 등 나머지 인프라가 한 번에 제거됩니다.

> ⚠️ 7번(스택 삭제)을 먼저 하지 마세요. ASG 서버가 스택의 타겟 그룹에 붙어 있는 상태에서 스택을 지우면 삭제가 막히거나 리소스가 남을 수 있습니다. 콘솔에서 직접 만든 1~5번을 먼저 정리해야 합니다.
