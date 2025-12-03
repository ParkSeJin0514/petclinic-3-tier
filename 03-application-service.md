# 🧩 애플리케이션 서비스

## 프로젝트 정보
- **시작일**: 2025년 10월 13일
- **종료일**: 2025년 10월 14일
- **상태**: Completed
- **담당자**: 박세진, 홍정호
- **기준 OS**: Amazon Linux 2023

## 목표
비전공자들도 쉽게 따라할 수 있도록 모든 설명과 코드를 상세히 작성합니다.

---

## 🔄 ALB (Application Load Balancer) 구성 이해

### 포트 매핑 구조

ALB 설정은 처음에는 복잡해 보이지만, 포트 흐름을 이해하면 명확합니다.

```
[Client] 
   ↓ (HTTP:80)
[External ALB]
   Listener: 80
   Target: 80
   ↓ (HTTP:80)
[WEB Server - Apache httpd]
   Listen: 80
   Proxy to Internal ALB
   ↓ (HTTP:80)
[Internal ALB]
   Listener: 80
   Target: 8080
   ↓ (HTTP:8080)
[WAS Server - Tomcat]
   Listen: 8080
```

### 헬스체크 경로
- **External ALB → WEB**: `/health.html`
- **Internal ALB → WAS**: `/petclinic/oups`

---

## 🌐 WEB Tier 설정 (Apache httpd)

### 1. Apache httpd 설치

```bash
# Apache 설치
sudo yum install httpd -y

# 서비스 시작 및 자동 실행 설정
sudo systemctl start httpd
sudo systemctl enable httpd
```

### 2. 헬스체크 파일 생성

```bash
# /var/www/html에 health.html 생성
echo "OK" | sudo tee /var/www/html/health.html
```

### 3. Reverse Proxy 설정

#### Reverse Proxy란?
클라이언트의 요청을 대신 받아 내부의 실제 서버(WAS)로 전달하고, 서버로부터 받은 응답을 다시 클라이언트에게 전달하는 중개 서버입니다.

#### 프록시 설정 파일 생성

```bash
# 프록시 설정 파일 생성
sudo vi /etc/httpd/conf.d/alb-proxy.conf
```

#### 프록시 설정 내용

```apache
<VirtualHost *:80>
    ServerName <도메인>
    ServerAlias <External ALB DNS>
    DocumentRoot /var/www/html

    # 헬스체크 경로만 로컬 처리
    <Directory /var/www/html>
        Require all granted
        Options -Indexes
        AllowOverride None
    </Directory>

    ProxyPreserveHost On
    ProxyRequests Off

    # 헬스체크 경로는 프록시 제외
    ProxyPass /health.html !

    # .jsp 파일 프록시
    ProxyPassMatch ^/(.+\.jsp)(.*)$ http://<Internal ALB>/$1$2

    # /petclinic 경로 프록시
    ProxyPass /petclinic http://<Internal ALB>/petclinic retry=0 timeout=60
    ProxyPassReverse /petclinic http://<Internal ALB>/petclinic

    # 로그 설정
    ErrorLog /var/log/httpd/was_error.log
    CustomLog /var/log/httpd/was_access.log combined
</VirtualHost>
```

#### 설정 적용

```bash
# 설정 테스트
sudo httpd -t

# Apache 재시작
sudo systemctl restart httpd
```

### 4. 정적 콘텐츠 (index.html) 생성

CloudFront + S3를 통해 서비스될 정적 페이지를 생성합니다.

