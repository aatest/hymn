# CLAUDE.md — 개발자 참조 문서

이 문서는 `hymn-viewer.html` 코드 구조와 주요 설계 결정을 설명합니다.  
AI 또는 개발자가 코드를 수정할 때 참조하세요.

---

## 파일 구조

단일 HTML 파일 (`hymn-viewer.html`) 안에 모든 것이 포함됩니다.

```
<head>
  <style>          ← 전체 CSS (CSS 변수 포함)
</head>
<body>
  <header>         ← 로고, 곡 수 뱃지, 저장/불러오기 버튼
  <div.main>
    <div.sidebar>  ← 악보 폴더 설정 바, 검색, 필터 탭, 목록 버튼, 곡 목록
    <div.viewer>   ← 악보 이미지 뷰어
  <div#modalOverlay>    ← 악보 추가 모달
  <div#catalogOverlay>  ← 찬송가/복음성가 목차 검색 모달
  <script>         ← 전체 JS 로직 + 내장 데이터
</body>
```

---

## 데이터 모델

### song 객체
```javascript
{
  id: "1234567890abc",   // Date.now() + random 문자열
  type: "hymn",          // "hymn" | "gospel"
  number: "93장",        // 표시용 번호 문자열 (찬송가: "N장", 복음성가: "N번", 직접입력: 자유)
  title: "예수 사랑하심은",
  images: [              // Base64 데이터 URL 배열
    "data:image/jpeg;base64,..."
  ],
  _order: 0              // IndexedDB 정렬용 인덱스 (dbPutAll에서 자동 부여)
}
```

### IndexedDB
```javascript
DB_NAME    = 'HymnViewerDB'
DB_VERSION = 1
STORE      = 'songs'     // keyPath: 'id'
```
- `dbGetAll()` — 전체 곡 로드 후 `_order` 기준 정렬
- `dbPutAll(songList)` — 기존 전체 삭제 후 `_order` 부여하여 재저장
- `initDB()` — DB 열기 + localStorage 마이그레이션 + 초기 로드

### localStorage 마이그레이션
- 앱 첫 실행 시 `localStorage.getItem('hymnSongs')`가 있으면 IndexedDB로 자동 이전 후 삭제
- 이후 localStorage는 사용하지 않음

### 내장 데이터 배열
```javascript
HYMN_DB    // 새찬송가 645곡: [{ n: 1, t: "만복의 근원 하나님" }, ...]
GOSPEL_DB  // 복음성가 198곡: [{ n: 1, t: "가장 높은 곳에서", a: "예수전도단" }, ...]
```
- `n`: 번호 (1부터 순번)
- `t`: 제목
- `a`: 아티스트/단체 (복음성가만, 없으면 `""`)

---

## 주요 전역 변수

```javascript
let songs = []          // 전체 곡 목록 (IndexedDB에서 로드)
let activeId = null     // 현재 선택된 곡 id
let currentPageIdx = 0  // 현재 표시 중인 페이지 인덱스
let zoomScale = 1       // 확대 배율 (1 = fit)
let searchQuery = ""    // 검색 필터 문자열
let filterMode = "all"  // "all" | "hymn" | "gospel"
let dragSrcId = null    // 드래그 중인 곡 id

// 목차 모달
let catalogType = "hymn"   // "hymn" | "gospel"
let catalogQuery = ""      // 목차 검색어

// 악보 이미지 폴더
let folderImageMap = {}    // { [num]: [dataUrl, ...] } — 번호별 이미지 배열
let folderName = ""        // 표시용 폴더명
let folderFileCount = 0    // 선택 폴더의 총 이미지 파일 수
let folderMappedCount = 0  // 번호 인식에 성공한 곡 수
```

---

## 핵심 함수

### 저장/로드 (IndexedDB)
```javascript
openDB()              // IndexedDB 열기 (Promise)
dbGetAll()            // 전체 곡 읽기 (Promise → song[])
dbPutAll(songList)    // 전체 곡 저장 (Promise), _order 자동 부여
save()                // songs → dbPutAll 호출 (async)
initDB()              // 앱 시작 시 DB 초기화 + 마이그레이션 + renderList
```

### 목록 관련
```javascript
filteredSongs()     // searchQuery + filterMode 적용한 목록 반환
renderList()        // 사이드바 목록 DOM 재렌더
openSong(id)        // 곡 선택 → 뷰어 열기
```

### 뷰어 관련
```javascript
showViewer(song)    // 뷰어 영역 표시 + 렌더
showNoSelection()   // 뷰어 영역 숨김 (안내 문구 표시)
renderPage(song)    // 현재 페이지 이미지 표시 + 줌 적용
renderThumbs(song)  // 하단 썸네일 렌더
```

### 이동 관련
```javascript
navigate(dir)         // 페이지 이동 (-1/+1), 곡 경계에서 자동 곡 전환
navigateSong(dir)     // 곡 단위 이동 (-1/+1)
getSongListIndex(id)  // filteredSongs() 기준 현재 곡 인덱스 반환
updateContNav()       // 곡 간 이동 버튼 텍스트/상태 갱신
```

