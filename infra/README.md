# MSA Infrastructure

마이크로서비스 아키텍처를 위한 인프라 컴포넌트들을 Kubernetes에 배포합니다.

## 전체 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│  ┌────────────┐  ┌─────────────┐  ┌──────────────────────┐ │
│  │ Namespace  │  │ Namespace   │  │ Namespace            │ │
│  │    db      │  │   cache     │  │   messaging          │ │
│  │            │  │             │  │                      │ │
│  │ ┌────────┐ │  │ ┌─────────┐ │  │ ┌──────────────────┐ │ │
│  │ │MariaDB │ │  │ │ Redis   │ │  │ │ Kafka (KRaft)    │ │ │
│  │ │        │ │  │ │         │ │  │ │                  │ │ │
│  │ │ 10Gi   │ │  │ │  5Gi    │ │  │ │      10Gi        │ │ │
│  │ └────────┘ │  │ └─────────┘ │  │ └──────────────────┘ │ │
│  └────────────┘  └─────────────┘  └──────────────────────┘ │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           Services (각 네임스페이스)                   │   │
│  │  - mariadb.db.svc.cluster.local:3306                 │   │
│  │  - redis.cache.svc.cluster.local:6379                │   │
│  │  - kafka.messaging.svc.cluster.local:9092            │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 배포된 컴포넌트

### 1. MariaDB
- **경로**: `mariadb/`
- **네임스페이스**: `db`
- **용도**: 관계형 데이터베이스
- **스토리지**: 10Gi
- **포트**: 3306
- **버전**: 10.11
- [상세 문서](./mariadb/README.md)

### 2. Redis
- **경로**: `redis/`
- **네임스페이스**: `cache`
- **용도**: 캐시 및 세션 저장소
- **스토리지**: 5Gi
- **포트**: 6379
- **버전**: 7.2-alpine
- **영속성**: AOF (Append Only File)
- [상세 문서](./redis/README.md)

### 3. Kafka (KRaft Mode)
- **경로**: `kafka/`
- **네임스페이스**: `messaging`
- **용도**: 이벤트 스트리밍 플랫폼
- **스토리지**: 10Gi
- **포트**: 9092 (클라이언트), 9093 (컨트롤러)
- **버전**: 3.8.1
- **특징**: Zookeeper 없는 KRaft 모드
- [상세 문서](./kafka/README.md)

## 빠른 시작

### 전체 인프라 배포
```bash
# 1. Namespace 생성
kubectl create namespace db
kubectl create namespace cache
kubectl create namespace messaging

# 2. MariaDB Secret 생성
kubectl create secret generic mariadb-secret \
  -n db \
  --from-literal=root-password=rootpassword123

# 3. MariaDB 배포
kubectl apply -f mariadb/service.yaml
kubectl apply -f mariadb/statefulset.yaml

# 4. Redis 배포
kubectl apply -f redis/service.yaml
kubectl apply -f redis/statefulset.yaml

# 5. Kafka 배포
kubectl apply -f kafka/service.yaml
kubectl apply -f kafka/statefulset.yaml

# 6. 배포 상태 확인
kubectl get pods -n db
kubectl get pods -n cache
kubectl get pods -n messaging
```

### 전체 인프라 삭제
```bash
# StatefulSet 및 Service 삭제
kubectl delete -f mariadb/
kubectl delete -f redis/
kubectl delete -f kafka/

# PVC 삭제 (데이터 완전 삭제)
kubectl delete pvc -n db --all
kubectl delete pvc -n cache --all
kubectl delete pvc -n messaging --all

# Secret 삭제
kubectl delete secret -n db mariadb-secret

# Namespace 삭제
kubectl delete namespace db
kubectl delete namespace cache
kubectl delete namespace messaging
```

## 서비스 연결 주소

각 마이크로서비스에서 다음 주소로 인프라에 접속합니다:

### MariaDB
```yaml
spring:
  datasource:
    url: jdbc:mariadb://mariadb.db.svc.cluster.local:3306/your_database
    username: your_user
    password: your_password
```

### Redis
```yaml
spring:
  data:
    redis:
      host: redis.cache.svc.cluster.local
      port: 6379
```

### Kafka
```yaml
spring:
  kafka:
    bootstrap-servers: kafka.messaging.svc.cluster.local:9092
```

## 로컬 개발 환경 접속

포트 포워딩을 사용하여 로컬에서 각 인프라에 접속할 수 있습니다:

```bash
# MariaDB
kubectl port-forward -n db svc/mariadb 3306:3306
# 접속: mysql -h 127.0.0.1 -P 3306 -u root -p

# Redis
kubectl port-forward -n cache svc/redis 6379:6379
# 접속: redis-cli -h localhost -p 6379

# Kafka
kubectl port-forward -n messaging svc/kafka 9092:9092
# 접속: kafka-topics.sh --bootstrap-server localhost:9092 --list
```

## 리소스 사용량

| 컴포넌트 | CPU 요청 | 메모리 요청 | 스토리지 | 총 리소스 |
|---------|---------|------------|----------|-----------|
| MariaDB | -       | -          | 10Gi     | ~1GB RAM  |
| Redis   | -       | -          | 5Gi      | ~256MB RAM|
| Kafka   | -       | -          | 10Gi     | ~1GB RAM  |
| **합계** | -      | -          | **25Gi** | ~2.3GB RAM|

**참고**: 현재 리소스 제한(limits/requests)을 설정하지 않아 필요에 따라 자동 조절됩니다.

## 스토리지 정보

모든 컴포넌트는 **K3s의 기본 StorageClass인 `local-path`**를 사용합니다.

### local-path 특징
- **타입**: 로컬 스토리지 (노드의 로컬 디스크)
- **경로**: `/var/lib/rancher/k3s/storage/`
- **장점**: 빠른 I/O, 추가 설정 불필요
- **단점**:
  - 노드 장애 시 데이터 손실 위험
  - 노드 간 이동 불가 (Pod가 다른 노드로 이동하면 데이터 접근 불가)

### 프로덕션 권장사항
프로덕션 환경에서는 다음과 같은 분산 스토리지 사용 권장:
- **NFS**: 네트워크 파일 시스템
- **Ceph/Rook**: 분산 블록 스토리지
- **Cloud Provider Storage**: AWS EBS, GCP Persistent Disk, Azure Disk

## 모니터링

### 전체 인프라 상태 확인
```bash
# 모든 Pod 상태
kubectl get pods -A | grep -E 'db|cache|messaging'

# 모든 PVC 상태
kubectl get pvc -A

# 모든 Service 확인
kubectl get svc -A | grep -E 'mariadb|redis|kafka'

# 노드별 Pod 분포
kubectl get pods -A -o wide | grep -E 'mariadb|redis|kafka'
```

### 로그 모니터링
```bash
# MariaDB 로그
kubectl logs -n db mariadb-0 -f

# Redis 로그
kubectl logs -n cache redis-0 -f

# Kafka 로그
kubectl logs -n messaging kafka-0 -f
```

### 리소스 사용량 확인
```bash
# Pod 리소스 사용량
kubectl top pods -n db
kubectl top pods -n cache
kubectl top pods -n messaging

# 노드 리소스 사용량
kubectl top nodes
```

## 네트워크 정책

현재 네트워크 정책이 설정되어 있지 않아 모든 Pod 간 통신이 가능합니다.

### 프로덕션 권장 네트워크 정책
```yaml
# 예시: MariaDB는 특정 서비스만 접근 가능
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: mariadb-network-policy
  namespace: db
spec:
  podSelector:
    matchLabels:
      app: mariadb
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: order-service
    - namespaceSelector:
        matchLabels:
          name: auth-service
    ports:
    - protocol: TCP
      port: 3306
```

## 보안 고려사항