#### 핵심 기능
- PetClinic 애플리케이션 소개 페이지
- CloudFront를 통한 정적 컨텐츠 제공
- WEB 서버에서 리다이렉션

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PetClinic에 오신 것을 환영합니다</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            padding: 20px;
        }
        .container {
            background: white;
            border-radius: 20px;
            box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
            max-width: 650px;
            width: 100%;
            overflow: hidden;
        }
        .header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            padding: 30px;
            text-align: center;
            color: white;
        }
        .btn-primary {
            padding: 12px 30px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            border-radius: 25px;
            text-decoration: none;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🐾 PetClinic에 오신 것을 환영합니다</h1>
            <p>반려동물을 위한 최고의 케어</p>
        </div>
        <div class="content">
            <h2>우리 애완용품 쇼핑몰에 오신 것을 환영합니다!</h2>
            <div class="buttons">
                <a href="/petclinic" class="btn btn-primary">PetClinic 시작하기</a>
            </div>
        </div>
    </div>
</body>
</html>
```

**배포 위치**: S3 버킷에 업로드 후 CloudFront를 통해 제공

---

## ☕ WAS Tier 설정 (Apache Tomcat)

### 1. Java 설치

```bash
# Amazon Corretto 17 설치 (Amazon의 OpenJDK)
sudo yum install java-17-amazon-corretto-devel -y

# Java 버전 확인
java -version
```

### 2. Tomcat 설치

```bash
# Tomcat 사용자 생성
sudo useradd -r -m -U -d /opt/tomcat -s /bin/false tomcat

# Tomcat 다운로드 (최신 버전 사용 권장)
cd /tmp
wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.x/bin/apache-tomcat-10.1.x.tar.gz

# 압축 해제
sudo tar -xzf apache-tomcat-10.1.x.tar.gz -C /opt/tomcat --strip-components=1

# 권한 설정
sudo chown -R tomcat:tomcat /opt/tomcat
sudo chmod +x /opt/tomcat/bin/*.sh
```

### 3. Tomcat systemd 서비스 설정

```bash
sudo vi /etc/systemd/system/tomcat.service
```

```ini
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking

Environment=JAVA_HOME=/usr/lib/jvm/java-17-amazon-corretto
Environment=CATALINA_PID=/opt/tomcat/temp/tomcat.pid
Environment=CATALINA_HOME=/opt/tomcat
Environment=CATALINA_BASE=/opt/tomcat
Environment='CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC'
Environment='JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom'

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

User=tomcat
Group=tomcat
UMask=0007
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
# 서비스 등록 및 시작
sudo systemctl daemon-reload
sudo systemctl enable tomcat
sudo systemctl start tomcat

# 상태 확인
sudo systemctl status tomcat
```

### 4. Spring PetClinic 애플리케이션 배포

```bash
# WAR 파일을 Tomcat webapps 디렉토리에 배포
sudo cp /path/to/petclinic.war /opt/tomcat/webapps/

# Tomcat 재시작
sudo systemctl restart tomcat

# 로그 확인
tail -f /opt/tomcat/logs/catalina.out
```

### 5. 데이터베이스 연결 설정

#### application.properties 설정

```properties
# Database Configuration
spring.datasource.url=jdbc:mysql://<RDS-Endpoint>:3306/petclinic?useSSL=true
spring.datasource.username=admin
spring.datasource.password=<your-password>
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# JPA Configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# Connection Pool
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
```

---

## 🗄️ RDS MySQL 설정

### 1. RDS 인스턴스 생성

**주요 설정**:
- **엔진**: MySQL 8.0
- **인스턴스 클래스**: db.t3.medium (운영환경에 맞게 조정)
- **스토리지**: 100GB gp3 (Auto Scaling 활성화)
- **Multi-AZ**: 예 (고가용성)
- **VPC**: petclinic-kr-vpc
- **서브넷 그룹**: Private DB Subnets
- **보안 그룹**: petclinic-kr-sg-db

### 2. 데이터베이스 및 사용자 생성

```sql
-- 데이터베이스 생성
CREATE DATABASE petclinic CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 사용자 생성 및 권한 부여
CREATE USER 'petclinic'@'%' IDENTIFIED BY '<strong-password>';
GRANT ALL PRIVILEGES ON petclinic.* TO 'petclinic'@'%';
FLUSH PRIVILEGES;

-- 확인
SHOW DATABASES;
SELECT user, host FROM mysql.user;
```

### 3. 초기 데이터 로드

Spring PetClinic 애플리케이션은 첫 실행 시 자동으로 테이블을 생성하고 샘플 데이터를 로드합니다.

```bash
# 애플리케이션 로그에서 확인
tail -f /opt/tomcat/logs/catalina.out | grep -i "schema"
```

---

## 🎨 CloudFront + S3 설정

### 1. S3 버킷 생성 및 설정

```bash
# S3 버킷 생성
aws s3 mb s3://petclinic-static-content

# 정적 파일 업로드
aws s3 cp index.html s3://petclinic-static-content/
aws s3 cp css/ s3://petclinic-static-content/css/ --recursive
aws s3 cp images/ s3://petclinic-static-content/images/ --recursive
```

### 2. CloudFront 배포 생성

**주요 설정**:
- **Origin**: S3 버킷
- **Viewer Protocol Policy**: Redirect HTTP to HTTPS
- **Caching**: CachingOptimized
- **Compress Objects**: Yes
- **Price Class**: Use All Edge Locations

### 3. WEB 서버에서 CloudFront 연동

```apache
# Apache 설정에 CloudFront 리다이렉션 추가
<VirtualHost *:80>
    # 정적 콘텐츠는 CloudFront로 리다이렉션
    RedirectMatch 301 ^/$ https://<cloudfront-domain>/index.html
    RedirectMatch 301 ^/css/(.*)$ https://<cloudfront-domain>/css/$1
    RedirectMatch 301 ^/images/(.*)$ https://<cloudfront-domain>/images/$1
</VirtualHost>
```

---

## 📦 Launch Template 및 AMI 생성

### 1. 인스턴스 준비 완료 후 AMI 생성

#### WEB 서버 AMI
```bash
# 모든 설정 완료 후 AMI 생성
aws ec2 create-image \
    --instance-id <web-instance-id> \
    --name "petclinic-kr-ami-web-v1.0" \
    --description "PetClinic WEB Server with Apache httpd configured" \
    --no-reboot
```

#### WAS 서버 AMI
```bash
aws ec2 create-image \
    --instance-id <was-instance-id> \
    --name "petclinic-kr-ami-was-v1.0" \
    --description "PetClinic WAS Server with Tomcat and application deployed" \
    --no-reboot
```

### 2. Launch Template 생성

#### WEB Launch Template
- **이름**: petclinic-kr-tmp-web
- **AMI**: petclinic-kr-ami-web-v1.0
- **인스턴스 타입**: t3.small
- **보안 그룹**: petclinic-kr-sg-web
- **User Data**: Node Exporter 설치 스크립트

#### WAS Launch Template
- **이름**: petclinic-kr-tmp-was
- **AMI**: petclinic-kr-ami-was-v1.0
- **인스턴스 타입**: t3.medium
- **보안 그룹**: petclinic-kr-sg-was
- **User Data**: Node Exporter 설치 스크립트

---

## 🔍 테스트 및 검증

### 1. 헬스체크 확인

```bash
# WEB 서버 헬스체크
curl http://<web-instance-ip>/health.html
# 결과: OK

# Internal ALB를 통한 WAS 헬스체크
curl http://<internal-alb-dns>/petclinic/oups
# 결과: PetClinic error page (정상)
```

### 2. 애플리케이션 접근 테스트

```bash
# External ALB를 통한 접근
curl -I http://<external-alb-dns>/petclinic

# 결과 확인
HTTP/1.1 200 OK
Content-Type: text/html;charset=UTF-8
```

### 3. 데이터베이스 연결 확인

```bash
# WAS 서버에서 RDS 연결 테스트
mysql -h <rds-endpoint> -u petclinic -p petclinic

# 테이블 확인
SHOW TABLES;
```

---

## 🚨 트러블슈팅

### 일반적인 문제와 해결 방법

#### 1. ALB 헬스체크 실패
```bash
# 원인: 방화벽 또는 서비스 미실행
# 해결:
sudo systemctl status httpd
sudo systemctl status tomcat

# 보안 그룹 확인
aws ec2 describe-security-groups --group-ids <sg-id>
```

#### 2. 502 Bad Gateway
```bash
# 원인: WAS 서버 응답 없음
# 해결:
# Tomcat 로그 확인
tail -f /opt/tomcat/logs/catalina.out

# Internal ALB 타겟 상태 확인
aws elbv2 describe-target-health --target-group-arn <tg-arn>
```

#### 3. 데이터베이스 연결 실패
```bash
# 원인: 보안 그룹 또는 자격 증명 오류
# 해결:
# RDS 보안 그룹 인바운드 규칙 확인
# WAS 보안 그룹이 허용되어 있는지 확인

# 연결 테스트
telnet <rds-endpoint> 3306
```

---

## 📝 운영 체크리스트

### 배포 전 확인사항
- [ ] Apache httpd 정상 작동
- [ ] Reverse Proxy 설정 올바름
- [ ] Tomcat 정상 실행
- [ ] 애플리케이션 WAR 파일 배포 완료
- [ ] RDS 연결 설정 확인
- [ ] 헬스체크 경로 응답 정상
- [ ] 보안 그룹 규칙 검증
- [ ] CloudFront 캐싱 동작 확인

### 배포 후 모니터링
- [ ] ALB 타겟 헬스 상태
- [ ] CPU/메모리 사용률
- [ ] 애플리케이션 로그
- [ ] 데이터베이스 연결 풀
- [ ] 응답 시간 메트릭
- [ ] 에러 로그 확인

---

## 🔄 다음 단계

애플리케이션 서비스 구성이 완료되었으므로 다음 단계를 진행합니다:

1. ✅ WEB/WAS 서버 설정 완료
2. ✅ ALB 및 타겟 그룹 설정
3. ✅ RDS 데이터베이스 구성
4. → [부하 테스트 및 보안 정책](./04-load-test-security.md)
5. → Auto Scaling 정책 적용
6. → 모니터링 스택 구축

---

## 📚 참고 자료

- [Apache httpd 문서](https://httpd.apache.org/docs/)
- [Apache Tomcat 문서](https://tomcat.apache.org/)
- [Spring PetClinic GitHub](https://github.com/spring-projects/spring-petclinic)
- [AWS ALB 사용 설명서](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/)