### 악보 이미지 폴더
```javascript
parseHymnFilename(filename)  // 파일명에서 { num, page } 추출, 인식 불가 시 null
getFolderImages(num)         // 번호에 해당하는 dataUrl[] 반환 (없으면 [])
updateFolderUI()             // 폴더 설정 바 UI 갱신
updateFolderScanInfo()       // 목차 모달 내 스캔 결과 표시줄 갱신
clearFolder()                // 폴더 설정 초기화
```

### 목차 모달
```javascript
openCatalog()         // 모달 열기
closeCatalog()        // 모달 닫기
renderCatalog()       // 현재 catalogType/catalogQuery에 맞게 목록 렌더
addHymnFromCatalog(num, title, type)  // 목차에서 곡 추가 (폴더 이미지 자동 연결)
highlight(text, q)    // 검색어 하이라이트 HTML 반환
```

---

## CSS 변수 (테마)

```css
:root {
  --bg            /* 전체 배경 (베이지) */
  --sidebar-bg    /* 사이드바 배경 (밝은 베이지) */
  --sidebar-text  /* 사이드바 텍스트 (진한 브라운) */
  --sidebar-muted /* 보조 텍스트 색 */
  --accent        /* 강조색 (황금/앰버) */
  --accent-light  /* 밝은 강조색 */
  --viewer-bg     /* 뷰어 배경 (밝은 아이보리) */
  --border        /* 테두리 색 */
  --hover         /* 호버 배경 */
  --active        /* 선택 강조색 */
  --tag-hymn      /* 찬송가 태그 배경 */
  --tag-gospel    /* 복음성가 태그 배경 */
  --shadow        /* 그림자 */
}
```

테마 전체 변경 시 `:root` 변수와 하드코딩된 색상(`rgba(196,146,42,...)` 계열)을 함께 교체해야 합니다.

---

## 주요 설계 결정

### 데이터 저장 방식 (IndexedDB)
- `localStorage` (~5MB 한계)에서 **IndexedDB**로 전환하여 용량 제한 제거
- 이미지는 FileReader API로 **Base64 Data URL**로 변환하여 `song.images` 배열에 저장
- `save()`는 `async` 함수 — `songs` 배열 변경 후 `await save()` 또는 `save()` 호출
- 순서 보장을 위해 `dbPutAll`에서 `_order` 인덱스를 부여하고, 로드 시 이 값으로 정렬

### 악보 이미지 폴더 자동 연결
- 브라우저 보안상 로컬 경로를 직접 접근할 수 없으므로 `<input webkitdirectory>`로 폴더 선택
- `parseHymnFilename()`이 파일명에서 번호(num)와 페이지(page)를 추출
- 지원 패턴: `093.jpg`, `찬송가093.jpg`, `새찬송가_093.jpg`, `hymn093.jpg`, `093-1.jpg` 등
- 모든 파일을 FileReader로 Base64 변환 후 `folderImageMap[num]`에 페이지 순 정렬하여 저장
- `addHymnFromCatalog()`에서 `getFolderImages(num)` 호출로 이미지 자동 첨부

### 이미지 표시 방식
- `zoomScale === 1` 일 때: `max-width`, `max-height` 기반으로 화면에 맞게 표시
- `zoomScale > 1` 일 때: `width: N%`로 확대, 스크롤로 탐색
- `transform: scale()` 방식은 초기 로드 시 크기 불일치로 폐기

### iOS 사파리 드래그 충돌 해결
- `draggable`을 기본 `false`로 두고 `.drag-handle`의 `mousedown` 시에만 `true`로 전환
- `dragend` 시 `draggable = false` 및 `dragSrcId = null` 초기화
- `dragleave`는 `e.relatedTarget`으로 실제 영역 이탈 여부를 확인하여 깜빡임 방지
- `touchend` 이벤트를 별도 등록하여 iOS 탭 동작 보장

### 곡 간 이동 경계 처리
- `navigate(dir)`: 마지막 페이지에서 +1 → `navigateSong(+1)`, 첫 페이지에서 -1 → `navigateSong(-1)` 호출
- `filteredSongs()` 기준으로 인덱스 계산 → 검색/필터 상태에서도 올바르게 동작

### 목차 중복 확인
- 이미 추가된 곡 체크: `songs.filter(s => s.type === catalogType).map(s => s.number)` 기준
- 찬송가는 `"N장"`, 복음성가는 `"N번"` 형식으로 number 저장

---

## 수정 시 주의사항

- `saveBtn` id는 악보 추가 모달 내부에 이미 사용 중 → 헤더 저장 버튼은 `exportBtn` 사용
- `songs` 배열 변경 후 반드시 `save()` 호출 (`async`이므로 필요 시 `await`)
- `renderList()` 호출 시 현재 `activeId`를 기준으로 active 클래스를 재적용하므로 `activeId` 변경 후 호출
- HYMN_DB / GOSPEL_DB는 읽기 전용 상수 — 런타임에 수정하지 않음
- `folderImageMap`은 세션 중에만 유지됨 — 페이지 새로고침 시 폴더를 다시 선택해야 함
