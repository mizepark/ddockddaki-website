# 추천수 미표시·착수 불가·대국 모드 미실행 — 원인 분석

**목적**: 땜빵 수정 없이, 현상과 코드 경로만으로 원인을 정리한다.  
**전제**: 사용자는 "참고도/힌트 분석"을 누르지 않았음 → 기본은 분석 모드 QUICK(또는 NORMAL).  
**관찰**: 다른 모드로 나갔다가 다시 분석 모드로 들어오면 **추천수는 표시**되지만 **착수는 여전히 안 됨**.

---

## 1. 추천수가 안 나오는 원인 (분석 모드 첫 진입 시)

추천수(`topSuggestions`)는 **분석 결과 콜백**이 `handleAnalysisResult`를 타고 들어와서만 갱신된다.

| 가능 원인 | 경로 | 설명 |
|-----------|------|------|
| **엔진 미준비** | `runAnalysisInternal` 시작 시 `if (!snapshot.isModelReady) return` | 모델/엔진이 준비되기 전에 분석 탭만 열면 분석이 아예 시작되지 않음. `updateAnalysisIntent`도 `if (!snapshot.isModelReady) return`으로 의도만 제출하고 실행은 스킵. |
| **분석 트리거 누락** | 분석 탭 첫 진입 시 `updateAnalysisIntent` 호출 경로 | 세션 전환 후 `onSessionSwitched`에서 `updateAnalysisIntent(SET_ACTIVE_SESSION)` 호출. 단, 그 직전에 `_uiState.value = result.newUiState`로 **복원된 상태**를 넣는데, 이때 `isModelReady`는 **base(이전 세션 UI 상태)**에서 옴. PLAY→ANALYSIS 전환이면 PLAY 시점의 isModelReady를 쓰므로, 타이밍에 따라 분석이 한 번 스킵될 수 있음. |
| **결과가 게이트에서 폐기됨** | `handleAnalysisResult` 진입 전 `AnalysisResultGate.shouldProcessResult` / `AnalysisSnapshotValidator.shouldProcessAnalysisResult` | `runEpoch < lastSubmittedEpoch`이면 이전 세대 결과로 간주해 **무시**. 의도 제출 시마다 `currentEpoch++` 하므로, 사용자 이벤트나 다른 갱신이 계속 들어오면 **가장 최근 분석 결과만** 통과하고 그 전 결과는 모두 버려짐. |
| **중간 결과만 오고 스킵** | `if (isIntermediate && result.totalVisits < 2) return` | 엔진이 중간 콜백만 보내고 totalVisits가 1 이하이면 **아무 상태 갱신도 하지 않고 return**. 이 경우 `topSuggestions`/`isInitialAnalysisReady`는 갱신되지 않고, **최종 결과**가 와야만 추천이 뜸. |
| **수순 불일치** | `result.analyzedMovesHash != computeMovesHash(current)` 또는 `analyzed.size != current.size` | 분석 요청 시점의 수순과 현재 수순이 다르면 **추천 비우기**만 하고 return. 탭 전환·다른 이벤트로 수순이 바뀌면 결과가 계속 버려질 수 있음. |

→ **정리**: 첫 진입 시에는 “엔진 준비 지연 + 분석 한 번도 안 돌거나, 돌아도 epoch/수순 불일치로 결과가 다 폐기”되면 추천수가 계속 비어 있을 수 있다.

---

## 2. 착수가 안 되는 원인 (분석 모드·대국 모드 공통 가능성)

착수는 **보드 탭 → onCellClicked / onBoardCellTapped → playMove** 한 경로로만 들어간다. 그 사이에 **여러 단계에서 조기 return**이 있다.

### 2.1 분석 모드 (MainScreen → EngineViewModel.onCellClicked)

| 단계 | 조건 | 위치 | 설명 |
|------|------|------|------|
| 1 | `state.overlay.suspendUiDuringAnalysis == true` | `onCellClicked` 첫 검사 | **참이면 그대로 return.** 탭 이벤트는 Board까지 오지만, **여기서 한 번도 playMove로 넘어가지 않음**. |
| 2 | (분석 모드가 아님) | playMove 내부 `if (analysisMode == HINT && !isBot) return@launch` | **HINT 모드일 때만** 인간 착수 차단. QUICK/NORMAL이면 통과. |
| 3 | `!ensureActiveGameTokens(snapshot).first` | playMove 내 토큰 검사 | **토큰 부족이면** 낙관적 수·돌 복원 후 **suspendUi = false**로 되돌리고 return. 돌은 안 올라감. **분석/대국 구분 없이 동일**. |
| 4 | `!ensureMoveIsLegal(...)` | playMove 합법성 검사 | 합법이 아니면 역시 복원 후 return. (엔진 미준비면 `ensureMoveIsLegal`이 true를 반환해 통과.) |

