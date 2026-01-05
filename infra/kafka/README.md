# Kafka (KRaft Mode) StatefulSet

Apache Kafka를 **KRaft 모드**로 Kubernetes StatefulSet에 배포합니다.

## KRaft 모드란?

**KRaft** (Kafka Raft)는 Zookeeper 없이 Kafka를 실행할 수 있게 해주는 새로운 합의 프로토콜입니다.

### Zookeeper vs KRaft

| 항목 | Zookeeper 방식 | KRaft 방식 |
|------|----------------|------------|
| 필요한 컴포넌트 | Zookeeper + Kafka | Kafka만 |
| 리소스 사용 | 높음 (2개 시스템) | 낮음 (1개 시스템) |
| 복잡도 | 높음 | 낮음 |
| 시작 속도 | 느림 | 빠름 |
| Apache Kafka 지원 | 3.x까지 지원 | 3.3+부터 프로덕션 레디 |
| 미래 | Deprecated | 기본 방식 (4.0+) |

## 구성 요소

- **Namespace**: `messaging`
- **StatefulSet**: `kafka` (replicas: 1)
- **Service**: `kafka` (ClusterIP, 9092)
- **Headless Service**: `kafka-headless` (StatefulSet용)
- **PVC**: `kafka-data-kafka-0` (10Gi)
- **StorageClass**: `local-path` (K3s 기본)

## 사전 요구사항

1. **Namespace 생성**
```bash
kubectl create namespace messaging
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
kubectl get pods -n messaging

# PVC 확인
kubectl get pvc -n messaging

# Service 확인
kubectl get svc -n messaging

# 로그 확인 (KRaft 모드 확인)
kubectl logs -n messaging kafka-0 | grep -i kraft
```

## Kafka 접속

### 1. 클러스터 내부에서 접속
다른 Pod에서 다음 주소로 접속:
```
Bootstrap Server: kafka.messaging.svc.cluster.local:9092
```

### 2. 로컬에서 접속 (포트 포워딩)
```bash
# 포트 포워딩 시작
kubectl port-forward -n messaging svc/kafka 9092:9092

# 다른 터미널에서 Kafka 명령 사용
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

### 3. Pod 내부에서 직접 명령 실행
```bash
# 토픽 목록 조회
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list

# 토픽 생성
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic my-topic \
  --partitions 3 \
  --replication-factor 1
```

## Kafka 기본 사용법

### 토픽 관리

#### 토픽 생성
```bash
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic order-events \
  --partitions 3 \
  --replication-factor 1
```

#### 토픽 목록 조회
```bash
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list
```

#### 토픽 상세 정보
```bash
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic order-events
```

#### 토픽 설정 변경
```bash
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name order-events \
  --alter \
  --add-config retention.ms=86400000
```

#### 토픽 삭제
```bash
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --delete \
  --topic order-events
```

### 메시지 프로듀싱 & 컨슈밍

#### 메시지 전송 (Producer)
```bash
# 콘솔 프로듀서 실행
kubectl exec -n messaging kafka-0 -it -- /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic order-events

# 메시지 입력 후 Enter (Ctrl+C로 종료)
> {"orderId": 1, "status": "created"}
> {"orderId": 2, "status": "processing"}
```

#### 메시지 수신 (Consumer)
```bash
# 콘솔 컨슈머 실행 (처음부터 읽기)
kubectl exec -n messaging kafka-0 -it -- /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order-events \
  --from-beginning

# 최신 메시지만 읽기
kubectl exec -n messaging kafka-0 -it -- /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order-events
```

#### 컨슈머 그룹으로 읽기
```bash
kubectl exec -n messaging kafka-0 -it -- /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order-events \
  --group order-service-group \
  --from-beginning
```

### 컨슈머 그룹 관리

#### 컨슈머 그룹 목록
```bash
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --list
```

#### 컨슈머 그룹 상세 정보
```bash
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group order-service-group \
  --describe
```

#### 컨슈머 그룹 오프셋 리셋
```bash
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group order-service-group \
  --topic order-events \
  --reset-offsets \
  --to-earliest \
  --execute
