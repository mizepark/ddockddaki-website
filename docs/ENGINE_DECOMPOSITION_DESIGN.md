# 엔진 분해 구조 설계 — “확실하게 분해”

**목적**: 한 번에 분해 방향을 제시하고, 수정 없이 구조 설계만 문서화한다.  
**원칙**: 책임 분리 → 부분 업데이트 제거 → 상태 불일치·동시성 오류 경로를 구조적으로 제거.

**구조적 봉쇄 7가지**(엔진 경로·리듀서·단일 큐·스냅샷·네이티브·도메인 타입·프로세스)는 [STRUCTURAL_GOALS.md](STRUCTURAL_GOALS.md) 참고.  
**직렬화·병목·CPU 최적화** 관점은 [SERIALIZATION_AND_CPU_OPTIMIZATION.md](SERIALIZATION_AND_CPU_OPTIMIZATION.md) 참고.

---

## 구현 완료 (현재 코드베이스)

| 컴포넌트 | 파일 | 비고 |
|----------|------|------|
| **RequestIdGenerator** | `RequestIdGenerator.kt` | 세션/종류별 ID 생성. ViewModel에서 사용. |
| **AnalysisSnapshotValidator** | `AnalysisSnapshotValidator.kt` | 분석 결과·정책·소유권·게임 스냅샷 게이트. `AnalysisResultGate` 위임. |
| **UiStateReducer** | `UiStateReducer.kt` | Init/Config/Policy/Ownership/GameStones/Overlay 순수 전이. ViewModel이 `_uiState.update { Reducer.apply*(...) }` 로 적용. |
| **EngineRuntimeCoordinator** | `EngineRuntimeCoordinator.kt` | InitCoordinator + Facade 래핑. ensureEngineReady, shutdown, initEngineForRecovery, restartAndEnsureReady. |
| **SessionStateCoordinator** | `SessionStateCoordinator.kt` | EngineSessionOrchestrator + LifecycleCoordinator 래핑. switchSession, getSavedGameKey. |
| **PolicyFetcher** | `PolicyFetcher.kt` | 정책 요청 ID + getPolicyHeatmap 위임. |
| **OwnershipFetcher** | `OwnershipFetcher.kt` | 소유권 요청 ID + getRootLead/getOwnershipHeatmap 위임. |
| **EngineOrchestrator** | `EngineOrchestrator.kt` | `runtimeCoordinator`·`sessionStateCoordinator` 사용. Init/Switch/ChangeConfig 이벤트를 위 코디네이터로 분산. |
| **LatencyController** | `LatencyController.kt` | prefMin/Target/Max, isLatencyLocked. computeDynamicLatencyMs, latencySeconds. |
| **VisitsBudgeter** | `VisitsBudgeter.kt` | computeAdaptiveVisits(목표 지연, msPerVisit, 모델/보드, visitsMin/Max). |
| **AnalysisRunner** | `AnalysisRunner.kt` | RunParams + run(engineFacade, params). 분석 1회 실행만 담당. |
| **AnalysisIntentFilter** | `AnalysisIntentFilter.kt` | shouldAccept(intent, lastRunIntent, isRunning, pending). 의도 유효성만 판정. |
| **AnalysisScheduler** | `AnalysisScheduler.kt` | decide(intent) → RunNow \| Defer(remainingMs). markRun, scheduleDeferred. 보호 구간·중복 방지. |
| **EngineStateActor** | `EngineStateActor.kt` | 단일 채널로 상태 전이 순차 실행. runOnStateThread, runOnStateThreadAndWait. (기존 SingleThreadExecutor 대체) |
| **OverlayStateController** | `OverlayStateController.kt` | setOverlayMode, clearOverlay, setVariationPopup, setSuspendUi. 오버레이 전용 순수 전이. |
| **EngineCalls** | `EngineCalls.kt` | executeWithTimeout, executeWithRetry. AIDL 호출 타임아웃/재시도/requestId·movesHash 태깅. EngineRepository에서 analyzeTop4·getPolicy·getOwnership 등 래핑. |
| **NativeSafetyGate** | `NativeSafetyGate.kt` | withIntArray(size) { block }. 배열 풀 사용 전 준비/사용 후 해제 강제. ViewModel의 withMoveInts에서 사용. |
| **EngineBootstrap** | `EngineBootstrap.kt` | Controller/Repository 생성·주입만 수행. App은 EngineFacade만 노출(엔진 경로 컴파일 타임 봉쇄). |

