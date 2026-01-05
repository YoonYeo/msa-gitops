# Redis StatefulSet

Redis 캐시 서버를 Kubernetes StatefulSet으로 배포합니다.

## 구성 요소

- **Namespace**: `cache`
- **StatefulSet**: `redis` (replicas: 1)
- **Service**: `redis` (ClusterIP)
- **PVC**: `redis-data-redis-0` (5Gi)
- **StorageClass**: `local-path` (K3s 기본)

## 사전 요구사항

1. **Namespace 생성**
```bash
kubectl create namespace cache
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
kubectl get pods -n cache

# PVC 확인
kubectl get pvc -n cache

# Service 확인
kubectl get svc -n cache

# 로그 확인
kubectl logs -n cache redis-0

# Redis 연결 테스트
kubectl exec -n cache redis-0 -- redis-cli ping
# 응답: PONG
```

## Redis 접속

### 1. 클러스터 내부에서 접속
다른 Pod에서 다음 주소로 접속:
```
Host: redis.cache.svc.cluster.local
Port: 6379
```

### 2. 로컬에서 접속 (포트 포워딩)
```bash
# 포트 포워딩 시작
kubectl port-forward -n cache svc/redis 6379:6379

# 다른 터미널에서 접속
redis-cli -h localhost -p 6379
```

### 3. Pod 내부에서 직접 접속
```bash
kubectl exec -n cache redis-0 -it -- redis-cli
```

## Redis 기본 사용법

### 데이터 저장 및 조회
```bash
# Pod 내부 접속
kubectl exec -n cache redis-0 -it -- redis-cli

# 문자열 저장
> SET mykey "Hello Redis"
OK

# 문자열 조회
> GET mykey
"Hello Redis"

# TTL과 함께 저장 (10초)
> SETEX tempkey 10 "temporary value"
OK

# 해시 저장
> HSET user:1000 name "John Doe"
> HSET user:1000 email "john@example.com"
> HGETALL user:1000

# 리스트 사용
> LPUSH mylist "item1"
> LPUSH mylist "item2"
> LRANGE mylist 0 -1

# 키 확인
> KEYS *

# 키 삭제
> DEL mykey

# 종료
> exit
```

### kubectl exec로 직접 명령 실행
```bash
# PING 테스트
kubectl exec -n cache redis-0 -- redis-cli ping

# 데이터 저장
kubectl exec -n cache redis-0 -- redis-cli SET test-key "test-value"

# 데이터 조회
kubectl exec -n cache redis-0 -- redis-cli GET test-key

# 모든 키 조회
kubectl exec -n cache redis-0 -- redis-cli KEYS "*"

# Redis 정보 확인
kubectl exec -n cache redis-0 -- redis-cli INFO server
```

## Spring Boot 연결 설정

### application.yml
```yaml
spring:
  data:
    redis:
      host: redis.cache.svc.cluster.local
      port: 6379
      # password 없음 (기본 설정)
      timeout: 60000
      lettuce:
        pool:
          max-active: 8
          max-idle: 8
          min-idle: 0
```

### build.gradle
```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
}
```

### Redis 사용 예제 코드

#### RedisConfig.java
```java
@Configuration
@EnableCaching
public class RedisConfig {

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory(
            "redis.cache.svc.cluster.local", 6379
        );
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }

    @Bean
    public CacheManager cacheManager() {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new StringRedisSerializer())
            )
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer())
            );

        return RedisCacheManager.builder(redisConnectionFactory())
            .cacheDefaults(config)
            .build();
    }
}
```

#### Service에서 캐시 사용
```java
@Service
public class UserService {

    @Cacheable(value = "users", key = "#userId")
    public User getUserById(Long userId) {
        // DB에서 조회 (캐시 미스 시에만 실행)
        return userRepository.findById(userId).orElse(null);
    }

    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        // DB 업데이트 후 캐시도 업데이트
        return userRepository.save(user);
    }

    @CacheEvict(value = "users", key = "#userId")
    public void deleteUser(Long userId) {
        // DB에서 삭제 후 캐시도 삭제
        userRepository.deleteById(userId);
    }
}
```

#### RedisTemplate 직접 사용
```java
@Service
@RequiredArgsConstructor
public class CacheService {

    private final RedisTemplate<String, Object> redisTemplate;

    public void saveValue(String key, String value) {
        redisTemplate.opsForValue().set(key, value);
    }

    public void saveValueWithExpiry(String key, String value, long timeout) {
        redisTemplate.opsForValue().set(key, value, timeout, TimeUnit.SECONDS);
    }

    public String getValue(String key) {
        return (String) redisTemplate.opsForValue().get(key);
    }

    public void deleteValue(String key) {
        redisTemplate.delete(key);
    }

    public void saveHash(String key, Map<String, Object> data) {
        redisTemplate.opsForHash().putAll(key, data);
    }

    public Map<Object, Object> getHash(String key) {
        return redisTemplate.opsForHash().entries(key);
    }
}
```

## Redis 영속성 설정

