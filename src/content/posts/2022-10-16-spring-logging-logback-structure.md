---
title: Spring logging 구조 및 팀 내 logback 사용 공유
pubDatetime: 2022-10-16T09:00:00+09:00
description: Spring의 로깅 구조(JCL/SLF4J/logback)와 logback의 Appender·Encoder·Logger 계층을 정리하고, 팀 logback.xml 개선 과정을 공유합니다.
tags:
  - tech-share
  - logback
  - spring-boot
---

Spring의 logging 구조와 logback.xml 설정에 대해 이해도가 낮아 개인적으로 공부하면서 작성한 글입니다.

## 스프링(Spring)에서 기본 로깅 구조

![](/images/spring-logging-logback-structure/img-01.png)

현재 Spring Framework에서는 별도의 설정을 하지 않을 경우, Apache의 **Jakarta Commons Logging(이하 JCL)** 을 사용하고 있습니다.

**Jakarta Commons Logging(이하 JCL)** 이란, Java 기반 logging 유틸리티입니다. logging을 위해서는 JCL을 기반으로 한 logging framework를 사용해야 하며 대표적인 예시로는 log4j, slf4j, logback 등이 있습니다. logging framework는 다음과 같이 구현되고 있습니다.

- JCL 구현체 (ex) log4j)
- `jcl-over-slf4j` 와 같은 브릿지 라이브러리를 이용한 구현 (ex) SLF4J, Logback)

### Setup

logback의 dependency로는 다음과 같은 종류가 있습니다.

- `logback-core`: Appender와 Layout 인터페이스가 존재하는 모듈
- `logback-classic`: `logback-core`와 `SLF4J API` 라이브러리를 포함하고 있음, Logger 클래스가 포함된 모듈
- `logback-access`: Servlet Container와 통합되어 HTTP 액세스에 대한 로깅 기능을 제공. Container 레벨에서 사용

따라서 logback을 이용하기 위해서는 앞서 본 내용을 생각한다면 다음 라이브러리들이 필요하다는 것을 알 수 있습니다.

- jcl-over-slf4j (어댑터 역할)
- logback-classic **(slf4j-api + logback-core + logger 클래스)**

```xml
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.2.6</version>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>jcl-over-slf4j</artifactId>
    <version>1.7.5</version>
</dependency>
```

## logback이란?

logback은 Log4j의 후속작으로 알려져 있는 로깅 시스템입니다. logback은 단순한 구조로 여러 다양한 기능들을 제공하는데 어떠한 기능을 제공하는지 logback의 구조와 함께 확인해 보겠습니다.

### logback 기본 구성

![](/images/spring-logging-logback-structure/img-02.png)

**logback 구성 요소 역할**

- **appender**: logging 이벤트 처리 역할을 담당. `Appender` 인터페이스를 구현해 이벤트를 처리 (**Logger는 logging 이벤트를 처리**)
- **logger**: 이벤트의 대상, 어떤 내용을 log로 남길 것인지 정의하며 log level을 선택적으로 설정할 수 있음
- **root**: 최상단의 logger를 root라 칭합니다. configuration 태그는 내부에 최대 1개의 root 태그를 갖고 0개 이상의 appender와 logger를 갖습니다.

## Appender

Appender는 logger로 정의한 내용들을 어떻게 처리할 것인지 처리 역할을 위임받은 클래스입니다. **Appender는 태그를 통하여 구성되며 name과 class 속성을 필수적으로 가져야만 합니다.**

### Appender 클래스 다이어그램

![](/images/spring-logging-logback-structure/img-03.png)

appender와 관련된 클래스는 각각 다음과 같은 기능을 담당합니다.

### Appender 종류

| Appender | 설명 |
| --- | --- |
| ConsoleAppender | 로그 이벤트를 `System.err` 또는 `System.out`에 추가 |
| FileAppender | 파일에 로그 이벤트를 추가 |
| RollingFileAppender | `FileAppender`를 상속하며 파일을 롤오버하는 기능 |
| SMTPAppender | 로깅 이벤트를 이메일로 발송 |
| CustomAppender | 사용자가 직접 정의하여 사용 |

#### 1. ConsoleAppender

logging 이벤트를 `System.err` 또는 `System.out`에 추가합니다. 이 때, 사용자가 지정한 인코더를 사용해 이벤트 형식을 지정합니다.

**예시**

