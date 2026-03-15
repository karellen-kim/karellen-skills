---
name: es-ops
description: >
  Elasticsearch 대규모 운영 전문가 스킬. 빅테크 프로덕션 환경에서 ES 클러스터를 안정적으로 운영하기 위한 실전 지식을 제공합니다.
  사용자가 다음과 관련된 질문을 할 때 반드시 사용하세요:
  - 색인 성능 문제 (벌크 색인, 429 rejection, refresh/translog 설정)
  - 쿼리 성능 문제 (Deep Pagination, 느린 집계, 슬로우 로그)
  - 벡터/임베딩 인덱스 운영 (HNSW, bbq_hnsw, 양자화, Off-Heap, vec/vex 파일)
  - 메모리 이슈 (OOM, 서킷 브레이커, JVM 힙, 페이지 캐시)
  - 클러스터 운영/장애대응 (RED 클러스터, 노드 추가/제거, 롤링 리스타트)
  - 샤드 설계, ILM, 스냅샷, CCR, Autoscaling
  - Kubernetes 환경에서의 ES 운영 (QoS, Off-Heap 설계)
  - Linux 메모리 모니터링 (페이지 캐시, pgscan_direct, pgmajfault)
  "ES 설정 어떻게 해?", "클러스터가 RED야", "벡터 검색이 느려", "OOM 났어", "샤드 몇 개로 해야 해?" 같은 말이 나오면 즉시 트리거.
---

# Elasticsearch 대규모 운영 스킬

빅테크 환경에서 ES를 프로덕션 수준으로 운영하기 위한 실전 지식 베이스.

## 스킬 사용 방법

이 스킬은 주제별 참조 파일로 구성되어 있습니다. 사용자 질문의 주제를 파악한 후, 해당 참조 파일을 읽어 정확한 정보를 제공하세요.

### 주제-파일 매핑

| 사용자 질문 유형 | 참조 파일 |
|---|---|
| 색인 성능, 벌크 색인, refresh, translog, 429 | `references/indexing.md` |
| 쿼리 최적화, Pagination, Aggregation, Slow Log | `references/querying.md` |
| 벡터 인덱스, HNSW, 양자화, 임베딩 재색인 | `references/vector.md` |
| OOM, 서킷 브레이커, JVM 힙, 메모리 이슈 케이스 | `references/memory.md` |
| 클러스터 운영, 노드 역할, 장애 대응, 롤링 리스타트 | `references/cluster-ops.md` |
| 스냅샷, CCR, Autoscaling, Searchable Snapshot | `references/data-management.md` |
| K8s 환경, QoS, 컨테이너 설정, Linux 메모리 모니터링 | `references/infra.md` |

### 복합 질문 처리

여러 주제가 얽힌 경우 (예: "벡터 인덱스 OOM") 관련 파일을 모두 읽어 통합된 답변 제공.

## 응답 원칙

1. **명령어 우선**: 진단/해결은 항상 실행 가능한 API 명령 또는 설정값과 함께 제시
2. **원인-증상-대응** 구조로 장애 대응 설명
3. **수치 기반**: "충분히" 대신 구체적 수치 제시 (예: "힙의 50%, 최대 ~30GB")
4. **트레이드오프 명시**: 설정 변경의 장단점을 함께 설명
5. **프로덕션 안전**: 데이터 유실 위험이 있는 작업은 명확히 경고
