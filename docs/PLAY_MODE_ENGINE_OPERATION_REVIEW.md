# 대국모드(Play Mode) 엔진 작동 구조 검토

대국모드에서의 **엔진 초기화**, **시작/정지/재개**, **기력(레벨) 변경 시 엔진 교체** 등 엔진 관련 함수·로직의 구성을 정리하고 효율성을 검토한 문서입니다.

---

## 1. 전체 아키텍처 요약

### 1.1 엔진 인스턴스 분리

| 구분 | 서비스 | Repository | Controller | 설정/모델 |
|------|--------|------------|------------|-----------|
| **분석(ANALYSIS)** | `KataGoService` | `analysisEngineRepository` | `analysisEngineController` | analysis.cfg, 단일 모델 |
| **대국(PLAY)** | `KataGoPlayService` | `playEngineRepository` | `playEngineController` | play.cfg, 레벨별 모델(B10·B6) |

- 앱(`KataGoApp`)에서 **분석용·대국용 엔진이 서로 다른 서비스로 분리**되어 있어, 메모리/프로세스 격리와 대국 전용 설정(play.cfg) 적용이 명확함.
- 실제로 **한 시점에는 한 세션만 활성**(`analyzingSession`)이며, 해당 세션의 `engineController`만 사용됨.

### 1.2 호출 흐름 (계층)

```
UI (PlayViewModel / AnalysisUiViewModel 등)
  → EngineViewModel (세션 전환, 초기화, 분석 트리거)
    → EngineController (큐 + 우선순위)
      → EngineRepository (서비스 바인딩, init/shutdown)
        → KataGoService / KataGoPlayService (AIDL)
          → NativeBridge → KataGo 네이티브
```

---

## 2. 엔진 초기화

### 2.1 앱 기동 시 (공통)

- **진입점**: `EngineViewModel.init` → `startEngineAfterLocaleSelected()` 또는 `startInitializationSequence()`.
- **순서**:
  1. `startInitializationSequence()`: 로케일 있으면 실행, `isInitialized`로 중복 방지.
  2. **모델 준비**: `AssetInstaller.prepareModel()` (에셋에서 복사 등).
  3. **엔진 초기화는 하지 않음** — `onPreparationCompleted()` → `restoreSavedGame()` 등만 수행.
- **실제 엔진 init 시점**: 사용자가 **해당 세션(분석/대국)으로 처음 들어갈 때** `setActiveSession()` 내부에서 `initializeEngine()` 호출.

즉, **지연 초기화(Lazy Init)** 구조라, 대국 탭을 열기 전까지 PLAY 엔진은 생성되지 않아 **앱 기동 비용이 적음**.

### 2.2 대국모드 최초 진입 / 세션 전환

- **진입**: `PlayViewModel.setActive(true)` → `setActiveSession(EngineViewModel.Session.PLAY)`.
- **로직**: `EngineViewModel.setActiveSession(session)` (라인 635~714).

동작 요약:

1. **이전 세션 상태 저장**  
   `persistGameState()` 후, 이전 세션(ANALYSIS/PLAY/OFFLINE)별로 `analysisSessionState` / `playSessionState` / `offlineSessionState`에 현재 UI 상태 캡처.
2. **분석 중단**  
   `engineOpMutex` + `stopAllAnalysis(expectedAnalyzeRequestId)`로 **두 컨트롤러 모두** 분석 중단.
3. **세션 전환**  
   `analyzingSession = session`, 요청 ID/스냅샷 초기화.
4. **UI 복원**  
   전환 대상 세션의 캐시된 상태 또는 디스크에서 복원 후 `_uiState` 반영.
5. **프로필 비교**  
   `desiredEngineProfile(session, boardSize, forceLowMem)` 계산.  
   PLAY일 때는 `playLevelIndex`가 포함되므로 **기력(레벨)이 다르면** `desired != currentEngineProfile`.
6. **재초기화 조건**  
   `currentEngineProfile != desired`일 때만:
   - `AssetInstaller.ensureConfigFile(..., writePlayConfig = (session == Session.PLAY))`로 **play.cfg 갱신**.
   - **이전 세션 엔진만** `controllerFor(previousSession).shutdown()`.
   - **350ms 대기** 후 `initializeEngine(forceLowMem, triggerInitialAnalysis = false)`로 **현재 세션 엔진** 초기화.
