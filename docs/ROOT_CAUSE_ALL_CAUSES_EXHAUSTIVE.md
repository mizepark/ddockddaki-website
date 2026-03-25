# 추천수 미표시·착수 불가·대국 모드 미실행 — 원인 전부 (완전 나열)

**목적**: 수정 없이, 가능한 **모든** 원인을 코드 경로·조건별로 빠짐없이 나열한다.  
**구성**: (A) 추천수 안 나옴 (B) 착수 안 됨·분석 모드 (C) 착수 안 됨·대국 모드 (D) 대국 모드 실행 안 됨 (E) 나갔다 들어오면 추천만 나오고 착수 안 됨.

---

## A. 추천수가 안 나오는 원인 (전부)

추천수(`topSuggestions`)는 **분석 결과가 `handleAnalysisResult`를 통과해 UI를 갱신할 때만** 바뀐다. 그 전에 막히는 지점을 전부 나열한다.

### A.1 분석이 아예 시작되지 않음

| # | 원인 | 파일·위치 | 조건·설명 |
|---|------|-----------|-----------|
| A.1.1 | `updateAnalysisIntent`에서 의도 제출 자체를 스킵 | EngineViewModel.kt `updateAnalysisIntent` | `if (!snapshot.isModelReady \|\| analyzingSession == Session.OFFLINE) return` → 모델 미준비 또는 오프라인 세션이면 의도 제출 안 함. |
| A.1.2 | `runAnalysisInternal` 진입 직후 launch 스킵 | EngineViewModel.kt `runAnalysisInternal` 내부 | `if (!snapshot.isModelReady) return@launch` → 스냅샷 시점에 모델 미준비면 분석 job 자체를 안 돌림. |
| A.1.3 | 오프라인 세션에서 분석 job 스킵 | EngineViewModel.kt `runAnalysisInternal` | `if (analyzingSession == Session.OFFLINE) { pendingAnalysis = false; return@launch }` → 오프라인일 때 분석 미실행. |
| A.1.4 | 세션 전환 후 `isModelReady`가 base(이전 세션) 값 | EngineViewModel `onSessionSwitched` → `updateAnalysisIntent` | `_uiState.value = result.newUiState` 후 바로 `updateAnalysisIntent` 호출. `newUiState`의 isModelReady는 `applySessionState(base, restored)`에서 **base** 유지. PLAY→ANALYSIS 전환이면 PLAY 시점 isModelReady 사용 → 타이밍에 따라 한 번 스킵 가능. |

### A.2 분석은 돌지만 결과 콜백이 호출되지 않음

| # | 원인 | 파일·위치 | 조건·설명 |
|---|------|-----------|-----------|
| A.2.1 | 분석 job이 취소됨 | EngineOrchestrator.kt | `EngineEvent.SwitchSession` 시 `analysisJob?.cancel()` → 탭 전환하면 진행 중 분석 job 취소. 결과는 오지만 적용 경로(아래 게이트)에서 폐기될 수 있음. |
| A.2.2 | 원격 호출 실패로 빈/예외 반환 | EngineRepository.kt, KataGoService | `service.analyzeTop4` → DeadObject/RemoteException 시 빈 결과 반환. `analyzeTop4 returned null` 등. |
| A.2.3 | 네이티브 예외·크래시 | NativeBridge / katago_jni | JNI 내부 예외 시 결과가 안 오거나 빈 배열. |
| A.2.4 | 분석 예외로 catch 블록만 탐 | EngineViewModel.kt 분석 job catch | `catch (t: Throwable)` 시 `handleAnalysisResult`가 아닌 에러 메시지·suspendUi 해제만 수행. 추천 갱신 없음. |

### A.3 결과는 오지만 `handleAnalysisResult` 진입 전에 폐기됨

| # | 원인 | 파일·위치 | 조건·설명 |
|---|------|-----------|-----------|
| A.3.1 | `AnalysisResultGate.shouldProcessResult` == false | EngineViewModel.kt `handleAnalysisResult` | `AnalysisSnapshotValidator.shouldProcessAnalysisResult` 아래 4가지 중 하나만 만족해도 false → **아무 UI 갱신 없이** return (suspendUi만 false로 해제). |

### A.4 `AnalysisSnapshotValidator.shouldProcessAnalysisResult`가 false를 반환하는 경우 (4가지)

