# 설치 후 로그 분석 (device_log_after_install.txt, 18:06)

## 요약

- **분석 엔진(21703)** 은 3초 분석으로 **visits=133**까지 도달함. (정상)
- **대국 엔진(21822)** 도 3초 분석으로 **visits=155**까지 도달함. (정상)
- 그런데 **extractAnalysisData: zero suggestions** 가 찍히는 구간은 전부 **totalVisits=2 또는 8** 인 경우뿐임.
- 133/155 방문이 끝난 직후에는 **extractAnalysisData 경고가 없음** → 해당 런에서는 **추천이 1개 이상** 나왔을 가능성이 큼. 그런데도 화면에 안 뜬다면 **앱 쪽에서 결과를 버리거나 덮어쓴 경우**를 의심해야 함.

---

## 타임라인

| 시각 | 프로세스 | 내용 |
|------|----------|------|
| 18:06:21 | 21822 | health check 수준: totalVisits=2 → zero suggestions |
| 18:06:22 | 21703 | 분석 시작 (Time=3, Visits=1000000). 중간에 totalVisits=8 → zero suggestions |
| 18:06:25.402 | 21703 | **Search finished visits=133 elapsed=3.11s** (정상 종료) |
| 18:06:25.471 | 21523 | getTopPv: engine returned null or empty array |
| 18:06:41 | 21822 | 분석 시작 (Time=3) |
| 18:06:44.665 | 21822 | **Search finished visits=155** elapsed=3.06s |
| 18:06:45.837 | 21822 | **Shutdown requested** → NativeBridge_shutdown, PlayService destroyed |
| 18:06:45.875 | 21822 | PlayService 재생성, b6.txt 로 재초기화 |
| 18:06:46.592 | 21822 | Search finished visits=2 (health check) → zero suggestions |

---

## 진단

1. **추천 0개가 나오는 경우**  
   - 모두 **totalVisits=2 또는 8** 일 때.  
   - 네이티브는 **numVisits > 1** 인 자식만 추천으로 쓰기 때문에, 방문이 거의 안 쌓인 상태에서는 당연히 zero suggestions.

2. **133/155 방문이 끝난 뒤**  
   - 해당 프로세스에서 **extractAnalysisData 경고가 한 번도 안 찍힘** → 그 시점에는 **추천이 1개 이상** 나왔을 가능성이 큼.  
   - 그런데도 화면에 안 나온다면:
     - **결과 게이트** (requestId·epoch·movesHash 불일치)로 버려지거나
     - **RESTORE_SAVED_GAME / INIT_ENGINE** 등으로 새 의도가 제출되면서, 그 직후의 짧은 분석(2방문) 결과로 덮어씌워졌을 수 있음.

3. **대국 엔진(21822)**  
   - 18:06:45에 **한 번 완전히 shutdown** 되고, 곧바로 **b6.txt** 로 다시 뜸.  
   - 그 뒤에는 health check(visits=2)만 보임 → 대국 탭에서도 추천/착수가 안 되는 구간이 생길 수 있음.

4. **기타**  
   - `Invalid resource ID 0x00000000` → 리소스 참조 오류 (UI 일부).  
   - `getTopPv: engine returned null or empty array` → 해당 시점에 엔진/검색 결과가 비어 있음.

---

## 권장 대응

1. **추천 필터는 유지**  
   - **numVisits > 1** 은 자살수·노이즈 차단용 최소 안전장치이므로 완화하지 않음. 방문이 2 이상 쌓이도록 근본 원인을 잡아야 함.
2. **결과 적용 경로 확인**  
   - 133/155 방문이 끝난 직후에 `handleAnalysisResult` 가 호출되는지, `AnalysisResultGate.shouldProcessResult` 에서 걸러지지 않는지 로그로 확인.
3. **Play 엔진 shutdown 원인**  
   - 18:06:45 에 **Shutdown requested** 를 호출하는 쪽(세션 전환, 메모리, onCleared 등)을 코드에서 찾아, 불필요한 shutdown 이면 조건 완화.
4. **자식 노드 방문이 2 이상 쌓이지 않는 이유**  
   - 루트 totalVisits=133/155 인데 자식이 전부 ≤1 이면, 검색 파라미터·중단 시점·트리 재사용 등 네이티브/앱 쪽 원인 조사 필요.

이 문서는 `device_log_after_install.txt` (18:06 구간) 기준 분석 결과입니다.
