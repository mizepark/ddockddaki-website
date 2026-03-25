# 엔진 구조 설계: 단일 경로·단일 루프·단일 직렬화 (오류 0 근접)

"제시해" 요청에 따른 **오류 0에 가장 가까운 구조적 설계안**을 구체적으로 정리한 문서다.

---

## 1. 목표 구조 (4개 컴포넌트)

| 컴포넌트 | 역할 | 규칙 |
|----------|------|------|
| **EngineFacade** | 엔진 초기화/분석/중단/재시작의 **유일한 입구** | 앱 내 어디서든 엔진 호출은 퍼사드 통과만 허용 |
| **EngineOrchestrator** | **상태머신 단일 루프**. 이벤트 큐를 직렬 처리 | 상태 전이는 오직 여기서만. 중복/무효 이벤트는 큐 단계에서 필터 |
| **SessionStore** | 세션 상태 **저장/복원 전담** | 세션 전환 시 캡처·persist·복원만 담당, 엔진/분석 호출 없음 |
| **ViewModel** | **UI 이벤트를 Orchestrator에 전달만** | 상태 전이·엔진 호출·세션 저장 로직 없음. 반영만 |

---

## 2. 상태머신 핵심 규칙

- **상태는 4개만 유지**: `Idle` → `Ready` → `Analyzing` → `Switching`
- **모든 이벤트는 큐로 들어가고 순서대로 처리**
- **중복 이벤트는 큐 단계에서 필터링** (동일 국면·동일 의도 스킵)
- **엔진 호출은 퍼사드 내부에서만 직렬화** (Mutex 1개)

### 2.1 상태 정의

```
Idle      : 앱 시작 직후, 초기화 전. 이벤트만 쌓임.
Ready     : 엔진 초기화 완료. 분석 시작·세션 전환·설정 변경 수용 가능.
Analyzing : 분석 실행 중. StartAnalysis 무시 또는 큐잉. SwitchSession 시 Stop 후 Switching.
Switching : 세션 전환 중(저장/복원). 완료 후 Ready.
```

### 2.2 허용 전이

- `Idle` → `Ready` : Init 성공
- `Ready` → `Analyzing` : StartAnalysis 수신
- `Analyzing` → `Ready` : 분석 완료/중단
- `Ready` → `Switching` : SwitchSession 수신
- `Analyzing` → `Switching` : SwitchSession 수신(내부에서 Stop 후 전이)
- `Switching` → `Ready` : 세션 복원 완료
- `Ready` → `Ready` : ChangeConfig(Restart 후 ensureReady)

---

## 3. 이벤트 처리 흐름 (의사코드)

```text
// ========== 단일 루프 (Orchestrator) ==========
loop:
  event = queue.take()  // 블로킹 또는 채널
  if duplicate(event) then continue

  switch state:
    Idle:
      if event == AppStart: run Init via Facade; on success state = Ready; on failure stay Idle
    Ready:
      if event == StartAnalysis: state = Analyzing; run Analysis via Facade; on done state = Ready
      if event == SwitchSession: state = Switching; SessionStore.save(current); SessionStore.restore(target); state = Ready; emit UpdateUI
      if event == ChangeConfig: Facade.restart(); Facade.ensureReady(); stay Ready
    Analyzing:
      if event == StartAnalysis: drop or re-queue for later
      if event == SwitchSession: Facade.stopAnalysis(); state = Switching; ... (동일)
    Switching:
      // 이벤트 대기 또는 무시 후 Ready로만 전이
```

### 3.1 진입점별 요약

| 시나리오 | 이벤트 | 상태 전이 | 실제 동작 |
|----------|--------|-----------|-----------|
| 앱 시작 | AppStart | Idle → Init → Ready | 모델 설치(필요 시) → Facade.ensureReady() |
| 분석 시작 | StartAnalysis | Ready → Analyzing → (완료) → Ready | Facade.runAnalysis() |
| 세션 전환 | SwitchSession | Ready/Analyzing → (필요 시 Stop) → Switching → Ready | SessionStore 저장/복원, UI 상태만 갱신 |
| 설정 변경 | ChangeConfig | Ready → Restart → Ready | Facade.restart() → ensureReady() |