### 현재 설정
- ✅ MariaDB root 비밀번호는 Secret으로 관리
- ❌ Redis 비밀번호 미설정 (클러스터 내부 전용)
- ❌ Kafka 인증 미설정 (PLAINTEXT 프로토콜)
- ❌ 네트워크 정책 미설정

### 프로덕션 권장사항
1. **Redis 비밀번호 설정**
```yaml
env:
  - name: REDIS_PASSWORD
    valueFrom:
      secretKeyRef:
        name: redis-secret
        key: password
```

2. **Kafka SASL 인증 활성화**
```yaml
env:
  - name: KAFKA_LISTENER_SECURITY_PROTOCOL_MAP
    value: "SASL_PLAINTEXT:SASL_PLAINTEXT,CONTROLLER:SASL_PLAINTEXT"
```

3. **TLS 암호화 적용**
4. **네트워크 정책 적용**
5. **Pod Security Standards 적용**

## 백업 전략

### 자동 백업 스크립트 (예시)
```bash
#!/bin/bash
BACKUP_DIR="/backups/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# MariaDB 백업
kubectl exec -n db mariadb-0 -- mysqldump -u root -prootpassword123 --all-databases > $BACKUP_DIR/mariadb.sql

# Redis 백업
kubectl exec -n cache redis-0 -- redis-cli BGSAVE
kubectl cp cache/redis-0:/data/dump.rdb $BACKUP_DIR/redis.rdb

# Kafka 데이터 백업
kubectl exec -n messaging kafka-0 -- tar czf /tmp/kafka-backup.tar.gz /var/lib/kafka/data/
kubectl cp messaging/kafka-0:/tmp/kafka-backup.tar.gz $BACKUP_DIR/kafka.tar.gz

echo "Backup completed: $BACKUP_DIR"
```

### CronJob으로 자동 백업 (Kubernetes)
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: infra-backup
spec:
  schedule: "0 2 * * *"  # 매일 새벽 2시
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: your-backup-image
            command: ["/backup.sh"]
          restartPolicy: OnFailure
```

## 문제 해결

### 일반적인 문제

#### Pod가 Pending 상태
```bash
# 원인 확인
kubectl describe pod -n <namespace> <pod-name>

# 주요 원인:
# 1. PVC가 Bound되지 않음
kubectl get pvc -n <namespace>

# 2. 노드 리소스 부족
kubectl describe node

# 3. StorageClass 없음
kubectl get storageclass
```

#### Pod가 CrashLoopBackOff
```bash
# 로그 확인
kubectl logs -n <namespace> <pod-name> --previous

# 이벤트 확인
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

#### 연결 실패
```bash
# Service DNS 확인
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup mariadb.db.svc.cluster.local

# 포트 확인
kubectl run -it --rm debug --image=busybox --restart=Never -- nc -zv mariadb.db.svc.cluster.local 3306
```

## 업그레이드

### 컴포넌트 버전 업그레이드
```bash
# 1. 이미지 태그 변경 (statefulset.yaml)
# image: mariadb:10.11 → mariadb:10.12

# 2. StatefulSet 업데이트
kubectl apply -f mariadb/statefulset.yaml

# 3. 롤링 업데이트 확인
kubectl rollout status statefulset -n db mariadb

# 4. 문제 발생 시 롤백
kubectl rollout undo statefulset -n db mariadb
```

## 추가 리소스

- [MariaDB 상세 문서](./mariadb/README.md)
- [Redis 상세 문서](./redis/README.md)
- [Kafka 상세 문서](./kafka/README.md)
- [Kubernetes 공식 문서](https://kubernetes.io/docs/)
- [K3s 공식 문서](https://docs.k3s.io/)

## 라이선스

이 인프라 구성은 다음 오픈소스 소프트웨어를 사용합니다:
- **MariaDB**: GPL v2
- **Redis**: BSD 3-Clause
- **Apache Kafka**: Apache License 2.0
