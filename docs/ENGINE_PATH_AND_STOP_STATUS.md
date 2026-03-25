# 엔진 경로 전체 · 중단 현황 (정밀 분석 요약)

**목적**: 엔진 단일 진입점·직렬화·결과 게이트와 “중단 호출이 남아 있는 지점”을 한 문서에서 정리한다.  
**기준**: “필요한 중단만 유지”(세션 전환, 재시작/종료, 크래시 복구). 그 외는 새 스냅샷 + epoch 무효화만.

---

## 1. 단일 진입점 · 직렬화 · 고정 경로 (현재 구조)

| 항목 | 내용 | 참조 |
|------|------|------|
| **단일 진입점** | 앱은 EngineFacade만 사용. Controller/Repository 직접 접근은 EngineBootstrap에서만 생성·주입. | EngineFacade.kt, EngineBootstrap.kt |
| **직렬화 실행** | KataGoEngineFacade 단일 워커(short/long 채널)로 엔진 호출 직렬화. 짧은 호출 우선. | KataGoEngineFacade.kt |
| **분석 실행 경로 고정** | ViewModel → EngineFacade. “분석 수행 경로”는 하나. | EngineViewModel.kt, EngineFacade.kt |
| **분석 의도 스케줄링** | AnalysisStateMachine / AnalysisScheduler가 보호 시간·중복 방지 담당. | AnalysisStateMachine.kt, AnalysisScheduler.kt |
| **결과 게이트** | requestId·보드 크기·**epoch** 일치 시에만 결과 반영. 이전 세대 결과 폐기. | AnalysisSnapshotValidator.kt |

---

## 2. 경합·중단 현황 — **현재(적용 후)**

### 2.1 제거 완료 (ViewModel 내 “간섭” 경로)

| 이전 문제 | 현재 상태 |
|-----------|-----------|
| UI 이벤트마다 stopAnalysis (착수/되돌리기/점프/변형 종료/오버레이) | **제거됨**. 해당 경로에서는 `updateAnalysisIntent(NO_STOP)`만 호출, stopAnalysis 없음. |
| 기본 정책 STOP_BEFORE (의도 갱신 시 기존 분석 먼저 중단) | **제거됨**. 기본값 NO_STOP, runAnalysisInternal에 stopBefore 분기 없음. |
| 힌트/딥 모드 목표 도달 시 stopAnalysis | **제거됨**. 목표 도달은 “결과 채택 조건”(shouldShowSuggestions)으로만 처리. |

### 2.2 stop이 호출되는 필수 상황만 (stop 남발 제거 후)

| 호출 위치 | 용도 |
|-----------|------|
| **EngineViewModel.onCleared()** | ViewModel 파괴 시에만 `stopAllAnalysis()` 호출. |
| **KataGoApp.onLowMemory()** | 시스템 onLowMemory 콜백 시에만 분석 중단·캐시 정리. |
| **KataGoService** (콜백 전송 실패 시) | RemoteException/DeadObjectException(클라이언트 연결 끊김) 시에만 NativeBridge.stopAnalysis(). |

**의도적으로 stop 하지 않는 경우**
- **EngineOrchestrator SwitchSession**: 탭 전환 시 엔진에 stop 보내지 않음. analysisJob만 cancel 해서 결과 미적용. (stop 남발 방지)
- **KataGoApp.onTrimMemory(모든 레벨)**: CRITICAL 포함 메모리 트림에서는 stopAnalysis 호출 안 함. clearCache·trimMemory만 수행.

ViewModel 내부에서는 **onCleared 한 곳**에서만 stopAllAnalysis를 호출한다.  
착수/Undo/Jump/변형/오버레이/의도 갱신/힌트·딥 목표 도달·탭 전환·메모리 트림에서는 **stop 호출 없음**.

### 2.3 필수 중단 신호 레이스 → 이제 (일상 사용에서는) 안 생김

**과거 우려**: stopAnalysis 호출 지점들, KataGoApp 메모리 트림, JNI stop 플래그가 겹치면 “stop 신호 ↔ 엔진 결과 반환” 타이밍 레이스로 **낮은 방문 수가 확정**될 수 있음.

| 구분 | 현재 상태 |
|------|-----------|
| **일상 사용 중** | 탭 전환·메모리 트림에서 **stop을 더 이상 보내지 않음**. 따라서 일반적인 분석 중에는 **stop 레이스가 발생할 경로가 없음** → 낮은 방문 수 확정은 구조적으로 발생하지 않음. |
| **필수 stop 3곳** | onCleared / onLowMemory / 클라이언트 끊김 시에는 stop을 쏨. 이때 JNI stop 플래그와 결과 반환 사이에 **이론적 레이스는 있을 수 있음**. 다만 (1) onCleared면 화면이 사라지는 중, (2) onLowMemory면 의도적으로 연산 중단해 메모리 회수, (3) 끊김 시엔 결과 수신자 없음 → **낮은 방문 수가 나와도 문제 되지 않음**. |

