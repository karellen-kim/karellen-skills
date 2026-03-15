# 데이터 관리 레퍼런스

## Snapshot & Restore

### 기본 원칙

> **파일 시스템 수준 백업은 지원되지 않음.** Snapshot API가 유일하게 안전한 백업 방법.

**스냅샷에 포함되는 것:**
- 클러스터 상태 (persistent 설정, 인덱스 템플릿, ILM, 인제스트 파이프라인)
- Feature States (security, kibana, fleet 등)
- 모든 데이터 스트림 및 인덱스

**포함되지 않는 것:**
- Transient 클러스터 설정
- 등록된 스냅샷 리포지토리 설정
- 노드 설정 파일 (`elasticsearch.yml`)

### SLM 다중 주기 전략

| 정책 | 주기 | 보관 기간 | max_count |
|------|------|-----------|-----------|
| `hourly-snapshots` | 매시간 | 1일 | 24 |
| `daily-snapshots` | 매일 23:45 | 30일 | 31 |
| `monthly-snapshots` | 매월 1일 | 1년 | 12 |

```
PUT _slm/policy/nightly-snapshots
{
  "schedule": "0 30 1 * * ?",
  "name": "<nightly-snap-{now/d}>",
  "repository": "my_repository",
  "config": {
    "indices": "*",
    "include_global_state": true
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}
```

### 스냅샷 복원

```
# 특정 인덱스 복원
POST _snapshot/my_repository/my_snapshot/_restore
{ "indices": "my-index" }

# 이름 변경하며 복원 (기존 데이터와 비교)
POST _snapshot/my_repository/my_snapshot/_restore
{
  "indices": "my-index",
  "rename_pattern": "(.+)",
  "rename_replacement": "restored-$1"
}

# 복원 모니터링
GET _cluster/health
GET {index}/_recovery
GET _cat/shards?v&h=index,shard,prirep,state,node,unassigned.reason&s=state
```

**복원 주의사항:**
- 동일 이름의 열린 인덱스가 있으면 복원 불가 → 삭제 후 복원 또는 이름 변경
- 다른 클러스터로 복원 시 리포지토리를 **read-only**로 등록

### 전체 클러스터 복원 절차

```
1. ILM 중지:          POST _ilm/stop
   ML 업그레이드 모드:  POST _ml/set_upgrade_mode?enabled=true
   Watcher 중지:       POST _watcher/_stop

2. 와일드카드 삭제 허용:
   PUT _cluster/settings
   { "persistent": { "action.destructive_requires_name": false } }

3. 기존 데이터 삭제:
   DELETE _data_stream/*?expand_wildcards=all
   DELETE *?expand_wildcards=all

4. 스냅샷 복원:
   POST _snapshot/my_repository/my_snapshot/_restore
   { "indices": "*", "include_global_state": true }

5. 기능 재활성화:
   POST _ilm/start
   POST _ml/set_upgrade_mode?enabled=false
   POST _watcher/_start
```

### Searchable Snapshots

| 옵션 | 티어 | 특성 |
|------|------|------|
| Fully Mounted | Hot, Cold | 전체 캐시 → 일반 인덱스와 유사한 검색 성능 |
| Partially Mounted | Frozen | 최근 검색 데이터만 로컬 캐시, 나머지는 리포지토리에서 fetch |

**운영 주의사항:**
- **레플리카 불필요** → 스냅샷 자체가 복원력 제공 (스토리지 50% 절감)
- `force_merge`를 1 segment로 수행 후 스냅샷 → 읽기 횟수 최소화
- 동일 리전의 리포지토리만 마운트 (크로스 리전은 데이터 전송 비용)
- **원본 스냅샷 삭제 금지** → 유일한 전체 데이터 복사본
- 롤링 리스타트 시 할당 비활성화 필수 (미설정 시 전체 스냅샷 재다운로드)

---

## Cross-Cluster Replication (CCR)

### 아키텍처 패턴

| 패턴 | 설명 | 사용 사례 |
|------|------|-----------|
| 단방향 DR | DC A(Leader) → DC B(Follower) | 기본 재해 복구 |
| 다중 DR | DC A → DC B + DC C | 고가용성 + DR |
| 양방향 복제 | DC A ↔ DC B | 수동 페일오버 불필요 |
| 중앙 집계 | DC A/B/C → 중앙 클러스터 | 글로벌 분석 |

**핵심 특성:**
- Follower는 **읽기 전용** → 매핑, alias 변경 불가 (Leader에서만 수정)
- `index.soft_deletes.retention_lease.period`: 기본 12시간 → Follower 클러스터 최대 오프라인 허용 시간

### CCR 복제되지 않는 항목

- 시스템 인덱스, ML 작업, 인덱스 템플릿, ILM/SLM 정책
- 사용자 권한 및 역할 매핑, 클러스터 설정, Searchable Snapshot 인덱스

> 보안 설정은 각 클러스터에서 **독립적으로** 구성. 정기적으로 `security` feature state 스냅샷으로 백업.

### 버전 호환성

단방향 구성에서 Follower는 Leader와 **동일하거나 최신** 버전이어야 함.

| Leader | Follower 호환 |
|--------|---------------|
| 7.17 | 7.17, 8.x, 9.x |
| 8.x–9.x | 8.x–9.x |

---

## Autoscaling

### Decider 종류

| Decider | 대상 | 기준 |
|---------|------|------|
| `reactive_storage` | 데이터 노드 | 현재 스토리지 요구량 (반응형) |
| `proactive_storage` | Hot 노드 | 수집 속도 기반 예측 (선제형) |
| `frozen_shards` | Frozen 노드 | 부분 마운트 샤드 수 기반 메모리 |
| `ml` | ML 노드 | ML 작업 메모리/CPU 요구량 |

### 정책 예시

```
# Hot 티어 선제적 스토리지 autoscaling
PUT /_autoscaling/policy/my_hot_policy
{
  "roles": ["data_hot"],
  "deciders": {
    "proactive_storage": { "forecast_window": "10m" }
  }
}

# ML 노드 autoscaling
PUT /_autoscaling/policy/my_ml_policy
{
  "roles": ["ml"],
  "deciders": {
    "ml": {
      "num_anomaly_jobs_in_queue": 5,
      "down_scale_delay": "30m"
    }
  }
}
```
