# YouTube 레퍼런스 수집기 - PRD

## 프로젝트 개요

ViewTrap 스타일의 YouTube 영상 분석 도구를 **단일 HTML 파일**로 제작한다.
검색어를 입력하면 영상 리스트가 나오고, 각 영상의 **기여도**와 **성과도** 지표가 계산되어 카드 형태로 표시된다. 좋은 레퍼런스는 북마크로 저장한다.

## 결과물

- 파일명: `index.html`
- 단일 HTML 파일 (HTML + CSS + JS 모두 인라인)
- 더블클릭하면 브라우저에서 바로 동작
- 외부 의존성: Pretendard, JetBrains Mono 웹폰트만 CDN 허용
- 그 외 라이브러리 사용 금지 (Vanilla JS only)

---

## 핵심 기능

### 1. API 키 관리
- 최초 실행 시 YouTube Data API v3 키 입력 모달 표시
- localStorage에 저장 (key: `youtube_api_key`)
- 우측 상단에 톱니바퀴 아이콘 → API 키 변경/제거 가능

### 2. 검색
입력 폼:
- 검색어 (필수)
- 정렬: `관련성` / `최신순` / `조회수순`
- 결과 개수: `10` / `20` / `50` (기본 20)
- 기간 필터: `전체` / `최근 1주` / `1개월` / `6개월` / `1년` (기본 전체)

### 3. 결과 카드
4열 그리드 (데스크탑) / 2열 (태블릿) / 1열 (모바일)

각 카드에 표시:
- 썸네일 (16:9, 영상 길이는 우측 하단에 오버레이)
- 영상 제목 (2줄 말줄임, 클릭 시 YouTube 새 탭으로 이동)
- 조회수 · 게시일
- 채널 아이콘 + 채널명
- 구독자 수 · 총 영상 수
- 기여도 등급 + 배수
- 성과도 등급 + 배수
- 북마크 버튼 (별 아이콘)

### 4. 북마크
- 별 아이콘 클릭 시 localStorage에 저장 (key: `youtube_bookmarks`)
- 상단에 `검색` / `북마크` 탭 전환
- 북마크 탭은 저장된 카드들을 동일 레이아웃으로 표시
- 별을 다시 클릭하면 북마크 해제
- 북마크 데이터 구조: `{videoId, title, thumbnail, viewCount, publishedAt, duration, channelTitle, channelId, channelThumbnail, subscriberCount, channelVideoCount, contributionRatio, performanceRatio, savedAt}`

---

## 메트릭 계산식

### 성과도 (Performance)
```
성과도 = 영상 조회수 / 채널 구독자 수
```
의미: 구독자 범위 밖으로 얼마나 퍼졌는지 (알고리즘 추천 강도)

### 기여도 (Contribution)
```
채널 평균 조회수 = 채널 총 조회수 / 채널 총 영상 수
기여도 = 영상 조회수 / 채널 평균 조회수
```
의미: 이 영상이 채널 내에서 얼마나 튀는지 (대표작 정도)

### 등급 기준

**성과도**
| 배수 | 등급 |
|------|------|
| ≥ 10 | Excellent |
| 3 ~ 10 | Great |
| 1 ~ 3 | Good |
| 0.5 ~ 1 | Soso |
| < 0.5 | Bad |

**기여도**
| 배수 | 등급 |
|------|------|
| ≥ 5 | Excellent |
| 2 ~ 5 | Great |
| 1 ~ 2 | Good |
| 0.5 ~ 1 | Soso |
| < 0.5 | Bad |

**예외 처리**: 구독자 0명 또는 채널 영상 0개일 때는 "N/A" 표시.

---

## YouTube Data API 명세

### 1단계: search.list (영상 ID 수집)
```
GET https://www.googleapis.com/youtube/v3/search

파라미터:
  part: snippet
  q: {검색어}
  type: video
  maxResults: {10|20|50}
  order: {relevance|date|viewCount}
  publishedAfter: {ISO datetime, 기간 필터 시}
  key: {API 키}
```
- 수집: `items[].id.videoId`, `items[].snippet.channelId`
- 비용: **100 units**

