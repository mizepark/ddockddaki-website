# 개인정보처리방침 호스팅 가이드 (Hosting Guide)

이 폴더(`docs`)에는 Google Play Console 등록에 필요한 **개인정보처리방침(Privacy Policy)** 파일이 포함되어 있습니다.
GitHub Pages를 사용하여 이 파일을 무료로 호스팅하고 URL을 생성할 수 있습니다.

## 1단계: GitHub에 업로드
이 프로젝트를 GitHub 리포지토리에 푸시(Push)합니다.
(이미 GitHub에 코드가 있다면, 이 `docs` 폴더가 포함되도록 커밋하고 푸시하세요.)

## 2단계: GitHub Pages 설정
1. GitHub 리포지토리 페이지로 이동합니다.
2. 상단 메뉴의 **Settings(설정)** 탭을 클릭합니다.
3. 왼쪽 사이드바에서 **Pages**를 클릭합니다.
4. **Build and deployment** 섹션의 **Source**에서 `Deploy from a branch`를 선택합니다.
5. **Branch** 설정에서:
   - 브랜치: `main` (또는 `master`)
   - 폴더: `/docs` (중요: `/ (root)`가 아닌 `/docs`를 선택하세요)
6. **Save(저장)** 버튼을 클릭합니다.

## 3단계: URL 확인 및 Play Console 등록
1. 설정 저장 후 약 1~2분 뒤, 상단에 **"Your site is live at..."** 메시지와 함께 URL이 표시됩니다.
2. 생성된 URL 형식은 보통 다음과 같습니다:
   `https://<사용자이름>.github.io/<리포지토리이름>/privacy_policy.html`
   (참고: `/docs`를 소스로 지정했으므로, 파일명 `privacy_policy.html`을 URL 뒤에 붙여야 할 수 있습니다. 만약 `index.html`이 없다면 파일명을 명시해야 합니다.)
3. 이 URL을 복사하여 **Google Play Console -> 앱 콘텐츠 -> 개인정보처리방침** 섹션에 붙여넣고 저장합니다.
