---
title: "Cubing Hub 랭킹 조회를 Redis ZSET으로 분리한 이유"
date: 2026-05-04 19:27:00 +0900
categories: [Troubleshooting, Database]
tags: [spring-boot, redis, mysql, ranking, performance, k6]
description: "MySQL 기반 랭킹 조회 병목을 Redis ZSET 읽기 모델로 분리한 과정과 검증 결과를 정리합니다."
---

Cubing Hub에는 큐빙 기록을 저장하면 종목별 PB(Personal Best)를 기준으로 전체 랭킹을 보여주는 기능이 있다. 처음에는 MySQL의 `user_pbs` 테이블을 기준으로 랭킹을 직접 조회했다. 데이터가 적을 때는 문제가 없었지만, 성능 검증용 데이터를 늘리자 랭킹 API가 서비스에서 가장 무거운 읽기 경로가 됐다.

이 글은 `GET /api/rankings`를 MySQL 직접 조회에서 Redis ZSET 읽기 모델로 분리한 이유와, 그 과정에서 유지하려고 했던 기준을 정리한 기록이다.

## 문제 상황

랭킹 조회는 쓰기보다 읽기가 훨씬 많은 기능이다. 사용자는 기록을 한 번 저장한 뒤에도 홈, 랭킹 페이지, 마이페이지를 오가며 같은 랭킹 데이터를 반복해서 조회한다.

기존 V1 구조는 MySQL의 `user_pbs`를 기준으로 매번 랭킹을 계산했다.

```text
GET /api/rankings
-> user_pbs 조회
-> eventType 기준 필터링
-> best_time_ms 기준 정렬
-> page / size 기준 응답
```

`300,000`건의 PB 데이터를 넣고 k6로 같은 시나리오를 반복 실행했을 때, V1 기준선은 다음과 같았다.

| 항목 | MySQL V1 |
| :--- | ---: |
| 평균 응답 시간 | 7,245.23ms |
| p95 | 12,429.58ms |
| 요청 처리율 | 4.21 req/s |
| 실패율 | 0.00% |

실패율은 0%였지만, 평균 응답 시간이 7초를 넘는 API를 정상이라고 보기는 어려웠다. 사용자가 보는 랭킹 화면에서는 이미 체감 지연이 큰 상태였다.

## 기존 구조의 한계

`user_pbs`는 사용자별, 종목별 대표 기록을 저장하는 기준 데이터다. 기록을 저장하거나 penalty를 바꾸거나 기록을 삭제하면 `records`를 기준으로 PB를 다시 계산하고 `user_pbs`를 갱신한다.

이 기준 데이터 자체는 필요하다. 문제는 매번 같은 정렬 결과를 만들기 위해 기준 데이터를 직접 읽는 구조였다.

랭킹 조회의 요구사항은 비교적 명확했다.

- 종목별 PB가 빠른 순서로 정렬되어야 한다.
- 동률이면 기록 생성 시각과 record id 기준으로 순서가 안정적이어야 한다.
- 응답의 `rank`는 검색 결과 안에서 다시 매긴 순위가 아니라 전체 랭킹 기준 순위여야 한다.
- Redis가 비어 있거나 준비되지 않은 상태에서도 API가 동작해야 한다.

그래서 해결 방향은 MySQL을 없애는 것이 아니라, 책임을 나누는 쪽으로 잡았다.

## 해결 방향

MySQL은 기준 데이터로 유지하고, 기본 랭킹 조회만 Redis 읽기 모델로 분리했다.

```text
records / user_pbs
-> 기준 데이터
-> PB 변경 시 Redis 읽기 모델 동기화
-> 기본 랭킹 조회는 Redis ZSET 사용
```

핵심은 Redis를 원본 데이터 저장소로 보지 않는 것이다. Redis는 조회 성능을 위한 보조 모델이고, 언제든 MySQL의 `user_pbs`를 기준으로 다시 만들 수 있어야 한다.

## Redis 읽기 모델 구조

Redis에는 종목별로 랭킹 데이터를 분리했다.

| Key | 타입 | 역할 |
| :--- | :--- | :--- |
| `ranking:v2:{eventType}:zset` | ZSET | 랭킹 정렬용 데이터 |
| `ranking:v2:{eventType}:nicknames` | HASH | `userId -> nickname` 조회 |
| `ranking:v2:{eventType}:members` | HASH | `userId -> member` 추적 |
| `ranking:v2:{eventType}:ready` | String | Redis 읽기 모델 준비 상태 |

ZSET의 `score`는 `best_time_ms`로 잡았다. 기록 시간이 낮을수록 좋은 랭킹이기 때문에 Redis ZSET의 오름차순 정렬과 잘 맞는다.

동률 처리는 `member` 문자열에 넣었다.

```text
{createdAtMillis}:{recordId}:{userId}
```