→ **정리**: “필수 중단 신호의 레이스로 인한 낮은 방문 수 확정”은 **일상 사용에서는 더 이상 발생하지 않고**, 필수 stop이 나가는 극히 드문 상황에서만 이론적 레이스가 있을 수 있으며 그때는 의도된/수용 가능한 동작이다.

### 2.4 구성 상한 불일치/적용 실패로 인한 조기 종료 → 완화된 상태

**과거 우려**: maxTime/maxVisits/maxPlayouts 제한이 낮거나 cfg 적용이 실패해, 실제 검색이 1~수십 방문으로 끝남. (근거: katago_jni.cpp 파라미터 적용, AssetInstaller의 cfg 기록, 프로젝트 자산 cfg의 낮은 상한.)

| 구분 | 현재 상태 |
|------|-----------|
| **AssetInstaller analysis.cfg** | `ensureConfigFile`가 **쓸 때** analysis.cfg에 `maxTime = 3.0`, `maxVisits = 100000` 고정 기록. `maxPlayouts`는 **기록하지 않음** → KataGo `loadSingleParams`에서 미지정 시 기본값 `1<<50` 사용. |
| **JNI analyzeTop4** | `localParams.maxPlayouts = std::max(localParams.maxVisits, g_globalParams.maxPlayouts)` 로 **항상 maxPlayouts ≥ maxVisits** 보장. cfg에 낮은 maxPlayouts가 있어도 이 호출에서는 상한이 잘리지 않음. |
| **초기화 경로** | `initEngine`에 넘기는 config 경로는 `AssetInstaller.getConfigFile(app)`(내부 저장소 파일) 하나. **assets의 analysis.cfg**(maxTime=1.6, maxVisits=1600)는 초기화 경로로 사용되지 않음. |
| **적용 실패 시** | `ensureConfigFile` 실패 시 예전/손상된 cfg가 남으면 낮은 상한으로 동작할 수 있음. **재시도 1회 + 성공/실패 로그**로 완화. |

**ensureConfigFile이 실패하는 이유 (드묾)**: (1) **저장 공간 부족** — `writeTextAtomic` 내부 `tmp.writeText()` 또는 `copyTo()` 시 IOException(NoSpaceLeftOnDevice). (2) **I/O 예외** — 내부 저장소 쓰기 권한/손상, 일부 기기에서 `renameTo` 실패 후 `copyTo`에서 예외. (3) **getCpuConfig 예외** — `DeviceSpec.readRamInfo(context)` / `DeviceSpec.read(context)` 실패(일부 커스텀 ROM·권한 제한). 정상 기기·저장 공간 있으면 실패하지 않으며, 그래서 “2회 연속 실패”는 **일시적 저장소 문제나 비정상 환경**에서만 발생한다.

→ **정리**: 구성 상한 불일치/적용 실패로 인한 조기 종료는 **cfg 기록값·JNI 상한 맞춤·단일 경로**로 대부분 막혀 있고, **ensureConfigFile이 2회 연속 실패할 때만** 이론적 가능성이 남는다.

---

## 3. 방문수 0과의 연결

- **빈 결과 재시도**: 최종 결과가 비어 있을 때 “0 방문/빈 추천”으로 간주하고 1회 재시도. (EngineViewModel, emptyFinalRetry 경로)
- **의미**: 엔진 미준비/예외/과거에는 “중단 간섭”도 같은 경로로 0 방문이 될 수 있었음.
- **현재**: UI 이벤트·의도 갱신·목표 도달에서 중단을 호출하지 않으므로, “간섭에 의한 0 방문” 경로는 구조적으로 제거된 상태. (미준비/예외만 남음)

---

## 4. 안정 장치 (이미 적용)

- **단일 큐 직렬화 + 결과 게이트**: 잘못된 결과 반영 차단. (KataGoEngineFacade, AnalysisSnapshotValidator)
- **Epoch**: 이전 세대 결과는 `runEpoch < lastSubmittedEpoch`로 폐기.
- **분석 의도 스케줄러**: 보호 시간 내 중복 실행 지연. (AnalysisScheduler)

---

## 5. 한계 정리

