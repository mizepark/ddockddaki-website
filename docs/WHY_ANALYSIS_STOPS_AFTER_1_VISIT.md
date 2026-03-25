# "막 시작한 분석이 1방문만 하고 끊기는 경우"가 자주 생기는 원인

---

## 1. 끊기는 메커니즘 (직접 원인)

`triggerAnalysis(0L, stopBefore = true)` 가 호출될 때마다:

1. **이전 분석 코루틴**을 `analysisJob?.cancel()` 로 취소하고  
2. **새 job**을 띄운 뒤, 그 job 안에서 **맨 처음** `stopAllAnalysis(expectedAnalyzeRequestId)` 를 호출한다.

그래서:

- **이전 분석**은 네이티브에서 `g_stopFlag = true` 로 곧바로 중단되고,
- 아직 1~몇 방문밖에 안 돌았으면 **totalVisits=1 (또는 소수)** 인 부분 결과만 나와서 반환된다.

즉, **“새 분석을 시작할 때마다 이전 분석을 반드시 한 번 멈추는” 구조**라서,  
이전 분석이 **시작한 지 얼마 안 됐을 때** 다음 트리거가 오면 **1방문만 하고 끊겨 나오는 경우**가 생긴다.

---

## 2. 왜 “자주” 생기나 (근본 원인)

**triggerAnalysis 를 부르는 곳이 많고, 호출이 자주 겹치기 때문**이다.

### 2.1 호출처가 많은 것

- **수순이 바뀌는 모든 경로**  
  `playMove`, `undo`, `redo`, `jumpToStart`, `jumpToEnd`, `jumpBack5`, `jumpForward5`  
  → 수순이 바뀔 때마다 대부분 `triggerAnalysis(stopBefore = false)` 또는 `triggerAnalysis(debounceMs = 0L, stopBefore = false)` 로 **즉시 재분석**을 건다.
- **세션/설정**  
  `setActiveSession` (탭 전환 후), `loadSgfContent` 등에서도 `triggerAnalysis(...)` 호출.
- **분석 모드/레이턴시**  
  `setTargetLatency(..., triggerNewAnalysis = true)`, `triggerQuickAnalysis`, `triggerHintAnalysis` 등.
- **분석 결과 처리 쪽**  
  - 빈 최종 결과 재시도: `triggerAnalysis(debounceMs = 0L)`  
  - 해시/수순 **불일치 시**: `triggerAnalysis(debounceMs = 150L)`  
  - 타임아웃 등 다른 경로에서의 재트리거.

그래서 **한 번 국면이 바뀐 뒤**에:

- playMove → triggerAnalysis  
- 그 직후/동시에 undo, redo, jump, 탭 전환, 불일치 재시작 등이 오면  
  **같은 “이전 분석”을 여러 번 연달아 취소**할 수 있고,  
  그때마다 “막 시작한 분석이 1방문만 하고 끊긴” 결과가 나올 수 있다.

### 2.2 debounce 와의 상호작용

- 기본이 `triggerAnalysis(debounceMs = 300L)` 인 경로가 있으면,  
  **300ms 뒤에** `triggerAnalysis(0L)` 이 한 번 더 호출된다.
- 그 300ms 사이에 **다른 이벤트**로 또 `triggerAnalysis(300)` 이 나가면,  
  **300ms 후**에 다시 `triggerAnalysis(0)` 이 실행되면서  
  **그때 막 돌기 시작한 분석**을 취소하게 된다.
- 그래서 “방금 시작한 분석만 1방문 하고 끊기는” 타이밍이 **debounce 주기(300ms, 150ms 등)와 겹치면서 반복**될 수 있다.

### 2.3 불일치 재시작(150ms)

- 해시/수순 불일치 시 `triggerAnalysis(debounceMs = 150L)` 로 **150ms 뒤 재시작**을 건다.
- 그 150ms 동안 돌고 있던 분석은, 150ms가 지나면 **무조건 취소**되고 새 분석이 시작된다.
- 저사양/부하 큰 기기에서는 150ms 안에 **1~수 방문**밖에 못 돌 수 있어,  
  **불일치가 나올 때마다** “1방문만 하고 끊긴 결과”가 한 번씩 나오기 쉽다.

---

## 3. 요약

| 구분 | 내용 |
|------|------|
| **직접 원인** | 새 분석을 시작할 때마다 `stopAllAnalysis` 로 **이전 분석을 반드시 멈추는** 구조. 그래서 “이전 분석”이 방금 시작됐거나 아직 많이 못 돌았을 때 멈추면 **1방문만 하고 끊긴 결과**가 나옴. |
| **근본 원인** | **triggerAnalysis 호출처가 많고**(undo/redo/jump/playMove/탭 전환/불일치 재시작/빈 결과 재시도 등), **호출이 자주 겹치거나** debounce(300ms, 150ms) 뒤에 한 번 더 나가면서, “막 시작한 분석”이 다음 트리거에 의해 곧바로 취소되는 상황이 **반복**됨. |

**한 줄**:  
“막 시작한 분석이 1방문만 하고 끊기는 경우”가 자주 생기는 이유는, **분석을 재시작할 때마다 이전 분석을 무조건 멈추는 설계**에다가, **triggerAnalysis를 부르는 곳이 많고 호출이 자주 겹쳐서** “방금 시작한 분석”이 다음 트리거에 의해 곧바로 stop 되기 때문이다.
