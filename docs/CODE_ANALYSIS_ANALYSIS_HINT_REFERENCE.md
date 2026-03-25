# 코드 분석: 참고도·힌트·분석 모드 추천수 동작

코드만으로 추적한 “참고도 분석”, “힌트 분석”, “분석 모드 추천수” 동작과, **작동 안 할 수 있는 경우 / 보이다가 사라지는 경우**를 정리했습니다.

---

## 1. 참고도 분석 (DEEP / isVariationAnalysisMode)

### 1.1 진입 경로

- **분석 모드:** `triggerDeepAnalysis()` → `confirmDeepAnalysis()`  
  - `isVariationAnalysisMode = true`, `analysisMode = DEEP`, `topSuggestions = emptyList()`, `isInitialAnalysisReady = false`  
  - `setTargetLatency(15초)`, `updateAnalysisIntent(CONFIRM_DEEP)`
- **대국 모드:** PlayScreenRoot → `onDeepAnalysis` → `playViewModel.triggerDeepAnalysis()` → 동일하게 `confirmDeepAnalysis()` 호출

### 1.2 분석 실행

- `runAnalysisInternal(intent)`: `mode == DEEP` → `targetMs = 30000L`, `maxVisits = 5000`
- `requestSession`: PLAY + DEEP이면 **Session.ANALYSIS** 엔진 사용 (`needsProEngine`)
- 분석 시작 시 **HINT/DEEP은 keepSuggestions = false** → 기존 추천 비우고 시작

### 1.3 결과 반영 (참고도용 추천 표시)

- **중간 결과:** `shouldShowSuggestions = (totalVisits >= 1000 && result.suggestions.isNotEmpty())` 이면 추천 표시
- **최종 결과:** `shouldShowFinalSuggestions = result.suggestions.isNotEmpty()` 이면 `topSuggestions` / `isInitialAnalysisReady` 설정

### 1.4 참고도 팝업 (추천수 클릭 시)

- `onCellClicked(coord)` → `suggestion = topSuggestions.find { it.coord == coord }`  
  → `(isVariationAnalysisMode || analysisMode == HINT)` 이고 `suggestion != null` 이고 `suggestion.pv.isNotEmpty()` 이면 `playVariation(suggestion)` 호출
- **`playVariation(suggestion)`:**  
  - `suggestion.pv`가 비어 있으면 로그만 하고 **return (참고도 안 열림)**  
  - PV를 보드 범위로 필터한 뒤 `variationPopupState` 설정, `startVariationPlaybackLoop()` 호출

### 1.5 참고도가 “작동 안 할 수 있는” 코드상 이유

| 구간 | 조건 | 결과 |
|------|------|------|
| 분석 트리거 | `updateAnalysisIntent()`에서 `!isModelReady` 또는 `analyzingSession == OFFLINE` | 의도 제출 자체가 안 됨 → 분석 미시작 |
| 오케스트레이터 | `StartAnalysis` 수신 시 상태가 `Ready`가 아님 (Idle/Switching/Analyzing) | 분석 스킵 |
| 결과 수락 | `handleAnalysisResult` 진입 전 `AnalysisResultGate.shouldProcessResult` 거절 (requestId/epoch/boardSize 불일치) | 결과 미반영 |
| 수순 불일치 | `result.analyzedMovesHash != computeMovesHash(current)` | `topSuggestions` 비우고 return → 추천 사라짐 |
| 참고도 열기 | `suggestion.pv.isEmpty()` | `playVariation`에서 바로 return → 팝업 안 뜸 |

---

## 2. 힌트 분석 (HINT)

### 2.1 진입 경로

- **대국 모드:** `onHint` → `playViewModel.triggerHintAnalysis()` → `engineViewModel.triggerHintAnalysis()`
- **내부:**  
  - `analysisMode = HINT`, `isInitialAnalysisReady = false`, `topSuggestions = emptyList()`, `isVariationAnalysisMode = false`  
  - `updateAnalysisIntent(TRIGGER_HINT)`

### 2.2 분석 실행

- `runAnalysisInternal`: `mode == HINT` → `targetMs = 20000L`, `maxVisits = 1000`
- PLAY + HINT → **Session.ANALYSIS** 엔진 사용
- 시작 시 `keepSuggestions = false` → 기존 추천 비움

### 2.3 결과 반영 (힌트용 추천 표시)

