# 실습 3. RDS MySQL 구축 (pri-db)

## 목표

- pri-db 서브넷에 RDS MySQL 생성
- Backend EC2에서 직접 접속 확인
- 테이블 초기화

## Step 1. DB Subnet Group 생성

RDS 콘솔 → Subnet groups → Create DB subnet group

- 이름: `guestbook-db-subnet-group`
- VPC: 실습 VPC
- Availability Zones: `ap-northeast-2a`, `ap-northeast-2c`
- Subnets: `pri-db-a`, `pri-db-c` 선택

## Step 2. RDS Security Group 생성

이름: `rds-sg`
VPC: 실습 VPC

**인바운드 규칙**

| 타입 | 포트 | 소스 | 설명 |
|---|---|---|---|
| MySQL/Aurora | 3306 | `backend-sg` | Backend EC2에서만 접근 |

⚠️ 소스는 `backend-sg`(보안 그룹) 자체를 지정. IP가 아니라 SG를 소스로 걸면 backend EC2가 늘어나도 규칙 수정이 필요 없다.

## Step 3. RDS 인스턴스 생성

RDS 콘솔 → 데이터베이스 생성

**생성 방식**
- **표준 생성 (Standard create / 전체 구성)** 선택 ⭐
- "손쉬운 생성(Easy create)"이 아니라 표준 생성을 써야 서브넷 그룹·SG·퍼블릭 액세스 등 네트워크 설정을 직접 지정할 수 있다. 실습 목적상 반드시 전체 구성으로 진행.

**엔진 옵션**
- 엔진: MySQL
- 버전: MySQL 8.0.x (최신 안정판)

**템플릿**
- 프리 티어 (실습용)

**설정**
- DB 인스턴스 식별자: `guestbook-db`
- 마스터 사용자: `admin`
- 자격 증명 관리: 셀프 관리 (Self managed)
- 마스터 암호: `devops123!` (실습용 — 운영에서는 절대 이런 약한 암호 금지)

**인스턴스 구성**
- `db.t3.micro` (프리 티어)

**스토리지**
- 범용 SSD (gp2)
- 할당 스토리지: 20 GiB
- 스토리지 자동 조정: 비활성화 (실습)

**연결**
- VPC: 실습 VPC
- DB Subnet Group: `guestbook-db-subnet-group`
- 퍼블릭 액세스: **아니오** ⭐ 중요
- VPC Security Group: `rds-sg`
- AZ: `ap-northeast-2a`

**추가 구성**
- 초기 데이터베이스 이름: (공란 — SQL로 생성)
- 백업 보관 기간: 1일 (실습)
- 자동 백업 창: 기본값
- 유지 관리: 기본값

생성까지 약 5~10분 소요.

## Step 4. 엔드포인트 확인

RDS 콘솔 → DB 인스턴스 → `guestbook-db` 클릭 → **엔드포인트** 복사

예시:
```
guestbook-db.xxxxxxxxxx.ap-northeast-2.rds.amazonaws.com
```

## Step 5. Backend EC2에서 MySQL 접속

backend-a(또는 backend-c)에 SSH로 접속한다. MySQL 클라이언트는 실습 2의 `start.sh`에서 이미 설치됨. (없다면 `sudo apt install -y mysql-client`)

RDS에 접속:

```bash
mysql -h <RDS_엔드포인트> -u admin -p
```

암호 입력 후 MySQL 프롬프트 진입:

```
mysql>
```

> 📌 RDS는 퍼블릭 액세스 "아니오"라서 인터넷에서 직접 못 붙는다. 같은 VPC 안의 backend EC2에서, `rds-sg`가 `backend-sg`를 허용하고 있기 때문에 접속이 된다.

## Step 6. 데이터베이스 초기화

backend EC2에는 `git clone`으로 받은 `init.sql`이 이미 있다 (`~/devops-3-tier-practice/backend/init.sql`).

**방법 1. 파일로 한 번에 실행 (권장)**

```bash
cd ~/devops-3-tier-practice/backend
mysql -h <RDS_엔드포인트> -u admin -p < init.sql
```

**방법 2. MySQL 프롬프트에서 직접 실행**

```sql
CREATE DATABASE IF NOT EXISTS guestbook
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

USE guestbook;

CREATE TABLE IF NOT EXISTS messages (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  content TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

INSERT INTO messages (name, content) VALUES
  ('관리자', 'AWS 3-Tier 실습 환영!'),
  ('테스터', '메시지를 작성하면 DB에 저장됩니다.');

SELECT * FROM messages;
```

## Step 7. Backend EC2에서 연결 테스트

backend-a, backend-c 각각:

```bash
# MySQL client로 직접 접속 테스트
mysql -h <RDS_엔드포인트> -u admin -p -e "SHOW DATABASES;"

# .env 업데이트
cd ~/devops-3-tier-practice/backend
nano .env
```

`.env` 업데이트:

```
PORT=8080
DB_HOST=guestbook-db.xxxxxxxxxx.ap-northeast-2.rds.amazonaws.com
DB_PORT=3306
DB_USER=admin
DB_PASSWORD=devops123!
DB_NAME=guestbook
```

## Step 8. Backend 실행

```bash
cd ~/devops-3-tier-practice/backend
pm2 start server.js --name backend
pm2 logs backend
```

로그에서 확인:
```
Server starting: ip-172-16-X-X (172.16.X.X)
DB connected successfully
Backend server ip-172-16-X-X running on port 8080
```

## Step 9. API 테스트

Backend EC2 내에서:

```bash
# 헬스체크
curl http://localhost:8080/api/health

# DB 헬스체크
curl http://localhost:8080/api/health/db

# 메시지 목록
curl http://localhost:8080/api/messages

# 메시지 작성
curl -X POST http://localhost:8080/api/messages \
  -H "Content-Type: application/json" \
  -d '{"name":"테스트","content":"Hello from EC2"}'
```

두 Backend EC2 모두에서 같은 데이터가 조회되는지 확인!
→ DB 공유 동작 증명

## 체크리스트

- [ ] DB Subnet Group이 pri-db-a, pri-db-c에 걸쳐있음
- [ ] RDS 퍼블릭 액세스 "아니오" 확인
- [ ] rds-sg가 backend-sg의 3306 허용
- [ ] Backend EC2에서 RDS 접속 성공
- [ ] guestbook DB와 messages 테이블 생성 완료
- [ ] Backend EC2 2대 모두에서 DB 연결 성공
- [ ] API 테스트 모두 통과

## 트러블슈팅

**RDS 접속이 안 될 때**
- rds-sg 인바운드 Source가 `backend-sg`인지 확인
- 포트 3306 열려 있는지
- RDS와 backend EC2가 같은 VPC인지

**`Can't connect to MySQL server`**
- 엔드포인트 주소 정확한지
- 포트 3306 맞는지
- 네트워크 확인: `telnet <엔드포인트> 3306`

**암호 오류**
- 마스터 암호 재확인
- 특수문자가 shell에 먹힐 수 있으니 `-p` 뒤에 암호 넣지 말고 프롬프트에서 입력

**`Unknown database 'guestbook'`**
- init.sql 실행했는지 확인
- `SHOW DATABASES;`로 존재 여부 확인

## 비용 주의 ⚠️

- RDS는 **생성 즉시 과금 시작**
- 프리 티어라도 다중 인스턴스 띄우면 과금
- 실습 끝나면 **반드시 삭제**:
  - 스냅샷 유지 옵션 선택 여부 확인
  - "최종 스냅샷 없이 삭제" 권장 (실습용)