```xml
<!--System.out에 로그를 추가합니다. -->
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
</appender>
```

#### 2. FileAppender

`OutputStreamAppender`의 서브클래스로서 파일에 로그 이벤트를 추가합니다. 대상 파일은 파일 옵션을 통해 지정되며 속성에 따라 파일이 추가되거나 분할됩니다.

**예시**

```xml
<!--testFile.log에 로그를 남깁니다.-->
<appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <file>testFile.log</file>
    <append>true</append>
    <immediateFlush>true</immediateFlush>
    <encoder>
      <pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n</pattern>
    </encoder>
</appender>
```

#### 3. RollingFileAppender

RollingFileAppender는 `FileAppender`를 상속하며 파일을 롤오버하는 기능으로 확장합니다. 롤오버의 예시로는 **날짜별 로그파일 작성, 시간별 로그파일 작성** 등 일정 기간, 혹은 조건 등에 따라 로그 파일을 분리하는 것입니다.

**예시**

```xml
<appenders>
    <rollingFile name="LogToFile" fileName="logs/application.log"
filePattern="logs/application.log.%d{yyyy-MM-dd-hh-mm}">
        <patternLayout pattern="${sys:FILE_LOG_PATTERN}" />
        <!-- 정책을 통해 로그파일 롤오버 => 1분마다 로그파일이 생성-->
        <policies>
            <timeBasedTriggeringPolicy interval="1" modulate="true" />
        </policies>
        <!-- 기본 롤오버 전략 => 생성된 로그 파일이 3개가 초과될 때 1개 삭제-->
        <defaultRolloverStrategy>
            <delete basePath="logs" maxDepth="1">
                <ifAccumulatedFileCount exceeds="3"/>
            </delete>
        </defaultRolloverStrategy>
    </rollingFile>
</appenders>
```

#### 4. SMTPAppender

SMTPAppender는 하나 이상의 고정 버퍼에 로깅 이벤트를 누적하고 사용자 지정 이벤트가 발생한 이후 해당 버퍼의 내용을 이메일로 발송합니다.

기본적으로 이메일 전송은 ERROR 레벨의 로깅 이벤트에 의해 트리거되며, 비동기식으로 수행됩니다.

**예시**

```xml
<appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">
  <smtpHost>gelog.smtp.com</smtpHost>
  <to>gelloger@example.com</to>
  <from>gelloger@example.com</from>
  <subject>TESTING: %logger{20} - %m</subject>
  <layout class="ch.qos.logback.classic.PatternLayout">
    <pattern>%date %-5level %logger{35} - %message%n</pattern>
  </layout>
</appender>

<root level="DEBUG">
  <appender-ref ref="EMAIL" /> <!--에러 발생시 이메일 발송 트리거 -->
</root>
```

#### 5. CustomAppender

CustomAppender는 사용자가 직접 구현하는 appender 클래스를 의미합니다.

`AppenderBase<ILoggingEvent>`를 상속해 클래스를 작성하는데 logback.xml에서 **해당 appender를 호출 시 `start()` 메소드는 자동으로 실행**되며 해당 logger가 동작할 경우 `append(ILoggingEvent iLoggingEvent)` 메소드가 동작을 수행합니다.

**예시**

```java
package com;

public class LoggingAppender extends AppenderBase<ILoggingEvent> {

    private static final SimpleDateFormat datetimeFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    @Override
    protected void append(ILoggingEvent iLoggingEvent) {
        System.out.println(iLoggingEvent.getFormattedMessage());
    }

    @Override
    public void start() {
        super.start();
    }

}
```

```xml
<!-- GellogerMapper.create 호출시 trace level로 EvidenceLoggingAppender가 이벤트 실행 -->
<appender name="customlog" class="com.EvidenceLoggingAppender">
</appender>

<logger name="test.gelog.GellogerMapper.create" level="trace">
    <appender-ref ref="customlog"/>
</logger>
```

### Encoder

Encoder는 들어오는 log 이벤트를 `OutputStream`에서 바이트 배열로 변환시키는 작업을 수행합니다. `logback 0.9.19` 버전 이후 출시된 Encoder는 Layout과 함께 log 출력값을 정의합니다.

#### 1. LayoutWrappingEncoder

`LayoutWrappingEncoder`는 Encoder와 Layout 간의 격차를 해소해주기 위해 Layout을 wrapping합니다.

