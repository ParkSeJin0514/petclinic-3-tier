# 🏗️ AWS 인프라 설계

## 프로젝트 정보
- **시작일**: 2025년 10월 2일
- **종료일**: 2025년 10월 13일
- **상태**: Completed
- **담당자**: 박세진, Seung Lo, 홍정호

---

## 🎯 AWS 3-Tier Architecture 개요

본 프로젝트는 고가용성, 확장성, 보안을 고려한 3계층 아키텍처를 AWS에 구현합니다.

```
Internet
    ↓
CloudFront (CDN)
    ↓
External ALB (Public Subnet)
    ↓
WEB Tier (Private Subnet) - Apache httpd
    ↓
Internal ALB (Private Subnet)
    ↓
WAS Tier (Private Subnet) - Tomcat
    ↓
DB Tier (Private Subnet) - RDS MySQL
```

---

## 📝 Service Naming Convention

일관된 네이밍 규칙으로 리소스 관리를 용이하게 합니다.

### 기본 네이밍 패턴
```
petclinic-kr-{resource-type}-{tier/function}-{az}
```

### 리소스별 네이밍 테이블

| AWS 리소스 | 이름 규칙 | 예시 |
|------------|-----------|------|
| **VPC** | petclinic-kr-vpc | petclinic-kr-vpc |
| **Subnet** | petclinic-kr-{public/private}-{tier}-{az} | petclinic-kr-public-a<br>petclinic-kr-private-web-a |
| **Routing Table** | petclinic-kr-rt-{public/private}-{az} | petclinic-kr-rt-public<br>petclinic-kr-rt-private-a |
| **Internet Gateway** | petclinic-kr-igw | petclinic-kr-igw |
| **EIP** | petclinic-kr-eip-nat-{az} | petclinic-kr-eip-nat-a |
| **NAT Gateway** | petclinic-kr-nat-{az} | petclinic-kr-nat-a |
| **NACL** | petclinic-kr-nacl-{tier} | petclinic-kr-nacl-web |
| **Security Group** | petclinic-kr-sg-{tier} | petclinic-kr-sg-web |
| **Instance** | petclinic-kr-{tier}-{az} | petclinic-kr-web-a |
| **Launch Template** | petclinic-kr-tmp-{tier} | petclinic-kr-tmp-web |
| **AMI** | petclinic-kr-ami-{tier} | petclinic-kr-ami-web |
| **EBS Volume** | petclinic-kr-vol-{tier} | petclinic-kr-vol-web |
| **Load Balancer** | petclinic-kr-alb-{pub/int} | petclinic-kr-alb-pub<br>petclinic-kr-alb-int |
| **Target Group** | petclinic-kr-tg-{alb-type} | petclinic-kr-tg-alb-pub<br>petclinic-kr-tg-alb-int |
| **Auto Scaling Group** | petclinic-kr-asg-{tier} | petclinic-kr-asg-web<br>petclinic-kr-asg-was |
| **CloudFront** | petclinic-cf | petclinic-cf |
| **RDS Database** | petclinic-kr-db | petclinic-kr-db |

---

## 🌐 VPC 설계

### VPC 구성
- **VPC Name**: petclinic-kr-vpc
- **IP CIDR**: 10.0.0.0/16
- **가용 IP**: 65,536개
- **리전**: ap-northeast-2 (Seoul)
- **가용 영역**: ap-northeast-2a, ap-northeast-2b

---

## 🔌 Subnet 설계

### Subnet 네이밍
- **Public Subnet**: petclinic-kr-public-{a, b}
- **Private Subnet**: petclinic-kr-private-{web, was, db}-{a, b}

### IP 대역 할당 전략

특정 IP 대역을 사용하여 한눈에 알아볼 수 있고 확장성 있게 구성했습니다.

#### Subnet CIDR 블록

| Subnet 유형 | AZ | CIDR 블록 | 가용 IP | 용도 |
|-------------|----|-----------|---------|----|
| **Public** | 2a | 10.0.0.0/24 | 256 | External ALB, NAT Gateway |
| **Public** | 2b | 10.0.10.0/24 | 256 | External ALB, NAT Gateway |
| **Private (WEB)** | 2a | 10.0.50.0/24 | 256 | WEB 서버 (Apache) |
| **Private (WEB)** | 2b | 10.0.60.0/24 | 256 | WEB 서버 (Apache) |
| **Private (WAS)** | 2a | 10.0.100.0/24 | 256 | WAS 서버 (Tomcat) |
| **Private (WAS)** | 2b | 10.0.110.0/24 | 256 | WAS 서버 (Tomcat) |
| **Private (DB)** | 2a | 10.0.150.0/24 | 256 | RDS MySQL |
| **Private (DB)** | 2b | 10.0.160.0/24 | 256 | RDS MySQL |

