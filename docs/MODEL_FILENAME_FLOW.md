# 앱이 정상 동작하려면: 모델 파일 이름이 맞아야 하는 흐름

이름을 **한 군데만** 보면 되도록, 실제 코드 흐름 기준으로만 적음.

---

## 1. 정해진 이름 (이걸로 통일)

| 역할 | 파일 이름 | 쓰는 곳 |
|------|-----------|--------|
| 에셋(압축, 읽는 소스) | **b10.bin.gz** | APK/에셋 `katago/` |
| 에셋(압축, 읽는 소스) | **b6.txt.gz** | APK/에셋 `katago/` |
| 디스크(설치 결과 = 엔진이 로드) | **b10.bin** | `filesDir/katago/b10.bin` |
| 디스크(설치 결과 = 엔진이 로드) | **b6.txt** | `filesDir/katago/b6.txt` |

B6은 반드시 **b6.txt** (`.bin` 아님). 네이티브가 .txt만 받음.

---

## 2. 앱이 이 이름을 쓰는 순서 (실제 동작)

1. **에셋에 있어야 하는 것**  
   `katago/b10.bin.gz`, `katago/b6.txt.gz`  
   → 없으면 `prepareModel`에서 예외.

2. **설치 단계 (prepareModel)**  
   - 위 두 .gz를 열어서 GZIP 해제.  
   - 결과를 **각각** `filesDir/katago/b10.bin`, `filesDir/katago/b6.txt` 에 저장.  
   → 여기서 쓰는 “설치 결과 이름”이 **b10.bin**, **b6.txt** 두 개뿐.

3. **준비 여부 확인 (isModelReady)**  
   - `filesDir/katago/b10.bin` 존재·크기·읽기 가능  
   - `filesDir/katago/b6.txt` 존재·크기·읽기 가능  
   → 둘 다 만족해야 “모델 준비됨”.

4. **엔진이 여는 파일**  
   - 분석: `getModelFile()` → `.../katago/b10.bin`  
   - 대국 0~6: `getModelFileForPlayLevel()` → `.../katago/b10.bin`  
   - 대국 7~30: `getModelFileForPlayLevel()` → `.../katago/b6.txt`  
   → 즉, 엔진은 **항상 b10.bin 또는 b6.txt** 만 연다.

---

## 3. 코드에서 이름이 나오는 곳 (전부 여기만 맞으면 됨)

- **AssetInstaller**  
  - 에셋 경로: `katago/b10.bin.gz`, `katago/b6.txt.gz`  
  - 설치 결과 이름: `b10.bin`, `b6.txt` (Triple 두 번째 값, destFilename)  
  - getModelFile: `"b10.bin"`  
  - isModelReady: `"b10.bin"`, `"b6.txt"`  
  - 정리용 delete: `b10.bin.gz`, `b6.txt.gz` (에셋 복사본 삭제 시)
- **PlayLevelModelPolicy**  
  - B10: `"b10.bin"`, B6: `"b6.txt"`  
  - getModelFileForPlayLevel은 이걸 쓰므로, 엔진에 넘기는 경로도 b10.bin / b6.txt.

에셋에는 **b10.bin.gz, b6.txt.gz** 만 두고,  
설치·검사·엔진 로드는 전부 **b10.bin, b6.txt** 로 맞추면 앱이 정상 동작하는 구조다.

---

## 4. 잘못 쓰면 생기는 일

- 에셋에 `b6.bin.gz` 만 있고 `b6.txt.gz` 가 없으면 → 설치 단계에서 “필수 에셋 없음”으로 실패.
- 설치 결과를 `b6.bin` 으로 쓰면 → 대국 7~30은 `b6.txt` 를 찾으므로 “파일 없음” / 엔진 초기화 실패.
- getModelFile / PlayLevelModelPolicy 에서 `b6.bin` 이라 적으면 → 실제 설치 파일은 `b6.txt` 라서 엔진이 못 찾음.

정리: **에셋 = b10.bin.gz, b6.txt.gz / 디스크·엔진 = b10.bin, b6.txt** 로 통일되어 있으면 정상 동작한다.
