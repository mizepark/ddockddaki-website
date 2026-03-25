# 엔진 레이어 단순화 원칙

**목표**: 변수(경로·분기·중복)를 줄여 크래시 가능성을 줄인다.

---

## 1. 한 가지 길만 두기

| 영역 | 원칙 |
|------|------|
| **엔진 호출** | UI → Orchestrator → Facade(단일 큐) → Controller → Repository(EngineCalls) → Native. 이 경로만 사용. |
| **IntArray(네이티브용)** | 획득/해제는 **NativeSafetyGate.withIntArray** 한 곳만. 직접 acquire/release 금지. |
| **분석 실행** | ViewModel → Facade 한 경로. |
| **결과 반영** | requestId·보드·epoch 게이트 한 번만 통과. |

→ 경로가 하나로 정해져 있으면, “어디서 터졌는지”와 “어디를 고쳐야 하는지”가 명확해진다.

---

## 2. 검증은 한 곳에서

- 네이티브에 넘기기 **직전** 한 곳에서만: 보드 크기·이동 수·버퍼 크기 등 **계약(상한·형식)** 검사.
- 지금은 NativeSafetyGate(크기 상한), Repository/Controller 일부에 흩어져 있음 → 가능하면 **한 레이어(예: Facade 진입 시 또는 Gate)**로 모을 것.

---

## 3. 분기·예외는 줄이기

- “같은 일을 하는데 경로만 다른” 코드가 있으면, **한 경로로 합치고 파라미터로만 구분**.
- 예외 처리: 가능하면 **한 계층에서만** (예: Repository 또는 EngineCalls). 상위는 Result/빈 값만 받고 분기 최소화.

---

## 4. 지금 코드에서 할 수 있는 최소 작업

1. **배열: Gate 한 경로로**  
   AnalysisUseCase 등에서 `NativeBridge.acquireIntArray`/`releaseIntArray` 직접 호출 제거 → **NativeSafetyGate.withIntArray**만 사용. (빈 보드면 `IntArray(0)` 한 번만 쓰고 해제 없음.)

2. **문서 유지**  
   이 파일처럼 “엔진은 이 한 경로만 쓴다”는 걸 한 페이지로 유지하고, 새 코드 추가 시 이 경로를 벗어나지 않기.

3. **추가 검증은 Gate 쪽에**  
   새로 인자 검증이 필요해지면, **기존 호출 경로를 바꾸지 말고** NativeSafetyGate 또는 그 직전 한 곳에만 추가.

---

## 5. 현재 구조가 최선인가? (기능 유지 전제)

**결론**: “완벽한 최선”이라기보다 **이미 합리적**이다. 기능 그대로 두고 레이어를 더 줄이려면 **AnalysisRunner 제거(ViewModel에서 Facade 직접 호출)** 하나가 실질적 개선이다.

| 레이어 | 역할 | 제거/합치기 |
|--------|------|-------------|
| **ViewModel** | UI 상태·의도·스케줄링·결과 게이트 | 유지 |
| **Orchestrator** | 이벤트 루프(Init/StartAnalysis/AnalysisDone/SwitchSession) | 유지. 없으면 ViewModel이 재진입·세션 전환 직접 처리 → 분기 폭증 |
| **AnalysisRunner** | RunParams 만들고 `engineFacade.runAnalysis(...)` 한 줄 위임 | **제거 가능** → ViewModel이 params 만들고 `engineFacade.runAnalysis(...)` 직접 호출. 기능 동일. |
| **Facade** | 단일 진입·short/long 직렬화·ensureReady·세션별 UseCase 선택 | 유지. 없으면 직렬화·진입점이 흩어짐 |
| **AnalysisUseCase** | 이동 배열 채우기·Controller 호출·결과 파싱 | 유지. Facade에 넣으면 Facade 비대해짐 |
| **EngineController** | requestId 추적·Repository 위임 | 유지. Repository에 합치면 Repository가 바인딩+서비스+requestId 모두 담당해 더 비대해짐 |
| **EngineRepository** | 바인딩·초기화·크래시 보호·EngineCalls 호출 | 유지 |
| **EngineCalls** | 타임아웃·재시도·NativeBridge | 유지 |

→ **할 수 있는 최소 조정**: AnalysisRunner 제거 후 “분석 실행 = ViewModel에서 RunParams 구성 + engineFacade.runAnalysis 한 번 호출”. 나머지 레이어는 각자 책임이 있어 없애면 오히려 한 파일에 책임이 몰림.

---

복잡할수록 변수가 많아진다. **한 경로, 한 게이트, 검증 한 곳**을 지키면 단순하고 안정에 가깝다.
