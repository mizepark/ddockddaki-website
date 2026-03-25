# 오류 0 구조 개선안: 불가능한 상태 제거·단일 전이

현재 코드 구조를 기준으로 **오류가 구조적으로 일어나지 않도록** 하는 구체적·실행 가능한 방법과 적용 순서.

---

## 1. 현재 구조의 강점 (이미 좋은 기반)

| 요소 | 위치 | 역할 |
|------|------|------|
| 단일 이벤트 큐 + 상태머신 | `EngineOrchestrator` | AppStart/StartAnalysis/SwitchSession/ChangeConfig/AnalysisDone 직렬 처리, 4상태(Idle/Ready/Analyzing/Switching) |
| 분석 의도 단일 상태머신 | `AnalysisStateMachine` | 의도 수집·필터·실행 시점만 결정 후 Orchestrator에 StartAnalysis 전달 |
| 세션 상태 캡처·복원 | `EngineLifecycleCoordinator` / `EngineSessionOrchestrator` | 세션별 상태 캡처·persist·복원 단일 책임 |
| 엔진 단일 진입점 | `EngineViewModel.kt` L64–L86, `EngineFacade` | init/analyze/stop/restart/close는 퍼사드만 사용 |

---

## 2. “오류 0”에 근접하기 위한 구조적 개선

### 2.1 불가능한 상태를 타입으로 제거

- **UiState** 를 sealed로 분리해 Idle/Ready/Analyzing/Switching 상태별로 **허용 필드만** 존재하게 함.
- “런타임 분기”가 아니라 **컴파일 단계에서 불가능 상태 차단**.

### 2.2 단일 스레드 상태 변경 규칙

- 엔진·분석 관련 **모든 상태 변경**을 **하나의 전용 디스패처(단일 스레드)**에서만 수행.
- 레이스 자체를 구조적으로 차단.

### 2.3 스냅샷 원자화

- `analysisMovesSnapshot` / `analysisRequestIdSnapshot` / `analysisBoardSizeSnapshot` 등 **분산된 mutable 필드**를 **단일 불변 스냅샷 데이터 클래스**로 묶고 **한 번에 교체**.
- “부분 업데이트 상태” 제거.

### 2.4 중요 로직 순수 함수화

- UiState → UiState 변환 로직을 **순수 함수**로 분리.
- 부작용 없이 **예측 가능한 전이**만 허용.

### 2.5 단일 진입점 강제

- 엔진 컨트롤러 직접 호출 가능 지점 봉쇄.
- 퍼사드/오케스트레이터 경유만 허용 (이미 주석으로 의도 있음 → 컴파일 단계 강제로 강화).

---

## 3. 이 앱에 맞춘 구체적 적용 순서

### Step 1: 상태 머신 단일화 강화

- **목표**: EngineViewModel 내부 mutable 상태를 **EngineRuntimeState** 같은 **단일 불변 모델**로 묶고, 변경은 **reduce(state, event)** 형태로만 수행.
- **구체**:
  - 분산 스냅샷 필드 → **AnalysisSnapshot** 불변 데이터 클래스로 묶고, `MutableStateFlow<AnalysisSnapshot>` 로 **한 번에 교체**.
  - (선택) 엔진 런타임 관련 나머지 필드도 **EngineRuntimeState** 로 묶고, 업데이트는 `_runtimeState.update { ... }` 또는 `reduce(current, event)` 한 경로만 사용.

### Step 2: “불가능한 상태” 제거

- **목표**: UiState 를 **sealed** 구조로 전환.
- **구체**: Idle / Ready / Analyzing / Switching 별로 허용 필드만 가지는 하위 타입 정의. Analyzing 일 때만 존재해야 하는 필드(예: 분석 중인 requestId, 보드 스냅샷)를 타입으로 분리.

### Step 3: 동시성 경계 고정

- **목표**: 엔진 연산·세션 스위치·분석 트리거를 **한 디스패처**에서 직렬 처리하도록 규칙화.
- **구체**: EngineOrchestrator 루프가 이미 단일 채널에서 동작하므로, ViewModel 쪽 상태 갱신을 **같은 디스패처**로 모으거나, “엔진/분석 상태를 바꿀 수 있는 스레드”를 하나로 고정.

### Step 4: 스냅샷 원자화 (Step 1과 연계)

- **목표**: 분산된 스냅샷 필드들을 **하나의 불변 구조**로 묶어, 일관성 없는 중간 상태를 근본적으로 제거.
- **구체**: `AnalysisSnapshot(moves, movesHash, budgetMs, boardSize, requestId)` 도입 후, 분석 시작·세션 전환 시 **한 번에 교체**만 허용.

---

## 4. 왜 이 방식이 “오류 0”에 가장 가깝나

- **잘못된 상태**는 “표현 불가능”해짐 (sealed + 상태별 필드).
- **동시성 오류**는 “일어나지 않도록 구조화” (단일 디스패처·단일 이벤트 루프).
- **흐름**은 “단일 이벤트 → 단일 전이”만 허용 (reduce / 단일 StateFlow 업데이트).

---

## 5. 구현 상태

| Step | 내용 | 상태 |
|------|------|------|
| Step 1 | EngineRuntimeState·AnalysisSnapshot 단일 불변 모델, 스냅샷 원자화 | **완료** |
| Step 2 | UiState sealed 전환 (상태별 허용 필드) | 미착수 |
| Step 3 | 동시성 경계 고정 (단일 디스패처 규칙화) | 미착수 |
| Step 4 | 스냅샷 원자화 (Step 1과 동시 적용) | **완료** (Step 1과 함께 적용) |

### Step 1·4 적용 내용

- **`EngineRuntimeState.kt`** 추가: **`AnalysisSnapshot`** 불변 데이터 클래스 (moves, movesHash, budgetMs, boardSize, requestId). `EMPTY` 상수로 초기/리셋 표현.
- **EngineViewModel**: `analysisMovesSnapshot` / `analysisMovesHashSnapshot` / `analysisBudgetMsSnapshot` / `analysisBoardSizeSnapshot` / `analysisRequestIdSnapshot` 분산 변수 제거 → **`_analysisSnapshot: MutableStateFlow<AnalysisSnapshot>`** 단일 보관, **한 번에 교체**만 허용.
  - 분석 시작 시: `_analysisSnapshot.value = AnalysisSnapshot(...)` 한 번만 설정.
  - 세션 전환 시: `_analysisSnapshot.value = AnalysisSnapshot.EMPTY`.
- 읽기는 모두 `analysisSnapshot.moves` 등으로 통일. **부분 업데이트 없음** → 중간 상태 불일치 제거.
