# 과도한 착수 제한이 걸린 이유 / 대국 모드 조건

## 1. “과도하게 제한”이 걸린 이유

### 1.1 우리가 **추가**한 제한 (이번 수정 과정에서)

**한 가지만** 추가했습니다.

- **위치**: `EngineViewModel.onCellClicked()` 안의 **"4.5" 블록**
- **내용**:  
  `analyzingSession == Session.ANALYSIS` 이고  
  `!isInitialAnalysisReady && topSuggestions.isEmpty()` 이면  
  **탭을 무시하고 return (playMove 호출 안 함)**
- **넣은 이유**:  
  당시 증상이 “분석 모드에서 추천수 안 뜨고, 빈 점 누르면 **해당 지점에는 둘 수 없습니다**만 뜬다”였기 때문에,  
  “추천이 아직 없을 때는 탭 자체를 무시해서 그 메시지가 안 나오게 하자”고 넣은 것입니다.
- **부작용**:  
  분석이 한 번도 안 돌거나, 결과가 비어 있으면 `isInitialAnalysisReady`/`topSuggestions`가 계속 비어 있고,  
  그동안 **분석 모드에서 아무 점이나 눌러도 착수가 전혀 안 되는** “과도한 제한”이 됐습니다.  
  → 그래서 이 4.5 블록은 **이미 제거한 상태**입니다.

정리하면, **착수 조건을 “이번에 여러 개 바꾼” 게 아니라, “그때 한 번 4.5만 넣었고, 그 한 블록이 과도한 제한의 직접 원인**입니다.

### 1.2 원래부터 있던 제한 (우리가 바꾼 적 없음)

- **`suspendUiDuringAnalysis`**  
  - 분석 **시작** 시 `overlay.suspendUiDuringAnalysis = (sessionSnapshot != Session.PLAY)`  
    → 분석 **세션**에서는 분석하는 동안 `true`.  
  - **해제**는 `handleAnalysisResult` 등에서 결과를 적용할 때만 `false`로 되돌림.  
  - 그래서 **분석이 실패하거나 결과가 스킵되면** 이 값이 `true`로 남고,  
    UI의 `tapAllowed`가 false가 되어 “분석 모드에서 착수 불가”가 계속되는 구조는 **원래 설계**입니다.  
  - 이 값은 “그동안 수정하면서 착수조건 제한을 바꾼” 부분이 아니라, **원래 코드/문서(ANALYSIS_MODE_ROOT_CAUSE, TANGLED_STATE 등)에 있던 동작**입니다.
- **`tapAllowed` / `canShowSuggestions`**  
  - `suspendUiDuringAnalysis == true`이면 착수·추천 표시가 막히는 식의 조건도 **원래부터** 있었고, 우리가 조건식 자체를 바꾼 적 없습니다.

즉, **과도하게 막힌 느낌**은  
(1) 우리가 넣은 **4.5 한 블록**이 “추천 없으면 무조건 탭 무시”로 착수를 막은 것,  
(2) 거기에 원래부터 있던 **분석 중 UI 잠금(suspendUiDuringAnalysis)** 이 겹친 결과입니다.

---

## 2. 대국 모드 조건이 “작살”난 건가?

**대국 모드 전용으로 “착수 조건”을 우리가 바꾼 적은 없습니다.**  
대국 모드에서 쓰는 조건은 예전부터 있던 그대로입니다.

- **봇 착수**  
  - `PlayViewModel`에서 `isBotMoveReady(state)`가 true여야 봇이 둠.  
  - `isBotMoveReady`는  
    `isModelReady`, `isEngineReady`, `!isVariationAnalysisMode`, `analysisMode != HINT`,  
    그리고 **`isInitialAnalysisReady || lastTotalVisits >= 1`** 를 요구합니다.  
  - 즉, **분석이 한 번이라도 끝나야** 봇이 수를 둡니다.  
  - 분석이 아예 안 돌거나, 계속 스킵되면 `isBotMoveReady`가 false라서 **봇이 안 두는 건 설계 그대로**입니다.
- **사용자 착수**  
  - `playMove(coord, isBot = false)` 안에서  
    `analysisMode == AnalysisMode.HINT`이면 **사람 착수만** return으로 막습니다 (원래부터 그렇게 되어 있음).  
  - 그리고 토큰 부족이면 `ensureActiveGameTokens`에서 허용 안 되고 착수 revert (이것도 원래 로직).

그래서 “대국 모드 조건을 우리가 망가뜨렸다”기보다는,  
**분석이 제대로 완료되지 않는 상황**(엔진/의도/결과 스킵 등)이면  
- 봇은 “준비 안 됨”으로 안 두고,  
- 사용자는 HINT 모드에 걸리거나 토큰에 걸리면 막히는  
**원래 구조**입니다.  
즉, **조건 자체를 우리가 작살 낸 게 아니라**, 그 조건이 의존하는 **분석/엔진 쪽이 안 돌면 대국 모드도 같이 막히는 구조**입니다.

---

## 3. 정리

| 구분 | 원인 | 우리가 착수 조건을 바꾼 적 있는지 |
|------|------|----------------------------------|
| 분석 모드 착수 안 됨 | (1) **4.5 블록**으로 “추천 없으면 탭 무시” 추가 → 과도한 제한 (이미 제거함) (2) 원래 있던 `suspendUiDuringAnalysis`로 분석 중 UI 잠금 | **4.5만 추가했다가 제거.** 나머지 조건은 건드리지 않음. |
| 대국 모드 봇/사용자 착수 | 봇은 “분석 한 번이라도 완료”에 의존, 사용자는 HINT/토큰 등 **기존 조건** 그대로. 분석이 안 돌면 둘 다 막힘. | **대국 모드 전용 착수 조건은 바꾼 적 없음.** |

그동안 수정하면서 **착수 조건을 여러 개 바꾼 적은 없고**,  
실제로 추가했다가 제거한 건 **“아직 추천 없을 때 탭 무시” 한 군데(4.5)** 뿐입니다.