```

## Spring Boot 연결 설정

### application.yml
```yaml
spring:
  kafka:
    bootstrap-servers: kafka.messaging.svc.cluster.local:9092

    # Producer 설정
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3

    # Consumer 설정
    consumer:
      group-id: order-service-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false
      properties:
        spring.json.trusted.packages: "*"

    # Listener 설정
    listener:
      ack-mode: manual
```

### build.gradle
```gradle
dependencies {
    implementation 'org.springframework.kafka:spring-kafka'
}
```

### Kafka 사용 예제 코드

#### KafkaConfig.java
```java
@Configuration
public class KafkaConfig {

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,
            "kafka.messaging.svc.cluster.local:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
            StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
            JsonSerializer.class);
        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }

    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,
            "kafka.messaging.svc.cluster.local:9092");
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "order-service");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
            StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
            JsonDeserializer.class);
        config.put(JsonDeserializer.TRUSTED_PACKAGES, "*");
        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object>
            kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
}
```

#### Producer 예제
```java
@Service
@RequiredArgsConstructor
public class OrderEventProducer {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public void sendOrderCreatedEvent(OrderEvent event) {
        kafkaTemplate.send("order-events", event.getOrderId().toString(), event)
            .whenComplete((result, ex) -> {
                if (ex == null) {
                    log.info("Message sent successfully: {}", event);
                } else {
                    log.error("Failed to send message: {}", event, ex);
                }
            });
    }
}
```

#### Consumer 예제
```java
@Component
@Slf4j
public class OrderEventConsumer {

    @KafkaListener(
        topics = "order-events",
        groupId = "order-service-group"
    )
    public void consumeOrderEvent(
        @Payload OrderEvent event,
        @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
        @Header(KafkaHeaders.OFFSET) long offset
    ) {
        log.info("Received message from topic: {}, partition: {}, offset: {}",
            topic, partition, offset);
        log.info("Event: {}", event);

        // 비즈니스 로직 처리
        processOrderEvent(event);
    }

    private void processOrderEvent(OrderEvent event) {
        // 주문 이벤트 처리 로직
    }
}
```

#### 이벤트 DTO
```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class OrderEvent {
    private Long orderId;
    private String status;
    private LocalDateTime createdAt;
}
```

## 서비스별 추천 토픽 구조

```bash
# Order Service
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic order.created --partitions 3 --replication-factor 1

kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic order.updated --partitions 3 --replication-factor 1

kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic order.cancelled --partitions 3 --replication-factor 1

# Auth Service
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic user.registered --partitions 3 --replication-factor 1

kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic user.login --partitions 3 --replication-factor 1
```

## 모니터링

### 브로커 정보 확인
```bash
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-broker-api-versions.sh \
  --bootstrap-server localhost:9092
```

### 클러스터 메타데이터
```bash
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-metadata.sh \
  --snapshot /var/lib/kafka/data/__cluster_metadata-0/00000000000000000000.log \
  --print
```

### 로그 확인
```bash
# 전체 로그
kubectl logs -n messaging kafka-0

# KRaft 관련 로그만
kubectl logs -n messaging kafka-0 | grep -i kraft

# 최근 50줄
kubectl logs -n messaging kafka-0 --tail=50

# 실시간 로그
kubectl logs -n messaging kafka-0 -f
```

## KRaft 설정 상세

### 주요 환경변수 설명

| 환경변수 | 값 | 설명 |
|---------|-----|------|
| `KAFKA_PROCESS_ROLES` | `broker,controller` | 브로커와 컨트롤러 역할 모두 수행 |
| `KAFKA_NODE_ID` | `1` | 노드 고유 ID |
| `KAFKA_CONTROLLER_QUORUM_VOTERS` | `1@kafka-0...` | 컨트롤러 쿼럼 투표자 목록 |
| `KAFKA_LISTENERS` | `PLAINTEXT://:9092,CONTROLLER://:9093` | 리스너 설정 |
| `KAFKA_ADVERTISED_LISTENERS` | `PLAINTEXT://kafka...` | 클라이언트에 광고할 주소 |
| `CLUSTER_ID` | `MkU3OEVBNTcwNTJENDM2Qk` | 클러스터 고유 ID |

