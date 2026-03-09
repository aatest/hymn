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
    <div.sidebar>  ← 검색, 필터 탭, 목록 버튼, 곡 목록
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
  id: "1234567890",      // Date.now() 문자열
  type: "hymn",          // "hymn" | "gospel"
  number: "93장",        // 표시용 번호 문자열 (찬송가: "N장", 복음성가: "N번", 직접입력: 자유)
  title: "예수 사랑하심은",
  images: [              // Base64 데이터 URL 배열
    "data:image/jpeg;base64,..."
  ]
}
```

### localStorage
```javascript
localStorage.getItem('hymnSongs')  // JSON.stringify(songs 배열)
```

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
let songs = []          // 전체 곡 목록 (localStorage에서 로드)
let activeId = null     // 현재 선택된 곡 id
let currentPageIdx = 0  // 현재 표시 중인 페이지 인덱스
let zoomScale = 1       // 확대 배율 (1 = fit)
let searchQuery = ""    // 검색 필터 문자열
let filterType = "all"  // "all" | "hymn" | "gospel"
let dragSrcId = null    // 드래그 중인 곡 id

// 목차 모달
let catalogType = "hymn"   // "hymn" | "gospel"
let catalogQuery = ""      // 목차 검색어
```

---

## 핵심 함수

### 목록 관련
```javascript
save()              // songs → localStorage 저장
filteredSongs()     // searchQuery + filterType 적용한 목록 반환
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

### 저장/불러오기
```javascript
// exportBtn 클릭 → songs 배열을 JSON으로 직렬화 → .json 파일 다운로드
// importBtn 클릭 → 파일 선택 → JSON 파싱 → 추가 또는 교체
```

### 목차 모달
```javascript
openCatalog()         // 모달 열기
closeCatalog()        // 모달 닫기
renderCatalog()       // 현재 catalogType/catalogQuery에 맞게 목록 렌더
addHymnFromCatalog(num, title, type)  // 목차에서 곡 추가
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

### 이미지 저장 방식
- 이미지는 FileReader API로 **Base64 Data URL**로 변환하여 song.images 배열에 저장
- localStorage 용량 한계(~5MB)로 인해 고해상도 이미지 다수 저장 시 용량 초과 가능
- 저장/불러오기(.json) 기능으로 백업 및 기기 간 이전 권장

### 이미지 표시 방식
- `zoomScale === 1` 일 때: `max-width`, `max-height` 기반으로 화면에 맞게 표시 (transform 미사용)
- `zoomScale > 1` 일 때: `width: N%`로 확대, 스크롤로 탐색
- `transform: scale()` 방식은 초기 로드 시 크기가 맞지 않아 폐기

### iOS 사파리 드래그 충돌 해결
- `div.draggable = true` 전체 적용 시 iOS에서 탭이 드래그로 인식되어 클릭 불가
- 해결: `draggable`을 기본 `false`로 두고, `.drag-handle`의 `mousedown` 시에만 `true`로 전환
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
- `songs` 배열 변경 후 반드시 `localStorage.setItem('hymnSongs', JSON.stringify(songs))` 호출
- `renderList()` 호출 시 현재 `activeId`를 기준으로 active 클래스를 재적용하므로 `activeId` 변경 후 호출
- HYMN_DB / GOSPEL_DB는 읽기 전용 상수 — 런타임에 수정하지 않음
