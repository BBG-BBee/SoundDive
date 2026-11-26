# 웹사이트

SoundDive 웹사이트를 만들 때의 규칙입니다. 컴포넌트는 `components/bundle.css`의 `sd-` 클래스로 구현되어 있고, 조립한 예시는 `components/LandingPage`(홈)와 `components/ActivityArchive`(활동 기록)입니다.

## 사이트의 목적과 구조

웹사이트는 두 가지 일을 합니다. 운영진에게는 **활동을 쌓아 두는 기록장**이고, 처음 들어온 사람에게는 **"이런 활동을 하는 이런 동아리구나"를 보여 주는 소개**입니다. 두 목적은 같은 재료, 곧 활동 기록으로 이어집니다. 기록이 쌓일수록 소개가 설득력을 얻습니다.

| 페이지 | 처음 온 사람이 얻는 것 | 운영진이 하는 일 |
| --- | --- | --- |
| 홈 | 누가 · 무엇을 · 어떻게 하는 동아리인지, 최근에 실제로 한 일 | 최근 기록 3개가 자동으로 바뀌게 둠 |
| 활동 기록 | 모임마다 어디서 무엇을 들었는지 | 모임이 끝날 때마다 한 건씩 추가 |
| 소개 | 설립 배경, 운영 방식(격주, 투표, 인원 기준), 운영진 | 기수가 바뀔 때 갱신 |
| 함께하기 | 소식을 받는 경로, 모집 시기 | 모집 때만 모집 안내를 덧붙임 |

- 홈과 소개는 특정 기수나 행사(OT 날짜, 모집 마감일)에 묶이지 않게 씁니다. 때마다 바뀌는 내용은 활동 기록과 공지에 둡니다.
- 첫 화면은 세 가지에 답합니다. 이름(SoundDive), 무엇을 하는 동아리인지 한 문장, 다음에 볼 곳(활동 기록) 하나.
- 메뉴는 3~4개(소개 · 활동 기록 · 함께하기)로 둡니다.

## 활동 기록 쓰기

모임 한 번에 기록 한 건(`ActivityEntry`)입니다. 모임 후 일주일 안에 운영진이 올립니다.

- **활동 종류**: LP카페 · 리스닝룸 · 음감회 · 탐방 가운데 하나. 이름을 바꾸지 않아야 거르기가 됩니다.
- **날짜** `YYYY.MM.DD`, **장소 · 인원**("망원동 LP카페 · 5명"). 대관 공간을 포함해 모든 장소는 장소 이름으로 씁니다.
- **제목**: 그날의 주제를 한 줄로("겨울 저녁, 재즈 피아노 한 장"). '정기 모임 3회차' 같은 이름은 쓰지 않습니다.
- **함께 들은 곡**: '아티스트 — 곡명' 1~5줄. 이 목록이 처음 온 사람에게 동아리의 취향을 가장 잘 보여 줍니다.
- **한 줄 감상**: 존댓말 한두 문장. 개인 이름 대신 '많은 분이', '한 멤버가'처럼 씁니다.
- **사진**: 공간, 음반, 턴테이블, 손, 뒷모습. 얼굴이 알아볼 수 있게 나오면 본인 동의를 받은 것만 올립니다.
- 기수별로 묶고("1기 2026 겨울"), 묶음 안에서는 최신 기록이 위입니다.

## 기본 설정

- 화면은 다크 단일 테마입니다. 기기가 밝은 모드여도 같은 에스프레소 화면을 보여 줍니다. 페이지 바탕은 `surface`, 글자는 `ink`.
- `<html lang="ko">`, `<meta name="theme-color" content="#221B16">`(`espresso`).
- 서체는 Google Fonts 링크 하나로 불러옵니다: `https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:opsz,wght@12..96,800&family=IBM+Plex+Sans+KR:wght@400;500;600&display=swap`
- 한글은 단어 단위로 줄을 바꿉니다: `word-break: keep-all; overflow-wrap: break-word`(`.sd-root`에 들어 있습니다).
- 값은 `tokens.json`에서 만든 CSS 변수(`--surface`, `--space-24`, `--radius-card` …)로만 씁니다. HEX 값을 직접 적지 않습니다.

```html
<!doctype html>
<html lang="ko">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="theme-color" content="#221B16">
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:opsz,wght@12..96,800&family=IBM+Plex+Sans+KR:wght@400;500;600&display=swap">
<link rel="stylesheet" href="tokens.css">  <!-- tokens.json에서 만든 CSS 변수 -->
<link rel="stylesheet" href="bundle.css">  <!-- components/bundle.css -->
</head>
<body class="sd-root">
  <header class="sd-header">…</header>
  <main>…</main>
  <footer class="sd-footer">…</footer>
</body>
</html>
```

## 레이아웃

