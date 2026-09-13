---
title: JDK 버전 업 제안
pubDatetime: 2022-09-15T09:00:00+09:00
description: JDK 8에서 11 이상으로 올려야 하는 이유(성능, GC, 주요 변경점)와 업그레이드 시 고려사항, jdeps/jdeprscan 도구 사용법을 정리합니다.
tags:
  - tech-share
  - proposal
  - java
---

## 개요

여기어때 기술블로그에서 해당 포스팅을 보고 자바의 버전별 세부 업데이트 사항에 대해 알아야 Java에 대해 보다 더 잘 이해할 수 있을 것이라 생각을 하게 되었습니다.

- 참고: [여기어때 기술블로그 - 우리팀이 JDK 17을 도입한 이유](https://techblog.gccompany.co.kr/%EC%9A%B0%EB%A6%AC%ED%8C%80%EC%9D%B4-jdk-17%EC%9D%84-%EB%8F%84%EC%9E%85%ED%95%9C-%EC%9D%B4%EC%9C%A0-ced2b754cd7)

찾아보니 왜 JDK 8에서 11 버전 이상으로 올려야만 하는지에 대해 구체적인 이유를 확인할 수 있었습니다. 이번 글은 왜 JDK 버전을 11 이상으로 올려야 하는지 이유와 고려사항에 대해 정리해 보았습니다.

## 왜 OpenJDK 8을 사용하는가?

가장 먼저 왜 대부분의 회사에서 JDK 8을 사용하고 있는지에 대해 생각해봐야 합니다. Oracle이 제공하는 아래 로드맵을 보시면 바로 이해할 수 있습니다.

![](/images/jdk-version-upgrade-proposal/img-01.png)

2022년이 아닌 **과거 시점**에서 봤을 경우 LTS가 가장 긴 OpenJDK8 버전을 이용하는 것이 운영적인 측면에서 타 JDK를 사용하는 것보다 가장 안정적인 선택이었습니다.

또한 국내에서 개발된 프로젝트는 대다수 Java 8로 개발되어 운영하고 있는 상황이기에 **제품들과의 호환성을 유지하고 안정적으로 운영하기 위해서 JDK 8을 사용**하고 있는 경우도 많을 것입니다.

회사에서 사용할 OpenJDK는 장기적인 측면으로 보았을 경우 **LTS가 최대한 오래 보장되는 버전**을 선택하는 것이 가장 바람직하기에 현재 선택할 수 있는 대안은 다음과 같습니다.

### 선택 가능한 Open JDK 버전

- OpenJDK 11 (2019 ~ 2026.09)
- OpenJDK 17 (2022 ~ 2029.09)

이 글은 **왜 JDK 11 버전 이상으로 업그레이드를 해야 하는가**에 대한 내용을 다루기 때문에 JDK 11 버전으로 업그레이드 할 경우 업데이트되는 내용 및 고려사항에 대해 중점적으로 다루어 보겠습니다.

### 성능 비교

성능비교는 LinkedIn의 전환 결과를 참고하였습니다.

성능 테스트 결과, LinkedIn 서비스는 Java 11로 전환하면서 성능이 저하된 사례는 **하나도 없었습니다**.

최상의 경우에는 약 **200%의 성능향상**을 보였으며 성능 향상은 대부분 GC의 변경에서 발생했습니다.

#### 지연시간 및 처리량 비교

![](/images/jdk-version-upgrade-proposal/img-02.png)

![](/images/jdk-version-upgrade-proposal/img-03.png)

![](/images/jdk-version-upgrade-proposal/img-04.png)

JDK 버전 업그레이드 전과 후를 비교했을 경우 처리량, 지연 시간 등 월등히 개선되는 것을 확인할 수 있었습니다.

어떠한 개선 사항이 있었길래 다음과 같은 성능 개선이 되었는지 JDK별 Release 정보 확인을 통해 확인해 봤습니다.

## 버전별 업데이트 사항

JDK 8에서 JDK 11까지는 버전별로 정말 많은 업데이트 사항이 존재하기 때문에 High-Level의 변경 사항 중 확인해볼 내용들을 일부 정리하였습니다.

Release 노트를 전체적으로 확인하고 싶으신 분이 있는 경우 다음 링크를 참고해주시면 감사하겠습니다.

### 참고

- [JDK 9 Release Note](https://www.oracle.com/java/technologies/javase/9-relnotes.html)
- [JDK 10 Release Note](https://www.oracle.com/java/technologies/javase/10-relnotes.html)
- [JDK 11 Release Note](https://www.oracle.com/java/technologies/javase/11-relnotes.html)

### JDK 9

#### 1. Module System

- Java 플랫폼을 모듈화하여 필요한 모듈만으로 경량화된 이미지를 만들 수 있게 되었습니다.
- 기존에는 JRE 일부분만 배포가 불가능했지만 Jigsaw를 통하여 **원하는 모듈만 모아 런타임 환경 이미지**를 만들 수 있게 되었습니다. (마이크로 서비스, 시작 시간 단축 등을 고려)

#### 2. Change default GC to G1

- **default GC**가 **Parallel GC -> G1 GC**로 변경되었습니다.

#### 3. Compact Strings

- java.lang.String, StringBuilder and StringBuffer 에서는 **UTF-16 문자 배열**에서 **(바이트 배열 + 1바이트 인코딩 flag field)** 로 변경되었습니다.
- 문자열의 내용을 기반으로 ISO-8859-1/Latin-1(문자당 1바이트) 또는 UTF-16(문자당 2바이트)으로 문자를 저장/인코딩한다.
- **String 객체가 문자를 저장하는 데 필요한 공간을 50%까지 줄일 수 있습니다.** (ex) 영어만 있을 경우)
- 비활성화하기 위해 새로운 jvm 옵션 -XX:-CompactStrings을 이용할 수 있다.

#### 4. Deprecation of Boxed Primitive Constructors

- 기존에 명시되어 있을 경우 이후 Primitive 생성자 사용 중단
  - intList.add(new Integer(347)); -> intList.add(347);
  - intList.remove(347); -> intList.remove(Integer.valueOf(347));

#### 5. StackWalker

- 특정 스레드를 로깅할 경우 스택의 스냅샷을 가져옴
- **특정 예외에 대해서만 로그를 남기도록 세분화된 제어 기능 제공**

#### 6. System class loader is No more In Java9

- 클래스 로더가 내부 클래스로 제공되며, **더 이상 인스턴스로 제공되지 않습니다.**

### JDK 10

#### 1. Improvements for docker containers

- Java 10 버전 이전에는 **CPU 제한 설정이 컨테이너 내부에서 적용이 안되는 문제**가 있었는데 이를 개선했습니다.
- JDK 8은 **jdk8u191 버전 이후로 개선이 되어 백포트**되었다.

#### 2. Local-variable type inference

- **var 변수 사용 가능**

#### 3. Add Optional.orElseThrow()

### JDK 11

#### 1. Add ZGC to GC List

- 저지연 수집기 추가 (JDK 11에서는 실험용으로 사용하여 비추천)

#### 2. Unified logging

- **JVM의 모든 구성요소에 대해 공통적인 로깅 시스템을 제공**
- 어떤 구성요소를 어떤 log level로, stack trace할 것인지 설정이 가능하다.
- JVM 충돌시 문제 원인을 진단하는데 사용이 가능

#### 3. Multi-release jar files

- **버전별 jar file로 릴리즈가 가능**

#### 4. Reactive Stream

- Non-Blocking Backpressure를 이용한 **비동기 스트림 처리 지원 API 추가**

#### 5. Collection Factory Method 기능 강화

- List, Set, Map 인터페이스에 immutable 생성을 할 수 있는 새로운 Method 추가

이 외에도 부가적인 추가 기능들이 추가/개선되었으니 더 자세히 변경 사항을 알고 싶으시면 위의 릴리즈 노트 링크를 참고해주시면 감사하겠습니다.

이제 다음과 같은 변경사항들로 인해 version migrate간 발생할 수 있는 문제가 여러 개가 존재하게 됩니다. 어떤 요소들을 고려하면서 버전을 업그레이드 해야되는지 알아보겠습니다.

## 업그레이드시 고려 사항

### 1. Compact String

- Compact String으로 인해 마이그레이션시 성능 저하가 크게 발생할 수 있습니다.
- String, StringBuffer, StringBuilder 클래스에서 성능 저하가 발생하는지 확인이 필요합니다.

### 2. Default GC의 변경

- JDK 8에서 11로 변경되면서 default GC가 변경되었습니다.
- 해당 부분 확인하여 변경 이후 영향도가 있는지 파악이 필요합니다.

### 3. @Deprecated 및 사용하지 않는 패키지 확인

- 버전 업그레이드시 @Deprecated와 javadoc이 충돌해 컴파일 에러가 발생할 수 있습니다.
- javadoc 및 @Deprecated 사용 여부를 확인해봐야 합니다.

### 4. 타사 라이브러리 확인

- JDK 11과 호환되지 않는 문제가 발생할 수 있습니다.
- 라이브러리 업데이트가 필요하며, 호환되는 버전이 없을 경우 직접 개발이 필요할 수도 있습니다.

### 5. ClassLoader 변경

- 클래스 로더 계층 구조가 Java 11에서부터 내부 클래스로 변경되어 캐스팅시 런타임간 `ClassCastException`이 발생할 수 있습니다.
- `ClassLoader.getPlatformClassLoader()`를 통해 부모 클래스 로더를 가져와야 할 상황이 발생할 수 있으니 주의해야 합니다.

### 6. Locale data 변경

- JDK 8에서는 기본적으로 활성화되어 있지 않은 기능이어서 지원되지 않는 로케일에 대해 다르게 작동할 수 있습니다.
- 해당 부분 테스트 확인 후 코드 수정이 필요합니다.

### 7. ClassCastException

- 버전 업그레이드시 checkcast가 좀 더 엄격해져 클래스 캐스팅 관련 에러가 발생할 수 있습니다.

### 8. Remove JAXB

- JDK 11 버전은 JAXB가 제거된 버전입니다. 따라서 기존에 동작하던 코드에서 ClassNotFound가 발생할 수 있습니다.
- 이 경우 다음 의존성을 추가하거나, 어떤 곳에서 발생했는지 확인을 통해 수정이 필요합니다.

```xml
<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
</dependency>
```

- 버전 업그레이드시 삭제된 dependency는 다음과 같습니다.
  - **JAF**: with com.sun.activation:javax.activation
  - **CORBA**: there is currently no artifact for this
  - **JTA**: javax.transaction:javax.transaction-api
  - **JAXB**: com.sun.xml.bind:jaxb-impl
  - **JAX-WS**: com.sun.xml.ws:jaxws-ri

## 버전 전환시 도움이 될만한 도구

Java 11에는 버전 업그레이드를 도와주는 두 가지 플러그인을 제공하고 있습니다.

**제공 플러그인**

- jdeprscan: 더 이상 사용되지 않거나 제거된 API 목록 제공
- jdeps: 어떤 클래스가 어떤 내부 클래스에 의존하는지 알 수 있는 종속성 분석기

각각의 플러그인을 어떻게 사용하고 어떤 내용을 알려주는지 확인해보겠습니다.

### jdeps

- 버전 업그레이드시 발생하는 변경되는 API 수정 목록 관련 제안을 해주는 기능을 제공하는 플러그인.

```text
jdeps --jdk-internals --multi-release 11 --class-path log4j-core-2.13.0.jar my-application.jar
Util.class -> JDK removed internal API
Util.class -> jdk.base
Util.class -> jdk.unsupported
   com.company.Util        -> sun.misc.BASE64Encoder        JDK internal API (JDK removed internal API)
   com.company.Util        -> sun.misc.Unsafe               JDK internal API (jdk.unsupported)
   com.company.Util        -> sun.nio.ch.Util               JDK internal API (java.base)

Warning: JDK internal APIs are unsupported and private to JDK implementation that are
subject to be removed or changed incompatibly and could break your application.
Please modify your code to eliminate dependence on any JDK internal APIs.
For the most recent update on JDK internal API replacements, please check:
<https://wiki.openjdk.java.net/display/JDK8/Java+Dependency+Analysis+Tool>

JDK Internal API                         Suggested Replacement
----------------                         ---------------------
sun.misc.BASE64Encoder                   Use java.util.Base64 @since 1.8
sun.misc.Unsafe                          See <http://openjdk.java.net/jeps/260>
```

### jdeprscan

jdeprscan을 통해 JDK 11 버전에서 더 이상 사용하지 않는 API 목록을 확인할 수 있습니다.

**명령어 목록**

- `-release 11` : JDK 11 버전에서 더 이상 사용되지 않는 API 목록.
- `jdeprscan --release 11 --list`: Java 8 버전 이후 deprecated된 API 목록 확인
- `jdeprscan --release 11 --list --for-removal`: 제거된 API 목록 확인

**예시**

```text
jdeprscan --release 11 --class-path log4j-api-2.13.0.jar my-application.jar

error: cannot find class sun/misc/BASE64Encoder
class com/company/Util uses deprecated method java/lang/Double::<init>(D)V
```

## 결론

어플리케이션 성능 개선 측면에서 JDK 버전 업그레이드는 필수적이라고 생각되는 자료였습니다.

다만, 버전 업그레이드간 발생할 수 있는 오류가 생각보다 많아 지금 작성한 글 이외에도 발생할 수 있는 오류에 대해 고려하며 버전 업그레이드를 진행하면 좋을 것 같습니다. (8 -> 17 보다는 8 -> 11 -> 17 업그레이드가 좀 더 안전하지 않을까.. 생각됩니다.)

## Reference

- [여기어때 기술블로그 - 우리팀이 JDK 17을 도입한 이유](https://techblog.gccompany.co.kr/%EC%9A%B0%EB%A6%AC%ED%8C%80%EC%9D%B4-jdk-17%EC%9D%84-%EB%8F%84%EC%9E%85%ED%95%9C-%EC%9D%B4%EC%9C%A0-ced2b754cd7)
- [transition-from-java-8-to-java-11](https://docs.microsoft.com/en-us/java/openjdk/transition-from-java-8-to-java-11)
- [linkedin-s-journey-to-java-11](https://engineering.linkedin.com/blog/2022/linkedin-s-journey-to-java-11)
- [how-to-upgrade-from-java-8-to-java-17](https://medium.com/javarevisited/how-to-upgrade-from-java-8-to-java-17-eb58f4554c6e)
- [JDK 9 All Release Note](https://www.oracle.com/java/technologies/javase/9-relnotes.html)
- [JDK 10 All Release Note](https://www.oracle.com/java/technologies/javase/10-relnotes.html)
- [JDK 11 All Release Note](https://www.oracle.com/java/technologies/javase/11all-relnotes.html)
