---
title: How to migrate Junit 5
pubDatetime: 2022-12-09T09:00:00+09:00
description: Junit4에서 Junit5로 마이그레이션하는 절차와 주의사항, 그리고 @Nested·@ParameterizedTest 등 Junit5 기능을 실제 테스트에 적용한 경험을 정리합니다.
tags:
  - tech-share
  - junit5
---

## Junit4 -> Junit 5 migration 과정

단순 Junit4 -> Junit 5 migration 과정을 공유드립니다.

해당 migration은 condition checker를 통해 시범적으로 시도했으며 junit 5의 기능을 도입한 것이 아닌 migration이라는 점 참고해주시면 감사하겠습니다.

해당 글 이후 junit 5 기능을 통해 테스트 코드 개선 작업을 진행해보고 추후 공유드리겠습니다.

### 1. Dependency 변경

Spring Boot는 2.2 부터 JUnit5가 기본으로 채택되었습니다.

![](/images/junit4-to-junit5-migration/img-01.png)

#### pom.xml

1. maven-surefire-plugin 2.22.0 버전 이상 plugin 추가

```xml
<plugin>
   <artifactId>maven-surefire-plugin</artifactId>
   <version>2.22.2</version>
</plugin>
```

2. Spring Boot 2.2 이상 버전 `spring-boot-starter-test` dependency 추가

```xml
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-test</artifactId>
<version>${spring-boot.version}</version>
<scope>test</scope>
```

3. 기존 junit4와 관련된 dependency 제거

```xml
<!-- 제거 해야합니다 -->
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>${junit.version}</version> <!-- 4.12 -->
    <scope>test</scope>
</dependency>
```

#### 주의사항 1. ScriptEvaluationException

다음과 같이 진행할 경우 다음 오류가 발생할 수 있습니다.

**Caused by: java.lang.ClassNotFoundException: org.junit.jupiter.api.extension.ScriptEvaluationException**

=> 해당 경우, junit4와 junit5를 혼합해 사용하면 발생하는 이슈였습니다. 오류 발생시 남아있는 junit4 logic을 확인해보시길 바랍니다.
(저 같은 경우 @RunWith를 @ExtendWith로 변경하지 않아 발생한 오류였습니다.)

#### 주의사항 2. maven-surefire-plugin version

말로는 2.22.0 버전 이상을 사용할 경우 이상이 없다고 하지만, 어플리케이션 테스트 관련 이슈가 존재해 plugin version을 2.22.2 버전을 사용하였습니다.

ex) @ExtendWith 실행시 오류 발생 등

### 2. Migration

![](/images/junit4-to-junit5-migration/img-02.png)

junit4와 junit5의 변경된 부분은 크게 두 가지 부분이 존재합니다.

- **annotation**
- **dependency** (org.junit.jupiter.api 사용)

#### 2-1 annotation

annotation은 크게 다음과 같은 내용들이 변경되었습니다.

1. **@RunWith(SpringRunner.class) -> @ExtendWith(SpringExtension.class)**
2. **@Rule -> @ExtendWith**
3. **@Before, @After -> @BeforeEach, @AfterEach**
4. **@BeforeClass, @AfterClass -> @BeforeAll, @AfterAll**
5. **@Ignore -> @Disabled or @DisabledIf(조건문)**

#### 2-2 dependency

1. junit5 는 `org.junit.jupiter.api`를 사용합니다.
2. 기존에 사용하던 org.junit.Assert는 AssertJ, Hamcrest, Truth 등으로 변경합니다.

## Junit5 기능 적용하기 (feat. Condition Checker)

migration 이후 개인적으로 어떤 부분이 junit 4에서 5로 기능을 수정하면 좋을까에 대한 고민을 해봤습니다.

가이드를 참고하더라도 실제로 적용해보지 않으니 어떤 기능이 개선된 것인지 감이 오지 않아 Condition Checker에 junit5의 일부 기능을 도입해 보았습니다.

추가된 어노테이션들을 대상으로 리팩토링을 진행하였습니다.

### 1. @DisplayName ([Junit5 공식 가이드](https://junit.org/junit5/docs/current/user-guide/) - 2.4. Display Names)

기존에는 테스트 코드 실행시 테스트 메소드 명칭이 나왔으나 junit5부터는 @DisplayName 어노테이션을 통해 통합 IDE에서 테스트 실행시 보여지는 메소드 명칭을 수정할 수 있습니다.

이를 통해 테스트 수행시 좀 더 해당 메소드가 수행하는 내용을 직관적으로 설명이 가능해집니다. 아래 테스트 내용을 통해 실행 결과가 어떻게 변경되었는지 보여드리겠습니다.

