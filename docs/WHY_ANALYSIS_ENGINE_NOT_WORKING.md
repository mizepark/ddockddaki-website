# 분석/엔진이 제대로 안 도는 이유

정리·단순화 과정에서 **의도적으로 넣은 제한**이, “한 번만 스킵돼도 재시도 불가”로 이어질 수 있는 경로가 있었고, 그걸 풀었습니다.

---

## 1. 원인 후보 (코드상)

### 1.1 의도 재제출이 아예 막히던 부분 (수정함)

**위치**: `EngineViewModel.updateAnalysisIntent()`

**기존 로직**:
```kotlin
if (retryToken == 0L && tentativeIntent.samePosition(lastEmittedIntent)) return
currentEpoch++
// ...
analysisStateMachine.submitIntent(newIntent)
```

- **의도**: “같은 국면·같은 모드면 의도 중복 제출 방지”
- **실제 효과**:  
  한 번 의도를 **제출만** 하고, 실제 실행은 **Defer** 되거나 오케스트레이터에서 **스킵**되면,  
  그 다음부터는 **같은 국면으로 판단해 `updateAnalysisIntent` 진입 시 바로 return** → **다시는 제출하지 않음** → 분석이 영원히 안 돌 수 있음.

**수정**: 위 `samePosition(lastEmittedIntent)` early return **제거**.  
→ 같은 국면이어도 **항상 epoch 올리고 제출**하도록 바꿈.  
실제 “중복 실행”은 상태 머신/스케줄러(보호 구간, `shouldAccept`)에서만 막고, **재제출 자체는 허용**.

### 1.2 그 외 (구조상 가능한 일)

- **로케일 미설정**  
  `getUserLocale() == null` 이면 `startInitializationSequence()`를 아예 안 부름 → AppStart 없음 → 오케스트레이터가 Idle 유지 → 분석 요청 전부 스킵.
- **초기화 실패**  
  `EngineInitResult.Failure` 면 오케스트레이터가 Ready로 안 넘어감 → StartAnalysis 항상 스킵.
- **세션 전환 직후**  
  전환된 UI 상태에 `isModelReady == false` 가 들어가 있으면, 그 시점에 호출된 `updateAnalysisIntent`가 첫 줄에서 return → 그 순간엔 의도 제출 안 함. (나중에 init 성공 시 `INIT_ENGINE`으로 한 번 더 넣는 건 이미 있음.)
- **결과는 오는데 UI에 안 반영**  
  `AnalysisResultGate`(requestId/epoch/boardSize 불일치), 수순 해시 불일치 등으로 `handleAnalysisResult`에서 결과를 버리면, “분석은 돌았는데 추천/봇이 안 움직이는” 현상이 날 수 있음.

---

## 2. “정리·단순화”와의 관계

- **뷰모델 정리/단순화** 자체가 “착수 조건”을 여러 개 바꾼 건 **아니라**고 보는 게 맞고,  
  실제로 과하게 막힌 건 **“아직 추천 없을 때 탭 무시(4.5)”** 한 군데였고, 그건 이미 제거한 상태.
- 대신 **분석이 아예 안 돌게 만드는** 쪽에는,  
  **“같은 국면이면 의도 재제출 막기”**가 있어서,  
  **한 번만 Defer/스킵돼도 이후 재시도가 불가능**한 구조가 됐을 수 있음.  
  그래서 그 제한만 풀었음 (같은 국면도 다시 제출 가능, 중복 실행은 스케줄러/게이트에서 처리).

---

## 3. 수정 요약

| 위치 | 변경 |
|------|------|
| `updateAnalysisIntent()` | `retryToken == 0L && tentativeIntent.samePosition(lastEmittedIntent)` 일 때 return 하던 로직 **제거** → 같은 국면이라도 epoch 올리고 **항상 제출**. |

이제 “한 번 스킵되면 영원히 분석이 안 돌던” 경로는 제거된 상태입니다.  
그래도 안 되면, 로그에서 **오케스트레이터 상태(Idle/Ready/Analyzing)** 와 **`intent_update` / `triggerAnalysis start`** 가 찍히는지, **init 성공/실패** 여부를 보면 다음 원인 좁히기 좋습니다.
