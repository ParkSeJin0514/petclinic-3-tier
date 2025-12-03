# 🔥 SLO 정책

## 프로젝트 정보
- **시작일**: 2025년 10월 16일  
- **종료일**: 2025년 11월 14일
- **상태**: Completed

---

## 📋 SLO (Service Level Objective)란?

### 정의
서비스가 달성해야 할 **구체적이고 측정 가능한 목표**로, 사용자 경험을 정량화한 지표입니다.

### 왜 SLO가 중요한가?
1. **명확한 목표**: 팀이 추구해야 할 서비스 품질 기준
2. **오류 예산**: 얼마나 많은 실패를 허용할 수 있는지 정량화
3. **우선순위 결정**: 기능 개발 vs 안정성 개선의 균형
4. **객관적 평가**: 서비스 상태를 객관적으로 평가

---

## 🎯 트래픽 가정 및 오류 예산 계산

### 1) 트래픽 가정 (동접 1,000명 → RPS 환산)

동시 접속자만으로는 정확한 요청량 계산이 어렵기 때문에, 사용자당 요청 빈도를 가정하여 RPS(초당 요청 수)를 계산합니다.

#### 시나리오별 계산

| 시나리오 | 인당 요청빈도 | RPS (초당요청) | 30일 총 요청 | SLO 99.9% 오류예산 | SLO 99.95% 오류예산 |
|----------|--------------|---------------|--------------|-------------------|---------------------|
| **보수적** | 1 req / 10s | 100 | 259,200,000 | 259,200 (0.1%) | 129,600 (0.05%) |
| **보통** | 1 req / 5s | 200 | 518,400,000 | 518,400 (0.1%) | 259,200 (0.05%) |

#### 계산 근거
```
보수적 시나리오:
- 동접: 1,000명
- 요청 빈도: 1회 / 10초
- RPS: 1,000명 × (1회/10초) = 100 RPS
- 1일 요청: 100 × 60 × 60 × 24 = 8,640,000
- 30일 요청: 8,640,000 × 30 = 259,200,000

99.9% SLO 오류예산:
- 259,200,000 × 0.001 = 259,200 요청
```

### 운영 팁
처음에는 **99.9% SLO**로 시작하여 오탐과 잡음을 완충하고, 시스템이 안정화되면 **99.95%**로 상향하는 것이 현실적입니다.

---

## 📊 2) 제안하는 SLO (30일 롤링 기준)

### 핵심 SLO 지표

#### 1. 가용성 (Availability)
**목표**: 99.9%

- **의미**: 1000번 요청 중 999번은 성공적으로 처리
- **오류 예산**: 0.1% (30일 기준 약 43분의 다운타임 허용)
- **측정 방법**: 성공 응답(2xx, 3xx) / 전체 요청

#### 2. 지연시간 (Latency)
**목표**: p95 응답시간 ≤ 400ms

- **의미**: 95%의 요청이 400ms 이내에 응답
- **초점**: 장바구니 및 결제 과정에 중점
- **측정 방법**: ALB TargetResponseTime p95

#### 3. 체크아웃 성공률 (Business SLO)
**목표**: ≥ 98.5%

- **의미**: 100건의 결제 시도 중 98.5건 이상 성공
- **비즈니스 영향**: 직접적인 매출 영향
- **측정 방법**: 성공한 결제 / 시도한 결제

---

## 📈 3) SLI (Service Level Indicator) 원천

### SLI란?
SLO를 측정하기 위한 **실제 메트릭**입니다.

### ALB 기준 (권장)

#### 주요 메트릭

| SLI | CloudWatch 메트릭 | 설명 |
|-----|------------------|------|
| **요청수** | `AWS/ApplicationELB` → `RequestCount` (Sum) | 전체 요청 수 |
| **ELB 오류** | `HTTPCode_ELB_5XX_Count` (Sum) | ALB 레벨 5XX 에러 |
| **타겟 오류** | `HTTPCode_Target_5XX_Count` (Sum) | 백엔드 5XX 에러 |
| **응답시간** | `TargetResponseTime` (p95) | 타겟 응답 시간 95백분위 |

