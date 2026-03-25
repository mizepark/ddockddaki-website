# 바둑돌 소리·진동 수정이 원인인지 확인 (수정 없이 확인만)

"바둑돌 진동과 소리 수정하다가 갑자기 이런 현상들이 생겼다"는 말에 맞춰, **소리/진동 쪽에서 건드린 부분만** 추적하고, 그 변경이 **분석 모드 깜박임·착수 불가·대국 소리/진동 안 남**의 원인이 될 수 있는지만 검토했습니다. 코드 수정은 하지 않았습니다.

---

## 1. 소리·진동 수정 시 실제로 건드린 곳

### 1.1 EngineViewModel.kt

- **추가한 것**: `moveSoundNonce` 갱신
  - `playMove` 내부 `_uiState.update`:  
    `moveSoundNonce = if (moveCountIncreased) System.nanoTime() else state.moveSoundNonce`  
    그리고 **기존 그대로**  
    `overlay = state.overlay.copy(suspendUiDuringAnalysis = false)` 유지.
  - `handleAnalysisResult` 중간/최종 결과 적용 시에도  
    `moveSoundNonce = if (moveCountIncreased) ... else ...` **추가**만 했고,  
    같은 블록 안의  
    `overlay = ... .copy(suspendUiDuringAnalysis = false)` 는 **그대로 둔 상태**.

- **제거하거나 바꾼 것**: 없음.  
  - `suspendUiDuringAnalysis`를 건드리는 줄을 지우거나 다른 값으로 바꾼 적 없음.  
  - 예외 catch·게이트 스킵·타임아웃 경로는 **당시에도** overlay 갱신이 없었고, 그 부분은 소리/진동 수정 때 손대지 않음.

**정리**: 소리용으로 넣은 건 `moveSoundNonce` 한 필드 추가뿐이고,  
분석/대국 꼬임의 직접 원인인 “실패·스킵 시 잠금 해제가 없다”는 구조는 **소리 수정 이전부터** 그렇게 되어 있었음.

---

### 1.2 Board.kt (착수 시 진동)

- **변경 내용**  
  - 착수 확정 시:  
    `runCatching { view.performHapticFeedback(CONFIRM 또는 KEYBOARD_TAP) }`  
    뒤에  
    `haptic.performHapticFeedback(TextHandleMove)`  
    한 줄 추가.  
  - 예전:  
    `.onFailure { haptic.performHapticFeedback(HapticFeedbackType.LongPress) }`  
    → 지금:  
    위 실패 처리 제거하고,  
    **항상** `haptic.performHapticFeedback(TextHandleMove)` 호출 후  
    `currentOnTap(coord)` 호출.

- **tap/진동 흐름**  
  - `preview?.let { coord -> ... currentOnTap(coord) }` 순서는 **그대로**.  
  - 진동 두 번(뷰 + Compose) 넣은 것만 추가였고,  
    `tapEnabled` / `onTap` / `currentOnTap` 호출 여부는 **변경 없음**.

**정리**: Board.kt 수정은 “진동 추가”만 있고,  
탭이 아예 안 먹히거나 분석만 착수 안 되는 현상의 **원인으로 보일 만한 변경(탭/조건 분기 제거·변경)** 은 없음.

---

### 1.3 PlayBoardSection.kt (대국 소리)

- **변경 내용**  
  - `LaunchedEffect(moveSoundNonce, shouldPlaySound)` 안 조건에서  
    `currentMoveCount > lastMoveCount` **제거**.  
  - 즉,  
    `soundInitDone && moveSoundNonce > lastMoveSoundNonce && shouldPlaySound`  
    만 보고 소리 재생.  
  - `lastMoveSoundNonce`, `lastMoveCount` 갱신은 그대로.

- **영향**  
  - 대국 화면에서 “소리만” 언제 튀는지 조건이 완화된 것뿐.  
  - `uiState` / `overlay` / `tapAllowed` / `onTap` / Board 쪽 로직은 **전혀 건드리지 않음**.

**정리**: 대국 소리 조건만 완화한 변경이라,  
분석 모드 착수 불가·깜박임·잠금 해제와는 **인과 관계 없음**.

---

### 1.4 BoardSection.kt (분석 탭)

