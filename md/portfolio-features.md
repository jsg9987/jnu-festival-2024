# JNU-Festival 프로젝트 포트폴리오 분석

> 작성일: 2026-04-23
> 목적: 오래된 학부 프로젝트(전남대 축제 플랫폼)를 BE 개발자 포트폴리오로 활용하기 위한 분석 및 활용 계획

---

## 0. 한눈에 요약

이 프로젝트는 **Spring Boot 3.3.4 + Java 17 + PostgreSQL + AWS S3** 기반의 모노레포(BE/FE 분리)로,
도메인 주도 패키지 구조에 9개 도메인 모듈을 갖추고 있음.
깔끔한 레이어드 아키텍처와 JWT 인증, S3 업로드, 테스트 코드가 갖춰져 있어 포트폴리오용으로 적합함.

- 총 **132개 Java 파일**, **21개 엔티티**, **15개 테스트 클래스**
- 60+ 커밋, develop/main 브랜치 운영
- Docker 빌드 가능, PR 템플릿 존재

---

## 1. 프로젝트 구성

### 1.1 모노레포 구조
```
jnu-festival/
├── jnu-festival-2024/    # Backend (Spring Boot 3.3.4 / Java 17 / Gradle)
└── JeonOn-FE/            # Frontend (React 18 + TS + Vite + Tailwind)
```

### 1.2 백엔드 패키지 구조 (DDD-like)
```
com.jnu.festival/
├── domain/                # 9개 도메인 모듈 (각각 controller/service/repository/entity/dto)
│   ├── user/             # 회원, JWT 로그인
│   ├── booth/            # 부스 (검색/랭킹/필터)
│   ├── zone/             # 축제 존(구역)
│   ├── partner/          # 협력사
│   ├── content/          # 콘텐츠 게시물
│   ├── comment/          # 부스 댓글
│   ├── like/             # 부스 좋아요 (소프트 삭제)
│   ├── bookmark/         # 부스/파트너/콘텐츠 북마크
│   ├── timecapsule/      # 타임캡슐 (스케줄러 메일 발송)
│   ├── feedback/         # 피드백 (이미지 첨부)
│   └── admin/            # 관리자 전용 CRUD
└── global/                # 공통 인프라
    ├── config/           # SecurityConfig, CorsConfig, S3Service, PasswordConfig
    ├── security/jwt/     # JWTUtil, JWTAuthenticationFilter, DefaultAuthenticationFilter
    ├── error/            # GlobalExceptionHandler (11종 예외)
    └── common/, util/
```

### 1.3 기술 스택 상세

| 카테고리 | 기술 |
|---|---|
| 언어 | Java 17 |
| 프레임워크 | Spring Boot 3.3.4 |
| 빌드 | Gradle |
| ORM | Spring Data JPA |
| DB | PostgreSQL (운영), H2 (테스트) |
| 보안 | Spring Security + JWT (jjwt 0.12.6) |
| 파일 | AWS S3 (aws-sdk-s3) |
| 매핑 | MapStruct |
| 검증 | Jakarta Validation |
| 메일 | Spring Mail |
| 테스트 | JUnit 5, Mockito, AssertJ |
| 인프라 | Docker (OpenJDK 17 기반) |

---

## 2. 코드 읽는 순서 (권장 학습 경로)

> 오래된 프로젝트를 다시 익힐 때는 **인프라 → 인증 → 도메인 단순 → 도메인 복잡 → 테스트** 순서가 효율적

### Step 1. 진입점 + 빌드 설정 (10분)
- `jnu-festival-2024/build.gradle` — 의존성 전체
- `jnu-festival-2024/Dockerfile` — 배포 컨테이너 구성
- `src/main/resources/application*.yml` — 환경 설정
- `FestivalApplication.java` — 메인 클래스

### Step 2. 글로벌 설정/공통 인프라 (30분)
- `global/config/SecurityConfig.java` — URL별 권한 정책, 필터 체인
- `global/config/CorsConfig.java` — CORS 화이트리스트
- `global/config/PasswordConfig.java` — BCrypt 위임 인코더
- `global/config/S3Service.java` — 파일 업로드
- `global/error/GlobalExceptionHandler.java` — 예외 → HTTP 응답 매핑
- `global/error/ErrorCode` enum