```java
package ch.qos.logback.core.encoder;

public class LayoutWrappingEncoder<E> extends EncoderBase<E> {

  protected Layout<E> layout;
  private Charset charset;

   public byte[] encode(E event) {
     String txt = layout.doLayout(event);
     return convertToBytes(txt);
  }

  private byte[] convertToBytes(String s) {
    if (charset == null) {
      return s.getBytes();
    } else {
      return s.getBytes(charset);
    }
  }
}
```

다음 로직을 통해 Encoder는 **Layout이 로그 이벤트를 정의해놓은 형식으로 변환한 문자열에 대해 바이트로 변환해 반환하는 역할을 수행**합니다.

#### 2. PatternLayoutEncoder

가장 일반적으로 사용되는 레이아웃으로, FileAppender 또는 FileAppender의 서브클래스는 PatternLayout으로 구성될 때마다 무조건 PatternLayoutEncoder를 사용해야 합니다.

logback은 로그 파일의 상단에 로그 출력 패턴을 정의할 수 있습니다. 기본적으로 패턴은 비활성화되어 있어 `outputPatternAsHeader`를 활성화시켜야 이용이 가능합니다.

**예시**

```xml
<appender name="FILE" class="ch.qos.logback.core.FileAppender">
  <file>foo.log</file>
  <encoder>
    <pattern>%d %-5level [%thread] %logger{0}: %msg%n</pattern>
    <outputPatternAsHeader>true</outputPatternAsHeader>
  </encoder>
</appender>
```

### Layout

Layout이란 들어오는 이벤트를 문자열로 변환하는 역할을 하는 logback 구성 요소입니다.

우리는 appender와 encoder, layout의 조합으로 들어오는 로그를 원하는 형식대로 구성할 수 있습니다.

**예시**

```java
package chapters.layouts;

import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.LayoutBase;

public class MySampleLayout extends LayoutBase<ILoggingEvent> {

  public String doLayout(ILoggingEvent event) {
    StringBuffer sbuf = new StringBuffer(128);
    sbuf.append(event.getTimeStamp() - event.getLoggingContextVO.getBirthTime());
    sbuf.append(" ");
    sbuf.append(event.getLevel());
    sbuf.append(" [");
    sbuf.append(event.getThreadName());
    sbuf.append("] ");
    sbuf.append(event.getLoggerName();
    sbuf.append(" - ");
    sbuf.append(event.getFormattedMessage());
    sbuf.append(CoreConstants.LINE_SEP);
    return sbuf.toString();
  }
}
```

```xml
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
  <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
    <!-- 위에서 만든 layout. log 이벤트가 다음 문자열로 들어옴 -->
    <layout class="chapters.layouts.MySampleLayout" />
  </encoder>
</appender>

<root level="DEBUG">
  <appender-ref ref="STDOUT" />
</root>
```

### Filter

필터를 통해 Appender는 특정 이벤트에 대해 조건을 가지고 필터링을 수행할 수 있습니다.

**필터 종류**

- ACCEPT: 허용, 남아있는 필터를 무시한 채 바로 이벤트를 처리
- NEUTRAL: 중립, 다음 필터를 확인함
- DENY: 거부, 즉시 이벤트를 버림

**예시**

```java
public class SampleFilter extends Filter<ILoggingEvent> {

  @Override
  public FilterReply decide(ILoggingEvent event) {
    // "sample"이라는 문자열을 포함할 경우 ACCEPT, 아닐 경우 다음 필터 확인
    if (event.getMessage().contains("sample")) {
      return FilterReply.ACCEPT;
    } else {
      return FilterReply.NEUTRAL;
    }
  }
}
```

```xml
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">

    <filter class="chapters.filters.SampleFilter" />

    <encoder>
      <pattern>
        %-4relative [%thread] %-5level %logger - %msg%n
      </pattern>
    </encoder>
</appender>
```

## Logger

로깅 작업을 수행하는 주체로 Logger 설정을 제외한 모든 로깅 기능이 Logger를 통해 처리됩니다. 사용자는 어플리케이션 내에서 사용할 Logger를 정의해야 하며, logger와 appender의 조합으로 로그 이벤트를 요구에 맞게 다양하게 처리할 수 있습니다.

### Log level

log level은 크게 6가지로 나뉘어지며 각각 다른 의미를 가지고 있습니다. 다음 표를 보고 필요한 log level을 logger에 작성해주시면 됩니다.

