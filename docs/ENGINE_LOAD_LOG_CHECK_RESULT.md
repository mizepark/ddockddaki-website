# 엔진 로드 로그 확인 결과

기기 로그(`device_log_engine_check.txt`) 기준으로 엔진이 제대로 로드되는지 확인한 결과입니다.

---

## 1. 엔진 초기화는 성공함 (네이티브)

**02-06 16:31** 및 **02-06 16:48** 앱 실행 구간에서 다음이 순서대로 찍힙니다.

| 단계 | 로그 (KataGoJNI) | 의미 |
|------|------------------|------|
| 1 | `JNI_OnLoad: Global tables initialized` | 네이티브 라이브러리 로드 |
| 2 | `setHomeDir: HOME=.../katago_home_analysis` (또는 `katago_home_play`) | 홈 디렉터리 설정 |
| 3 | `initEngine: Initialized NNEvaluator` | NN 평가기 생성 |
| 4 | `initEngine: ModelFile=.../b10.bin size=12003218 bytes ...` | 모델 파일 로드 |
| 5 | `initEngine: Loaded. Visits=100000, Threads=7, Time=3.00` (또는 30.00) | 설정 반영 |
| 6 | `initEngine: Warmup successful` | 워밍업 성공 |
| 7 | `initEngine: Main Search object initialized` | 메인 검색 객체 생성 완료 |

- **분석 프로세스**(PID 11101, 13586)와 **대국 프로세스**(PID 11259, 13709) 모두 위 시퀀스가 완료됨.
- `initEngine: Reused model. Updated params: ...` 도 보임 → 같은 프로세스에서 모델 재사용 정상 동작.

**결론: 네이티브 쪽 엔진은 제대로 로드되고 있음.**

---

## 2. 앱 쪽: Locale 미설정으로 초기화 지연

다음 로그가 매 실행 시 찍힙니다.

```
I/EngineViewModel(10748): init
I/EngineViewModel(10748): Locale not set, deferring engine initialization
```

- 엔진 **실제 로드**는 그 후에 **서비스 바인드/init** 경로로 진행되어, 위 1번처럼 네이티브 init은 성공함.
- 따라서 “엔진이 아예 안 올라온다” 수준의 실패는 아님. 다만 **앱이 로케일을 설정하기 전까지는 엔진 초기화를 미루는** 동작이 있음.

---

## 3. 초기화 실패/ensureReady 실패 로그는 없음

- `Engine initialization failed` (EngineViewModel) → **없음**
- `Engine ensureReady failed` (EngineInitCoordinator / KataGoEngineFacade) → **없음**
- `Analysis returned empty suggestions ... (0 visits = engine not ready or exception)` → **없음**

즉, 로그 상으로는 **초기화 실패나 ensureReady 실패**로 인한 “엔진 미준비” 메시지는 보이지 않음.

---

## 4. 착수 불가와 연결되는 로그: isLegal_failed

**02-06 16:06:40~41** 구간에 다음이 여러 번 찍힘.

```
W/ErrorHandler( 5910): EngineViewModel: isLegal_failed
    at ... EngineViewModel.ensureMoveIsLegal(EngineViewModel.kt:2749)
    at ... EngineViewModel.playMove$1$legal$1.invokeSuspend(EngineViewModel.kt:1101)
```

- `ensureMoveIsLegal` → `engineFacade.isLegal` → 네이티브 `isLegal` 호출에서 **예외가 나거나 false가 반환**되면 `isLegal_failed` 로 기록됨.
- 그 시점에 **엔진(네이티브)은 이미 로드된 상태**이므로, “엔진 미로드”보다는 **IPC 타임아웃/연결 끊김, 또는 일시적 비정상 상태**에서 legal 체크가 실패한 가능성이 큼.

**결론: “착수 불가”는 코드상 legal 체크 실패(IPC/예외 포함)와 일치하며, 로그상 엔진 로드 자체는 된 뒤에 발생한 실패로 보는 것이 맞음.**

---

## 5. 기타

- **NativeBridge_shutdown: Engine state cleared**  
  - 02-06 16:31:45 에 1회. 대국 쪽 모델을 b6.txt 로 바꾸는 과정에서 PLAY 프로세스 재초기화 시 정상적으로 한 번 호출된 것으로 보면 됨.
- **extractAnalysisData: zero suggestions after filter**  
  - 분석은 돌아가지만 자식 노드 방문이 1 이하로만 쌓여 추천 0개가 나오는 경우. 방문 상한 제거(시간만 사용) 후 재확인 필요.

---

## 6. 요약

| 항목 | 결과 |
|------|------|
| 네이티브 엔진 로드 | ✅ 정상 (NNEvaluator, Warmup, Main Search initialized) |
| 앱 쪽 초기화 실패 로그 | ❌ 없음 |
| ensureReady 실패 로그 | ❌ 없음 |
| Locale not set | ⚠️ 매 실행 시 출력, 초기화 지연만 있고 이후 네이티브 init은 성공 |
| isLegal_failed | ⚠️ 16:06 구간에 다수 → 착수 시 legal 체크 실패(IPC/예외 가능성) |

**정리: 엔진은 제대로 로드되고 있음. “착수 불가”는 엔진 미로드보다는 legal 체크 실패(IPC/예외 또는 일시적 상태)와 연결되는 것으로 보는 것이 타당함.**
