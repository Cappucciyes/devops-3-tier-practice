# DevOps 3-Tier Practice

AWS 3-Tier 아키텍처 실습용 방명록 애플리케이션

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

## 실습 순서

본 Repo의 docs 참고

1. VPC 구축 (CloudFormation)
2. Backend EC2 2대 구축 (pub-svc)
3. RDS MySQL 구축 (pri-db)
4. ALB 구축 및 연결 (pub-elb)
5. Frontend S3 배포
6. CloudFront 연결 (s3, alb)

## 주요 확인 포인트

### 로드밸런싱 동작 확인
- 프론트엔드에서 `응답한 서버: ip-xxx-xxx` 표시
- 새로고침 시 서버 ID 변경 확인 → ALB 동작 증명

### 장애 시뮬레이션
- EC2 1대 중지 → ALB가 건강한 EC2로만 트래픽 전달
- 서비스 지속 확인

실습 끝나면 **반드시 전체 리소스 삭제**:

## 삭제 순서 (역순):
1. CloudFront Distribution (Disable → Delete)
2. S3 버킷 (비우고 삭제)
3. ALB
4. Target Group
5. RDS (스냅샷 없이)
6. EC2 인스턴스 (모두)
7. Elastic IP (해제)
8. CloudFormation Stack 삭제