# 🚀 확장 기능 구현

## 개요

본 문서는 기본 3-Tier 아키텍처를 넘어서 추가로 구현할 수 있는 확장 기능들을 다룹니다.

---

## 🎯 단기 개선 사항 (1~3개월)

### 1. 예측 스케일링 (Predictive Scaling)

#### 개요
머신러닝 기반으로 트래픽 패턴을 학습하여 사전에 리소스를 확장합니다.

#### 구현 방법
```python
# AWS Auto Scaling의 Predictive Scaling 정책
{
    "PredictiveScalingConfiguration": {
        "MetricSpecifications": [
            {
                "TargetValue": 50.0,
                "PredefinedLoadMetricSpecification": {
                    "PredefinedLoadMetricType": "ASGTotalCPUUtilization"
                },
                "PredefinedScalingMetricSpecification": {
                    "PredefinedScalingMetricType": "ASGAverageCPUUtilization"
                }
            }
        ],
        "Mode": "ForecastAndScale",  # 예측 및 실행
        "SchedulingBufferTime": 600   # 10분 버퍼
    }
}
```

#### 장점
- 트래픽 급증 전 사전 대응
- 부팅 시간을 고려한 여유 확보
- 사용자 경험 개선

---

### 2. 세션 클러스터링 (Session Clustering)

#### 개요
여러 WAS 인스턴스 간 세션 공유로 Sticky Session 불필요

#### ElastiCache Redis를 통한 세션 관리

```xml
<!-- Spring Session with Redis -->
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>io.lettuce</groupId>
    <artifactId>lettuce-core</artifactId>
</dependency>
```

```properties
# application.properties
spring.session.store-type=redis
spring.redis.host=petclinic-redis.xxxxx.ng.0001.apne2.cache.amazonaws.com
spring.redis.port=6379
spring.session.timeout=1800
```

#### 장점
- 인스턴스 재시작 시에도 세션 유지
- 로드 밸런싱 효율 향상
- Scale-in 시 사용자 세션 손실 없음

---

### 3. 데이터베이스 읽기 복제본 (Read Replica)

#### 구성
```
[Primary RDS] (Write)
    ↓ (Replication)
[Read Replica 1] (Read-only)
[Read Replica 2] (Read-only)
```

#### 애플리케이션 설정
```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    @Primary
    public DataSource writeDataSource() {
        // Primary DB (쓰기)
        return DataSourceBuilder.create()
            .url("jdbc:mysql://primary-endpoint:3306/petclinic")
            .build();
    }
    
    @Bean
    public DataSource readDataSource() {
        // Read Replica (읽기)
        return DataSourceBuilder.create()
            .url("jdbc:mysql://read-replica-endpoint:3306/petclinic")
            .build();
    }
}
```

---

### 4. CloudFront 캐싱 최적화

#### 캐시 정책
```json
{
    "CachePolicyConfig": {
        "Name": "petclinic-optimized-cache",
        "MinTTL": 1,
        "MaxTTL": 31536000,
        "DefaultTTL": 86400,
        "ParametersInCacheKeyAndForwardedToOrigin": {
            "EnableAcceptEncodingGzip": true,
            "EnableAcceptEncodingBrotli": true,
            "QueryStringsConfig": {
                "QueryStringBehavior": "whitelist",
                "QueryStrings": ["id", "category"]
            },
            "HeadersConfig": {
                "HeaderBehavior": "whitelist",
                "Headers": ["CloudFront-Viewer-Country"]
            }
        }
    }
}
```

#### Lambda@Edge로 동적 캐싱
```javascript
// Origin Request에서 쿠키 제거
exports.handler = async (event) => {
    const request = event.Records[0].cf.request;
    
    // 불필요한 쿠키 제거 (캐시 키 최적화)
    if (request.headers.cookie) {
        const cookies = request.headers.cookie[0].value
            .split(';')
            .filter(cookie => cookie.includes('essential_cookie'));
        
        if (cookies.length > 0) {
            request.headers.cookie[0].value = cookies.join(';');
        } else {
            delete request.headers.cookie;
        }
    }
    
    return request;
};
```

---

## 🔧 중기 개선 사항 (3~6개월)

### 1. Blue-Green 배포

#### 개요
무중단 배포를 위한 Blue-Green 전략 구현

