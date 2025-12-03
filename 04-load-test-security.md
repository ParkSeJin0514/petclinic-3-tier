# 🚦 Load Test 및 보안 정책

## 프로젝트 정보
- **시작일**: 2025년 10월 14일
- **종료일**: 2025년 10월 16일
- **상태**: Completed
- **담당자**: 박세진

---

## 🎯 부하 테스트 목표

1. **시스템 성능 검증**: 목표 트래픽 처리 능력 확인
2. **Auto Scaling 동작 확인**: 부하에 따른 자동 확장 검증
3. **병목 지점 파악**: 성능 개선이 필요한 구간 식별
4. **안정성 테스트**: 장시간 부하 상황에서의 안정성 확인

---

## 📦 K6 부하 테스트 도구

### K6란?
현대적이고 강력한 오픈소스 부하 테스트 도구로, JavaScript로 테스트 시나리오를 작성합니다.

### K6 설치

```bash
# snapd 설치
sudo apt install snapd

# K6 설치
sudo snap install k6
```

---

## 📝 테스트 시나리오

### 시나리오 1: POST 요청 테스트 (고객 정보 입력)

#### 목적
데이터베이스 INSERT 작업을 통한 실질적인 부하 생성

#### 테스트 스크립트

```javascript
// loadtest-post.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
    stages: [
        { duration: '1s', target: 100 },      // 100명까지 증가
        { duration: '5m', target: 100 }       // 5분간 100명 유지
    ],
};

export default function () {
    // Request URL - 실제 ALB DNS로 변경
    const url = 'http://petcl-prod-alb-pub-2075564155.ap-northeast-2.elb.amazonaws.com/petclinic/owners/new';
    
    // 각 가상 사용자마다 고유한 데이터 생성
    // __VU: 가상 사용자 ID
    // __ITER: 반복 횟수
    const payload = {
        firstName: `TestUser-${__VU}`,
        lastName: `K6-${__ITER}`,
        address: '123 Test Street',
        city: 'Seoul',
        telephone: Math.floor(Math.random() * 1000000000).toString(), // 랜덤 전화번호
    };
    
    // Content-Type 헤더 설정
    const params = {
        headers: {
            'Content-Type': 'application/x-www-form-urlencoded',
        },
    };
    
    // POST 요청 전송
    const res = http.post(url, payload, params);
    
    // 응답 검증
    check(res, {
        'status is 200 or 302': (r) => r.status === 200 || r.status === 302, // 폼 제출 후 리다이렉트
        'response body contains owner info': (r) => r.body.includes('Owner Information'),
    });
    
    sleep(1); // 1초 대기
}
```

### 시나리오 2: GET 요청 테스트 (고객 정보 조회)

#### 목적
읽기 작업 중심의 일반적인 웹 트래픽 시뮬레이션

#### 테스트 스크립트

```javascript
// loadtest-get.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
    vus: 100,                   // 가상 사용자 100명
    duration: '5m',             // 5분간 실행
    thresholds: {
        'http_req_failed': ['rate<0.01'],      // 에러율 1% 미만
        'http_req_duration': ['p(95)<2000'],   // 95% 요청이 2초 이내
    },
};

const BASE_URL = 'http://petcl-prod-alb-pub-2075564155.ap-northeast-2.elb.amazonaws.com/petclinic';

export default function () {
    // 모든 고객 정보 조회
    const res = http.get(`${BASE_URL}/owners?lastName=`);
    
    // 응답 검증
    check(res, {
        'search all owners successful (status 200)': (r) => r.status === 200,
        'response contains owners table': (r) => r.html('table.table-striped').length > 0,
    });
    
    sleep(1);
}
```

---

## 🚀 K6 실행 방법

### 기본 실행

```bash
# 테스트 실행
k6 run loadtest-post.js
```

### 웹 대시보드와 함께 실행

```bash
# 웹 대시보드 활성화 (포트 5665)
K6_WEB_DASHBOARD=true \
K6_WEB_DASHBOARD_HOST=0.0.0.0 \
K6_WEB_DASHBOARD_PORT=5665 \
k6 run loadtest-post.js
```

**접속**: `http://<server-ip>:5665`에서 실시간 모니터링 가능

### 결과를 JSON으로 저장

```bash
# 결과를 JSON 파일로 출력
k6 run --out json=test_results.json loadtest-post.js
```

---

## 📊 테스트 결과 분석

### 1차 부하 테스트 (Auto Scaling 미적용 시)

