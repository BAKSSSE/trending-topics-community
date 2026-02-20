# CLAUDE.md — Laravel Community (MySQL FULLTEXT)

## 0) 프로젝트 목표
- Laravel 기반 커뮤니티 웹 개발
- 게시글 / 댓글 / 좋아요 / 신고 / 알림 / 검색 기능 포함
- 추후에 자체 실시간 검색어 랭킹 기능 추가 예정
- 검색은 MySQL FULLTEXT 기반으로 구현
- 우선순위: 보안 > 성능 > 유지보수 > 확장성

---

## 1) 기술 스택
- PHP 8.3+
- Laravel 12.x
- MySQL 8.0 (InnoDB)
- Cache: Redis (선택)
- Queue: redis 또는 database
- Search: MySQL FULLTEXT

---

## 2) 아키텍처 규칙

### Controller
- Request 검증
- Policy 권한 체크
- Service 호출
- Resource 또는 View 반환
- 비즈니스 로직 금지

### Service
- 핵심 도메인 로직 처리
- DB::transaction 사용
- 카운트 증감 / 상태 변경 등 처리

### Policy
- 작성자/관리자 권한 구분
- 모든 수정/삭제는 Policy 필수

### Request
- FormRequest 사용
- Validation 로직 재사용 가능하게 설계

---

## 3) 도메인 모델

- User
- Board (카테고리)
- Post
- Comment
- Reaction (좋아요)
- Report
- Notification

---

## 4) 게시글 정책

- soft delete 사용
- 목록은 cursor pagination 기본
- 조회수는 중복 방지 처리
- 좋아요는 UNIQUE(user_id, target_type, target_id)
- 삭제 시 댓글은 마스킹 처리

---

## 5) FULLTEXT 검색 정책

### 인덱스
- posts(title, body)에 FULLTEXT 인덱스 적용

### 검색 방식
- 기본은 BOOLEAN MODE 사용
- 단어별 +prefix* 방식 사용
- MATCH(title, body) AGAINST(? IN BOOLEAN MODE)

### 정렬
- 기본: 검색 점수(score) DESC
- 보조: id DESC

### 필터
- board_id 필터 가능
- 삭제된 글 제외
- 차단 사용자 글 제외

---

## 6) DB 인덱스 규칙

- FK 컬럼은 반드시 인덱스
- posts 복합 인덱스:
(board_id, created_at, id)
- comments 복합 인덱스:
(post_id, parent_id, created_at, id)
- reactions UNIQUE:
(user_id, target_type, target_id)

---

## 7) Eloquent 성능 규칙

- N+1 금지 (with 사용)
- select 필요한 컬럼만 조회
- withCount 사용 권장
- 대량 연산은 chunk 사용

---

## 8) 보안 정책

- XSS escape 기본
- CSRF 적용
- SQL 바인딩 필수
- Rate limit 적용
- 파일 업로드 MIME/용량 제한
- 관리자 페이지 미들웨어 보호

---

## 9) 성능 전략

- 검색은 FULLTEXT 사용 (LIKE 금지)
- 조회수/좋아요 카운트 경합 방지
- 쿼리 변경 시 EXPLAIN 확인

---

## 10) 테스트

- Feature test: CRUD, 권한, 좋아요 중복 방지
- soft delete 검증
- 검색 정확도 테스트

---

## 11) Claude 작업 지시 규칙

작업 전 항상:

1) 목표 정의
2) 변경 파일 목록
3) 마이그레이션 여부
4) 인덱스 영향
5) 캐시/큐 필요 여부
6) 테스트 계획

구현은 작은 PR 단위로 나눈다.
운영 성능에 영향이 있는 변경은 쿼리 + 인덱스 전략을 함께 제시한다.