| # | 원인 | AnalysisSnapshotValidator.kt | 조건·설명 |
|---|------|-----------------------------|-----------|
| A.4.1 | 중간 결과인데 현재 "분석 중"이 아님 | L19 | `if (isIntermediate && !isCurrentlyAnalyzing) return false` → 중간 콜백이 왔는데 이미 status가 ANALYZING이 아니면 폐기. |
| A.4.2 | requestId 불일치 | L20 | `if (resultAnalyzedRequestId != lastAnalysisRequestId) return false` → 다른 요청의 결과로 간주해 폐기. |
| A.4.3 | 보드 크기 불일치 | L21 | `if (currentBoardSize != analysisBoardSizeSnapshot) return false` → 9/13/19 등 크기 달라지면 폐기. |
| A.4.4 | 이전 세대 결과(epoch 폐기) | L22 | `if (runEpoch < lastSubmittedEpoch) return false` → 의도 제출 시마다 `currentEpoch++` 하므로, 그 사이에 온 결과는 모두 폐기. |

### A.5 `handleAnalysisResult` 진입 후 return으로 추천 갱신 없음 (또는 추천 비움)

| # | 원인 | 파일·위치 | 조건·설명 |
|---|------|-----------|-----------|
| A.5.1 | 중간 결과 + totalVisits &lt; 2 스킵 | EngineViewModel.kt L2097 | `if (isIntermediate && result.totalVisits < 2) return` → 아무 상태 갱신 없이 return. topSuggestions/isInitialAnalysisReady 갱신 없음. |
| A.5.2 | 수순 해시 불일치 | L2103–2113 | `result.analyzedMovesHash != computeMovesHash(current)` → **topSuggestions 비우기** + isInitialAnalysisReady = false 후 return. |
| A.5.3 | 수순 길이 불일치 | L2114–2126 | `analyzed.size != current.size` → 동일하게 추천 비우기 후 return. |
| A.5.4 | 타임아웃 결과 | L2128–2144 | `result.timedOut == true` → 추천 갱신 없이 status/메시지만 바꾸고 return. |
| A.5.5 | 최종 결과인데 suggestions 비어 있음 | L2146–2181 | `!isIntermediate && result.suggestions.isEmpty()` → 추천 갱신 없이 empty 재시도 로직만 타고 return(또는 재시도 후 동일). |

→ **정리**: 추천이 안 나오려면 위 A.1~A.5 중 **하나라도** 만족하면 된다. 여러 개가 동시에 만족할 수도 있다.

---

## B. 착수가 안 되는 원인 — 분석 모드 (전부)

착수 경로: **MainScreen → Board(탭) → callbacks.onCellClicked(coord) → EngineViewModel.onCellClicked(coord) → playMove(coord)**.  
그 전·중간에 **return 또는 revert**가 나오는 지점 전부.

### B.1 보드에서 탭이 “허용”되지 않음 (tapAllowed == false)

| # | 원인 | BoardSection.kt | 조건·설명 |
|---|------|-----------------|-----------|
| B.1.1 | isPreparing == true | L475 | `isPreparing = (isVariationAnalysisMode \|\| isHintMode) && !isInitialAnalysisReady` → 참고도/힌트 모드이고 아직 분석 준비 안 됐으면 true → tapAllowed false. |
| B.1.2 | isModelReady && 아무 조건도 불만족 | L479–483 | `tapAllowed = !isPreparing && ( isModelReady && (canShowSuggestions \|\| variationPopupState != null \|\| (topSuggestions.notEmpty && isInitialAnalysisReady)) \|\| !isModelReady )` → isModelReady인데 canShowSuggestions·팝업·추천+ready 모두 false면 tapAllowed false. |
| B.1.3 | canShowSuggestions == false | L476–477 | `canShowSuggestions = !suspendUiDuringAnalysis && (...)` → **suspendUiDuringAnalysis == true**이면 canShowSuggestions false. (단, topSuggestions.notEmpty && isInitialAnalysisReady면 tapAllowed는 true가 될 수 있음.) |

### B.2 onCellClicked에서 playMove 호출 전에 return

| # | 원인 | EngineViewModel.kt `onCellClicked` | 조건·설명 |
|---|------|-----------------------------------|-----------|
| B.2.1 | 참고도 팝업 열림 | L977–980 | `variationPopupState != null` → stopVariation()만 하고 return. 착수 아님. |
| B.2.2 | **suspendUiDuringAnalysis == true** | L981 | **참이면 무조건 return.** playMove로 안 감. (가장 흔한 “착수 안 됨” 원인 후보.) |
| B.2.3 | 추천수 클릭 + (참고도 모드 또는 힌트 모드) | L984–991 | `suggestion != null && (isVariationAnalysisMode \|\| analysisMode == HINT)` → playVariation만 하고 return. |
| B.2.4 | 힌트 모드 + suggestion != null + pv 비어 있음 | L996–997 | `analysisMode == HINT`이고 suggestion은 있지만 pv 없으면 return. |
| B.2.5 | 참고도 분석 모드 + 추천수 외 좌표 | L999–1002 | `isVariationAnalysisMode` → 일반 착수 차단, return. |