### 2단계: videos.list (영상 통계)
```
GET https://www.googleapis.com/youtube/v3/videos

파라미터:
  part: statistics,snippet,contentDetails
  id: {video IDs 콤마 구분, 최대 50개}
  key: {API 키}
```
필요 필드:
- `statistics.viewCount`
- `snippet.title`, `snippet.publishedAt`, `snippet.channelTitle`, `snippet.channelId`
- `snippet.thumbnails.medium.url`
- `contentDetails.duration` (ISO 8601: PT3M2S 형식)

비용: **1 unit**

### 3단계: channels.list (채널 통계)
```
GET https://www.googleapis.com/youtube/v3/channels

파라미터:
  part: statistics,snippet
  id: {channel IDs 콤마 구분, 중복 제거, 최대 50개}
  key: {API 키}
```
필요 필드:
- `statistics.subscriberCount`
- `statistics.viewCount` (채널 총 조회수)
- `statistics.videoCount` (총 영상 수)
- `snippet.thumbnails.default.url`

비용: **1 unit**

### 데이터 머지
영상에 채널 정보를 `channelId`로 매칭 → 메트릭 계산 → 카드 렌더링

---

## UI 디자인

### 톤
ViewTrap이 일반 SaaS 앱이라면, 이건 **데이터 분석 노트북**에 가까운 차분한 톤. 시그니처는 **모든 숫자를 모노스페이스 폰트로 표시**해서 데이터 비교가 한눈에 가능하도록.

### 색상 토큰
```css
--bg: #FAFAF8;
--card-bg: #FFFFFF;
--text-primary: #1A1A1A;
--text-secondary: #6B6B6B;
--text-tertiary: #9CA3AF;
--border: #E5E5E0;
--accent: #3730A3;        /* 딥 인디고 - 버튼, 강조 */
--accent-hover: #312E81;
```

### 등급 색상
```css
--grade-excellent: #166534;  /* 딥 그린 */
--grade-great: #16A34A;      /* 그린 */
--grade-good: #65A30D;       /* 라임 그린 */
--grade-soso: #CA8A04;       /* 앰버 */
--grade-bad: #6B7280;        /* 회색 */
```
배지는 배경색 + 흰 텍스트, 또는 라이트 배경 + 진한 텍스트 (선택).

### 타이포그래피
- 본문/UI: **Pretendard** (Pretendard CDN: `https://cdn.jsdelivr.net/gh/orioncactus/pretendard/dist/web/static/pretendard.css`)
- 숫자/메트릭: **JetBrains Mono** (Google Fonts)
- 모노 적용 대상: 조회수, 구독자 수, 영상 수, 게시일, 영상 길이, 메트릭 배수 (예: `2.3x`)

### 레이아웃 (대략)
```
┌─────────────────────────────────────────────────────────┐
│ YouTube 레퍼런스 수집기  [검색][북마크]    [⚙ API 키]    │
├─────────────────────────────────────────────────────────┤
│ [검색어..............] [정렬▼] [개수▼] [기간▼] [검색]   │
├─────────────────────────────────────────────────────────┤
│ 검색 결과 20개 · 할당량 102/10,000                       │
├─────────────────────────────────────────────────────────┤
│ ┌────┐ ┌────┐ ┌────┐ ┌────┐                            │
│ │카드│ │카드│ │카드│ │카드│                            │
│ └────┘ └────┘ └────┘ └────┘                            │
└─────────────────────────────────────────────────────────┘
```

### 카드 구조
```
┌─────────────────────────┐
│   [썸네일      03:02]   │  ← 16:9
├─────────────────────────┤
│ 영상 제목 두 줄까지     │
│ 노출. 말줄임 처리       │
│                         │
│ 72.9만 회 · 2026.03.22  │  ← 모노
├─────────────────────────┤
│ [🎨] 채널명             │
│      1,790명 · 420개    │  ← 모노
├─────────────────────────┤
│  기여도      성과도     │
│  [Good ]   [Great ]     │
│  2.3x       5.1x        │  ← 모노
├─────────────────────────┤
│              [☆]        │
└─────────────────────────┘
```

---

## 숫자 포맷

### 한국식 단위 변환
- 1억 이상: `1.2억`
- 1만 이상: `72.9만`
- 그 미만: `3,500` (콤마)