#### 구성
```
[Current Environment - Blue]
    ALB Target Group 1 (100% 트래픽)

[New Environment - Green]
    ALB Target Group 2 (0% 트래픽)

배포 → 테스트 → 트래픽 전환 (0% → 100%)
```

#### 구현 스크립트
```bash
#!/bin/bash
# Blue-Green 배포 스크립트

# 1. Green 환경에 새 버전 배포
echo "Deploying to Green environment..."
aws autoscaling update-auto-scaling-group \
    --auto-scaling-group-name petclinic-asg-was-green \
    --launch-template LaunchTemplateName=petclinic-tmp-was-v2

# 2. 헬스체크 대기
echo "Waiting for health checks..."
aws elbv2 wait target-in-service \
    --target-group-arn $GREEN_TG_ARN

# 3. 스모크 테스트
echo "Running smoke tests..."
./smoke-tests.sh $GREEN_ALB_DNS

# 4. 트래픽 전환
echo "Switching traffic to Green..."
aws elbv2 modify-listener \
    --listener-arn $LISTENER_ARN \
    --default-actions Type=forward,TargetGroupArn=$GREEN_TG_ARN

# 5. 모니터링 (10분)
echo "Monitoring Green environment..."
sleep 600

# 6. 롤백 또는 완료
if [ $ERROR_RATE -gt 1 ]; then
    echo "Rolling back to Blue..."
    aws elbv2 modify-listener \
        --listener-arn $LISTENER_ARN \
        --default-actions Type=forward,TargetGroupArn=$BLUE_TG_ARN
else
    echo "Deployment successful!"
fi
```

---

### 2. CI/CD 파이프라인

#### GitHub Actions 워크플로우
```yaml
# .github/workflows/deploy.yml
name: Deploy to AWS

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          
      - name: Build with Maven
        run: mvn clean package
        
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-2
          
      - name: Upload to S3
        run: |
          aws s3 cp target/petclinic.war \
            s3://petclinic-artifacts/petclinic-${{ github.sha }}.war
          
      - name: Create AMI
        run: |
          # Packer로 새 AMI 생성
          packer build -var "war_version=${{ github.sha }}" packer.json
          
      - name: Update Launch Template
        run: |
          # 새 AMI로 Launch Template 업데이트
          aws ec2 create-launch-template-version \
            --launch-template-name petclinic-kr-tmp-was \
            --source-version 1 \
            --launch-template-data "ImageId=$NEW_AMI_ID"
          
      - name: Start Blue-Green Deployment
        run: ./scripts/blue-green-deploy.sh
```

---

### 3. 멀티 리전 재해 복구 (DR)

#### 아키텍처
```
[Primary Region: Seoul]
    ↓ (RDS Cross-Region Replica)
    ↓ (S3 Cross-Region Replication)
[DR Region: Tokyo]
```

#### Route 53 헬스체크 및 Failover
```json
{
    "HealthCheckConfig": {
        "Type": "HTTPS",
        "ResourcePath": "/health",
        "FullyQualifiedDomainName": "petclinic.example.com",
        "Port": 443,
        "RequestInterval": 30,
        "FailureThreshold": 3
    }
}
```

```json
{
    "ResourceRecordSet": {
        "Name": "petclinic.example.com",
        "Type": "A",
        "SetIdentifier": "Primary",
        "Failover": "PRIMARY",
        "AliasTarget": {
            "HostedZoneId": "Z1234567890ABC",
            "DNSName": "seoul-alb.amazonaws.com",
            "EvaluateTargetHealth": true
        },
        "HealthCheckId": "abc123"
    }
}
```

---

## 🚀 장기 개선 사항 (6개월 이상)

### 1. 컨테이너화 (ECS/EKS 전환)

#### EKS 아키텍처
```
[EKS Cluster]
  ├─ [Node Group 1 - Web Pods]
  │   └─ Nginx Ingress Controller
  ├─ [Node Group 2 - App Pods]
  │   └─ Spring Boot Application
  └─ [Node Group 3 - Data Pods]
      └─ Redis, Monitoring
```

#### Kubernetes Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: petclinic-was
spec:
  replicas: 3
  selector:
    matchLabels:
      app: petclinic-was
  template:
    metadata:
      labels:
        app: petclinic-was
    spec:
      containers:
      - name: petclinic
        image: petclinic/was:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 10
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: petclinic-was-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: petclinic-was
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