### Step 3. 인증/인가 흐름 (45분)
> 요청 한 건이 어떻게 인증되고 사용자 컨텍스트가 주입되는지 따라가본다

1. `domain/user/controller/AuthController` — 로그인 진입점
2. `global/security/jwt/DefaultAuthenticationFilter.java:34-134` — 회원가입+인증 필터
3. `global/security/jwt/JWTUtil.java:16-51` — 토큰 생성/검증
4. `global/security/jwt/JWTAuthenticationFilter` — 후속 요청 토큰 검증
5. `UserDetailsServiceImpl` — 사용자 로드
6. `domain/user/entity/User.java` — Role enum, accessToken 필드

### Step 4. 단순 조회 도메인 — 워밍업 (30분)
- **Zone** 도메인 전체 (가장 단순한 CRUD)
- **Partner** 도메인 (이미지/북마크 추가)
- **Content** 도메인 (Partner와 거의 동일)

### Step 5. 핵심 도메인 — Booth (1시간)
- `domain/booth/entity/Booth.java` — location/category/period enum
- `domain/booth/repository/BoothRepository.java:14` — `@Query` 동적 필터
- `domain/booth/service/BoothService.java:161` — 페이지네이션 Top 5 랭킹
- `domain/booth/controller/BoothController.java` — 11개 엔드포인트

### Step 6. 사용자 참여 도메인 (45분)
- `domain/comment/` — 작성/삭제 권한 체크
- `domain/like/` — `@SQLDelete`로 소프트 삭제
- `domain/bookmark/` — 3종 북마크 다형성 비교

### Step 7. 특수 기능 (30분)
- `domain/timecapsule/service/TimecapsuleService.java:23` — `@Scheduled` 메일 발송
- `domain/feedback/` — 이미지 다중 업로드
- `domain/admin/` — 관리자 CRUD 묶음

### Step 8. 테스트 코드 (30분)
- `BoothControllerTest`, `BoothServiceTest`, `BoothRepositoryTest`
- `BoothImageRepositoryTest`, `PartnerBookmarkControllerTest`
- H2 인메모리 DB 활용 패턴

**총 예상 소요: 4시간 30분 ~ 5시간**

---

## 3. 포트폴리오 활용 계획

### 3.1 한 줄 소개 (이력서/포트폴리오 상단용)

> "전남대학교 축제 정보 플랫폼의 백엔드 개발 — Spring Boot 3.3 + JPA + PostgreSQL + AWS S3 기반 9개 도메인 REST API 설계 및 구현, JWT 인증, 스케줄러 기반 타임캡슐 메일 발송, 도메인별 단위/통합 테스트 작성"

### 3.2 어필 포인트 (강한 순서)

#### ① 도메인 주도 패키지 구조와 레이어 분리 (★★★)
- 9개 도메인을 독립 패키지로 분리, 각각 controller/service/repository/entity/dto 5계층
- **면접 멘트**: "도메인별로 폴더를 분리해 응집도를 높였고, 새 기능 추가 시 다른 도메인을 건드리지 않도록 설계했습니다."

#### ② JWT + Spring Security 무상태 인증 (★★★)
- 커스텀 필터 2개 (DefaultAuthenticationFilter / JWTAuthenticationFilter)
- BCrypt 위임 인코더, Role 기반 URL 권한 분리 (`/admins/**` → ADMIN)
- **면접 멘트**: "STATELESS 세션 정책으로 확장 가능한 인증 구조를 만들었고, 로그인/검증 필터를 분리해 책임을 나눴습니다."

#### ③ S3 파일 업로드 + 보안 검증 (★★★)
- UUID 파일명으로 충돌 방지, 확장자 화이트리스트(jpg/png/jpeg)
- 멀티파트 + `@RequestPart` 처리, 5개 도메인(부스/콘텐츠/파트너/피드백/타임캡슐)에서 재사용
- **면접 멘트**: "확장자 화이트리스트로 임의 파일 업로드 취약점을 막고, UUID로 동시 업로드 충돌을 방지했습니다."

#### ④ 글로벌 예외 처리 일원화 (★★)
- `@RestControllerAdvice`로 11종 예외 → 일관된 에러 응답 포맷
- ErrorCode enum으로 메시지/상태 코드 중앙 관리
- **면접 멘트**: "ErrorCode를 중앙화해 클라이언트가 동일한 응답 스키마로 에러를 처리할 수 있게 했습니다."