| 로그 | Level |
| --- | --- |
| FATAL | 애플리케이션이 이벤트를 발생했거나 중요한 비즈니스 기능 중 하나가 더 이상 작동하지 않는 상태일 경우를 알려주는 로그 Level |
| ERROR | 애플리케이션이 하나 이상의 기능이 제대로 작동하지 않는 문제에 부딪힐 때 사용해야 하는 로그 Level |
| WARN | 애플리케이션이 문제 또는 프로세스에 방해가 될 수 있는 상황에 예기치 않은 일이 발생했음을 나타내는 로그 Level |
| INFO | 애플리케이션이 특정 상태에 들어갔는지 등을 나타내는 표준 로그 Level, 일반적으로 정보제공을 위해 사용 |
| DEBUG | 문제를 해결하는데 필요할 수 있는 정보 제공, 모든 것이 올바르게 정상적으로 동작하는지 확인하기 위해 테스트 환경에서 실행할 경우 사용 |
| TRACE | 애플리케이션의 모든 상황을 완벽하게 파악하는 상황에서 사용, debug의 윗 수준으로 log 정보를 매우 상세하게 나타냄 |

### Logger Hierarchy

사용자가 호출한 Logger 객체가 어떤 설정을 따르는지 이해하기 위해서는 Logger Hierarchy에 대해 알고 있어야 합니다.

내부적으로 설정 파일에 정의된 각 Logger 설정에 따라 LoggerConfig 오브젝트가 생성되며, **Logger Name에 따라 오브젝트 간 부모-자식 관계가 성립**합니다.

- 예시) "X.Y" Logger의 부모는 "X"이고, "X" Logger의 부모는 Root Logger(최상위)

#### Logger Hierarchy 규칙

1. 호출한 Logger Name과 동일한 Logger가 있는 경우, 해당 Logger 설정을 따릅니다.
2. 동일한 Logger는 없지만, Parent Logger가 존재하는 경우, Parent Logger 설정을 따릅니다.
3. Parent Logger도 존재하지 않는 경우, Root Logger 설정을 따릅니다.

#### Logger Hierarchy 예시

기존 이메일 서비스의 구조로 한 번 예시를 작성해보겠습니다.

```xml
<logger name="com.toast.cloud.notification"></logger>
<logger name="com.toast.cloud.condition"></logger>
<logger name="com.toast.cloud"></logger>
<root></root>
```

예시는 `root` → `com.toast.cloud` → `com.toast.cloud.notification` / `com.toast.cloud.condition` 순의 Logger Hierarchy를 가지게 됩니다.

각 Logger name에 logger가 존재하지 않을 경우, 상위 logger의 설정을 따르게 됩니다.

- `com.toast.cloud`, `com.toast.cloud.notification` Logger가 없을 경우
  - root 설정을 따라간다

### Logger additivity

additivity는 상위 레벨의 로거에 전달할지 전달 유무를 설정하는 값입니다. 해당 설정을 통해 Logger Hierarchy 기능과 결합하여 선택적인 로깅이 가능해집니다.

> The output of a log statement of logger L will go to all the appenders in L and its ancestors. This is the meaning of the term "appender additivity".
> However, if an ancestor of logger L, say P, has the additivity flag set to false, then L's output will be directed to all the appenders in L and its ancestors up to and including P but not the appenders in any of the ancestors of P.
> Loggers have their additivity flag set to true by default.

## 팀에서 사용하는 logback 기능

현재 팀 서비스 내부에서 사용하는 logback 기능은 대표적으로는 다음과 같습니다.

1. ~~**SmtpAppender를 이용한 에러메일 발송** (Deprecated - Audit Log로 전환)~~

   ![SMTPAppender로 발송된 에러 메일 예시](/images/spring-logging-logback-structure/img-04.png)

2. **RollingFileAppender를 이용한 로그 파일 롤오버**
   1. 압축 기능 (`batch.log.%d{yyyy-MM-dd}.%i.log.gz`)
   2. 일정 기간 유지 후 삭제

   ![일자별로 롤오버·압축된 로그 파일 목록](/images/spring-logging-logback-structure/img-05.png)

3. **AsyncAppender를 이용한 logncrash 이용 (Audit Log 생성)**
4. **로컬에서 에러 출력용으로 사용하는 ConsoleAppender**

### logback.xml 개선 과정간 이슈

