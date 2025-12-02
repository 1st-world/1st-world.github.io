---
title: "Chirpy 테마로 내 GitHub 블로그 만들기 (1부)"
author: 1st-world
date: 2025-10-10 02:30:00 +0900
last_modified_at: 2025-10-11 23:30:00 +0900
categories: [Blogging]
tags: [tutorial, github-pages, jekyll, chirpy]
pin: false
---

## 프롤로그: 개발자에게 블로그란?

개발을 공부하며 마주치는 수많은 지식과 오류들. 어딘가에 기록하지 않으면 휘발되기 마련입니다. 저 역시 학습 내용을 체계적으로 정리해서 기록하고 싶다는 생각에 개발 블로그를 만들기로 결심했습니다.

여러 선택지 중 제가 정한 조건은 다음과 같았습니다.

1. **완전 무료:** 개발 및 유지 비용이 발생하지 않아야 합니다.
2. **Markdown 지원:** 개발자에게 익숙하고 편한 방식으로 글을 쓸 수 있어야 합니다.
3. **쉬운 유지보수:** 글을 추가하고 관리하는 과정이 복잡하지 않아야 합니다.
4. **준수한 디자인:** 복잡한 커스터마이징 없이도 쓸 만한 디자인이어야 합니다.

이 모든 조건을 만족하는 조합 중 하나가 바로 **GitHub Pages + Jekyll (Chirpy 테마)**였습니다. 이 글은 저처럼 처음 개발 블로그를 시작하는 분들을 위해, 제가 겪었던 설치 및 설정 과정을 기록한 가이드입니다.

---

## 1. 블로그 생성: 로컬에 아무것도 설치하지 않고 시작하기

가장 먼저 마주한 궁금증은 'Ruby 같은 것까지 내 컴퓨터에 설치해야만 쓸 수 있나?'였습니다. 다른 많은 가이드에서 설치를 언급했지만, 결론부터 말하면 **Chirpy 테마를 그대로 사용하는 경우, 아무것도 설치할 필요가 없습니다.**

모든 과정은 GitHub 내에서 진행 가능합니다.

### 1-1. Chirpy Starter 템플릿으로 저장소 만들기

