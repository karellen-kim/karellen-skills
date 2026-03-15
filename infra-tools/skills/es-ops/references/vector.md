# 벡터/임베딩 인덱스 운영 레퍼런스

## 벡터 인덱스 설계 원칙

| 결정 항목 | 권장값 | 이유 |
|-----------|--------|------|
| 양자화 타입 | `bbq_hnsw` (768d+) | 메모리 96% 절약, rescore로 정확도 보완 |
| `_source` 벡터 필드 | 제외 | Fetch Phase 성능 + 디스크 절약 |
| refresh_interval | `30s` | HNSW 그래프 빌드 비용 고려 |
| 샤드 크기 | 5~20GB | 벡터 머지 CPU 비용이 텍스트보다 높음 |
| Merge 스레드 | 2 이하 | HNSW 재구축이 CPU 집약적 |

## 양자화 전략 선택 기준

| 문서 수 | 양자화 타입 | 특성 |
|---------|-------------|------|
| 100만 미만 | `int8_hnsw` | 정확도 우선 |
| 100만~1000만 | `bbq_hnsw` | 메모리-정확도 균형 |
| 1000만+ | `bbq_disk` | 디스크 기반, 비용 최적 |

## 동적 색인 시 HNSW 그래프 영향

- refresh마다 새 세그먼트 = 새 HNSW 그래프 → 세그먼트 수 ↑ = 검색 지연 ↑
- 세그먼트 머지 시 HNSW 그래프 **재구축** → 일반 역색인 머지보다 10~100배 느림
- 작은 세그먼트의 양자화 품질 저하 가능

**대응 전략:**
1. `refresh_interval` 늘리기 (30s~60s)
2. 머지 정책 조정: `max_merged_segment` 2~5GB
3. 벌크 색인 후 `force_merge` (읽기 전용만)
4. 색인 노드와 검색 노드 분리

## 모델 변경 시 무중단 재색인 절차

```
1. 새 인덱스 생성 (새 모델 차원/설정)
2. 백그라운드 재색인 + 파이프라인으로 새 모델 임베딩 생성
3. 전환 중 동시 쓰기 (v1 + v2 인덱스 모두)
4. alias 원자적 전환 (POST _aliases)
5. 구 인덱스 삭제
```

## 하이브리드 검색 (벡터 + 텍스트)

- **RRF (Reciprocal Rank Fusion)** 권장 → 점수 스케일 차이 자동 해결
- kNN은 DFS Phase 사용 → 추가 네트워크 라운드트립 발생
- 벡터 검색은 CPU 집약적 → 검색 스레드 풀 모니터링 필수
- 코디네이팅 노드 분리 권장

## vec/vex/veq 파일 이해

| 파일 | 내용 | 페이지 캐시 요구 |
|------|------|-----------------|
| `.vec` | 원본 벡터 데이터 (raw float/byte) | 높음 — rescore 시 필요 |
| `.vex` | HNSW 그래프 구조 (노드 연결 정보) | **필수** — 검색 시 항상 접근 |
| `.veq` | 양자화된 벡터 (int8, bbq 등) | 높음 — 양자화 검색 시 사용 |

> HNSW 그래프는 탐색 중 랜덤 액세스. 그래프 **전체가 페이지 캐시에** 있어야 함.
> 일부만 적재된 상태 → 각 홉마다 디스크 I/O → ms → 초 단위 지연 급증.

## 파일 크기 손계산 공식

```
veq 크기 = replica_count × doc_count × bytes_per_element × dim
vex 크기 = replica_count × doc_count × (2 × m) × 4bytes

bytes_per_element: float32=4, int8=1, bbq≈0.125 (1bit)
2×m: Level 0 edge 수 (상한값으로 안전하게 과대 추정)
```

**실전 예시** (`embedding-commerce-search-taca`, int8, 96d, m=12, replica=1, 8천만 문서):
```
veq = 2 × 80M × 1 × 96  ≈ 14.3GB
vex = 2 × 80M × 24 × 4  ≈ 14.3GB
합계 ≈ 28.6GB  (실제 API 확인값: ~27GB — 손계산과 근접)
```

## 파일 크기 API 확인

```
GET {index}/_stats/segments?include_segment_file_sizes=true
GET /_cat/segments/{index}?v&h=index,shard,segment,size
```

## cosine vs dot_product

벡터를 사전 정규화(L2 norm=1)했다면 두 함수의 결과는 동일.
**색인 전 벡터 정규화 + `dot_product` 사용** → 검색 시 정규화 단계 스킵 → 약간의 성능 이점.

## 정적 색인 파이프라인 권장 순서

```
1. 인덱스 생성
   - number_of_replicas: 0
   - refresh_interval: "-1"
   - translog.durability: "async"

2. 벌크 색인

3. refresh 실행
   POST /{index}/_refresh

4. force_merge (필수 — alias 전환 전에 수행)
   POST /{index}/_forcemerge?max_num_segments=1

5. 레플리카 복원
   PUT /{index}/_settings { "number_of_replicas": 1 }

6. preload/warmup 실행 (vec, vex, veq)

7. alias 원자적 전환

8. 구 인덱스 삭제 (캐시 즉시 반환)
```

> force merge를 **alias 전환 전에** 수행. 전환 후 수행하면 머지 중 검색 성능 불안정.

## preload 설정

```
PUT /{index}/_settings
{
  "index.store.preload": ["vec", "vex", "veq"]
}
```

**주의**: Off-Heap이 부족한 상태에서 preload 설정 시 → 기존 인덱스 캐시를 밀어내는 **풍선 효과** 발생.

## 검색 튜닝

- `num_candidates` = k × 10 (기본 시작값)
- `early_termination` 활용
- `num_candidates`를 올릴수록 recall ↑, 속도 ↓ — 트레이드오프
