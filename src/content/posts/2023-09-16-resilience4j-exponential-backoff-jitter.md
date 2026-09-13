---
title: Resilience4j로 Exponential Backoff And Jitter 도입하기
pubDatetime: 2023-09-16T09:00:00+09:00
description: Exponential Backoff와 Jitter의 개념을 살펴보고 Resilience4j-retry의 RetryConfig·IntervalFunction으로 적용하는 방법과 주의점을 설명합니다.
tags:
  - tech-share
  - resilience4j
  - spring-boot
---
팀원분이 [Exponential Backoff And Jitter](https://aws.amazon.com/ko/blogs/architecture/exponential-backoff-and-jitter/) 글을 공유해 주셨는데 당시에는 가볍게 보고 넘겼습니다. 최근에 갑자기 exponential backoff 로직을 사용할 일이 생겨서 해당 글을 다시 정독했고 읽으면서 적용하면 좋을 것 같다는 생각을 가지게 되었습니다.

이 글은 Resilence4j-retry library를 활용해 Exponential Backoff And Jitter를 적용하는 방법에 대해 공유하는 시간을 가져보고자 합니다.

## Exponential Backoff And Jitter란?

서비스 로직이 동작하는 과정에서 외부 시스템(ex) DB, 네트워크)과 함께 동작하는 과정에서 deadlock, 네트워크 오류 등 예상치 못한 오류들이 발생할 수 있습니다.