- **과거**: “중단 호출이 분산된 구조” → 직렬화/게이트로 방어하지만 간섭 발생 자체는 남아 있음.
- **현재**: ViewModel에서는 **필수 중단(onCleared)만** 호출하고, 나머지는 새 스냅샷 + epoch 무효화로 처리.  
  → **“간섭이 없는 구조”에 가깝게 정리된 상태**.  
  세션 전환·재시작·종료·복구는 “엔진 컨텍스트 교체/종료”이므로 유지.

---

## 6. 코드 검증 (문서-코드 일치 확인)

문서가 “중단 최소화 완료”라고 주장하는 내용이 실제 코드와 맞는지, 아래 위치에서 확인할 수 있다. **해당 라인에 아래와 다르게 stop 호출이 있으면** 브랜치/버전이 다르거나 변경이 반영되지 않은 것이다.

| 확인 항목 | 파일·위치 | 있어야 하는 내용 (현재 코드 기준) |
|-----------|-----------|-----------------------------------|
| 착수 시 중단 없음 | EngineViewModel.kt, playMove 내부 (~L1061 부근) | `val tStart` → 바로 `// 1. Check Tokens First`·`ensureActiveGameTokens`. **stopAnalysis 호출 없음**. |
| 변형 종료 시 중단 없음 | EngineViewModel.kt, stopVariation (~L1315 부근) | `viewModelScope.launch { _uiState.update { ... }` 후 `triggerQuickAnalysis(stopBefore = false)`. **stopAnalysis 호출 없음**. |
| Undo/Jump 시 중단 없음 | EngineViewModel.kt, jumpToStart/jumpBack5 등 (~L1388, L1413, L1476, L1511, L1550, L1614 부근) | `_uiState.update`·`persistGameState`·`refreshStonesAsync`·`updateAnalysisIntent(NO_STOP)`. **stopDeferred/stopAnalysis 없음**. |
| 오버레이에서 중단 없음 | EngineViewModel.kt, 소유권 오버레이 (~L696 부근) | `ownershipFetchJob` 내부에서 `engineOpMutex.withLock { try { val snapshot ... fetchRootLead ... }`. **canInterruptAnalysis/stopAllAnalysis 블록 없음**. |
| 기본 NO_STOP, stopBefore 분기 없음 | EngineViewModel.kt, updateAnalysisIntent (~L1897), runAnalysisInternal (~L1925) | `stopPolicy = ... NO_STOP` 기본값. `tentativeIntent`·`currentEpoch++`·`epoch = currentEpoch`. runAnalysisInternal 시작에 **stopBefore 분기·stopAllAnalysis 호출 없음**, `lastRunEpoch = intent.epoch` 있음. |
| 힌트/딥 목표 도달 중단 없음 | EngineViewModel.kt, handleAnalysisResult (~L2088 부근) | 게이트 통과 후 주석 "HINT/DEEP 목표 도달은 결과 채택 조건으로만 처리. 중단 호출 없음". **stopAllAnalysis 호출 없음**. |
| epoch 기반 게이트 | AnalysisSnapshotValidator.kt (L10–L24) | `shouldProcessAnalysisResult(..., runEpoch, lastSubmittedEpoch)`. 내부에 `if (runEpoch < lastSubmittedEpoch) return false`. |
| Intent에 epoch | AnalysisStateMachine.kt (L20–L27) | `AnalysisIntent(..., epoch: Long = 0L)`. KDoc "세대. 결과 게이트에서 이전 세대 결과 폐기용." |
| ViewModel에서 stop 호출 한 곳 | EngineViewModel.kt | `stopAllAnalysis` **호출**은 **onCleared()** 내부(L1887) `runCatching { stopAllAnalysis() }` **한 곳만**. (정의는 L252–254.) |

---

## 7. 엔진 관련 오류 가능성 정리

**“가능성이 이제 없다”가 아니라 “구조적으로 많이 줄었다”**가 맞다.

| 구분 | 상태 | 비고 |
|------|------|------|
| **불필요한 중단/간섭** | 구조적으로 제거됨 | UI 이벤트·의도 갱신·목표 도달에서 stop 제거, epoch로 이전 결과 폐기. |
| **방문 0/1에서 끊김 (간섭 때문)** | 가능성 크게 감소 | 간섭 경로 제거로 “의도적 중단”에 의한 0 visits는 거의 제거. |
| **엔진 초기화 실패·미준비** | 여전히 가능 | 첫 초기화 전·복구 중·모델 없음 등에서 빈 결과/0 visits 가능. |
| **네이티브 크래시·타임아웃** | 여전히 가능 | JNI/프로세스 죽음·DeadObject·타임아웃 시 예외·빈 결과 가능. |
| **필수 중단으로 인한 끊김** | 의도된 동작 | **onCleared·onLowMemory·클라이언트 끊김** 시에만 stop 호출. (세션 전환·메모리 트림에서는 stop 안 함.) |
| **구성 상한 불일치/적용 실패** | 완화됨 | analysis.cfg에 maxTime=3, maxVisits=100000 기록; JNI에서 maxPlayouts≥maxVisits 강제. ensureConfigFile 실패 시에만 낮은 상한 가능. |

