# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GitHub Posting Box는 Tistory 블로그의 최신 게시글을 GitHub README에 자동으로 업데이트하는 Java 애플리케이션입니다. GitHub Actions를 통해 매일 자동 실행되며, 블로그를 크롤링하여 최신 6개 게시글을 테이블 형태로 README에 표시합니다.

## Build and Run Commands

```bash
# Build JAR file
./gradlew fatJar

# Run application (requires ACCESS_TOKEN environment variable)
java -DACCESS_TOKEN=<token> -jar ./build/libs/postingbox-all*.jar

# Run tests
./gradlew test
```

## Architecture and Core Flow

### 1. Main Execution Flow
- `PostingboxApplication.main()` → `PostingService.updatePostingBox()`
- 블로그 크롤링 → README 날짜 체크 → 새 글 있으면 GitHub API 실행

### 2. Key Components

**Data Extraction Pipeline:**
- `HtmlSupporter`: JSoup을 사용한 HTML 파싱
  - Lazy loading 이미지 처리 (`data-src` 속성 우선)
  - 중복 키 처리 (merge function 사용)
- `BoardService.generateBoards()`: 추출된 데이터를 Board 객체로 변환
- `Board`: 게시글 데이터 (제목, 링크, 요약, 이미지, 날짜)

**GitHub Integration:**
- `GitHubClient`: kohsuke/github-api 라이브러리 래핑
- Branch 생성 → img/ 폴더 삭제 → README 업데이트 → 이미지 업로드 → PR 머지
- 실패 시 branch 자동 삭제

**Image Processing:**
- `FileUtil.toBufferedImage()`: URL에서 이미지 다운로드
- `FileUtil.resize()`: 임시 디렉토리에 리사이징된 이미지 저장
- 최대 크기: 400x200px (2:1 비율 유지)

### 3. Critical Configuration

**application.yml 구조:**
```yaml
blog:
  url: 블로그 주 URL
  contents-class-name: 게시글 컨테이너 클래스명
  title-class-name: 제목 클래스명
  summary-class-name: 요약 클래스명
  date-class-name: 날짜 클래스명
github:
  repo-name: owner/repository
```

**날짜 형식:**
- 패턴: `YY.MM.DD` (정규식: `\d{2}\.\d{2}\.\d{2}`)
- 2자리 연도는 2000년대로 자동 변환

### 4. Error Handling Patterns

**이미지 처리:**
- null/빈 URL → null 반환 (에러 방지)
- 이미지 없는 게시글도 정상 처리
- 임시 디렉토리 사용 (JAR 실행 환경 대응)

**HTML 파싱:**
- 중복 링크 → 첫 번째 값 유지
- Lazy loading → `data-src` 우선, `src` 대체

**GitHub API:**
- 모든 작업을 branch에서 수행
- 실패 시 branch 삭제로 깨끗한 롤백

## Common Issues and Solutions

1. **"Duplicate key" 에러**: `Collectors.toMap()`에 merge function 추가 필요
2. **"protocol = https host = null" 에러**: 빈 이미지 URL 처리 필요
3. **이미지 저장 실패**:
   - Lazy loading (`data-src`) 속성 확인
   - 임시 디렉토리 사용 확인
4. **README 업데이트 안 됨**: 최신 게시글 날짜가 이미 README에 있는지 확인

## GitHub Actions Workflow

- 실행 시간: 매일 한국시간 01:00 (UTC 16:00)
- 트리거: schedule, push to main, workflow_dispatch
- 필수 Secret: `ACCESS_TOKEN` (GitHub Personal Access Token)
- Java 11 사용 (Temurin distribution)