**변경 전**

![](/images/junit4-to-junit5-migration/img-03.png)

**변경 후**

![](/images/junit4-to-junit5-migration/img-04.png)

해당 코드를 작성하지 않은 사람이 테스트 코드를 수행할 경우 어떤 기능을 테스트하고 있는지 이전보다 더 직관적으로 보여주는 것을 확인할 수 있습니다.

### 2. @Disabled ([Junit5 공식 가이드](https://junit.org/junit5/docs/current/user-guide/) - 2.8. Conditional Test Execution)

Junit4에서 사용되는 @Ignore가 Junit5에서는 @Disabled로 변경되었습니다.

이 뿐만 아니라, @DisabledIf, @EnabledIf, @EnabledOnOs 등 다양한 조건문 어노테이션을 통해 테스트를 조건부로 실행시킬 수 있습니다.

**예시**

```java
@Disabled("서버마다 공통된 mount 정보가 없어 테스트 불가능.")
@Test
public void test_get_detail() throws Exception {
    // Given
    NasStatus nasStatus = new NasStatus(getNasPath());

    // When
    String result = nasStatus.getDetail();

    // Then
    assertEquals("Nas path : devfs\n" + "Mount local path : /dev/\n" + "Total : 189\n" + "Used : 189\n" + "Usage percent : 100%\n", result);
}
```

### 3. @TestMethodOrder ([Junit5 공식 가이드](https://junit.org/junit5/docs/current/user-guide/) - 2.10. Test Execution Order)

기존 junit4에서는 `@FixMethodOrder`와 `MethodSorters`의 조합으로 테스트 순서를 정의할 수 있었습니다. 하지만 해당 어노테이션으로는 테스트 메소드 순서 정의에 있어 기능이 적어 일부 제한이 존재하였습니다.
(MethodSorters.DEFAULT, MethodSorters.JVM, MethodSorters.NAME_ASCENDING 세 개가 전부)

Junit5에서는 `@TestMethodOrder` 및 `TestClassOrder`를 지원하며 `@Order`를 통해 원하는 테스트 클래스/메소드 실행 순서를 정의하여 실행이 가능해집니다.

**예시**

```java
@TestMethodOrder(OrderAnnotation.class)
class OrderedTestsDemo {
    @Test
    @Order(1)
    void nullValues() {
        // perform assertions against null values
    }
    @Test
    @Order(2)
    void emptyValues() {
        // perform assertions against empty values
    }
    @Test
    @Order(3)
    void validValues() {
        // perform assertions against valid values
    }
}
```

해당 어노테이션은 ConditionInfoCheckerTest -> ConditionCheckerTest 테스트 클래스에 적용되었습니다.

### 4. @Nested ([Junit5 공식 가이드](https://junit.org/junit5/docs/current/user-guide/) - 2.12. Nested Tests)

기존 junit4에서는 inner class를 이용한 테스트 그룹핑이 불가능하였으나 junit5부터는 @Nested 어노테이션을 이용할 시 **테스트간 그룹핑이 가능**해졌습니다.

하나의 테스트 클래스에서 테스트를 수행하지만 수행하는 테스트의 기능이 여러가지일 경우 기능별로 묶어 수행 결과에 대해 좀 더 직관적으로 확인이 가능해집니다.

`ConditionInfoCheckerTest.java`를 통해 예시를 들어보겠습니다.

해당 테스트 클래스에서는 다음과 같은 목록으로 그룹핑이 가능했습니다.

**목록 그룹핑**

- Condition Checker 기본
- HealthContributorRegistry를 사용할 경우
- ReactiveHealthContributorRegistry를 사용할 경우

이제 이들을 @Nested로 그룹화하여 inner class로 코드를 리팩토링하였습니다.

**기존 결과 (변경 전)**

![](/images/junit4-to-junit5-migration/img-05.png)

**변경 후 결과**

![](/images/junit4-to-junit5-migration/img-06.png)

다음과 같이 변경된 이후 좋았던 것은 **그룹핑 전엔 보이지 않았던 기능별 테스트 목록이 조금 더 직관적으로 보였다**는 부분입니다.

@DisplayName과 함께 사용하니 그룹별로 어떤 기능의 테스트가 미흡한지 보다 더 잘 알 수 있었습니다.

### 5. @RepeatedTest ([Junit5 공식 가이드](https://junit.org/junit5/docs/current/user-guide/) - 2.15. Repeated Tests)

반복 테스트로서 해당 어노테이션을 정의한 테스트는 원하는 반복 횟수를 정의하면 정의한 횟수만큼 테스트를 수행합니다.