### 클러스터 ID 생성 방법
```bash
# 새로운 클러스터 ID 생성 (필요시)
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-storage.sh random-uuid
```

## 스토리지 정보

- **StorageClass**: `local-path` (K3s의 기본 로컬 스토리지)
- **Capacity**: 10Gi
- **AccessMode**: ReadWriteOnce (RWO)
- **데이터 경로**: `/var/lib/kafka/data` (Pod 내부)
- **실제 저장 위치**: K3s 노드의 로컬 디스크

### 저장되는 데이터
```bash
# 데이터 디렉토리 확인
kubectl exec -n messaging kafka-0 -- ls -lh /var/lib/kafka/data/

# 토픽 데이터 확인
kubectl exec -n messaging kafka-0 -- du -sh /var/lib/kafka/data/*
```

## 성능 튜닝

### 파티션 수 결정
- **단일 토픽**: 3-5개 파티션 권장
- **높은 처리량**: 파티션 수 증가
- **순서 보장 필요**: 파티션 1개 사용

### 복제 팩터
- 현재: 1 (단일 브로커)
- 프로덕션: 3 권장 (3개 이상의 브로커 필요)

### 보존 기간 설정
```bash
# 7일 보존
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name order-events \
  --alter \
  --add-config retention.ms=604800000
```

## 백업 및 복구

### 데이터 백업
```bash
# 전체 데이터 디렉토리 백업
kubectl exec -n messaging kafka-0 -- tar czf /tmp/kafka-backup.tar.gz /var/lib/kafka/data/
kubectl cp messaging/kafka-0:/tmp/kafka-backup.tar.gz ./kafka-backup-$(date +%Y%m%d).tar.gz
```

### 토픽 메타데이터 백업
```bash
# 토픽 목록 저장
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list > topics-backup.txt

# 각 토픽 설정 저장
for topic in $(cat topics-backup.txt); do
  kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --describe --topic $topic > topic-$topic-config.txt
done
```

## 삭제

```bash
# StatefulSet 삭제
kubectl delete -f statefulset.yaml

# Service 삭제
kubectl delete -f service.yaml

# PVC 삭제 (데이터 완전 삭제)
kubectl delete pvc -n messaging kafka-data-kafka-0

# Namespace 삭제 (선택사항)
kubectl delete namespace messaging
```

## 문제 해결

### Pod가 CrashLoopBackOff
```bash
# 로그 확인
kubectl logs -n messaging kafka-0 --previous

# 상세 정보
kubectl describe pod -n messaging kafka-0

# 클러스터 ID 문제인 경우 재생성
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-storage.sh random-uuid
```

### 메시지 전송 실패
```bash
# 브로커 상태 확인
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-broker-api-versions.sh \
  --bootstrap-server localhost:9092

# 토픽 존재 확인
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list
```

### 컨슈머 지연 (Lag)
```bash
# 컨슈머 그룹 지연 확인
kubectl exec -n messaging kafka-0 -- /opt/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group order-service-group \
  --describe

# LAG 컬럼에서 지연 확인
```

## 참고사항

- **KRaft 모드**: Zookeeper 불필요, 단순한 아키텍처
- **단일 브로커**: 고가용성 구성 아님, 개발/테스트 환경용
- **복제 팩터 1**: 데이터 손실 위험 있음
- **로컬 스토리지**: 노드 장애 시 데이터 손실 가능
- **포트**: 9092 (클라이언트), 9093 (컨트롤러)
- **ClusterIP**: 클러스터 외부에서 직접 접근 불가

## 추가 학습 자료

- [Apache Kafka 공식 문서](https://kafka.apache.org/documentation/)
- [KRaft 소개](https://kafka.apache.org/documentation/#kraft)
- [Spring Kafka 문서](https://spring.io/projects/spring-kafka)