- **중간 결과** (엔진이 `onIntermediateResult` 콜백 호출 시):  
  `shouldShowSuggestions = result.suggestions.isNotEmpty() && ((totalElapsedMs >= 20_000L) || (totalElapsedMs >= 10_000L && result.totalVisits >= 1000))`  
  → 20초 경과 또는 (10초 경과 + 1000수) 일 때만 추천 표시
- **최종 결과:** `result.suggestions.isNotEmpty()` 이면 표시, HINT일 때는 상위 4개만 (`ensuredSuggestions.take(4)`)

### 2.4 힌트가 “작동 안 할 수 있는” 코드상 이유

| 구간 | 조건 | 결과 |
|------|------|------|
| 분석 미시작 | 참고도와 동일 (isModelReady, 오케스트레이터 상태, intent 스킵 등) | 추천/참고도 둘 다 없음 |
| 중간 결과 없음 | 네이티브가 `onIntermediateResult`를 호출하지 않으면 | 중간에 추천이 안 뜨고, **최종 한 번만** 반영됨 (최종에 suggestions 있으면 그때 표시) |
| 최종 빈 결과 | `!isIntermediate && result.suggestions.isEmpty()` | 추천 0개로 처리, 재시도 1회 후 포기 가능 |
| requestId/epoch 불일치 | 새 분석이 이미 시작된 뒤 이전 결과 도착 | `handleAnalysisResult`에서 결과 버림 |

---

## 3. 분석 모드 추천수 (NORMAL/QUICK)

### 3.1 표시 조건 (BoardSection)

- `canShowSuggestions = !suspendUiDuringAnalysis && (!isVariationAnalysisMode || isInitialAnalysisReady)`  
  → 참고도 모드가 아니거나, 참고도 모드면 “분석 준비 완료”일 때만 추천 표시
- `suggestions = if (canShowSuggestions) uiState.analysis.topSuggestions else emptyList()`  
  → `canShowSuggestions`가 false면 **항상 빈 리스트**로 그려서 “안 보임”

### 3.2 분석 모드에서 분석 시작 시

- `runAnalysisInternal`에서 **ANALYSIS 세션**이면 `suspendUiDuringAnalysis = true`  
  → 분석 중에는 `canShowSuggestions = false` → **분석이 끝나기 전까지 추천수 비표시** (의도된 동작)
- NORMAL/QUICK는 `keepSuggestions = true` → 점프/undo 등으로 국면만 바뀌기 전까지는 기존 추천 유지

### 3.3 추천수가 “안 보이는” 코드상 이유

| 원인 | 위치/조건 |
|------|-----------|
| 분석이 한 번도 실행 안 됨 | 저장된 게임 없을 때 트리거 누락(이미 수정), 또는 `updateAnalysisIntent` 조기 return, 오케스트레이터 스킵 |
| 표시 플래그 | `suspendUiDuringAnalysis == true` (분석 중) → `canShowSuggestions = false` |
| 참고도 모드인데 준비 전 | `isVariationAnalysisMode && !isInitialAnalysisReady` → `canShowSuggestions = false` |
| 데이터 자체가 비어 있음 | `topSuggestions`가 비어 있음 (아래 “보이다가 사라지는 경우” 참고) |

---

## 4. 추천수 “보이다가 갑자기 안 보이는” 경우 (코드상 가능한 경로)

아래는 모두 **`topSuggestions = emptyList()` 또는 `isInitialAnalysisReady = false`** 를 설정하는 지점입니다.

### 4.1 사용자 액션으로 인한 비우기

| 액션 | 파일/라인 근거 | 효과 |
|------|----------------|------|
| 새 대국 | `newGame()` | topSuggestions, isInitialAnalysisReady 초기화 |
| 점프(처음/끝/5전/5후) | `jumpToStart`, `jumpToEnd`, `jumpBack5`, `jumpForward5` | topSuggestions 비움, 모드에 따라 DEEP/QUICK 유지 |
| Undo / Redo | `undo()`, `redo()` | topSuggestions 비움 |
| 힌트 버튼 | `triggerHintAnalysis()` | topSuggestions 비움 후 새 분석 시작 |
| 참고도 분석 시작 | `confirmDeepAnalysis()` | topSuggestions 비움 후 DEEP 분석 시작 |
| SGF 로드 | `loadSgf` | topSuggestions, isInitialAnalysisReady 초기화 |
| 대국 복귀 | `returnToGame()` | isInitialAnalysisReady = false (topSuggestions는 유지 가능) |