Chirpy 테마는 기본 템플릿을 제공합니다. [Chirpy Starter 템플릿 저장소](https://github.com/cotes2020/chirpy-starter)로 이동한 후, 우측 상단의 <kbd>Use this template</kbd> → <kbd>Create a new repository</kbd> 버튼을 눌러 나의 새로운 저장소를 만들 수 있습니다.

### 1-2. 저장소 이름 변경

저장소 이름(Repository name)은 `[내 GitHub 아이디].github.io` 형식을 준수해야 합니다. 제 계정(`1st-world`)을 예시로 들자면 `1st-world.github.io` 형태가 되는 거죠. 이 규칙을 지켜야 GitHub Pages가 정상적으로 블로그를 호스팅해 줍니다.

만약 저장소 이름을 이미 마음대로 지어서 생성했다면, 해당 저장소의 `Settings` 탭으로 이동하여 저장소 이름을 `[내 GitHub 아이디].github.io` 형식으로 변경해야 합니다.

---

## 2. 블로그 설정: `_config.yml` 파일 정복하기

블로그의 설정 대부분은 루트 디렉토리에 있는 `_config.yml` 파일 하나로 관리할 수 있습니다. 처음 열어보면 내용이 많아 부담스러울 수 있지만, 핵심적인 부분만 수정하면 됩니다.

### 2-1. 필수 정보 입력하기

블로그를 내 것으로 만들기 위해 아래 항목들을 수정합니다.

```yaml
# 웹 페이지의 기본 언어 설정
lang: en

# 글 발행 시간 등을 표시하기 위한 시간대 설정
timezone: Asia/Seoul

# 블로그 제목 입력
title: 1st-world

# 블로그 제목 하단에 표시할 부제(subtitle) 혹은 짧은 소개 입력
tagline: Hello World! 나의 첫 번째 세계

# 검색 엔진에 노출할 블로그 설명 입력
description: >- # used by seo meta and the atom feed
  웹 및 앱 개발, 클라우드, AI 등 프로그래밍 관련 다양한 내용을 기록하고 공유하는 블로그입니다.

# 내 블로그 주소 입력 (끝에 '/' 붙이지 말 것)
url: "https://1st-world.github.io"

github:
  # 내 GitHub 아이디(username) 입력
  username: 1st-world

twitter:
  # 내 Twitter 아이디 입력 (계정이 없거나 입력하기 싫으면 무시)
  username: 

social:
  # 글 작성 시 기본 저자 및 Footer 내 저작권에 표시할 이름 입력
  name: 1st-world
  # 이메일 주소 입력
  email: example@gmail.com
  # 소셜 링크 목록 (프로필 영역에 아이콘으로 표시)
  links:
    # 첫 번째 URL은 Footer 내 저작권자 이름에 연결
    # - https://twitter.com/username # change to your Twitter homepage
    - https://github.com/1st-world
    # Uncomment below to add more social links
    # - https://www.facebook.com/username
    # - https://www.linkedin.com/in/username
```

그 외에도 다양한 요소들을 선택적으로 수정할 수 있습니다.

* `webmaster_verifications` : Google, Bing 등 검색 엔진에 블로그 등록 및 소유권 인증 시 설정
* `analytics` : 방문자 집계 등을 위한 웹 분석 도구 설정
* `theme_mode` : 테마 색상 모드를 고정하려면 light 또는 dark 중 선택 (비워두면 시스템 설정에 따름)
* `comments` : 댓글 기능 설정 (사용할 댓글 시스템 선택 등)
* `pwa` : 블로그를 모바일 환경에서 앱처럼 '홈 화면에 추가' 가능하게 하는 기능

### 2-2. 프로필 사진(Avatar) 및 Favicon 설정

#### Avatar 설정

`avatar` 항목에 프로필 사진 경로를 입력해야 합니다. 보통 `/assets/img/` 폴더에 사진 파일을 저장한 후 지정합니다. 그런데 `/assets` 폴더에 `img` 폴더가 없더군요. GitHub 웹사이트에서 폴더를 만드는 팁은 다음과 같습니다.

1. `/assets` 폴더로 이동
2. `Add file` > `Create new file` 선택
3. 파일 이름 입력란에 `img/.gitkeep` 이라고 입력 후 커밋  
이렇게 하면 `img` 폴더가 생성됩니다.

생성된 `/assets/img/` 폴더에 프로필 사진을 업로드하고, `_config.yml` 파일의 `avatar` 항목에 `/assets/img/[내 사진].png` 와 같이 경로를 적어줍니다.

```yaml
# the avatar on sidebar, support local or CORS resources
# 사이드바에 표시할 프로필 사진 (사진 업로드 후 해당 경로 입력)
avatar: /assets/img/my_profile_photo.png
```

#### Favicon 설정

블로그 탭에 표시되는 작은 아이콘인 Favicon은 그냥 `/assets/img/favicons/` 폴더에 파일들을 업로드하기만 하면 됩니다. 원하는 이미지를 Favicon으로 변환 가능한 웹사이트가 정말 많은데 그중 마음에 드는 걸 이용하면 돼요. 저는 [RealFaviconGenerator](https://realfavicongenerator.net/) 에 이미지를 업로드한 후 변환된 Favicon 파일들을 받아 그대로 사용했습니다.

### 2-3. `baseurl`은 건드리지 않기

`baseurl` 항목은 `""` (빈 문자열) 그대로 두었습니다. 여기서 `baseurl`은 블로그 주소에서 도메인 뒤에 붙는 '하위 폴더' 경로를 지정할 때 사용합니다. GitHub Pages에는 두 가지 종류의 주소가 있기 때문인데요.

1. **사용자 페이지 (User/Organization Page)**
  * **저장소 이름:** `[아이디].github.io`
  * **블로그 주소:** `https://[아이디].github.io`
  * **설명:** 블로그가 도메인의 최상위 경로(`/`)에 바로 위치합니다. 하위 폴더가 없으므로 `baseurl`은 비워두어야 합니다(`""`).

2. **프로젝트 페이지 (Project Page)**
  * **저장소 이름:** `my-project` (일반적인 저장소 이름)
  * **블로그 주소:** `https://[아이디].github.io/my-project`
  * **설명:** 블로그가 `/my-project` 라는 하위 폴더 경로에 위치합니다. 이런 경우에는 `baseurl: "/my-project"` 라고 설정해 주어야 합니다.

우리는 앞서 저장소 이름을 `[아이디].github.io`로 만들었기에 `baseurl`은 빈 문자열로 두어야 합니다.

## 1부 끝

여기까지 진행했다면, 기본적인 사용자 정보와 프로필이 적용된 나만의 블로그가 성공적으로 생성됐을 거예요. `https://[내 GitHub 아이디].github.io` 주소로 접속해 확인해보세요!

[다음 포스트](/posts/creating-github-blog-with-chirpy-2/)에서는 실제로 블로그에 글을 작성하는 방법, Front Matter 작성법이나 파일 이름 규칙 등 기본적인 포스트 관리를 위해 꼭 알아야 하는 부분을 다루겠습니다.
