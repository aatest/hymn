# CLAUDE.md — 개발자 참조 문서

이 문서는 `hymn-viewer.html` 코드 구조와 주요 설계 결정을 설명합니다.
AI 또는 개발자가 코드를 수정할 때 참조하세요.

> 단일 HTML 파일로 동작하는 찬송가·복음성가 악보 뷰어. CSS·JS·내장 데이터가 한 파일에 모두 포함됩니다.

---

## 파일 구조

```
<head>
  <style>          ← 전체 CSS (CSS 변수 + 반응형 미디어쿼리 포함)
</head>
<body>
  <header>             ← 로고, 곡 수 뱃지, 저장/불러오기 버튼
  <div.main>
    <div.sidebar>      ← (숨은 folderInput), 목차/추가 버튼, 검색·필터, 목록
    <div.viewer>       ← 악보 이미지 뷰어 (툴바 + 곡 이동 바 + 이미지 영역 + 썸네일)
  <div#modalOverlay>   ← 악보 추가 모달
  <div#catalogOverlay> ← 찬송가/복음성가 목차 모달 (악보 매칭 폴더 바 포함)
  <script>             ← 전체 JS 로직 + 내장 데이터
</body>
```

### 사이드바 내부 순서 (위→아래)
1. 숨은 `<input id="folderInput" webkitdirectory>` — 목차 모달이 사용
2. `#catalogBtn` (📖 찬송가 목차에서 악보 추가)
3. `#addBtn` (＋ 악보 추가)
4. `.search-area` — 검색 입력 + 전체/찬송가/복음성가 필터 탭
5. `.list-header` — "목록 (N곡)" + 전체 삭제
6. `.song-list` — 곡 목록

---

## 데이터 모델

### song 객체
```javascript
{
  id: "1234567890",      // Date.now() 문자열
  type: "hymn",          // "hymn" | "gospel"
  number: "93장",        // 표시용 번호 (찬송가 "N장", 복음성가 "N번", 직접입력 자유)
  title: "예수 사랑하심은",
  images: [              // Base64 데이터 URL 배열 (페이지 순)
    "data:image/jpeg;base64,..."
  ]
}
```

### 영구 저장 — IndexedDB
localStorage가 아니라 **IndexedDB**를 사용합니다. (이미지 Base64 누적 시 localStorage 5MB 한계를 넘기 쉬워 전환됨)

```javascript
const DB_NAME = 'HymnViewerDB';
const DB_VERSION = 1;
const STORE = 'songs';          // keyPath: "id"
```
- `initDB()` — 시작 시 DB 열고 곡 목록 로드. **구버전 `localStorage('hymnSongs')` 데이터가 있으면 자동 마이그레이션 후 삭제.**
- `save()` — `async`. `dbPutAll(songs)`로 전체 저장.
- `dbGetAll()` / `dbPutAll(list)` / `dbDeleteOne(id)` — 저장소 접근 헬퍼.

### 내장 데이터 배열 (읽기 전용 상수)
```javascript
HYMN_DB    // 새찬송가 645곡: [{ n: 1, t: "만복의 근원 하나님" }, ...]
```
- `HYMN_DB`: `n`(1~645 연속), `t`(제목). **제목은 "새찬송가 악보" 폴더의 파일명에서 추출한 표기 기준.**

---

## 주요 전역 변수

```javascript
let songs = []           // 전체 곡 목록 (IndexedDB에서 로드)
let activeId = null      // 현재 선택된 곡 id
let currentPageIdx = 0   // 현재 표시 중인 페이지 인덱스
let zoomScale = 1        // 확대 배율 (1 = 화면 맞춤)
let filterMode = 'all'   // "all" | "hymn" | "gospel"  (※ filterType 아님)
let searchQuery = ''     // 검색 필터 문자열
let pendingFiles = []    // 추가 모달에서 대기 중인 이미지 [{name, dataUrl}]
let dragSrcId = null     // 드래그 중인 곡 id
let db = null            // IndexedDB 핸들

// 악보 매칭 폴더 (목차 모달에서 사용)
let folderImageMap = {}  // 번호 → [dataUrl, ...]
let folderName = ''      // 표시용 폴더명
let folderFileCount = 0  // 총 이미지 파일 수
let folderMappedCount = 0// 번호 매핑된 파일 수

// 목차 모달
let catalogType = 'hymn' // "hymn" | "gospel"
let catalogQuery = ''    // 목차 검색어
```

