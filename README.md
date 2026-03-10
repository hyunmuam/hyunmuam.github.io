# Hyun's Dev Blog

> 공부한 것을 기록하고 공유하는 개발 블로그. 다양한 기술 주제의 학습 과정과 경험을 정리합니다.

🔗 **방문하기 : [https://hyunmuam.github.io](https://hyunmuam.github.io)**

## 📚 다루는 주제 (Topics)

특정 분야에 한정하지 않고, 개발자로서 성장하며 배우고 경험한 모든 것을 기록합니다.
- **Backend / Frontend**: 프레임워크 학습 및 구현 경험
- **Troubleshooting**: 개발 중 겪은 문제와 해결 과정
- **Computer Science**: 자료구조, 알고리즘, CS 기초 지식
- **Retrospect**: 성장을 위한 회고 및 생각 정리

## 코드 저장소 (Code Repository)

블로그 포스트에 사용된 예제 코드 및 실습 프로젝트 코드는 아래 저장소에서 확인하실 수 있습니다.
- 🔗 **[hyunmuam/blog-code](https://github.com/hyunmuam/blog-code)**

## 🛠 기술 스택 (Tech Stack)

이 블로그는 다음 기술을 사용하여 구축되었습니다.
- **Static Site Generator**: [Jekyll](https://jekyllrb.com/)
- **Theme**: [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy)
- **Hosting**: GitHub Pages
- **Comments**: Giscus
- **Analytics**: Google Analytics

## 🚀 로컬 환경 실행 가이드 (Local Development)

공식 [Getting Started](https://chirpy.cotes.page/posts/getting-started/) 문서를 참고한 로컬 로컬 환경 구축 방법입니다.

### 1. 환경 설정 (Environment Setup)
Unix 계열 환경(macOS/Linux)에서 네이티브로 설정하거나, Docker 기반의 Dev Containers를 사용할 수 있습니다.

#### Native 설정 (macOS/Linux 권장)
1. 시스템에 [Ruby](https://www.ruby-lang.org/en/downloads/)와 [Git](https://git-scm.com/)이 설치되어 있는지 확인합니다.
2. [Jekyll 설치 가이드](https://jekyllrb.com/docs/installation/)를 참고하여 호스트 환경을 구성합니다.
3. 프로젝트 루트 디렉토리에서 패키지를 설치합니다:
   ```bash
   bundle install
   ```

#### Dev Containers 설정 (Windows 권장)
1. Docker Desktop과 VS Code를 설치합니다.
2. VS Code에서 [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)을 설치합니다.
3. 리포지토리를 클론하고 VS Code를 통해 **컨테이너 볼륨**에서 프로젝트를 엽니다. 환경 설정이 자동으로 진행됩니다.

### 2. 로컬 서버 실행 (Start Jekyll Server)

모든 의존성이 설치된 후, 다음 명령어로 로컬 서버를 구동합니다:

```bash
bundle exec jekyll serve
```

> **Tip:** 초안(Draft) 글도 함께 보거나 실시간으로 변경사항을 반영하려면 다음 명령어를 사용하세요:
> `bundle exec jekyll serve --drafts --livereload`

잠시 후 `http://127.0.0.1:4000` 에서 블로그를 확인할 수 있습니다.

## 📝 글 작성 템플릿 (Front Matter)

새 글을 작성할 때 활용하는 기본 포맷 파일 형식: `YYYY-MM-DD-제목.md`

```yaml
---
title: 글 제목
date: YYYY-MM-DD HH:MM:SS +/-TTTT
categories: [대분류]
tags: [태그1, 태그2] # TAG names should always be lowercase
---
```

## 📝 커밋 컨벤션 (Commit Convention)

혼자 운영하지만 일관된 저장소 관리와 리마인드를 위해 아래의 규칙을 따릅니다.

- `feat` : 새로운 기능 추가 (예: SEO 설정, UI/CSS 변경, 플러그인 추가)
- `post` : 새 글 작성 또는 임시 저장
- `edit` : 기존 글 내용 수정, 오타 교정, 내용 보강
- `chore` : 설정 파일 변경, 패키지 업데이트 (예: `_config.yml` 수정)
- `fix` : 버그나 오류 수정 (예: 깨진 링크 수정, 렌더링 오류 해결)

**✏️ 예시:**
```text
post: [Spring Boot] 캐시 적용해보기 포스팅
edit: [WebSocket] 오타 수정 및 내용 보강
chore: 구글 애널리틱스 설정 추가
```

## 📄 License

- **테마 소스코드**: 원작자(Cotes Chung)의 [MIT License](LICENSE)를 따릅니다.
- **블로그 포스트(콘텐츠)**: 별도로 명시되지 않은 한 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.ko) 라이센스를 따릅니다.
