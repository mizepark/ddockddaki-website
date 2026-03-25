# 구조적 목표 7가지 — 컴파일/런타임 봉쇄

**목적**: 엔진·분석·UI 경로를 구조적으로 봉쇄하여 오류 경로를 제거한다.  
**원칙**: 단일 진입점, 순수 전이, 단일 큐, 스냅샷 게이트, 네이티브 규칙, 도메인 타입, 프로세스 역할.

---

## 1. 엔진 호출 경로 컴파일 타임 봉쇄

| 항목 | 내용 |
|------|------|
| **목표** | EngineController / EngineRepository 직접 접근을 구조적으로 불가능하게 만들기 |
| **방법** | EngineFacade만 외부에 노출하고, Controller/Repository는 생성부(EngineBootstrap)에서만 생성·주입. App에서는 `engineFacade`만 노출. |
| **근거** | 단일 진입점 설계가 이미 있음. 강화: App에서 Controller/Repository 제거, EngineBootstrap에서만 생성. |
| **참조** | EngineFacade, EngineController, EngineBootstrap |

---

## 2. 상태 전이 리듀서 단일화

| 항목 | 내용 |
|------|------|
| **목표** | “상태 변경은 무조건 순수 함수로만” |
| **방법** | `(State, Event) → (State, Effect)` 구조로 고정. Effect 실행은 Orchestrator에서만. |
| **근거** | EngineStateActor / EngineOrchestrator가 이미 분리돼 있어 이 방향으로 수렴이 자연스러움. |
| **참조** | EngineStateActor, EngineOrchestrator, UiStateReducer, OverlayStateController |

---

## 3. 분석/세션/엔진 이벤트의 단일 큐 고정

| 항목 | 내용 |
|------|------|
| **목표** | 동시성 경로 자체를 없앰 |
| **방법** | 모든 경로가 `EngineOrchestrator.post`로만 들어오게 구조화. |
| **근거** | 단일 이벤트 큐 구조가 이미 있음. |
| **참조** | EngineOrchestrator |

---

## 4. 스냅샷 불일치 결과의 전면 폐기

| 항목 | 내용 |
|------|------|
| **목표** | 잘못된 결과가 상태를 오염시키지 않도록 원천 차단 |
| **방법** | requestId / movesHash / boardSize 불일치 시 “반영 자체를 금지”하는 정책을 모든 결과 경로에 적용. |
| **근거** | AnalysisSnapshotValidator 게이트가 이미 존재. 모든 결과 경로가 이를 통과하는지 검사·유지. |
| **참조** | AnalysisSnapshotValidator, AnalysisResultGate |

---

## 5. 네이티브 경계 단일 스레드 + 안전 종료 규칙 고정

| 항목 | 내용 |
|------|------|
| **목표** | JNI 호출 레이스 / 중단 시그널 충돌 자체 제거 |
| **방법** | Facade 단일 Mutex와 NativeBridge의 activeCallCount 정책을 “모든 경로”에 강제. |
| **근거** | KataGoEngineFacade Mutex, NativeBridge activeCallCount 기반 설계가 이미 있음. |
| **참조** | KataGoEngineFacade, NativeBridge, NativeSafetyGate, EngineCalls |

---

## 6. 도메인 타입 강화 (불가능한 입력 제거)

| 항목 | 내용 |
|------|------|
| **목표** | 잘못된 좌표/보드 사이즈/상태 조합을 타입으로 차단 |
| **방법** | BoardSize, Move, Session, EnginePhase를 값 객체/제한 타입으로 만들고, 생성 시 불변성 보장. |
| **근거** | 일부 검증이 NativeBridge에 있으나, 상위 레이어에서 타입으로 봉쇄하면 더 구조적. EngineDomainTypes에 BoardSize 등 이미 존재. |
| **참조** | EngineDomainTypes, EngineTypes, NativeBridge |

---

## 7. 프로세스 역할 고정 (엔진/UI 분리)

| 항목 | 내용 |
|------|------|
| **목표** | UI 초기화가 엔진 프로세스에 들어가는 실수를 구조적으로 제거 |
| **방법** | 엔진 프로세스에서는 UI 관련 초기화 경로 자체를 제거. 메인 프로세스만 UI/광고/웹뷰 초기화. |
| **근거** | KataGoApp에서 `isEngineProcess()` 체크로 이미 분리. 엔진 프로세스는 early return. |
| **참조** | KataGoApp |

---

## 구현 체크리스트

| # | 목표 | 상태 | 비고 |
|---|------|------|------|
| 1 | 엔진 경로 봉쇄 | 완료 | EngineBootstrap 도입, App에서 Controller/Repository 제거, Facade만 노출 |
| 2 | 리듀서 단일화 | 유지 | (State, Event)→(State, Effect) 문서화·점검 |
| 3 | 단일 큐 | 유지 | 모든 이벤트가 post 경유 여부 점검 |
| 4 | 스냅샷 폐기 | 유지 | 모든 결과 경로가 Validator 경유 |
| 5 | 네이티브 규칙 | 유지 | Mutex + activeCallCount 일관 적용 |
| 6 | 도메인 타입 | 확장 | BoardSize/Move/Session/EnginePhase 값 객체 확대 |
| 7 | 프로세스 역할 | 유지 | 엔진 프로세스에서 UI 초기화 없음 |