### 4.2 결과 처리 도중 비우기

| 조건 | 위치 | 효과 |
|------|------|------|
| 수순 불일치 | `handleAnalysisResult`: `result.analyzedMovesHash != computeMovesHash(current)` | topSuggestions 비우고 return (“유령 추천” 방지) |
| moves 개수 불일치 | `handleAnalysisResult`: `analyzed.size != current.size` | topSuggestions 비우고 return |
| 게이트 거절 | `AnalysisResultGate.shouldProcessResult` false → 결과 전체 미적용 | 이전 추천은 그대로지만, **새 결과로 갱신이 안 됨** (다음 액션에서 비워질 수 있음) |

### 4.3 분석 시작 시 비우기 (HINT/DEEP)

- `runAnalysisInternal`에서 `keepSuggestions = (mode == QUICK || mode == NORMAL)`  
  → **HINT/DEEP일 때는** 분석 시작 시 `preservedSuggestions = emptyList()`, `preservedReady = false`  
  → “힌트/참고도 분석 시작” 순간 **추천이 사라진 것처럼 보임** (의도된 동작, 준비 중 UI로 전환)

### 4.4 세션/복원

- `EngineSessionOrchestrator.switchSession`: 복원된 `SessionState`의 analysis 사용 → 캐시/디스크에 따라 topSuggestions가 빈 상태로 복원될 수 있음
- `applySnapshot`(복원): topSuggestions, isInitialAnalysisReady 초기화

---

## 5. 작동 여부 요약 (코드 기준)

- **참고도 분석:**  
  - 트리거 → 오케스트레이터 → DEEP 분석 → 결과 반영 → 추천수 클릭 시 `playVariation` 까지 경로는 일치.  
  - **실제로 안 되면:** 분석 미시작(isModelReady/오케스트레이터), 결과 게이트/수순 불일치로 미반영, 또는 **PV 비어 있어서 참고도 팝업이 안 열리는 경우**를 의심할 수 있음.
- **힌트 분석:**  
  - 트리거 → TRIGGER_HINT intent → ANALYSIS 엔진으로 HINT 분석 → 중간/최종 결과 반영 경로 존재.  
  - **실제로 안 되면:** 위와 동일 + “중간 콜백 미호출”이면 20초/10초 조건은 의미 없고, **최종 결과만** 반영 여부가 중요.
- **분석 모드 추천수:**  
  - `canShowSuggestions`와 `topSuggestions`만 맞으면 표시됨.  
  - **안 보이면:** 분석 미실행, `suspendUiDuringAnalysis`/참고도 모드+미준비, 또는 위 “비우기” 경로 중 하나가 실행된 경우.
- **보이다가 사라짐:**  
  - 점프/undo/redo/힌트/참고도 시작/새대국/SGF 로드 시 **항상** 비우도록 되어 있음.  
  - 그 외에 **수순 불일치·moves 개수 불일치** 시에도 비워서, “분석 중에 사용자가 수를 두거나 점프한 경우” 추천이 사라지는 것이 코드상 정상 동작입니다.

---

## 6. 추가로 확인하면 좋은 코드 포인트

1. **엔진 콜백**  
   - `onIntermediateResult`가 HINT/DEEP 분석 중에 실제로 호출되는지(네이티브/엔진 바인딩).  
   - 호출이 없으면 힌트/참고도 모두 “최종 결과 한 번”만 반영됨.

2. **jumpForward5의 모드**  
   - `jumpForward5`만 `analysisMode = AnalysisMode.QUICK`로 고정하고 `isVariationAnalysisMode`는 건드리지 않음.  
   - 참고도 모드에서 5후로 이동 시 모드와 추천 표시 조건이 의도와 맞는지 한 번 더 확인할 만함.

3. **returnToGame 후**  
   - `isInitialAnalysisReady = false`만 설정하고 topSuggestions는 그대로 둠.  
   - 이후 `triggerQuickAnalysis`로 새 분석이 돌면 갱신되므로, “잠깐 추천이 남아 있다가 새 결과로 바뀌는” 동작은 코드와 부합함.

이 문서는 **코드 상의 분기와 상태 변경만**을 기준으로 한 분석이며, 실제 동작은 빌드/엔진/디스크 상태에 따라 달라질 수 있습니다.