### B.3 playMove 내부에서 return 또는 revert (돌이 안 올라감)

| # | 원인 | EngineViewModel.kt `playMove` | 조건·설명 |
|---|------|-------------------------------|-----------|
| B.3.1 | 힌트 모드 + 인간 턴 | L1012 | `if (analysisMode == AnalysisMode.HINT && !isBot) return@launch` → 힌트 모드에서는 인간 착수 자체를 막음. |
| B.3.2 | 참고도 팝업 열림 | L1015–1018 | `variationPopupState != null` → stopVariation() 후 계속 진행할 수 있으나, 이후 검사에서 막힐 수 있음. |
| B.3.3 | 좌표 범위 밖 | L1020–1027 | `coord.x/y !in 0 until boardSize` → 에러 메시지만 갱신 후 return. |
| B.3.4 | 이미 돌이 있는 좌표 | L1029–1036 | `snapshot.game.stones.any { it.coord == coord }` → 동일. |
| B.3.5 | **토큰 부족** | L1062–1094 | `ensureActiveGameTokens(snapshot).first == false` → 낙관적 수·돌 복원, suspendUi = false, return. **분석/대국 구분 없이 동일.** |
| B.3.6 | **합법성 검사 실패** | L1099–1142 | `!ensureMoveIsLegal(...)` → 동일하게 복원 후 return. (단, `!snapshot.isModelReady`이면 ensureMoveIsLegal이 true 반환 → 통과.) |
| B.3.7 | deviation charge 시 토큰 부족 | L1146–1188 | playMove 중 chargeNeeded 시 `ensureActiveGameTokens` 재호출, false면 복원 후 return. |

---

## C. 착수가 안 되는 원인 — 대국 모드 (전부)

경로: **PlayScreen → Board(탭) → playViewModel.onBoardCellTapped(coord) → analysisFacade.playMove(coord)**.  
PlayViewModel.onBoardCellTapped·playMove 쪽에서 막히는 지점 전부.

### C.1 onBoardCellTapped에서 playMove 호출 전에 return

| # | 원인 | PlayViewModel.kt `onBoardCellTapped` | 조건·설명 |
|---|------|--------------------------------------|-----------|
| C.1.1 | **화면 비활성 또는 세션 미시작** | L726 | `if (!_isActive.value \|\| !session.started) return` → 대국 탭이 비활성이거나 “대국 시작” 전이면 막힘. |
| C.1.2 | **suspendUiDuringAnalysis == true** | L729 | 분석 모드와 **동일한 overlay** 참조. true면 return. |
| C.1.3 | 참고도 팝업 열림 | L732–735 | stopVariation 후 return. |
| C.1.4 | 힌트 모드 + 준비됨 + 비추천 좌표 | L741–744 | `_showHintPopup.value = true` 후 return. |
| C.1.5 | 힌트 모드 + 추천 좌표 + PV 있음 | L746–750 | playVariation 후 return. |
| C.1.6 | 참고도 분석 모드 | L754–757 | 참고도만 처리하고 return. |
| C.1.7 | **봇 턴** | L759–763 | `!isHumanTurn` → return. 인간 턴일 때만 playMove 호출. |

### C.2 playMove 내부 (EngineViewModel 공유)

B.3과 동일: 토큰 부족(B.3.5), 합법성 실패(B.3.6), deviation charge 토큰(C.1과 별개로 B.3.7), 좌표/중복 돌 등.

---

## D. 대국 모드 “실행이 안 된다”로 보이는 원인 (전부)

“실행이 안 된다” = 대국 시작·착수·봇 응답이 안 되는 느낌일 때, 가능한 원인 전부.

| # | 원인 | 관련 위치 | 설명 |
|---|------|-----------|------|
| D.1 | 세션 미시작 (`session.started == false`) | PlayViewModel, startGame 등 | “대국 시작”을 누르기 전이거나, 시작 플로우가 완료되지 않음. |
| D.2 | 화면 비활성 (`_isActive.value == false`) | PlayViewModel `_isActive` | 대국 탭이 포커스가 아니거나 활성 플래그가 false. |
| D.3 | suspendUiDuringAnalysis == true | 동일 overlay | 분석 모드와 같은 상태. 탭이 onBoardCellTapped에서 막힘. |
| D.4 | 봇 턴에서 인간이 탭 | onBoardCellTapped | 인간 턴이 아니면 착수 진입 자체를 안 함. |
| D.5 | 토큰 부족 | playMove 내 ensureActiveGameTokens | 착수 시도 시마다 revert. |
| D.6 | isBotMoveReady == false로 봇이 안 둠 | PlayViewModel isBotMoveReady | `!isModelReady \|\| !isEngineReady \|\| isEngineInitializing \|\| isVariationAnalysisMode \|\| analysisMode == HINT \|\| variationPopupState != null` 등이면 false → 봇 턴이 와도 봇이 수를 안 둠. |
| D.7 | 엔진/모델 미준비 | isModelReady, isEngineReady | 초기화 전이거나 세션 전환 직후 엔진이 아직 안 올라옴. |