AnalysisStateMachine은 AnalysisScheduler·AnalysisIntentFilter 사용으로 리팩터 완료.

---

## 1. 즉시 적용 가능한 분해 구조 (5개 코어)

| 컴포넌트 | 책임 | 현재 대응 |
|----------|------|-----------|
| **EngineRuntimeCoordinator** | 엔진 초기화 / 재시작 / 헬스체크만 담당 | EngineInitCoordinator + Facade 호출을 이 레이어로 고정 |
| **SessionStateCoordinator** | 세션 전환 / 저장 / 복원만 담당 | EngineSessionOrchestrator + LifecycleCoordinator 통합 |
| **AnalysisRunner** | 분석 실행 / 중단 / 결과 수집만 담당 | `runAnalysisInternal` + 결과 게이팅 로직 분리 |
| **UiStateReducer** | UI 상태 변경은 여기서만 수행 | ViewModel은 이벤트만 전달하고, 결과만 적용 |
| **EventLoop (Orchestrator)** | 단일 이벤트 큐 유지, 이벤트 처리 결과를 위 4개로 분산 | 기존 Orchestrator 유지, 처리 위임만 재배치 |

- ViewModel: **이벤트 라우팅만** 담당. 상태를 직접 바꾸지 않음.
- 각 컴포넌트는 **단일 책임** → 상태 전이 분기 수 감소.
- 이벤트 루프만 남기고 나머지는 **입력→출력** 구조 → 레이스/동시성 오류를 구조적으로 차단.

---

## 2. 왜 “오류 0”에 가까워지는가

- **책임 분리** → “부분 업데이트”가 사라지고, 상태 불일치 경로가 구조적으로 제거된다.
- **단일 책임** → 각 컴포넌트의 상태 전이 분기 수가 급격히 감소한다.
- **이벤트 루프 + 순수 입력→출력** → 동시성 오류가 구조적으로 차단된다.

---

## 3. 권장 분해 순서 (중복 작업 없이)

| 순서 | 대상 | 이유 |
|------|------|------|
| **1** | **UiStateReducer** | UI 업데이트 경로를 먼저 고정하면 이후 분해가 안전해짐 |
| **2** | **AnalysisRunner** | 분석 실행/중단/결과 처리 변수를 격리 |
| **3** | **SessionStateCoordinator** | 세션 전환은 오류 위험이 크므로 독립 |
| **4** | **EngineRuntimeCoordinator** | 초기화/재시작 경로 완전 격리 |
| **5** | **ViewModel → 이벤트 라우팅만** | 상태를 직접 바꾸지 않고 이벤트만 전달 |

---

## 4. 핵심 분해 추가

### 4.1 Policy / Ownership 파이프라인

| 컴포넌트 | 책임 |
|----------|------|
| **PolicyFetcher** | 정책 히트맵 요청/응답만 담당. 요청 ID·보드 크기 스냅샷 체크(게이트) 로직을 클래스 내부로 이동 → 중복·누락 제거 |
| **OwnershipFetcher** | 소유권/리드 요청/응답만 담당. 동일하게 게이트 로직 내부화 |

- 현재 ViewModel에 흩어진 policy/ownership fetch + `isSamePolicyOverlaySnapshot` / `isSameOwnershipOverlaySnapshot` 를 각 Fetcher로 이전.

### 4.2 Intent 필터 / 스케줄러

| 컴포넌트 | 책임 |
|----------|------|
| **AnalysisIntentFilter** | 의도 유효성 판정만 수행 (어떤 의도를 받아들일지) |
| **AnalysisScheduler** | 보호 시간, 중복 방지, 러닝 윈도우 계산만 수행 |