현재 **AOF (Append Only File)** 모드가 활성화되어 있습니다.

### AOF란?
- 모든 쓰기 명령을 로그 파일에 기록
- Redis 재시작 시 AOF 파일을 재생하여 데이터 복구
- 데이터 손실 최소화 (일반적으로 1초 이내의 데이터만 손실 가능)

### AOF 파일 위치
- Pod 내부: `/data/appendonly.aof`
- PVC를 통해 영구 저장

### 데이터 확인
```bash
# AOF 파일 확인
kubectl exec -n cache redis-0 -- ls -lh /data/

# Redis 영속성 정보 확인
kubectl exec -n cache redis-0 -- redis-cli INFO persistence
```

## 모니터링

### Redis 정보 확인
```bash
# 서버 정보
kubectl exec -n cache redis-0 -- redis-cli INFO server

# 메모리 사용량
kubectl exec -n cache redis-0 -- redis-cli INFO memory

# 통계 정보
kubectl exec -n cache redis-0 -- redis-cli INFO stats

# 클라이언트 연결 정보
kubectl exec -n cache redis-0 -- redis-cli INFO clients

# 복제 정보
kubectl exec -n cache redis-0 -- redis-cli INFO replication
```

### 실시간 모니터링
```bash
# 실시간 명령 모니터링
kubectl exec -n cache redis-0 -it -- redis-cli MONITOR

# 느린 쿼리 로그 확인
kubectl exec -n cache redis-0 -- redis-cli SLOWLOG GET 10
```

## 백업 및 복구

### RDB 스냅샷 생성
```bash
# 수동 스냅샷 생성
kubectl exec -n cache redis-0 -- redis-cli BGSAVE

# 스냅샷 파일 확인
kubectl exec -n cache redis-0 -- ls -lh /data/dump.rdb
```

### 데이터 백업
```bash
# AOF 파일 백업
kubectl cp cache/redis-0:/data/appendonly.aof ./redis-backup-$(date +%Y%m%d).aof

# RDB 파일 백업 (있는 경우)
kubectl cp cache/redis-0:/data/dump.rdb ./redis-backup-$(date +%Y%m%d).rdb
```

### 데이터 복구
```bash
# 백업 파일을 Pod에 복사
kubectl cp ./redis-backup.aof cache/redis-0:/data/appendonly.aof

# Redis 재시작
kubectl delete pod -n cache redis-0
# StatefulSet이 자동으로 새 Pod 생성
```

## 스토리지 정보

- **StorageClass**: `local-path` (K3s의 기본 로컬 스토리지)
- **Capacity**: 5Gi
- **AccessMode**: ReadWriteOnce (RWO)
- **데이터 경로**: `/data` (Pod 내부)
- **실제 저장 위치**: K3s 노드의 로컬 디스크 (`/var/lib/rancher/k3s/storage/`)

## 삭제

```bash
# StatefulSet 삭제
kubectl delete -f statefulset.yaml

# Service 삭제
kubectl delete -f service.yaml

# PVC 삭제 (데이터 완전 삭제)
kubectl delete pvc -n cache redis-data-redis-0

# Namespace 삭제 (선택사항)
kubectl delete namespace cache
```

## 문제 해결

### Pod가 시작되지 않음
```bash
# Pod 상세 정보 확인
kubectl describe pod -n cache redis-0

# 로그 확인
kubectl logs -n cache redis-0

# Events 확인
kubectl get events -n cache --sort-by='.lastTimestamp'
```

### 연결 실패
```bash
# Service 확인
kubectl get svc -n cache redis

# PING 테스트
kubectl exec -n cache redis-0 -- redis-cli ping

# 네트워크 확인 (다른 Pod에서)
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
# nc -zv redis.cache.svc.cluster.local 6379
```

### 메모리 부족
```bash
# 메모리 사용량 확인
kubectl exec -n cache redis-0 -- redis-cli INFO memory

# 모든 데이터 삭제 (주의!)
kubectl exec -n cache redis-0 -- redis-cli FLUSHALL

# 특정 패턴의 키 삭제
kubectl exec -n cache redis-0 -- redis-cli --scan --pattern "temp:*" | xargs redis-cli DEL
```

## 성능 튜닝

### Maxmemory 설정 (필요시)
현재는 기본 설정 사용 중. 메모리 제한이 필요한 경우 StatefulSet에 다음 환경변수 추가:

```yaml
env:
  - name: REDIS_MAXMEMORY
    value: "256mb"
  - name: REDIS_MAXMEMORY_POLICY
    value: "allkeys-lru"  # LRU 정책으로 오래된 키부터 삭제
```

## 참고사항

- **비밀번호 없음**: 현재 인증 설정 없음, 클러스터 내부 전용
- **단일 인스턴스**: 고가용성(HA) 구성 아님
- **AOF 영속성**: 모든 쓰기 작업이 로그 파일에 기록됨
- **로컬 스토리지**: 노드 장애 시 데이터 손실 가능
- **ClusterIP**: 클러스터 외부에서 직접 접근 불가, 포트 포워딩 필요