즉, **실제로 “착수가 안 된다”**로 이어지는 건 다음 둘이다.

- **`suspendUiDuringAnalysis == true`**  
  - 한 번 true가 되면, **그걸 false로 되돌리는 경로**가 결과 처리·revert·타임아웃 등 제한적이라, 특정 시나리오에서는 계속 true로 남을 수 있음.
- **토큰 부족**  
  - `ensureActiveGameTokens`가 false를 주면 **항상 착수 revert**. 분석 모드에서도 대국 모드와 동일한 토큰 검사가 적용됨.

### 2.2 대국 모드 (PlayScreen → PlayViewModel.onBoardCellTapped → playMove)

| 단계 | 조건 | 위치 | 설명 |
|------|------|------|------|
| 1 | `!_isActive.value || !session.started` | `onBoardCellTapped` | 세션이 시작 전이거나 비활성이면 return. |
| 2 | `state.overlay.suspendUiDuringAnalysis` | 위와 동일 | **분석 모드와 같은 overlay 상태**를 참조. suspendUi가 true면 착수 진입 자체가 막힘. |
| 3 | `!isHumanTurn` | 턴 체크 | 봇 턴이면 return. |
| 4 | (이후 동일) | playMove | 토큰·합법성 검사는 분석 모드와 동일. |

→ **대국 모드가 “실행이 안 된다”**는,  
- 세션/활성 플래그 문제이거나,  
- **동일한 `suspendUiDuringAnalysis`**로 탭이 막히거나,  
- **동일한 토큰 검사**로 revert되는 경우로 같은 원인일 가능성이 크다.

---

## 3. “나갔다 들어오면 추천은 나오는데 착수만 안 됨”이 나오는 이유

- **나갔다 들어올 때**  
  - 세션 전환으로 **다른 세션 상태를 저장했다가, 다시 ANALYSIS 세션을 복원**한다.  
  - `EngineSessionOrchestrator.switchSession` → `applySessionState(currentUiState, restored)`  
  - **복원되는 것**: `SessionState(game, analysis, overlay, ...)`  
  - 즉 **`overlay` 전체**(`suspendUiDuringAnalysis` 포함)가 **저장된 그대로** 다시 들어온다.

- **추천이 나오는 이유**  
  - 세션 복원 직후 `updateAnalysisIntent(SET_ACTIVE_SESSION)`으로 **새 분석**이 한 번 더 돌고,  
  - 그 결과가 epoch/수순 게이트를 통과하면 `handleAnalysisResult`에서 **topSuggestions / isInitialAnalysisReady**가 갱신된다.  
  - 그래서 “다시 들어왔을 때는 추천이 보인다”.

- **착수만 안 되는 이유**  
  - 복원된 **overlay**에 **`suspendUiDuringAnalysis == true`**가 들어 있을 수 있다.  
  - 예: 이전에 분석 탭에서 한 번 탭해서 playMove가 시작되었고, 그때 suspendUi가 true가 된 뒤, **결과가 오기 전에** 다른 탭으로 나갔을 때.  
  - 그 상태가 **세션 캡처**에 포함되어 저장되고, 다시 ANALYSIS로 돌아오면 **같은 overlay가 그대로 복원**된다.  
  - `runAnalysisInternal` 시작 시점에 `suspendUiDuringAnalysis = false`로 한 번 덮어쓰긴 하지만, **그게 실행되기 전**에 사용자가 탭하면 여전히 (복원된) true라서 **onCellClicked에서 return**된다.  
  - 더 근본적으로는, **“추천을 그리는데 쓰는 상태”**와 **“탭을 막는 상태”**가 **서로 다른 플래그/경로**라서,  
    - 추천: `analysis.topSuggestions`, `analysis.isInitialAnalysisReady` (결과 콜백으로만 갱신)  
    - 착수 차단: `overlay.suspendUiDuringAnalysis` (playMove·결과·revert·타임아웃 등 여러 경로에서만 해제)  
  - **세션 복원은 overlay를 그대로 가져오므로**, “추천은 최신 분석으로 채워지는데, suspendUi만 예전에 true로 남아 있는” 조합이 가능하다.