7. **분석 트리거**  
   `persistGameState()` 후 `triggerAnalysis(debounceMs = 0L)`.

효율성:

- **같은 프로필(같은 세션·레벨·보드 크기 등)이면** 엔진을 다시 초기화하지 않고, 분석만 다시 걸어줌 → **불필요한 shutdown/init 제거**.
- **PLAY 전용 설정**은 `session == Session.PLAY`일 때만 `writePlayConfig = true`로 갱신해, 분석용 설정과 혼선이 없음.

### 2.3 대국 시작 시 (기력 반영)

- **진입**: `PlayViewModel.startGame()` (라인 556~711).
- **순서**:
  1. 선택 기력에 따라 **레벨별 파라미터** 계산 (chosenMoveTemperature, wideRootNoise, cpuct 등).
  2. `SettingsRepository`에 **maxTime, chosenMoveTemperature, avoidRepeatedPattern** 등 저장.
  3. **엔진 준비**: `engineViewModel.ensurePlayEngineReadyForLevel(selectedLevelIndex)`.
     - `setPlayLevelForModelSelection(levelIndex)`로 현재 선택 기력을 저장.
     - `setActiveSession(Session.PLAY)` 호출 → 위 2.2 절과 동일하게, **현재 `playLevelIndex`가 반영된 desired**와 비교되어, 레벨이 바뀌었으면 이전 PLAY 엔진 shutdown 후 새 레벨로 init.
     - `_uiState.value.isEngineReady`가 될 때까지 **최대 125초** `StateFlow`로 대기 (`withTimeoutOrNull(125_000L)`).
  4. 주석대로 **play.cfg는 `setActiveSession(PLAY)` 안에서 이미 갱신**되므로, 여기서 중복 `ensureConfigFile` 호출을 하지 않음 (라인 690).

효율성:

- **기력 변경 시 엔진 교체**는 `EngineProfile.playLevelIndex` 비교로 한 곳(`setActiveSession`)에서만 처리되어 일관적임.
- **모델 선택**: `AssetInstaller.getModelFileForPlayLevel(context, levelIndex)`  
  - 0~6: B10 (b10.bin)  
  - 7~30: B6 (b6.txt)  
  레벨만으로 파일이 결정되므로 분기 단순함.

### 2.4 initializeEngine() 내부 (PLAY 시)

- **위치**: `EngineViewModel.initializeEngine()` (라인 2700~2945).
- PLAY일 때:
  - **모델**: `AssetInstaller.getModelFileForPlayLevel(app, settingsRepository.getPlayLevelIndex())`.
  - **설정 파일**: `Session.PLAY` → `playConfigFile` (play.cfg).
  - **Human 모델**: `wantsHuman == (Session.PLAY && humanProp > 0 && humanFile 존재)`일 때만 human 모델 경로 전달.
- **공통**:
  - CPU/저메모/램 구간별 batch·cache 등 튜닝.
  - `engineController.initEngine(context, modelPath, configPath, humanModelPath, boardSize)` 호출.
  - **120초 타임아웃**으로 감싼 뒤, **헬스체크**용 `analyzeTop4(emptyMoves, 0.5s, 2 visits)` 실행해 Zombie 상태 방지.
  - 성공 시 `currentEngineProfile`에 `playLevelIndex` 저장, 실패 시 크래시 카운트/CPU 폴백 등 복구 로직 존재.

효율성:

- PLAY/ANALYSIS 분기가 `analyzingSession` 한 곳으로 정리되어 있음.
- **한 번에 한 세션만** init하므로, “현재 활성 세션 엔진”만 초기화하는 구조가 유지됨.

---

## 3. 엔진 정지 / 재개

### 3.1 분석 중단 (stop)

- **EngineViewModel**: `stopAllAnalysis(expectedAnalyzeRequestId)`  
  - 분석·대국 **두 컨트롤러 모두** `stopAnalysis(expectedAnalyzeRequestId)` 호출.
- **EngineController**: `stopAnalysis()`는 **ActionMsg(priority = STOP)** 로 전달.
  - **STOP**은 “현재 실행 중인 작업이 있어도 대기하지 않고 바로 process”되므로, 분석이 진행 중이어도 **우선적으로** 중단 요청이 처리됨.
  - 내부적으로 `engineRepository.stopAnalysis()` 호출 → 네이티브 측 분석 중단.