---

### 2. 서버리스 아키텍처 일부 도입

#### API Gateway + Lambda
```
[CloudFront]
    ↓
[API Gateway]
    ├─ /api/products → [Lambda: Product Service]
    ├─ /api/orders → [Lambda: Order Service]
    └─ /api/customers → [Lambda: Customer Service]
         ↓
    [DynamoDB / Aurora Serverless]
```

#### 장점
- 완전한 자동 확장
- 사용한 만큼만 과금
- 운영 부담 최소화
- Cold Start 고려 필요

---

### 3. 마이크로서비스 전환

#### 서비스 분리
```
[API Gateway]
    ├─ Product Service (독립 DB)
    ├─ Order Service (독립 DB)
    ├─ Customer Service (독립 DB)
    ├─ Inventory Service (독립 DB)
    └─ Notification Service (Event-driven)
```

#### Event-Driven 통신
```
[Order Service]
    ↓ (주문 생성 이벤트)
[EventBridge / SNS]
    ↓
[Lambda Functions]
    ├─ 재고 차감 (Inventory)
    ├─ 이메일 발송 (Notification)
    └─ 분석 데이터 저장 (Analytics)
```

---

## 🔐 보안 강화

### 1. AWS Secrets Manager 통합
```python
import boto3
import json

def get_db_credentials():
    secret_name = "petclinic/db/credentials"
    region_name = "ap-northeast-2"
    
    session = boto3.session.Session()
    client = session.client(
        service_name='secretsmanager',
        region_name=region_name
    )
    
    response = client.get_secret_value(SecretId=secret_name)
    secret = json.loads(response['SecretString'])
    
    return secret['username'], secret['password']
```

### 2. AWS KMS를 통한 암호화
```bash
# S3 버킷 기본 암호화
aws s3api put-bucket-encryption \
    --bucket petclinic-static-content \
    --server-side-encryption-configuration '{
        "Rules": [{
            "ApplyServerSideEncryptionByDefault": {
                "SSEAlgorithm": "aws:kms",
                "KMSMasterKeyID": "arn:aws:kms:..."
            }
        }]
    }'
```

---

## 📊 고급 모니터링

### 1. X-Ray를 통한 분산 추적
```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-xray-recorder-sdk-spring</artifactId>
</dependency>
```

```java
@Configuration
public class XRayConfig {
    @Bean
    public Filter TracingFilter() {
        return new AWSXRayServletFilter("PetClinic");
    }
}
```

### 2. CloudWatch ServiceLens
- X-Ray + CloudWatch Logs + Metrics 통합
- 엔드투엔드 요청 추적
- 서비스 맵 자동 생성

---

## 📝 구현 우선순위

### High Priority (즉시)
1. ✅ 세션 클러스터링 (ElastiCache Redis)
2. ✅ 데이터베이스 Read Replica
3. ✅ CloudFront 캐싱 최적화

### Medium Priority (3개월)
4. ⏳ Blue-Green 배포
5. ⏳ CI/CD 파이프라인
6. ⏳ 멀티 리전 DR

### Low Priority (6개월+)
7. 📅 컨테이너화 (EKS)
8. 📅 서버리스 일부 도입
9. 📅 마이크로서비스 전환

---

## 🎯 성공 지표

각 기능 구현 후 다음 지표로 성과를 측정합니다:

| 기능 | 측정 지표 | 목표 |
|------|----------|------|
| 예측 스케일링 | Scale-out 지연 시간 | < 1분 |
| 세션 클러스터링 | 세션 손실률 | 0% |
| Read Replica | 읽기 쿼리 응답 시간 | 30% 개선 |
| Blue-Green | 배포 중 다운타임 | 0초 |
| CI/CD | 배포 소요 시간 | < 15분 |

---

## 📚 참고 자료

- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS 서버리스 애플리케이션 렌즈](https://docs.aws.amazon.com/wellarchitected/latest/serverless-applications-lens/)
- [Kubernetes 패턴](https://k8spatterns.io/)
- [마이크로서비스 패턴](https://microservices.io/patterns/)
- [12 Factor App](https://12factor.net/)