#### 4XX 오류 처리
- `HTTPCode_Target_4XX_Count` / `HTTPCode_ELB_4XX_Count`
- 4xx는 클라이언트 오류이므로 SLO에 포함할지는 **정책에 따라 결정**
- 권장: 비즈니스 로직 오류(400, 404)는 제외, 인증 오류(401, 403)는 포함 고려

### CloudFront 기준 (선택적 병행)

| SLI | CloudWatch 메트릭 | 설명 |
|-----|------------------|------|
| **총 에러율** | `AWS/CloudFront` → `TotalErrorRate` (Average, %) | 전체 에러 비율 |
| **요청수** | `Requests` (Sum) | CloudFront 요청 수 |
| **지연시간** | `OriginLatency` 또는 `TotalTime` (p95) | 오리진 또는 전체 지연 |

### 업무 SLI (체크아웃 성공률)

커스텀 메트릭으로 구현:
- **방법 1**: CloudWatch EMF (Embedded Metric Format)
- **방법 2**: 애플리케이션 로그 → CloudWatch Logs Insights
- **임시 대안**: ALB 2XX 응답을 대리 지표로 사용

```python
# Python 예제 - EMF로 커스텀 메트릭 전송
import json

def log_checkout_metric(success):
    metric_data = {
        "_aws": {
            "Timestamp": int(time.time() * 1000),
            "CloudWatchMetrics": [{
                "Namespace": "PetClinic/Business",
                "Dimensions": [["Environment"]],
                "Metrics": [
                    {"Name": "CheckoutAttempts", "Unit": "Count"},
                    {"Name": "CheckoutSuccess", "Unit": "Count"}
                ]
            }]
        },
        "Environment": "production",
        "CheckoutAttempts": 1,
        "CheckoutSuccess": 1 if success else 0
    }
    print(json.dumps(metric_data))
```

---

## 🧮 4) SLI 수식

### 가용성 (Availability %)

```
availability_pct = 100 * (RequestCount - (ELB_5XX + Target_5XX)) / max(RequestCount, 1)
```

**Grafana PromQL 예시**:
```promql
100 * (
    sum(increase(cloudwatch_aws_application_elb_request_count_sum[30d])) -
    (sum(increase(cloudwatch_aws_application_elb_http_code_elb_5xx_count_sum[30d])) +
     sum(increase(cloudwatch_aws_application_elb_http_code_target_5xx_count_sum[30d])))
) / 
sum(increase(cloudwatch_aws_application_elb_request_count_sum[30d]))
```

### 지연시간 p95 (ms)

```
latency_p95_ms = 1000 * p95(TargetResponseTime)
```

**Grafana Query**:
```
1000 * cloudwatch_aws_application_elb_target_response_time_p95
```

### 체크아웃 성공률 (%)

```
checkout_success_pct = 100 * CheckoutSuccess / max(CheckoutAttempts, 1)
```

**Grafana Query**:
```promql
100 * 
sum(increase(petclinic_checkout_success[30d])) /
sum(increase(petclinic_checkout_attempts[30d]))
```

### 오류율 및 번레이트 (Burn Rate)

```
error_rate = (ELB_5XX + Target_5XX) / max(RequestCount, 1)
burn_rate = error_rate / error_budget

# 예: SLO 99.9% (error_budget = 0.001)
burn_rate = error_rate / 0.001
```

**번레이트 해석**:
- `burn_rate = 1`: 정확히 오류 예산 소비 속도
- `burn_rate > 1`: 오류 예산을 빠르게 소비 중 (경고!)
- `burn_rate < 1`: 오류 예산 내에서 안전

---

## 📊 5) Grafana 대시보드 구성

### 상단: SLO 카드 (Stat 패널) 3개

#### 패널 1: 가용성 (30일)