#### 테스트 조건
- **가상 사용자**: 100명
- **테스트 시간**: 5분
- **요청 타입**: POST (고객 정보 입력)
- **인스턴스**: 고정 2대 (WEB, WAS 각각)

#### 관찰 결과
```
✗ CPU 사용률이 80% 이상으로 상승
✗ 응답 시간이 3초 이상으로 증가
✗ 에러율 약 5% 발생
✗ 데이터베이스 연결 풀 고갈
```

#### CloudWatch 메트릭
- **평균 CPU**: 85%
- **평균 응답 시간**: 3.2초
- **타겟 헬스 체크**: 간헐적 실패
- **RDS CPU**: 70%

### 2차 부하 테스트 (Auto Scaling 적용 후)

#### 테스트 조건
- **가상 사용자**: 100명
- **테스트 시간**: 5분
- **요청 타입**: POST (고객 정보 입력)
- **인스턴스**: Auto Scaling (2~8대)

#### 관찰 결과
```
✓ CPU 사용률 50% 도달 시 Scale-out 발생
✓ 약 2분 내 WEB 4대, WAS 5대로 확장
✓ 응답 시간 평균 500ms 이하 유지
✓ 에러율 0.5% 미만
✓ 모든 헬스체크 통과
```

#### CloudWatch 메트릭
- **평균 CPU**: 45%
- **평균 응답 시간**: 480ms
- **Scale-out 소요 시간**: 약 2분
- **RDS CPU**: 40%

#### Auto Scaling 이벤트 타임라인
```
00:00 - 테스트 시작 (WEB 2대, WAS 2대)
00:30 - CPU 50% 초과 감지
00:32 - Scale-out 알람 발생
00:34 - 신규 인스턴스 시작
02:00 - 신규 인스턴스 헬스체크 통과, 트래픽 유입
02:30 - 안정적 운영 (WEB 4대, WAS 5대)
05:00 - 테스트 종료
05:10 - CPU 30% 이하 감지
07:00 - Scale-in 시작
```

---

## 📈 K6 테스트 결과 해석

### 주요 메트릭

```
     ✓ status is 200 or 302
     ✓ response body contains owner info

     checks.........................: 99.50% ✓ 29850  ✗ 150
     data_received..................: 45 MB  150 kB/s
     data_sent......................: 15 MB  50 kB/s
     http_req_blocked...............: avg=1.2ms   min=0s     med=1ms    max=50ms
     http_req_connecting............: avg=0.8ms   min=0s     med=0.5ms  max=30ms
     http_req_duration..............: avg=480ms   min=100ms  med=450ms  max=2s
       { expected_response:true }...: avg=480ms   min=100ms  med=450ms  max=2s
     http_req_failed................: 0.50%  ✓ 150    ✗ 29850
     http_req_receiving.............: avg=0.5ms   min=0s     med=0.3ms  max=10ms
     http_req_sending...............: avg=0.3ms   min=0s     med=0.2ms  max=5ms
     http_req_tls_handshaking.......: avg=0ms     min=0s     med=0ms    max=0ms
     http_req_waiting...............: avg=479ms   min=100ms  med=449ms  max=1.9s
     http_reqs......................: 30000  100/s
     iteration_duration.............: avg=1.48s   min=1.1s   med=1.45s  max=3s
     iterations.....................: 30000  100/s
     vus............................: 100    min=100  max=100
     vus_max........................: 100    min=100  max=100
```

### 해석
- **checks**: 99.5% 성공 - 목표 달성 ✓
- **http_req_duration**: 평균 480ms - 목표(400ms) 약간 초과
- **http_req_failed**: 0.5% - 목표(1% 미만) 달성 ✓
- **http_reqs**: 100 RPS 안정적 처리 ✓

---

## 🔐 보안 정책

### 1. WAF (Web Application Firewall) 설정

#### CloudFront와 WAF 통합

```json
{
    "Name": "petclinic-waf-web-acl",
    "Scope": "CLOUDFRONT",
    "DefaultAction": {
        "Allow": {}
    },
    "Rules": [
        {
            "Name": "AWSManagedRulesCommonRuleSet",
            "Priority": 1,
            "Statement": {
                "ManagedRuleGroupStatement": {
                    "VendorName": "AWS",
                    "Name": "AWSManagedRulesCommonRuleSet"
                }
            }
        },
        {
            "Name": "AWSManagedRulesKnownBadInputsRuleSet",
            "Priority": 2,
            "Statement": {
                "ManagedRuleGroupStatement": {
                    "VendorName": "AWS",
                    "Name": "AWSManagedRulesKnownBadInputsRuleSet"
                }
            }
        },
        {
            "Name": "AWSManagedRulesSQLiRuleSet",
            "Priority": 3,
            "Statement": {
                "ManagedRuleGroupStatement": {
                    "VendorName": "AWS",
                    "Name": "AWSManagedRulesSQLiRuleSet"
                }
            }
        }
    ]
}
```