---

## 핵심 함수

### 목록 / 저장
```javascript
filteredSongs()     // searchQuery + filterMode 적용한 목록 반환
renderList()        // 사이드바 목록 DOM 재렌더 (activeId 기준 active 적용)
save()              // async — songs → IndexedDB 저장
openSong(id)        // 곡 선택 → 뷰어 열기
```

### 뷰어 / 이미지
```javascript
showViewer(song)    // 뷰어 영역 표시 + 제목/태그/페이지 렌더
showNoSelection()   // 뷰어 숨김 (안내 문구)
renderPage(song)    // 현재 페이지 src 설정 + fitSheetImage 호출
fitSheetImage()     // 현재 이미지를 '영역 맞춤 크기(px) × zoomScale'로 사이징
renderThumbs(song)  // 하단 썸네일 렌더 (＋ 페이지 추가 포함)
zoom(delta) / resetZoom()
```

### 이동
```javascript
navigate(dir)         // 페이지 ±1, 곡 경계에서 navigateSong 자동 호출
navigateSong(dir)     // 곡 단위 ±1
getSongListIndex(id)  // filteredSongs() 기준 현재 곡 인덱스
updateContNav()       // 곡 간 이동 버튼 텍스트/상태 갱신
```

### 모달 / 목차 / 폴더
```javascript
openModal() / closeModal()              // 악보 추가 모달
handleFiles(files) / renderFilePreview()// 추가 모달 이미지 처리/미리보기
openAddImageForSong(song)               // 썸네일 ＋ 로 곡에 페이지 추가
openCatalog() / closeCatalog()          // 목차 모달
renderCatalog()                         // catalogType/catalogQuery 기준 목록 렌더
addHymnFromCatalog(num, title, type)    // 목차에서 곡 추가 (폴더 이미지 자동 연결)
highlight(text, q)                      // 검색어 하이라이트 HTML
parseHymnFilename(filename)             // 파일명 → {num, page}
getFolderImages(num)                    // 번호에 매칭된 폴더 이미지 배열
updateFolderScanInfo()                  // 목차 모달의 '악보 매칭 폴더' 바 상태 갱신
updateFolderUI()                        // → updateFolderScanInfo()로 위임 (사이드바 바는 제거됨)
clearFolder()                           // 폴더 매칭 해제
```

### 화면/입력 보조
```javascript
setAppHeight()       // window.innerHeight → CSS 변수 --app-h
initSidebarToggle()  // 사이드바 접기/펼치기
initDragHandle()/createGhost()/performReorder()/cleanupTouch()  // 드래그 정렬
initSwipe()          // 모바일 스와이프 페이지/곡 이동
showHint()           // 키보드 단축키 안내
```

---

## CSS 변수 (테마)

```css
:root {
  --bg            /* 전체 배경 (베이지) */
  --sidebar-bg    /* 사이드바 배경 */
  --sidebar-text  /* 사이드바 텍스트 */
  --accent        /* 강조색 (황금/앰버) */
  --accent-light  /* 밝은 강조색 */
  --border        /* 테두리 */
  --shadow        /* 그림자 */
  /* 등 */
}
```
테마 변경 시 `:root` 변수와 하드코딩된 `rgba(196,146,42,...)` 계열 색을 함께 교체해야 합니다.

### 반응형
- `@media (max-width: 600px)` — 헤더 축소, 뷰어 툴바 **세로 2행 분리**(제목 행 / 페이지·줌 행), 버튼·폰트 축소
- `@media (max-width: 380px)` — 더 좁은 화면용 추가 축소

---

## 주요 설계 결정

### 이미지 표시 / 줌 (px 기반 — 중요)
- 표시 크기는 **JS에서 직접 px로 계산**: `기본 맞춤 크기 × zoomScale`.
  - `fitSheetImage()`가 `imageArea`의 가용 크기와 이미지 `naturalWidth/Height`로 기본 배율을 구하고, 거기에 `zoomScale`을 곱해 `width/height`(px)를 지정.
