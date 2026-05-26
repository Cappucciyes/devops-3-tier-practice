# AWS VPC 구축 가이드

VPC, Subnet, Internet Gateway, NAT Gateway, VPC Endpoint를 **AWS 콘솔**에서 구성하는 실무 가이드입니다.

---

## 1. 아키텍처 개요

```
VPC (172.16.0.0/23) - 실습할때는 밑에 표보고 붙여넣으세요!
├── AZ-a ─────────────────────────────┐   AZ-c ─────────────────────────────┐
│   pub-elb       172.16.0.0/27       │   pub-elb       172.16.0.32/27      │
│   pub-nat       172.16.0.64/27      │   pub-nat       172.16.0.96/27      │
│   pub-svc(임시) 172.16.0.128/27     │   pub-svc(임시) 172.16.0.160/27     │
│   pri-elb       172.16.0.192/27     │   pri-elb       172.16.0.224/27     │
│   pri-svc       172.16.1.0/27       │   pri-svc       172.16.1.32/27      │
│   pri-db        172.16.1.64/27      │   pri-db        172.16.1.96/27      │
│   pri-mgmt      172.16.1.128/27     │   pri-mgmt      172.16.1.160/27     │
└─────────────────────────────────────┘  └─────────────────────────────────┘

  IGW ── Internet
```

### 서브넷 역할

| 서브넷 | 유형 | 용도 |
|---|---|---|
| `pub-elb` | Public | 외부 ALB/NLB |
| `pub-nat` | Public | NAT Gateway |
| `pub-svc (임시)` | Public | **Bastion**. 운영자 SSH 진입점. 임시 운영 자원도 같이 둘 수 있으나 영구 자원 배치는 지양 |
| `pri-elb` | Private | 내부 ALB |
| `pri-svc` | Private | 애플리케이션 EC2 |
| `pri-db` | Private | RDS |
| `pri-mgmt` | Private | CI/CD 도구(Jenkins 등), 모니터링 도구(Prometheus, Grafana 등) |

> **`pub-svc (임시) / Bastion` 운영 가이드**
> - SSH 인바운드는 회사/관리자 IP로만 허용 (Security Group)
> - **SSM Session Manager, VPN 등을 도입하여 점진적 폐기**
> - 영구 자원 배치 금지, 정기 점검으로 미사용 자원 회수

---

## 2. CIDR 설계

### 대역 요약

| 항목 | 값 |
|---|---|
| VPC CIDR | `172.16.0.0/23` (512 IP) |
| Subnet 마스크 | `/27` (서브넷당 32 IP) |
| 사용 가능 IP | 서브넷당 **27개** (AWS가 서브넷마다 5개 예약) |
| 총 서브넷 수 | 14개 (7종 × 2 AZ) |
| 여유 | `/27` 슬롯 2개 (`172.16.1.192/27`, `172.16.1.224/27`) |

### 전체 서브넷 표

| Name | CIDR | AZ | 유형 | 용도 |
|---|---|---|---|---|
| `pub-elb-a` | 172.16.0.0/27 | ap-northeast-2a | Public | 외부 ALB |
| `pub-elb-c` | 172.16.0.32/27 | ap-northeast-2c | Public | 외부 ALB |
| `pub-nat-a` | 172.16.0.64/27 | ap-northeast-2a | Public | NAT GW |
| `pub-nat-c` | 172.16.0.96/27 | ap-northeast-2c | Public | NAT GW |
| `pub-svc-a` | 172.16.0.128/27 | ap-northeast-2a | Public | Bastion (임시) |
| `pub-svc-c` | 172.16.0.160/27 | ap-northeast-2c | Public | Bastion (임시) |
| `pri-elb-a` | 172.16.0.192/27 | ap-northeast-2a | Private | 내부 ALB |
| `pri-elb-c` | 172.16.0.224/27 | ap-northeast-2c | Private | 내부 ALB |
| `pri-svc-a` | 172.16.1.0/27 | ap-northeast-2a | Private | EC2 |
| `pri-svc-c` | 172.16.1.32/27 | ap-northeast-2c | Private | EC2 |
| `pri-db-a` | 172.16.1.64/27 | ap-northeast-2a | Private | RDS |
| `pri-db-c` | 172.16.1.96/27 | ap-northeast-2c | Private | RDS |
| `pri-mgmt-a` | 172.16.1.128/27 | ap-northeast-2a | Private | CI/CD, 모니터링 |
| `pri-mgmt-c` | 172.16.1.160/27 | ap-northeast-2c | Private | CI/CD, 모니터링 |

