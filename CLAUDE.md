# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Hugo 기반 개인 블로그 프로젝트. GitHub Pages로 배포 예정.

## Key Requirements (from README.md)

- Hugo 정적 사이트 생성기 사용
- GitHub repo를 통한 배포
- 심플한 테마 (인기 테마 중 선택)
- 마크다운 기반 콘텐츠 편집 및 쉬운 배포 워크플로우
- SEO 친화적이되, AI/크롤러 무단 수집 방지 (robots.txt, meta tag 등)
- 저작권 보호 장치 포함
- Docker 컨테이너화 가능성 고려
- 개인화된 md→html/css 디자인

## Commands

```bash
# Hugo 개발 서버 실행
hugo server -D

# 사이트 빌드
hugo

# 새 포스트 생성
hugo new posts/my-post.md
```

## Language

- 한국어로 응답
