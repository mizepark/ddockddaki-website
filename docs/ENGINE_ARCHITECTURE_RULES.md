# 엔진 레이어 아키텍처 원칙

다음 6가지 원칙을 지키면 구조적 오류를 줄이고 안정성을 높일 수 있다.

---

## 1. 불가능한 상태를 타입으로 제거

- **위치**: `EngineTypes.kt`, `EnginePhase`, `EngineOrchestratorState`
- 상태를 **sealed class / enum**으로 모델링하고, **when(exhaustive)** 로 전이를 강제한다.
- `EnginePhase`가 이 방향으로 설계되어 있으므로, **확장 시 반드시** 새 상태/이벤트를 when에서 모두 처리해 컴파일러가 누락을 막게 한다.
- 새 enum 값 추가 시 `EngineOrchestrator.processEvent` 및 `UiState.enginePhase` 반영 경로를 함께 추가한다.

---

## 2. 단일 이벤트 큐 + 단일 스레드 원칙

- **위치**: `EngineOrchestrator.kt`
- 엔진/세션/분석 이벤트는 **반드시 `EngineOrchestrator.post()`를 통해서만** 진입한다.
- ViewModel → `post(EngineEvent)` → `processEvent` 한 갈래로만 진입하도록 강제한다.
- 이 구조가 깨지면(다른 경로로 엔진·세션·분석을 직접 호출하면) 오류 가능성이 급증한다.

---

## 3. 도메인 타입으로 입력 자체를 제한

- **위치**: [EngineDomainTypes.kt](../app/src/main/java/com/katago/android/engine/EngineDomainTypes.kt)
- `Int`/`String` 대신 의미 있는 타입(예: `PositiveVisits`, `BoardSize`, `MaxVisits`)으로 값을 만들고,
  **생성 실패 시 null**을 반환해 상태가 생성되지 않게 한다.
- 이렇게 하면 “잘못된 값이 로직에 들어오는 상황” 자체가 제거된다.
- 확장 시 동일 패턴을 유지한다(팩토리 `of(raw): T?`, private 생성자).

---

## 4. 순수 함수 중심의 코어 로직

- **위치**: `GameUseCase`, `AnalysisUseCase`
- 게임 로직/분석 결과 파싱은 **부작용 없는 순수 함수**로 유지한다.
- 이미 두 UseCase가 이 패턴에 가깝게 작성되어 있으므로, 이 원칙을 유지하면 외부 이벤트에 흔들리지 않는다.
- I/O·엔진 호출은 UseCase 경계 밖(Repository/ViewModel)에서만 수행한다.

---

## 5. 경계 계층 분리 고정

- **위치**: `EngineRepository`, `EngineFacade`, `NativeBridge`
- **Native/Service 접근은 `EngineRepository`(및 퍼사드)로 완전히 격리**한다.
- 상위 계층은 “실패 가능성”을 **타입**(Result/throw/suspend)으로 받는 구조로 설계한다.
- 이 경계를 더 엄격히 유지하면 안정성이 올라간다.

---

## 6. 상태 전이는 오직 한 곳에서만

- **위치**: `EngineViewModel`
- **UI 상태 변화는 `EngineViewModel`에서만** 일어나게 하고,
  다른 컴포넌트(UseCase, Repository, Orchestrator 콜백 내부에서의 직접 수정 등)가 `_uiState`를 직접 바꾸지 못하게 한다.
- Orchestrator/InitCoordinator는 **콜백으로 ViewModel에 결과만 전달**하고, **실제 `_uiState.update`는 ViewModel 내부**에서만 수행한다.
- 이렇게 하면 “예기치 않은 상태 변경”을 원천적으로 차단한다.
