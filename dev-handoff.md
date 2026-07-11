# 🔧 FitLog 개발 인계 문서 (Development Handoff)

**작성자: 이태호**
**버전**: 1.0 (Final)
**최종 확정일**: 2025-07-11
**대상 독자**: 프론트엔드/백엔드 개발자

> 본 문서는 FitLog의 기획 산출물을 **개발자가 즉시 구현 시작할 수 있는 형태**로 정리한 개발 인계 문서입니다.
> 화면별 기능, API 스펙, 데이터 스키마, 상태 관리, 예외 처리를 통합했습니다.

---

## 📑 목차

1. [기술 스택 권장 사항](#1-기술-스택-권장-사항)
2. [데이터 모델 (Data Schema)](#2-데이터-모델-data-schema)
3. [API 명세 (REST API Spec)](#3-api-명세-rest-api-spec)
4. [상태 관리 (State Management)](#4-상태-관리-state-management)
5. [화면별 개발 명세](#5-화면별-개발-명세)
6. [예외 처리 통합 정의](#6-예외-처리-통합-정의)
7. [공통 컴포넌트 리스트](#7-공통-컴포넌트-리스트)
8. [개발 순서 권장 (Sprint Plan)](#8-개발-순서-권장-sprint-plan)

---

## 1. 기술 스택 권장 사항

| 영역 | 추천 스택 | 근거 |
|------|-----------|------|
| Mobile | React Native 0.74+ / Expo | 크로스플랫폼, 팀 학습 곡선 완만 |
| 상태 관리 | Zustand (경량) 또는 Redux Toolkit | 로컬 상태 + 서버 캐시 분리 |
| 서버 통신 | React Query (TanStack Query) | 캐시·재시도·오프라인 대응 자동화 |
| Backend | Spring Boot 3.x (Java 21) | 학습된 스택 + 안정성 |
| DB | PostgreSQL 15+ | 관계 무결성 + JSON 컬럼 지원 |
| Auth | JWT + OAuth 2.0 (Kakao/Google) | 소셜 로그인 (P03 명세) |
| 스토리지 | AWS S3 (프로필 이미지) | 저비용, 표준 |
| CI/CD | GitHub Actions | 레포와 통합 |

> 대안: Backend를 FastAPI(Python)로 가면 AI 코칭 모델 통합이 쉬움. 팀 선호에 따라 결정.

---

## 2. 데이터 모델 (Data Schema)

### 핵심 엔티티 관계도

```
User (1) ─── (N) Workout ─── (N) Set
                 │
                 └── (N) Exercise
User (1) ─── (N) Goal
User (1) ─── (N) Notification
```

### Entity 정의

#### 🧑 User
| 필드 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | UUID | PK | 사용자 ID |
| email | VARCHAR(255) | UNIQUE, NOT NULL | 이메일 |
| nickname | VARCHAR(20) | NOT NULL | 닉네임 (P07-TXT-01 참조, 20자 제한) |
| height_cm | DECIMAL(5,2) | | 키 (P05 입력) |
| weight_kg | DECIMAL(5,2) | | 몸무게 (P05 입력) |
| birth_date | DATE | | 나이 계산용 |
| gender | ENUM | | male/female/other |
| created_at | TIMESTAMP | NOT NULL | 가입 시각 |
| provider | ENUM | | email/kakao/google |

#### 💪 Workout (운동 세션)
| 필드 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | UUID | PK | |
| user_id | UUID | FK, NOT NULL | |
| exercise_id | UUID | FK, NOT NULL | 운동 종목 |
| started_at | TIMESTAMP | NOT NULL | 시작 시각 |
| ended_at | TIMESTAMP | | 완료 시각 |
| memo | TEXT | | 최대 500자 (P12 명세) |
| is_synced | BOOLEAN | DEFAULT true | 오프라인 저장 여부 |
| created_at | TIMESTAMP | NOT NULL | |

#### 🏋️ Set (세트)
| 필드 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | UUID | PK | |
| workout_id | UUID | FK, NOT NULL | |
| set_number | INTEGER | NOT NULL | 세트 순번 (1~20) |
| weight_kg | DECIMAL(5,1) | NOT NULL | 중량 (0.5~999.5, 0.5 단위) |
| rep_count | INTEGER | NOT NULL | 횟수 (1~999) |
| is_completed | BOOLEAN | DEFAULT false | 세트 완료 여부 |
| rest_seconds | INTEGER | DEFAULT 60 | 휴식 시간 |

#### 🏃 Exercise (운동 종목)
| 필드 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | UUID | PK | |
| name | VARCHAR(100) | NOT NULL | 운동명 |
| category | ENUM | NOT NULL | strength/cardio/home |
| target_muscles | VARCHAR[] | | ["chest", "triceps"] |
| pose_image_url | VARCHAR(500) | | 자세 이미지 |
| is_custom | BOOLEAN | DEFAULT false | 사용자 추가 여부 |
| created_by | UUID | FK | 커스텀 경우 사용자 ID |

#### 🎯 Goal (운동 목표)
| 필드 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | UUID | PK | |
| user_id | UUID | FK, NOT NULL | |
| goal_type | ENUM | NOT NULL | lose/gain/maintain |
| target_weight_kg | DECIMAL(5,2) | | 목표 체중 |
| created_at | TIMESTAMP | NOT NULL | |
| updated_at | TIMESTAMP | | 최근 수정 (P22) |

#### ⭐ Favorite (즐겨찾기)
| 필드 | 타입 | 제약 | 설명 |
|------|------|------|------|
| user_id | UUID | FK, PK | |
| exercise_id | UUID | FK, PK | |
| created_at | TIMESTAMP | NOT NULL | |

**제약**: 사용자당 최대 50개 (P09-BTN-02 명세)

---

## 3. API 명세 (REST API Spec)

Base URL: `https://api.fitlog.app/v1`
인증: `Authorization: Bearer {jwt_token}`

### 🔐 Auth

#### POST /auth/signup
회원가입

**Request**:
```json
{
  "email": "user@example.com",
  "password": "password123!",
  "nickname": "이태호",
  "provider": "email"
}
```

**Response 201**:
```json
{
  "token": "eyJhbGc...",
  "user": { "id": "uuid", "email": "...", "nickname": "..." }
}
```

**Errors**: 400 (validation), 409 (email exists)

#### POST /auth/login
로그인 (P03)

### 🏠 User

#### GET /users/me
현재 사용자 정보 조회 (P07 인사말 표시용)

#### PATCH /users/me
프로필 수정 (P21)

**Request**:
```json
{ "nickname": "새닉네임" }
```

#### POST /users/me/body-info
신체 정보 입력 (P05)

**Request**:
```json
{ "height_cm": 178, "weight_kg": 85.5, "gender": "male", "birth_date": "1997-03-15" }
```

### 💪 Workouts

#### GET /workouts/today-summary
오늘의 요약 (P07-CARD-01 데이터)

**Response 200**:
```json
{
  "total_duration_min": 45,
  "total_calories": 320,
  "total_volume_kg": 1200
}
```

**Response (빈 상태)**:
```json
{ "total_duration_min": 0, "total_calories": 0, "total_volume_kg": 0, "is_empty": true }
```

#### GET /workouts/recent?limit=5
최근 운동 리스트 (P07-LIST-01)

#### POST /workouts
운동 시작 (P11 진입 시)

**Request**:
```json
{ "exercise_id": "uuid", "routine_id": "uuid?" }
```

**Response 201**: `{ "workout_id": "uuid", "started_at": "..." }`

#### POST /workouts/{id}/sets
세트 기록 (F11-04, F11-06)

**Request**:
```json
{ "set_number": 1, "weight_kg": 60.0, "rep_count": 10, "is_completed": true }
```

**Validation**:
- `weight_kg`: 0.5 ≤ x ≤ 999.5, 0.5 단위
- `rep_count`: 1 ≤ x ≤ 999
- `set_number`: 1 ≤ x ≤ 20

**Response 400 (범위 초과)**:
```json
{ "error": "INVALID_RANGE", "field": "weight_kg", "message": "0.5~999.5kg 사이로 입력해주세요" }
```

#### PATCH /workouts/{id}/complete
운동 완료 (P11 → P12 진입 시)

**Request**:
```json
{ "ended_at": "2025-07-11T09:45:00Z", "memo": "오늘 컨디션 좋음" }
```

**Response 200**: 요약 데이터
```json
{
  "workout_id": "uuid",
  "duration_min": 45,
  "calories": 320,
  "total_volume_kg": 1200,
  "sets": [ /* Set[] */ ]
}
```

### 🔍 Exercises

#### GET /exercises?category=strength&search=벤치
운동 종목 검색 (P09)

**Query Params**:
- `category`: strength/cardio/home
- `search`: 검색어 (2자 이상, F09-02)

**Response (결과 있음)**:
```json
{
  "total": 15,
  "items": [
    { "id": "uuid", "name": "벤치프레스", "category": "strength", "target_muscles": ["chest"], "is_favorite": true }
  ]
}
```

**Response (결과 없음, F09-06)**:
```json
{
  "total": 0,
  "items": [],
  "suggestions": ["벤치프레스", "벤치딥"]
}
```

#### POST /exercises/favorite
즐겨찾기 토글 (F09-04)

**Response 400 (50개 초과)**:
```json
{ "error": "FAVORITE_LIMIT", "message": "즐겨찾기는 최대 50개까지 가능합니다" }
```

### 📊 Statistics

#### GET /stats/dashboard?period=week
통계 대시보드 데이터 (P16)

**Query**: `period=week|month|year`

**Response**:
```json
{
  "summary": { "workout_days": 4, "total_min": 180, "total_volume_kg": 4800 },
  "volume_chart": [
    { "date": "2025-07-05", "volume_kg": 1200 },
    { "date": "2025-07-06", "volume_kg": 1450 }
  ],
  "muscle_distribution": [
    { "muscle": "chest", "ratio": 0.35 },
    { "muscle": "back", "ratio": 0.25 }
  ],
  "personal_records": [
    { "exercise": "벤치프레스", "weight_kg": 75, "date": "2025-07-10" }
  ]
}
```

### 🤖 AI Recommendation

#### GET /ai/recommend-routine
AI 추천 루틴 (P07-CARD-02, P13)

**Response**:
```json
{
  "routine_id": "uuid",
  "name": "상체 집중 루틴",
  "estimated_min": 45,
  "difficulty": 3,
  "exercises": [
    { "exercise_id": "uuid", "name": "벤치프레스", "sets": 3, "target_reps": 10 }
  ]
}
```

**Response (데이터 부족, 3회 미만)**:
```json
{ "error": "INSUFFICIENT_DATA", "message": "운동 기록을 더 쌓으면 AI 추천을 받을 수 있어요", "required_workouts": 3 }
```

---

## 4. 상태 관리 (State Management)

### 클라이언트 상태 분류

| 상태 유형 | 저장 위치 | 예시 |
|----------|----------|------|
| 서버 캐시 | React Query | 오늘 요약, 통계, 운동 리스트 |
| 인증 상태 | Zustand + SecureStore | JWT 토큰, 유저 정보 |
| UI 상태 | 컴포넌트 로컬 | 모달 열림, 탭 선택 |
| 진행 중 운동 | Zustand + AsyncStorage | 현재 운동, 세트 입력 (F11 자동저장) |
| 오프라인 큐 | AsyncStorage | 네트워크 끊김 시 저장할 데이터 |

### 핵심 상태 정의

```typescript
// authStore.ts
interface AuthState {
  token: string | null;
  user: User | null;
  login: (email, password) => Promise<void>;
  logout: () => void;
}

// workoutStore.ts (진행 중 운동)
interface WorkoutState {
  currentWorkout: Workout | null;
  currentSets: Set[];
  restTimerSeconds: number;
  isRestTimerActive: boolean;
  addSet: (set: Set) => void;
  completeSet: (setNumber: number) => void;
  finishWorkout: () => Promise<Summary>;
  autoSaveToLocal: () => void; // 30초마다 (F11 명세)
}

// offlineQueueStore.ts
interface OfflineQueueState {
  pendingWorkouts: Workout[];
  addToQueue: (workout: Workout) => void;
  syncQueue: () => Promise<void>; // 재연결 시 자동 실행
}
```

---

## 5. 화면별 개발 명세

각 화면별 상세 명세는 [functional-spec.md](./functional-spec.md)를 참조. 여기서는 **개발 관점 추가 정보**만 기록.

### P07 홈 (대시보드)

| 항목 | 상세 |
|------|------|
| Route | `/home` |
| 컴포넌트 | `<HomeScreen>` |
| 데이터 조회 | React Query: `useToday()`, `useRecentWorkouts()`, `useAIRecommendation()` |
| 새로고침 | Pull-to-refresh (F07-06) → `queryClient.invalidateQueries(['home'])` |
| 조건부 렌더링 | `if (isEmpty) → <EmptyState />` (v2 반영) |

### P09 운동 종류 선택

| 항목 | 상세 |
|------|------|
| Route | `/exercises` |
| 검색 로직 | 2자 이상 입력 시 `useDebouncedValue(300ms)` 후 `useExerciseSearch()` |
| 무한 스크롤 | React Query `useInfiniteQuery`, 페이지당 20개 |
| 빈 상태 처리 | `if (results.length === 0) → <NoSearchResult suggestions={...} />` |

### P11 운동 상세 / 기록

| 항목 | 상세 |
|------|------|
| Route | `/workout/{id}` |
| 로컬 자동 저장 | `setInterval(() => saveToLocal(), 30000)` |
| 휴식 타이머 | `<RestTimerModal isFullScreen />` — 세트 체크 시 자동 트리거 |
| 오프라인 대응 | `if (!isOnline) → workoutStore.autoSaveToLocal() + Toast` |
| 뒤로가기 확인 | `Alert.alert("운동을 중단하시겠어요?")` |

### P12 운동 완료 요약

| 항목 | 상세 |
|------|------|
| Route | `/workout/{id}/summary` |
| 진입 애니메이션 | Lottie 2.5s + `Haptics.impactAsync()` (F12-01) |
| 메모 저장 | 1초 debounce 후 PATCH 요청 |
| 공유 | `Share.share()` (RN API), 이미지 생성 실패 시 텍스트 폴백 |

### P16 통계 대시보드

| 항목 | 상세 |
|------|------|
| Route | `/stats` |
| 차트 라이브러리 | `victory-native` 또는 `react-native-svg-charts` |
| 기간 변경 | `useState<Period>` → 쿼리 키 변경 → 자동 재조회 |
| 데이터 없음 | `if (chart.data.length === 0) → <EmptyChart />` |

---

## 6. 예외 처리 통합 정의

Mission 03 IA에 정의된 4개 예외 화면 + 상세 처리 정책.

### E01. 404 (Not Found)

| 항목 | 내용 |
|------|------|
| 트리거 | 딥링크 / 존재하지 않는 리소스 접근 |
| UI | 일러스트 + "요청하신 페이지를 찾을 수 없습니다" |
| CTA | "홈으로 돌아가기" (P07 이동) |
| 데이터 로깅 | Sentry로 URL + user_id 전송 |

### E02. 네트워크 오류 (Network Error)

| 항목 | 내용 |
|------|------|
| 트리거 | fetch failure / 타임아웃 5초 초과 |
| UI | 상단 배너: "⚠️ 오프라인 모드 (자동 저장 중)" |
| 특수 처리 (P11) | 세트 입력값 로컬 저장, 재연결 시 자동 동기화 |
| 재시도 | React Query 자동 재시도 3회 (exponential backoff) |

### E03. 빈 상태 (Empty State)

컨텍스트별로 다르게 처리:

| 화면 | 조건 | 표시 |
|------|------|------|
| P07 | 운동 기록 0건 | "첫 운동을 시작해보세요!" + 큰 CTA (v2 확정) |
| P09 | 검색 결과 0건 | "'{검색어}'에 대한 결과가 없습니다" + 유사 검색어 |
| P16 | 통계 데이터 0건 | "데이터가 쌓이면 통계가 표시됩니다" |
| P18 | 캘린더 이력 0건 | "이 달엔 운동 기록이 없어요" |

### E04. 로딩 상태 (Loading)

| 화면 | 로딩 UI |
|------|--------|
| P07 카드 | 스켈레톤 (회색 막대) |
| P16 차트 | 스켈레톤 차트 |
| P09 리스트 | 상단 스피너 + 스켈레톤 아이템 3개 |
| 전체 화면 로딩 | 최소 표시 시간 200ms (플리커 방지) |

### 공통 예외 처리 정책

- **API 오류 응답 표준**: `{ error: "ERROR_CODE", message: "사용자용 메시지", field?: "..." }`
- **Toast 표시 시간**: 성공 2초, 오류 4초
- **재시도 정책**: 5xx → 자동 재시도 / 4xx → 사용자에게 노출
- **오류 로깅**: 5xx만 Sentry로 전송 (개인정보 스크러빙)

---

## 7. 공통 컴포넌트 리스트

우선 개발해야 할 공통 컴포넌트 (Design System 기반):

| 컴포넌트 | 사용 화면 | 우선순위 |
|---------|----------|---------|
| `<PrimaryButton>` | P07, P11, P12 등 | P0 |
| `<Input>` (숫자 키패드) | P11 중량/횟수 입력 | P0 |
| `<Card>` | P07, P12, P16 | P0 |
| `<EmptyState>` | P07, P09, P16 | P0 |
| `<Toast>` | 전역 | P0 |
| `<TabBar>` | 모든 메인 화면 | P0 |
| `<RestTimerModal>` | P11 | P1 |
| `<LineChart>`, `<DoughnutChart>` | P16 | P1 |
| `<SkeletonLoader>` | 전역 | P1 |

---

## 8. 개발 순서 권장 (Sprint Plan)

MVP 범위는 [`mvp-scope.md`](./mvp-scope.md) 참조. 여기서는 개발 순서 권장.

### Sprint 1 (Week 1-2): 인증 + 홈 뼈대
- P01 스플래시, P03 로그인, P04 회원가입, P05-P06 초기 설정
- Backend: Auth API, User API
- 공통 컴포넌트: Button, Input, Card

### Sprint 2 (Week 3-4): 운동 기록 핵심
- P09 운동 종류, P11 운동 상세 (세트 입력), P12 완료 요약
- Backend: Workouts, Sets, Exercises API
- 오프라인 대응 로직

### Sprint 3 (Week 5-6): 통계 + AI
- P16 통계 대시보드, P07 홈 (요약 카드)
- Backend: Statistics API, AI Recommendation API (초기 규칙 기반)
- 차트 컴포넌트

### Sprint 4 (Week 7): 예외 처리 + QA
- E01-E04 예외 화면
- 오프라인 큐 동기화 검증
- QA + 버그 픽스

---

## 📎 관련 산출물

| 산출물 | 파일 |
|--------|------|
| 통합 기획서 | [`proposal.md`](./proposal.md) |
| 최종 확정 와이어프레임 | [`wireframe-v2.png`](./wireframe-v2.png) |
| 기능 명세서 (베이스) | [`functional-spec.md`](./functional-spec.md) |
| MVP 범위 정의 | [`mvp-scope.md`](./mvp-scope.md) |
| UX 테스트 결과 | [`test-results.md`](./test-results.md) |
