# MariaDB StatefulSet

MariaDB 데이터베이스를 Kubernetes StatefulSet으로 배포합니다.

## 구성 요소

- **Namespace**: `db`
- **StatefulSet**: `mariadb` (replicas: 1)
- **Service**: `mariadb` (ClusterIP)
- **PVC**: `mariadb-data-mariadb-0` (10Gi)
- **StorageClass**: `local-path` (K3s 기본)

## 사전 요구사항

1. **Namespace 생성**
```bash
kubectl create namespace db
```

2. **Secret 생성**
```bash
kubectl create secret generic mariadb-secret \
  -n db \
  --from-literal=root-password=rootpassword123
```

## 배포

```bash
# Service 배포
kubectl apply -f service.yaml

# StatefulSet 배포
kubectl apply -f statefulset.yaml
```

## 배포 확인

```bash
# Pod 상태 확인
kubectl get pods -n db

# PVC 확인
kubectl get pvc -n db

# Service 확인
kubectl get svc -n db

# 로그 확인
kubectl logs -n db mariadb-0
```

## 데이터베이스 접속

### 1. 클러스터 내부에서 접속
다른 Pod에서 다음 주소로 접속:
```
Host: mariadb.db.svc.cluster.local
Port: 3306
User: root
Password: rootpassword123
```

### 2. 로컬에서 접속 (포트 포워딩)
```bash
# 포트 포워딩 시작
kubectl port-forward -n db svc/mariadb 3306:3306

# 다른 터미널에서 접속
mysql -h 127.0.0.1 -P 3306 -u root -p
# Password: rootpassword123
```

### 3. Pod 내부에서 직접 접속
```bash
kubectl exec -n db mariadb-0 -it -- mysql -u root -p
```

## 데이터베이스 및 유저 생성

### Auth Service용 데이터베이스
```sql
CREATE DATABASE IF NOT EXISTS auth_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS 'auth_user'@'%' IDENTIFIED BY 'auth_password123';
GRANT ALL PRIVILEGES ON auth_db.* TO 'auth_user'@'%';
FLUSH PRIVILEGES;
```

### Order Service용 데이터베이스
```sql
CREATE DATABASE IF NOT EXISTS order_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS 'order_user'@'%' IDENTIFIED BY 'order_password123';
GRANT ALL PRIVILEGES ON order_db.* TO 'order_user'@'%';
FLUSH PRIVILEGES;
```

### kubectl로 실행
```bash
kubectl exec -n db mariadb-0 -- mysql -u root -prootpassword123 -e "
CREATE DATABASE IF NOT EXISTS auth_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE IF NOT EXISTS order_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS 'auth_user'@'%' IDENTIFIED BY 'auth_password123';
GRANT ALL PRIVILEGES ON auth_db.* TO 'auth_user'@'%';
CREATE USER IF NOT EXISTS 'order_user'@'%' IDENTIFIED BY 'order_password123';
GRANT ALL PRIVILEGES ON order_db.* TO 'order_user'@'%';
FLUSH PRIVILEGES;
"
```

## Spring Boot 연결 설정

### application.yml
```yaml
spring:
  datasource:
    url: jdbc:mariadb://mariadb.db.svc.cluster.local:3306/auth_db
    username: auth_user
    password: auth_password123
    driver-class-name: org.mariadb.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

### build.gradle
```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.mariadb.jdbc:mariadb-java-client'
}
```

## 스토리지 정보

- **StorageClass**: `local-path` (K3s의 기본 로컬 스토리지)
- **Capacity**: 10Gi
- **AccessMode**: ReadWriteOnce (RWO)
- **데이터 경로**: `/var/lib/mysql` (Pod 내부)
- **실제 저장 위치**: K3s 노드의 로컬 디스크 (`/var/lib/rancher/k3s/storage/`)

## 백업 및 복구

### 데이터베이스 백업
```bash
# 특정 데이터베이스 백업
kubectl exec -n db mariadb-0 -- mysqldump -u root -prootpassword123 auth_db > auth_db_backup.sql

# 모든 데이터베이스 백업
kubectl exec -n db mariadb-0 -- mysqldump -u root -prootpassword123 --all-databases > all_databases_backup.sql
```

### 데이터베이스 복구
```bash
# 백업 파일을 Pod에 복사
kubectl cp auth_db_backup.sql db/mariadb-0:/tmp/

# 복구 실행
kubectl exec -n db mariadb-0 -- mysql -u root -prootpassword123 auth_db < /tmp/auth_db_backup.sql
```

## 삭제

```bash
# StatefulSet 삭제
kubectl delete -f statefulset.yaml

# Service 삭제
kubectl delete -f service.yaml

# PVC 삭제 (데이터 완전 삭제)
kubectl delete pvc -n db mariadb-data-mariadb-0

# Secret 삭제
kubectl delete secret -n db mariadb-secret

# Namespace 삭제 (선택사항)
kubectl delete namespace db
```

## 문제 해결

### Pod가 Pending 상태
```bash
# Pod 상세 정보 확인
kubectl describe pod -n db mariadb-0

# Events 확인
kubectl get events -n db --sort-by='.lastTimestamp'
```

### 연결 실패
```bash
# Service 확인
kubectl get svc -n db mariadb

# Pod 로그 확인
kubectl logs -n db mariadb-0 --tail=50

# Pod 내부에서 연결 테스트
kubectl exec -n db mariadb-0 -- mysql -u root -prootpassword123 -e "SELECT 1;"
```

### PVC가 Bound 되지 않음
```bash
# PVC 상태 확인
kubectl get pvc -n db

# PV 확인
kubectl get pv

# StorageClass 확인
kubectl get storageclass
```

## 참고사항

- **ClusterIP 서비스**: 클러스터 외부에서 직접 접근 불가, 포트 포워딩 필요
- **단일 인스턴스**: 고가용성(HA) 구성 아님, 개발/테스트 환경용
- **로컬 스토리지**: 노드 장애 시 데이터 손실 가능, 중요 데이터는 정기 백업 필요
- **비밀번호 관리**: 프로덕션 환경에서는 더 강력한 비밀번호 사용 권장