- **폭**: 내용은 `sd-container`(최대 `container-max` 1200px) 안에 둡니다. 읽는 글은 `sd-measure`(최대 `container-text` 720px, 한글 한 줄 약 40자)로 더 좁힙니다.
- **좌우 여백**: 768px 미만 `gutter-mobile`(20px), 768px 이상 `gutter-tablet`(32px), 1200px 이상 `gutter-desktop`(48px).
- **중단점**: `bp-tablet` 768px, `bp-desktop` 1200px. 작은 화면부터 설계하고 넓어질수록 열을 늘립니다. 모바일 한 열 → 태블릿 2~3열 → 데스크톱 12열 그리드(간격 `space-24`).
- **세로 리듬(울림)**: 묶음 안 `space-8`~`space-16`, 묶음 사이 `space-24`~`space-48`, 섹션 사이 `space-96`(모바일 `space-64`). 머리 묶음(`SectionHeading`)과 본문 사이는 `space-48`.
- **정렬**: 첫 화면, 한 문장 표제, 행동을 권하는 섹션만 가운데 정렬합니다. 읽는 섹션(활동, 일정, 기록)은 왼쪽 정렬입니다. 한 섹션 안에서 두 정렬을 섞지 않습니다.
- **순서(서사)**: 위에서 아래로 소리 → 깊이. 첫 화면에 가장 중요한 것 하나(무엇인가, 무엇을 하면 되는가), 아래로 갈수록 세부와 연락처. 푸터가 가장 어둡고 조용합니다.

## 첫 화면

- 포스터 락업을 다시 흘립니다: 마크 → 워드마크(`web-display`) → 140px 구분선 → 태그라인 → 동아리를 정의하는 한 문장(`web-lead`) → 버튼(주 행동 하나 + 보조 하나까지). 미리보기는 `components/Hero`.
- 바탕은 `sd-backdrop sd-grain`(`backdrop-poster` + 그레인). 그레인은 바탕에만 겁니다.
- 마크 폭은 768px 이상 288px, 미만 176px. 첫 화면 높이 안에 버튼까지 보이게 합니다.

## 타이포그래피

| 스타일 | 데스크톱 | 768px 미만 | 굵기 | 쓰는 곳 |
| --- | --- | --- | --- | --- |
| `web-display` | 72px | 48px(`web-display-sm`) | 800 | 워드마크, 짧은 영문 표제 |
| `web-statement` | 72px | 40px(`web-statement-sm`) | 600 | 한 줄로 말하는 한글 표제(섹션당 하나) |
| `web-h1` | 48px | 32px(`web-h1-sm`) | 600 | 페이지 제목(페이지당 하나) |
| `web-h2` | 32px | 26px | 600 | 섹션 제목 |
| `web-h3` | 22px | 22px | 600 | 카드·소단락 제목 |
| `web-lead` | 20px | 20px | 500 | 섹션을 여는 설명, `ink-muted` |
| `web-body` | 17px | 17px | 400 | 본문, 줄간격 1.8 |
| `web-small` | 14px | 14px | 400 | 캡션·출처·푸터, `ink-muted`(최소 크기) |
| `web-label` | 14px | 14px | 500 | 머리말·분류, 자간 0.14em, `ink-muted` |
| `web-button` | 16px | 16px | 600 | 버튼 글자 |
| `web-nav` | 15px | 15px | 500 | 상단 메뉴 |

- 본문 글자는 16px보다 작게 쓰지 않습니다. 14px는 캡션과 라벨에만 씁니다.
- 영문 대문자 머리말(ABOUT, EVENTS) 대신 한글 머리말을 씁니다.

## 색과 상태

- **오렌지는 한 화면 높이에 한 곳**입니다. 주 행동 버튼, 행동 문구 한 줄, '지금'을 가리키는 점 가운데 하나만 씁니다. 같은 화면에 두 번째 행동이 있으면 보조 버튼이나 글자 버튼으로 낮춥니다.
- **링크**: `ink` 글자에 1px `line` 밑줄(간격 0.25em), 마우스를 올리면 밑줄이 `ink`로 바뀝니다. 방문한 링크도 같은 색입니다.
- **마우스 올림**: 색·테두리만 150ms에 바꿉니다. 주 행동 버튼은 오렌지를 그대로 두고 `ink` 테두리를 더합니다. 보조 요소는 `surface-raised` 면과 `ink` 테두리.
- **키보드 포커스**: 모든 누를 수 있는 요소의 `:focus-visible`에 `focus-ring`(바탕색 2px 틈 + cream 2px 고리). 바깥선(outline)을 지우면 반드시 이 고리를 대신 둡니다.
- **현재 위치**(메뉴의 지금 페이지): `ink` 글자 + `line` 밑줄. 오렌지로 표시하지 않습니다.
- **선택됨**(필터, 탭): `surface-raised` 면 + `ink` 테두리.
- **비활성**: `ink-muted` 글자, 점선 `line` 테두리. 반투명으로 흐리게 만들지 않습니다.
- **알림·오류**: 아직 상태 색(초록·빨강)은 없습니다. 오류는 `ink` 글자와 `line` 테두리 상자에 "입력한 날짜를 다시 확인해 주세요"처럼 문장으로 알리고, 오렌지는 쓰지 않습니다. 색만으로 뜻을 전하지 않습니다.

