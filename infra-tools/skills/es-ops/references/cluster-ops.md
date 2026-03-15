# 클러스터 운영 레퍼런스

## 핵심 모니터링 지표

| 지표 | 정상 기준 |
|------|-----------|
| 클러스터 헬스 | `status: green`, `unassigned_shards: 0` |
| JVM 힙 사용률 | < 75% |
| GC old gen | < 1초/분 |
| CPU | < 80% |
| 디스크 | < 85% |
| indexing_pressure.rejections | 0 |
| search_latency_p99 | SLA 이내 |
| breaker.tripped | 0 |

## 알림 우선순위

| 등급 | 조건 |
|------|------|
| P1 | 클러스터 RED, GC old > 5초 |
| P2 | 힙 > 85% 5분 지속, 디스크 > 85%, 검색 p99 > SLA |
| P3 | Indexing rejection > 0 |

## 디스크 워터마크

| 단계 | 임계값 | 동작 |
|------|--------|------|
| Low | 85% | 신규 샤드 할당 중단 — 즉시 용량 확보 계획 |
| High | 90% | 다른 노드로 샤드 이동 시작 |
| Flood | 95% | 인덱스 자동 **읽기 전용** — 쓰기 불가 |

**Flood 복구 절차:**
```
1. 디스크 공간 확보
2. 읽기 전용 해제:
   PUT /{index}/_settings
   { "index.blocks.read_only_allow_delete": null }
```

## 노드 역할 분리 (대규모 클러스터)

| 역할 | 수량 | 스펙 | node.roles |
|------|------|------|------------|
| 마스터 전용 | 3~5개 | 4C/8GB | `[master]` |
| 코디네이팅 전용 | 2~4개 | 16C/32GB | `[]` |
| Hot 데이터 | N개 | 32C+/64~128GB/NVMe SSD | `[data_hot, data_content, ingest]` |
| Warm 데이터 | N개 | 16C/64GB/대용량 SSD | `[data_warm]` |
| ML (필요시) | N개 | GPU 또는 고성능 CPU | `[ml]` |

## 롤링 리스타트 (무중단)

```
1. 레플리카 샤드 할당 비활성화
   PUT _cluster/settings
   { "persistent": { "cluster.routing.allocation.enable": "primaries" } }

2. Flush
   POST /_flush

3. 노드 종료
   systemctl stop elasticsearch

4. 유지보수

5. 노드 시작
   systemctl start elasticsearch

6. 합류 확인
   GET _cat/nodes?v

7. 샤드 할당 재활성화
   PUT _cluster/settings
   { "persistent": { "cluster.routing.allocation.enable": null } }

8. 헬스 GREEN 확인 후 다음 노드
```

> 디스크 > 85% 노드는 재시작이 느림 — 재시작 전 디스크 사용률 확인.

## 전체 클러스터 재시작

롤링 리스타트와 동일하나 모든 노드 동시 종료.

**핵심 차이점**: **마스터 전용 노드를 먼저 시작** → 쿼럼 형성 대기 → 데이터 노드 시작.

```
GET _cat/health    # yellow까지 대기
GET _cat/nodes     # 전체 노드 합류 확인
GET _cat/recovery  # 복구 진행 모니터링
```

## 노드 추가

1. 동일 `cluster.name` 설정
2. `discovery.seed_hosts`에 기존 마스터 후보 노드 주소 설정
3. 노드 시작 → 자동 합류, 레플리카 자동 할당

## 노드 제거 절차

```
1. 샤드 이동 시작 (데이터 노드만)
   PUT /_cluster/settings
   { "persistent": { "cluster.routing.allocation.exclude._name": "<node_name>" } }

2. 샤드 이동 완료 확인
   GET /_cat/shards?v    # 해당 노드 샤드 0개 확인

3. 노드 종료
   systemctl stop elasticsearch

4. 설정 정리
   PUT /_cluster/settings
   { "persistent": { "cluster.routing.allocation.exclude._name": null } }
```

**마스터 후보 노드 제거 주의사항:**
- 절반 이상 동시 제거 시 클러스터 불능
- **하나씩** 제거하고 재구성 대기
- 2개만 남은 경우 Voting Configuration Exclusion API 사용

## 장애 대응

### 클러스터 RED

```
1. 원인 확인
   GET _cluster/allocation/explain

2. 즉시 조치 (원인별):
   노드 장애    → 복구/교체
   디스크 부족  → 공간 확보
   쿼럼 미달    → 마스터 복구
   할당 실패    → POST _cluster/reroute (allocate_stale_primary)
```

### 검색 지연 급증

```
1. CPU 병목 확인: GET _nodes/hot_threads
2. 큐 적체 확인:  GET _nodes/stats/thread_pool
3. 세그먼트 확인: GET _cat/segments?v&s=size:desc

원인별 대응:
  머지 경합  → 스레드 조정
  GC 지연    → 힙 여유 확보
  디스크 I/O → 메모리 증설
  슬로우 쿼리 → slowlog 활성화
  핫스팟     → 커스텀 라우팅
```

## 프로덕션 필수 설정 체크리스트

| 설정 | 권장값 예시 | 설명 |
|------|------------|------|
| `path.data` | `/var/data/elasticsearch` | `$ES_HOME` 외부 — 업그레이드 시 삭제 방지 |
| `path.logs` | `/var/log/elasticsearch` | `$ES_HOME` 외부 |
| `cluster.name` | `search-prod` | 환경별 고유 이름 |
| `node.name` | `prod-data-1` | 인스턴스별 고유 식별자 |
| `network.host` | `192.168.1.10` | 설정 시 프로덕션 모드 전환 (부트스트랩 체크 강화) |
| `discovery.seed_hosts` | `["master-1", "master-2"]` | 마스터 후보 노드 주소 |
| `cluster.initial_master_nodes` | `["master-a", "master-b", "master-c"]` | **최초 부트스트랩 시만** — 이후 제거 |

## 스레드 풀 (기본값이 대부분 적절)

| 풀 | 크기 |
|----|------|
| search | `(processors × 3) / 2 + 1` |
| write | `processors` |