- **소리/진동 수정 당시**  
  - 대화 요약에는 “PlayBoardSection과 BoardSection의 **중복 진동 제거**”만 나옴.  
  - BoardSection은 **Board**를 쓰고, 진동은 Board.kt 안에서만 발생.  
  - 따라서 “중복 제거”는 Board 쪽에서 LongPress → TextHandleMove로 바꾼 것과 연관될 수 있음.  
  - BoardSection에서 **tapAllowed / onTap / suggestions** 를 바꾼 기록은 없음.

- **이후 별도 수정**  
  - “tapAllowed를 추천 표시와 일치”시키는 작업에서  
    `canShowSuggestions` 도입,  
    `tapAllowed`와 `suggestions` 조건을 그에 맞춤.  
  - 이건 **소리/진동 수정과는 다른** “분석 UI 잠금/탭 허용” 보완 작업임.

**정리**:  
- 소리/진동만 손댄 단계에서는 BoardSection의 **탭 허용·추천 표시 로직을 바꾼 적 없음**.  
- 지금 보이는 BoardSection 쪽 동작은 “잠금 해제 + tapAllowed 정렬” 수정의 영향이 크고,  
  “바둑돌 소리·진동 수정”과는 직접 연결되지 않음.

---

## 2. 현상별로 보면

| 현상 | 소리/진동 수정과의 연관 가능성 |
|------|----------------------------------|
| **분석 모드 착수 불가 / 깜박임** | EngineViewModel에서 overlay/suspendUiDuringAnalysis 를 **제거·변경한 부분 없음**. 실패·스킵 시 잠금 해제가 없는 것은 **원래 구조**이고, 소리 수정은 그 경로를 건드리지 않음. → **원인으로 보기 어렵다.** |
| **대국 모드 소리/진동 안 남** | 소리: moveSoundNonce 추가·LaunchedEffect 완화는 “소리 나게 하자”는 방향. (샘플링 등 다른 요인으로 여전히 안 날 수는 있음.) 진동: Board.kt는 진동 **추가**만 함. → **이 수정들이 “갑자기 없앴다”라고 보기는 어렵고**, “원래도 불완전했거나 다른 요인과 겹친다”에 가깝다. |
| **분석 모드 소리는 나는데 진동만 안 남** | 분석 탭은 BoardSection이고, 진동은 Board.kt의 onTap 안에서만 발생. Board.kt는 “진동 추가”만 했고 onTap 호출 순서는 그대로. 진동이 안 나는 이유는 **tapAllowed가 false인 구간이 길어서 onTap이 호출되지 않는 것**이고, 그건 suspendUiDuringAnalysis·tapAllowed 쪽 설계 문제라, **진동 “수정” 자체가 원인이라기보다는** 같은 현상(착수 불가)의 다른 표현으로 보는 게 맞다. |

---

## 3. 결론 (확인만)

- **바둑돌 소리·진동을 수정할 때**  
  - EngineViewModel: `moveSoundNonce` **추가**만 했고,  
    `suspendUiDuringAnalysis` / overlay 를 지우거나 바꾼 적 **없음**.  
  - Board.kt: 진동 **추가** 및 실패 시 LongPress → TextHandleMove 변경만 있고,  
    탭 처리·`currentOnTap` 호출 순서는 **동일**.  
  - PlayBoardSection: 소리 LaunchedEffect 조건 **완화**만 있고,  
    탭/overlay/분석 로직은 **미수정**.  
  - BoardSection: 소리/진동 수정 단계에서는 **tapAllowed·suggestions·onTap** 을 건드리지 않음.

- 따라서  
  - “분석 모드 깜박임·착수 불가”는  
    **소리/진동 수정이 원인이라기보다**,  
    예전부터 있던 “잠금 해제 경로가 결과 적용 한 곳뿐”인 구조와,  
    그걸 보완한 **이후의** 잠금 해제·tapAllowed 정렬 수정이 더 관련 있음.  
  - “대국/분석에서 소리·진동이 안 난다”는 것은  
    진동/소리 **코드 자체를 갑자기 망가뜨린 변경**이라기보다,  
    탭 허용 조건·샘플링·suspendUiDuringAnalysis 꼬임 등 **다른 요인**과 같이 보는 편이 자연스럽습니다.

즉, **확인만 한 결과**,  
지금 보이는 현상들의 **직접적인 원인은 “바둑돌 진동과 소리 수정” 그 자체보다는**,  
잠금 해제가 한 경로뿐이던 점·tapAllowed와 추천 표시 불일치·늦은 분석 결과 처리 같은 **구조/다른 수정**에 더 가깝다고 보는 게 맞습니다.