→ **정리**: “간섭/불필요한 중단”에 의한 오류는 구조적으로 줄었고, **엔진 미준비·예외·크래시·필수 중단**에 의한 오류 가능성은 남아 있다.

---

## 8. CPU 전용 및 FP32 사용

| 항목 | 현재 코드 |
|------|-----------|
| **CPU 전용** | KataGoApp 시작 시 `setCpuModeEnabled(true)` 고정. KataGoEngineFacade에서도 CPU 모드 강제. 반복 크래시 시 CPU로 fallback. |
| **FP32만 사용** | EngineViewModel 내 `val useFp16 = false` 고정. config에도 FP32 경로 사용 (엔진 설정 키는 네이티브 계약). |
| **튜닝** | CPU 전용 빌드에서 tuned 파일은 쓰지 않음(또는 삭제). |

→ **CPU 전용 앱으로서 FP32만 사용하도록 되어 있다.**  
설정 파일·AIDL 메서드명에 남는 키/이름은 네이티브 엔진 계약으로만 사용되며, 런타임에서는 사용하지 않음.

---

## 9. 참고도 PV(수순) 상한과 “수가 적게 나오는” 이유

| 항목 | 내용 |
|------|------|
| **상한** | 참고도(분석) 결과에서 후보당 PV는 **최대 26수**까지. 네이티브 `kMaxAnalysisPvLen = 26`, `kFloatsPerAnalysisSuggestion = 59`(7+26×2)로 상수화. |
| **15→26** | 이전 15수 상한 하드코딩 제거 후 26수로 확장. JNI·Kotlin 파싱 모두 동일 상한 사용. |
| **5·10처럼 적게 나오는 이유** | 26은 **상한**일 뿐, 각 수의 PV 길이는 **엔진이 그 수에서 실제로 탐색한 주변 수순 길이**에 따름. 어떤 수는 그 가지에서 탐색이 짧거나, 국면이 빨리 끝나서 5·10개만 있고, 다른 수는 26개까지 있을 수 있음. 즉 “최대 26까지 줄 수 있다”는 의미이고, 5·10처럼 적게 나오는 것은 **정상 동작**이다. |
| **rootDesiredPerChildVisitsCoeff** | 최소 방문 유도용(우리 앱에서는 미사용). cfg 허용 범위: **0.0 ~ 100.0** (실수). 0=미적용. |

---

## 10. 참조 문서

- [NO_STOP_EPOCH_DESIGN.md](NO_STOP_EPOCH_DESIGN.md) — 중단 제거·epoch 설계 및 구현 요약.
- [STRUCTURAL_GOALS.md](STRUCTURAL_GOALS.md) — 구조적 봉쇄 7가지.
- 분석 PV 상한·포맷: `katago_jni.cpp`의 `kMaxAnalysisPvLen`, `kFloatsPerAnalysisSuggestion`; Kotlin `AnalysisUseCase.kt`의 `floatsPerSuggestion = 59`.

---

## 11. 실기기 로그 수집 (재현·원인 확인용)

### 절차

1. **버퍼 비우기**
   ```bash
   adb logcat -c
   ```
2. **앱 실행** (재현 시나리오 진행)
3. **수집** — 아래 중 하나로 덤프 후 필터
   - 전체 덤프 (최근 N줄): `adb logcat -d -t 5000 > logcat_after_restart.txt`
   - **앱만 수집 (권장)**: `adb logcat -d -t 3000 | findstr /i "Katago KataGoJNI katago_jni_cpu NativeBridge KataGoService EngineService binding prepareModel ensureReady 10767"`  
     (10767은 해당 기기에서 com.katago.android UID일 수 있음. 필요 시 `adb shell pm list packages -U`로 확인)

### 2026-02-07 수집 결과 요약

- **로그 파일**: `docs/logcat_after_restart.txt` (버퍼 클리어 후 15초 대기 → `adb logcat -d -t 5000` 저장)
- **내용**: 해당 구간에 **com.katago.android** 로그가 거의 없음 (시스템/라운처 위주). 앱을 15초 안에 켜지 않았거나, 앱 로그 태그가 다른 경우로 추정.
- **다음 수집 시**: 위 “앱만 수집” 명령으로 **앱을 켠 직후** 10~20초 뒤에 실행하면 KataGo·엔진·바인딩 관련 라인만 골라서 확인하기 좋음.
