# 📊 모니터링 및 알림

## 프로젝트 정보
- **시작일**: 2025년 10월 15일
- **종료일**: 2025년 10월 17일
- **상태**: Completed
- **담당자**: 박세진

---

## 🎯 모니터링 목표

1. **실시간 시스템 상태 파악**: 서비스 건강도 실시간 모니터링
2. **성능 병목 지점 식별**: 리소스 사용률 및 응답 시간 추적
3. **사전 장애 감지**: 임계값 기반 알림으로 장애 예방
4. **용량 계획**: 트렌드 분석을 통한 리소스 계획

---

## 🏗️ 모니터링 아키텍처

```
[EC2 Instances (ASG)]
       ↓ (메트릭 수집)
[Node Exporter :9100]
       ↓ (스크래핑)
[Prometheus :9090]
       ↓ (데이터 소스)
[Grafana Cloud]
       ↓ (시각화 및 알림)
[Slack / Email]
```

### 구성 요소
- **Node Exporter**: EC2 인스턴스 시스템 메트릭 수집
- **Prometheus**: 메트릭 저장 및 쿼리
- **Grafana**: 시각화 및 대시보드
- **CloudWatch**: AWS 네이티브 모니터링

---

## 📦 Node Exporter 설치 및 설정

### Node Exporter란?
Prometheus를 위한 하드웨어 및 OS 메트릭 수집기로, CPU, 메모리, 디스크, 네트워크 등의 시스템 메트릭을 제공합니다.

### User Data 스크립트로 자동 설치

Launch Template의 User Data에 다음 스크립트를 추가하여 인스턴스 시작 시 자동 설치합니다.

```bash
#!/bin/bash

# 1. 시스템 업데이트 및 필수 패키지 설치
echo "시스템 업데이트 및 설치 시작..."
sudo yum update -y
sudo yum install -y wget

# 2. Node Exporter 설정 변수
# 최신 버전을 확인하여 사용하세요
NODE_EXPORTER_VERSION="1.7.0"
TAR_FILE="node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz"
DOWNLOAD_URL="https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/${TAR_FILE}"

# 3. Node Exporter 다운로드 및 압축 해제
echo "Node Exporter 다운로드 및 압축 해제 시작..."
cd /tmp
wget "${DOWNLOAD_URL}"
tar xvfz "${TAR_FILE}"

# 4. 바이너리 파일 이동 및 권한 설정
sudo cp node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64/node_exporter /usr/local/bin/
sudo chown root:root /usr/local/bin/node_exporter
sudo rm -rf node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64*

# 5. Node Exporter 전용 사용자 생성
sudo useradd -rs /bin/false node_exporter

# 6. systemd 서비스 파일 생성
cat << EOF | sudo tee /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

# 7. 서비스 시작 및 활성화
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter

echo "Node Exporter 서비스가 성공적으로 시작되었습니다."
```

### 수동 설치 (필요 시)

```bash
# 다운로드
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz

# 압축 해제
tar xvfz node_exporter-1.7.0.linux-amd64.tar.gz

# 바이너리 복사
sudo cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/

# 사용자 생성
sudo useradd -rs /bin/false node_exporter

# 서비스 파일 생성 (위와 동일)
# ...

# 서비스 시작
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

### Node Exporter 확인

```bash
# 서비스 상태 확인
sudo systemctl status node_exporter

# 메트릭 확인
curl http://localhost:9100/metrics

# 특정 메트릭 필터
curl http://localhost:9100/metrics | grep node_cpu
```

---

## 🔍 Prometheus 설치 및 설정

### Prometheus 설치

**별도의 EC2 인스턴스**에 Prometheus 서버를 구축합니다.

```bash
# 1. Prometheus 버전 지정
VERSION="2.50.0"
TAR_FILE="prometheus-${VERSION}.linux-amd64.tar.gz"

# 2. 다운로드
cd /tmp
echo "Downloading Prometheus v${VERSION}..."
wget "https://github.com/prometheus/prometheus/releases/download/v${VERSION}/${TAR_FILE}"

# 3. 압축 해제
echo "Extracting files..."
tar xvfz "${TAR_FILE}"

# 4. 디렉토리 이동
cd prometheus-${VERSION}.linux-amd64/

# 5. 바이너리 복사
echo "Copying binaries to /usr/local/bin/..."
sudo cp prometheus /usr/local/bin/
sudo cp promtool /usr/local/bin/

# 6. 디렉토리 생성
sudo mkdir -p /var/lib/prometheus
sudo mkdir -p /etc/prometheus

# 7. 설정 파일 복사
sudo cp -r consoles /etc/prometheus
sudo cp -r console_libraries /etc/prometheus

# 8. 사용자 생성
sudo useradd --no-create-home --shell /bin/false prometheus

# 9. 권한 설정
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
```

### Prometheus 설정 파일

```yaml
# /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'petclinic-kr'
    environment: 'production'

# Alertmanager 설정 (옵션)
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - 'localhost:9093'

# 규칙 파일 로드
rule_files:
  - '/etc/prometheus/rules/*.yml'