### IP 대역 규칙
- **Private 1 (WEB-A)**: 10.0.50.%
- **Private 2 (WEB-B)**: 10.0.60.%
- **Private 3 (WAS-A)**: 10.0.100.%
- **Private 4 (WAS-B)**: 10.0.110.%
- **Private 5 (DB-A)**: 10.0.150.%
- **Private 6 (DB-B)**: 10.0.160.%

### 설계 원칙
1. **가독성**: 각 Tier별로 쉽게 구분 가능한 대역
2. **확장성**: 각 서브넷마다 충분한 IP 공간 확보
3. **격리**: 계층별로 명확히 분리된 네트워크
4. **여유 공간**: 향후 추가 서브넷 생성 가능

---

## 🛣️ Routing Table 설계

### 설계 원칙
라우팅 테이블을 AZ별로 분리하여 다음과 같은 이점을 확보합니다:
- **고가용성**: AZ 장애 시 반대편 NAT에 의존하지 않음
- **비용 절감**: 크로스 AZ 데이터 전송 비용 감소
- **지연 최소화**: 같은 AZ 내에서 NAT 통신

### Routing Table 구성

#### Public Routing Table
- **이름**: petclinic-kr-rt-public
- **연결 서브넷**: Public Subnet A, B
- **라우팅 규칙**:
  - 10.0.0.0/16 → local (VPC 내부 통신)
  - 0.0.0.0/0 → Internet Gateway

#### Private Routing Table A
- **이름**: petclinic-kr-rt-private-a
- **연결 서브넷**: Private Web-A, WAS-A, DB-A
- **라우팅 규칙**:
  - 10.0.0.0/16 → local
  - 0.0.0.0/0 → NAT Gateway A

#### Private Routing Table B
- **이름**: petclinic-kr-rt-private-b
- **연결 서브넷**: Private Web-B, WAS-B, DB-B
- **라우팅 규칙**:
  - 10.0.0.0/16 → local
  - 0.0.0.0/0 → NAT Gateway B

---

## 🔐 NAT Gateway 및 EIP

### NAT Gateway 전략
AZ별로 독립적인 NAT Gateway를 구성하여 가용성을 높입니다.

#### NAT Gateway A
- **이름**: petclinic-kr-nat-a
- **위치**: Public Subnet A
- **EIP**: petclinic-kr-eip-nat-a
- **서비스 대상**: Private Subnet A 계열

#### NAT Gateway B
- **이름**: petclinic-kr-nat-b
- **위치**: Public Subnet B
- **EIP**: petclinic-kr-eip-nat-b
- **서비스 대상**: Private Subnet B 계열

### 장점
1. **고가용성**: 한쪽 AZ 장애 시에도 다른 AZ는 정상 운영
2. **성능**: 크로스 AZ 트래픽 없이 같은 AZ 내 처리
3. **비용**: 데이터 전송 비용 절감

---

## 🛡️ Security Group (SG) 설계

### 보안 그룹 구성 원칙
- **최소 권한 원칙**: 필요한 포트와 소스만 허용
- **계층별 격리**: 각 Tier는 필요한 통신만 허용
- **모든 아웃바운드 허용**: 기본적으로 모든 아웃바운드 트래픽 허용

### Security Group 상세

#### 1. External ALB Security Group
**이름**: petclinic-kr-sg-alb-ext

**Inbound Rules**:
| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| HTTP | TCP | 80 | 0.0.0.0/0 | 인터넷에서 HTTP 트래픽 |
| HTTPS | TCP | 443 | 0.0.0.0/0 | 인터넷에서 HTTPS 트래픽 |

**Outbound Rules**:
| Type | Protocol | Port | Destination | Description |
|------|----------|------|-------------|-------------|
| All | All | All | 0.0.0.0/0 | 모든 아웃바운드 허용 |

---

#### 2. WEB Tier Security Group
**이름**: petclinic-kr-sg-web

**Inbound Rules**:
| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| HTTP | TCP | 80 | petclinic-kr-sg-alb-ext | External ALB에서만 |
| SSH | TCP | 22 | petclinic-kr-sg-bastion | Bastion에서만 |
| Custom | TCP | 9100 | petclinic-kr-sg-prometheus | Node Exporter |

**Outbound Rules**:
| Type | Protocol | Port | Destination | Description |
|------|----------|------|-------------|-------------|
| All | All | All | 0.0.0.0/0 | 모든 아웃바운드 허용 |

---

#### 3. Internal ALB Security Group
**이름**: petclinic-kr-sg-alb-int