이런 경우를 대비하여 개발자들은 fault tolerance(결함 내성)이 있는 코드를 작성하게 되었습니다. 대표적인 예가 retry 로직입니다. 동작하는 코드가 특정 예외가 발생할 경우 해당 로직을 재시도할 수 있도록 코드를 구현하는 것인데 이와 관련된 대표적인 라이브러리로는 [Resilence4j](https://github.com/resilience4j/resilience4j), [Spring Retry](https://www.baeldung.com/spring-retry)가 있습니다.

Backoff는 이러한 재시도 동작을 `어느 시점에 할 것인가`에 대한 정의입니다. 3초 이후에 재시도 로직을 동작시키려 설정할 수도 있고 재시도 횟수에 따라 속도를 다르게 설정할 수도 있습니다.

**Exponential Backoff**는 이러한 설정과 관련이 있습니다. 다음 예시를 보시죠

### 예시 (N명의 사용자가 특정 로직에서 경합이 발생한 경우)

1명만 사용할 수 있는 로직에 N명이 들어와 deadlock 상황이 발생했다고 가정해봅시다. N명의 인원이 deadlock이 발생해 재시도를 진행할 경우 횟수마다 N-1명이 다시 경합을 벌이는 상황이 발생합니다.
그렇게 되면 결국 N^2의 시간이 소요되죠.

#### Backoff

![](/images/resilience4j-exponential-backoff-jitter/img-01.png)

매번의 재시도 로직이 돌 때마다 경합이 발생하는 것은 서비스에서는 굉장히 불편한 상황일겁니다. 이러한 문제를 개선하기 위해 각각의 사용자가 재시도 동작을 하는 시간을 재시도 횟수에 비례하여 증가시킵니다.
이를 **Exponential Backoff**라 부릅니다.

#### Exponential Backoff

> sleep = min(cap, base * 2^attempt)

![](/images/resilience4j-exponential-backoff-jitter/img-02.png)

![](/images/resilience4j-exponential-backoff-jitter/img-03.png)

**Exponential Backoff** 를 적용하더라도 N명이 호출되는 시간이 재시도 횟수만큼 증가한 것 뿐이지 재시도간 경합이 발생하는 문제는 계속되기 때문에 문제가 크게 개선되지 않은 것을 확인하실 수 있습니다.

#### Jitter

> sleep = random_between(0, base * 2^attempt)

Jitter는 무작위성, 즉 Random으로 이해하셔도 좋을 것 같습니다. 재시도를 호출하는 시간에 무작위성을 부여하여 각 사용자별로 다른 시간대에 재시도를 하게 만들어 경합을 해소시킵니다.

![](/images/resilience4j-exponential-backoff-jitter/img-04.png)

예시를 통해 fault tolerance(결함 내성)과 retry, Exponential Backoff 및 Jitter에 대해 알아보았습니다. 이제 Resilence4j에서 Exponential Backoff and Jitter를 적용하는 방법을 알아볼까요?

## Resilence4j

Resilence4j는 fault tolerance(결함 내성)를 위해 만들어진 라이브러리입니다. 다양한 기능들이 있고, 조합하여 멋진 코드를 작성할 수도 있으나 우리는 재시도 로직에 대해서 알아보고 있으니 Resilence4j-retry만 알아보면 될 것 같습니다.

사실 Resilence4j-retry에서는 모든 기능을 제공하고 있습니다. 따라서 약간의 사용법만 알면 될 것 같습니다.

### 알아야할 내용

- RetryConfig
- IntervalFunction
- Retry

#### RetryConfig

모든 재시도 로직은 RetryConfig에서 시작됩니다. 설정을 통해 `retry를 어떻게 할 것인지`를 설정합니다.

#### 기본 생성자를 통한 설정개념 알아보기

```java
 private RetryConfig() {
        this.retryExceptions = new Class[0];
        this.ignoreExceptions = new Class[0];
        this.maxAttempts = 3;
        this.failAfterMaxAttempts = false;
        this.writableStackTraceEnabled = true;
        this.intervalBiFunction = DEFAULT_INTERVAL_BI_FUNCTION;
    }
```

| 설정값 | 정의 |
| --- | --- |
| retryExceptions | 재시도해야할 예외를 정의 |
| ignoreExceptions | 재시도하지 않을 예외를 정의 |
| maxAttempts | 최대 시도 횟수 |
| failAfterMaxAttempts | 최대 시도 후 실패 여부 |
| writableStackTraceEnabled | Stack Trace를 출력할지 여부 |
| intervalBiFunction | 재시도 backoff 관련 함수 정의 |

따라서 기본 생성자로 정의할 경우 다음과 같은 의미를 가집니다.

> 최대 재시도 횟수는 3번이며 3번의 재시도 시도 후 실패처리는 하지 않습니다.
> 재시도 간격은 DEFAULT로 정의된 함수를 사용하며 stackTrace를 허용합니다.

#### IntervalFunction

IntervalFunction은 앞서 RetryConfig 설정에서 본 `intervalBiFunction`에 들어가는 함수입니다.

다양한 함수를 제공하고 있지만 Exponential backoff and Jitter를 적용하기 위해선 그 중 아래 함수를 이용하셔야 합니다.

IntervalFunction.java

```java
    static IntervalFunction ofExponentialRandomBackoff(long initialIntervalMillis, double multiplier, double randomizationFactor) {
        IntervalFunctionCompanion.checkInterval(initialIntervalMillis);
        IntervalFunctionCompanion.checkMultiplier(multiplier);
        IntervalFunctionCompanion.checkRandomizationFactor(randomizationFactor);
        return (attempt) -> {
            IntervalFunctionCompanion.checkAttempt((long)attempt);
            long interval = (Long)of(initialIntervalMillis, (x) -> {
                return (long)((double)x * multiplier);
            }).apply(attempt);
            return (long)IntervalFunctionCompanion.randomize((double)interval, randomizationFactor);
        };
    }
```

IntervalFunction의 initialIntervalMillis, multiplier, randomizationFactor 세가지 값을 통해 재시도 처리간 Exponential backoff and Jitter를 구성할 수 있습니다.

#### Exponential backoff

```java
long interval = (Long)of(initialIntervalMillis, (x) -> {
                return (long)((double)x * multiplier);
              }).apply(attempt);
```

초기 재시도 간격을 설정하는 `initialIntervalMillis`과 재시도 호출간 재시도 호출 시간 간격을 늘려주는 `multiplier`로 구성되어 있습니다.

#### Jitter

```java
return (long)IntervalFunctionCompanion.randomize((double)interval, randomizationFactor);

static double randomize(final double current, final double randomizationFactor) {
        double delta = randomizationFactor * current;
        double min = current - delta;
        double max = current + delta;
        return min + Math.random() * (max - min + 1.0);
}
```

randomize에서는 Exponential backoff값인 `initialIntervalMillis` * `multiplier`을 randomizationFactor라는 값을 통하여 랜덤화합니다.

아래 예시를 통해 randomize값이 어떻게 변하는지 확인해보세요.

**INITIAL_INTERVAL = 3, multiplier=3 일 경우 재시도 간격 분석**

> multiplier X INITIAL_INTERVAL = 9
> delta = 0.5 X 9 = 4.5
> min = 9 - 4.5 = 4.5
> max = 9 + 4.5 = 13.5
> nextInterval = 4.5 + ( random 수(0.0 ~ 1.0) X 9)
> => 4.5초 ~ 13.5 초 소모

#### Retry

이제 retry를 어떻게 할 것인지 설정하는 `RetryConfig`와 재시도 간격을 어떻게 할 것인지 설정하는 `IntervalFunction`에 대해 알아보았습니다. Retry는 이제 설정값을 넣어 재시도를 수행시켜주는 인터페이스입니다.

```java
    private RetryConfig getRetryConfigApplyJitter()    {
        IntervalFunction intervalFn = IntervalFunction.ofExponentialRandomBackoff(3,3,0.5);
        return RetryConfig.custom()
            .maxAttempts(MAX_RETRIES)
            .intervalFunction(intervalFn)
            .retryOnException(throwable -> true)
            .failAfterMaxAttempts(true)
            .build();

   }

    public Retry createRetry(String name) {
        return Retry.of(name, getRetryConfigApplyJitter());
    }
```

Retry 구현을 한뒤, 아래와 같이 사용할 수 있습니다.

```java
Retry retry = createRetry("for test");

try {
     response = retry.executeSupplier(() -> testRepository.test());
} catch (Exception e) {
     throw MaxRetriesExceededException.createMaxRetriesExceededException(retry);
}
```

RetryRegistry라는 레지스트리 기능을 하는 클래스도 존재하지만, 해당 글에서는 사용하지 않도록 하겠습니다.

### 주의사항!

디버깅을 하면서 재시도 로직이 정상적으로 예측 범위 내에서 동작하는지 확인하는 과정에서 이상한 점을 발견했습니다. 특이점은 다음과 같습니다.

1. **첫번째 시도를 재시도로 판단**합니다.
2. 두번째 재시도에는 **INITIAL_INTERVAL을 그대로 사용**합니다.
3. 3번째 시도부터 Exponential backoff and Jitter가 적용되는데 **Exponential backoff 값은 INITIAL_INTERVAL * DEFAULT_MULTIPLIER ^ (attempt-2) 값**이며 해당 상태에서 Jitter가 적용됩니다.

따라서 3번째부터 Exponential Backoff And Jitter 값이 적용되는 점 참고 부탁드립니다.

#### 예시

#### Jitter Logic

```
# randomize function (Math.random() =>  0.0 and less than 1.0.)

double delta = randomizationFactor * current;
double min = current - delta;
double max = current + delta;
return min + Math.random() * (max - min + 1.0);
```

> INITIAL_INTERVAL = 10000, DEFAULT_MULTIPLIER = 2.0, MAX_RETRIES = 5 인 경우
> randomFactor가 0인 경우 : 0 10 20 40 80 (2m 30s)
> randomFactor가 0.2인 경우 : 0, 10, 16 ~ 24, 32 ~ 48, 64 ~ 96, 128 ~ 192 (min 2m 8s, max 3m 12s)
> randomFactor가 0.4인 경우 : 0, 10, 12 ~ 28, 24 ~ 56, 48 ~ 112, 96 ~ 224 (min 1m 36s, max 3m 44s)

> INITIAL_INTERVAL = 4000, DEFAULT_MULTIPLIER = 2.0, MAX_RETRIES = 6인 경우
> randomFactor가 0인 경우 : 0 4 8 16 32 64 (2m 4s)
> randomFactor가 0.2인 경우 : 0, 4, 6.4 ~ 9.6, 12.8 ~ 19.2, 25.6 ~ 38.4, 51.2 ~ 76.8 (min 1m 42s, max 2m 34s)
> randomFactor가 0.4인 경우 : 0, 4, 4.8 ~ 11.2, 9.6 ~ 22.4, 19.2 ~ 44.8, 38.4 ~ 89.6 (min 1m 16s, max 2m 58s)

> INITIAL_INTERVAL = 6000, DEFAULT_MULTIPLIER = 2.0, MAX_RETRIES = 6인 경우
> randomFactor가 0인 경우 : 0 6 12 24 48 96 (3m 6s)
> randomFactor가 0.2인 경우 : 0, 6, 9.6 ~ 14.4, 19.2 ~ 28.8, 38.4 ~ 57.6, 76.8 ~ 115.2 (min 2m 34s, max 3m 51s)
> randomFactor가 0.4인 경우 : 0, 6, 7.2 ~ 16.8, 14.4 ~ 33.6, 28.8 ~ 67.2, 57.6 ~ 134.4 (min 1m 55s, max 4m 29s)

## 별첨.

Resilence4j에는 다양한 기능이 존재합니다. 해당 라이브러리를 통해 fault tolerance(결함 내성)에 대한 전반적인 이해도를 높여보셔도 좋을 것 같습니다.

읽어주셔서 감사합니다.

## Reference

- [https://en.wikipedia.org/wiki/Exponential_backoff](https://en.wikipedia.org/wiki/Exponential_backoff)
- [https://aws.amazon.com/ko/blogs/architecture/exponential-backoff-and-jitter/](https://aws.amazon.com/ko/blogs/architecture/exponential-backoff-and-jitter/)
- [https://resilience4j.readme.io/docs](https://resilience4j.readme.io/docs)
