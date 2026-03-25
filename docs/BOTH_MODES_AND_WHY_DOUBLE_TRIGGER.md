# 현상이 분석/대국 둘 다인지, 그리고 트리거가 이중이 되는 이유 (설명만)

---

## 1. 분석 모드·대국 모드 둘 다 해당하는가?

**둘 다 해당한다.**

- 분석 탭·대국 탭 모두 **같은 EngineViewModel**을 쓴다.  
  (분석: AnalysisUiViewModel → EngineViewModel, 대국: PlayViewModel → analysisFacade → EngineViewModel)
- `triggerAnalysis`는 EngineViewModel에만 있고, **탭을 구분하지 않고** 한 군데서만 동작한다.
- 그래서 **분석 모드에서 수를 두거나 undo/redo를 하든**, **대국 모드에서 수를 두거나 탭을 전환하든**,  
  같은 `triggerAnalysis`가 불리고, 같은 “시작 직후 이중 트리거 → 1방문에서 끊김” 메커니즘이 적용될 수 있다.
- 따라서 **현상(visits=1, 시작하자마자 끊김)은 분석 모드·대국 모드 둘 다에서 생길 수 있다.**

---

## 2. 트리거가 이중이 되는 이유

**“시작 직후”에 두 번째 트리거가 들어오는** 경로는 대략 아래와 같다.

### 2.1 파이프라인 안에서 다시 트리거 (가장 유력)

- **handleAnalysisResult 안에서** `triggerAnalysis`를 다시 부르는 구간이 있다.  
  - 빈 최종 결과 재시도: `triggerAnalysis(debounceMs = 0L)`  
  - 해시/수순 불일치 시: `triggerAnalysis(debounceMs = 150L)`  
  - 타임아웃 등 다른 재시작 경로
- 흐름이 이렇게 된다.  
  - 한 번 분석을 **시작** → 네이티브 검색 돌기 시작  
  - **얼마 안 가서** (또는 1방문만 하고) 중간/최종 결과가 한 번 들어옴  
  - 그 결과를 처리하는 **handleAnalysisResult** 안에서 위 조건에 걸려 **triggerAnalysis(0 또는 150)** 호출  
  - 그러면 **지금 돌고 있던 analysis job이 cancel**되고, **새 job**이 뜨면서 **stopAllAnalysis** 호출  
  - 그래서 **방금 돌기 시작한(또는 결과 내보낸) 검색이 1방문에서 끊긴 것처럼** 보인다.
- 즉, **“분석이 한 번 시작됐는데, 그 분석의 결과 처리 로직이 같은 분석을 다시 트리거”**해서 이중으로 들어가는 것이다.

### 2.2 같은 사용자 동작에서 두 경로가 거의 동시에 호출

- **한 번의 사용자 동작**(수 두기, undo, 탭 전환 등) 뒤에  
  **서로 다른 코드 경로**가 둘 다 `triggerAnalysis`를 부를 수 있다.
- 예:  
  - **playMove** 끝에서 `triggerAnalysis(stopBefore = false)` 한 번 (기본 300ms debounce라 300ms 뒤에 실제 시작).  
  - 그 전에 또는 직후에 **다른 경로**(state 갱신에 반응하는 flow, LaunchedEffect, setTargetLatency, newGame 등)에서 `triggerAnalysis(0)`가 나가면  
  - “즉시 시작” 한 번 + “300ms 뒤 시작” 한 번이 겹쳐서, **같은 국면에 대해 트리거가 두 번** 들어갈 수 있다.
- **setActiveSession**에서 탭 전환 시 `triggerAnalysis(0)` 한 번 호출.  
  그 직후 **해당 탭의 UI/ViewModel**에서 (LaunchedEffect, 초기화, flow 수집 등으로) 또 `triggerAnalysis`가 한 번 나가면  
  시작 직후 이중 트리거가 된다.

### 2.3 debounce 300ms와 “즉시 시작”이 겹치는 경우

- 많은 호출처가 **기본값**으로 `triggerAnalysis(debounceMs = 300L)`를 쓴다.  
  → “300ms 뒤에 triggerAnalysis(0) 한 번 더”가 **스케줄**만 되고, 300ms 안에 **다른 경로**가 `triggerAnalysis(0)`를 호출하면  
  - 먼저 **즉시 시작**이 한 번 나가고,  
  - 300ms 후에 예전에 스케줄된 **한 번 더**가 나갈 수 있다.  
- 이때 **reqId 비교**로 “이미 새 분석이 시작됐으면 스케줄된 건 실행 안 함”처럼 되어 있어도,  
  **“즉시 시작”이 두 경로에서 연달아(수십 ms 간격으로)** 나가면,  
  첫 번째로 시작한 분석이 두 번째 triggerAnalysis(0)에 의해 **시작 직후 취소**되는 이중 트리거가 된다.

---

## 3. 요약

| 질문 | 답 |
|------|----|
| **분석/대국 둘 다 해당?** | 예. 같은 EngineViewModel·같은 triggerAnalysis를 쓰므로 **둘 다**에서 visits=1·시작 직후 끊김이 나올 수 있다. |
| **트리거가 이중이 되는 이유** | (1) **handleAnalysisResult 안에서** 빈 결과 재시도·불일치 재시작 등으로 **같은 분석을 다시 트리거**해서, 돌고 있던 job이 취소되고 막 시작한 검색이 끊김. (2) **한 동작에 대해** playMove/setActiveSession 등과 **다른 경로**(flow, LaunchedEffect, 설정 반영 등)가 **거의 동시에** triggerAnalysis를 호출. (3) **300ms debounce로 스케줄된 “한 번 더”**와 **즉시 시작(0ms)** 이 겹쳐서, 시작 직후 한 번 더 트리거가 나감. |

즉, **“시작 직후 이중 트리거”**는  
- **파이프라인 내부에서의 재시작**(handleAnalysisResult → triggerAnalysis)과  
- **여러 호출처가 같은 시점에 triggerAnalysis를 부르는 것**  
두 가지가 겹쳐서 생기는 것으로 보면 된다.