**Inbound Rules**:
| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| HTTP | TCP | 80 | petclinic-kr-sg-web | WEB Tier에서만 |

**Outbound Rules**:
| Type | Protocol | Port | Destination | Description |
|------|----------|------|-------------|-------------|
| All | All | All | 0.0.0.0/0 | 모든 아웃바운드 허용 |

---

#### 4. WAS Tier Security Group
**이름**: petclinic-kr-sg-was

**Inbound Rules**:
| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| Custom | TCP | 8080 | petclinic-kr-sg-alb-int | Internal ALB에서만 |
| SSH | TCP | 22 | petclinic-kr-sg-bastion | Bastion에서만 |
| Custom | TCP | 9100 | petclinic-kr-sg-prometheus | Node Exporter |

**Outbound Rules**:
| Type | Protocol | Port | Destination | Description |
|------|----------|------|-------------|-------------|
| All | All | All | 0.0.0.0/0 | 모든 아웃바운드 허용 |

---

#### 5. DB Tier Security Group
**이름**: petclinic-kr-sg-db

**Inbound Rules**:
| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| MySQL | TCP | 3306 | petclinic-kr-sg-was | WAS Tier에서만 |
| MySQL | TCP | 3306 | petclinic-kr-sg-bastion | 관리 목적 |

**Outbound Rules**:
| Type | Protocol | Port | Destination | Description |
|------|----------|------|-------------|-------------|
| All | All | All | 0.0.0.0/0 | 모든 아웃바운드 허용 |

---

#### 6. Bastion Host Security Group
**이름**: petclinic-kr-sg-bastion

**Inbound Rules**:
| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| SSH | TCP | 22 | [관리자 IP] | 특정 IP에서만 SSH |

**Outbound Rules**:
| Type | Protocol | Port | Destination | Description |
|------|----------|------|-------------|-------------|
| All | All | All | 0.0.0.0/0 | 모든 아웃바운드 허용 |

---

#### 7. Prometheus Server Security Group
**이름**: petclinic-kr-sg-prometheus

**Inbound Rules**:
| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| Custom | TCP | 9090 | petclinic-kr-sg-bastion | Prometheus UI 접근 |
| SSH | TCP | 22 | petclinic-kr-sg-bastion | SSH 접근 |

**Outbound Rules**:
| Type | Protocol | Port | Destination | Description |
|------|----------|------|-------------|-------------|
| All | All | All | 0.0.0.0/0 | 메트릭 수집 및 Grafana 연동 |

---

## 🚧 NACL (Network Access Control List) 설계

### NACL 특징
- **서브넷 단위 방화벽**: 서브넷 레벨에서 작동
- **Stateless**: 요청과 응답을 별도로 허용해야 함
- **규칙 번호**: 낮은 번호가 우선 (100, 200, 300...)
- **임시 포트**: 1024-65535 범위 고려 필요

### 운영 전략
실무에서는 **기본 NACL (Allow All) + Security Group 중심 운영**이 베스트 프랙티스입니다.
- NACL: 서브넷 레벨 기본 보호
- Security Group: 세밀한 트래픽 제어

### Public Subnet NACL (External ALB용)
**이름**: petclinic-kr-nacl-public

#### Inbound Rules
| Rule # | Type | Protocol | Port Range | Source | Allow/Deny |
|--------|------|----------|------------|--------|------------|
| 100 | HTTP | TCP | 80 | 0.0.0.0/0 | ALLOW |
| 110 | HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 120 | Custom | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |
| * | All | All | All | 0.0.0.0/0 | DENY |

#### Outbound Rules
| Rule # | Type | Protocol | Port Range | Destination | Allow/Deny |
|--------|------|----------|------------|-------------|------------|
| 100 | HTTP | TCP | 80 | 0.0.0.0/0 | ALLOW |
| 110 | HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 120 | Custom | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |
| * | All | All | All | 0.0.0.0/0 | DENY |

### Private Subnet NACL (WEB/WAS)
**이름**: petclinic-kr-nacl-private

#### Inbound Rules
| Rule # | Type | Protocol | Port Range | Source | Allow/Deny |
|--------|------|----------|------------|--------|------------|
| 100 | HTTP | TCP | 80 | 10.0.0.0/16 | ALLOW |
| 110 | Custom | TCP | 8080 | 10.0.0.0/16 | ALLOW |
| 120 | SSH | TCP | 22 | 10.0.0.0/16 | ALLOW |
| 130 | Custom | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |
| * | All | All | All | 0.0.0.0/0 | DENY |

#### Outbound Rules
| Rule # | Type | Protocol | Port Range | Destination | Allow/Deny |
|--------|------|----------|------------|-------------|------------|
| 100 | All | All | All | 0.0.0.0/0 | ALLOW |
| * | All | All | All | 0.0.0.0/0 | DENY |

