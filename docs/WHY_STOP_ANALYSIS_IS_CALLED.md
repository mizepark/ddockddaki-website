# stopAnalysis()가 호출되는 이유 — 왜 “불필요한 중단 없음”과 공존하는가

요청: 그동안 불필요한 중단/간섭을 없앴다고 했는데, 왜 세션 전환·메모리·onCleared 등에서 stopAnalysis가 여전히 호출되는지 **설명만** 정리.

---

## 1. “불필요한 중단/간섭 제거”가 의미한 범위

제거·완화한 것은 **분석 실행/UI 흐름 쪽**이다.

- **의도 제출**: 같은 국면이어도 재제출 막지 않음 (같은 국면 early return 제거).
- **분석 중 UI**: 분석 돌아가는 동안에도 바둑판 탭 허용 (`suspendUiDuringAnalysis = false`).
- **탭/모드**: “아직 추천 없을 때 탭 무시” 블록 제거로, 분석 모드에서도 착수 시도 가능.

즉, **“분석을 돌리거나 결과를 쓰는 데 불필요하게 막지 않는다”** 에 맞춘 것이고,  
**“stopAnalysis()를 호출하는 코드 자체를 전부 없앤다”** 는 게 아니다.

---

## 2. stopAnalysis()가 호출되는 곳과 이유

호출 위치는 다음 네 가지뿐이다. 각각 **시스템/생명주기/정합성** 때문에 넣어 둔 것이다.

### (1) 세션 전환 시 — EngineOrchestrator (SwitchSession)

- **코드**: `EngineOrchestrator.processEvent(SwitchSession)` 에서 `from != to` 일 때  
  `state == Analyzing` 이면 `stopAnalysis(ANALYSIS)`, `stopAnalysis(PLAY)` 호출 후 `analysisJob?.cancel()`.
- **이유**:  
  탭을 **분석 ↔ 대국**으로 바꾸면 **세션(분석 엔진/대국 엔진)이 바뀐다**.  
  그대로 두면 “지금 돌고 있는 분석”은 **이전 세션** 것이고, 전환 후에는 **새 세션** 기준으로 UI·저장·다음 분석이 이뤄진다.  
  이전 세션 분석을 멈추지 않으면:
  - 결과가 나왔을 때 어느 세션/어느 화면에 반영할지 모호해지고,
  - 새 세션에서 새 분석을 시작할 때 이전 검색과 상태가 섞일 수 있다.  
  그래서 **세션 전환 시 진행 중 분석을 끊는 것**은 **상태 정합성**을 위한 설계 선택이다.

### (2) 메모리 압박 시 — KataGoApp (onTrimMemory / onLowMemory)

- **코드**:  
  - `onTrimMemory`: `TRIM_MEMORY_RUNNING_CRITICAL` 이상일 때만 `stopAnalysis` + 캐시/trim.  
  - `onLowMemory`: `onLowMemory()` 콜백에서 `stopAnalysis` + 캐시/trim.
- **이유**:  
  시스템이 “메모리 부족, 정리해라”라고 할 때 **검색을 멈추지 않으면**,  
  엔진이 계속 메모리를 쓰다가 **OOM·앱 강제 종료**로 이어질 수 있다.  
  여기서 stop은 **앱이 죽지 않게 하기 위한 방어 로직**이라, 제거 대상이 아니라 **유지해야 하는 동작**이다.

### (3) ViewModel 파괴 시 — EngineViewModel.onCleared()

- **코드**: `onCleared()` 에서 `stopAllAnalysis()` → `shutdownAllEngines()` → `engineFacade.close()`.
- **이유**:  
  화면을 나가거나 프로세스가 정리될 때 ViewModel이 사라지면,  
  그때까지 돌리던 분석은 **결과를 받을 주체(ViewModel)가 없어진다**.  
  그대로 두면 리소스만 쓰고, Binder/콜백 쪽에서도 죽은 클라이언트를 참조할 수 있다.  
  그래서 **ViewModel 정리 시 분석 중단 + 엔진 정리**는 **리소스·라이프사이클** 상 필요한 동작이다.

### (4) Binder 콜백 실패 시 (클라이언트 연결 끊김) — KataGoService

- **코드**: `onIntermediateResult` 콜백을 Binder로 보내다가  
  `RemoteException` / `DeadObjectException` 이 나면 “Client disconnected” 로그 후 `NativeBridge.stopAnalysis()`.
- **이유**:  
  엔진은 **별도 프로세스**에서 돌고, 앱(클라이언트)에 콜백을 Binder로 보낸다.  
  앱이 이미 죽었거나 연결이 끊긴 상태면, 계속 콜백을 보내려다 **크래시/에러**가 난다.  
  그래서 “연결 끊김으로 보이는 예외 → 분석 중단”은 **Binder/프로세스 안정성**을 위한 처리다.

---

## 3. 정리

- **“불필요한 중단/간섭이 없다”** 고 한 것은  
  **분석 의도·스케줄·UI** 에서 불필요하게 분석을 막거나 사용자 동작을 막지 않는다는 뜻이다.
- **stopAnalysis()가 호출되는 위 네 곳**은  
  **세션 정합성(탭 전환), 메모리 방어(trim/low memory), 라이프사이클(ViewModel 정리), Binder 안정성(연결 끊김)** 을 위해 **의도적으로** 둔 **필요한 중단**이다.
- 따라서 “왜 이런 중단이 생기냐”에 대한 답은:  
  **제거하려고 만든 게 아니라, 그런 상황에서는 검색을 멈춰야 한다는 요구(설계) 때문에 처음부터 넣어 둔 것**이다.

다만 **세션 전환** 시에는 “탭만 바꾼 것”인데도 진행 중인 분석이 끊기므로,  
탭을 자주 바꾸면 시간 예산을 다 쓰기 전에 검색이 종료될 수 있다.  
그건 “불필요한 간섭”이 아니라 **세션 전환 시 상태를 깔끔히 하려는 설계의 부작용**에 가깝다.