#### ⑤ 소프트 삭제(@SQLDelete) (★★)
- Like/Bookmark 엔티티에 적용 — 데이터 복구 가능, 통계 추적 가능
- **면접 멘트**: "좋아요 취소 시 물리 삭제 대신 소프트 삭제로 사용자 행동 데이터를 보존했습니다."

#### ⑥ JPA 동적 쿼리 + Lazy Loading (★★)
- `@Query`로 location/category/period 선택적 필터링 (BoothRepository)
- 모든 연관관계 `FetchType.LAZY`로 N+1 방지 의도
- **면접 멘트**: "Lazy 로딩 + 필요한 곳만 fetch join으로 N+1을 관리했습니다."

#### ⑦ 스케줄러 기반 타임캡슐 메일 발송 (★★)
- `@Scheduled`로 정해진 시점에 메일 자동 발송
- **면접 멘트**: "Spring 스케줄러 + Spring Mail로 사용자가 작성한 미래 메시지를 자동 발송하는 비동기 작업을 구현했습니다."

#### ⑧ 단위/통합 테스트 (★★)
- 15개 테스트 클래스, JUnit 5 + Mockito + AssertJ + H2
- Controller / Service / Repository 3계층 테스트
- **면접 멘트**: "Repository는 H2로 통합, Service는 Mockito로 단위, Controller는 MockMvc로 검증해 계층별 테스트 전략을 분리했습니다."

### 3.3 포트폴리오 문서 구성안

```
1. 프로젝트 개요 (1줄 + 기간 + 팀 규모 + 본인 역할)
2. 기술 스택 배지 (Spring Boot 3.3 / Java 17 / PostgreSQL / JPA / S3 / JWT / Docker)
3. 시스템 아키텍처 다이어그램   ← 직접 그려야 함 (없음)
4. ERD                          ← 직접 그려야 함 (없음)
5. API 명세 표 (아래 30개 엔드포인트 정리)
6. 본인이 담당한 도메인/기능 (예: 부스 도메인 전체, JWT 인증, S3 통합)
7. 기술적 의사결정 3~5개 (소프트 삭제 도입 이유, JWT vs Session 선택 이유 등)
8. 트러블슈팅 1~2건 (Git log/PR 코멘트에서 발굴 권장)
9. 회고 (배운 점, 개선하고 싶은 점 — 아래 3.4와 연결)
```

### 3.4 "지금이라면 이렇게 개선할 부분" — 면접 단골 질문 대비

탐색 결과 다음이 **없음** — 솔직하게 회고로 풀면 오히려 성숙해 보임:

- ❌ Redis 캐싱 (부스 랭킹/조회 캐싱 가능)
- ❌ 동시성 제어 (좋아요 동시 클릭, 분산락)
- ❌ Refresh Token (Access Token만 존재)
- ❌ Swagger / Spring REST Docs (API 문서화)
- ❌ CI/CD 워크플로우 (.github/workflows 비어있음)
- ❌ 모니터링 (Actuator, Prometheus 등)
- ❌ ERD 자동 생성/문서

**활용 팁**: "회고" 섹션에 "지금이라면 X를 추가할 것이다" 형식으로 2~3개만 적으면 학습 의지를 보여줄 수 있음.

### 3.5 선택적 후속 작업 (포트폴리오 강화용)

본 프로젝트 코드를 **건드리지 않고** 별도 문서로 만들 수 있는 산출물:

1. **README.md 작성** (현재 백엔드에 없음) — 프로젝트 소개, 실행 방법, API 표
2. **ERD 그리기** — dbdiagram.io 또는 ERD Online으로 엔티티 21개 시각화
3. **시스템 아키텍처 다이어그램** — Excalidraw, Draw.io
4. **API 명세 정리** — Notion/GitHub Wiki에 30개 엔드포인트 표

---

## 4. 주요 REST API 엔드포인트 (총 30개+)

### 인증
- `POST /api/v1/login` — 회원가입 + 로그인 (JWT 발급)
- `POST /api/v1/logout` — 로그아웃

### 유저
- `GET /api/v1/users` — 사용자 정보 조회
- `GET /api/v1/users/bookmarks/partners` — 파트너 북마크 목록
- `GET /api/v1/users/bookmarks/contents` — 콘텐츠 북마크 목록
- `GET /api/v1/users/bookmarks/booths` — 부스 북마크 목록

