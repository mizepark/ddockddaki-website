# 로그 전부 수집 방법

앱은 **모든 우선순위(V/D/I/W/E)** 를 logcat으로 출력하고, **app.log** 파일에도 동일 로그를 기록함 (`AppLogger.enableFileLogging`).

## app.log 위치 및 가져오기

- **경로**: 앱 전용 외부 저장소 `Android/data/com.katago.android/files/logs/app.log`
- **adb로 가져오기** (기기 연결 후):
  ```bash
  adb pull /sdcard/Android/data/com.katago.android/files/logs/app.log .
  ```
  또는 (기기마다 다를 수 있음):
  ```bash
  adb shell run-as com.katago.android cat files/logs/app.log > app.log
  ```
  외부 저장소를 쓰면 run-as 없이 pull 가능.

## 이번 설치 로그 전부 잡기

1. **설치 직후 기기 logcat 버퍼 비우기**
   ```bash
   adb logcat -c
   ```

2. **앱 실행 후 재현·동작**

3. **전부 덤프 (원인 분석용)**
   ```bash
   adb logcat -d -v time > device_log_full.txt
   ```
   또는 라인 제한:
   ```bash
   adb logcat -d -v time -t 50000 > device_log_full.txt
   ```

4. **앱·엔진 관련만 필터 (선택)**
   ```bash
   adb logcat -d -v time | findstr /i "Katago com.katago.android KataGoJNI EngineViewModel PlayMode PlayViewModel Timber StressTest NativeBridge ensureConfigFile analyzeTop4 isLegal initEngine healthCheck"
   ```

## 실시간으로 보면서 저장

```bash
adb logcat -c
adb logcat -v time | tee device_log_live.txt
```

- `Ctrl+C`로 중단 시 `device_log_live.txt`에 그때까지 로그가 저장됨.
