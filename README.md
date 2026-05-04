# xxh3898.github.io

`xxh3898`의 GitHub Pages 기술 블로그입니다.

## 목적

- Java, Spring, Database, Infra 문제 해결 과정을 공개 글로 정리합니다.
- PKM에 남긴 프로젝트 기록과 트러블슈팅을 공개 가능한 기술 글로 다듬습니다.
- 포트폴리오 `https://www.chochiho.cloud`와 연결해 백엔드 개발 역량의 근거 자료로 사용합니다.

## 기술 구성

- GitHub Pages
- Jekyll
- Chirpy starter
- GitHub Actions Pages deploy
- giscus comments
- self-hosted Chirpy static assets

## 로컬 Ruby

Chirpy `7.5.x`는 Ruby `3.1` 이상 `4.0` 미만을 요구합니다. macOS 기본 Ruby `2.6.x`로 실행하지 않습니다.

Homebrew 기준:

```bash
brew install ruby@3.4
export PATH="/opt/homebrew/opt/ruby@3.4/bin:$PATH"
ruby -v
bundle -v
```

## 로컬 실행

```bash
export PATH="/opt/homebrew/opt/ruby@3.4/bin:$PATH"
bundle install
bundle lock --add-platform x86_64-linux
bundle exec jekyll serve
```

로컬 URL:

```text
http://127.0.0.1:4000
```

## 검증

```bash
export PATH="/opt/homebrew/opt/ruby@3.4/bin:$PATH"
bundle exec jekyll build
bash tools/test.sh
```

## 글 작성 규칙

- 글 파일은 `_posts/YYYY-MM-DD-title.md` 형식으로 작성합니다.
- 파일명은 영어 kebab-case로 작성합니다.
- `date`에는 `+0900` timezone을 포함합니다.
- 모든 글은 `title`, `date`, `categories`, `tags`, `description`을 작성합니다.
- 이미지가 있으면 `alt`를 반드시 작성합니다.
- 공개 글에는 secret, 실제 `.env`, 운영 token, 고객사/회사 내부정보를 넣지 않습니다.

문제 해결형 글 구조:

```markdown
---
title: "문제 제목"
date: 2026-05-04 21:00:00 +0900
categories: [Troubleshooting, Infra]
tags: [docker, aws, deployment]
description: "문제의 원인과 해결 과정을 요약합니다."
---

## 문제 상황

## 환경

## 원인 후보

## 확인 과정

## 최종 원인

## 해결

## 검증

## 배운 점
```

## 배포

GitHub Actions의 `Build and Deploy` workflow가 `main` branch push 또는 수동 실행 시 Pages artifact를 build/deploy합니다.

GitHub repository Settings에서 Pages source는 `GitHub Actions`로 설정합니다.
