# giscus 댓글 설정

이 저장소는 `giscus`를 댓글 시스템으로 사용한다. GitHub Discussions의 `General` category를 기준으로 글 URL 경로와 discussion을 연결한다.

## 현재 확인한 값

| 항목 | 값 |
| :--- | :--- |
| Repository | `xxh3898/xxh3898.github.io` |
| Repository ID | `R_kgDOSTVLdw` |
| Repository visibility | `public` |
| Discussions | `true` |
| Category | `General` |
| Category ID | `DIC_kwDOSTVLd84C8TcZ` |
| Mapping | `pathname` |

## 현재 설정

`_config.yml`의 댓글 설정은 아래 값을 기준으로 유지한다.

```yml
comments:
  provider: giscus
  giscus:
    repo: xxh3898/xxh3898.github.io
    repo_id: R_kgDOSTVLdw
    category: General
    category_id: DIC_kwDOSTVLd84C8TcZ
    mapping: pathname
    strict: 0
    input_position: bottom
    lang: ko
    reactions_enabled: 1
```

## 검증

설정을 반영한 뒤 로컬에서 빌드한다.

```bash
export PATH="/opt/homebrew/opt/ruby@3.4/bin:$PATH"
bundle exec jekyll build
bash tools/test.sh
```

배포 후 게시글 하단에 `giscus` 댓글 iframe이 표시되는지 확인한다.