---

## 3. VPC 생성

**경로**: VPC 콘솔 → **Your VPCs** → **Create VPC**

| 항목 | 값 |
|---|---|
| Resources to create | **VPC only** |
| Name tag | `my-vpc` |
| IPv4 CIDR block | `172.16.0.0/23` |
| IPv6 CIDR block | No IPv6 CIDR block |
| Tenancy | **Default** ← 변경하지 말 것 |

### Tenancy 옵션 설명

VPC(또는 인스턴스)를 생성할 때, 인스턴스가 실행될 **물리 하드웨어를 어떻게 공유할지** 결정하는 옵션입니다.

| 옵션 | 의미 |
|---|---|
| **Default** | 다른 AWS 고객들과 물리 서버를 공유. 일반적인 방식 |
| **Dedicated** | 내 계정 전용 물리 서버에서만 인스턴스 실행 (다른 고객과 하드웨어 공유 X) |

VPC 생성 시 Tenancy를 `Dedicated`로 지정하면, **그 VPC에서 띄우는 모든 인스턴스가 전용 하드웨어로 강제**됩니다.

**왜 쓰는가?**: 금융·의료·정부 등에서 "다른 회사와 물리 서버 공유 금지" 같은 컴플라이언스 요구가 있을 때 사용. Dedicated는 시간당 비용이 크게 상승하고 (전용 하드웨어 요금이 별도 부과됨), 일반적인 서비스에는 불필요합니다.

### 생성 후 설정

**Actions → Edit VPC settings**:
- ✅ Enable DNS resolution
- ✅ Enable DNS hostnames

> EC2가 퍼블릭 DNS 이름을 받으려면 두 옵션 모두 켜야 함

---

## 4. Subnet 생성

**경로**: VPC 콘솔 → **Subnets** → **Create subnet**

