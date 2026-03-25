# 분석 모드 추천수 안 나올 때 로그 확인

분석 모드에서 추천수(후보수)가 안 나오면 아래 로그로 원인을 좁힐 수 있습니다.

## 1. 로그 수집 (기기 연결 후)

```bash
adb logcat -c
# 앱에서 분석 탭 열고, 바둑판 터치해 분석 한 번 돌린 뒤
adb logcat -d | findstr /i "AnalysisResultGate parseRawResult Analysis empty AnalysisResult HintDebug Analysis Metrics"
```

PowerShell:

```powershell
adb logcat -c
# 앱에서 분석 실행 후
adb logcat -d | Select-String -Pattern "AnalysisResultGate|parseRawResult|Analysis empty|AnalysisResult|HintDebug|Analysis Metrics"
```

## 2. 로그 의미

| 로그 (일부) | 의미 |
|-------------|------|
| **AnalysisResultGate: DROPPED** | 결과가 게이트에서 걸림. `reason=` 확인. |
| `reason=requestId_mismatch` | 다른 분석 요청 결과가 도착함(이전 요청 결과 폐기). |
| `reason=epoch_stale` | 새 분석이 이미 시작돼 이전 세대 결과로 폐기됨. |
| `reason=movesHash mismatch` | 수순이 바뀐 뒤 도착한 결과(유령 추천 방지로 무시). |
| **AnalysisResult: movesHash mismatch (ghost)** | 수순 불일치로 추천 비움. |
| **Analysis empty suggestions** | 최종 결과에 suggestions가 0개. totalVisits, mode 확인. |
| **parseRawResult: OLD format used** | 네이티브가 59 float 포맷이 아님. 빌드/엔진 불일치 가능. |
| **parseRawResult: finalSuggestions=0 but suggestionsRaw=** | 파싱은 됐지만 minVisits 필터로 전부 제거됨. |
| **parseRawResult: outSize=** | 네이티브에서 받은 float 개수. 0이면 엔진 빈 결과. |

## 3. 자주 나오는 원인

- **게이트 DROPPED (requestId/epoch)**  
  분석 중에 화면 전환이나 빠른 터치로 새 분석이 시작되면 이전 결과는 버려짐. 정상 동작.
- **movesHash mismatch**  
  분석 도중 착수/되돌리기로 수순이 바뀌면 해당 결과는 적용 안 함. 정상 동작.
- **totalVisits=0 또는 빈 배열**  
  엔진 미준비, 예외, 또는 분석 중단으로 빈 결과가 온 경우.  
  `parseRawResult: outSize=0` 또는 `n=0`이면 네이티브가 빈 배열 반환.
- **OLD format**  
  앱은 59 float/후보 포맷을 기대하는데 다른 크기가 오면 구 포맷으로 파싱.  
  네이티브(katago_jni)와 Kotlin 포맷이 맞는지 확인.

위 로그까지 수집해 두면 "왜 이번에만 추천이 안 나왔는지" 추적할 수 있습니다.

---

## 4. 봇이 분석을 안 하는 이유 (대국 모드)

**원인**: 분석 탭에서 분석이 돌아가는 중에 **대국 탭으로 전환**하면, 오케스트레이터만 job을 취소하고 **ViewModel 쪽 `analysisJob`과 `pendingAnalysis`는 그대로** 남음. 그러면 `getRunnerState()`가 `isAnalysisRunning == true` 또는 `pendingAnalysis == true`를 반환하고, 스케줄러가 **새 의도(대국 국면 분석)를 계속 거절**해서 봇 턴에서 분석이 한 번도 안 돌고, `isBotMoveReady`도 false로만 남음.

**수정**: 세션 전환 시 `onSessionSwitched`에서 **ViewModel의 `analysisJob?.cancel()`, `analysisJob = null`, `pendingAnalysis = false`** 를 수행하도록 함. 전환 후 새 세션에서 제출된 분석 의도가 정상적으로 RunNow로 실행됨.

**로그**: 전환 시 이전에 분석이 살아 있었으면 `SessionSwitch: cancelling stale analysis job and clearing pendingAnalysis for new session=PLAY` 가 보임.

---

## 5. 설치 전에 오류 미리 잡기 (세션 전환·분석 수락)

기기에 설치한 뒤에만 발견되는 타이밍/탭 전환 오류를 **빌드·테스트 단계에서** 줄이려면:

1. **단위 테스트 실행**  
   `AnalysisSchedulerSessionSwitchTest` 가 스케줄러 계약을 검증한다:
   - runner state가 비어 있으면(`isAnalysisRunning=false`, `pendingAnalysis=false`) → `decide()`는 **RunNow** (새 세션 의도 수락).
   - state가 살아 있으면 → **Defer** (중복 방지).  
   스케줄러 로직을 바꿔서 "state 비어 있어도 Defer" 하면 이 테스트가 실패하므로, **설치 전에** `./gradlew :app:testDebugUnitTest --tests "com.katago.android.engine.AnalysisSchedulerSessionSwitchTest"` 로 확인할 수 있다.

2. **ViewModel 정리 코드 유지**  
   위 테스트는 "스케줄러가 state 비었을 때 수락하는지"만 검증한다. **세션 전환 시 ViewModel이 state를 비우는지**는 이 테스트로는 잡지 못한다. 따라서 `EngineViewModel.onSessionSwitched` 안의 `analysisJob?.cancel()`, `analysisJob = null`, `pendingAnalysis = false` 는 **삭제하거나 건너뛰면 안 된다.** (회귀 시 다시 "탭 전환 후 봇이 분석 안 함"이 기기에서만 재현된다.)

3. **추가로 하고 싶을 때**  
   "세션 전환 후에도 반드시 분석 의도가 수락된다"를 자동으로 검증하려면, `EngineViewModel`(또는 테스트용 하네스)에 세션 전환 콜백을 주입해 호출한 뒤, `getRunnerState()`가 `(false, false)`를 반환하는지 검사하는 테스트를 추가할 수 있다. 그 경우 ViewModel이 테스트용 runner state getter를 노출하거나, 의도 제출 → `onRunRequested` 호출 여부로 간접 검증해야 한다.

**전환·기능 변환 전체**에 대한 테스트 계획과 **추천수/봇 실패 시 자동 재시도** 정리는 → [TRANSITIONS_AND_ANALYSIS_TEST_PLAN.md](TRANSITIONS_AND_ANALYSIS_TEST_PLAN.md) 참고.
