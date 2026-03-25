# 기기 스트레스 테스트 계획

기기에서 **이전 데이터 없이** 클린 빌드 설치 후 **10만 회 스트레스 테스트**를 실행하고, 엔진 초기화·중단·대국·힌트/참고도·기력별 동작을 검증하기 위한 절차와 체크리스트입니다.

---

## 1. 클린 설치 및 10만 회 스트레스 실행 (한 번에)

**목표**: 캐시/이전 데이터가 섞이지 않도록 정리한 뒤, 최신 코드만으로 빌드·설치하고 기기에서 10만 회 스트레스 테스트를 돌립니다.

### 1.1 한 번에 실행 (권장)

```powershell
# 프로젝트 루트에서 (Cursor/Android Studio 종료 권장)
.\stress_test_full_cycle.ps1
```

- Gradle 데몬 정리, `build`/`app\build`/`app\.cxx`/`katago_assets\build` 삭제
- 기존 앱 **완전 제거** (`adb uninstall`) → 이전 데이터 미혼합
- `bundleProdDebug` 클린 빌드 (--no-build-cache --rerun-tasks)
- bundletool로 APKS 생성 후 기기 설치
- 앱 자동 실행 + **STRESS_TEST** 인텐트로 **100,000 스텝** 시작

### 1.2 수동 단계 (빌드/설치만 하고 테스트는 나중에)

```powershell
# 1) 캐시·빌드 삭제 후 클린 빌드 & 설치 (이전 데이터 없음)
.\clean_build_install.ps1

# 2) 이미 설치된 앱에서 10만 회 스트레스만 시작
.\run_stress_test_only.ps1 -Steps 100000
```

### 1.3 로그 확인

```bash
adb logcat -s StressTest:I StressTest:W StressTest:E
```

- `step=100000` 및 `COVERAGE [...]` 로그가 나오면 정상 종료.
- `ABORTED` 시 `error=` 원인 확인 후 수정 → `.\stress_test_full_cycle.ps1` 재실행.

### 1.4 COVERAGE 해석

스트레스 테스트가 끝나면 `COVERAGE [Completed] ...` 로그에 기능별 실행 횟수가 찍힙니다.  
**어떤 항목이 0이면** 해당 경로가 한 번도 실행되지 않은 것이므로, 시나리오 보강 후 스크립트를 다시 돌립니다.

| 항목 | 의미 |
|------|------|
| hintTriggered | 힌트 분석 트리거 횟수 |
| hintHadPv / hintNoPv | 힌트 후 수순 있음/없음 |
| refOpenSteps, refNext, refPrev, refJumpStart, refJumpEnd, refToggle | 참고도 재생 조작 |
| refReturnToGame, refStartGame, refLevel, refUndo, refRedo | 참고도 중 돌발 이벤트 |
| levelChange, undo, redo, jumpToStart, jumpToEnd | 대국 중 이벤트 |
| playMove, newGame | 착수·새 게임 |

---

## 2. 엔진 초기화가 발생하는 모든 이벤트

아래 이벤트에서 엔진 초기화(또는 재초기화)가 발생할 수 있습니다. 스트레스 테스트는 이 중 상당수를 자동으로 거칩니다.

| # | 이벤트 | 발생 경로 | 스트레스 테스트 |
|---|--------|-----------|:---------------:|
| 1 | 앱 최초 실행 | AppStart → ensureReady → initSession(ANALYSIS/PLAY) | ✅ (테스트 시작 시) |
| 2 | 분석 → 대국 전환 | setActiveSession(PLAY) → SwitchSession → initSession(PLAY) | ✅ (대국 진입) |
| 3 | 대국 → 분석 전환 | setActiveSession(ANALYSIS) → SwitchSession → initSession(ANALYSIS) | 수동 |
| 4 | 대국 기력(레벨) 변경 | selectLevel + startGame → ensurePlayEngineReadyForLevel | ✅ (levelChange) |
| 5 | 설정 변경 후 재시작 | ChangeConfig → restartAndEnsureReady | 수동 |
| 6 | 크래시 복구 | initEngineForRecovery | 수동/예외 시 |
| 7 | 메모리 압박 후 | onTrimMemory/onLowMemory → (재초기화 경로) | 수동 |

---

## 3. 엔진 중단이 발생하는 모든 이벤트

아래에서만 `stopAnalysis`/`stopAllAnalysis`가 호출됩니다. 그 외(착수/Undo/점프/힌트/참고도 등)에서는 중단 호출 없음.

