# DevOps 3-Tier Practice

AWS 3-Tier 아키텍처 실습용 방명록 애플리케이션

<img width="1263" height="538" alt="image" src="https://github.com/user-attachments/assets/1d7a61f2-9982-4af4-89d9-957ce8e80713" />


## 아키텍처

```
사용자
  ↓
CloudFront (CDN)
  ↓              ↓
S3 (Frontend)   ALB
                 ↓
               EC2 × 2 (Backend, pub-svc)
                 ↓
               RDS MySQL (pri-db)

```

## 기술 스택

### Frontend
- 순수 HTML + CSS + JavaScript (빌드 없음)
- S3 정적 웹 호스팅

### Backend
- Node.js 20 + Express
- MySQL2 (DB 드라이버)
- PM2 (프로세스 매니저)

### Database
- AWS RDS MySQL 8.0

## 디렉토리 구조

```
devops-3-tier-practice/
├── README.md
├── 3tier-app-cloudformation.yaml   # 전체 3-Tier 자동 배포 템플릿
├── frontend/
│   ├── index.html
│   ├── app.js
│   └── style.css
├── backend/
│   ├── server.js
│   ├── package.json
│   ├── init.sql
│   ├── start.sh
│   └── .env
└── docs/
    ├── 01-vpc-subnet-setup.md
    ├── 02-backend-ec2-setup.md
    ├── 03-rds-setup.md
    ├── 04-alb-setup.md
    ├── 05-frontend-s3-setup.md
    ├── 06-cloudfront-setup.md
    └── cheat-vpc-cloudformation.yaml
```

---

# CloudFormation 자동 배포

`docs/`의 수동 단계(VPC → EC2 → RDS → ALB → S3 → CloudFront)를 **한 개의 스택으로 자동화**한 템플릿입니다.
파일 하나만 업로드하면 전체 3-Tier 아키텍처가 통째로 생성됩니다.

📄 **템플릿 파일: `3tier-app-cloudformation.yaml`**

## 생성 리소스 목록

| 구분 | 리소스 |
|---|---|
| 네트워크 | VPC `172.16.0.0/23`, 서브넷 14개(7종 × 2AZ), IGW, **NAT Gateway × 2**, 라우팅, S3 Gateway Endpoint |
| 보안그룹 | `alb-sg` / `backend-sg` / `rds-sg` — **실습용으로 in/out 0.0.0.0/0 전체개방** |
| DB | RDS MySQL 8.0 (db.t3.micro, **백업·강화모니터링·PI 모두 끈 최소 구성**, 비공개) |
| 앱 | EC2 × 2 (pub-svc) — 부팅 시 `git clone → npm install → pm2 start` **자동 배포** |
| 로드밸런서 | ALB(pub-elb) + Target Group(:8080) + Listener(:80) |
| 정적/CDN | S3 + CloudFront(OAC) — `/` → S3, `/api/*` → ALB |
| 자동화 | 프론트 3파일 **S3 자동 업로드**(Lambda), **DB 스키마 자동 초기화**(Lambda, `init.sql`) |

> ⚠️ 보안그룹을 전부 열어둔 것은 **실습 단순화용**입니다. 운영에는 사용하지 마세요.

## 실행 방법 — 콘솔 (파일 첨부)

1. AWS 콘솔에서 리전을 **서울(ap-northeast-2)** 로 선택
2. **CloudFormation → Create stack → With new resources (standard)**
3. *Prepare template* = **Template is ready** →
   *Template source* = **Upload a template file** → **Choose file** 로 `3tier-app-cloudformation.yaml` 첨부 → **Next**
4. **Stack name** 입력 → **Next**
   - 스택 이름은 **소문자 + 28자 이내** (예: `guestbook-3tier`)
   - 파라미터는 전부 기본값으로 둬도 됨 (`DBPassword=devops123!`, `KeyName=공란` 등)
5. *Configure stack options* → 맨 아래 **IAM 리소스 생성 동의 체크박스** ☑ 필수 → **Next**
   > ☑ I acknowledge that AWS CloudFormation might create IAM resources