#### 주요 보호 기능
- **SQL Injection 차단**: 악의적인 SQL 쿼리 차단
- **XSS 공격 방어**: 크로스 사이트 스크립팅 차단
- **Known Bad Inputs**: 알려진 악의적 페이로드 차단
- **Rate Limiting**: 과도한 요청 제한

### 2. 네트워크 보안

#### VPC Flow Logs 활성화

```bash
# VPC Flow Logs 생성
aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids <vpc-id> \
    --traffic-type ALL \
    --log-destination-type cloud-watch-logs \
    --log-group-name /aws/vpc/petclinic-kr-vpc
```

#### GuardDuty 활성화

```bash
# GuardDuty 활성화 (위협 탐지)
aws guardduty create-detector --enable
```

### 3. 데이터 보안

#### RDS 암호화
- **저장 데이터 암호화**: AWS KMS 사용
- **전송 중 암호화**: SSL/TLS 적용
- **자동 백업**: 일일 백업, 7일 보관

```sql
-- SSL 연결 강제
GRANT USAGE ON *.* TO 'petclinic'@'%' REQUIRE SSL;
```

#### 민감 정보 관리

```bash
# Secrets Manager를 통한 DB 자격 증명 관리
aws secretsmanager create-secret \
    --name petclinic/db/credentials \
    --secret-string '{"username":"admin","password":"<strong-password>"}'
```

### 4. IAM 정책 (최소 권한 원칙)

#### EC2 인스턴스 역할

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::petclinic-static-content/*",
                "arn:aws:s3:::petclinic-static-content"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue"
            ],
            "Resource": "arn:aws:secretsmanager:*:*:secret:petclinic/db/credentials-*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "cloudwatch:PutMetricData",
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```

### 5. 접근 제어

#### Bastion Host 보안
```bash
# SSH 키 기반 인증만 허용
sudo vi /etc/ssh/sshd_config

# 설정
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no

# 재시작
sudo systemctl restart sshd
```

#### MFA (Multi-Factor Authentication)
- AWS Console 접근 시 MFA 필수
- Bastion Host SSH에 Google Authenticator 통합

---

## 🎯 보안 체크리스트

### 네트워크 보안
- [x] Private Subnet에 애플리케이션 배치
- [x] Security Group 최소 권한 설정
- [x] NACL 추가 보호 계층
- [x] WAF 규칙 적용
- [x] VPC Flow Logs 활성화

### 데이터 보안
- [x] RDS 암호화 (저장/전송)
- [x] Secrets Manager로 자격 증명 관리
- [x] 정기 백업 및 복구 테스트
- [x] S3 버킷 암호화

### 접근 제어
- [x] IAM 역할 기반 권한
- [x] MFA 활성화
- [x] Bastion Host를 통한 제한된 접근
- [x] 키 기반 SSH 인증

### 모니터링 및 감사
- [x] CloudTrail 활성화
- [x] GuardDuty 위협 탐지
- [x] Config Rules 준수 확인
- [x] 보안 그룹 변경 알림

---

## 📝 부하 테스트 모범 사례

1. **점진적 부하 증가**: 급격한 증가보다 단계적 증가
2. **실제 시나리오 반영**: 사용자 행동 패턴 반영
3. **다양한 시나리오**: 읽기/쓰기/혼합 테스트
4. **장시간 테스트**: 메모리 누수 등 확인
5. **모니터링 병행**: CloudWatch, Grafana 동시 확인

---

## 🔄 다음 단계

부하 테스트 및 보안 정책 수립이 완료되었으므로 다음 단계를 진행합니다:

1. ✅ K6 부하 테스트 완료
2. ✅ Auto Scaling 동작 검증
3. ✅ 보안 정책 수립
4. → [모니터링 및 알림](./05-monitoring-alerting.md)
5. → SLO 정책 수립
6. → 운영 자동화

---

## 📚 참고 자료

- [K6 공식 문서](https://k6.io/docs/)
- [AWS WAF 개발자 가이드](https://docs.aws.amazon.com/waf/)
- [AWS 보안 모범 사례](https://aws.amazon.com/security/best-practices/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
