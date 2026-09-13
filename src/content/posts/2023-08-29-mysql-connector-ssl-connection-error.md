---
title: MySQL Connector SSL 접속 오류 해결 회고
pubDatetime: 2023-08-29T09:00:00+09:00
description: JDK 8 업데이트로 TLS 1.0/1.1이 비활성화되면서 구버전 MySQL Connector의 useSSL=true 접속이 실패한 원인과 해결 방안을 정리합니다.
tags:
  - incident-retro
  - mysql
  - java
---
## TL;DR

1. 배포 실패한 서비스는 5.1.42 버전의 mysql connector를 사용 중이었습니다.
2. 서비스 배포 전후 변경사항으로는 jdk 버전이 8u242b08에서 8u362b09으로 업데이트 되었습니다.
3. 변경된 8u362b09에서는 TLS 1.0, TLS 1.1 버전을 지원하지 않습니다.
4. mysql 5.5.45+, 5.6.26+, 5.7.6+ 버전에서는 명시적으로 useSSL=true/false를 작성해줘야 합니다. (*5.1.42도 포함되는 것 같음*)
5. useSSL=true의 경우, mysql connector 버전에 따라 사용되는 TLS 버전이 다릅니다. 현재 서비스에서 사용되는 버전은 5.1.42 버전으로 TLS 1.2 미만 버전입니다.
6. jdk에서 해당 TLS 버전을 지원해주지 않기 때문에 SSL 통신 에러가 발생합니다.
7. jdk 설정을 수정하거나, SSL 통신을 사용하지 않거나, mysql connector 버전을 최신화하여 문제를 해결할 수 있습니다.

## 회고 시작

회사 서비스 배포날. 어김없이 찾아온 오류에 대해 회고하는 시간을 가져보고자 합니다.

### 타임라인

> 07:55 : 서비스 배포 시작
> 08:07 : 서비스 배포간 mysql 접속 실패 발생
> 08:20 : JDBC 접속 url의 useSSL 설정 제거 후 배포 - 배포 실패
> 08:20 ~ 08:30 : 원인 파악 및 분석
> 08:40 : 원인 확인 후 배포 테스트 통과
> 09:17 : 서비스 배포 완료

### 배포 서비스 특징

mysql 서버 설정은 다음과 같습니다.

#### application.yml

```yaml
jdbc:
  url: jdbc:mysql://0.0.0.0:13306?allowMultiQueries=true&useSSL=true&useUnicode=true&characterEncoding=utf8
```

#### pom.xml

```xml
<dependency>
   <groupId>mysql</groupId>
   <artifactId>mysql-connector-java</artifactId>
   <scope>runtime</scope>
   <version>5.1.42</version>
</dependency>
```

### 배포 실패 원인 분석

배포간 인지한 내용은 다음과 같습니다.

#### 1. 서비스 내 코드 변경사항이 없었습니다.

해당 서비스는 변경사항이 적어 4월 11일에 배포된 이후 4달이 지난 8월 29일에 배포가 진행되었습니다. 약 4개월동안 변경사항은 거의 없었으며, mysql 통신과는 무관한 변경이었습니다.

#### 2. 잘 되는 mysql 서버와의 통신

서비스 배포간 통신 오류가 발생해 롤백을 한 결과 정상적으로 배포되어 통신간의 문제는 아닌것을 확인했습니다.

#### 3. 원인이 될 수 있는 다양한 기능들

실패 원인으로 회사 내 보안 정책으로 사용 중인 기능들을 완전히 배제할 수 없었습니다.

mysql과의 통신은 잘 되는 상태에서 갑작스럽게 통신이 안되는 원인이 서비스 내 코드의 변경사항이 아니었다는 결론이 도출되었고, 서비스에 영향을 줄 만한 변경사항이 4개월동안 어떤 것이 있었을지 확인해봤습니다.

### 4월 25일 배포간 변경사항

