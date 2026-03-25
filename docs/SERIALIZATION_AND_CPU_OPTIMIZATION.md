# 직렬화·단순화 관점의 병목/안정성 및 CPU 최적화

**목적**: Facade·Controller·Repository·Validator의 직렬화 구조와 CPU 단일보드 환경에서의 성능·안정성 포인트를 정리한다.  
**관련**: [ENGINE_DECOMPOSITION_DESIGN.md](ENGINE_DECOMPOSITION_DESIGN.md), [STRUCTURAL_GOALS.md](STRUCTURAL_GOALS.md).

---

## 1. 직렬화·단순화 관점에서의 병목/안정성 포인트

### 1.1 Facade Mutex 직렬화

| 항목 | 내용 |
|------|------|
| **역할** | 모든 엔진 호출이 단일 Mutex로 직렬화 → 레이스 제거 |
| **효과** | 안정성 ↑, 동시성 ↓ |
| **특성** | CPU 단일보드 목표에는 적합. 분석과 보드 색상/합법수 조회가 동시에 들어오면 대기 시간 증가 가능. |
| **참조** | `KataGoEngineFacade` |

### 1.2 EngineController의 단일 실행 큐

| 항목 | 내용 |
|------|------|
| **역할** | Repository 호출을 Controller 내부 큐로 직렬화 |
| **특성** | Facade Mutex와 겹쳐 **이중 직렬화** 가능성. 안정성은 높으나 성능 비용 여지 있음. |
| **참조** | `EngineController` |

### 1.3 서비스 경계 단일화

| 항목 | 내용 |
|------|------|
| **역할** | EngineRepository가 서비스 바인딩을 단일화 |
| **효과** | DeadObject/바인딩 불안정 오류를 구조적으로 감소 |
| **참조** | `EngineRepository` |

### 1.4 스냅샷 기반 결과 게이트

| 항목 | 내용 |
|------|------|
| **역할** | requestId / movesHash 불일치 결과는 즉시 폐기 |
| **효과** | “오래된 결과가 UI 상태를 오염시키는 오류”를 구조적으로 차단 |
| **참조** | `AnalysisSnapshotValidator` |

---

## 2. CPU 성능 최적화 관점 (구조 기반 개선 방향)

### 2.1 직렬화 구간 최소화

| 항목 | 내용 |
|------|------|
| **현상** | 퍼사드 Mutex와 엔진 컨트롤러 큐가 모두 직렬화에 관여 |
| **방향** | CPU 단일보드 관점에서는 **단일 직렬화 레이어**만 두는 구조가 가장 단순하고 빠름. |
| **비고** | Facade Mutex vs Controller 큐 중 하나로 수렴하는 설계 검토. |

### 2.2 분석 외 호출의 우선순위 구조화

| 항목 | 내용 |
|------|------|
| **현상** | 분석이 길어질 때, 정책/소유권/보드색 조회 같은 **짧은 호출**이 대기열에서 지연될 수 있음. |
| **방향** | 단일 큐를 유지하되 **“짧은 호출 우선 처리”** 설계로 CPU 체감 성능 향상. |
| **참조** | `EngineController` |

### 2.3 분석 실행의 보호 윈도우 최적화

| 항목 | 내용 |
|------|------|
| **현상** | AnalysisStateMachine이 최소 보호 시간을 둠. CPU 한정에서는 과도한 보호 시간이 응답성을 떨어뜨릴 수 있음. |
| **방향** | 보호 시간은 **“오류 방지”와 “실시간성”** 사이의 핵심 파라미터. 튜닝 대상. |
| **참조** | `AnalysisStateMachine`, `AnalysisScheduler` |

### 2.4 JNI 데이터 풀 재사용 유지

| 항목 | 내용 |
|------|------|
| **역할** | NativeBridge의 IntArray 풀로 GC 부담 감소, CPU 캐시 효율 향상 |
| **효과** | CPU-only 최적화에서 중요한 **할당 억제**가 이미 반영됨. 유지 권장. |
| **참조** | `NativeBridge` |

---

## 3. 핵심 결론 (구조 요약)

### 3.1 현재 강점

- **CPU 단일보드/직렬화/단순화** 관점에서 이미 적절한 구조:
  - 단일 퍼사드
  - 단일 이벤트 큐
  - 단일 UI 상태 전이
  - JNI 경계 안전성 (NativeSafetyGate, EngineCalls)

### 3.2 구조적 오류 방지

- **단일 진입 경로 고정**과 **스냅샷 게이트**가 핵심이며 이미 구현됨.

### 3.3 CPU 성능 최적화 우선순위

| 순위 | 방향 | 효과 |
|------|------|------|
| 1 | **직렬화 레이어 중복 제거** | Facade vs Controller 큐 단일화 검토 |
| 2 | **짧은 호출 우선 처리** | 정책/소유권/보드색 등 짧은 호출 대기 시간 감소 |
| 3 | **분석 보호 윈도우 조정** | 오류 방지와 실시간성 균형 파라미터화 |

---

## 4. 구현 체크리스트 (참고용)

| 항목 | 상태 | 비고 |
|------|------|------|
| 직렬화 레이어 단일화 | 완료 | EngineController 내부 큐 제거, Facade 단일 워커만 직렬화 |
| 짧은 호출 우선 처리 | 완료 | KataGoEngineFacade: shortChannel/longChannel, 워커가 short 우선 처리 |
| 보호 윈도우 파라미터화 | 완료 | EngineAnalyzeConstants.MIN_ANALYSIS_PROTECT_MS 단일 상수·KDoc으로 튜닝 지점 명시 |
| JNI 풀 유지 | 유지 | NativeBridge + NativeSafetyGate 현 구조 유지 |
