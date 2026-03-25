# 앱 동작 위험 요소 분석 및 시뮬레이션 한계

설치 전에 “제대로 작동 안 할 수 있는 부분”을 정리하고, **코드 시뮬레이션으로 왜 잡기 어려웠는지**를 설명합니다.

---

## 1. 이미 수정한 위험 요소 (참고)

| 위험 | 원인 | 수정 |
|------|------|------|
| 분석이 한 번도 안 돌아감 | `restoreSavedGame()`에서 저장된 게임 없을 때 `return`만 하고 `updateAnalysisIntent` 미호출 | `snapshot == null`이어도 `updateAnalysisIntent(RESTORE_SAVED_GAME)` 호출 |
| 엔진 준비 직후 분석 누락 | 초기화 성공 후 분석 트리거가 `restoreSavedGame` 경로에만 의존 | `applyInitSuccessUi()` 마지막에 `updateAnalysisIntent(INIT_ENGINE)` 추가 |
| "해당 지점에는 둘 수 없습니다" | 추천 나오기 전 빈 점 탭 → `playMove` → 불합법 처리 | `onCellClicked`에서 추천 없을 때 탭 무시 |

---

## 2. 아직 남아 있을 수 있는 위험 요소

### 2.1 분석이 트리거되지 않는 조건 (조용한 실패)

- **`updateAnalysisIntent()` 진입 전 조기 return**
  - `!snapshot.isModelReady` → **분석 의도 자체를 제출하지 않음.**  
    (세션 전환 직후 UI 상태에 아직 `isModelReady == false`가 반영된 경우 등)
  - `analyzingSession == Session.OFFLINE` → 오프라인에서는 의도 제출 안 함 (의도된 동작).
  - `retryToken == 0 && tentativeIntent.samePosition(lastEmittedIntent)` → **동일 국면이면 의도 갱신 스킵.**  
    빠르게 같은 intent가 두 번 들어오면 두 번째는 submit 자체를 안 함.

- **오케스트레이터에서 StartAnalysis 스킵**
  - `processEvent(StartAnalysis)` 시 `s != Ready`이면 **그냥 return.**  
    Idle(초기화 전), Switching(세션 전환 중), Analyzing(이미 분석 중)일 때 새 분석은 시작되지 않음.
  - 따라서 “AppStart가 아직 안 끝났는데 사용자가 분석 탭에서 뭔가 했다” 같은 **이벤트 순서**에 민감.

- **스케줄러/필터에서 실행 스킵**
  - `AnalysisScheduler.decide()`: 보호 구간(`minAnalysisProtectMs`) 안이면 `Defer` → 나중에 한 번만 재시도.
  - `AnalysisIntentFilter.shouldAccept()`: `samePosition(lastRunIntent) && (isAnalysisRunning || pendingAnalysis)`이면 **수락 안 함.**  
    “같은 국면인데 이미 분석 중/대기 중”이면 새 실행이 영구히 스킵될 수 있음 (다음 사용자 이벤트에서 새 intent가 올 때까지).

→ **공통점:** “분석이 한 번도 시작되지 않는다”는 **UI/라이프사이클/타이밍**이 개입하고, **실제 엔진/디스크/SharedPreferences**를 쓰지 않는 단위 시뮬레이션에서는 이 경로를 재현하기 어렵다.

### 2.2 분석 결과가 반영되지 않는 조건

- **`handleAnalysisResult` 호출 자체가 안 됨**
  - `runAnalysisInternal` 안에서 `if (isActive && reqId == lastAnalysisRequestId)` 일 때만 `handleAnalysisResult` 호출.
  - `!isActive`(예: scope 취소) 또는 `reqId != lastAnalysisRequestId`(새 분석이 이미 시작됨)면 **결과를 버림.**

- **`AnalysisResultGate.shouldProcessResult`에서 거절**
  - `resultAnalyzedRequestId != lastAnalysisRequestId`, `currentBoardSize != analysisBoardSizeSnapshot`, `runEpoch < lastSubmittedEpoch` 등이면 **결과 적용 안 함.**  
    시뮬레이션은 이 validator 계약만 검증하지, “실제로 결과가 UI에 반영되는지”는 검증하지 않음.

