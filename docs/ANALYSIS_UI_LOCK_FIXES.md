# 분석 UI 잠금/불일치/탭 허용 수정 요약

의견과 제시하신 내용을 반영해 적용한 수정 사항입니다.

---

## 1. UI 잠금 해제를 모든 실패/스킵 경로에서 수행 (EngineViewModel.kt)

**의도**: 분석 시작 시 `suspendUiDuringAnalysis = true`로 잠그는데, 결과 스킵/예외 시 해제가 없어 "추천은 보이는데 착수 불가"가 발생. 모든 해당 경로에서 `suspendUiDuringAnalysis = false`로 되돌림.

### 1.1 예외 catch (triggerAnalysis)

```kotlin
} catch (t: Throwable) {
    ErrorHandler.recordExpectedError("EngineViewModel", "analysis_failed", t)
    val detail = app.getString(R.string.analysis_event_error, t.localizedMessage ?: t.javaClass.simpleName)
    statusSnapshot = StatusSnapshot(StatusType.ERROR, errorMessage = detail)
    _uiState.update {
        it.copy(
            statusMessage = detail,
            overlay = it.overlay.copy(suspendUiDuringAnalysis = false)
        )
    }
}
```

### 1.2 게이트 스킵 시 (handleAnalysisResult 직후 return)

```kotlin
if (!AnalysisResultGate.shouldProcessResult(...)) {
    _uiState.update { it.copy(overlay = it.overlay.copy(suspendUiDuringAnalysis = false)) }
    return@withContext
}
```

### 1.3 타임아웃 시 (result.timedOut)

```kotlin
if (result.timedOut) {
    ...
    _uiState.update {
        it.copy(
            statusMessage = detail,
            overlay = it.overlay.copy(suspendUiDuringAnalysis = false)
        )
    }
    return@withContext
}
```

---

## 2. 불일치(해시/수순) 시 즉시 재분석 트리거 (EngineViewModel.kt)

**의도**: 불일치 시 추천만 비우고 끝나면 "추천 표시/착수 불가"처럼 보일 수 있음. 불일치 시점에 현재 수순 기준으로 재분석을 한 번 트리거해 안정화. 짧은 debounce(150ms)로 연쇄 트리거 완화.

### 해시 불일치 시 (analyzedMovesHash != computeMovesHash(current))

기존: 추천 비우기 + `suspendUiDuringAnalysis = false` 후 return.  
추가: **`triggerAnalysis(debounceMs = 150L)`** 호출.

- 크기 불일치(analyzed.size != current.size) 구간은 이미 `triggerAnalysis(debounceMs = 0L)` 있음 → 유지.

---

## 3. 수순 변경 시 분석 결과 “늦게 들어옴” 완화 (EngineViewModel.kt)

**의도**: 수를 두거나 되돌릴 때 이전 분석 결과가 늦게 들어오면 불일치 발생. 수순이 바뀌는 시점에 **요청 ID를 무효화**해 늦게 도착한 결과가 게이트에서 스킵되도록 함.

다음 진입점에서 **수순 변경 직후** `lastAnalysisRequestId = 0L` 추가:

- `playMove`: 좌표/돌 검사 직후, `preActionId` 전
- `undo`: `moves == snapshot.game.moves` return 직후
- `redo`: `finalMoves == snapshot.game.moves` return 직후
- `jumpToStart`: `moves.isEmpty()` return 직후
- `jumpToEnd`: `redoStack.isEmpty()` return 직후
- `jumpBack5`: `moves.isEmpty()` return 직후
- `jumpForward5`: `redoStack.isEmpty()` return 직후

---

## 4. BoardSection tapAllowed를 추천 표시와 동일 기준으로 (BoardSection.kt)

**의도**: 지금은 `suspendUiDuringAnalysis`만으로 탭을 막아, 이 값이 꼬이면 추천이 있어도 탭이 막힘. “추천 표시/분석 준비”와 “탭 허용”을 같은 기준으로 맞춤.

### 변경 내용

- **추천 표시 조건**을 한 번만 정의:  
  `canShowSuggestions = !suspendUiDuringAnalysis && (!isVariationAnalysisMode || isInitialAnalysisReady)`
- **tapAllowed**:  
  `isModelReady && !isPreparing && (canShowSuggestions || variationPopupState != null || (topSuggestions.isNotEmpty() && isInitialAnalysisReady))`  
  → 추천이 실제로 있을 때도 탭 허용되도록 보완.
- **Board suggestions**:  
  `if (canShowSuggestions) uiState.analysis.topSuggestions else emptyList()`  
  → 추천 표시와 동일한 조건 사용.

---

## 파일별 변경 요약

| 파일 | 변경 |
|------|------|
| **EngineViewModel.kt** | (1) 예외 catch에서 overlay.copy(suspendUiDuringAnalysis = false) (2) 게이트 return 직전 동일 (3) timedOut 시 동일 (4) 해시 불일치 시 triggerAnalysis(150L) (5) playMove/undo/redo/jumpToStart/jumpToEnd/jumpBack5/jumpForward5에서 lastAnalysisRequestId = 0L |
| **BoardSection.kt** | canShowSuggestions 도입, tapAllowed를 추천 표시·보완 조건과 일치, suggestions를 canShowSuggestions 기반으로 통일 |

수정만 반영했고, 빌드/동작 확인은 프로젝트에서 진행하면 됩니다.