- ViewModel은 **의도 객체를 받아 라우팅만** 담당.
- 참고: 현재 `AnalysisStateMachine`의 역할을 Filter + Scheduler로 분리하는 방향.

### 4.3 Latency / Visits 튜닝

| 컴포넌트 | 책임 |
|----------|------|
| **LatencyController** | prefMin / Target / Max 관리와 실측 대비 튜닝 |
| **VisitsBudgeter** | maxVisits 산정 로직만 독립 |

- 분석 예산 계산이 **상태 변화와 분리**되어 결정적 동작.

### 4.4 Snapshot Validator

| 컴포넌트 | 책임 |
|----------|------|
| **AnalysisSnapshotValidator** | 요청별 스냅샷 유효성 검사(세션, 보드 크기, requestId 일치)만 수행 |

- 결과 반영 조건을 **한 곳에 모아** 누락 방지.
- 현재 `SnapshotValidator` / `AnalysisResultGate` 등과 정렬.

---

## 5. 상태·동시성 계층 분해

### 5.1 RequestIdGenerator

- **세션/종류별 ID 생성 규칙**을 독립 모듈로 이동.
- ID 충돌/마스킹 오류 경로 제거.
- 참고: `EngineViewModel.kt` 내 `nextRequestId(session, kind)` (예: L231–L243 부근) 로직 이전.

### 5.2 SingleThreadExecutor → 단일 액터

- **EngineStateExecutor** (SingleThreadExecutor) 대신 **단일 액터(코루틴 채널)** 로 상태 전이 처리.
- 스레드 자원/라이프사이클 관리 단순화.
- 참고: `EngineViewModel.kt` L88–L91 부근.

---

## 6. JNI / 서비스 경계 추가 격리

### 6.1 Repository 호출 래퍼

| 컴포넌트 | 책임 |
|----------|------|
| **EngineCalls** | analyze / getPolicy / getOwnership 등 AIDL 호출을 **명령 객체**로 감싸, 재시도/타임아웃/불변 스냅샷 태깅을 일관 처리 |

- 참고: `EngineRepository.kt` 사용처를 이 계층으로 통과시키는 방향.

### 6.2 NativeBridge 안전 계층

| 컴포넌트 | 책임 |
|----------|------|
| **NativeSafetyGate** | activeCallCount, 배열 풀 관리와 검증을 래핑. **“사용 전 준비 / 사용 후 해제”** 를 강제 |

- 참고: `NativeBridge.kt` 의 배열 풀/호출 수 제어를 이 게이트로 감싸는 방향.

---

## 7. UI 경계

### 7.1 OverlayStateController

| 컴포넌트 | 책임 |
|----------|------|
| **OverlayStateController** | 팝업/오버레이 표시 상태를 **엔진 결과와 분리된 전용 리듀서**로 관리 → UI 레이스 제거 |

- 참고: `EngineLifecycleCoordinator.kt` L138–L162 부근 오버레이 관련 상태를 이 컨트롤러로 이전하는 방향.

---

## 8. 이점 요약

- **단일 책임 + 결정적·함수형에 가까운 구조** → 부분 업데이트로 인한 상태 불일치가 구조적으로 사라짐.
- **동시성**은 이벤트 큐(및 단일 액터) 하나로 수렴.
- **외부 경계(JNI/AIDL)** 는 명령 객체/게이트로 감싸 재진입·타임아웃·라이프사이클을 강제.
- **ViewModel**은 라우팅·조립만 담당하여 실수 가능성이 낮아짐.

---

## 9. 정리

- **더 분해할 수 있다.**  
  Policy/Ownership, Intent 필터/스케줄러, Latency/Visits, Snapshot Validator, RequestIdGenerator, UI Overlay, JNI/서비스 게이트를 각각 독립시키면 **EngineViewModel은 얇아지고**, 구조적으로 **오류 경로가 거의 사라진다.**
- 본 문서는 **구조 설계만 제시**하며, 코드 수정은 하지 않는다. 구현 시 위 순서와 책임 분리를 참고하여 단계별로 적용하면 “한 번에 분해”가 가능하고, 같은 유형의 문제 반복을 줄일 수 있다.
