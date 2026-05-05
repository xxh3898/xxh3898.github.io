---
title: "malformed refresh_token이 로그인까지 막을 때 복구 endpoint를 분리한 이유"
date: 2026-05-05 22:46:00 +0900
categories: [Troubleshooting, Auth]
tags: [spring-security, jwt, refresh-token, cookie, react]
description: "오염된 refresh_token cookie가 로그인과 세션 복구를 함께 막는 문제를 cookie Path 계약과 복구 endpoint 분리로 해결한 과정을 정리합니다."
---

Cubing Hub는 Access Token을 메모리에 저장하고, Refresh Token은 `HttpOnly` cookie로만 전달한다. 새로고침하면 프런트는 `refresh -> /api/me` 순서로 사용자 컨텍스트를 복구하고, 보호 API에서 `401`이 나면 refresh 후 원래 요청을 한 번 다시 보낸다.

이 구조는 브라우저 저장소에 Access Token을 남기지 않는다는 장점이 있다. 대신 Refresh Token cookie가 비정상 상태로 남았을 때는 프런트에서 직접 지울 수 없다. `HttpOnly` cookie는 JavaScript로 접근할 수 없기 때문에 서버가 만료 응답을 내려줘야 한다.

이번 문제는 토큰 검증 로직 자체보다 cookie의 `Path` 범위와 복구 endpoint 위치가 맞물리면서 생겼다.

## 문제 상황

사용자가 브라우저에 남은 `refresh_token` 값을 변조하거나, 어떤 이유로 비정상 값이 남으면 앱 초기 bootstrap refresh가 실패했다. 여기까지만 보면 일반적인 인증 실패처럼 보인다.

문제는 그 다음이었다. 사용자가 다시 로그인하려고 해도 `POST /api/auth/login` 요청까지 흔들렸다. 사용자는 "refresh가 실패했다"가 아니라 "로그인이 고장 났다"로 받아들이는 상태가 된다.

흐름을 단순화하면 아래와 같다.

```text
브라우저에 malformed refresh_token cookie가 남음
-> 앱 초기 refresh 실패
-> 사용자가 다시 로그인 시도
-> /api/auth/login 요청에도 같은 cookie가 자동 전송됨
-> login과 bootstrap이 같은 오염 상태를 공유함
```

Refresh Token은 서버가 새 Access Token을 발급할 때만 쓰는 값이다. 그런데 그 cookie가 로그인 요청에도 붙으면, 새 세션을 시작해야 할 경로가 이전 세션의 오염 상태에 영향을 받는다.

## 원인 후보

처음에는 로그인 API나 CORS, JWT 파싱 로직을 의심하기 쉽다. 실제로 인증 장애가 브라우저에서는 단순한 `400`이나 네트워크 오류처럼 보일 때가 많기 때문이다.

하지만 이 경우 핵심은 아래 세 가지였다.

- `refresh_token` cookie가 `Path=/api/auth`로 발급된다.
- 브라우저는 cookie `Path`와 맞는 요청에 해당 cookie를 자동으로 붙인다.
- `/api/auth/login`, `/api/auth/refresh`, `/api/auth/logout`이 같은 cookie 영향권에 들어간다.

즉 malformed cookie가 남으면 refresh뿐 아니라 login에도 같은 값이 실릴 수 있다. 인증 성공 흐름만 보면 큰 문제가 없어 보이지만, 실패 복구 흐름에서는 login과 refresh가 너무 가까운 경계에 묶여 있었다.

## 최종 원인

최종 원인은 invalid token 일반론이 아니라 cookie scope 설계 문제였다.

`Path=/api/auth`는 refresh API에 cookie를 보내기에는 편하다. 하지만 같은 경로 아래에 login, refresh, logout, 복구 endpoint를 모두 두면 문제가 생긴다.

특히 cookie를 지우는 API를 `/api/auth` 아래에 두면 복구 요청 자체도 같은 bad cookie를 다시 받는다.

```text
POST /api/auth/clear-refresh-cookie
-> 문제 cookie가 다시 자동 전송됨
-> 복구 endpoint도 오염된 요청이 됨
```