---

## 📐 아키텍처 다이어그램

```
                               [Users]
                                  |
                                  ↓
                          [CloudFront + WAF]
                                  |
                                  ↓
                      ┌───────────────────────┐
                      │   Internet Gateway    │
                      └───────────────────────┘
                                  |
         ┌────────────────────────┴────────────────────────┐
         │                 Public Subnet                   │
         │  ┌──────────────────────────────────────────┐  │
         │  │       External ALB (80, 443)             │  │
         │  └──────────────────────────────────────────┘  │
         │  ┌──────────────┐         ┌──────────────┐    │
         │  │ NAT Gateway-A│         │ NAT Gateway-B│    │
         │  └──────────────┘         └──────────────┘    │
         └────────────────────────────────────────────────┘
                      |                      |
         ┌────────────┴────────┐  ┌─────────┴──────────┐
         │  Private Subnet A   │  │  Private Subnet B  │
         │                     │  │                    │
         │  ┌──────────────┐  │  │  ┌──────────────┐ │
         │  │  WEB Tier    │  │  │  │  WEB Tier    │ │
         │  │  (Apache)    │  │  │  │  (Apache)    │ │
         │  └──────────────┘  │  │  └──────────────┘ │
         │         ↓           │  │         ↓         │
         │  ┌──────────────┐  │  │  ┌──────────────┐ │
         │  │ Internal ALB │←─┼──┼─→│ Internal ALB │ │
         │  └──────────────┘  │  │  └──────────────┘ │
         │         ↓           │  │         ↓         │
         │  ┌──────────────┐  │  │  ┌──────────────┐ │
         │  │  WAS Tier    │  │  │  │  WAS Tier    │ │
         │  │  (Tomcat)    │  │  │  │  (Tomcat)    │ │
         │  └──────────────┘  │  │  └──────────────┘ │
         │         ↓           │  │         ↓         │
         │  ┌──────────────┐  │  │  ┌──────────────┐ │
         │  │  DB Subnet   │  │  │  │  DB Subnet   │ │
         │  └──────────────┘  │  │  └──────────────┘ │
         └─────────────────────┘  └────────────────────┘
                      └──────────┬──────────┘
                                 ↓
                          ┌──────────────┐
                          │  RDS MySQL   │
                          │  (Multi-AZ)  │
                          └──────────────┘
```

---

## 🎯 설계 핵심 원칙

### 1. 고가용성 (High Availability)
- Multi-AZ 구성으로 단일 장애점 제거
- AZ별 독립적인 NAT Gateway
- RDS Multi-AZ 배포

### 2. 보안 (Security)
- 다층 방어: CloudFront WAF → NACL → Security Group
- Private Subnet에 모든 애플리케이션 배치
- Bastion Host를 통한 관리 접근만 허용

### 3. 확장성 (Scalability)
- Auto Scaling Group을 통한 자동 확장
- 충분한 IP 주소 공간 확보
- 계층별 독립적 확장 가능

### 4. 성능 (Performance)
- AZ별 독립 라우팅으로 지연 최소화
- CloudFront CDN으로 정적 콘텐츠 가속
- ALB를 통한 효율적인 로드 밸런싱

### 5. 비용 최적화 (Cost Optimization)
- 크로스 AZ 트래픽 최소화
- NAT Gateway를 AZ별로 배치하여 비용 절감
- Auto Scaling으로 리소스 효율화

---

## 📚 참고 사항

### 모범 사례
1. **IP 대역 계획**: 향후 확장을 고려한 충분한 여유 확보
2. **네이밍 규칙**: 일관된 네이밍으로 관리 용이성 향상
3. **보안 계층화**: NACL과 Security Group 조합 활용
4. **문서화**: 모든 설정을 문서화하여 유지보수 편의성 확보

### 주의사항
1. NAT Gateway는 단일 AZ 내에서만 작동 (크로스 AZ 불가)
2. NACL은 Stateless이므로 양방향 규칙 필요
3. Security Group 변경 시 실시간 적용됨 (주의 필요)
4. EIP는 제한된 리소스이므로 필요한 만큼만 할당

---

## 🔄 다음 단계

인프라 설계가 완료되었으므로 다음 단계를 진행합니다:

1. ✅ VPC 및 네트워크 리소스 생성
2. ✅ 보안 그룹 및 NACL 설정
3. → [애플리케이션 서비스 구성](./03-application-service.md)
4. → RDS 데이터베이스 설정
5. → Auto Scaling 및 ALB 구성
6. → 모니터링 스택 구축
