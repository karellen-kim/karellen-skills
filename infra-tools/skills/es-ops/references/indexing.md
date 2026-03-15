# 색인 운영 레퍼런스

## 초기 대량 적재 (Bulk Load) 최적 순서

| 단계 | 설정 | 이유 |
|------|------|------|
| 1. 인덱스 생성 | `number_of_replicas: 0` | 복제 I/O 제거 |
| 2. refresh 비활성화 | `refresh_interval: "-1"` | 세그먼트 생성 오버헤드 제거 |
| 3. Translog 완화 | `translog.durability: "async"` | fsync 비용 절감 |
| 4. 벌크 요청 | 요청당 5~15MB, 1000~5000 문서 | 최적 배치 크기 |
| 5. 병렬 처리 | 코어 수 × 1~2 스레드 | CPU 활용 극대화 |
| 6. 완료 후 복원 | `refresh_interval: "1s"`, `replicas: 1` | 정상 운영 복귀 |
| 7. (읽기 전용만) | `_forcemerge?max_num_segments=1` | 이후 검색 최적화 |

> ⚠️ `translog.durability: "async"` 는 노드 비정상 종료 시 데이터 유실 가능. 초기 적재 후 반드시 복원.

## 실시간 스트림 색인 권장 설정

| 항목 | 권장값 | 이유 |
|------|--------|------|
| refresh_interval | `5s` ~ `30s` | 기본 1s는 과도한 세그먼트 생성 |
| translog.durability | `request` (기본 유지) | 데이터 유실 방지 |
| 벌크 크기 | 5~10MB | 네트워크 효율 + 지연 균형 |
| replicas | 1 이상 | HA 보장 |

## 샤드 수 결정 공식

```
Primary 샤드 수 = ceil(예상 총 데이터 크기 / 목표 샤드 크기)

목표 샤드 크기:
  일반 검색:   10~30GB
  로그/시계열: 30~50GB
  벡터 인덱스: 5~20GB  (HNSW 그래프 메모리 고려)
```

**노드당 샤드 수 상한**: 힙 1GB당 20개 이하

## 시계열 데이터 ILM 정책

```
Hot:    max_size: 50GB 또는 max_age: 1d → rollover
Warm:   min_age: 7d  → shrink(1 shard) + force_merge(1 segment)
Cold:   min_age: 30d → searchable_snapshot
Delete: min_age: 90d → 삭제
```

## Indexing Pressure 모니터링 및 429 대응

```
GET _nodes/stats/indexing_pressure

임계값:
  coordinating_in_bytes > 힙의 10% → 429
  primary_in_bytes      > 힙의 10% → 429
  replica_in_bytes      > 힙의 15% → 429
```

**429 대응 순서:**
1. 벌크 요청 크기 줄이기
2. 클라이언트 스레드 수 줄이기
3. 지수 백오프 재시도 적용
4. 노드 수평 확장

## `refresh_interval: "-1"` 에서도 세그먼트가 생성되는 경우

1. **Indexing Buffer 초과**: `indices.memory.index_buffer_size` (기본: 힙의 10%) 초과 시 flush → 새 세그먼트
2. **Translog 크기 초과**: `index.translog.flush_threshold_size` (기본: 512MB) 초과 시 flush → 새 세그먼트

벡터 인덱스 대량 적재 시 이 두 트리거를 인지하고 세그먼트 수를 모니터링할 것.