**Query A - ALB RequestCount**:
```
Metric: AWS/ApplicationELB - RequestCount
Stat: Sum
Period: 60s (또는 300s)
Dimensions: LoadBalancer, TargetGroup
```

**Query B - ALB HTTPCode_ELB_5XX_Count**:
```
Metric: AWS/ApplicationELB - HTTPCode_ELB_5XX_Count
Stat: Sum
Period: 60s
```

**Query C - ALB HTTPCode_Target_5XX_Count**:
```
Metric: AWS/ApplicationELB - HTTPCode_Target_5XX_Count
Stat: Sum
Period: 60s
```

**Expression E**:
```
100 * (A - (B + C)) / max(A, 1)
```

**패널 설정**:
- Panel time override: **Last 30 days**
- Unit: **Percent (0-100)**
- Threshold: 
  - Green: ≥ 99.9
  - Orange: 99.5 - 99.9
  - Red: < 99.5

---

#### 패널 2: ALB p95 응답시간 (30일)

**Query D - TargetResponseTime**:
```
Metric: AWS/ApplicationELB - TargetResponseTime
Stat: p95
Period: 60s
```

**Expression F**:
```
1000 * D  # 초를 밀리초로 변환
```

**패널 설정**:
- Panel time override: **Last 30 days**
- Unit: **milliseconds (ms)**
- Threshold:
  - Green: ≤ 400
  - Orange: 400 - 600
  - Red: > 600

---

#### 패널 3: 체크아웃 성공률 (30일)

**Query X - CheckoutAttempts**:
```
Namespace: PetClinic/Business
Metric: CheckoutAttempts
Stat: Sum
Period: 60s
```

**Query Y - CheckoutSuccess**:
```
Namespace: PetClinic/Business
Metric: CheckoutSuccess
Stat: Sum
Period: 60s
```

**Expression Z**:
```
100 * Y / max(X, 1)
```

**패널 설정**:
- Panel time override: **Last 30 days**
- Unit: **Percent (0-100)**
- Threshold:
  - Green: ≥ 98.5
  - Orange: 97 - 98.5
  - Red: < 97

---

### 중간: 오류 예산 소진 추이 (Time Series)

```promql
# 오류 예산 남은 비율
100 * (1 - (
    sum(increase(errors[30d])) / 
    (sum(increase(requests[30d])) * 0.001)
))
```

**패널 설정**:
- Y축: 0-100%
- 기준선: 0% (예산 소진 완료)
- 색상: 녹색 → 노란색 → 빨간색

---

### 하단: 상세 메트릭

#### 1. 요청량 추이
```
sum(rate(requests[5m])) by (tier)
```

#### 2. 에러율 추이
```
sum(rate(errors[5m])) / sum(rate(requests[5m]))
```

#### 3. 응답 시간 분포 (p50, p95, p99)
```
histogram_quantile(0.50, rate(response_time_bucket[5m]))
histogram_quantile(0.95, rate(response_time_bucket[5m]))
histogram_quantile(0.99, rate(response_time_bucket[5m]))
```

---

## 🚨 6) SLO 기반 알림 정책

### Multi-Window Multi-Burn-Rate Alerts

Google SRE 워크북의 권장 방법으로, 여러 시간 창과 번레이트를 조합하여 알림합니다.

#### Alert 1: 빠른 소진 (Page)
```yaml
- alert: ErrorBudgetFastBurn
  expr: |
    (
      sum(rate(errors[1h])) / sum(rate(requests[1h])) > 0.014
    and
      sum(rate(errors[5m])) / sum(rate(requests[5m])) > 0.014
    )
  for: 2m
  labels:
    severity: critical
    page: true
  annotations:
    summary: "Error budget burning too fast"
    description: "At current rate, 2% of monthly budget will be consumed in 1 hour"
```

**설명**:
- 1시간 window: 14× burn rate (0.001 × 14 = 0.014)
- 5분 window: 빠른 변화 감지
- 2분 for: False positive 감소