### 게시일
- ISO → `YYYY.MM.DD` (예: `2026.03.22`)

### 영상 길이 (ISO 8601 duration → 시:분:초)
- `PT3M2S` → `03:02`
- `PT1H23M45S` → `1:23:45`
- `PT45S` → `00:45`

---

## 에러 처리

| 상황 | 메시지 |
|------|--------|
| API 키 미입력 | 모달 표시: "YouTube Data API v3 키를 입력해주세요" + 발급 가이드 링크 |
| 401/403 invalid key | "API 키가 유효하지 않습니다. 다시 확인해주세요." → 키 모달 자동 표시 |
| 403 quotaExceeded | "오늘의 API 할당량을 초과했습니다. 내일 다시 시도하세요." |
| 검색 결과 없음 | 빈 상태 일러스트 + "검색 결과가 없습니다. 다른 키워드로 시도해보세요." |
| 네트워크 오류 | "연결에 실패했습니다. 인터넷 연결을 확인해주세요." |

---

## 할당량 추적
- localStorage에 `youtube_quota_used` (오늘 사용량)와 `youtube_quota_date` (날짜) 저장
- 날짜가 바뀌면 자동 리셋
- 검색 1회당 +102 (search 100 + videos 1 + channels 1)
- 상단 결과 카운트 옆에 `할당량 102/10,000` 형식으로 표시
- 10,000 초과 시 검색 버튼 비활성화 + 경고

---

## 코드 구조 (단일 파일 내 함수 분리)

```js
// API 키 관리
function getApiKey()
function setApiKey(key)
function showApiKeyModal()

// API 호출
async function searchVideos(keyword, options)   // search.list
async function fetchVideoDetails(videoIds)       // videos.list
async function fetchChannelDetails(channelIds)   // channels.list

// 데이터 가공
function mergeVideoAndChannel(videos, channels)
function calculateMetrics(video, channel)
function gradePerformance(ratio)
function gradeContribution(ratio)

// 포맷터
function formatKoreanNumber(n)
function formatDate(iso)
function formatDuration(iso8601)

// 렌더링
function renderCard(item)
function renderResults(items)
function renderEmptyState()
function renderError(message)

// 북마크
function getBookmarks()
function addBookmark(item)
function removeBookmark(videoId)
function isBookmarked(videoId)

// 탭
function switchTab(tabName)  // 'search' | 'bookmark'

// 할당량
function getQuotaUsed()
function addQuotaUsage(units)
function resetQuotaIfNewDay()
```

---

## 작업 순서 (단계별 빌드)

> 한 번에 다 짜지 말고 단계별로 만들고 동작 확인 후 다음 단계로.

1. **HTML 뼈대** — 헤더, 탭, 검색 폼, 결과 영역
2. **CSS 토큰 + 기본 스타일** — 색상/타이포 변수, 헤더, 검색 폼
3. **API 키 모달** — 입력/저장/표시 동작 확인
4. **검색 API 체인** — `search.list → videos.list → channels.list` 호출 후 콘솔에 데이터 확인
5. **데이터 머지 + 메트릭 계산** — 콘솔로 확인
6. **카드 렌더링** — 카드 컴포넌트 CSS + DOM 생성
7. **포맷터 적용** — 숫자/날짜/영상길이
8. **북마크 기능** — 별 클릭 → localStorage 저장 → 탭 전환 → 북마크 카드 표시
9. **에러 처리 + 빈 상태**
10. **할당량 추적 UI**
11. **모바일 반응형 점검**

각 단계 끝나면 동작 확인 후 다음으로.

---

## 작업 시 주의사항

- **외부 라이브러리 금지**: jQuery, React 등 일체 사용 안 함. Vanilla JS만.
- **CSS 변수 사용**: 색상/간격은 `:root`에 변수로 정의 후 참조.
- **주석은 한국어**로 작성.
- **async/await + try/catch** 일관되게.
- API 호출은 `URLSearchParams`로 파라미터 조립.
- 채널 ID는 **중복 제거 후** channels.list 호출 (할당량 절약).
- 카드 클릭 시 영상 새 탭으로: `https://www.youtube.com/watch?v={videoId}`
- 코드를 한 번에 다 출력하지 말고 단계별로 만들면서 진행.
