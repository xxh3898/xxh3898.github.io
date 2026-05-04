---
title: 프로젝트
icon: fas fa-diagram-project
order: 4
---

공개 포트폴리오의 프로젝트를 블로그 글과 연결하기 위한 페이지입니다. 프로젝트 소개는 [포트폴리오](https://www.chochiho.cloud)에 두고, 이 블로그에서는 문제 해결 과정과 기술 판단을 글로 남깁니다.

## Cubing Hub

큐빙 기록, 학습, 랭킹, 커뮤니티, 피드백을 하나의 서비스 흐름으로 묶은 1인 풀스택 프로젝트입니다. 첫 달 블로그 글은 Cubing Hub의 인증, 랭킹 성능 개선, 테스트, 배포 검증을 중심으로 작성합니다.

| 항목 | 내용 |
| :--- | :--- |
| 기간 | 2026.03.23 - 2026.04.24 |
| 역할 | 1인 Full Stack, 기획/설계/개발/배포/검증 |
| Service | [www.cubing-hub.com](https://www.cubing-hub.com) |
| GitHub | [github.com/xxh3898/cubing-hub](https://github.com/xxh3898/cubing-hub) |
| Stack | Java 17, Spring Boot, Spring Security, JPA, QueryDSL, MySQL, Redis, React, Vite, AWS, Docker, Nginx |

작성 우선순위:

- Redis ZSET 기반 랭킹 읽기 모델 분리
- Access Token blacklist와 Refresh Token Rotation 설계
- Spring REST Docs와 MockMvc 기반 API 문서화
- Testcontainers로 MySQL/Redis 통합 테스트 구성
- S3, CloudFront, EC2, RDS 분리 배포와 운영 검증

## CalmDesk

5인 팀으로 진행한 B2B HR·웰빙 SaaS 프로젝트입니다. 담당 기능과 트러블슈팅 중 블로그에 남길 핵심은 관리자 팀 멤버 조회 API의 N+1 쿼리와 EC2 CPU 크레딧 고갈 문제입니다.

| 항목 | 내용 |
| :--- | :--- |
| 기간 | 2026.01.06 - 2026.02.27 |
| 역할 | Backend, 이슈 관리자 |
| Team GitHub | [github.com/Team-Code808](https://github.com/Team-Code808) |
| Fork | [github.com/xxh3898/CalmDeskBackend](https://github.com/xxh3898/CalmDeskBackend) |
| Stack | Java 17, Spring Boot, Spring Security, JPA, MySQL, Redis, WebSocket, SSE, Docker, GitHub Actions, AWS EC2, Nginx |

작성 후보:

- JPA N+1 쿼리를 fetch join과 bulk query로 줄인 과정
- k6 부하 테스트로 응답 지연과 RPS 개선을 검증한 과정
- WebSocket 연결에서 JWT 인증을 처리한 방식

## MediFlow

6인 팀으로 진행한 병원 ERP 프로젝트입니다. DB Lead 역할을 맡아 데이터 무결성과 SQL 구조 정리를 담당했습니다.

| 항목 | 내용 |
| :--- | :--- |
| 기간 | 2025.10.23 - 2025.11.19 |
| 역할 | DB Lead, Backend |
| GitHub | [github.com/Team-W3C/W3C-Project](https://github.com/Team-W3C/W3C-Project) |
| Stack | Java, Spring Boot, MyBatis, Oracle, JSP, JSTL |

작성 후보:

- DB 상태값 기준을 통일하고 `CHECK` 제약조건으로 무결성을 보장한 과정
- Oracle `VIEW`, `LISTAGG`, 서브쿼리로 반복 조회 구조를 줄인 과정
- 팀 개발 환경에서 더미 데이터와 스키마 기준을 맞춘 과정