## 모션

- **등장(rise)**: 투명도 0 → 1, 아래 16px → 제자리, 500ms ease-out, 요소마다 100ms 간격(`sd-rise`와 `--sd-i`). 한 화면의 등장은 1초 안에 끝냅니다. 첫 화면과 섹션 머리, 카드에만 씁니다.
- **상태 전환**: 150ms ease-out, 색과 테두리만.
- 마크는 움직이지 않습니다(등장할 때 나타나는 것만). 파문을 퍼뜨리거나 물방울을 떨어뜨리는 애니메이션, 회전, 깜빡임을 만들지 않습니다.
- 패럴랙스, 스크롤에 따라 움직이는 장식, 끝없이 반복되는 움직임, 자동 재생 소리는 쓰지 않습니다. 집중을 깨뜨립니다.
- `prefers-reduced-motion: reduce`이면 등장 효과를 끕니다(`bundle.css`에 들어 있습니다).

## 이미지

- 실제 활동 사진을 먼저 씁니다. LP카페·리스닝룸의 어둑하고 따뜻한 실내, 음반 재킷, 턴테이블, 스피커, 사람의 뒷모습이나 손.
- 커피잔보다 음반과 사람이 주인공인 장면을 고릅니다. 헤드폰을 끼고 웃는 스톡 사진이나 음표 그래픽은 쓰지 않습니다.
- 얼굴이 알아볼 수 있게 나오는 사진은 본인 동의를 받은 것만 올립니다.
- 사진은 `PhotoFrame`(`radius-frame`, 바탕 `surface-sunken`)에 넣고, 글은 사진 밖(캡션이나 옆 문단)에 둡니다. 사진 위에 글자를 얹거나, 흐림·발광·강한 색 필터를 씌우지 않습니다.
- 사진은 상자를 채우고(cover), 화면 캡처·지도·QR은 잘리지 않게(contain) 넣습니다.
- 모든 이미지에 무엇이 보이는지 한 문장 대체 글(alt)을 답니다. 장식용 그레인과 배경은 `aria-hidden`으로 숨깁니다.

## 아이콘

- 기능 아이콘은 Lucide(`lucide-static@1.71.0`)를 2px 선, 둥근 끝으로 씁니다. 크기 20·24px, 색 `ink-muted`(마우스를 올리면 `ink`).
- 아이콘만 있는 버튼에는 `aria-label`을 답니다(예: "메뉴 열기").
- 음표·헤드폰·스피커 아이콘을 장식으로 쓰지 않습니다. 재생·일시정지·음량 같은 플레이어 조작만 예외입니다.

## 접근성

| 글자 / 바탕 | `surface` | `surface-raised` | `surface-sunken` |
| --- | --- | --- | --- |
| `ink` | 13.8:1 | 12.0:1 | 15.5:1 |
| `ink-muted` | 6.2:1 | 5.4:1 | 7.0:1 |
| `accent` | 5.9:1 | 5.1:1 | 6.7:1 |

- `accent` 면 위의 글자는 `on-accent`(6.7:1)만 씁니다.
- 누를 수 있는 요소는 높이 44px 이상(버튼 48px). 작은 버튼(40px)은 헤더처럼 공간이 좁은 곳에만 씁니다.
- 키보드만으로 모든 링크와 버튼에 닿을 수 있어야 하고, 포커스 고리가 늘 보여야 합니다.
- 색만으로 뜻을 전하지 않습니다. 타임라인의 오렌지 점 옆에는 '오늘' 같은 말을 함께 적습니다.

## 파비콘과 공유 이미지

- **파비콘과 앱 아이콘**: `assets/Icons/`의 파일을 그대로 씁니다. 16px부터 512px까지 모두 같은 파비콘 마크 하나에서 나온 것이고, 원본 마크를 줄여 쓰지 않습니다.

```html
<link rel="icon" href="/favicon.ico" sizes="48x48">
<link rel="icon" href="/sounddive-favicon.svg" type="image/svg+xml">
<link rel="apple-touch-icon" href="/apple-touch-icon-180.png">
<link rel="manifest" href="/site.webmanifest">
<meta name="theme-color" content="#221B16">
```

```json
{
  "name": "SoundDive",
  "short_name": "SoundDive",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icon-maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ],
  "theme_color": "#221B16",
  "background_color": "#221B16",
  "display": "standalone"
}
```
- **공유 이미지(Open Graph) 1200×630**: `backdrop-poster` 바탕 + 그레인 `grain-opacity`. 왼쪽 절반 가운데에 마크(높이 약 400px), 오른쪽에 워드마크(약 96px) → 140px 구분선 → 태그라인(`kr-label` 비율). 행동 문구가 필요할 때만 그 아래 `cta` 한 줄.
