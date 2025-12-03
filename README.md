# 🏗️ AWS 3-Tier Architecture - PetClinic 프로젝트

## 📋 프로젝트 개요

소규모 온라인 쇼핑몰(애완용품 쇼핑몰)의 안정적인 서비스 운영을 위한 AWS 기반 3-Tier 아키텍처 구축 프로젝트입니다.

### 대상 서비스 정보
- **업종**: 애완용품 쇼핑몰 (소규모)
- **일평균 방문자**: 300명
- **피크 타임**: 1,000 ~ 2,000명
- **월평균 주문**: 30건

### 프로젝트 기간
- **시작일**: 2025년 10월 1일
- **종료일**: 2025년 10월 17일
- **상태**: Completed

### 참여자
- 박세진
- Seung Lo
- 홍정호

---

## 🎯 주요 목표

1. **고가용성(High Availability)**: Multi-AZ 구성을 통한 장애 대응
2. **자동 확장성(Auto Scaling)**: 트래픽 패턴에 따른 동적 리소스 조정
3. **비용 최적화**: 시간대별 스케줄링을 통한 효율적인 자원 관리
4. **보안**: 다층 보안 구성 (NACL, Security Group, Private Subnet)
5. **모니터링**: Prometheus + Grafana를 통한 실시간 모니터링 및 알림

---

## 🏛️ 아키텍처 구성

### 3-Tier Architecture
```
[Internet]
    ↓
[CloudFront + WAF]
    ↓
[External ALB] (Public Subnet)
    ↓
[WEB Tier - Apache/httpd] (Private Subnet)
    ↓
[Internal ALB]
    ↓
[WAS Tier - Tomcat] (Private Subnet)
    ↓
[DB Tier - RDS MySQL] (Private Subnet)
```

### 주요 AWS 서비스
- **네트워크**: VPC, Subnet (Public/Private), NAT Gateway, Internet Gateway
- **컴퓨팅**: EC2 Auto Scaling Groups, Launch Templates
- **로드밸런싱**: Application Load Balancer (External/Internal)
- **데이터베이스**: RDS MySQL (Multi-AZ)
- **스토리지**: S3 (정적 컨텐츠)
- **CDN**: CloudFront
- **모니터링**: CloudWatch, Prometheus, Grafana
- **보안**: WAF, Security Groups, NACL

---

## 📚 문서 구성

본 프로젝트는 다음과 같이 상세 문서로 구성되어 있습니다:

1. **[방향성 및 시나리오](./01-direction-scenario.md)**
   - 트래픽 패턴 분석
   - Auto Scaling 정책
   - 시간대별 스케일링 시나리오

2. **[AWS 인프라 설계](./02-infrastructure-design.md)**
   - VPC 및 네트워크 구성
   - 서비스 네이밍 규칙
   - 보안 그룹 및 NACL 설정
   - 리소스 배치 전략

3. **[애플리케이션 서비스](./03-application-service.md)**
   - WEB 서버 (Apache httpd) 설정
   - WAS 서버 (Tomcat) 설정
   - Reverse Proxy 구성
   - ALB 설정 및 헬스체크

4. **[부하 테스트 및 보안](./04-load-test-security.md)**
   - K6를 이용한 부하 테스트
   - 테스트 시나리오 및 결과
   - 보안 정책 수립

5. **[모니터링 및 알림](./05-monitoring-alerting.md)**
   - Node Exporter 설치 및 설정
   - Prometheus 구성
   - Grafana 대시보드 구축
   - 알림 규칙 설정

6. **[SLO 정책](./06-slo-policy.md)**
   - Service Level Objective 정의
   - SLI (Service Level Indicator) 설정
   - 오류 예산 관리
   - Grafana 대시보드 구성

7. **[확장 기능 구현](./07-advanced-features.md)**
   - 추가 기능 구현 계획
   - 성능 최적화 방안

---

## 🔄 Auto Scaling 정책

### 동적 Auto Scaling
- **Scale-out 조건**: CPU 사용률 50% 이상
- **Scale-in 조건**: CPU 사용률 30% 이하
- **최소 인스턴스**: 2대 (고가용성 유지)
- **최대 인스턴스**: 8대 (비용 통제)

### 시간대별 스케줄링