- **수순 불일치로 인한 무시**
  - `result.analyzedMovesHash != SnapshotValidator.computeMovesHash(current)` 이면 **topSuggestions 비우고 return.**  
    분석 중에 사용자가 수를 두거나 점프하면 흔히 발생.

→ **시뮬레이션:** epoch/requestId/boardSize 불일치 시 “수락 안 함”만 검증.  
  **안 하는 것:** `_uiState.update`까지 이어지는 전체 플로우, 그리고 “언제 불일치가 발생하는지”에 대한 앱 시나리오.

### 2.3 초기화/진입 경로

- **엔진 초기화가 아예 안 시작**
  - `init()` 안에서 `viewModelScope.launch(Dispatchers.IO) { if (settingsRepository.getUserLocale() != null) startInitializationSequence() }`.  
    **로케일이 null이면 AppStart를 보내지 않음** → 오케스트레이터는 계속 Idle → 어떤 분석도 시작 안 됨.  
    (사용자가 로케일 선택하면 `startEngineAfterLocaleSelected()`로 나중에 시작 가능.)

- **AppStart 실패**
  - `EngineInitResult.Failure`면 `_state`는 Idle 유지 → 이후 모든 `StartAnalysis`는 오케스트레이터에서 스킵.  
    “모델 준비 실패” 등은 시뮬레이션에서 네이티브/에셋 없이 재현 불가.

- **세션 전환 시점**
  - 사용자가 **분석 탭을 먼저 열고**, 그 시점에 아직 `isModelReady == false`인 상태가 `newUiState`에 들어가면,  
    `onSessionSwitched`에서 호출하는 `updateAnalysisIntent(SET_ACTIVE_SESSION)`이 **첫 줄에서 return**할 수 있음.  
    (이후 `applyInitSuccessUi` → `restoreSavedGame` / `INIT_ENGINE`으로 보완하는 수정이 있음.)

→ **시뮬레이션:** `EngineViewModel` 전체를 띄우지 않고, **순수 함수/Validator/UseCase** 위주로만 돌리므로 “언제 init이 되고, 언제 isModelReady가 true가 되는지”를 다루지 않음.

### 2.4 기타

- **ensureMoveIsLegal 실패 시 “해당 지점에는 둘 수 없습니다”**
  - 분석 모드에서도 빈 점 탭 시 `playMove` → `ensureMoveIsLegal` 호출.  
    엔진이 불합법으로 판단하면(자살, 코, 또는 엔진/보드 상태 불일치) 해당 메시지 표시.  
    “추천 없을 때 탭 무시” 수정으로 대부분 완화되나, **추천이 나온 뒤 잘못된 점을 눌렀을 때**는 그대로 노출됨.

- **Defer 후 실행 조건**
  - `scheduleDeferred`에서 `remainingMs` 후에 `latest == null || !latest.samePosition(intent)` 또는 `intent.samePosition(lastRunIntent)`면 **실행하지 않음.**  
    의도가 바뀌었거나 이미 같은 국면이 실행된 경우 의도적으로 스킵하는 동작이지만, 타이밍에 따라 “분석이 한동안 안 돌아가는” 느낌을 줄 수 있음.

---

## 3. 코드 시뮬레이션이 못 잡는 이유

현재 시뮬레이션 테스트들의 성격:

| 테스트 | 하는 일 | 하지 않는 일 |
|--------|---------|----------------|
| **EngineEventSequenceSimulationTest** | `AnalysisSnapshotValidator.shouldProcessAnalysisResult` 등 **validator 계약** (epoch/requestId/boardSize) 검증, 직렬·병렬·교차 시나리오 | `EngineViewModel`/오케스트레이터/실제 `_uiState` 갱신, `isModelReady`/`loadGameState`/init 순서 |
| **AppWideSimulationTest** | SimpleGoBoard, SnapshotValidator, GameUseCase, Sgf, LocaleHelper, ScoreUtils, 토큰 수학 등 **순수 도메인/유틸** | ViewModel·오케스트레이터·세션 전환·분석 트리거 경로 |
| **PlayModeSimulationTest 등** | 플레이/점프/실패 시나리오 등 **도메인 시퀀스** | 실제 Android Context, 엔진 서비스, 디스크, `isModelReady`가 false인 상태에서의 진입 경로 |

