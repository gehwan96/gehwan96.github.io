---
title: 재시도 로직 미동작 이슈 회고
pubDatetime: 2022-07-29T09:00:00+09:00
description: spring-retry @Retryable이 동작하지 않은 원인을 JDK Dynamic Proxy와 CGLIB Proxy의 차이, final 메소드 제약으로 풀어냅니다.
tags:
  - incident-retro
  - spring-boot
---
## 개요

이번에도 어김없이 에러가 발생했다.. 이번 업무는 서비스 특정 로직에서 RuntimeException 혹은 네트워크 오류로 인해 에러가 발생할 경우 해당 메소드를 재시도하는 로직을 추가하는 것이였다.

재시도를 위해 구글링을 진행했고 spring-retry에서 제공하는 @Retryable이라는 깜찍한 녀석이 있었고 사용법도 쉬워보여 바로 적용했다가 정상적으로 처리가 되지 않는 문제가 있어 3일을 헤맸다...

따라서 오늘은 에러와 함께 @Retryable에 대해 알아보는 시간을 가지려고 한다.

### @Retryable이란?

@Retryable은 특정 Exception이 발생했을 경우 일정 횟수만큼 재시도할 수 있는 어노테이션이다. 사용자가 재시도를 원할 경우 해당 메소드 바로 위에 @Retryable 어노테이션을 붙여주면 저절로 재시도가 되는 것이다. 해당 어노테이션은 **Spring AOP를 기반으로 구현**되었으며 spring-retry를 사용할 시 spring-aop가 구현된 spring-aspects dependency 또한 추가해줘야 이용이 가능하다.

사용법은 다음과 같다

#### @Retryable 사용법

1. @Configuration 어노테이션을 사용한 클래스 바로 위에 @EnableRetry를 추가해준다.
2. 사용하고 싶은 메소드 위에 @Retryable을 붙여준다.
3. Exception이 발생해 재시도를 원하는 만큼 진행했지만 계속해서 Exception이 발생할 경우, @Recover 어노테이션을 통해 사후처리를 진행한다.

**예시**

```java
@Configuration
@EnableRetry
public class AppConfig {

}

public class TestService(){
	@Retryable
    public Response test() {
    	try {
        	System.out.println("test");
        } catch(CustomException e) {
			throw e;
        }
    }

    @Recover
    public Response recover(CustomException e){
    	return new Response();
    }
}
```

### 에러상황

문제는 다음 상황에서 발생했다.

```java
public interface A {

	@Retryable(includes = RuntimeException.class, maxAttempts = 3)
	public Response test();

   	@Recover
    public Response recover();
}

public abstract class B implements A {
	public void test();

    @Override
    public Response execute(){
    	this.test();
    }

    public Response recover();
}

public class C extends B {

	@Override
    public final test() {
    	try {
        	System.out.println("test");
        } catch(RuntimeException e) {
			throw e;
        }
    }

    @Override
    public Response recover() {
    	return new Response();
    }
}

public class RetryService {
	@Autowired
   	private C c;

    public run(){
    	c.test();
    }
}
```

클래스간의 관계는 다음과 같다.

```text
A extends B -> B implements C
```

- A Class: 재시도 로직이 추가된 인터페이스
- B Class: C 인터페이스를 구현하는 추상 클래스
- C Class: 추상클래스 B를 상속받은 C 인터페이스의 구현체

짧게 요약하자면 재시도 로직이 포함된 A인터페이스를 구현한 구현체 C를 실제로 동작시킬 경우, RuntimeException 에러가 발생하면 재시도를 해야하지만 재시도가 아닌 에러를 던지는 문제이다.

해당 에러는 어떻게 해결할까? 눈물을 흘리며 @Retryable에 대해 좀 더 알아보자

### @Retryable 분석

먼저 @Retryable을 사용하기 위해서는 dependency에 다음 두가지 의존성을 추가해야 한다.

