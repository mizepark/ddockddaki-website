# "방문수 1" 로그가 반복되는 이유 (설명만)

`parseRawResult: ... totalVisits=1` 로그가 device_play_log 등에 계속 찍히는 이유를 코드 흐름 기준으로만 정리한 문서입니다.

---

## 1. totalVisits=1이 나오는 경로

- **로그 위치**: `AnalysisUseCase.kt`의 `parseRawResult()`  
  - `totalVisits` = 네이티브에서 넘어온 `rawResult` 배열의 **마지막 요소** (`out[out.size - 1].toInt()`).
- **진입**: `parseRawResult`는 **분석 결과**를 파싱할 때만 호출됨.  
  - `AnalysisUseCase.analyze()` → 네이티브 `analyzeTop4` 호출 → 반환된 FloatArray로 `parseRawResult()` 호출.  
  - 정책 heatmap(`nativeGetPolicyHeatmap`)은 별도 API라 `parseRawResult`를 타지 않음.

즉, **totalVisits=1 로그는 모두 “메인 분석(analyzeTop4)”이 반환한 결과**에서 나옵니다.

---

## 2. 네이티브에서 totalVisits=1이 나오는 경우

`katago_jni.cpp` 쪽 흐름은 다음과 같습니다.

- **analyzeTop4**  
  - `runWholeSearch(g_stopFlag)` 로 MCTS 검색 실행.  
  - **검색이 끝나면** (정상 종료든 중간에 중단이든) `extractAnalysisData()`로 현재 트리에서 결과를 꺼내서 반환.  
  - 그 배열 마지막에 `totalVisits`(root visits)가 들어감.
- **stopAnalysis**  
  - `g_stopFlag.store(true)` 만 설정.  
  - `runWholeSearch`는 이 플래그를 보면서 **가능한 한 빨리** 검색을 중단하고 반환.

그래서:

- **검색이 시작된 직후** `stopAnalysis`가 호출되면,  
  - 검색은 1~몇 방문만 하고 `g_stopFlag`에 의해 중단되고,  
  - 그 시점의 트리로 `extractAnalysisData`가 호출되어 **totalVisits=1 (또는 매우 작은 값)** 인 결과가 나올 수 있음.
- 주석에도  
  `"search stops at maxPlayouts (e.g. 1) and we get 'analysis failed' with totalVisits=1"`  
  라고 되어 있음.  
  즉, **“거의 돌지 않고 끝난 검색”**이 1방문 결과를 반환하는 것은 의도된 경로임.

정책 heatmap용 경로(android-policy, maxVisits=1)는 **parseRawResult를 거치지 않으므로**,  
로그에 찍히는 totalVisits=1과는 **무관**합니다.

---

## 3. 왜 “계속” 찍히는가

- **분석이 자주 취소되는 상황**이면, totalVisits=1 결과도 자주 발생함.
  1. **수를 둘 때**  
     `playMove` → 이전 분석 `stopAllAnalysis` → 새 분석 `triggerAnalysis`.  
     이전 분석이 막 시작됐을 때 stop이 걸리면, 그 요청은 1방문만 하고 끝나고, 그 결과가 parseRawResult를 타서 totalVisits=1 로그.
  2. **재분석이 잦을 때**  
     debounce 후 `triggerAnalysis(0L)` 등으로 새 분석을 시작하면, 직전 분석 job이 취소되고 네이티브에 `stopAnalysis`가 감.  
     직전 분석이 1방문만 하고 끝나면 역시 totalVisits=1.
  3. **탭 전환**  
     `setActiveSession`에서 `analysisJob?.cancel()` + `stopAllAnalysis`.  
     진행 중이던 분석이 바로 중단되면 1방문 결과가 나올 수 있음.
  4. **불일치 후 재분석**  
     해시/수순 불일치 시 `triggerAnalysis(debounceMs = 150L)` 등으로 재시작하면,  
     그 직전에 돌고 있던 분석이 중단되며 1방문 결과가 한 번 더 나올 수 있음.

그래서 **“방문수 1” 로그가 반복되는 것**은:

- **버그라기보다**,  
  “분석 시작 → 곧바로 취소(stop)됨 → 1방문만 수행된 부분 결과가 그대로 반환됨”  
  이 흐름이 **자주 일어나기 때문**으로 보는 것이 맞음.
- 수를 두거나, 재분석을 자주 트리거하거나, 탭을 오가거나, 불일치 후 재분석이 일어날 때마다  
  **이전 분석 하나가 1방문으로 끊겨 나오는 것**이 로그에 계속 쌓이는 구조임.

---

## 4. 요약

| 질문 | 답 |
|------|----|
| totalVisits=1은 어디서? | 메인 분석(analyzeTop4) 반환값을 `parseRawResult`가 파싱할 때. 정책 heatmap 경로 아님. |
| 왜 1이 나오나? | 검색이 시작 직후 `g_stopFlag`로 중단되어, 1방문만 하고 끝난 결과가 반환되기 때문. |
| 왜 반복되나? | 수 두기·재분석 트리거·탭 전환·불일치 후 재분석 등으로 “방금 시작한 분석을 곧 취소”하는 일이 자주 발생하고, 그때마다 1방문 부분 결과가 한 번씩 반환되어 로그에 찍힘. |

**한 줄**:  
방문수 1 로그는 **분석이 거의 시작되자마자 stop 되어 1방문만 수행된 부분 결과**가 반환될 때마다 찍히며, 그런 “빠른 취소”가 자주 일어나는 구조라서 로그에 반복적으로 나타나는 것이다.