# 스크래핑 대상
scrape_configs:
  # Prometheus 자체 모니터링
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # WEB Tier Node Exporters
  - job_name: 'web-tier'
    ec2_sd_configs:
      - region: ap-northeast-2
        port: 9100
        filters:
          - name: tag:Tier
            values: ['web']
          - name: instance-state-name
            values: ['running']
    relabel_configs:
      - source_labels: [__meta_ec2_tag_Name]
        target_label: instance_name
      - source_labels: [__meta_ec2_availability_zone]
        target_label: availability_zone
      - source_labels: [__meta_ec2_instance_id]
        target_label: instance_id

  # WAS Tier Node Exporters
  - job_name: 'was-tier'
    ec2_sd_configs:
      - region: ap-northeast-2
        port: 9100
        filters:
          - name: tag:Tier
            values: ['was']
          - name: instance-state-name
            values: ['running']
    relabel_configs:
      - source_labels: [__meta_ec2_tag_Name]
        target_label: instance_name
      - source_labels: [__meta_ec2_availability_zone]
        target_label: availability_zone
      - source_labels: [__meta_ec2_instance_id]
        target_label: instance_id

  # Static configuration (백업용)
  - job_name: 'static-web'
    static_configs:
      - targets:
        - '10.0.50.10:9100'  # WEB-A
        - '10.0.60.10:9100'  # WEB-B
        labels:
          tier: 'web'

  - job_name: 'static-was'
    static_configs:
      - targets:
        - '10.0.100.10:9100'  # WAS-A
        - '10.0.110.10:9100'  # WAS-B
        labels:
          tier: 'was'
```

### Prometheus systemd 서비스

```ini
# /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --storage.tsdb.retention.time=30d \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

```bash
# 서비스 등록 및 시작
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus

# 상태 확인
sudo systemctl status prometheus

# Prometheus 웹 UI 접속
# http://<prometheus-server-ip>:9090
```

---

## 📈 Grafana Cloud 연동

### Grafana Cloud 선택 이유
- **관리 부담 감소**: 서버 관리 불필요
- **높은 가용성**: 99.9% SLA 제공
- **자동 백업**: 대시보드 및 데이터 자동 백업
- **무료 티어**: 소규모 프로젝트에 적합

### Prometheus → Grafana Cloud 연동