VPC를 `my-vpc`로 선택한 뒤, [전체 서브넷 표](#전체-서브넷-표)대로 **14개 서브넷을 모두 생성**합니다.

각 서브넷마다 입력값:

| 항목 | 값 |
|---|---|
| Subnet name | 표의 Name 값 (예: `pub-elb-a`) |
| Availability Zone | 표의 AZ 값 |
| IPv4 CIDR block | 표의 CIDR 값 |

### Public 서브넷 자동 퍼블릭 IP 할당

`pub-*` 서브넷 6개에 대해서만 적용:
1. 서브넷 선택 → **Actions → Edit subnet settings**
2. ✅ **Enable auto-assign public IPv4 address**
3. Save

> Private 서브넷은 절대 켜지 말 것

---

## 5. Internet Gateway (IGW) 생성 및 연결

### 5-1. IGW 생성
**경로**: VPC 콘솔 → **Internet gateways** → **Create internet gateway**

| 항목 | 값 |
|---|---|
| Name tag | `my-igw` |

### 5-2. VPC에 연결
1. 생성된 IGW 선택
2. **Actions → Attach to VPC**
3. VPC: `my-vpc` → **Attach**

> IGW는 VPC당 1개만 연결 가능. **Route Table에 0.0.0.0/0 → IGW 경로가 있어야** 실제로 통신됩니다.

---

## 6. NAT Gateway 생성

### 6-1. Zonal vs Regional NAT Gateway 선택

NAT Gateway는 두 가지 배포 방식 중 선택할 수 있습니다.

| 항목 | Zonal NAT (기존) | Regional NAT (2025.11 신규) |
|---|---|---|
| 배치 단위 | AZ당 1개 직접 생성 | VPC당 1개, AWS가 AZ 자동 확장/축소 |
| 퍼블릭 서브넷 필요 | 필요 | 불필요 |
| Private NAT 지원 | 지원 | 미지원 |
| AZ 확장 시 작업 | 서브넷·EIP·라우트 테이블 수동 작업 | AWS가 자동 (최대 60분 소요) |
| 트래픽 경로 추적 | AZ 단위로 명확, 디버깅 용이 | 처리 AZ 추적 어려움, 디버깅 복잡 |
| 시간당 요금 | active AZ 수만큼 과금 | 동일 (active AZ 수만큼 과금) |
| 비용 리스크 | 라우트 실수 시 cross-AZ 데이터 전송 비용 발생 | 자동 축소로 유휴 비용 발생 안 함, 라우트 실수 차단 |

**선택 가이드**
- **Zonal**: AZ 격리·라우팅 가시성·Private NAT가 필요한 경우. 현재 가이드 표준
- **Regional**: 단순한 운영을 우선하고 디버깅 깊이가 덜 중요한 경우, 또는 멀티 AZ 운영 부담을 줄이고 싶을 때

> 본 가이드는 **Zonal 방식**으로 진행합니다 (가시성 + Private NAT 호환).

### 6-2. Zonal NAT Gateway 생성 (AZ별 1개씩)

**경로**: VPC 콘솔 → **NAT gateways** → **Create NAT gateway**

#### AZ-a NAT Gateway

| 항목 | 값 |
|---|---|
| Name | `nat-a` |
| Subnet | `pub-nat-a` |
| Connectivity type | Public |
| Elastic IP allocation ID | **Allocate Elastic IP** 클릭 |

#### AZ-c NAT Gateway

| 항목 | 값 |
|---|---|
| Name | `nat-c` |
| Subnet | `pub-nat-c` |
| Connectivity type | Public |
| Elastic IP allocation ID | **Allocate Elastic IP** 클릭 |

> NAT Gateway는 시간당 + 데이터 처리당 과금. 비운영 환경은 단일 NAT으로 비용 절감 가능, 운영 환경은 AZ당 1개 권장.

---

## 7. Route Table 구성

### 7-1. 생성할 라우팅 테이블

| 라우팅 테이블 이름 | 연결할 서브넷 | 0.0.0.0/0 다음 홉 |
|---|---|---|
| `rt-public` | pub-elb-a, pub-elb-c, pub-nat-a, pub-nat-c, pub-svc-a, pub-svc-c | **IGW** (`my-igw`) |
| `rt-private-a` | pri-elb-a, pri-svc-a, pri-mgmt-a | **NAT GW** (`nat-a`) |
| `rt-private-c` | pri-elb-c, pri-svc-c, pri-mgmt-c | **NAT GW** (`nat-c`) |
| `rt-db-a` | pri-db-a | **None** |
| `rt-db-c` | pri-db-c | **None** |

> Private 라우팅 테이블을 AZ별로 분리하는 이유: **같은 AZ의 NAT을 사용**해야 cross-AZ 데이터 전송 비용을 막고 장애 격리가 가능

### 7-2. 생성 절차 (각 라우팅 테이블 공통)

**경로**: VPC 콘솔 → **Route tables** → **Create route table**

1. Name 입력 (`rt-public` 등) → VPC: `my-vpc` 선택 → Create
2. **Routes 탭 → Edit routes → Add route**
   - Destination: `0.0.0.0/0`
   - Target: 위 표 참조 (IGW 또는 해당 AZ NAT GW)
3. **Subnet associations 탭 → Edit subnet associations**
   - 위 표의 "연결할 서브넷" 모두 체크 → Save

### 7-3. 결과 확인

| 서브넷 | 0.0.0.0/0 Target |
|---|---|
| pub-* | igw-xxxx |
| pri-*-a | nat-xxxx (nat-a) |
| pri-*-c | nat-xxxx (nat-c) |
| pri-db-a | None |
| pri-db-c | None |

---

## 8. 검증 체크리스트

### 8-1. 콘솔에서 확인

- [ ] VPC `my-vpc` 가 `172.16.0.0/23` 으로 생성됨, Tenancy `Default`
- [ ] 서브넷 14개 모두 정상 생성, AZ/CIDR 일치
- [ ] IGW `my-igw` 가 `my-vpc` 에 Attached
- [ ] NAT GW 2개 Available, 각각 EIP 할당
- [ ] 라우팅 테이블 5개 생성 및 서브넷 연결 완료