→ 따라서 **“다른 모드 나갔다가 다시 분석 모드로 들어오면 추천수는 표시되는데 착수만 안 된다”**는,  
- **복원된 overlay에 suspendUiDuringAnalysis == true가 섞여 들어오는 것**과  
- **탭 허용 여부와 실제 처리(onCellClicked/playMove)가 서로 다른 상태에 의존하는 구조**  
가 합쳐진 결과로 설명된다.

---

## 4. 상태가 “굉장히 꼬여 있다”고 보는 근거

1. **같은 화면인데 “보이는 것”과 “동작”이 다른 상태에 의존**  
   - 추천 표시: `analysis.topSuggestions`, `isInitialAnalysisReady`  
   - 착수 허용(보드 쪽): `tapAllowed` = f(isPreparing, isModelReady, canShowSuggestions, variationPopupState, topSuggestions, isInitialAnalysisReady)  
   - 착수 실제 처리: `onCellClicked`에서 **가장 먼저** `overlay.suspendUiDuringAnalysis`만 보고 return  
   - 그래서 **추천은 있어도**(tapAllowed가 true여도) **suspendUi가 true면 탭은 무시**되는 불일치가 생김.

2. **suspendUiDuringAnalysis의 “해제”가 한 곳이 아님**  
   - playMove 성공 경로, playMove revert(토큰/합법성 실패), handleAnalysisResult 여러 분기, shouldProcessResult 실패 시, 타임아웃 job 등 **여러 경로**에서만 false로 되돌림.  
   - **한 번 true가 된 뒤** 그 경로들이 한 번도 타지 않으면(결과 미수신, 예외, epoch 폐기 등) **계속 true로 남을 수 있음**.

3. **세션 복원이 “전체 overlay”를 그대로 가져옴**  
   - `captureSessionState` / `applySessionState`가 **overlay 전체**를 저장·복원하므로,  
   - “이전에 착수 중이던” suspendUi가 **다른 탭으로 나갔다 들어와도 그대로 유지**되고,  
   - 새로 도는 분석 결과만 반영되면 **추천은 보이지만 탭은 막힌** 상태가 재현된다.

4. **분석 모드와 대국 모드가 같은 엔진/ViewModel 상태를 공유**  
   - `analyzingSession`, `_uiState`(game, analysis, overlay, tokenState 등)를 같이 쓰므로,  
   - 한쪽에서 꼬인 overlay/토큰 상태가 **다른 모드에서도 동일하게** 착수 불가로 이어질 수 있음.

5. **엔진 “실행”과 “착수/추천”이 한 흐름이 아님**  
   - “엔진이 제대로 실행이 안 되는 것 같다”는 체감은,  
   - 실제로는 **엔진은 돌고 결과도 오는데**(다시 들어오면 추천이 보이므로),  
   - **그 결과를 써서 갱신하는 쪽**(epoch, 수순, empty 처리)과 **탭을 허용하는 쪽**(suspendUi, 토큰)이 **따로따로**라서,  
   - “실행이 안 된다”가 “추천이 안 나온다” + “착수도 안 된다”로 동시에 느껴지는 구조다.

---

## 5. 요약 표

| 현상 | 가능 원인 (코드 경로 기준) |
|------|----------------------------|
| **추천수 안 나옴 (첫 진입)** | 엔진 미준비로 분석 미실행, epoch/수순 게이트로 결과 폐기, totalVisits&lt;2 중간만 오고 스킵, 수순 불일치로 추천만 비움. |
| **착수 안 됨 (분석)** | onCellClicked에서 `suspendUiDuringAnalysis == true`로 return, 또는 playMove 내 토큰/합법성 실패로 revert. |
| **착수 안 됨 (대국)** | 동일한 suspendUi, 또는 세션 미시작/비활성, 봇 턴, 동일 토큰 검사. |
| **나갔다 들어오면 추천만 나오고 착수는 안 됨** | 세션 복원 시 **overlay(및 suspendUi)가 예전 값으로 복원**되고, 새 분석만 반영되어 추천만 채워짐. 탭은 여전히 suspendUi로 막힘. |
| **전체가 “엔진이 안 돼는 것 같다”** | 엔진 실행·결과 수신·추천 갱신·탭 허용이 **서로 다른 상태/경로**에 의존해, 한쪽만 꼬여도 “안 되는 것처럼” 보이는 구조적 얽힘. |

---

**이 문서는 원인 분석과 설명만 담으며, 수정 제안이나 땜빵 패치는 포함하지 않는다.**
