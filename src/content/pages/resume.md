---
title: "Resume"
description: "박세환 경력기술서 — Java & Spring Boot 백엔드 개발자"
---

## 박세환

- 이메일: gehwan96@gmail.com
- GitHub: [github.com/gehwan96](https://github.com/gehwan96)

5년차 Java & Spring Boot 백엔드 개발자입니다.

현재 쿠팡, 무신사, 배달의민족 등 대형 고객사가 사용하는 **월 평균 3~4억 건 규모의 Email 서비스를 개발·배포·운영**하고 있습니다. 대규모 메시징 서비스의 안정성과 확장성을 고민하며, 통합 발송 플랫폼인 Notification Hub와 카카오톡 브랜드 메시지의 설계 및 개발에도 참여해 **초기 기획부터 서비스 출시까지 전 과정을 경험**했습니다.

개발뿐 아니라 **팀의 생산성과 협업 효율을 높이는 환경을 만드는 것**에도 관심을 두고 있습니다. 레거시 리팩토링, 신규 입사자 개발 가이드, PR 템플릿과 장애 히스토리 문서화로 팀의 개발 효율을 끌어올렸고, 최근에는 AI를 활용해 서비스 운영 대시보드를 구축하고 성능테스트 플랫폼을 개발하는 등 **새로운 기술과 아이디어를 빠르게 검증하고 실제 업무에 적용**하고 있습니다.

## 기술 스택

| Category | Skills |
| --- | --- |
| Backend | Java (8/11/21), Kotlin, Spring Boot 2/3 |
| Data | MySQL, MongoDB, JPA, MyBatis, QueryDSL |
| Infrastructure | Kubernetes, Nginx, RabbitMQ |
| DevOps & Observability | Jenkins, SonarQube, GitHub Actions, Grafana |
| Testing & Architecture | JUnit 4/5, OpenAPI 3, Hexagonal Architecture |
| AI Development Tools | Codex, Claude Code |

## 근무 이력

### NHN Cloud · 메시징플랫폼개발팀 · 백엔드 개발자

_2022.02 ~ 현재_

- **Email 서비스 개발 (주 담당 업무)**
  - API 신규 설계 및 개발
  - 시스템 구조 개선 (Legacy Code Refactoring, Re-architecture)
  - 표준화 수행(RFC 적용): RFC 7208, RFC 6376, RFC 7489, RFC 5322
  - 발송 성능 개선
  - 서비스 운영 및 관리, 모니터링
- **카카오톡 브랜드 메시지 개발 (주 담당 업무)**
  - API 신규 설계 및 개발
  - 서비스 출시 참여
  - 서비스 운영 및 관리, 모니터링
- **Notification Hub**: Email, SMS, RCS, KTB, Push 통합 발송 플랫폼 개발
  - API 신규 설계 및 개발
  - 서비스 설계/개발/출시 참여
- **인프라 Kubernetes 기반 전환**
  - Notification 전체 상품 **VM → Kubernetes(NKS) 기반 전환**
  - 컨테이너 기반 서비스 아키텍처 **설계 및 구성**

### TODO(회사명) · 백엔드 개발자

_2021.01 ~ 2022.02_

- **Monolith 시스템 Microservice 시스템 전환 작업**
- LG Uplus 전산 시스템 모바일 단말 판매 기능 개발

## 업무 경험

### Notification 상품 Kubernetes 기반 전환

_2025.07 ~ 2026.06_

Notification 상품(Email, SMS, Push, KTB) **VM → Kubernetes(NKS) 기반 인프라 전환**

- 내부 아키텍처 설계 및 Helm 차트 작성
- 개발, 서비스 환경 구성
- Email 서비스 환경 전환

### 브랜드 메시지 서비스 개발

_2025.01 ~ 현재_

카카오 신규 서비스 브랜드 메시지 API 개발 참여

- 발송 API 개발
- 사용자 가이드 작성 및 서비스 출시 총괄

### Notification Hub 서비스 개발

_2023.07 ~ 2023.10, 2024.02 ~ 2024.12_

신규 서비스 Notification Hub 설계 및 개발 참여

- 첨부파일, 상세설정 API 설계 및 개발
- API 예외 응답 다국어 처리 구조 설계 및 개발
- Email 발송 API 개발
- 미터링 개발
- 사용 기술: `Hexagonal architecture`, `DDD`, `OpenAPI specification 3`, `Kotlin`, `RabbitMQ`, `MongoDB`, `MySQL`

### Email 발송 로직 리팩토링

_2023.11 ~ 2024.01_

- **3개 모듈에 중복 구현**된 2000여 줄의 발송 로직 리팩토링
- 절차지향적 코드를 객체지향적으로 개선하여 유지보수성 향상
- 코드 라인 수 60% 감소 및 책임 분리를 통해 가독성 개선

### Gmail 발신자 정책 변경 대응

_2023.11 ~ 2024.02_

2024년 2월 Gmail 발신자 정책 변경에 따른 대응 작업 수행

- DMARC 검증 시나리오 추가 및 API 유효성 로직 개선 (RFC 7208, RFC 7489)
- 도메인 인증 기능 강화를 통한 서비스 도메인 평판 관리 성능 개선
- 원클릭 수신거부 기능 제공

### 안정화 TF

_2023.08 ~ 2023.09_

팀 내에서 발생하는 장애 원인을 분석하고 개선하는 업무 수행

- Watchdog 및 Grafana 모니터링 알람 개선
- MDC 및 logback을 활용한 로깅 고도화
- SonarQube 도입
- Grafana 대시보드 시각화를 통한 운영 업무 개선

### Email 서비스 내부 개선

_2022.04 ~ 2025.04_

- 인프라 이중화 작업으로 **서비스 안정성 확보**
- 운영용 API 개발로 **서비스 장애 대응 시간 단축**
- 멀티 모듈 프로젝트로 재구성 및 공통화로 **코드 재사용성 강화**
- Nginx Reverse Proxy 및 HTTP Method Blocking 설정으로 **API 무작위 공격 대응**
- 커서 기반 조회 도입으로 **조회 API slow query 개선**
- 내부 발송 로직 분석·최적화로 **발송량 20% 증가, 최종 응답 soft bounce 90% 감소**
- 테스트 코드 수행 시간 지연 원인 분석·개선으로 **4분 54초 → 2분 07초 단축**
- CI 빌드 속도 최적화로 **수행 시간 20% 단축**
- 서비스 전체 테스트 코드 JUnit 5 마이그레이션 완료

### 사내 오픈소스 Starter Library 개발

_2022.04 ~ 2022.07, 2024.10_

- 외부 시스템과의 통신 검증 기능 개발
- MongoDB, Redis 등 다양한 외부 서버 연동 지원
- JDK 8 / 11 / 21 호환성 제공
- Autoconfiguration을 활용한 Starter 라이브러리 개발

### Ucube 시스템 개발

_2021.01 ~ 2022.02_

- 모바일 단말 판매 로직 개발
- Monolith Architecture에서 Microservice Architecture 전환 경험
- 사용 기술: `Spring Boot`, `Java 11`, `JPA`, `QueryDSL`

## 자격증

- CKA (Certified Kubernetes Administrator) — 2026.01.25 ~ 2028.01.24
