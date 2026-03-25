# 앱 상태 꼬임 근본 원인 파악 (수정 없이 파악만)

다른 오류 수정 과정에서 생긴 것으로 보이는 “앱이 꼬여 있다”는 느낌의 근본 원인을, **수정 제안 없이** 구조와 데이터 흐름 기준으로만 정리한 문서입니다.

---

## 1. 구조 요약: 단일 _uiState + 세션별 캐시

- **EngineViewModel**에는 **하나의 `_uiState`**만 있음.
- 탭(분석/대국/오프라인) 전환 시 `setActiveSession(session)`이 호출되면:
  - **이전 세션**의 현재 `_uiState`를 세션별 캐시에 저장  
    (`analysisSessionState` / `playSessionState` / `offlineSessionState`)
  - **현재 세션**의 캐시(또는 디스크)를 읽어서 `_uiState`를 **완전히 교체**
- 따라서 “지금 보이는 화면”은 항상 **현재 활성 세션 하나**의 상태만 반영하고, 나머지 세션은 **캐시에만** 남는다.

---

## 2. 꼬임 원인 1: suspendUiDuringAnalysis 잠금 해제 경로가 하나뿐

- 분석 모드에서 **착수 허용**은  
  `tapAllowed = ... && (!suspendUiDuringAnalysis || variationPopupState != null) && ...`  
  로 결정됨.  
  즉 **분석 세션에서는 `suspendUiDuringAnalysis == true`인 동안** 참고도 팝업이 없으면 착수가 막힘.

- **true로 설정되는 곳**: `triggerAnalysis()` 진입 시 한 곳  
  → `overlay.suspendUiDuringAnalysis = (sessionSnapshot != Session.PLAY)`  
  (분석 세션이면 항상 true)

- **false로 되돌리는 곳**: **오직 `handleAnalysisResult()` 내부**  
  → 중간/최종 결과를 적용할 때만 `suspendUiDuringAnalysis = false` 로 업데이트

그래서 아래처럼 되면 **“잠금 해제”가 영원히 일어나지 않음**.

- **분석 중 예외** (타임아웃, 엔진 크래시 등):  
  `triggerAnalysis`의 `catch`에서 `statusMessage`만 갱신하고  
  **`suspendUiDuringAnalysis`는 건드리지 않음**  
  → 한 번 실패하면 그 세션의 `_uiState`는 계속 true 유지.

- **분석 결과가 AnalysisResultGate에서 스킵** (requestId/boardSize 불일치 등):  
  `handleAnalysisResult`에서 `shouldProcessResult == false`면  
  **그냥 return만 하고** overlay를 수정하지 않음  
  → 마찬가지로 true 유지.

- **탭 전환으로 분석이 취소된 경우**:  
  `setActiveSession(PLAY)` 등에서 `analysisJob?.cancel()` 하고  
  `lastAnalysisRequestId`를 0으로 리셋함.  
  나중에 (취소가 늦게 적용되면) 분석이 끝나도  
  `reqId != lastAnalysisRequestId` 이라서 **`handleAnalysisResult`를 아예 호출하지 않음**  
  → 그 전에 이미 `suspendUiDuringAnalysis = true` 로 들어간 상태가  
  **캐시에 그대로 저장**되고, 다시 분석 탭으로 돌아오면 그 캐시가 복원되므로  
  “착수 안 됨”이 그대로 유지됨.

정리하면, **“분석 시작 시에는 항상 UI를 잠그는데, 실패/스킵/탭 전환 시에는 그 잠금을 풀어주는 경로가 없다”**는 한 가지 설계가, 분석 모드 착수 불가·깜박임 느낌의 직접 원인이다.

---

## 3. 꼬임 원인 2: “현재 _uiState”와 “이 분석이 속한 세션”이 어긋날 수 있음

- **분석 job**은 `triggerAnalysis()`에서 `viewModelScope.launch(Dispatchers.IO)` 로 시작됨.
- job 안에서는 **시작 시점**의 `sessionSnapshot` / `analyzingSession` / `_uiState.value` 스냅샷만 사용하고,  
  **완료 시점**에 “지금 _uiState가 어떤 세션인지”는 다시 확인하지 않음.
- `handleAnalysisResult`는 **현재 `_uiState`**를 무조건 업데이트함:  
  `_uiState.update { ... }` → “지금 화면에 보이는 세션”에 분석 결과를 씀.

따라서:

- 분석 탭에서 분석을 켠 뒤, **결과가 오기 전에** 대국 탭으로 전환하면  
  `setActiveSession(PLAY)` 가 호출되고,  
  **이미 `_uiState`는 대국 세션 상태로 교체**된 뒤임.
- 그 다음 (취소가 늦게 되면) 분석 job이 끝나서 `handleAnalysisResult`가 호출되면,  
  **현재 _uiState = 대국 세션**이므로,  
  분석 결과(추천, suspendUiDuringAnalysis = false 등)가 **대국 세션 상태에** 들어갈 수 있음.
- 반대로, **분석 세션 쪽**은 그 결과를 받을 기회가 없음  
  (이미 캐시로 밀려났고, `handleAnalysisResult`는 캐시가 아니라 `_uiState`만 갱신).

실제로는 `lastAnalysisRequestId`를 0으로 리셋하기 때문에,  
탭 전환 후 완료되는 분석은 `handleAnalysisResult`가 호출되지 않도록 되어 있어서,  
“분석 결과가 대국 세션에 써진다”는 레이스는 제한적일 수 있음.  
하지만 **“분석 세션 쪽은 결과를 한 번도 못 받고, 캐시에는 suspendUiDuringAnalysis = true 인 채로 남는다”** 구조는 그대로라,  
탭 전환 후 다시 분석 탭으로 돌아오면 꼬인 것처럼 보이는 현상(착수 안 됨, 깜박임)의 배경이 됨.