---

## 4. 구조 설계 요약 (4단일)

| 원칙 | 내용 |
|------|------|
| **단일 진입점** | EngineFacade 외 엔진 호출 금지 (init/analysis/stop/restart 모두 퍼사드) |
| **단일 루프** | 상태 전이는 오직 EngineOrchestrator만 수행. 이벤트 큐 직렬 처리 |
| **단일 락** | 직렬화는 퍼사드 내부 Mutex 1곳만. Orchestrator는 “순서 보장”만, 별도 락 없음 |
| **단일 책임** | ViewModel은 UI 이벤트 → Orchestrator 전달 + Orchestrator 결과 → UI 상태 반영만 |

---

## 5. 현재 코드와의 매핑

| 목표 컴포넌트 | 현재 구현 | 비고 |
|---------------|-----------|------|
| EngineFacade | `EngineFacade` + `KataGoEngineFacade` | 이미 단일 진입·단일 Mutex |
| EngineOrchestrator | **`EngineOrchestrator`** (명시적 4상태 + 단일 이벤트 큐) | **개념적으로 하나의 “이벤트 루프”로 통합 가능**. 현재는 Init/분석/세션/설정변경 이벤트 단일 루프에서 일괄 처리 |
| SessionStore | `EngineSessionOrchestrator`의 switchSession 내부 (캡처·persist·복원) | 역할은 SessionStore와 동일. 이름만 SessionStore로 정리 가능 |
| ViewModel | `EngineViewModel` | 초기화/분석/세션 전환 시 Coordinator·StateMachine·Orchestrator 호출 + UI 반영만 하도록 이미 축소됨 |

### 5.1 설계안에 더 가깝게 가는 방향 (선택)

- **명시적 4상태**: `Idle/Ready/Analyzing/Switching`을 enum으로 두고, Orchestrator가 이 enum만으로 전이하도록 하면 “상태가 한 곳에만 있다”가 명확해짐.
- **단일 이벤트 큐**: AppStart, StartAnalysis, SwitchSession, ChangeConfig를 한 Channel/Queue로 넣고, 한 코루틴에서만 `receive()` → 상태별 분기 처리하면 “단일 루프”가 코드로 드러남.
- **SessionStore 이름**: 세션 저장/복원만 담당하는 타입을 `SessionStore`로 분리(또는 `EngineSessionOrchestrator`를 SessionStore 역할로 문서화)하면 “저장/복원 전담”이 더 분명해짐.

---

## 6. 왜 이 구조가 오류를 줄이는가

| 원인 | 대응 |
|------|------|
| **여러 경로에서 init/analyze** | 경로 1개: Orchestrator → Facade만. “예외 경로” 제거 |
| **상태 전이가 여러 곳에서 발생** | 전이는 Orchestrator 단일 루프에서만. 레이스 원천 차단 |
| **동시성 제어 지점이 여러 개** | 직렬화는 퍼사드 1곳. 추적·검증 난이도 감소 |
| **ViewModel이 비즈니스+UI 혼재** | ViewModel은 “이벤트 전달 + UI 반영”만. 책임 단순화로 실수 구간 축소 |

---

## 7. 체크리스트 (구현/리뷰용)

- [ ] 엔진 호출은 모두 `EngineFacade`를 통하는가?
- [ ] “분석을 시작할지” 결정하는 곳이 한 루프(또는 한 상태머신)인가?
- [ ] 세션 저장/복원은 전용 컴포넌트(SessionStore/Orchestrator 내 저장 로직)만 수행하는가?
- [ ] ViewModel이 상태 전이·엔진·저장 로직을 직접 하지 않고, Orchestrator에 이벤트만 넘기고 결과만 반영하는가?
- [ ] Mutex(또는 동일 직렬화 수단)가 퍼사드 내부 한 곳뿐인가?

이 문서는 **목표 구조**와 **현재 구현 매핑**, 그리고 **오류 0에 가깝게 가기 위한 설계 요약**을 담고 있다.