```xml
		<dependency>
            <groupId>org.springframework.retry</groupId>
            <artifactId>spring-retry</artifactId>
			<version>1.3.3</version>
			<scope>provided</scope>
        </dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-aspects</artifactId>
			<version>5.2.20.RELEASE</version>
			<scope>provided</scope>
		</dependency>
```

보통 하나의 의존성만 추가하면 될텐데 이놈은 하나를 사용하려면 두 가지를 추가해야 된다니.. 참 그지같이 만들었다고 생각할 법하다. 이유를 알기 위해 앞서말한 @Retryable 사용법을 생각해보자

> @Configuration 어노테이션을 사용한 클래스 바로 위에 @EnableRetry를 추가해준다.

1. 사용하고 싶은 메소드 위에 @Retryable을 붙여준다.
2. Exception이 발생해 재시도를 원하는 만큼 진행했지만 계속해서 Exception이 발생할 경우, @Recover 어노테이션을 통해 사후처리를 진행한다.

먼저, @EnableRetry가 눈에 띈다. 이놈은 뭘까..?

#### @EnableRetry

먼저, EnableRetry에 코드를 확인해보자.

**@EnableRetry 코드**

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@EnableAspectJAutoProxy(
    proxyTargetClass = false
)
@Import({RetryConfiguration.class})
@Documented
public @interface EnableRetry {
    boolean proxyTargetClass() default false;
}
```

다른것보다 @EnableAspectJAutoProxy가 눈에 확 띈다. 이놈도 들어가보자.

**@EnableAspectJAutoProxy 코드**

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({AspectJAutoProxyRegistrar.class})
public @interface EnableAspectJAutoProxy {
    boolean proxyTargetClass() default false;

    boolean exposeProxy() default false;
}
```

AspectJAutoProxyRegistrar.class를 @Import로 감싼 문구가 있다. 해당 클래스까지만 가보자.

**AspectJAutoProxyRegistrar.class**

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
    AspectJAutoProxyRegistrar() {
    }

    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
        AnnotationAttributes enableAspectJAutoProxy = AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
        if (enableAspectJAutoProxy != null) {
            if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
                AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
            }

            if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
                AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
            }
        }

    }


public class AopConfigUtils {