Redis ZSET은 score가 같으면 member 문자열을 기준으로 정렬한다. 그래서 member를 고정 길이 숫자 문자열로 만들면 `created_at asc -> record.id asc -> user.id asc` 순서를 유지할 수 있다.

## 조회 경로 분리

조회 경로는 단순하게 나눴다.

```java
boolean isRedisReady = rankingRedisService.isReady(eventType);

RankingPageResponse rankingPage = !StringUtils.hasText(nickname) && isRedisReady
        ? rankingRedisService.getRankings(eventType, page, size)
        : getRankingsFromMySql(eventType, nickname, page, size);
```

기본 랭킹 조회는 Redis가 준비되어 있을 때만 Redis를 사용한다. 반대로 아래 경우에는 MySQL 경로를 유지했다.

- `nickname` 검색 요청
- Redis ready key가 없는 상태
- Redis 읽기 모델을 재구축하기 전 상태

닉네임 검색까지 Redis로 옮기지 않은 이유는 범위 통제 때문이다. 부분 검색을 Redis에서 처리하려면 secondary index나 별도 검색 모델이 필요하다. 첫 개선의 목적은 가장 자주 호출되는 기본 랭킹 조회를 안정적으로 빠르게 만드는 것이었기 때문에, 검색은 기존 MySQL 경로를 유지했다.

## 쓰기 동기화

기록이 바뀌면 기준 데이터와 읽기 모델을 함께 맞춰야 한다.

Cubing Hub에서는 기록 저장, penalty 수정, 기록 삭제 후 PB가 바뀌는 경우 Redis 랭킹도 갱신한다.

```text
PB가 새로 생김 또는 변경됨
-> user_pbs 저장
-> rankingRedisService.sync(userPB)

PB가 삭제됨
-> user_pbs 삭제
-> rankingRedisService.remove(eventType, userId)
```

또한 운영 Redis가 비었거나 재구축이 필요할 때는 MySQL `user_pbs`를 기준으로 Redis 랭킹 모델을 다시 만들 수 있게 했다. 재구축이 끝나면 `ready` key를 세팅하고, 그 전까지는 MySQL fallback 경로를 사용한다.

이렇게 하면 Redis가 비어 있는 순간에도 API 계약을 깨지 않고 서비스를 유지할 수 있다.

## 검증 결과

같은 k6 시나리오에서 V1과 V2를 비교했다.

![Cubing Hub MySQL V1 랭킹 조회 k6 Grafana 결과](/assets/img/posts/2026-05-04-cubing-hub-ranking-redis-zset/rankings-mysql-v1.png)
_MySQL V1 기준 랭킹 조회 Grafana 대시보드_

![Cubing Hub Redis V2 랭킹 조회 k6 Grafana 결과](/assets/img/posts/2026-05-04-cubing-hub-ranking-redis-zset/rankings-redis-v2.png)
_Redis V2 기준 랭킹 조회 Grafana 대시보드_

| 항목 | MySQL V1 | Redis V2 | 변화 |
| :--- | ---: | ---: | ---: |
| 평균 응답 시간 | 7,245.23ms | 21.10ms | -99.71% |
| p95 | 12,429.58ms | 36.94ms | -99.70% |
| 최대 응답 시간 | 13,288.98ms | 94.53ms | -99.29% |
| 요청 처리율 | 4.21 req/s | 1,502.77 req/s | +35,610.49% |
| 실패율 | 0.00% | 0.00% | 유지 |

단순히 응답 시간이 줄어든 것보다 더 중요하게 본 지점은 실패율이 늘지 않았다는 점이다. 읽기 경로를 Redis로 옮기면서도 API 계약과 fallback 경로를 유지했기 때문에, 성능 개선과 안정성을 같이 가져갈 수 있었다.

## 배운 점

첫 번째로, Redis를 도입한다고 해서 기준 데이터를 Redis로 옮겨야 하는 것은 아니다. 이 경우 기준 데이터는 MySQL의 `records`와 `user_pbs`가 맡고, Redis는 읽기 성능을 위한 파생 모델로 두는 편이 더 안전했다.

두 번째로, 성능 개선은 빠른 경로만 만드는 것으로 끝나지 않는다. Redis가 준비되지 않은 상태, 재구축 중인 상태, 검색 요청처럼 Redis가 처리하지 않는 조건을 같이 설계해야 운영에서 깨지지 않는다.

세 번째로, 랭킹처럼 정렬 기준이 명확하고 읽기 빈도가 높은 기능은 읽기 모델 분리 효과가 크다. 반대로 검색, 필터, 정렬 조건이 자주 바뀌는 기능이라면 Redis ZSET만으로 해결하려고 하기보다 별도 검색 모델이나 MySQL 최적화를 다시 검토해야 한다.

이번 개선은 “MySQL이 느리니 Redis로 바꿨다”가 아니라, 기준 데이터와 조회 모델의 책임을 분리한 작업이었다. 그 차이를 분명히 해두는 것이 이후 운영과 장애 대응에도 더 유리했다.