변경사항 중 [4월 25일 - Email TLS 장애 회고](https://velog.io/@gehwan96/Java-JDK-8-%EB%B2%84%EC%A0%84-%EC%97%85%EB%8D%B0%EC%9D%B4%ED%8A%B8%EA%B0%84-TLS-%EB%B2%84%EC%A0%84%EC%9C%BC%EB%A1%9C-%EC%9D%B8%ED%95%9C-%EB%B0%9C%EC%86%A1-%EC%8B%A4%ED%8C%A8-%EC%9D%B4%EC%8A%88-%ED%9A%8C%EA%B3%A0) 내용이 떠올랐습니다.

> jdk 버전이 openjdk 8u242b08에서 openjdk 8u362b09 버전으로 업데이트 되었습니다.

- 8u242b08에서 8u362b09 개선사항
  - TLS1.0 TLS1.1은 사용이 불가능하다.

해당 내용과 연관이 있을 것이라 파악하고 관련 내용을 찾아보았습니다. 아래 두 가지 내용을 토대로 원인이 파악되었습니다.

- [MySql SSL 설정 끄기, JDBC 연결 오류 해결 useSSL=false](https://codingcoding.tistory.com/995)
- [6.9 Connecting Securely Using SSL](https://dev.mysql.com/doc/connector-j/8.1/en/connector-j-reference-using-ssl.html)

#### mysql connector 버전별 특이사항

> Establishing **SSL connection without** server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements **SSL connection** must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. **You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true** and provide truststore for server certificate verification.

#### mysql connector 버전별 SSL 버전

> Since Connector/J 8.0.28, the connection property enabledTLSProtocols has been renamed to tlsVersions, and enabledSSLCipherSuites has been renamed to tlsCiphersuites; the original names remain as aliases.

- For Connector/J 8.0.26 and later: **TLSv1 and TLSv1.1 were deprecated** in Connector/J **8.0.26 and removed in release 8.0.28**; the removed values are considered invalid for use with connection options and session settings. Connections can be made using the more-secure TLSv1.2 and TLSv1.3 protocols. Using TLSv1.3 requires that the server be compiled with OpenSSL 1.1.1 or higher and Connector/J be run with a JVM that supports TLSv1.3 (for example, Oracle Java 8u261 and above).

> For Connector/J 8.0.18 and earlier when connecting to MySQL Community Server 5.6 and 5.7 using the JDBC API: Due to compatibility issues with MySQL Server compiled with yaSSL, Connector/J does not enable connections with TLSv1.2 and higher by default. When connecting to servers that restrict connections to use those higher TLS versions, enable them explicitly by setting the Connector/J connection property enabledTLSProtocols (e.g., set enabledTLSProtocols=TLSv1.2,TLSv1.3).

주요 내용은 다음과 같습니다.

- MySQL 5.5.45+, 5.6.26+, 5.7.6+ 버전에서는 명시적으로 `useSSL=false` 또는 `useSSL=true`를 명시해줘야 합니다.
- MySQL Connector는 8.0.18 이전 버전에서는 TLSv1.2 이상 버전을 기본으로는 지원하지 않습니다. 사용하고 싶은 경우 명시적으로 작성하여 이용해야 합니다.
- 8.0.26 버전에서 TLSv1 and TLSv1.1은 deprecated가 되었으며 8.0.28 버전부터는 제거되었습니다.

## 결론

배포 실패한 서비스는 5.1.42 버전의 mysql connector를 사용중이므로 TLS1.2 미만의 버전으로 통신을 시도하였고, 변경된 JDK로 인해 통신이 실패하였습니다.

## 해결 방안

해결 방안은 크게 세 가지가 존재했습니다.

### 1. DisableAlgorithm 설정을 통한 TLS1.0, 1.1 재사용

[[jdk] JAVA TLS 접속 보안 이슈 해결(TLSv1, TLSv 1.1)](https://espania.tistory.com/410) 과 동일한 방법입니다.

1. 설치된 jdk 의 java.security 파일을 vim 으로 접근합니다.
   `vim /usr/local/java/jre/lib/security/java.security`
2. TLSv1 검색하여 jdk.tls.disabledAlgorithms 라고 적혀있는 부분을 찾습니다.
3. 해당 부분에서 TLSv1, TLSv1.1 을 제거 하고 다음과 같이 설정해 준 후, 저장합니다.

### 2. useSSL=false로 수정합니다.

현재 발생하는 이슈는 useSSL=true로 설정하여 SSL 통신을 시도하지만 업데이트된 JDK에서 TLS 1.0, 1.1 통신을 지원하지 않기 때문에 발생한 이슈입니다.

useSSL=false로 수정할 경우 SSL 통신을 하지 않기 때문에 정상적으로 mysql 서버와 통신이 가능해집니다.

### 3. mysql connector 버전을 올립니다.

mysql connector 버전을 8.0.26 버전 이상으로 최신화합니다.

팀에서는 해당 해결방안 중 SSL 통신이 꼭 필요하지는 않아 2번 방안으로 수정을 진행해 손쉽게 문제를 해결할 수 있었습니다.