#### 1. Grafana Cloud 계정 생성
1. [Grafana Cloud](https://grafana.com/products/cloud/) 접속
2. 무료 계정 생성
3. 스택 생성

#### 2. Prometheus 원격 쓰기 설정

```yaml
# /etc/prometheus/prometheus.yml에 추가
remote_write:
  - url: https://prometheus-prod-01-eu-west-0.grafana.net/api/prom/push
    basic_auth:
      username: <your-grafana-instance-id>
      password: <your-grafana-api-key>
```

```bash
# Prometheus 재시작
sudo systemctl restart prometheus

# 로그 확인
sudo journalctl -u prometheus -f
```

### Grafana 대시보드 구성

#### 주요 대시보드

1. **시스템 오버뷰**
   - 전체 인스턴스 상태
   - CPU/메모리 사용률
   - 네트워크 트래픽
   - 디스크 I/O

2. **WEB Tier 모니터링**
   - Apache httpd 요청 수
   - 응답 시간
   - 에러율
   - 연결 수

3. **WAS Tier 모니터링**
   - Tomcat 스레드 풀
   - JVM 메모리
   - GC 활동
   - 애플리케이션 메트릭

4. **Auto Scaling 모니터링**
   - 인스턴스 수 변화
   - Scale-out/in 이벤트
   - CPU 트리거 상태

---

## 🚨 알림 규칙 설정

### Prometheus Alert Rules

```yaml
# /etc/prometheus/rules/alerts.yml
groups:
  - name: instance_alerts
    interval: 30s
    rules:
      # CPU 사용률 높음
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
          team: devops
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is above 80% (current value: {{ $value }}%)"

      # 메모리 사용률 높음
      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
          team: devops
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is above 85% (current value: {{ $value }}%)"

      # 디스크 공간 부족
      - alert: LowDiskSpace
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 15
        for: 10m
        labels:
          severity: critical
          team: devops
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Disk space is below 15% (current value: {{ $value }}%)"

      # 인스턴스 다운
      - alert: InstanceDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
          team: devops
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: "{{ $labels.job }} instance {{ $labels.instance }} has been down for more than 2 minutes"

  - name: application_alerts
    interval: 30s
    rules:
      # 응답 시간 느림
      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "High response time detected"
          description: "95th percentile response time is above 2 seconds (current value: {{ $value }}s)"

      # 에러율 높음
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.01
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate detected"
          description: "Error rate is above 1% (current value: {{ $value }}%)"
```

### Grafana Alert 설정

Grafana Cloud에서 알림 채널을 설정합니다.

#### Slack 통합

1. Grafana Cloud → Alerting → Contact points
2. "New contact point" 클릭
3. 설정:
   - **Name**: Slack DevOps Team
   - **Type**: Slack
   - **Webhook URL**: `https://hooks.slack.com/services/YOUR/WEBHOOK/URL`
   - **Message**: 
   ```
   🚨 Alert: {{ .GroupLabels.alertname }}
   
   Severity: {{ .GroupLabels.severity }}
   Instance: {{ .Labels.instance }}
   
   {{ .Annotations.description }}
   ```

#### Email 알림

1. Contact points → New contact point
2. 설정:
   - **Name**: DevOps Email
   - **Type**: Email
   - **Addresses**: devops@example.com
   - **Subject**: `[{{ .GroupLabels.severity }}] {{ .GroupLabels.alertname }}`

---

## 📊 주요 모니터링 메트릭

### 시스템 메트릭

| 메트릭 | PromQL 쿼리 | 임계값 |
|--------|-------------|--------|
| CPU 사용률 | `100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` | > 80% |
| 메모리 사용률 | `(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100` | > 85% |
| 디스크 사용률 | `100 - ((node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100)` | > 85% |
| 네트워크 수신 | `rate(node_network_receive_bytes_total[5m])` | - |
| 네트워크 송신 | `rate(node_network_transmit_bytes_total[5m])` | - |

### 애플리케이션 메트릭

| 메트릭 | CloudWatch | 임계값 |
|--------|-----------|--------|
| ALB 요청 수 | `RequestCount` | - |
| ALB 타겟 응답 시간 | `TargetResponseTime` | p95 > 2s |
| ALB 5XX 에러 | `HTTPCode_Target_5XX_Count` | > 10/min |
| ALB 타겟 헬스 | `HealthyHostCount` | < 최소 인스턴스 |
| RDS CPU | `CPUUtilization` | > 75% |
| RDS 연결 수 | `DatabaseConnections` | > 80% 최대값 |

---

## 🎨 Grafana 대시보드 예제

### 시스템 오버뷰 패널

```json
{
  "title": "CPU Usage by Instance",
  "targets": [
    {
      "expr": "100 - (avg by(instance_name) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
      "legendFormat": "{{ instance_name }}"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "thresholds": {
        "steps": [
          { "value": 0, "color": "green" },
          { "value": 50, "color": "yellow" },
          { "value": 80, "color": "red" }
        ]
      },
      "unit": "percent"
    }
  }
}
```

### Auto Scaling 이벤트 패널

```json
{
  "title": "Instance Count by Tier",
  "targets": [
    {
      "expr": "count by(tier) (up{job=~\"web-tier|was-tier\"} == 1)",
      "legendFormat": "{{ tier }}"
    }
  ],
  "type": "timeseries"
}
```

---

## 🔄 CloudWatch 통합

### CloudWatch Logs 수집

```bash
# CloudWatch Logs 에이전트 설치
sudo yum install amazon-cloudwatch-agent -y

# 설정 파일 생성
sudo vi /opt/aws/amazon-cloudwatch-agent/etc/config.json
```

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/httpd/access_log",
            "log_group_name": "/aws/ec2/petclinic/web/access",
            "log_stream_name": "{instance_id}"
          },
          {
            "file_path": "/var/log/httpd/error_log",
            "log_group_name": "/aws/ec2/petclinic/web/error",
            "log_stream_name": "{instance_id}"
          },
          {
            "file_path": "/opt/tomcat/logs/catalina.out",
            "log_group_name": "/aws/ec2/petclinic/was/catalina",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
```

```bash
# 에이전트 시작
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config \
    -m ec2 \
    -s \
    -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json
```

---

## 📝 모니터링 체크리스트

### 일일 체크
- [ ] 전체 인스턴스 상태 확인
- [ ] CPU/메모리 사용률 정상 범위
- [ ] ALB 타겟 헬스 체크 통과
- [ ] 에러 로그 확인
- [ ] 알림 발생 여부 점검

### 주간 체크
- [ ] 트래픽 패턴 분석
- [ ] Auto Scaling 이벤트 검토
- [ ] 디스크 공간 트렌드
- [ ] 응답 시간 트렌드
- [ ] 비용 증감 확인

### 월간 체크
- [ ] 용량 계획 업데이트
- [ ] 알림 임계값 조정
- [ ] 대시보드 최적화
- [ ] 메트릭 보관 정책 검토
- [ ] 보안 이벤트 분석

---

## 🔄 다음 단계

모니터링 및 알림 시스템 구축이 완료되었으므로 다음 단계를 진행합니다:

1. ✅ Node Exporter 설치 완료
2. ✅ Prometheus 구성 완료
3. ✅ Grafana Cloud 연동 완료
4. ✅ 알림 규칙 설정 완료
5. → [SLO 정책](./06-slo-policy.md)
6. → 지속적 개선

---

## 📚 참고 자료

- [Prometheus 공식 문서](https://prometheus.io/docs/)
- [Grafana 공식 문서](https://grafana.com/docs/)
- [Node Exporter](https://github.com/prometheus/node_exporter)
- [AWS CloudWatch 문서](https://docs.aws.amazon.com/cloudwatch/)
