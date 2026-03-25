# “빈 결과” 원인 정리 (코드상 4가지 수렴)

분석/헬스체크에서 **빈 결과**가 나오면 심각한 문제다.  
코드상 원인은 아래 **네 가지**로 수렴한다. 디버깅 시 이 순서로 확인하면 된다.

---

## 1) 네이티브 준비 실패

- **의미**: 라이브러리 로드 실패 또는 홈 디렉터리 설정 실패 → `ensureNativeReady`가 예외를 던짐 → 분석 실패 → 빈 결과로 귀결.
- **위치**:
  - `KataGoService.onBind` (또는 `KataGoPlayService.onBind`)
  - `KataGoService.ensureNativeReady()`  
    → `nativeInitError != null` 이면 `RemoteException("Native init failed: ...")`  
    → `!isNativeReady` 이면 `RemoteException("Native engine is not ready")`
- **확인**: 서비스 프로세스 로그에서 `Native init failed` / `Native engine is not ready` 검색.

---

## 2) 서비스/바인더 불안정

- **의미**: 바인더가 죽었거나 `RemoteException`이 발생하면, 호출 측에서 빈 배열을 반환하도록 되어 있음.
- **위치**:
  - `EngineRepository.analyzeTop4`  
    - `service?.asBinder()?.isBinderAlive != true` → `FloatArray(0)` 반환, `markServiceDead()`  
    - `catch (e: Exception)` (RemoteException 등) → `FloatArray(0)` 반환, `markServiceDead()`
- **확인**: 로그 `"Dead binder detected in analyzeTop4"` / `"Remote analyzeTop4 failed"`.

---

## 3) 네이티브 분석 자체 실패

- **의미**: 서비스 내부에서 `analyzeTop4` 호출 중 예외 발생 → 빈 배열로 처리됨.
- **위치**:
  - `KataGoService` (또는 `KataGoPlayService`) 내부 `analyzeTop4` 구현  
    - `ensureNativeReady()` 통과 후 `NativeBridge.analyzeTop4(...)` 실행  
    - `catch (t: Throwable)` → `FloatArray(0)` 반환, `Timber.e(t, "Remote analyzeTop4 failed")`
- **확인**: 서비스 프로세스 로그에서 `Remote analyzeTop4 failed` + 스택트레이스. 네이티브(JNI) 예외 여부 확인.

---

## 4) 초기화 타이밍 경합 (좀비 상태)

- **의미**: `shutdown` 직후 재초기화/헬스체크가 겹치면, 엔진이 “응답은 하지만 결과는 0”인 좀비 상태로 들어감.
- **위치**:
  - `EngineViewModel.initializeEngine`  
    - `engineController.initEngine(...)` 직후 `runHealthCheck()`  
    - 2회 전체 초기화 재시도 시 첫 실패 후 `engineController.shutdown()` → `delay(2500L)` → 재 init
  - `NativeBridge.shutdown` (네이티브에서 `g_eval`/`g_search` 리셋)  
    - 이 직후 다른 스레드/호출에서 `analyzeTop4`가 들어오면 `g_eval == null` 등으로 빈 결과 반환 가능.
- **확인**: 헬스체크 로그 `"Engine Health Check attempt ... empty or no suggestions"`, `"Engine not initialized (check 1/2)"` (네이티브 로그).

---

## 정리

| 원인 | 계층 | 빈 결과로 이어지는 경로 |
|------|------|-------------------------|
| 1) 네이티브 준비 실패 | 서비스 | `ensureNativeReady` 예외 → 호출부에서 예외 처리 시 빈 배열 또는 2번으로 전이 |
| 2) 서비스/바인더 불안정 | Repository | Dead binder / RemoteException → `FloatArray(0)` |
| 3) 네이티브 분석 실패 | 서비스 | `NativeBridge.analyzeTop4` 예외 → `FloatArray(0)` |
| 4) 초기화 타이밍 경합 | ViewModel + 네이티브 | shutdown 직후 재 init/헬스체크 겹침 → 좀비 → 빈 결과 |

**“빈 결과”는 엔진이 제대로 준비되지 않았거나, 서비스 연결이 끊겼거나, 네이티브 호출이 실패한 상태를 의미한다.**  
로그와 코드 구조상 위 네 가지 중 하나가 실제 원인이다.