최근 SMTPAppender에선 수집이 되지만 Audit Log에 수집이 안 되는 문제 상황을 인지하여 logback.xml 파일의 설정 내용을 분석하였습니다.

#### 기존 구성 예제

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="30 seconds">
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        ...
    </appender>

    ...

    <logger name="com.toast.cloud.notification" level="info" additivity="false">
        <appender-ref ref="FILE"/>
        <appender-ref ref="error-smtp"/>
    </logger>

    <logger name="com.toast.cloud.condition" level="info" additivity="false">
        <appender-ref ref="FILE"/>
        <appender-ref ref="error-smtp"/>
    </logger>

    <logger name="audit-log" additivity="false">
        <appender-ref ref="audit-log"/>
    </logger>

    <logger name="msg-invalid" level="info" additivity="false">
        <appender-ref ref="msg-invalid"/>
    </logger>

    <logger name="feign.Logger" level="info" additivity="false">
        <appender-ref ref="FILE"/>
        <appender-ref ref="error-smtp"/>
    </logger>

    <logger name="com.nhnent.toastmail" level="info" additivity="false">
        <appender-ref ref="FILE"/>
        <appender-ref ref="error-smtp"/>
    </logger>

    <logger name="freemarker.runtime" level="warn" additivity="false">
        <appender-ref ref="FILE"/>
    </logger>

    <root level="warn">
        <appender-ref ref="FILE"/>
        <appender-ref ref="error-smtp"/>
        <appender-ref ref="error-audit-log"/>
    </root>
</configuration>
```

#### xml 정리

| Logger Name | level | appender list | additivity |
| --- | --- | --- | --- |
| root | warn | FILE, error-smtp, error-audit-log | false |
| com.toast.cloud.notification | info | FILE, error-smtp | false |
| com.toast.cloud.condition | info | FILE, error-smtp | false |
| audit-log | info | audit-log | false |
| msg-invalid | info | msg-invalid | false |
| feign.Logger | info | FILE, error-smtp | false |
| com.nhnent.toastmail | info | FILE, error-smtp | false |
| freemarker.runtime | warn | FILE | false |

#### Logger 구현 정리

```java
package com.nhnent.toastmail.common.exception;

@ControllerAdvice
public class GlobalExceptionHandler {
    private static final Log log = LogFactory.getLog(GlobalException.class);
    private static final Log invalidLog = LogFactory.getLog("msg-invalid");
}
```

특이사항은 다음과 같습니다.

- error-audit-log는 root에만 정의되어 있다.
- audit-log, msg-invalid는 독립적으로 동작한다.
- 모든 logger에는 additivity가 존재한다.
- audit-log, msg-invalid를 제외한 대부분의 logger는 FILE과 error-smtp로 구성되어 있다.
- 실질적으로 구현된 Logger는 msg-invalid, com.nhnent.toastmail.common.exception 만 존재합니다.

#### 해결 방법

해결 방법은 다음과 같습니다.

1. 분리한 logger를 통합합니다.
   - 현재 모든 logger는 info level로 동일한 appender를 이용하고 있기 때문에 분리해 관리할 필요가 없습니다.
   - 따라서 전부 root level에 통합한 뒤 분리할 대상만 별도의 logger로 분리합니다.
2. 분리한 logger는 additivity를 true로 설정합니다.

smtpAppender를 제거한 나머지 결과를 보면 다음과 같습니다.

#### 개선

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="30 seconds" debug="true">
    <logger name="audit-log" additivity="true">
        <appender-ref ref="audit-log"/>
    </logger>

    <logger name="msg-invalid" additivity="true">
        <appender-ref ref="msg-invalid"/>
    </logger>

    <root>
        <appender-ref ref="error-audit-log"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

리팩토링 이후 깔끔한 구조로 기존에 요구하던 모든 로직을 수행할 수 있게 되었습니다.

읽어주셔서 감사합니다.

## Reference

- https://ckddn9496.tistory.com/79
- https://www.baeldung.com/logback
- https://www.baeldung.com/log4j2-custom-appender
- https://logback.qos.ch/manual/appenders.html
- https://tecoble.techcourse.co.kr/post/2021-08-07-logback-tutorial/
- https://logback.qos.ch/manual/architecture.html#basic_selection
- https://www.egovframe.go.kr/wiki/doku.php?id=egovframework:rte3:fdl:설정_파일을_사용하는_방법