- **CSS `zoom` + 뷰포트 단위(`vh`)는 사용하지 않음.** 크롬/삼성인터넷에서 `zoom`이 걸린 요소 안의 `vh`가 배율로 나뉘어 *배율을 올릴수록 오히려 작아지는* 버그가 있어 폐기함.
- 재맞춤 트리거:
  - 이미지 `onload` → `requestAnimationFrame(fitSheetImage)` (레이아웃 확정 후 측정)
  - **`ResizeObserver(imageArea)`** → 사이드바 토글·회전·툴바 변화·초기 레이아웃 확정 시 자동 재맞춤
- `transform: scale()` 방식은 초기 로드 시 크기가 맞지 않아 이전에 폐기됨.

### 모바일/삼성인터넷 보정
- **폰트 부스팅 차단**: `html { -webkit-text-size-adjust:100%; text-size-adjust:100%; }` — 좁은 칸의 글자가 멋대로 거대해지는 문제 해결.
- **실제 보이는 높이**: `body { height: var(--app-h, 100vh); }` + `setAppHeight()`가 `window.innerHeight`를 `--app-h`에 반영. `100vh`가 하단 툴바까지 포함해 **맨 아래 썸네일이 잘리는 문제** 해결.
- **iOS 핀치 줌 보정**: `visualViewport`로, 실제 확대(>1.05배) 후 1배로 복귀했을 때만 `scrollTo(0,0)`. 삼성인터넷의 주소창 변화로 인한 무한 스크롤 충돌을 피하려고 조건을 엄격히 둠.

### 뷰어 제목 영역
- 제목 줄(`#vTitle`)에 **찬송가/복음성가 태그를 인라인으로 앞에** 붙임: `<span class="song-tag ...">찬송가</span><span class="vtitle-text">제목</span>`.
- `.viewer-title { display:flex; align-items:center; }`로 태그·제목 세로 중앙 정렬. 명조↔고딕 메트릭 차이로 태그에 `top` 미세 보정값 사용.
- 별도 부제 줄(`#vSubtitle`)은 비워 두며 `:empty`일 때 숨김.

### 악보 매칭 폴더 (목차 모달 내부)
- 폴더 매칭 UI는 **사이드바가 아니라 목차 모달 안**에 있음(처음 사용자가 폴더 선택 필요성을 인지하도록).
- 폴더 미설정 시 안내 + `📁 악보 폴더 선택`, 설정 시 인식 결과 + `폴더 변경/해제` 표시.
- 숨은 `#folderInput`(webkitdirectory)을 모달 버튼이 공유 사용. 복음성가 탭에서는 폴더 바를 숨김.
- 진행 표시("악보 읽는 중… n/total")는 모달의 `#folderScanText`에 출력.

### iOS 드래그 충돌 해결
- 전역 `draggable=true`는 iOS에서 탭이 드래그로 인식되어 클릭 불가 → `.drag-handle`의 `mousedown` 시에만 `draggable` 전환, 터치는 long-press 기반 고스트 드래그(`initDragHandle`)로 처리.

### 곡 간 이동 경계 처리
- `navigate(dir)`: 마지막 페이지에서 +1 → `navigateSong(+1)`, 첫 페이지에서 -1 → `navigateSong(-1)`.
- `filteredSongs()` 기준 인덱스 계산 → 검색/필터 상태에서도 정확.

---

## 수정 시 주의사항

- `save()`는 **async** — 호출부에서 필요 시 `await`. `songs` 변경 후 반드시 `save()` 호출.
- 영구 저장은 **IndexedDB**(`HymnViewerDB`/`songs`). localStorage가 아님. 구버전 데이터는 `initDB()`에서 1회 마이그레이션.
- 필터 상태 변수명은 **`filterMode`** (문서 구버전의 `filterType` 아님).
- `renderList()`는 `activeId` 기준 active 적용 → `activeId` 변경 후 호출.
- 이미지 사이징을 바꿀 땐 **CSS `zoom`/`vh`를 다시 도입하지 말 것** (위 줌 버그). 반드시 `fitSheetImage()` px 방식 유지.
- `HYMN_DB` / `GOSPEL_DB`는 읽기 전용 상수 — 런타임에 수정하지 않음.
- 헤더 저장 버튼은 `#exportBtn`, 불러오기는 `#importBtn`. (`saveBtn` id는 추가 모달 내부에서 사용 중)
