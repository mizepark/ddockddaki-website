# 분석 모드 깜박임·착수 불가 / 대국·분석 소리·진동 원인 정리

## 1. 분석 모드 “착수가 아예 안 됨”의 근본 원인

### 1.1 착수 허용 조건 (BoardSection.kt)

분석 탭의 바둑판 터치 허용 여부는 아래 한 줄로 결정됩니다.

```kotlin
// BoardSection.kt:476
val tapAllowed = uiState.isModelReady
    && (!uiState.overlay.suspendUiDuringAnalysis || uiState.overlay.variationPopupState != null)
    && !isPreparing
```

즉, **분석 모드에서는 `suspendUiDuringAnalysis == true`인 동안에는 참고도 팝업이 없을 때 착수가 항상 막힙니다.**

### 1.2 suspendUiDuringAnalysis가 true로 설정되는 시점

- **한 곳에서만 true로 설정됨**: `EngineViewModel.triggerAnalysis()` 진입 시 (약 2156~2162줄)
  - 분석이 **시작될 때마다** `overlay.suspendUiDuringAnalysis = (sessionSnapshot != Session.PLAY)` 로 설정
  - 분석 세션(ANALYSIS)이면 **무조건 true**

### 1.3 suspendUiDuringAnalysis가 false로 되돌아가는 시점

- **오직 `handleAnalysisResult()` 안에서만** false로 복구됨 (2383, 2434줄 등)
  - 중간/최종 분석 결과를 적용할 때 `overlay.copy(suspendUiDuringAnalysis = false)` 로 업데이트

### 1.4 왜 “한 번 막히면 계속 막히는가”

다음 두 경우에는 **`handleAnalysisResult`가 호출되지 않거나, 호출돼도 overlay를 갱신하지 못합니다.**

1. **분석 중 예외 발생** (엔진 타임아웃, 크래시, IPC 오류 등)
   - `triggerAnalysis` 내부 `catch (t: Throwable)` (2199~2203)에서:
     - `statusMessage`만 갱신하고
     - **`suspendUiDuringAnalysis`는 전혀 건드리지 않음**
   - `finally`에서는 `pendingAnalysis = false`만 하고, overlay는 그대로
   - → **한 번 분석이 실패하면 suspendUiDuringAnalysis가 영원히 true로 남아 착수 불가**

2. **분석 결과가 AnalysisResultGate에서 스킵되는 경우**
   - `handleAnalysisResult` 진입 직후 `AnalysisResultGate.shouldProcessResult()`가 false를 반환하면:
     - requestId 불일치(새 분석이 이미 시작됨)
     - boardSize 불일치
     - 중간 결과인데 이미 “분석 중”이 아님 등
   - 이때는 **그냥 return만 하고** overlay를 수정하지 않음
   - → **이전 분석 시작 시 설정된 suspendUiDuringAnalysis = true가 해제되지 않아 착수 불가 유지**

정리하면, **“분석 모드가 갑자기 착수도 안 된다”의 근본 원인은 “분석 시작 시에는 항상 UI를 잠그는데, 분석 실패나 결과 스킵 시에는 그 잠금을 풀어주는 코드가 실행되지 않는다”**입니다.

---

## 2. 분석 모드 “깜박임”의 근본 원인

- 분석이 **시작될 때마다** `suspendUiDuringAnalysis = true` → tapAllowed가 false, 추천/오버레이도 비활성
- **중간/최종 결과가 올 때마다** `handleAnalysisResult`에서 `suspendUiDuringAnalysis = false` → 다시 터치·추천 활성화
- QUICK/일반 분석처럼 분석이 자주 반복 트리거되면:
  - true → false → (다음 분석 시작) → true → false → …
- 이때마다 `tapAllowed`, 추천 수, 오버레이 상태가 바뀌면서 **화면이 계속 갱신되어 깜박이는 것처럼** 보입니다.
- 또한 분석이 실패하거나 결과가 스킵되면, 위 1.4처럼 true에서 멈춰 있어서 “계속 비활성”처럼 보일 수도 있습니다.

---

## 3. 대국 모드 “소리·진동 안 남”의 가능 원인

- **소리**: PlayBoardSection에서는 `state.analysis.moveSoundNonce`가 바뀔 때만 소리를 냄. 이 state는 PlayViewModel이 `analysisFacade.uiState`를 **33ms 샘플링**한 값. 샘플링/전달 타이밍에 따라 nonce가 한 프레임만 올라갔다가 바로 다음 상태로 덮이면, “소리만 트리거하는 변경”이 UI에 반영되지 않아 소리가 안 날 수 있음.
- **진동**: Board.kt의 착수 확정 시점(onTap 호출 직전)에서만 진동 발생. 대국 모드에서도 **tapAllowed가 true일 때만** onTap이 호출됨. 대국 모드의 tapAllowed는 “준비 중이 아님, 참고도 팝업 없음” 등인데, 이 조건이 자주 false이거나, 세션/상태가 꼬여서 탭이 무시되면 진동도 나지 않음.