6. **Submit** → 약 **20~35분** 후 `CREATE_COMPLETE`
7. **Outputs** 탭의 **`CloudFrontURL`** 로 접속
   - 스택 완료 ≠ 앱 준비 완료. **3~5분 더** 기다린 뒤 `tg-<스택명>` 타겟이 **healthy** 가 되면 정상 동작

## CloudFormation 배포 순서

CloudFormation은 리소스 간 의존성(`Ref`/`GetAtt`/`DependsOn`)을 따라 아래 순서로 생성합니다.
두 갈래가 병렬로 시작하지만, **CloudFront가 ALB 주소를 필요로 해서** 결국 한 줄기로 합쳐집니다.

```
네트워크 → RDS → DbInit(스키마) → EC2 → TG/ALB → CloudFront
              └ (병렬) S3 버킷 → FrontendUpload(프론트 업로드)
```

| 단계 | 내용 | 비고 |
|---|---|---|
| **Phase 0 · 네트워크** | VPC → 서브넷 14개 · IGW · EIP 2개 → **NAT × 2** → 라우팅·서브넷 연결 → S3 Endpoint · SG 3개 | ~2–3분 |
| **Phase 1 · 데이터·런타임** | **RDS(MySQL) 생성** / S3 버킷 / IAM 역할 / Lambda 2개 | RDS ~10분 ⏳ |
| **Phase 2 · 초기화** | **FrontendUpload**: 버킷에 프론트 3파일 업로드 · **DbInit**: RDS 준비 후 `init.sql` 실행(DB/테이블/샘플) | Lambda 커스텀 리소스 |
| **Phase 3 · 앱 서버** | EC2 × 2 부팅 → UserData가 `git clone → .env(RDS 엔드포인트 자동 주입) → npm install → pm2 start` | `DependsOn: DbInit` |
| **Phase 4 · 로드밸런서** | Target Group(EC2 등록) → ALB → Listener(80→TG) | |
| **Phase 5 · CDN** | CloudFront 배포(S3=`/`, ALB=`/api/*`, OAC) → 버킷 정책(OAC 허용) | ~15–20분 ⏳ |

- **임계 경로**: `RDS → DbInit → EC2 → ALB → CloudFront` 가 직렬이라 전체 약 **20~35분** 소요
- **RDS 엔드포인트는 자동 주입**: 생성된 RDS 주소를 `!GetAtt` 로 EC2의 `.env` 와 DbInit Lambda에 자동으로 꽂아줌 (수동 입력 불필요)
- **CloudFront 루트 설정**: `DefaultRootObject: index.html` 로 `/` 접근 시 프론트가 바로 뜸

## 완료 후 트래픽 흐름

```
사용자 → CloudFront (HTTPS)
   ├ Default(*) → S3 (프론트, OAC 보호 / S3 직접 접근은 차단)
   └ /api/*    → ALB(:80) → Target Group → EC2(:8080) → RDS(:3306)
```

## 삭제

- 실패한 스택은 보통 `ROLLBACK_COMPLETE` 상태라 **업데이트 불가 → 삭제 후 재생성**해야 합니다.
- S3 비우기는 Lambda가 삭제 시 자동 처리하고, RDS·버킷은 `DeletionPolicy: Delete` 라 함께 제거됩니다.
- **비용 주의**: NAT Gateway × 2, RDS, ALB, CloudFront 가 과금되므로 실습 후 반드시 삭제하세요.

---

# 수동 실습 (docs)

CloudFormation 없이 콘솔에서 단계별로 직접 구축하려면 main branch의 `docs/` 를 따르세요.

1. VPC 구축 (CloudFormation: `docs/cheat-vpc-cloudformation.yaml`)
2. Backend EC2 2대 구축 (pub-svc)
3. RDS MySQL 구축 (pri-db)
4. ALB 구축 및 연결 (pub-elb)
5. Frontend S3 배포
6. CloudFront 연결 (s3, alb)