정리하면:

1. **전제 조건(init, isModelReady, 저장된 게임 유무)을 다루지 않음**  
   시뮬레이션은 “엔진이 이미 준비되어 있고, UI 상태가 이미 어떤 값이다”라고 가정하지 않고, **그런 가정 자체가 있는 코드 경로**를 타지 않음.  
   따라서 “저장된 게임 없을 때 분석 트리거를 안 보냄” 같은 **초기 진입 경로 버그**는 재현되지 않음.

2. **이벤트 순서와 라이프사이클을 재현하지 않음**  
   “AppStart 완료 전에 SwitchSession이 먼저 처리된다”, “세션 전환 직후에는 아직 isModelReady가 false다” 같은 **실제 앱 이벤트 순서**를 시뮬레이션하지 않음.  
   오케스트레이터의 `when (event)`와 ViewModel의 `updateAnalysisIntent` **진입 조건**은 단위 테스트에서 호출되지 않음.

3. **조용한 실패(no-op)를 assertion으로 보지 않음**  
   “분석이 시작되지 않았다”, “결과가 반영되지 않았다”를 검증하려면 **ViewModel/오케스트레이터를 띄우고**, `updateAnalysisIntent` 호출 후 `_state`/`_uiState`를 확인해야 함.  
   현재 시뮬레이션은 **순수 함수/Validator**만 보므로 “의도가 submit되지 않음”이나 “StartAnalysis가 스킵됨”을 잡지 않음.

4. **외부 의존성(엔진, 디스크, 설정)을 모킹하지 않음**  
   `loadGameState()`가 null을 반환하는 경로, `getUserLocale()`이 null인 경로, `AssetInstaller.isModelReady()`가 false인 경로는 **Context/디스크 없이** 돌리는 테스트에서 재현되지 않음.

---

## 4. 설치 전 체크리스트 (권장)

- [ ] **로케일 미설정 시:** 앱 실행 → 로케일 선택 전에는 분석/대국 탭에서 “모델 준비 중” 등만 보이고, 분석이 시작되지 않는지.
- [ ] **로케일 설정 후 첫 실행:** 저장된 게임 없이 분석 탭 진입 → 빈 바둑에서 일정 시간 내에 추천수가 나오는지.
- [ ] **분석 탭 첫 진입이 엔진 초기화보다 빠른 경우:** 앱 실행 후 바로 분석 탭으로 이동 → 초기화 완료 후 추천수/참고도가 나오는지.
- [ ] **대국 탭:** 힌트 버튼 클릭 시 힌트 분석이 진행되는지.
- [ ] **참고도:** 분석 모드에서 참고도 분석 시작 후, 추천수 클릭 시 참고도가 열리는지.
- [ ] **빈 점 탭:** 추천이 나오기 전 빈 점 탭 시 “해당 지점에는 둘 수 없습니다”가 안 뜨는지.

---

## 5. 시뮬레이션으로 보강하고 싶을 때 (선택)

- **통합/시나리오 테스트:**  
  `EngineViewModel`(또는 테스트용 하네스) + 모킹된 `EngineFacade`/`SettingsRepository`/`GameStateUseCase`를 두고,  
  “저장된 게임 없음” → `restoreSavedGame()` → `updateAnalysisIntent` 호출 여부 및 `analysisStateMachine`에 intent가 들어갔는지 assertion.
- **오케스트레이터 상태 전이:**  
  Idle → AppStart → Ready, Ready → StartAnalysis → Analyzing → AnalysisDone → Ready 등 **상태만** 검증하는 작은 테스트.
- **updateAnalysisIntent 조기 return 시나리오:**  
  `isModelReady == false` 또는 `samePosition(lastEmittedIntent)`일 때 `submitIntent`가 호출되지 않음을 모킹으로 확인.

이렇게 하면 “분석이 왜 안 돌아갔는지”를 코드 레벨에서 더 잘 좁혀나갈 수 있습니다.
