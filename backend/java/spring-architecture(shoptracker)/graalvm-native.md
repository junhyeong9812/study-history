# GraalVM Native Image

> JVM 없이 실행되는 단일 네이티브 바이너리 빌드.

## 개념

기존 Java: `.class` → JVM이 JIT 컴파일하면서 실행.
GraalVM Native Image: 빌드 시점에 **AOT(Ahead-Of-Time)** 컴파일 → OS 네이티브 실행 파일.

장점:
- 시작 시간 ~50ms (JVM ~2초)
- 메모리 ~50MB (JVM ~200MB)
- 단일 정적 바이너리 (JRE 불필요)

단점:
- 빌드 시간 길음 (3~10분)
- 동적 기능(reflection, dynamic proxy, resource loading)에 메타데이터 설정 필요
- Peak throughput은 JIT보다 약간 낮을 수 있음 (PGO로 보완 가능)

## 동작 원리

1. **Static analysis (Closed-world assumption)**: 코드 그래프를 따라가며 도달 가능한 모든 클래스/메서드를 찾음.
2. **Reachability에 없는 코드는 제거**: 사용 안 하는 라이브러리 코드가 빠짐 → 작은 바이너리.
3. **Reflection/JNI/Resource 메타데이터**: 빌드 시점에 분석 못 하는 동적 호출은 메타데이터(`reachability-metadata.json`)로 명시.
4. **AOT 컴파일**: GraalVM의 native-image 도구가 머신코드 생성.

## Spring Boot 지원

Boot 3+ 가 native build 1급 지원:
- 자동 설정 시 reflection 메타데이터 생성
- `@Bean`/`@Component` 자동 등록
- Spring AOT 단계가 빌드 전에 추가 코드 생성

```groovy
// build.gradle (PHASE6에서 추가 권장)
plugins {
    id 'java'
    id 'org.springframework.boot' version '4.0.2'
    id 'io.spring.dependency-management' version '1.1.7'
    id 'org.graalvm.buildtools.native' version '0.10.4'
}
```

빌드:
```bash
./gradlew nativeCompile
./build/native/nativeCompile/shoptracker
```

## 이 프로젝트의 잠재 이슈

| 영역 | 이슈 | 대응 |
|------|------|------|
| Hibernate proxy | reflection 다용 | Spring Boot가 자동 처리 |
| `@JdbcTypeCode(SqlTypes.JSON)` | reflection | hint 추가 필요할 수 있음 |
| Lombok | annotation processor | 빌드 타임이라 OK |
| Spring Modulith Scenario | 테스트만 | native 빌드에는 영향 없음 |
| OpenTelemetry agent | byte-code instrument | `spring-boot-starter-opentelemetry`는 빌드타임 통합이라 OK |

## 비교표

| | JVM (JRE) | GraalVM Native |
|---|-----------|----------------|
| 이미지 크기 (Docker) | ~300MB | ~80MB |
| 시작 시간 | ~2초 | ~50ms |
| 메모리(idle) | ~200MB | ~50MB |
| Peak throughput | 100% (JIT 워밍업 후) | ~85~95% |
| 빌드 시간 | ~30초 | ~5분 |
| Reflection | 완벽 | 메타데이터 필요 |
| 디버깅 | 풍부 | 제한적 |

## 언제 쓸 가치가 있나

- **서버리스/Lambda**: cold start가 비용 → 50ms 시작이 결정적
- **CLI 도구**: 사용자가 빠른 시작 기대
- **마이크로서비스 + autoscaling**: 새 인스턴스 시작이 잦음
- **메모리 비용 민감**: 컨테이너 메모리 절약

쓰지 말아야 할 때:
- 학습/내부 도구 — 빌드 시간이 개발 속도를 깎음
- Peak throughput 이 절대 우선 — JIT가 살짝 빠를 수 있음
- 동적 코드(스크립팅, 플러그인 로딩)가 필수

## 이 프로젝트에서

`Dockerfile.native` 가 PHASE6에 포함되어 있지만 권장은 **JVM이미지로 학습** + **네이티브는 체험**. 학습 프로젝트의 본질에서 벗어나기 쉽다.

## FastAPI/Python 대응

직접 대응 없음. Python은 인터프리터 기반으로 native binary가 일반적이지 않음(Nuitka, PyInstaller 정도). 시작 시간 측면에서는 Python이 JVM보다 빠르지만, 메모리/throughput에서는 GraalVM Native가 더 유리한 영역이 많다.
