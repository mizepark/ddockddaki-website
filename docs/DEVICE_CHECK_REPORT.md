# 실기기 확인 보고서 (실패 요인 체크)

**수집 일시**: 2026-02-07  
**저장 위치**: `docs/device_check_logcat_app.txt`, `docs/device_check_logcat_engine.txt`, `docs/device_check_logcat_engine_play.txt`

---

## 1. ABI 확인

```bash
adb shell getprop ro.product.cpu.abi
```

**결과**: `arm64-v8a`  
**판정**: 정상. katago_jni_cpu(arm64-v8a) 로딩에 적합.

---

## 2. 네이티브 로딩 로그

**확인 방법**: `:engine` 프로세스(PID 9249) 로그에서 확인.  
(메인 앱 프로세스(8188)에는 NativeBridge/KataGoService 로그 없음 — 서비스가 별도 프로세스이기 때문)

**저장 파일**: `docs/device_check_logcat_engine.txt`

**결과**:
- `libkatago_jni_cpu.so` 로드: **ok**
- `KataGoService bound`
- `Native library loaded: katago_jni_cpu (CPU-only build)`
- `Native HOME set to: /data/user/0/com.katago.android/files/katago_home_analysis`
- `initEngine: ... b10.bin (exists=true, size=12003218)` → **NNEvaluator 초기화·Warmup 성공**
- `analyzeTop4: Search finished visits=...` → 분석 실행 정상

**판정**: 네이티브 로딩 실패 없음. 엔진 초기화·분석 호출 정상.

**참고**: 가끔 `KataGoJNI: extractAnalysisData: empty data` 로그 발생.  
→ 검색 종료 직후 getAnalysisData가 비어 있는 타이밍에서 발생 가능. Kotlin 쪽에서 빈 결과 시 재시도(emptyFinalRetryCount) 처리 있음.

---

## 3. 서비스 바인딩

**메인 앱 로그** (`device_check_logcat_app.txt`):
- `EngineService connected` (2회)
- `LifecycleServiceBinder: Binding service to application.`

**엔진 프로세스 로그** (`device_check_logcat_engine.txt`):
- `KataGoService created in process: 9249`
- `KataGoService bound`

**판정**: 바인딩 실패/죽음/타임아웃 없음. 연결 정상.

---

## 4. 모델 파일 설치 상태

```bash
adb shell run-as com.katago.android ls -l files/katago
```

**결과**:
```
total 24604
-rw------- 1 u0_a767 u0_a767      741 2026-02-07 11:45 analysis.cfg
-rw------- 1 u0_a767 u0_a767 12003218 2026-02-07 04:21 b10.bin
-rw------- 1 u0_a767 u0_a767      661 2026-02-07 11:45 play.cfg
-rw------- 1 u0_a767 u0_a767 13146385 2026-02-07 04:21 b6.txt
```

**판정**: b10.bin, b6.txt, 설정 파일 모두 존재·크기 정상.

---

## 5. 프로세스 구성

| PID  | 프로세스명                    | 비고           |
|------|-------------------------------|----------------|
| 8188 | com.katago.android            | 메인 앱(UI)    |
| 9249 | com.katago.android:engine     | KataGoService, 네이티브 로드·분석 |
| 9476 | com.katago.android:engine_play| 대국 전용 서비스 |

**네이티브/엔진 관련 로그는 `:engine`(9249)에서만 출력됨.**  
logcat 필터 시 `--pid=9249` 또는 `adb logcat -d \| findstr "9249"` 로 확인.

---

## 6. 다음에 로그 다시 수집할 때

1. **전체(앱+엔진)**:  
   `adb logcat -c` 후 앱 사용 →  
   `adb logcat -d -t 5000 > docs\logcat_full.txt`
2. **앱 프로세스만**:  
   `adb logcat -d --pid=$(adb shell pidof com.katago.android) > docs\logcat_app.txt`
3. **엔진 프로세스만** (네이티브/분석 로그):  
   `adb logcat -d --pid=$(adb shell pidof com.katago.android:engine) > docs\logcat_engine.txt`

---

## 7. 요약

| 항목           | 결과 |
|----------------|------|
| ABI            | arm64-v8a 정상 |
| 네이티브 로딩  | libkatago_jni_cpu.so 로드·초기화 성공 |
| 서비스 바인딩  | EngineService connected, KataGoService bound |
| 모델 파일      | b10.bin, b6.txt 설치·크기 정상 |

**이번 수집 기준으로는 ABI/네이티브/바인딩/모델 모두 이상 없음.**  
“앱이 제대로 안 된다”가 **분석 결과가 안 나오는 현상**이면, `extractAnalysisData: empty data` 발생 구간과 UI의 빈 결과 재시도 로직을 추가로 추적하는 것이 좋음.