---

## E. “나갔다 들어오면 추천만 나오고 착수만 안 됨” — 관련 원인 전부

이 현상은 **추천 갱신 경로**와 **착수 차단 경로**가 서로 다른 상태에 의존할 때만 나온다.

| # | 원인 | 설명 |
|---|------|------|
| E.1 | **세션 복원 시 overlay를 그대로 복원** | `EngineSessionOrchestrator.switchSession` → `applySessionState(currentUiState, restored)` → `overlay = session.overlay`. 이전에 `suspendUiDuringAnalysis == true`로 나갔으면 그대로 복원됨. |
| E.2 | **복원 직후 분석만 새로 돌고 결과는 반영됨** | `updateAnalysisIntent(SET_ACTIVE_SESSION)`으로 새 분석 실행 → 결과가 epoch/수순 게이트 통과 시 `handleAnalysisResult`에서 **topSuggestions, isInitialAnalysisReady**만 갱신. **overlay는 결과 경로에서 suspendUi = false로 덮어쓰지만**, 그 시점 전에 사용자가 탭하면 아직 복원된 true라서 onCellClicked에서 return. |
| E.3 | **runAnalysisInternal 시작 시 suspendUi = false** | runAnalysisInternal 내부에서 `overlay.copy(suspendUiDuringAnalysis = false)` 설정. 다만 **비동기로** 분석이 시작되므로, 복원 직후 곧바로 탭하면 아직 이 업데이트가 적용되기 전일 수 있음. |
| E.4 | **tapAllowed와 onCellClicked 조건 불일치** | tapAllowed는 `topSuggestions.notEmpty && isInitialAnalysisReady`만 만족해도 true 가능. 반면 onCellClicked는 **가장 먼저** `suspendUiDuringAnalysis`만 보고 true면 return. 그래서 “추천은 보이는데(탭 허용처럼 보이는데) 탭하면 무시”되는 조합 가능. |

---

## F. suspendUiDuringAnalysis가 true로 “고정”될 수 있는 경로 (해제 안 되는 경우)

한 번 true가 된 뒤, 아래 **해제 경로**를 한 번도 타지 않으면 계속 true로 남을 수 있음.

| # | true가 되는 곳 | false로 되돌리는 곳 (해제 경로) |
|---|----------------|--------------------------------|
| - | playMove 시작 시 (낙관적 업데이트 직후) | playMove 성공 시(상태 갱신 시), 토큰 실패 revert, 합법성 실패 revert, deviation charge 실패 revert, handleAnalysisResult 여러 분기(shouldProcess false, 수순 불일치, 타임아웃, empty, 중간/최종 갱신), 분석 job catch 블록, **타임아웃 job(20초)** |

즉, **결과가 안 오거나**(엔진/서비스/네이티브 문제), **결과가 게이트에서 다 폐기**되면, playMove 성공 경로와 handleAnalysisResult 내부 해제 경로를 타지 않아 **suspendUi가 true로 남을 수 있음**. 세션 복원 시 이 값이 그대로 복원되면 E.1·E.2와 결합됨.

---

## G. 요약 — 원인 개수

- **추천수 안 나옴**: A.1~A.5까지 **20개 이상**의 서로 다른 조건(분석 미시작, 콜백 미호출, 게이트 4종, handleAnalysisResult 내 return 5종 등).
- **착수 안 됨(분석)**: B.1~B.3에서 **10개 이상** (tapAllowed 3종, onCellClicked 5종, playMove 내 7종).
- **착수 안 됨(대국)**: C.1·C.2 + D **10개 이상**.
- **나갔다 들어오면 추천만 나오고 착수 안 됨**: E.1~E.4 + F와 결합.

**여러 원인이 동시에 만족될 수 있고**, “추천 안 나옴”과 “착수 안 됨”은 **서로 다른 상태·경로**에 의해 발생하므로, **원인이 여러 개**인 것이 정상이다.

---

**이 문서는 원인 나열과 설명만 담으며, 수정 제안은 포함하지 않는다.**