복구 endpoint가 존재해도, 그 endpoint가 원인 cookie의 `Path` 안에 있으면 사용자는 여전히 빠져나오기 어렵다.

## 해결

해결 방향은 복구 endpoint를 원인 cookie의 path 밖으로 분리하는 것이었다.

```text
POST /api/session/clear-refresh-cookie
```

이 endpoint는 `/api/auth` 아래에 두지 않는다. 그러면 브라우저는 `Path=/api/auth`인 `refresh_token` cookie를 `/api/session/...` 요청에는 자동으로 붙이지 않는다.

요청은 오염된 cookie 없이 들어오고, 서버는 응답에서 기존 cookie와 같은 이름과 `Path=/api/auth`를 지정해 만료 처리한다.

```text
Set-Cookie: refresh_token=; Path=/api/auth; Max-Age=0; HttpOnly; ...
```

여기서 중요한 점은 "복구 요청을 오염시키지 않는 것"과 "기존 cookie와 같은 Path로 만료시키는 것"을 분리해서 보는 것이다. 요청 경로는 `/api/session`이지만, 만료 대상 cookie의 Path는 원래 발급 Path와 맞춰야 한다.

프런트도 모든 실패에서 무조건 복구를 시도하지 않게 했다. guest 첫 방문처럼 `refresh_token`이 없는 정상 상황까지 매번 cleanup 대상으로 보면, 정상 흐름과 장애 흐름이 섞인다.

그래서 복구 시도는 아래처럼 실제 cookie 오염 가능성이 큰 경우로 좁혔다.

- malformed refresh token으로 판단되는 경우
- Refresh Token 재사용 감지처럼 세션 전체 정리가 필요한 경우
- 인증 재시도 중 `401`이 반복되는 경우
- 브라우저 또는 네트워크 레벨에서 요청이 비정상적으로 거부되는 경우

반대로 단순 `missing cookie`는 정상적인 비로그인 방문에서도 발생할 수 있으므로 복구 루틴의 기본 대상에서 제외했다.

## 검증

검증은 성공 흐름보다 실패 복구 흐름을 중심으로 봤다.

첫 번째로, 잘못된 `refresh_token` cookie가 남아 있을 때 앱 초기 refresh가 실패해도 사용자 컨텍스트가 정리되는지 확인했다.

두 번째로, 같은 상태에서 로그인 화면으로 이동한 뒤 복구 endpoint를 호출하면 기존 cookie가 만료되고, 이후 로그인 요청이 이전 bad cookie에 다시 묶이지 않는지 확인했다.

세 번째로, guest 첫 방문처럼 cookie가 없는 정상 흐름에서 불필요한 clear 요청이 반복되지 않는지 확인했다. 복구 로직이 너무 넓으면 장애는 줄어도 요청 노이즈와 원인 분리 비용이 늘어난다.

마지막으로 REST Docs와 인증 설계 문서를 함께 맞춰, 새 endpoint가 임시 우회가 아니라 인증 계약의 일부로 남도록 정리했다.

## 배운 점

Refresh Token cookie를 쓰는 구조에서는 인증 성공 흐름만으로 설계를 판단하면 부족하다. 실제 사용자는 정상 token보다 비정상 cookie가 남은 상태에서 더 큰 장애를 겪는다.

`HttpOnly` cookie는 보안상 장점이 있지만, 프런트가 직접 삭제할 수 없다는 운영상 제약도 함께 생긴다. 그래서 서버 주도 만료 endpoint와 그 endpoint의 위치까지 인증 설계 안에 포함해야 한다.

또한 복구 endpoint는 "있다"보다 "원인 cookie의 영향을 받지 않는 위치에 있다"가 더 중요하다. `Path`가 맞지 않으면 복구 요청 자체가 같은 문제를 다시 들고 들어올 수 있다.

이번 해결은 토큰 알고리즘을 바꾼 일이 아니었다. 브라우저 cookie 계약과 서버 인증 경로를 분리해, 사용자가 오염된 세션 상태에서 다시 로그인할 수 있는 출구를 만든 작업이었다.
