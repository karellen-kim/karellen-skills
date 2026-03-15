# 메모리 이슈 레퍼런스

## JVM 힙 메모리 구조

```
JVM 힙 (권장: 물리 메모리의 50%, 최대 ~30GB)
  ├── Lucene 세그먼트 메타데이터 (FST, BKD Tree)
  ├── Fielddata (text 필드 집계 — 피해야 함)
  ├── Shard Request Cache (힙의 1%)
  ├── Indexing Buffer (힙의 10%)
  ├── Node Query Cache (힙의 10%)
  └── 여유 공간 (GC 효율 위해 30%+ 확보)

OS 파일 캐시 (Off-Heap: 나머지 물리 메모리)
  ├── Lucene 세그먼트 파일
  ├── HNSW 그래프 (.vex)
  ├── 양자화 벡터 (.veq)
  ├── Stored Fields
  └── Doc Values
```

## JVM 힙 핵심 설정 규칙

```yaml
# jvm.options
-Xms30g
-Xmx30g          # Xms = Xmx 반드시 동일
-XX:+UseG1GC
-XX:HeapDumpPath=/var/lib/elasticsearch
-XX:ErrorFile=/var/log/elasticsearch/hs_err_pid%p.log
```

- 힙 = 물리 메모리의 50%
- **최대 ~30.5GB** (Compressed OOPs 한계 — 32GB 이상은 포인터 압축 해제로 오히려 성능 저하)
- 벡터 인덱스 전용 노드 (128GB RAM): 30GB 힙 + 98GB OS 파일 캐시

## OOM 시나리오별 원인-증상-대응

### 시나리오 1: Fielddata 힙 폭발
- **원인**: text 필드에 집계/정렬
- **증상**: `CircuitBreakingException: [fielddata] Data too large`
- **대응**: keyword 필드 또는 doc_values 사용, keyword sub-field 활용

### 시나리오 2: 대형 Aggregation
- **원인**: terms 집계 size 10만+, 중첩 집계 버킷 폭발
- **증상**: `CircuitBreakingException: [request] Data too large`
- **대응**: composite aggregation, 중첩 깊이 제한, transform으로 사전 집계

### 시나리오 3: 벡터 인덱스 메모리 초과
- **원인**: HNSW 그래프 + 벡터 데이터가 OS 파일 캐시 초과
- **증상**: kNN 검색 ms → 초 단위 지연 급증
- **메모리 추정** (float32, 768d, m=16, 1000만 벡터): ~32GB / BBQ 적용 시 ~2.3GB (93% 절약)
- **대응**: 양자화 적용, `_source` 벡터 제외, 벡터 전용 노드 분리

### 시나리오 4: 세그먼트 메타데이터 과다
- **원인**: 샤드 수 과도(수만 개), FST 힙 상주
- **대응**: 노드당 힙 1GB당 샤드 20개 이하, ILM shrink, force_merge

### 시나리오 5: Bulk Rejection (429)
- **원인**: Indexing Pressure 한도 초과
- **대응**: 벌크 크기 축소, 동시성 낮추기, 지수 백오프, 노드 확장

## 서킷 브레이커 구조 및 모니터링

```
total (힙의 95%)       — 전체 메모리 상한
  ├── fielddata (힙의 40%)  — 필드데이터 캐시
  ├── request (힙의 60%)    — 단일 요청 메모리
  └── in_flight_requests (100%) — 전송 중 요청

모니터링:
GET _nodes/stats/breaker
  → tripped 값이 0이 아니면 메모리 압박 징후
```

## Off-Heap 메모리 설계

```
Off-Heap = 총 물리 메모리 - JVM 힙 - OS 커널/오버헤드(~2GB)

예시: 64GB RAM, 30GB 힙
  Off-Heap ≈ 64 - 30 - 2 = 32GB
```

**벡터 인덱스 전용 노드 Heap:Off-Heap 비율**: 1:3 이상 (일반 노드의 1:1과 다름)

```
벡터 노드 메모리 설계 예시 (768d, float32, 1000만 벡터):
  vec ≈ 768 × 4byte × 10M ≈ 28.7GB
  vex ≈ (16 × 2 × 4byte × 10M) ≈ 1.2GB (m=16)
  최소 Off-Heap: 35GB+
  노드 총 메모리: 30GB(힙) + 35GB(Off-Heap) + 2GB(OS) ≈ 67GB+
```

## OOM 발생 시 즉각 대응

```
1. 큰 쿼리/집계 취소
   GET _tasks
   POST _tasks/{id}/_cancel

2. 힙 덤프 분석 (Eclipse MAT)

3. 원인별 대응 (위 시나리오 참조)
```

## ECH/ECE JVM 메모리 압박 수준

| 수준 | 상태 | 대응 |
|------|------|------|
| < 75% | 정상 | 모니터링 유지 |
| 75%~95% | 경고 | GC 빈번 → 원인 분석 및 스케일 업 검토 |
| > 95% | 위험 | 서킷 브레이커 발동 → **즉시 클러스터 리사이즈** |
