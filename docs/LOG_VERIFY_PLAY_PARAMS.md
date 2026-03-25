# 대국 모드 로그로 확인하는 항목

코드가 아니라 **기기 logcat**으로 아래 항목이 실제 적용되는지 확인할 수 있습니다.

---

## 1. 로그 수집 방법

1. **로그 비우기**  
   ```bash
   adb logcat -c
   ```
2. **재현**  
   앱 실행 → 인공지능과 대국 → 레벨 선택 → **대국시작** → 봇이 한 수 이상 두기까지 진행
3. **덤프**  
   ```bash
   adb logcat -d -t 8000 > device_log.txt
   ```
4. **앱 관련만 필터** (PowerShell)  
   ```powershell
   cd d:\Mr_Ba_teach\scripts
   .\capture_katago_log.ps1 -Lines 8000 -OutFile "device_katago_log.txt"
   ```

---

## 2. 로그에서 확인할 내용

### (1) 대국 봇에 적용된 파라미터

**태그:** `PlayMode`  
**메시지 예:**  
`startGame: levelIndex=... levelLabel=... humanColor=... botIsBlack=... timeRange=... randTemp=... avoidUtility=... scoreUtility=...`

- `levelIndex` / `levelLabel`: 선택한 기력
- `randTemp`: chosenMoveTemperature (프로=0, 그 외 구간별 값)
- `avoidUtility`, `scoreUtility`: 반복패턴/점수 유틸리티

**태그:** `Timber` (클래스명으로 나올 수 있음)  
**메시지 예:**  
`play.cfg written: chosenMoveTemp=... wideRootNoise=... cpuct=... cpuctLog=... cpuctBase=... avoidRepeatUtility=... staticScoreUtility=... maxVisits=...`

- 대국용 config(play.cfg)에 실제로 쓴 값들
- `chosenMoveTemp`=0 이면 프로, 0보다 크면 비프로 구간

### (2) 기력이 낮을 때 낮은 승률 수를 고르는 로직

**태그:** `PlayMode`  
**메시지 예:**

- **프로(level 0):**  
  `selectBotMove: level=0 pool=... pick=best winrate=...`  
  → 항상 best(최고 승률) 선택
- **최저기력(level 30):**  
  `selectBotMove: level=30 tier=최저기력 pool=... pick=lowest winrate=...`  
  → 최저 승률 위주 풀에서 선택
- **고수1~ (level 7~29):**  
  `selectBotMove: level=... tier=고수1~ pool=... pick winrate=...`  
  → 구간별 완만한 승률
- **초고수(level 1~6):**  
  `selectBotMove: level=... tier=초고수 pool=... pick winrate=...`  
  → 빠르게 강한 수 쪽으로

로그에 `tier=최저기력` / `pick=lowest` / `pick winrate=0.xx` 등이 보이면, 기력별 승률 선택 로직이 **실제로 실행된 것**으로 볼 수 있습니다.

### (3) 현재 쓰는 엔진

**태그:** `Timber` (EngineViewModel 등)  
**메시지 예:**

- `Engine config select: session=PLAY cpu=... lowMem=... selected=play.cfg`  
  → 대국 시 **play.cfg** 사용
- `Engine session=PLAY (PLAY=대국엔진 ANALYSIS=분석엔진) model=...`  
  → 대국 엔진(PLAY) + 사용 중인 모델 파일명
- `Model file=... length=...`  
  → 실제 로드된 모델 파일명 (b10.bin / b6.txt 등)
- `initializeEngine done config=play.cfg modelFile=... modelLabel=...`  
  → 초기화 완료 시 config=play.cfg, 모델명 확인

**서비스 구분:**  
- 대국: `KataGoPlayService` / `playEngineController` → 로그상 `session=PLAY`, `config=play.cfg`  
- 분석: `KataGoService` / analysis 엔진 → `session=ANALYSIS`, `config=analysis.cfg`

---

## 3. grep 예시 (수집한 device_log.txt 기준)

```bash
# 대국 시작 시 파라미터
grep "startGame:" device_log.txt

# 봇 수 선택 (기력/승률)
grep "selectBotMove:" device_log.txt

# play.cfg 기록된 파라미터
grep "play.cfg written:" device_log.txt

# 엔진 세션·모델
grep "Engine session=\|Model file=\|Engine config select:\|initializeEngine done" device_log.txt
```

PowerShell:

```powershell
Select-String -Path device_log.txt -Pattern "startGame:|selectBotMove:|play.cfg written:|Engine session=|Model file=|Engine config select:"
```

---

정리하면, **코드가 아니라 로그에서** 다음을 확인할 수 있습니다.

- 대국 봇 파라미터: `startGame:` + `play.cfg written:`
- 기력 낮을 때 낮은 승률 선택: `selectBotMove:` 의 `tier=` / `pick=` / `winrate=`
- 현재 엔진: `Engine session=PLAY`, `Model file=...`, `config=play.cfg`

재현 후 위 순서대로 logcat을 떠서 grep/필터하면 됩니다.