| # | 이벤트 | 용도 |
|---|--------|------|
| 1 | ViewModel 파괴 | onCleared() → stopAllAnalysis() |
| 2 | 세션 전환 | SwitchSession(ANALYSIS↔PLAY) 시 분석 중이면 먼저 중단 |
| 3 | 메모리 압박 | onTrimMemory(CRITICAL) / onLowMemory() |
| 4 | 결과 전달 불가 | KataGoService에서 RemoteException/DeadObjectException 시 |

**검증**: 대국 모드에서 착수/Undo/점프/힌트/참고도 열기·닫기만으로는 엔진 중단이 발생하지 않아야 함(불필요한 중단·경합·간섭 없음).

---

## 4. 대국 모드 검증 체크리스트

### 4.1 기력별 200수 이상

| 기력(레벨) | 200수 이상 진행 | 봇 방문수 정상 | 방문 0 아님 | 확인 |
|------------|:---------------:|:--------------:|:-----------:|:----:|
| 0 (프로)   | ☐ | ☐ | ☐ |  |
| 1~6 (초고수) | ☐ | ☐ | ☐ |  |
| 7~30 (B6)  | ☐ | ☐ | ☐ |  |

- 스트레스 테스트: 레벨 0~30 순환, 최대 300수까지 진행 후 새 게임. 200수 구간은 자동으로 포함됨.
- 수동: 각 기력에서 200수 이상 두고, 중간에 힌트/참고도 호출 후에도 착수·봇 응답이 정상인지 확인.

### 4.2 200수 도중 힌트·참고도·수순 제공

| 단계 | 동작 | 확인 |
|------|------|:----:|
| 1 | 대국 중 힌트 분석 트리거 | ☐ |
| 2 | 힌트 결과에서 한 수 클릭 → 참고도(수순) 팝업 열림 | ☐ |
| 3 | 참고도에서 Next/Prev/Jump Start/End/재생 토글 | ☐ |
| 4 | 참고도 닫기 → 대국으로 복귀, 착수 계속 가능 | ☐ |

스트레스 테스트에서 `hintTriggered` → `onCellClicked(coord)`(참고도 열기) → 참고도 내 Next/Prev/Jump/닫기/돌발(새 게임·레벨 변경·Undo/Redo)을 반복합니다.

### 4.3 힌트 분석 중 발생 가능한 이벤트

| 이벤트 | 스트레스 테스트 | 수동 |
|--------|:---------------:|:----:|
| 참고도 Next/Prev/Jump Start/End/재생 | ✅ | ☐ |
| 참고도에서 "게임으로 돌아가기" (returnToGame) | ✅ | ☐ |
| 참고도 열린 상태에서 새 게임(startGame) | ✅ | ☐ |
| 참고도 열린 상태에서 레벨 변경 + startGame | ✅ | ☐ |
| 참고도 열린 상태에서 Undo/Redo | ✅ | ☐ |
| 힌트 트리거 직후 곧바로 닫기(returnToGame) | ✅ | ☐ |

---

## 5. 엔진·기력·파라미터 검증

| 항목 | 확인 방법 |
|------|-----------|
| 착수 정상 | 200수 이상 대국에서 흑/백 교대, 합법수만 두는지 |
| 봇 방문수 | 로그/UI에서 방문수 표시가 0이 아닌 값으로 나오는지 |
| 방문 0 회피 | 빈 결과 재시도 후에도 0이면 미준비/예외로만 발생하는지 (간섭으로 인한 0은 없어야 함) |
| 기력별 파라미터 | 레벨 0~6(B10), 7~30(B6)에서 설정된 강도/시간대로 동작하는지 |
| B10/B6 전환 | 레벨 0↔7 전환 시 대국 엔진이 해당 모델로 재초기화되는지 |

---

## 6. 요약 실행 순서

1. **클린 + 설치 + 10만 회 한 번에**  
   `.\stress_test_full_cycle.ps1`

2. **로그 보면서 완료 대기**  
   `adb logcat -s StressTest:I StressTest:W StressTest:E`  
   - `Completed steps=100000` 및 `COVERAGE [Completed]` 확인.

3. **COVERAGE에서 0인 항목**  
   있으면 해당 시나리오가 한 번도 타지 않은 것이므로, 로직/시나리오 보강 후 1번부터 재실행.

4. **수동 보완**  
   엔진 초기화/중단 이벤트(§2, §3) 중 스트레스 테스트가 안 거치는 것(세션 전환, 설정 변경, 메모리 압박 등)은 수동으로 한 번씩 확인.

이 순서대로 진행하면 **이전 데이터 없이 최신 코드만으로** 기기 스트레스 테스트 10만 회를 실행하고, 결과(완료 여부·COVERAGE)를 확인할 수 있습니다.
