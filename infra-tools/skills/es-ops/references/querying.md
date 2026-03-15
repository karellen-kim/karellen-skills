# 쿼리 운영 레퍼런스

## 검색 성능 최적화 체크리스트

| 항목 | 최적화 방법 | 영향도 |
|------|-------------|--------|
| `filter` vs `must` | 점수 불필요한 조건은 `filter`에 | 높음 — 캐시 + 점수 생략 |
| `_source` 필터링 | 필요한 필드만 반환 | 중간 — 네트워크 절감 |
| `size` 제한 | 필요한 만큼만 | 중간 — 힙 절감 |
| `track_total_hits` | `false` 또는 정수 | 높음 — 전체 카운트 비용 제거 |
| 인덱스 정렬 | 자주 정렬하는 필드로 미리 정렬 | 높음 — early termination |
| `profile: true` | 병목 쿼리 진단 시 사용 | 진단용 |

## Pagination 방식 비교

| 방식 | 용도 | 주의 |
|------|------|------|
| `from + size` | 소량 페이지네이션 | `from: 100000+`은 OOM 위험 |
| `search_after + PIT` | 안전한 대량 페이지네이션 | **권장** |
| `scroll` | 대량 export 전용 | keep_alive 최소화 |

> `max_result_window` 조정 금지 — 기본 10,000은 OOM 방지 보호 장치.

## Aggregation 메모리 관리

- 고카디널리티 terms → **composite aggregation** + 페이지네이션
- 중첩 집계 폭발 방지: 깊이/버킷 크기 제한
- 스크립트 기반 집계 → **runtime field** 또는 색인 시 전처리로 대체

## 쿼리 타임아웃 설정

```
요청 수준:     GET /index/_search?timeout=10s
클러스터 수준: PUT _cluster/settings
               { "persistent": { "search.default_search_timeout": "30s" } }

장시간 쿼리 취소:
  GET _tasks?actions=*search*
  POST _tasks/{id}/_cancel
```

## Slow Log 설정 및 운영

### 검색 Slow Log 설정

```
PUT /my-index/_settings
{
  "index.search.slowlog.threshold.query.warn":  "10s",
  "index.search.slowlog.threshold.query.info":  "5s",
  "index.search.slowlog.threshold.query.debug": "2s",
  "index.search.slowlog.threshold.query.trace": "500ms",
  "index.search.slowlog.threshold.fetch.warn":  "1s",
  "index.search.slowlog.include.user": true
}
```

### 색인 Slow Log 설정

```
PUT /my-index/_settings
{
  "index.indexing.slowlog.threshold.index.warn":  "10s",
  "index.indexing.slowlog.threshold.index.info":  "5s",
  "index.indexing.slowlog.source": "1000",
  "index.indexing.slowlog.include.user": true
}
```

### 현재 설정 확인

```
GET _all/_settings?expand_wildcards=all&filter_path=*.settings.index.*.slowlog
```

### Slow Log 운영 원칙

| 원칙 | 이유 |
|------|------|
| 트러블슈팅 시에만 활성화 | 클러스터에 부하 |
| **특정 인덱스**에만 설정 | 로그 볼륨 최소화 |
| 높은 임계값(30s)부터 시작 | 점진적으로 낮춰 원인 파악 |
| `X-Opaque-ID` 헤더 활용 | Slow Log에 자동 포함되어 요청 추적 가능 |

> Slow Log는 완료된 이벤트만 기록. 서킷 브레이커로 중단된 쿼리는 기록되지 않으므로, 이 경우 감사 로그(Audit Logging) 추가 활성화.

### 초기 트러블슈팅 시작점

```
PUT /*/_settings
{
  "index.search.slowlog.include.user": true,
  "index.search.slowlog.threshold.query.warn": "30s",
  "index.search.slowlog.threshold.fetch.warn": "30s"
}
```
