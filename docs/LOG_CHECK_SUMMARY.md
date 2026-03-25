# 로그 확인 요약

## 1. 최근 기기 로그 (02-06 16:31 기준, 클린 설치 후)

### 정상 동작
- **분석 실행:** `triggerAnalysis start mode=QUICK`, `AnalysisUseCase parseRawResult: finalSuggestions=14~15`, `totalVisits=119~134`
- **결과 반영:** `EngineViewModel handleAnalysisResult: TopSuggestion winrate ... pvSize=6`, `EngineOrchestrator process state=Ready event=AnalysisDone`
- 엔진 분석 → 결과 파싱 → UI 반영 → AnalysisDone 까지 흐름 정상

### 경고 (기능 저하 가능)
- **PV 빈 경우:** `EngineRepository getTopPv: engine returned null or empty array` → `getTopPv: returning 15 PVs, first pvSize=0`  
  → 일부 추천수의 PV(참고도 수순)가 비어 있을 수 있음. 해당 추천 클릭 시 "이 추천수에는 참고도가 없습니다" 메시지 표시하는 수정 반영됨.

### 시스템/외부 라이브러리 (앱 코드 수정 불필요)
- `MediaCodec: Media Quality Service not found.` — 미디어 코덱
- `Failed to query component interface for required system resources: 6` — CCodec/시스템
- ExoPlayer/Ads/CCodec 관련 로그 — 광고/미디어

---

## 2. 이전 캡처 로그 (device_play_log_current.txt, 02-04)

### 반복되는 에러 (앱이 아닌 쪽 가능성 큼)
- **Invalid resource ID 0x00000000** — 여러 시점에서 출력. 앱의 `getString(R.string.xxx)`는 상수 리소스만 사용하므로, 광고/Google Play Services 등 다른 라이브러리에서 리소스 ID 0 사용 가능성.
- **No package ID 6a found for resource ID 0x6a0b0013** — 다른 패키지(예: Google Play Services) 리소스 참조 시 흔히 나오는 메시지.

### 앱 동작 관련
- **Locale not set, deferring engine initialization** — 로케일 미설정 시 엔진 초기화를 미루는 의도된 동작.
- **triggerAnalysis start mode=QUICK / DEEP** — 분석 모드 전환 및 실행 로그 정상.
- **RAM Profile: CPU MODE** — CPU 모드로 동작.

### 기기/GPU (앱 수정 대상 아님)
- **GPUAUX: Null anb** — GPU/컴포저
- **[perfctl] FPSGO** — 기기 벤더 성능 제어

---

## 3. 결론

- **분석·추천·오케스트레이터:** 최근 로그상 정상 동작.
- **참고도 PV 없음:** 엔진이 PV를 비워 주는 경우가 있어, 해당 시 사용자 메시지 처리만 앱에서 추가한 상태.
- **Invalid resource ID / No package ID 6a:** 앱 리소스가 아닌 외부 라이브러리/시스템 쪽 이슈로 보는 것이 타당함.

추가로 확인할 때는 다음으로 앱 로그만 보면 됨:
```bash
adb logcat -s EngineViewModel:* EngineOrchestrator:* AnalysisUseCase:* analysis_state_machine:*
```