---

## 4. 꼬임 원인 3: UI 쪽에서 “analysis”라는 이름과 실제 세션의 불일치

- **MainScreenUiState** / **PlayScreenUiState** 모두  
  화면에 보여줄 상태를 담는 필드 이름이 **`analysis: UiState`** 로 같음.
- 분석 탭: `state.analysis` = EngineViewModel의 **분석 세션** UiState (맞음).
- 대국 탭: `state.analysis` = EngineViewModel의 **대국 세션** UiState  
  (PlayViewModel이 `analysisFacade.uiState` → `engineViewModel.uiState` 를 그대로 노출하고,  
  대국 탭일 때 이게 PLAY 세션 상태이기 때문).

즉, **같은 필드 이름 `analysis`가 탭에 따라 서로 다른 세션을 가리킴**.  
이걸 모르고 “analysis = 분석 세션”이라고 가정하고 로직을 넣으면,  
대국 탭에서 잘못된 조건/플래그를 참고하게 되어 “꼬인 것 같다”는 동작이 나올 수 있음.  
(지금도 소리/진동이 “분석만 되고 대국은 안 된다” 등으로 갈라지는 것과 무관하지 않을 수 있음.)

---

## 5. 꼬임 원인 4: 세션 전환 시 “나가면서 풀어주기”가 없음

- **AnalysisUiViewModel.setActive(false)** (분석 탭을 나갈 때):  
  `setActiveSession(...)` 을 호출하지 않음.  
  즉 “지금 분석 세션 상태를 저장하고, 다른 세션으로 전환한다”는 처리는  
  **다음 탭의 setActive(true) → setActiveSession(PLAY/OFFLINE)** 에서만 일어남.
- 따라서 “분석 탭을 나갈 때,  
  현재 분석 세션 상태에 대해 suspendUiDuringAnalysis 를 false 로 만들어서 캐시에 저장한다” 같은  
  **정리 단계가 없음**.  
  그대로 캐시에 들어가기 때문에,  
  “분석 실패/스킵/취소로 한 번도 unlock 이 안 된 상태”가 그대로 보존됨.

---

## 6. 꼬임 원인 5: 소리/진동이 탭·경로마다 다른 이유

- **분석 모드 소리**: BoardSection에서는 **moveCount** 기반 `LaunchedEffect(moveCount)` 로만 소리 재생.  
  → moveCount만 증가하면 소리 나옴. (moveSoundNonce 미사용.)
- **대국 모드 소리**: PlayBoardSection에서는 **moveSoundNonce** 기반으로 재생.  
  → 엔진이 nonce를 올리는 시점과, PlayViewModel이 **33ms 샘플링**한 uiState가 맞지 않으면 소리 누락 가능.
- **진동**: 두 모드 모두 Board.kt의 **onTap(coord)** 호출 시에만 발생.  
  분석 모드에서는 `tapAllowed == false` 인 시간이 길어서  
  (suspendUiDuringAnalysis 때문에) onTap이 호출되지 않으면 진동이 안 남.  
  대국 모드에서도 “tap이 실제로 허용되는지”에 따라 onTap 호출 여부가 갈림.

즉, **같은 “착수”인데**  
- 어떤 state를 쓰는지(analysis vs play),  
- 어떤 플래그로 허용하는지(tapAllowed),  
- 소리는 nonce vs moveCount 중 뭘 쓰는지  

가 탭/화면마다 달라서, “분석은 소리만 나고 진동은 안 나고, 대국은 둘 다 안 난다” 같은 **불균형**이 생길 수 있음.

---

## 7. 요약: “꼬여 있다”고 느껴지는 근본 원인

| 구분 | 내용 |
|------|------|
| **구조** | 단일 `_uiState` + 세션 전환 시 캐시로 스왑. 모든 비동기 작업은 “현재 _uiState”에만 씀. |
| **잠금 해제** | `suspendUiDuringAnalysis`를 false로 되돌리는 경로가 `handleAnalysisResult` 하나뿐. 실패/스킵/탭 전환 시에는 해제되지 않아, 분석 세션 캐시가 “잠긴 채” 저장됨. |
| **탭 전환** | 분석 job 취소 + requestId 리셋으로 “다른 세션에 결과 쓰기”는 막지만, 분석 세션 쪽은 결과를 못 받고 캐시는 갱신되지 않아, 다시 분석 탭으로 오면 이전에 잠긴 상태가 복원됨. |
| **네이밍** | `state.analysis`가 분석 탭에서는 분석 세션, 대국 탭에서는 대국 세션을 가리켜, 같은 이름으로 다른 의미가 혼재함. |
| **소리/진동** | 분석은 moveCount 기반 소리 + tapAllowed에 묶인 진동, 대국은 moveSoundNonce + 샘플링 + tapAllowed. 경로가 나뉘어 있어 한쪽만 동작하거나 둘 다 안 하는 현상이 나올 수 있음. |

**한 문장으로**  
다른 오류 수정 과정에서 생긴 “꼬임”의 근본은,  
**“분석 UI 잠금(suspendUiDuringAnalysis)을 풀 수 있는 경로가 결과 적용 한 곳뿐인데, 실패·스킵·탭 전환 시에는 그 경로가 타지 않아서, 분석 세션 상태가 캐시에 잠긴 채로 남고, 같은 단일 _uiState 구조와 섞여서 복원 시 착수 불가·깜박임·소리/진동 불균형이 나타난다”** 로 정리할 수 있습니다.

이 문서는 **원인 파악과 설명만** 담고 있으며, 수정 방법은 포함하지 않습니다.
