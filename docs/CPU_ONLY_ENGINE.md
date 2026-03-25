CPU-only 엔진 구성 요약

- Android 앱 네이티브 빌드는 Eigen(CPU) 백엔드만 사용한다.
- OpenCL/CUDA/Metal 등 GPU 백엔드는 원본 KataGo 소스(katago_src)에 구현이 포함돼 있어도, 앱 빌드 경로에서는 컴파일/링크하지 않는다.
- 이 리포지토리의 katago_src는 업스트림 원본 동기화/참조를 위한 보관 영역이며, 실제 APK에 포함되는 엔진은 app/src/main/cpp/CMakeLists.txt에서 빌드되는 타깃에 의해 결정된다.

근거 파일

- Android 네이티브 빌드 타깃: app/src/main/cpp/CMakeLists.txt
- JNI 백엔드 라벨: app/src/main/cpp/katago_jni.cpp
- 업스트림 백엔드 분기 정의(미사용): app/src/main/cpp/katago_src/cpp/CMakeLists.txt