@RepeatedTest를 테스트 메서드에 붙일 경우 다음과 같은 실행 결과를 얻을 수 있습니다.

**예시 - ConditionInfoCheckerTest**

![](/images/junit4-to-junit5-migration/img-07.png)

### 6. @ParameterizedTest ([Junit5 공식 가이드](https://junit.org/junit5/docs/current/user-guide/) - 2.16. Parameterized Tests)

기존 junit4에서의 테스트 메소드는 파라미터를 받을 수 없는 구조로 구현되어 있었습니다. 따라서 테스트 코드 작성시 내부에서 테스트 입력 값은 메소드 내부에서 하드코딩해 생성하거나 @BeforeClass, @Before를 통해 값을 생성하였습니다.

Junit5 부터는 @ParameterizedTest를 통해 테스트 데이터를 어노테이션을 통해 파라미터값으로 입력받을 수 있습니다.

**대표 어노테이션**

- @CsvSource: Csv 파일과 같은 값으로 테스트 파라미터를 받을 수 있음
- @ValueSource: type을 지정해 해당 타입의 값을 파라미터로 정의할 수 있음
- @EnumSource: Enum 클래스를 정의 또는 인용하여 테스트 가능
- @MethodSource: 메소드 결과 값으로 반환된 결과를 사용할 수 있음 (외부 메소드도 적용 가능)
  - 주의 사항: 반드시 인자의 stream을 만들어야 함

해당 어노테이션들은 테스트 수행시 명시적으로 또는 암시적으로 사용할 수 있는 기능이 존재합니다.

**암시적 선언 예시**

```java
@ParameterizedTest
@ValueSource(strings = "42 Cats")
void testWithImplicitFallbackArgumentConversion(Book book) {
  assertEquals("42 Cats", book.getTitle());
}
```

**명시적 선언 예시**

```java
@ParameterizedTest
@ValueSource(strings = "42 Cats")
void testWithImplicitFallbackArgumentConversion(String title) {
  assertEquals("42 Cats", title);
}
```

이제 NotificationServiceStatusTest를 통해 해당 테스트의 예시를 들어보겠습니다.

**기존 코드 (변경 전)**

```java
public class NotificationServiceStatusTest {

    @Test
    public void test_notification_3368_stats_status_false() {
        assertFalse(new NotificationServiceStatus(WRONG_DOMAIN, NotificationService.STATS).check());
    }

    @Test
    public void test_notification_3368_fileIE_status_true() {
        assertTrue(new NotificationServiceStatus(conditionConfiguration.getStatsApiUrl(), NotificationService.FILEIE).check());
    }

    @Test
    public void test_notification_3368_fileIE_status_false() {
        assertFalse(new NotificationServiceStatus(WRONG_DOMAIN, NotificationService.FILEIE).check());
    }

    @Test
    public void test_notification_3368_webHook_status_true() {
        assertTrue(new NotificationServiceStatus(conditionConfiguration.getStatsApiUrl(), NotificationService.WEBHOOK).check());
    }

    @Test
    public void test_notification_3368_webHook_status_false() {
        assertFalse(new NotificationServiceStatus(WRONG_DOMAIN, NotificationService.WEBHOOK).check());
    }

    @Test
    public void test_notification_3368_tag_status_true() {
        assertTrue(new NotificationServiceStatus(conditionConfiguration.getStatsApiUrl(), NotificationService.TAG).check());
    }

    @Test
    public void test_notification_3368_tag_status_false() {
        assertFalse(new NotificationServiceStatus(WRONG_DOMAIN, NotificationService.TAG).check());
    }

}
```

**실행 결과 (변경 전)**

![](/images/junit4-to-junit5-migration/img-08.png)

**변경 후**

해당 코드는 모두 NotificationService enum 클래스의 값으로 테스트를 수행하므로 @EnumSource를 이용한 테스트로 통합하였습니다.

```java
public class NotificationServiceStatusTest {

    @ParameterizedTest
    @EnumSource(NotificationService.class)
    @DisplayName("NotificationServiceStatus Status check using NotificationService - SUCCESS")
    public void test_notification_3368_notificationServiceStatus_true(NotificationService notificationService) {
        assertTrue(new NotificationServiceStatus(conditionConfiguration.getStatsApiUrl(), notificationService).check());
    }

    @ParameterizedTest
    @EnumSource(NotificationService.class)
    @DisplayName("NotificationServiceStatus Status check using NotificationService and Wrong Domain - FAIL")
    public void test_notification_3368_notificationServiceStatus_false(NotificationService notificationService) {
        assertFalse(new NotificationServiceStatus(WRONG_DOMAIN, notificationService).check());
    }
}
```