효율성:

- **우선순위 큐**로 인해 “분석 중”인 상태에서도 사용자 전환/새 분석 요청 시 빠르게 멈출 수 있음.

### 3.2 엔진 종료 (shutdown)

- **호출처**:
  - **세션 전환 시**: 이전 세션만 `controllerFor(previousSession).shutdown()` (setActiveSession 내부).
  - **메모리 압박**: `KataGoApp` ComponentCallbacks2에서 `TRIM_MEMORY_*` 시 두 컨트롤러 모두 `stopAnalysis()` (shutdown은 아님).
  - **재초기화/폴백**: CPU 강제 전환·VRAM OOM 재시도 등에서 `engineController.shutdown()` 후 `initializeEngine()` 재호출.
- **EngineController**: `shutdown()` → `action { engineRepository.shutdown() }` → 서비스 unbind 및 네이티브 `NativeBridge.shutdown()`.

효율성:

- **세션 전환 시에는 “이전 세션 엔진만”** shutdown하므로, 리소스 해제가 명확하고, 350ms 후 새 세션 엔진만 init하는 흐름이 일관됨.

### 3.3 재개 (다시 분석 시작)

- **재개**는 “새 분석 요청”으로 이뤄짐: `setActiveSession()` 마지막의 `triggerAnalysis(debounceMs = 0L)` 또는 `newGame()` 후 `triggerAnalysis()`.
- 엔진을 **다시 켜는 전용 API**는 없고, “세션 전환 + 필요 시 init + triggerAnalysis”가 재개 역할을 함.

---

## 4. 대국모드에서의 기력(레벨) 변경 시 엔진 교체

### 4.1 프로필 비교

- `EngineProfile`에 **playLevelIndex**가 포함됨 (라인 76~84).
- `desiredEngineProfile(session, ...)`에서 PLAY일 때 `settingsRepository.getPlayLevelIndex()`를 넣음.
- 따라서:
  - **같은 PLAY 세션이라도 레벨이 다르면** `currentEngineProfile != desired` → **엔진 재초기화** 발생.

### 4.2 언제 반영되나

- **대국 탭 진입 시**: `setActive(active = true)` → `setPlayLevelForModelSelection(_settingsState.value.selectedLevelIndex)` 후 `setActiveSession(PLAY)`.  
  → 이때 desired에 **현재 선택된 레벨**이 들어가므로, 이전에 다른 레벨로 대국했었다면 엔진이 새 레벨로 교체됨.
- **대국 시작 버튼**: `startGame()` → `ensurePlayEngineReadyForLevel(selectedLevelIndex)` → `setPlayLevelForModelSelection(levelIndex)` → `setActiveSession(PLAY)`.  
  → **startGame() 시점의 선택 기력**으로 play.cfg·모델이 결정되고, 필요 시 엔진이 그에 맞게 교체됨.

즉, **기력 변경 후 엔진이 바뀌는 시점**은  
“(대국 탭 활성화 시) 또는 (대국 시작 시)”이며, **대국 중에 설정에서만 레벨을 바꾼 경우**에는 **다음 대국 시작**에서 반영됨.  
(한 판 도중에 엔진을 바꾸지 않아, 일관된 기력으로 한 게임이 진행됨.)

### 4.3 모델 파일 선택

- `AssetInstaller.getModelFileForPlayLevel(context, levelIndex)`:
  - **0~6**: B10 (b10.bin).
  - **7~30**: B6 (b6.txt).

레벨만으로 모델이 정해지므로, **엔진 교체 시 사용할 모델**이 항상 명확함.

---

## 5. restartEngineForNewConfig()

- **위치**: EngineViewModel 라인 2667~2696.
- **동작**:
  - `engineOpJob` 취소 후, `engineOpMutex` 안에서:
    - UI에 “준비 중” 메시지.
    - `analysisJob` 취소, `stopAnalysis(analysisRequestIdSnapshot)`로 **현재 세션 엔진**만 분석 중단.
    - `initializeEngine(forceLowMem = false)` 호출 (현재 `analyzingSession` 기준).