    public static void forceAutoProxyCreatorToUseClassProxying(BeanDefinitionRegistry registry) {
        if (registry.containsBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator")) {
            BeanDefinition definition = registry.getBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator");
            definition.getPropertyValues().add("proxyTargetClass", Boolean.TRUE);
        }

    }
}
    public static void forceAutoProxyCreatorToExposeProxy(BeanDefinitionRegistry registry) {
        if (registry.containsBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator")) {
            BeanDefinition definition = registry.getBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator");
            definition.getPropertyValues().add("exposeProxy", Boolean.TRUE);
        }

    }

}
```

코드를 보면 registerBeanDefinitions 메소드에서 proxyTargetClass, exposeProxy의 boolean값을 보고 AopConfigUtils에서 강제로 프록시를 생성해주는 로직을 타고있는 것을 알 수 있다.

#### 코드 요약

1. @EnableAspectJAutoProxy를 @EnableRetry에서 호출한다. (이로인해 null이 아님)
2. @EnableAspectJAutoProxy에서는 설정한 proxyTargetClass boolean 값이 true일 경우 springframework에서 제공하는 BeanDefinition에 internalAutoProxyCreator가 프로퍼티 값으로 추가된다.
3. 2번에서 추가된 org.springframework.aop.config.internalAutoProxyCreator 가 Bean으로 정의될 경우 추후에 Application이 실행될 때 Bean 생성 과정에서 @Retryable을 정의한 메소드를 찾아 spring-aop를 적용시켜 재시도를 가능하게 만든다. 없을 경우엔 JDK Dynamic Proxy가 동적으로 생성해준다.
4. @EnableRetry에서 RetryConfiguration.class를 @Import로 선언하고 있는데 이로인해 3번 과정에서 internalAutoProxyCreator가 메소드를 찾을 경우 RetryConfiguration에서 buildPointcut을 진행해 해당 메소드의 pointcut을 만들어줘 해당 메소드를 실행시 재시도가 가능하도록 만들어준다.

이 때, @EnableAspectJAutoProxy에 proxyTargetClass가 무엇인지 해당 값에 대해 알아보자.

#### proxyTargetClass값의 의미

proxyTargetClass는 proxy를 붙일때 **어디에다가 붙일 것이냐**를 물어보는 값이다.
해당 값이 proxyTargetClass=false일 경우, 해당 클래스가 구현체인지 확인하며 인터페이스가 존재할 경우, 해당 구현체의 인터페이스인 A에 프록시를 생성한다.

반대로, proxyTargetClass=true일 경우, 해당 구현체를 상속받아 프록시를 생성한다.

**그게 뭔데 씹덕아..?**

![Bean - 인터페이스 유무에 따라 JDK Dynamic Proxy / CGLib 선택](/images/spring-retryable-not-working/img-01.png)

Spring Aop에서는 두 가지로 프록시를 생성할 수 있는데, 이 때 생성되는 과정에서 어떤 proxy로 생성되었느냐가 중요하다. 이유는 바로 **프록시가 무엇을 참조해 생성되었는지에 따라 Retry가 가능한지 여부가 달라지기 때문**이다.

앞서 proxyTargetClass=false(default)로 선언될 경우 C 클래스 구현체를 @Autowired해도 retry가 진행되지 않는다. 반면에, A 인터페이스를 @Autowired하면 해당 인터페이스에 프록시가 생성되었기 때문에 retry가 가능해진다.

반대로 proxyTargetClass=true가 될 경우, 구현체를 상속받아 프록시를 생성하기 때문에 C 클래스를 @Autowired해도 Retry가 정확히 잘 된다.

한 번 공부한 내용을 토대로 실전으로 들어가보자. 먼저 인터페이스에 프록시를 생성하는 JDK Dynamic Proxy를 적용해보겠다. default값이므로 아무것도 건드리지 않고 A 인터페이스를 Autowired해 test 메소드를 진행해보겠다.

그럴경우 다음과 같은 결과가 나온다. (로그는 저 코드가 아니라 retry 로직 아무거나 찍었다.)

![RetryTemplate Retry: count=0, count=1 로그](/images/spring-retryable-not-working/img-02.png)

인터페이스는 잘 되는 것을 확인했고... 난 구현체가 retry가 가능한 것을 확인해야 되기 때문에 CGLIB Proxy를 이용할 계획이다. 따라서 proxyTargetClass=true로 설정하고 진행하겠다.

**안되네?**

![java.lang.NullPointerException](/images/spring-retryable-not-working/img-03.png)

... 엥? 무슨 일일까.... 왜 안될까.. 이것만 있으면 분명 되는데..?

### Proxy

안되는 이유를 알아보기 전에 먼저 Spring AOP에 대한 전반적인 개념을 조금 더 보충할 필요가 있을 것 같다. Spring AOP는 Proxy의 메커니즘을 기반으로 AOP Proxy를 제공하고 있기 때문에 프록시가 무엇인지 가장 먼저 알아보는 시간을 가질 것이다.

프록시의 사전적 정의는 다음과 같다.

> authority given to a person to act for someone else, such as by voting for them in an election, or the person who this authority is given to

번역하자면 프록시란 누군가를 대신하여 뭔가를 수행하는 권한 자체 또는 그 권한을 받은 주체를 의미한다.

![Spring AOP Proxy 개념도](/images/spring-retryable-not-working/img-04.png)

앞서 말했듯이 Spring AOP는 Proxy의 메커니즘을 기반으로 AOP Proxy를 제공하고 있다. Spring AOP는 IOC Container의 Bean이 수행하는 기능을 **Proxy를 생성하여 권한을 주고** 해당 **proxy가 해당 Bean이 수행되기 전, 수행 중, 수행 이후 등 다양한 시점에서 개발자가 원하는 기능을 수행할 수 있도록 도움**을 준다.

이 때, Spring AOP에서 생성하는 Proxy에는 JDK Dynamic Proxy와 CGLIB Proxy 두가지로 나뉘는데 두 가지의 proxy는 각기 다른 방식으로 생성되기 때문에 좀 더 알아볼 필요가 있다.

---

#### JDK Dynamic Proxy vs CGLIB Proxy

#### JDK Dynamic Proxy

![JDK Dynamic Proxy 구조](/images/spring-retryable-not-working/img-05.png)

JDK Dynamic Proxy는 Reflection을 기반으로 Proxy를 생성한다. ProxyFactory에 의해 타깃의 인터페이스를 상속한 Proxy 객체를 생성한다. 이 때 주의할 것은 **인터페이스를 상속하여 Proxy 객체를 생성하기 때문에 AOP를 이용하려면 @Autowired시 인터페이스를 호출해야 원하는 기능을 정상적으로 동작시킨다.**

Proxy가 기본적으로 인터페이스에 대한 Proxy만을 생성해주기 때문에 JDK Dynamic Proxy를 이용할 경우 인터페이스로 구현한 **구현체의 주입**에 대해 좀 더 주의할 필요가 있다.

#### CGLIB Proxy

![CGLIB Proxy 구조](/images/spring-retryable-not-working/img-06.png)

CGLib은 Code Generator Library의 약자로, 클래스의 바이트코드를 조작하여 Proxy 객체를 생성해주는 라이브러리이다.

CGLIB Proxy 같은 경우, 인터페이스가 아닌 타깃의 클래스에 대해서도 Proxy를 생성할 수 있는데 Enhancer라는 클래스를 통해 Proxy를 생성할 수 있다.

Enhancer는 클래스에 대한 Proxy를 생성할 경우 **클래스를 상속**하여 Proxy를 생성하기 때문에 두 가지의 제한이 존재한다.

**클래스 상속을 통한 Proxy 생성시 주의사항**

- **final 상수**에 대한 메소드를 override할 수 없어 final 메소드는 AOP가 불가능하다.
- 상속을 통한 프록시 생성으로 재정의가 불가능하다.

---

#### 그럼 내 문제는..?

앞서 말했듯이 CGLIB Proxy를 이용할 경우 다음과 같은 주의사항이 있다.

- **Class를 상속받아 생성하기 때문에 Final 메소드 또는 클래스에 대해 재정의를 할 수 없으므로 Proxy를 생성할 수 없다.**
- -> 내 코드를 보면 구현체 C 클래스에 public final로 정의된 메소드를 볼 수 있다. **지금 내 문제는 final로 인해 프록시가 해당 메소드를 생성하지 못해 Retry가 정상적으로 동작하지 못하는 것이었다!**

### 결론

@Retryable을 사용할 때에는 다음과 같은 주의사항이 있다.

- @Retryable을 붙이는 클래스는 **인터페이스를 구현하는 구현체인가?** 에 따라 어떤 프록시를 사용할지 고려해야 한다.
  - JDK Dynamic Proxy
  - CGLIB Proxy
- CGLIB Proxy를 사용할 경우, **@Retryable할 메소드가 final로 정의되어 있는지 확인**한다.
- @EnableRetry는 **@Configuration을 정의한 클래스에 같이** 어노테이션을 붙여준다.
- @Recover 메소드는 **@Retryable에서 발생한 Exception을 파라미터로 받고 재시도하는 메소드를 리턴값으로 정의**해야 정상적으로 동작한다.

Retry를 사용하는 방법은 이 외에도 RetryTemplate, Retryer 등 다양한 방법이 있으니 좀 더 공부해봐도 좋을듯 하다.

### Reference

- [JDK Dynamic Proxy vs CGLIB Proxy](https://gmoon92.github.io/spring/aop/2019/04/20/jdk-dynamic-proxy-and-cglib.html)
- [Spring-retry](https://www.baeldung.com/spring-retry)