#### Alert 2: 중간 소진 (Ticket)
```yaml
- alert: ErrorBudgetModerateBurn
  expr: |
    (
      sum(rate(errors[6h])) / sum(rate(requests[6h])) > 0.006
    and
      sum(rate(errors[30m])) / sum(rate(requests[30m])) > 0.006
    )
  for: 15m
  labels:
    severity: warning
    page: false
  annotations:
    summary: "Error budget burning moderately"
    description: "At current rate, 5% of monthly budget will be consumed in 6 hours"
```

#### Alert 3: 느린 소진 (Monitoring)
```yaml
- alert: ErrorBudgetSlowBurn
  expr: |
    (
      sum(rate(errors[3d])) / sum(rate(requests[3d])) > 0.001
    )
  for: 1h
  labels:
    severity: info
    page: false
  annotations:
    summary: "Error budget depleting"
    description: "Current error rate exactly at SLO threshold over 3 days"
```

---

## 📋 7) SLO 운영 프로세스

### 주간 리뷰
- [ ] 현재 SLO 달성률 확인
- [ ] 오류 예산 소진율 점검
- [ ] 주요 장애 및 원인 분석
- [ ] 다음 주 리스크 요인 식별

### 월간 리뷰
- [ ] 30일 SLO 달성 여부 평가
- [ ] SLO 목표 적정성 검토
- [ ] 오류 예산 기반 우선순위 조정
- [ ] 다음 달 목표 및 계획 수립

### 분기 리뷰
- [ ] SLO 목표 재설정 검토
- [ ] 사용자 피드백과 SLO 상관관계 분석
- [ ] 측정 방법론 개선
- [ ] 장기 트렌드 분석

---

## 🎯 8) 오류 예산 사용 원칙

### 예산이 남은 경우 (> 50%)
**적극적 혁신 모드**
- 새로운 기능 개발 가속
- 공격적인 배포 주기
- 실험적 기능 테스트
- 성능 최적화 실험

### 예산이 부족한 경우 (< 25%)
**안정성 우선 모드**
- 새로운 기능 개발 중단/연기
- 안정성 개선에 집중
- 배포 동결 또는 제한
- 모니터링 강화
- Post-mortem 및 근본 원인 분석

### 예산이 고갈된 경우 (0%)
**긴급 대응 모드**
- 모든 비필수 변경 중단
- 24/7 On-call 강화
- 경영진 에스컬레이션
- 고객 커뮤니케이션
- 복구 계획 실행

---

## 📝 SLO 체크리스트

### 설정 단계
- [x] SLO 목표 정의 (99.9%)
- [x] SLI 메트릭 선정 (ALB, CloudWatch)
- [x] 측정 기간 결정 (30일 롤링)
- [x] 오류 예산 계산
- [x] Grafana 대시보드 구축
- [x] 알림 규칙 설정

### 운영 단계
- [ ] 일일 SLO 모니터링
- [ ] 주간 SLO 리뷰
- [ ] 월간 SLO 평가
- [ ] 분기별 목표 재설정
- [ ] 지속적 개선

---

## 🔄 다음 단계

SLO 정책 수립이 완료되었으므로 다음 단계를 진행합니다:

1. ✅ SLO 목표 정의 완료
2. ✅ SLI 메트릭 설정 완료
3. ✅ 오류 예산 정책 수립 완료
4. ✅ Grafana 대시보드 구성 완료
5. → 지속적 모니터링 및 개선
6. → [확장 기능 구현](./07-advanced-features.md)

---

## 📚 참고 자료

- [Google SRE Book - SLO 챕터](https://sre.google/sre-book/service-level-objectives/)
- [Google SRE Workbook - SLO 구현](https://sre.google/workbook/implementing-slos/)
- [Alex Hidalgo - Implementing Service Level Objectives](https://www.alex-hidalgo.com/)
- [AWS Well-Architected Framework - 신뢰성](https://aws.amazon.com/architecture/well-architected/)