- **의미**: 설정 변경(예: CPU 모드, 스레드 수 등) 후 **현재 활성 세션의 엔진만** 재시작할 때 사용.
- **호출처**: 코드베이스 내에서 이 함수를 호출하는 UI/다른 클래스는 검색 시 발견되지 않음.  
  → 설정 화면 등에서 “엔진 재시작” 버튼이 있다면 해당 버튼에서 호출하도록 연결해 두는 것이 좋음.

---

## 6. 세션 전환 시 shutdown–init 사이 delay(350ms)의 의미와 연관성

### 6.1 350ms가 의미하는 것

세션 전환(분석 ↔ 대국) 시 **프로필이 바뀌면** 다음 순서로 동작한다.

1. **이전 세션 엔진 `shutdown()`**  
   - `EngineController.shutdown()` → `EngineRepository.shutdown()`  
   - AIDL로 서비스의 `shutdown()` 호출 → 서비스 내부에서 `NativeBridge.shutdown()` (진행 중인 네이티브 호출이 끝날 때까지 최대 30초 대기 후 정리)  
   - 그 다음 `unbindService(serviceConnection)` + `stopService(Intent)` 호출  
   - **`shutdown()`이 return하는 시점** = 네이티브 정리 완료 + unbind/stop 호출까지 완료한 시점

2. **여기서 350ms 대기** (과거 고정값, 현재는 기기별 안전 범위 내 값 사용)

3. **새 세션 엔진 `initializeEngine()`**  
   - 다른 서비스(KataGoService ↔ KataGoPlayService)에 `bindService` 후 `initEngine()` 호출

**350ms(또는 기기별 delay)가 필요한 이유**

- `unbindService()` / `stopService()`는 **호출만 하고 return**하며, 실제 프로세스 정리(onDestroy, 리소스 해제)는 **시스템이 비동기로** 수행한다.
- 이전 엔진 프로세스가 **완전히 정리되기 전에** 새 엔진을 bind/init하면:
  - 두 프로세스가 잠깐 동시에 메모리를 쓰거나
  - 새 서비스 바인딩/init 시 리소스 부족으로 실패·크래시가 날 수 있다.
- 따라서 **“shutdown() 반환 후, unbind/process teardown이 끝날 시간”**을 주기 위한 **고의적인 대기**가 필요하고, 그 값이 350ms(및 기기별 조정)이다.

### 6.2 코드 내 다른 delay와의 연관성 (불안정 방지)

| 위치 | delay | 용도 | 조정 시 유의점 |
|------|-------|------|----------------|
| **세션 전환** (setActiveSession) | **350ms** (기기별 200~600ms) | 이전 엔진 shutdown 반환 후, 새 엔진 init 전 대기 | 너무 짧으면 저사양에서 init 실패/불안정; 상한으로 체감 지연만 증가 |
| CPU 강제 전환 후 | 500ms | shutdown 후 init 전 | 동일 서비스 재시작이 아니라 모드 전환이므로 500ms 유지 권장 |
| VRAM OOM → Safe Mode 재시도 전 | 500ms | shutdown 후 init 전 | 리소스 해제 대기, 500ms 유지 권장 |
| CPU 폴백 → 재시도 전 | 1000ms | shutdown 후 init 전 | 프로세스/라이브러리 완전 정리 목적, 1000ms 유지 권장 |
| 크래시 루프 → CPU 강제 전환 후 | 1000ms | shutdown 후 init 전 | 위와 동일 |

- **세션 전환 delay만** 기기별로 200~600ms 범위에서 조정하고, **나머지 500/1000ms는** “더 심한 재시작”용으로 그대로 두는 것이 안정성에 유리하다.
- 세션 전환 delay를 **한 곳 상수/함수**로 두고, 다른 delay와 섞어 쓰지 않도록 하면 조정 시 연관성 파악이 쉽다.

### 6.3 정리

- **350ms** = “이전 엔진 shutdown 반환 후, **unbind/프로세스 teardown이 끝날 시간**”을 주기 위한 **의도된 대기**.
- 기기별로 **200~600ms** 안에서만 조정하고, 그 외 delay(500/1000)는 유지하면 **조정이 불안정으로 이어지지 않도록** 연관성을 지킬 수 있다.

---

## 7. EngineController 큐 구조

