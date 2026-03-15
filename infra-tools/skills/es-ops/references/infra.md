# 인프라 운영 레퍼런스 (Kubernetes + Linux)

## Kubernetes 환경 ES 운영

### QoS 클래스와 리소스 설정

| QoS 클래스 | 조건 | OOM Kill 우선순위 | ES 데이터 노드 적합성 |
|------------|------|-------------------|----------------------|
| **Guaranteed** | `requests == limits` | 가장 낮음 (안전) | **권장** |
| Burstable | `requests < limits` | 중간 | 비권장 — 메모리 변동 시 kill 대상 |
| BestEffort | 미설정 | 가장 높음 (위험) | **절대 사용 금지** |

**데이터 노드 권장 설정 (requests = limits → Guaranteed QoS):**

```yaml
resources:
  requests:
    memory: "64Gi"
    cpu: "16"
  limits:
    memory: "64Gi"    # requests와 동일 필수
    cpu: "16"
```

### Heap vs Off-Heap 비율 설계

**컨테이너 메모리 limits: 64Gi 기준:**

| 노드 유형 | JVM Heap | Off-Heap | 비고 |
|-----------|----------|----------|------|
| 일반 검색 노드 | 30GB (50%) | ~32GB | 표준 설정 |
| 벡터 전용 노드 | 16GB (25%) | ~46GB | Off-Heap 최대화 |

> JVM Heap을 너무 줄이면 GC 압박 + 서킷 브레이커 빈번 발동 → **최소 16GB 이상 유지**

### 코디네이팅 노드 분리의 이점 (대규모 벡터 검색)

- 데이터 노드의 Off-Heap을 벡터 캐시에 **전용**으로 사용 가능
- scatter-gather 연산으로 인한 힙 부하를 격리
- 벡터 검색의 DFS Phase 네트워크 라운드트립 부하 분리

```yaml
# 코디네이팅 전용 노드
node.roles: []    # 빈 배열 = 코디네이팅 전용
```

---

## Linux 메모리 모니터링

### 핵심 메모리 지표

| 지표 | 출처 | 의미 |
|------|------|------|
| `MemFree` | `/proc/meminfo` | 완전히 미사용 메모리 (매우 낮은 것이 **정상**) |
| `MemAvailable` | `/proc/meminfo` | **실질적 가용 메모리** (Free + 회수 가능 캐시) |
| `Cached` | `/proc/meminfo` | 페이지 캐시 크기 (ES 성능에 직접 영향) |
| `Active(file)` / `Inactive(file)` | `/proc/meminfo` | 최근 접근/미접근 파일 캐시 비율 |

> `MemFree`가 낮다고 메모리 부족이 아님. Linux는 가용 메모리를 페이지 캐시로 적극 활용.
> **`MemAvailable`이 실질적인 가용 메모리 지표.**

### 메모리 압박 감지

```bash
# 직접 회수(direct reclaim) 발생 여부 — 메모리 압박의 핵심 지표
grep pgscan_direct /proc/vmstat

# pgscan_direct가 지속 증가하면:
#   → 메모리 할당 시 즉시 할당 불가
#   → 커널이 동기적으로 페이지 회수 중
#   → I/O 대기 발생 → 지연 증가

# page fault 모니터링
grep pgfault /proc/vmstat     # minor + major fault 총합
grep pgmajfault /proc/vmstat  # major fault (디스크 I/O 필요)
```

| 지표 | 정상 | 이상 |
|------|------|------|
| `pgscan_direct` | 0 또는 미증가 | 지속 증가 → **Off-Heap 부족** |
| `pgmajfault` | 낮은 빈도 | 급증 → 페이지 캐시 미스 다발 |

### 벡터 인덱스 노드 모니터링 체크리스트

```bash
# 1. Off-Heap 여유 확인
cat /proc/meminfo | grep -E "MemTotal|MemAvailable|Cached|Active\(file\)|Inactive\(file\)"

# 2. 메모리 압박 확인
grep -E "pgscan_direct|pgmajfault" /proc/vmstat

# 3. ES 노드 메모리 사용 (API)
GET _nodes/stats/os?filter_path=nodes.*.os.mem

# 4. 벡터 인덱스 크기 확인
GET _cat/indices/my-vector-*?v&h=index,store.size,pri.store.size
```

### 알림 기준

| 조건 | 등급 | 대응 |
|------|------|------|
| `MemAvailable` < 총 메모리의 10% | P2 | Off-Heap 부족 → 노드 증설 또는 인덱스 분산 |
| `pgscan_direct` 지속 증가 | P2 | 메모리 압박 → 즉시 원인 분석 |
| `pgmajfault` 급증 + 벡터 검색 p99 급등 | P1 | 페이지 캐시 미스 → 워밍업/Off-Heap 확보 |

### 인덱스 삭제와 메모리 반환

인덱스 삭제(`DELETE`) 또는 닫기(`POST _close`) 시 `munmap` 시스템 콜로 메모리 맵핑 **즉시 해제** → 페이지 캐시 즉시 반환.

| 동작 | 메커니즘 | 특성 |
|------|----------|------|
| 인덱스 삭제/닫기 | `munmap` → 즉시 해제 | 예측 가능, 즉시 반환 |
| 메모리 압박에 의한 회수 | LRU eviction | OS 판단에 의존, 비예측적 |

> 벡터 인덱스 alias 전환 후 **구 인덱스를 삭제하면 해당 vec/vex 파일의 캐시가 즉시 반환**되어 새 인덱스의 캐시 공간 확보.
