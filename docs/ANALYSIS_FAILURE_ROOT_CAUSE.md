# 분석 실패 근본 원인 및 수정 요약

## 1. 근본 원인

### 1.1 분석 모드에서 분석 엔진이 초기화되지 않은 채 `analyze()` 호출

- **흐름**: 분석 탭(ANALYSIS)에서 `triggerAnalysis()` → `useCaseFor(ANALYSIS).analyze()` → **KataGoService**(분석 전용 엔진) 호출.
- **문제**: `ensureAnalysisEngineReadyForHint()`는 **PLAY + HINT/DEEP** 일 때만 호출되고, **ANALYSIS 세션**일 때는 호출되지 않음.
- **결과**: 다음 같은 경우에 분석 엔진이 한 번도 초기화되지 않은 상태로 `analyze()`가 호출됨.
  - 앱 기동 후 모델 준비가 끝나고 `onPreparationCompleted()`가 **비동기**로 `initializeEngine()`를 호출하는데, 그 전에 사용자가 **PLAY**로 전환한 경우 → PLAY 엔진만 초기화되고, 다시 분석 탭으로 돌아와도 **분석 엔진은 미초기화**.
  - 또는 세션 전환/재시작 등으로 분석 엔진만 아직 초기화되지 않은 상태에서 분석 요청이 먼저 나간 경우.
- **증상**: KataGoService는 바인딩·네이티브 로드는 되어 있지만 `initEngine()`가 호출되지 않아, `analyzeTop4()`가 빈 배열을 반환하거나 내부적으로 실패 → **“분석에 실패했습니다”** 반복.

### 1.2 엔진 초기화 실패 시에도 분석 트리거 (이전 수정)

- **문제**: `setActiveSession(ANALYSIS)`에서 분석 엔진을 **방금** 초기화했는데 실패해도, `isModelReady`가 이전(PLAY) 세션 값으로 남아 있어 `triggerAnalysis()`가 계속 호출됨.
- **결과**: 엔진 미준비 상태에서 분석만 반복 → 빈 결과 → “분석에 실패했습니다” 반복.

### 1.3 엔진 초기화 경로에서의 세션 구분

- `initializeEngine()`는 **현재 `analyzingSession`**에 따라 `engineController`(분석/대국)를 선택함.
- `onPreparationCompleted()`는 **비동기 job**에서 `initializeEngine()`를 호출하므로, job이 **실행되는 시점**의 `analyzingSession`으로 엔진이 결정됨.
- 따라서 “모델 준비 완료 → 분석 엔진 초기화” 직전에 사용자가 PLAY로 바꾸면 **PLAY 엔진만** 초기화되고, 분석 엔진은 초기화되지 않을 수 있음.

---

## 2. 수정 사항

### 2.1 분석 모드에서 분석 전용 엔진 보장 (근본 수정)

**파일**: `EngineViewModel.kt` – `triggerAnalysis()` 내부

- **변경**: `analysisSessionSnapshot == Session.ANALYSIS`일 때, 분석 요청 직전에 **반드시** `ensureAnalysisEngineReadyForHint(snapshot.game.boardSize)` 호출.
- **효과**: 분석 탭에서 분석을 돌릴 때마다, 분석 전용 엔진(KataGoService)이 한 번도 초기화되지 않았으면 그 시점에 초기화한 뒤 `analyze()` 호출.  
  → 분석 엔진 미초기화로 인한 빈 결과/실패 반복 제거.

### 2.2 세션 전환 시 “방금 초기화”한 엔진만 성공했을 때 분석 트리거 (이전 수정 유지)

**파일**: `EngineViewModel.kt` – `setActiveSession()`

- **변경**  
  - 세션 전환 시 **엔진을 새로 초기화하는 경우** (`didInit == true`):  
    - 먼저 `isModelReady = false`로 설정해 이전 세션의 “준비됨” 상태 제거.  
    - 초기화 후 `triggerAnalysis()`는 **초기화가 성공했을 때만** 호출 (`!didInit || ready`).  
  - 초기화 실패 시: `triggerAnalysis()` 호출하지 않고, `statusMessage = msg_model_preparing` 및 로그 출력.
- **효과**: 엔진 초기화 실패 시 “분석 실패” 메시지가 반복되지 않음.

### 2.3 진단 로그

- **빈 결과 시**: `handleAnalysisResult()`에서 `result.suggestions.isEmpty()`일 때  
  `totalVisits`, `session`, `isModelReady` 로그로 원인 구분 (엔진 미준비 vs 파싱/필터).
- **분석 엔진 보장 호출 시**: `ensureAnalysisEngineReadyForHint()`에서  
  분석 엔진을 실제로 초기화할 때 `everInit` 값 로그.
- **엔진 초기화 시작 시**: `initializeEngine()`에서  
  `session=ANALYSIS|PLAY` 로그로 어떤 엔진을 초기화하는지 확인 가능.

---

## 3. 로그로 확인하는 방법

- **분석 엔진이 지금 초기화되는지**  
  - `ensureAnalysisEngineReadyForHint: initializing analysis engine (everInit=...)`
- **어떤 세션 엔진을 초기화하는지**  
  - `initializeEngine start session=ANALYSIS` / `session=PLAY`
- **빈 결과 원인**  
  - `Analysis returned empty suggestions: totalVisits=... session=... isModelReady=...`  
  - `totalVisits=0` → 엔진 미준비/예외 가능성 큼.

---

## 4. 요약

| 원인 | 수정 |
|------|------|
| 분석 모드에서 분석 엔진 미초기화 상태로 `analyze()` 호출 | ANALYSIS 세션일 때 `triggerAnalysis()` 직전에 `ensureAnalysisEngineReadyForHint()` 호출로 분석 엔진 보장 |
| 세션 전환 후 초기화 실패해도 이전 세션의 `isModelReady`로 분석 반복 | 전환 시 `isModelReady=false` 후, 초기화 성공 시에만 `triggerAnalysis()` 호출 |
| 원인 추적 어려움 | 빈 결과/분석 엔진 보장/엔진 초기화 시점에 로그 추가 |

엔진 초기화 경로(세션 구분, 실패 시 분석 스킵)와 분석 호출 경로(ANALYSIS일 때 분석 엔진 보장)를 함께 수정해, “분석에 실패했습니다”가 반복되던 근본 원인을 제거했다.