- **Channel + backlog**로 메시지 처리.
- **메시지 종류**:
  - **CallMsg**: 일반 호출 (init, getPolicyHeatmap 등) — 순서대로 처리.
  - **AnalyzeMsg**: 분석 요청 — `runningAnalyzeRequestId` 관리, 완료 시 null로 복원.
  - **ActionMsg**:  
    - **STOP**: `stopAnalysis` — **실행 중인 작업이 있어도** 큐에 넣지 않고 바로 `process()` 호출.  
    - **NORMAL**: shutdown 등 — 실행 중이면 backlog에 추가.

효율성:

- **분석 중단**이 우선 처리되어, 사용자 경험과 리소스 해제가 좋음.
- init / analyze / shutdown이 **한 스레드(scope)** 에서 순차 처리되므로, 동시성 버그 가능성이 줄어듦.

---

## 8. 종합 평가 및 개선 제안

### 8.1 잘 구성된 점

1. **세션별 엔진 분리**  
   분석(ANALYSIS)과 대국(PLAY)이 서비스·Repository·Controller까지 분리되어, 설정(analysis.cfg / play.cfg)과 모델 선택이 혼재하지 않음.
2. **프로필 기반 재초기화**  
   `EngineProfile`(configKind, playLevelIndex, boardSize 등) 비교로 “필요할 때만” shutdown/init을 수행해, 불필요한 재초기화가 적음.
3. **지연 초기화**  
   앱 기동 시에는 모델 준비만 하고, 해당 세션 진입 시에만 엔진을 init해 기동 비용이 적음.
4. **정지 우선순위**  
   STOP 우선순위로 분석 중단이 즉시 반영되어, 전환·취소 시 반응이 좋음.
5. **대국 전용 설정·모델**  
   play.cfg와 `getModelFileForPlayLevel`로 레벨별 모델/파라미터가 명확히 구분됨.
6. **복구 로직**  
   크래시 카운트, 헬스체크, CPU/저메모 폴백, 타임아웃 등으로 엔진 비정상 종료에 대한 대응이 있음.

### 8.2 개선 여지

1. **setActiveSession 내 shutdown–init delay**  
   이전 엔진 shutdown 후 350ms 고정 대기. 기기별로 프로세스 정리 시간이 다를 수 있어, 필요 시 “서비스 연결 해제 완료” 등을 확인한 뒤 다음 init을 하거나, delay를 설정 가능하게 두는 방안 검토.
2. **ensurePlayEngineReadyForLevel 타임아웃 125초**  
   매우 긴 대기. 초기화가 오래 걸리는 기기에서만 의미 있으나, 중간에 “준비 중” 진행 상태를 보여주거나, 타임아웃 시 사용자에게 재시도/설정 확인 유도 메시지를 주는 것이 좋음.  
   → **125초 타임아웃이 실제로 발생하는 경우**는 아래 §9 참고.
3. **restartEngineForNewConfig 노출**  
   설정 변경 후 엔진 재시작이 필요한 경우를 위해, 설정 UI에서 이 함수를 호출하는 경로가 있는지 확인하고, 없으면 버튼/자동 호출 조건을 추가하는 것이 좋음.
4. **EngineViewModel 규모**  
   초기화·세션 전환·분석·복구 등이 한 ViewModel에 모여 있어 파일이 매우 큼. “엔진 생명주기/세션 전환” 전담 클래스로 분리하면 유지보수와 테스트가 수월해질 수 있음.

---

## 9. 요약 표

| 항목 | 구현 방식 | 효율성 |
|------|-----------|--------|
| 엔진 초기화 (앱 기동) | 모델만 준비, 엔진은 세션 진입 시 지연 초기화 | 좋음 (불필요한 init 없음) |
| 세션 전환 (PLAY ↔ ANALYSIS) | 프로필 비교 → 불일치 시 이전 shutdown → 350ms → 현재 init | 좋음 (필요 시에만 재초기화) |
| 대국 시작 시 기력 반영 | ensurePlayEngineReadyForLevel → setActiveSession(PLAY)에서 playLevelIndex 반영 | 좋음 (한 경로로 일관 처리) |
| 분석 중단 | stopAllAnalysis + STOP 우선순위 | 좋음 (즉시 반영) |
| 엔진 종료 | 세션 전환 시 이전 세션만 shutdown | 좋음 (리소스 명확) |
| 레벨별 모델/설정 | getModelFileForPlayLevel + play.cfg | 좋음 (역할 분리 명확) |
| 설정 변경 후 재시작 | restartEngineForNewConfig (호출처 확인 필요) | 보통 (UI 연동 확인 권장) |