### 부스 (공개)
- `GET /api/v1/booths` — 부스 목록 (location, period, category 필터)
- `GET /api/v1/booths/{boothId}` — 부스 상세
- `GET /api/v1/booths/search` — 부스 검색
- `GET /api/v1/booths/ranks` — 부스 랭킹 (좋아요 순)
- `GET /api/v1/booths/{boothId}/comments` — 댓글 목록

### 부스 (인증 필요)
- `POST /api/v1/booths/{boothId}/comments` — 댓글 작성
- `DELETE /api/v1/booths/{boothId}/comments/{commentId}` — 댓글 삭제
- `POST /api/v1/booths/{boothId}/likes` — 좋아요
- `DELETE /api/v1/booths/{boothId}/likes` — 좋아요 취소
- `POST /api/v1/booths/{boothId}/bookmarks` — 부스 북마크

### 파트너/콘텐츠
- `GET /api/v1/partners` — 파트너 목록
- `GET /api/v1/partners/{partnerId}` — 파트너 상세
- `GET /api/v1/contents` — 콘텐츠 목록
- `GET /api/v1/contents/{contentId}` — 콘텐츠 상세
- `POST /api/v1/partners/{partnerId}/bookmarks` — 파트너 북마크
- `POST /api/v1/contents/{contentId}/bookmarks` — 콘텐츠 북마크

### 존/피드백/타임캡슐
- `GET /api/v1/zones` — 존 목록 (location 필터)
- `POST /api/v1/feedbacks` — 피드백 작성 (이미지 업로드)
- `POST /api/v1/timecapsules` — 타임캡슐 작성 (이미지 업로드)
- `GET /api/v1/timecapsules` — 타임캡슐 목록
- `DELETE /api/v1/timecapsules/{timecapsuleId}` — 타임캡슐 삭제

### 관리자 (ADMIN 권한)
- `POST /api/v1/admins/zones` / `DELETE /api/v1/admins/zones/{zoneId}`
- `POST /api/v1/admins/partners` / `DELETE /api/v1/admins/partners/{partnerId}`
- `POST /api/v1/admins/contents` / `DELETE /api/v1/admins/contents/{contentId}`
- `POST /api/v1/admins/booths` / `DELETE /api/v1/admins/booths/{boothId}`
- `GET /api/v1/admins/feedbacks` — 피드백 목록 (category 필터)

---

## 5. 핵심 파일 레퍼런스 (즉시 참조용)

| 항목 | 경로 |
|---|---|
| 빌드 설정 | `jnu-festival-2024/build.gradle` |
| 도커 | `jnu-festival-2024/Dockerfile` |
| 보안 설정 | `src/main/java/com/jnu/festival/global/config/SecurityConfig.java:31-87` |
| JWT 유틸 | `global/security/jwt/JWTUtil.java:16-51` |
| 인증 필터 | `global/security/jwt/DefaultAuthenticationFilter.java:34-134` |
| S3 서비스 | `global/config/S3Service.java` |
| 글로벌 예외 | `global/error/GlobalExceptionHandler.java` |
| 부스 서비스 | `domain/booth/service/BoothService.java:161` |
| 부스 레포지토리 | `domain/booth/repository/BoothRepository.java:14` |
| 소프트 삭제 | `domain/bookmark/entity/BoothBookmark.java:15` |
| 스케줄러 | `domain/timecapsule/service/TimecapsuleService.java:23` |
| 테스트 디렉토리 | `src/test/java/com/jnu/festival/` |

---

## 6. 다음 단계 체크리스트

- [ ] Step 1~3 (인프라 + 인증) 읽기 — 1시간 30분
- [ ] Git log로 본인이 담당한 도메인/커밋 확인 (`git log --author="<본인 이름>"`)
- [ ] Step 4~7 (도메인 코드) 읽기 — 약 3시간
- [ ] Step 8 (테스트) 확인
- [ ] 포트폴리오 어필 포인트 ①~⑧ 중 본인이 직접 구현한 것 표시
- [ ] README.md 작성 (현재 없음, ROI 가장 높음)
- [ ] ERD 작성 (dbdiagram.io 추천)
- [ ] 시스템 아키텍처 다이어그램 작성
- [ ] 트러블슈팅 사례 1~2건 발굴 (Git PR/Issue/커밋 메시지 검토)