| 시간대 | WEB | WAS | 설명 |
|--------|-----|-----|------|
| 00:00 ~ 09:00 | 2대 | 2대 | 최소 유지 (새벽/이른 아침) |
| 09:00 ~ 18:50 | 2대 | 2대 | 정상 운영 |
| 18:50 | 3~4대 | 3~5대 | 피크 타임 대비 사전 스케일 아웃 |
| 19:00 ~ 22:00 | 4~6대 | 5~7대 | 피크 타임 (동적 조정) |
| 22:00 ~ 24:00 | 2~3대 | 2~3대 | 점진적 스케일 인 |

---

## 🛡️ 보안 구성

### 네트워크 보안
- **Public Subnet**: External ALB만 배치
- **Private Subnet**: WEB, WAS, DB 모두 Private 영역에 배치
- **NAT Gateway**: AZ별 독립 NAT (크로스 AZ 비용 절감)

### 보안 계층
1. **CloudFront + WAF**: DDoS 방어, SQL Injection 차단
2. **NACL**: 서브넷 레벨 방화벽
3. **Security Group**: 인스턴스 레벨 방화벽
4. **IAM Role**: 최소 권한 원칙 적용

---

## 📊 모니터링 구성

### 메트릭 수집
- **Node Exporter**: EC2 인스턴스 시스템 메트릭
- **Prometheus**: 메트릭 수집 및 저장
- **Grafana**: 시각화 및 대시보드

### 주요 모니터링 지표
- CPU 사용률
- 메모리 사용률
- 디스크 I/O
- 네트워크 트래픽
- ALB 요청 수 및 응답 시간
- RDS 성능 지표

---

## 🎯 SLO (Service Level Objective)

### 가용성 목표
- **SLO**: 99.9% (30일 롤링 기준)
- **오류 예산**: 0.1%

### 성능 목표
- **응답 시간**: p95 ≤ 400ms
- **체크아웃 성공률**: ≥ 98.5%

### 트래픽 가정
| 시나리오 | 인당 요청빈도 | RPS | 30일 총 요청 | 오류예산 (99.9%) |
|----------|---------------|-----|--------------|------------------|
| 보수적 | 1 req / 10s | 100 | 259,200,000 | 259,200 |
| 보통 | 1 req / 5s | 200 | 518,400,000 | 518,400 |

---

## 🚀 시작하기

### 사전 요구사항
- AWS 계정
- AWS CLI 설치 및 구성
- Terraform (인프라 코드화 시)
- 기본적인 Linux 명령어 지식

### 배포 순서
1. VPC 및 네트워크 리소스 생성
2. 보안 그룹 및 NACL 설정
3. RDS 데이터베이스 생성
4. Launch Template 및 AMI 준비
5. ALB 및 Target Group 설정
6. Auto Scaling Group 생성
7. CloudFront 및 WAF 설정
8. 모니터링 스택 구축

---

## 📈 부하 테스트 결과

### K6 테스트 환경
- **가상 사용자**: 100명
- **테스트 기간**: 5분
- **시나리오**: POST (고객 정보 입력), GET (고객 정보 조회)

### 테스트 결과
- Auto Scaling이 정상 작동하여 부하에 따라 자동으로 인스턴스 증설
- 목표 응답시간 내 처리 확인
- 에러율 1% 미만 유지

---

## 💡 주요 학습 포인트

1. **ALB 구성의 복잡성**: External/Internal ALB의 포트 매핑 이해
2. **Private Subnet 운영**: NAT Gateway를 통한 아웃바운드 통신
3. **Auto Scaling 정책**: 동적 정책과 스케줄 기반 정책의 조합
4. **Prometheus 통합**: ASG 환경에서의 서비스 디스커버리
5. **비전공자 친화적 문서화**: 모든 설정을 단계별로 상세히 기록

---

## 📝 향후 개선 사항

- [ ] Terraform을 이용한 IaC 구현
- [ ] CI/CD 파이프라인 구축
- [ ] Blue-Green 배포 전략 도입
- [ ] 컨테이너화 (ECS/EKS) 검토
- [ ] 비용 최적화 추가 분석
- [ ] 재해 복구(DR) 전략 수립

---

## 🤝 기여

프로젝트에 대한 피드백이나 개선 제안은 언제든 환영합니다.

---

## 📄 라이선스

이 프로젝트는 학습 및 참고 목적으로 작성되었습니다.

---

## 📞 문의

프로젝트 관련 문의사항이 있으시면 Issues를 통해 연락해 주세요.