전체적으로 **대국모드에서의 엔진 초기화·정지·재개·기력별 엔진 교체**는 일관된 프로필 비교와 세션별 분리로 **효율적으로 구성**되어 있으며, 위 개선 제안을 반영하면 더 안정적이고 유지보수하기 쉬운 구조가 될 수 있습니다.

---

## 10. 엔진 준비 타임아웃(60초)이 발생하는 경우

`ensurePlayEngineReadyForLevel(levelIndex)`는 **대국 시작 버튼**(`PlayViewModel.startGame()`)에서만 호출됩니다.

### 9.1 동작 요약 (60초 타임아웃 + 실패 시 대국 미시작)

1. `setPlayLevelForModelSelection(levelIndex)` → `setActiveSession(Session.PLAY)` 호출.
2. **`setActiveSession(PLAY)`는 엔진 초기화를 기다리지 않고 반환**합니다.  
   내부에서 `engineOpJob = viewModelScope.launch { ... }`로 초기화를 **비동기**로만 실행합니다.
3. 그 직후 `_uiState.value.isEngineReady`가 이미 true면(이미 같은 프로필로 PLAY 엔진이 떠 있는 경우) 곧바로 **true 반환**.
4. 그렇지 않으면 **최대 60초 동안** `_uiState`에서 `isEngineReady == true`가 나올 때까지 대기합니다.  
   `withTimeoutOrNull(125_000L) { _uiState.drop(1).first { it.isEngineReady } }`  
   → 60초 안에 true가 나오면 **true 반환**. 한 번도 안 나오면 **타임아웃**: statusMessage에 재시도 안내 설정 후 **false 반환**.
5. `startGame()`은 반환값을 사용합니다. **false이면 대국을 시작하지 않고** return. **true일 때만** 세션 시작 + newGame() 진행.  
   따라서 **60초 타임아웃이 나면 = “엔진이 준비되지 않은 상태로 대국이 시작된 것처럼”** UI만 바뀌고, 봇 수가 안 나오거나 “준비 중”으로 멈춰 있을 수 있습니다.

즉, **125초 타임아웃이 “생긴다”**는 것은  
**“대국 시작을 눌렀는데, 125초가 지날 때까지 PLAY 엔진 초기화가 끝나지 않아서 `isEngineReady`가 true로 한 번도 안 바뀐 경우”**를 말합니다.

### 10.2 60초 안에 초기화가 끝나지 않는 대표적인 경우

| 원인 | 설명 |
|------|------|
| **초기화 지연** | `initEngine()`은 내부적으로 120초 타임아웃. 첫 실행 등으로 초기화가 60초 넘게 걸리면, 60초 대기 상한에 먼저 걸려 타임아웃. |
| **초기화 실패 후 재시도** | 120초 init 타임아웃/크래시 후 CPU 폴백·Safe Mode 등 재시도가 이어지면, 총 시간이 60초를 넘길 수 있음. |
| **헬스체크 실패** | init 성공 후 헬스체크 실패 시 예외로 종료되어 `isEngineReady`가 설정되지 않음 → 60초 대기 후 타임아웃. |
| **서비스/저속 스토리지** | 서비스 바인딩·모델 로딩이 매우 느린 기기에서 60초 내 완료되지 않음. |

### 10.3 정리

- **60초**는 “엔진이 준비될 때까지 기다리는 **상한**”입니다.  
  정상 기기에서는 10~30초 안에 끝나므로, 60초가 지나도 안 되면 재시도/안내가 낫습니다.
- **125초 타임아웃이 “생기는” 경우**는  
  **엔진 초기화가 125초 안에 성공하지 못하는 모든 경우**(120초 init 타임아웃, 재시도 지연, 헬스체크 실패, 서비스/기기 지연 등)입니다.
- 타임아웃 시 `ensurePlayEngineReadyForLevel`은 예외를 던지지 않고 그냥 return하므로, **사용자에게는 “대국이 시작됐는데 엔진이 준비되지 않았다”**는 상태가 노출될 수 있어, “준비 중” UI나 125초 초과 시 “엔진 준비 실패·재시도” 안내를 주는 것이 좋습니다.