즉, “대국 모드에서 소리·진동이 안 난다”는 것은  
- **소리**: moveSoundNonce 기반 트리거가 샘플링/상태 구조 때문에 한 번씩 누락되는 상황  
- **진동**: 탭이 실제로 허용되지 않거나(onTap까지 도달하지 않거나), 진동 코드 경로가 다른 이유로 실행되지 않는 상황  
과 연결됩니다.

---

## 4. 분석 모드 “소리는 나는데 진동은 안 남”의 근본 원인

- **소리**: BoardSection에서는 **moveCount** 기반 `LaunchedEffect(moveCount)`로 착수 시 소리 재생. moveCount만 증가하면 소리는 남.
- **진동**: Board.kt에서 **onTap(coord)** 가 호출될 때만 착수 확정 진동 발생.  
  분석 모드에서는 **tapAllowed가 false이면** `onTap` 람다 안에서 `if (!tapAllowed) return@Board`로 바로 나가므로, **진동 코드까지 도달하지 않음**.
- 분석 모드에서 tapAllowed가 false인 시간이 긴 이유는 위 1~2와 동일:
  - 분석이 시작될 때마다 `suspendUiDuringAnalysis = true`이고,
  - 결과가 오거나 실패/스킵 시 overlay를 풀어주지 않으면 계속 true로 남음.
- 따라서 “착수가 안 되는” 구간과 “진동이 안 나는” 구간이 같고, **분석 모드에서 진동이 안 나는 이유는 “대부분의 시간 동안 tapAllowed가 false라서 onTap이 호출되지 않기 때문”**으로 설명할 수 있습니다.

---

## 5. 요약 표

| 현상 | 근본 원인 (요약) |
|------|------------------|
| 분석 모드 착수 안 됨 | 분석 시작 시 `suspendUiDuringAnalysis = true`로 고정되는데, **분석 실패 시 catch에서, 결과 스킵 시 AnalysisResultGate return 시** 이 값을 false로 되돌리는 처리가 없어서 UI가 영구 잠김. |
| 분석 모드 깜박임 | 분석 시작 시 true → 결과 수신 시 false를 반복하면서 tapAllowed·추천·오버레이가 자주 바뀜. 분석이 자주 트리거될수록 깜박임. |
| 대국 모드 소리/진동 안 남 | 소리: moveSoundNonce가 샘플링/상태 구조 때문에 UI에 반영되지 않거나 조건 미충족. 진동: tap이 허용되지 않거나 onTap까지 도달하지 않음. |
| 분석 모드 소리만 나고 진동 안 남 | 소리는 moveCount 기반이라 착수만 반영되면 재생됨. 진동은 onTap 호출 시에만 발생하는데, 분석 모드에서는 suspendUiDuringAnalysis 때문에 tapAllowed가 대부분 false라 onTap이 호출되지 않음. |

---

## 6. 수정 시 권장 방향 (참고)

- **분석 실패/스킵 시에도 UI 잠금 해제**:  
  `triggerAnalysis`의 catch 블록, 그리고 `handleAnalysisResult`에서 `AnalysisResultGate`로 return하기 **직전**에  
  `_uiState.update { it.copy(overlay = it.overlay.copy(suspendUiDuringAnalysis = false)) }` 를 호출하거나,  
  분석 job이 끝날 때(finally 또는 취소/완료 공통 경로) 한 번만 `suspendUiDuringAnalysis = false`로 되돌리도록 보장.
- **깜박임 완화**:  
  분석 모드에서 “착수 허용”을 분석 시작 시 무조건 막지 않도록 하거나,  
  `suspendUiDuringAnalysis`를 쓰는 조건을 완화(예: QUICK에서는 true로 두지 않기, 또는 중간 결과가 올 때만 짧게 true)하는 방안 검토.
- **대국 모드 소리**:  
  moveSoundNonce 변경을 33ms 샘플과 분리해 구독하거나, nonce만 별도 Flow로 노출해 소리 LaunchedEffect가 놓치지 않도록 하는 방안 검토.
- **진동**:  
  분석 모드에서도 “탭은 막되 진동만 준다”가 필요하면, tapAllowed와 무관하게 “좌표 터치” 시점에 진동을 주는 별도 경로를 두는 방안 검토.

이 문서는 **원인 파악 및 설명만** 담고 있으며, 실제 코드 수정은 별도 작업으로 진행하면 됩니다.