**실행 결과 (변경 후)**

![](/images/junit4-to-junit5-migration/img-09.png)

### 7. @MethodSource ([Junit5 공식 가이드](https://junit.org/junit5/docs/current/user-guide/) - 2.16. Parameterized Tests)

@ParameterizedTest의 연장선상에 있는 기능입니다. @MethodSource를 사용할 경우 호출한 메소드를 테스트 메소드의 파라미터로 받을 수 있게 됩니다.

주의사항으로서는 무조건 Stream 객체로 반환해야 된다는 점이 있습니다.

다음은 NotificationServiceStatusTest에서 반영한 내용입니다.

```java
@ParameterizedTest
@MethodSource("statsStatusSuccessDomainList")
@DisplayName("NotificationServiceStatus Status check - SUCCESS(basic, add last slash, HTTP protocol)")
public void test_notification_3368_stats_status_true(NotificationServiceStatus notificationServiceStatus) {
    assertTrue(notificationServiceStatus.check());
}

static Stream<NotificationServiceStatus> statsStatusSuccessDomainList() {
    return Stream.of(new NotificationServiceStatus(conditionConfiguration.getStatsApiUrl(), NotificationService.STATS),
            new NotificationServiceStatus(conditionConfiguration.getStatsApiUrl() + "/", NotificationService.STATS),
            new NotificationServiceStatus("http://" + conditionConfiguration.getStatsApiUrl(), NotificationService.STATS));
}
```

### 8. @NullAndEmptySource ([Junit5 공식 가이드](https://junit.org/junit5/docs/current/user-guide/) - 2.16. Parameterized Tests)

@ParameterizedTest의 연장선상에 있는 어노테이션입니다. 해당 어노테이션을 추가할 경우 Null과 빈값을 파라미터로 받을 수 있습니다.

각각 호출하고 싶은 경우 @NullSource 및 @EmptySource를 사용하시면 될 것 같습니다.

### 주의사항

inner class 내에서 @BeforeAll을 적용할 경우 정상적으로 테스트 코드가 적용되지 않는 오류가 발생했습니다. 오류 내용은 다음과 같습니다.

```text
org.junit.platform.commons.JUnitException: @BeforeAll method 'public void com.toast.cloud.condition.master.ConditionInfoCheckerTest.beforeClass() throws java.io.IOException' must be static unless the test class is annotated with @TestInstance(Lifecycle.PER_CLASS).
```

오류의 마지막 문구에 @TestInstance(Lifecycle.PER_CLASS) 적용시 static을 적용하지 않아도 된다는 문구가 존재하여 @TestInstance(Lifecycle.PER_CLASS)를 사용해 테스트 코드를 작성하였습니다.

## 결론

현재 적용한 기능 말고도 여러 기능이 있지만 Condition Checker는 테스트 코드가 큰 복잡성이 없어서 모든 기능을 다 사용해보진 못한 것 같습니다.

Condition Checker 테스트 코드에 junit5를 적용하면서 느낀 것은 junit5 기능 중 @Nested와 @Parameterized 테스트만 잘 사용해도 코드 작성 효율이 꽤나 높아질 것 같았습니다.

`2.18. Dynamic Tests` 및 `5. Extension Model` 에 대한 내용은 Condition Checker에서 다룬 부분이 없는데 Email 에도 점진적으로 적용해 나가면서 junit5의 해당 기능들에 대해 추가로 공유하겠습니다. (동적 테스트 관련 내용입니다.)

읽어주셔서 감사합니다.

## 별첨. 미사용 기능

동적 테스트와 관련된 기능이 많이 포함되어 있으며 해당 부분은 [junit5 dynamic test](https://www.baeldung.com/junit5-dynamic-tests) 및 [Junit5 User Guide(공식 문서)](https://junit.org/junit5/docs/current/user-guide/)를 참고해주시면 감사하겠습니다.

### 미사용 `junit.jupiter.api` 목록

- @TestFactory
- @TestTemplate
- @TestInstance
- @Tag
- @DisplayNameGeneration
- @RegisterExtension
- @Timeout
- @TempDir

## Reference

- [Junit5 User Guide(공식 문서)](https://junit.org/junit5/docs/current/user-guide/)
- [Junit5 한국어 번역 가이드](https://donghyeon.dev/junit/2021/04/11/JUnit5-%EC%99%84%EB%B2%BD-%EA%B0%80%EC%9D%B4%EB%93%9C/)
