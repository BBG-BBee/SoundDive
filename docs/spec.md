# SoundDive 개발 명세

Nov 26, 2026 · @silverbug

## 개요

이 문서는 SoundDive 웹사이트를 바이브 코딩으로 구현할 때 AI와 사람이 함께 따르는 **구현 기준**입니다. 무엇을 만드는지는 [웹사이트 기획서](https://claude.ai/artifact/DTrUbZwsp3KWJcnWLTCQay), 언제 하는지는 [개발 마일스톤](https://claude.ai/artifact/J4wvTyEFEGq2qnz6AJuYYm), 어떻게 보이는지는 디자인 시스템이 정하고, 이 문서는 어떻게 만드는지를 정합니다. 문서끼리 충돌하거나 문서에 없는 결정이 필요하면 AI는 임의로 정하지 않고 먼저 질문합니다.

**범위:** 1·2단계 전체입니다. 공개 사이트, 관리자 화면, 자동 보강(카카오·유튜브), 모집·결산 페이지, 도메인 연결까지 12월 4일에 마칩니다. 3단계 항목(전사본 자동 정리, 얼굴 자동 모자이크, 누적 통계)과 멤버 로그인·댓글은 만들지 않습니다.

**용어:** 화면 문구에서는 디자인 시스템 규칙에 따라 사람을 '멤버', 동아리를 꾸리는 사람을 '운영진'이라고 부릅니다. 코드와 DB에서는 영문 이름을 씁니다.

| 화면 용어 | 코드·DB | 뜻 |
| --- | --- | --- |
| 회차 | activity | 모임 한 번 |
| 멤버 | member | 닉네임으로만 기록하는 참여자 |
| 선곡 | track | 멤버가 고른 곡 한 개 |
| 감상 | comment | 작성자 없는 코멘트 |
| 정리본 | memo(YAML) | 녹음을 AI로 정리해 붙여넣는 텍스트 |
| 게시 | publish | 초안을 공개 대상으로 확정하고 사이트를 다시 빌드 |
| 마감 | cohort.closed\_at | 기수 마지막 회차 게시로 결산을 공개하는 상태 |

## 아키텍처와 저장소 구조

&#91;embedded content: 시스템 구성 · 관리자와 공개 사이트 두 Worker\]

관리자 Worker만 D1과 R2에 씁니다. 공개 사이트는 GitHub Actions가 빌드할 때 D1을 한 번 읽어 만든 정적 파일이라, 방문자의 요청은 DB에 닿지 않고 비밀값도 거치지 않습니다. 사진은 R2의 공개 주소에서 바로 내려받습니다.

### 기술 구성

| 영역 | 선택 | 이유 |
| --- | --- | --- |
| 공개 사이트 | Astro 정적 빌드 + Workers 정적 자산 | 요청이 Worker 코드를 거치지 않아 CPU 한도와 무관 |
| 관리자 | Astro 서버 렌더링 + @astrojs/cloudflare + React 아일랜드(client:only) | 입력 상태는 브라우저가 들고, 서버는 읽기·쓰기만 |
| DB | D1(SQLite), ORM 없이 SQL | 기획서의 스키마·뷰·트리거를 그대로 드러내는 포트폴리오 |
| 파일 | R2 | 사진 공개 주소, 내보내기(egress) 요금 없음 |
| 인증 | Cloudflare Access + jose 재검증 | 운영진 2명, 비밀번호를 직접 저장하지 않음 |
| 검증·파싱 | zod, yaml | 같은 스키마를 서버·브라우저·내보내기에서 공유, 줄 번호가 있는 YAML 오류 |
| 사진 | exifr + canvas | 브라우저에서 줄이고 EXIF 제거 |
| 테스트·스크립트 | Vitest, tsx | 파서·검증 함수 테스트, TS 스크립트 실행 |
| 저장소 | pnpm 워크스페이스 | 두 앱이 packages/를 함께 씀 |
| 배포 | GitHub Actions + wrangler | 게시 → workflow\_dispatch → 빌드·배포(9장) |

버전은 설치 시점의 안정판을 씁니다. Astro와 @astrojs/cloudflare는 서로 맞는 주 버전을 고르고, Node.js는 그 Astro가 요구하는 LTS를 `.nvmrc`에 고정합니다.

### 폴더 구조

```text
sounddive/
├─ apps/
│  ├─ site/                  공개 사이트 (output: 'static')
│  │  ├─ src/pages/          5장의 라우트
│  │  ├─ src/data/           빌드 때 만든 JSON (커밋 안 함)
│  │  ├─ scripts/export-data.ts
│  │  ├─ public/             로고·파비콘(사람이 넣음), _headers
│  │  └─ wrangler.jsonc      정적 자산만, 바인딩 없음
│  └─ admin/                 관리자 (output: 'server')
│     ├─ src/pages/          6장의 화면과 src/pages/api/
│     ├─ src/components/     React 아일랜드(회차 편집기 등)
│     ├─ src/lib/            db, kakao, youtube, github, image, access
│     ├─ src/middleware.ts   10장
│     ├─ .dev.vars           로컬 비밀값 (커밋 안 함)
│     └─ wrangler.jsonc      D1·R2 바인딩, migrations_dir
├─ packages/
│  ├─ db/                    migrations/0001~0003, seed/dev.sql
│  ├─ shared/                타입, zod 스키마, 정리본 파서, labels, 날짜 변환
│  └─ ui/                    tokens.css, bundle.css, Astro 컴포넌트, scripts/build-tokens.ts
├─ docs/                     spec.md, plan.md, milestones.md, design/
├─ .github/workflows/        deploy-site.yml, deploy-admin.yml
├─ CLAUDE.md
├─ .nvmrc  package.json  pnpm-workspace.yaml  vitest.config.ts
└─ .gitignore                node_modules, dist, .astro, .wrangler, .dev.vars, apps/site/src/data/
```

### 루트 명령

```json
{
  "scripts": {
    "dev:admin": "pnpm --filter admin dev",
    "dev:site": "pnpm --filter site export-data:local && pnpm --filter site dev",
    "db:migrate:local": "pnpm --filter admin exec wrangler d1 migrations apply sounddive --local",
    "db:seed:local": "pnpm --filter admin exec wrangler d1 execute sounddive --local --file ../../packages/db/seed/dev.sql",
    "test": "vitest run",
    "build": "pnpm -r build"
  }
}
```

로컬 D1은 `apps/admin/.wrangler/state` 한 곳만 씁니다. 관리자 개발 서버와 위 명령은 이 기본 경로를 그대로 쓰고, 공개 사이트의 `export-data:local`만 `--persist-to ../admin/.wrangler/state`로 같은 곳을 가리킵니다. 관리자에서 고친 데이터가 공개 사이트 미리보기에 나오면 제대로 연결된 것입니다.

## 환경 변수·바인딩·시크릿

비밀값은 관리자 Worker와 GitHub Actions에만 둡니다. 공개 사이트 Worker는 정적 파일만 내보내므로 시크릿도 DB 바인딩도 없습니다.

| 이름 | 종류 | 위치 | 용도 |
| --- | --- | --- | --- |
| `DB` | D1 바인딩 | admin | 읽기·쓰기 |
| `PHOTOS` | R2 바인딩 | admin | 사진 업로드·삭제 |
| `KAKAO_REST_KEY` | 시크릿 | admin | 카카오 장소 검색 |
| `YOUTUBE_API_KEY` | 시크릿 | admin | 유튜브 검색 |
| `GITHUB_DISPATCH_TOKEN` | 시크릿 | admin | 게시 때 사이트 재빌드 요청. 이 저장소 하나에 Actions 쓰기 권한만 주는 fine-grained 토큰 |
| `GITHUB_REPO` | 변수 | admin | `소유자/저장소` |
| `ACCESS_TEAM_DOMAIN` | 변수 | admin | Access JWT 발급자, 예: `팀이름.cloudflareaccess.com` |
| `ACCESS_AUD` | 변수 | admin | Access 애플리케이션의 AUD 태그 |
| `ADMIN_EMAILS` | 시크릿 | admin | 허용 이메일(쉼표 구분). Access와 이중으로 확인. 공개 저장소에 운영진 이메일이 드러나지 않도록 wrangler.jsonc가 아니라 시크릿으로 넣음 |
| `PUBLIC_PHOTO_BASE` | 변수 | admin, site 빌드 | 사진 공개 주소. 도메인 전에는 r2.dev, 후에는 `https://photos.도메인` |
| `SITE_URL` | 변수 | site 빌드 | OG 태그의 절대 주소 |
| `CLOUDFLARE_API_TOKEN` | Actions 시크릿 | GitHub | 배포와 빌드 때 D1 읽기 |
| `CLOUDFLARE_ACCOUNT_ID` | Actions 시크릿 | GitHub | 계정 식별 |

- 로컬 개발용 값은 `apps/admin/.dev.vars`에 두고, 이 파일은 처음부터 `.gitignore`에 넣습니다.
- 운영 시크릿은 `wrangler secret put 이름`으로 사람이 직접 입력합니다. 값은 AI 대화나 코드에 붙여넣지 않습니다.
- `CLOUDFLARE_API_TOKEN`은 Workers 배포와 D1 읽기에 필요한 권한만 줍니다. `GITHUB_DISPATCH_TOKEN`은 만료일이 있으므로 만료 전에 교체합니다.
- 유튜브 API 키는 Google Cloud에서 'YouTube Data API v3'만 쓸 수 있도록 제한합니다.

## 데이터베이스

스키마의 원본은 기획서의 '스키마 초안'입니다. 그대로 `0001_init.sql`로 옮기고, 구현에 필요한 보완은 `0002`부터 새 파일로 더합니다. 이미 적용한 마이그레이션 파일은 고치지 않습니다.

### 마이그레이션

| 파일 | 내용 |
| --- | --- |
| `0001_init.sql` | 기획서 스키마 전체(테이블 9개, 트리거 2개, 뷰 2개) |
| `0002_indexes_public.sql` | 조회용 인덱스, 선곡 순서 유일 제약, 공개용 뷰 |
| `0003_setting.sql` | 모집 페이지 설정을 담는 setting 테이블 |

```sql
-- 0002_indexes_public.sql
CREATE INDEX idx_activity_cohort      ON activity(cohort_id, seq);
CREATE INDEX idx_activity_published   ON activity(published_at, status);
CREATE UNIQUE INDEX uq_track_order    ON track(activity_id, ord);  -- 감상을 곳에 연결할 때 쓰는 키
CREATE INDEX idx_track_member         ON track(member_id);
CREATE INDEX idx_comment_activity     ON comment(activity_id);
CREATE INDEX idx_comment_track        ON comment(track_id);
CREATE INDEX idx_photo_activity       ON photo(activity_id, ord);
CREATE INDEX idx_attendance_member    ON attendance(member_id);
CREATE INDEX idx_activity_place_place ON activity_place(place_id);

-- 공개 빌드는 이 뷰만 기준으로 읽는다(초안·무산 회차 차단)
CREATE VIEW public_activity AS
SELECT id, cohort_id, seq, date, type, theme, note, published_at
FROM activity
WHERE published_at IS NOT NULL AND status = 'done';

-- 0003_setting.sql
CREATE TABLE setting (
  key        TEXT PRIMARY KEY,
  value      TEXT NOT NULL,
  updated_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);
INSERT INTO setting (key, value) VALUES
  ('recruit.status', 'closed'),   -- open | closed
  ('recruit.form_url', ''),
  ('recruit.period', ''),         -- 화면에 그대로 쓰는 기간 문구
  ('recruit.faq', '[]');          -- [{"q": "", "a": ""}]
```

### 값의 형식

- **날짜**는 한국 날짜 기준 `YYYY-MM-DD`, **시각**은 UTC ISO 8601로 저장하고 화면에서는 Asia/Seoul로 표시합니다.
- **참/거짓**은 0·1 정수입니다.
- **사진**은 DB에 `object_key`만 두고 주소는 `PUBLIC_PHOTO_BASE + key`로 만듭니다. 도메인을 바꿔도 DB를 고칠 필요가 없습니다.
- **유튜브 썸네일**은 저장하지 않고 `https://i.ytimg.com/vi/{youtube_id}/mqdefault.jpg`로 만듭니다.

### 쿼리 규칙

1. SQL은 `packages/db/src/queries`의 함수에만 두고, 값은 `prepare().bind()`로만 넣습니다. 문자열 연결로 SQL을 만들지 않습니다.
2. 공개 데이터는 `public_activity` 뷰에서 출발합니다. 참석 명단의 개인별 행, 별칭(aliases), 초안·무산 회차는 공개 빌드에서 읽지 않습니다.
3. 회차 하나의 저장은 `DB.batch()` 한 번(트랜잭션)으로 보냅니다. 다중 행 INSERT를 쓰되 [쿼리당 바인딩 100개 한도](https://developers.cloudflare.com/d1/platform/limits)에 맞춰 나눕니다. 예를 들어 7열짜리 track은 한 문장에 14행까지입니다.
4. 무료 플랜은 요청 한 번에 D1 쿼리 50개까지입니다. 목록 화면에서 항목마다 따로 조회하는 N+1 쿼리를 쓰지 않습니다.
5. 감상은 같은 배치 안에서 `(SELECT id FROM track WHERE activity_id = ? AND ord = ?)` 서브쿼리로 곡에 연결합니다.
6. 릴레이션 표 보기는 테이블·뷰 이름을 `sqlite_master`에서 읽은 목록과 대조한 뒤에만 SQL에 넣습니다.

### 시드와 운영 데이터

- `seed/dev.sql`은 로컬 전용입니다. 1기, 멤버 3명(닉네임A\~C), 장소 2곳과 함께 네 가지 회차를 넣습니다: 게시된 방문 회차(곡 4개, 한 명이 2곡, 곡별 감상), 게시된 투어 회차(선곡 없이 회차 감상만), 무산 회차, 초안 회차. 마지막 두 회차가 공개 화면에 안 나오는지 확인하는 용도입니다.
- 운영 DB에는 시드를 넣지 않습니다. 1기(시작일 2026-12-05)는 관리자 화면에서 추가합니다.
- 원격 마이그레이션은 사람이 `wrangler d1 migrations apply sounddive --remote`로 직접 실행하고 CI에서는 자동으로 돌리지 않습니다. 잘못 적용했을 때는 Time Travel(무료 7일)로 되돌립니다.

## 공개 사이트

모든 페이지를 빌드할 때 정적으로 만듭니다(`output: 'static'`). 빌드 직전에 내보내기 스크립트가 D1에서 공개 데이터를 읽어 JSON으로 저장하고, 페이지는 이 JSON만 읽습니다. 그래서 공개 사이트 Worker에는 DB 바인딩도 비밀값도 없고, 무료 플랜의 CPU 한도에도 걸리지 않습니다.

### 데이터 내보내기

- `apps/site/scripts/export-data.ts`가 `wrangler d1 execute sounddive --json --command "…"`로 쿼리를 실행합니다. CI에서는 `--remote`, 로컬에서는 `--local`을 붙입니다.
- D1 바인딩 설정은 admin의 것을 같이 씁니다(`--config ../admin/wrangler.jsonc`). 로컬에서는 admin 개발 서버와 같은 `--persist-to` 경로를 써서, 관리자 화면에서 입력한 데이터가 공개 사이트 미리보기에 그대로 나오게 합니다.
- 결과는 `apps/site/src/data/*.json`(저장소에 올리지 않음)에 저장하고, 공유 zod 스키마로 검증한 뒤 빌드합니다.

```ts
// packages/shared/src/public-types.ts
export type PublicActivity = {
  slug: string;                 // `${cohort}-${seq}`, 예: "1-3"
  cohort: number; seq: number;
  date: string;                 // YYYY-MM-DD
  type: 'visit' | 'rental' | 'tour';
  theme: string | null;
  note: string | null;
  attendees: number;            // activity_attendees 뷰
  places: { id: string; name: string; mapUrl: string | null }[]; // 직접 입력한 장소(m- ID)는 지도 링크 없음
  coverKey: string | null;
  photoKeys: string[];
  tracks: {
    ord: number;
    nickname: string | null;    // 미매칭이면 null
    artist: string; title: string;
    reason: string | null;
    youtubeId: string | null;
    comments: string[];         // 작성자 없음
  }[];
  comments: string[];           // 회차 전체 감상
};
```

### 라우트

| 경로 | 내용 | 디자인 시스템 컴포넌트 |
| --- | --- | --- |
| `/` | 첨 화면(태그라인), 누가·무엇을·어떻게, 최근 회차 3개, 모집 중이면 지원 안내 | Hero, SectionHeading, ActivityEntry, Button (LandingPage 예시 참고) |
| `/activities` | 기수별로 묶은 회차 갤러리, 유형(LP카페·리스닝룸, 음감회, 탐방) 필터, 대표 사진 썸네일 | ActivityArchive, ActivityEntry, Tag |
| `/activities/[slug]` | 사진, 날짜·장소(카카오맵 링크)·주제·참여 인원, 선곡 목록(곡·아티스트·닉네임·선정 이유·유튜브), 곡별 감상, 회차 감상, 후기 | ActivityEntry, PhotoFrame |
| `/places` | 다녀온 장소 목록(유형별), 방문 횟수·최근 방문일 | Card, Tag |
| `/places/[id]` | 장소 정보, 음향 시스템, 방문 회차, 그곳에서 들은 곡 | SectionHeading, ActivityEntry |
| `/tracks` | 전체 선곡. 아티스트·닉네임 정렬, 페이지 안 검색 | SectionHeading |
| `/recap/[cohort]` | 마감된 기수만 생성. 회차·무산·장소·선곡 수, 최다 방문 장소, 회차 타임라인 | Timeline, Card |
| `/join` | 모집 상태·기간·활동 방식·FAQ, 지원하기(구글폼), 인스타그램 DM | SectionHeading, Button |
| `/404` | 없는 페이지 안내와 홈 링크 | — |

모든 페이지는 SiteHeader(내비게이션: 소개·활동 기록·함께하기)와 SiteFooter를 씁니다. 디자인 시스템의 '메뉴 3\~4개' 규칙에 맞춰 /activities·/places·/tracks는 '활동 기록' 아래 하위 탭(회차·장소·선곡)으로 오가고, '소개'는 홈의 소개 섹션(/#about)으로 갑니다(11장).

### 렌더링 규칙

- **보이는 것:** 게시된 done 회차만. 사람은 닉네임으로만, 참여는 인원수로만, 감상은 작성자 없이 보여 줍니다. 대관 장소도 다른 장소와 같이 장소 이름과 지도 링크를 보여 줍니다.
- **유튜브:** 썸네일과 '유튜브에서 듣기' 링크만 둡니다(새 창). 임베드 플레이어는 쓰지 않습니다.
- **사진:** 목록에는 썸네일(`_t.webp`), 상세에는 원본을 씁니다. `loading="lazy"`를 붙이고, 비율은 PhotoFrame의 고정 틀로 맞춰 화면이 흔들리지 않게 합니다. 대체 텍스트는 '{장소} {날짜} 활동 사진 {n}'입니다.
- **자바스크립트:** 기본은 0입니다. `/activities`의 유형 필터와 `/tracks`의 검색만 작은 스크립트로 처리합니다.
- **공유 미리보기:** 페이지마다 `og:title`, `og:description`, `og:image`(대표 사진의 절대 주소, 없으면 디자인 시스템의 `sounddive-profile-1080.png`), `og:locale=ko_KR`을 넣습니다.
- **검색 엔진:** `sitemap.xml`과 `robots.txt`를 만듭니다.
- **모집 페이지:** `recruit.status`가 open이면 '지원하기'를 화면의 유일한 주 버튼으로 둡니다. closed이면 다음 모집 소식을 인스타그램에서 확인하라는 안내를 둡니다.

## 관리자 화면

관리자 Worker는 서버 렌더링(`output: 'server'`)으로 돌고, 모든 요청이 Access와 미들웨어의 JWT 검증을 거칩니다(10장). 서버는 데이터를 읽고 쓰는 일만 하고, 상태가 많은 입력 폼은 브라우저에서만 도는 React 아일랜드(`client:only`)로 만듭니다. 서버가 화면을 그리는 일을 줄여 무료 플랜의 CPU 10ms 한도를 피하기 위해서입니다.

### 화면

| 경로 | 기능 |
| --- | --- |
| `/` | 대시보드: 초안 회차, 최근 게시, 경고(다음 기수가 시작됐는데 마감 안 된 기수, 유튜브 미연결 곡, 미매칭 닉네임) |
| `/activities` | 회차 목록(초안·게시·무산), 새 회차 |
| `/activities/[id]` | 회차 편집기(아래) |
| `/members` | 닉네임·별칭 추가·수정 |
| `/cohorts` | 기수 추가(시작일), 마감 상태, 실수로 마감했을 때 되돌리기 |
| `/settings` | 모집 상태·지원서 링크·기간 문구·FAQ |
| `/tables` | 테이블·뷰 목록과 행 수 |
| `/tables/[name]` | 표 보기: 50행씩 페이지, 열 정렬, 외래 키 칸은 참조 행 링크, 읽기 전용 |
| `/recap/[cohort]` | 실시간 결산(마감 전에도 표시) |
| `/guide` | 정리본 형식과 AI 프롬프트(복사 버튼) |

### 회차 편집기

'새 회차'를 누르면 날짜만 든 초안을 먼저 만들고 편집기로 이동합니다. 회차 ID가 있어야 사진을 올릴 수 있기 때문입니다. 편집기는 위에서 아래로 다섯 단계입니다.

1. **기본 정보:** 날짜, 유형, 주제, 상태(done/cancelled), '기수 마지막 회차' 체크
2. **장소:** 카카오 검색으로 고르기, 여러 곳과 순서, 처음 가는 곳이면 유형·음향 시스템 입력
3. **사진:** 여러 장 업로드(8장 처리 규칙), 순서 바꾸기, 대표 사진, 삭제
4. **정리본:** 붙여넣기 → 미리보기(오류·경고) → 적용. 적용하면 선곡·감상·참석 명단이 폼에 채워지고, 미매칭 닉네임은 멤버를 직접 고릅니다. 곡마다 유튜브 후보 3개가 썸네일로 뜨고, URL을 붙여넣어 바꿀 수도 있습니다.
5. **저장·게시:** 초안 저장, 게시(공개 사이트 재빌드 요청), 게시 취소. 게시 버튼이 이 화면의 유일한 오렌지 버튼입니다.

### API 계약

모든 API는 `/api` 아래에 있고 JSON을 주고받습니다. 입력은 zod로 검증하고, 오류는 `{ error: { code, message } }` 형태에 한국어 메시지를 담습니다.

| 메서드·경로 | 요청 | 응답 |
| --- | --- | --- |
| `POST /api/activities` | `{ date }` | `{ id }` (초안 생성) |
| `PUT /api/activities/:id` | `ActivityDraft` | `{ ok }` (전체 교체 저장) |
| `POST /api/activities/:id/publish` | — | `{ ok, rebuildRequested }` |
| `POST /api/activities/:id/unpublish` | — | `{ ok, rebuildRequested }` |
| `DELETE /api/activities/:id` | — | `{ ok }` (초안만) |
| `POST /api/memo/parse` | `{ activityId, yaml }` | `MemoPreview` (DB에 쓰지 않음) |
| `GET /api/places/search?q=` | — | `{ items: PlaceHit[] }` |
| `POST /api/places` | `PlaceInput` | `{ ok }` (있으면 갱신) |
| `GET /api/youtube/search?q=` | — | `{ items: VideoHit[] }` (최대 3개) |
| `POST /api/photos` | multipart: `activityId`, `file`, `thumb`, `takenAt` | `{ id, objectKey }` |
| `PATCH /api/photos/:id` | `{ ord?, isCover? }` | `{ ok }` |
| `DELETE /api/photos/:id` | — | `{ ok }` (R2 객체도 삭제) |
| `GET·POST·PATCH /api/members`, `/api/cohorts`, `/api/settings` | — | 기본 CRUD |
| `GET /api/tables`, `GET /api/tables/:name?page=&sort=` | — | 표 보기 데이터 |

```ts
// packages/shared/src/admin-types.ts
export type ActivityDraft = {
  date: string;                       // YYYY-MM-DD
  type: 'visit' | 'rental' | 'tour';
  theme: string | null;
  status: 'done' | 'cancelled';
  isLast: boolean;
  attendeesManual: number | null;
  note: string | null;
  places: { id: string; ord: number }[];
  attendees: number[];                // member id. 선곡자는 트리거가 추가
  tracks: {
    ord: number;
    memberId: number | null;
    artist: string; title: string;
    reason: string | null;
    youtubeId: string | null;
    comments: string[];
  }[];
  comments: string[];                 // 회차 전체 감상
};
export type PlaceHit = { id: string; name: string; address: string; x: number; y: number; mapUrl: string; category: string };
export type VideoHit = { videoId: string; title: string; channel: string; thumbnail: string };
```

**저장 순서(`PUT /api/activities/:id`, 배치 하나):** 기본 정보 UPDATE → 이 회차의 activity\_place·attendance·comment·track 삭제 → activity\_place·attendance INSERT → track INSERT(선곡자는 트리거가 참석에 추가) → comment INSERT(곡은 `ord` 서브쿼리로 연결). 게시된 회차를 수정하면 저장 후 재빌드를 요청합니다.

## 정리본 파서

파서는 `packages/shared/src/memo/`에 두고, DB를 모르는 순수 함수로 만듭니다. 입력은 YAML 문자열과 멤버 목록, 출력은 미리보기 구조와 오류·경고입니다. 형식의 예시는 기획서의 '정리본 형식'을 따릅니다.

```ts
// packages/shared/src/memo/schema.ts
import { z } from 'zod';
const text = z.string().trim().min(1);
export const Memo = z.object({
  theme: text.optional(),
  attendees: z.array(text).optional(),
  attendees_count: z.number().int().positive().optional(),
  tracks: z.array(z.object({
    by: text,
    artist: text,
    title: text,
    reason: text.optional(),
    comments: z.array(text).optional(),
  }).strict()).optional(),
  comments: z.array(text).optional(),
  note: text.optional(),
}).strict();

// parseMemo(yaml: string, members: Member[]): MemoPreview
// MemoPreview = { ok, errors: Issue[], warnings: Issue[], preview }
// Issue = { message: string; line?: number }
```

### 닉네임 매칭

1. 정규화: 유니코드 NFC, 앞뒤 공백과 전각 공백 제거, 중간 공백 제거, 영문은 소문자로 비교합니다.
2. 닉네임과 정확히 같으면 그 멤버, 아니면 별칭(쉼표 구분)에서 찾습니다.
3. 두 명 이상에 걸리거나 없으면 `memberId: null`로 두고 경고를 냅니다. 편집기에서 운영진이 고릅니다.

### 오류와 경고

| 상황 | 구분 | 메시지 예시 |
| --- | --- | --- |
| YAML 문법 오류 | 오류 | "12번째 줄의 들여쓰기를 확인해 주세요." |
| 알 수 없는 키, 빈 곡명·아티스트 | 오류 | "tracks 2번째 곡에 title이 없습니다." |
| 닉네임 미매칭 | 경고 | "'닉네임D'와 일치하는 멤버가 없습니다." |
| 감상에 멤버 닉네임·별칭이 들어 있음 | 경고 | "감상에 '닉네임A'가 들어 있습니다. 작성자가 드러나지 않는지 확인해 주세요." |
| 방문·대관 회차인데 선곡이 없음 | 경고 | "선곡이 없습니다. 투어가 아니라면 확인해 주세요." |
| 같은 곡이 두 번 | 경고 | "같은 곡이 두 번 들어 있습니다." |

선곡 순서(`ord`)는 YAML에 적힌 순서(1부터)이고, 참석 명단은 `attendees`와 선곡자를 합친 것입니다.

### 테스트 케이스(Vitest)

1. 정상: 멤버 2명, 곡 3개, 곡별 감상과 회차 감상 → 미리보기 구조 일치
2. 한 사람이 여러 곡 → `ord` 1…n, 같은 `memberId`
3. 투어: `tracks` 없이 `comments`만 → 오류 없음
4. `attendees` 없음 → 참석 명단 = 선곡자
5. `attendees_count`만 있음 → `attendeesManual`에 반영
6. 별칭으로 매칭
7. 미매칭 닉네임 → 경고, `memberId: null`
8. 알 수 없는 키 → 오류
9. YAML 문법 오류 → 줄 번호가 담긴 오류
10. 감상에 닉네임 포함 → 경고
11. 전각 공백·앞뒤 공백이 섞인 닉네임 → 정규화 후 매칭

### 정리용 AI 프롬프트

관리자 화면 `/guide`에 아래 프롬프트와 형식 예시를 복사 버튼과 함께 둡니다.

```text
아래는 음악감상 동아리 모임 녹음의 전사본 발췌입니다. 아래 YAML 형식으로만 정리해 주세요.

규칙
- 화자는 대응표의 닉네임으로 바꿉니다. 대응표: {참석자1: 닉네임A, 참석자2: 닉네임B}
- tracks: 곡마다 고른 사람(by), 아티스트, 곡명, 고른 이유를 적습니다. 한 사람이 여러 곡을 골랐으면 곡마다 항목을 만듭니다.
- comments: 다른 사람이 말한 감상만 넣습니다. 누가 말했는지, 이름과 닉네임은 쓰지 않습니다.
- 말하지 않은 내용을 지어내지 않고, 짧게 다듬기만 합니다. 언급되지 않은 항목은 빼고 추측으로 채우지 않습니다.
- YAML 밖의 설명은 붙이지 않습니다.

형식
(기획서의 정리본 형식 예시)

전사본
(여기에 붙여넣기)
```

## 외부 연동과 사진 처리

카카오와 유튜브 API는 관리자 Worker 서버에서만 부릅니다. 키는 브라우저로 내려가지 않고, 브라우저는 `/api/places/search`와 `/api/youtube/search`만 압니다. 두 검색 모두 글자를 칠 때마다가 아니라 Enter나 검색 버튼을 누를 때만 보냅니다. Worker 요청 수와 유튜브 검색 한도를 아끼기 위해서입니다.

### 카카오 장소 검색

```ts
// apps/admin/src/lib/kakao.ts
const res = await fetch(
  `https://dapi.kakao.com/v2/local/search/keyword.json?query=${encodeURIComponent(q)}&size=10`,
  { headers: { Authorization: `KakaoAK ${env.KAKAO_REST_KEY}` } },
);
// documents[] → PlaceHit
// id ← id, name ← place_name, address ← road_address_name || address_name
// x ← Number(x), y ← Number(y), mapUrl ← place_url, category ← category_name
```

- 주소는 도로명 주소(`road_address_name`)를 저장하고, 비어 있으면 지번 주소(`address_name`)를 씁니다.
- 저장(`POST /api/places`)은 `INSERT … ON CONFLICT(id) DO UPDATE SET name, address, x, y`로 합니다. 운영진이 입력한 `type`과 `system`은 덮어쓰지 않습니다.
- 공개 사이트의 지도 링크는 DB에 두지 않고 `https://place.map.kakao.com/{id}`로 만듭니다. 첫 연동 때 검색 응답의 `place_url`과 같은 형식인지 한 번 확인합니다.
- 카카오에 없는 장소(개인 대관 공간 등)는 직접 입력합니다. ID는 `m-`로 시작하는 임의 문자열이고, 좌표와 지도 링크는 비워 둡니다(`place`의 `address`·`x`·`y`는 NULL 허용).
- 응답이 401이면 "카카오 키 설정을 확인해 주세요.", 그 밖의 실패는 "장소 검색이 안 됩니다. 직접 입력해 주세요."를 보여 주고 직접 입력 칸을 엽니다.

### 유튜브 검색

```ts
// apps/admin/src/lib/youtube.ts
const url = new URL('https://www.googleapis.com/youtube/v3/search');
url.search = new URLSearchParams({
  part: 'snippet', type: 'video', maxResults: '3',
  q: `${artist} ${title}`, key: env.YOUTUBE_API_KEY,
}).toString();
// items[] → VideoHit
// videoId ← id.videoId, title ← snippet.title(HTML 엔티티를 풀어서)
// channel ← snippet.channelTitle, thumbnail ← snippet.thumbnails.medium.url
```

- 검색 한도는 기본 하루 100회이고, 태평양 시간 자정(한국 시간으로 겨울에는 오후 5시)에 초기화됩니다. 그래서 정리본을 적용할 때 모든 곡을 자동으로 검색하지 않습니다. 편집기의 '유튜브 찾기'를 누르면 `youtubeId`가 빈 곡만 차례로 검색해 첫 결과를 기본으로 고르고, 후보 3개를 썸네일로 보여 줍니다. 같은 '아티스트 + 곡명'의 결과는 편집기 상태에 보관해 다시 검색하지 않습니다.
- 한도를 넘기면 403 응답(사유 `quotaExceeded`)이 옵니다. "오늘 유튜브 검색 한도를 다 썼습니다. 링크를 붙여 넣거나 오후 5시 이후에 다시 찾아 주세요."를 보여 줍니다.
- 링크 붙여넣기는 한도를 쓰지 않습니다. `youtu.be/{id}`, `youtube.com/watch?v={id}`, `music.youtube.com/watch?v={id}`, `youtube.com/shorts/{id}`에서 ID를 뽑고 `^[A-Za-z0-9_-]{11}$`로 검사합니다.
- DB에는 `youtube_id`만 저장합니다. 썸네일은 `https://i.ytimg.com/vi/{id}/mqdefault.jpg`, 듣기 링크는 `https://www.youtube.com/watch?v={id}`로 만듭니다.

### 사진 처리

사진은 브라우저에서 줄이고 다시 인코딩한 뒤 올립니다. 서버는 받은 파일을 저장만 하므로 CPU 한도에 걸리지 않고, 캔버스로 다시 그리는 과정에서 위치 정보를 포함한 EXIF가 모두 사라집니다. 처리 코드는 `apps/admin/src/lib/image.ts`에 둡니다.

1. 파일 고르기: `<input type="file" accept="image/*" multiple>`
2. 촬영 시각: `exifr.parse(file, ['DateTimeOriginal'])`로 읽어 UTC ISO 문자열로 바꿉니다. 없으면 NULL입니다. 첫 사진의 촬영일이 회차 날짜와 다르면 "회차 날짜를 촬영일(YYYY.MM.DD)로 바꿀까요?"를 한 번 묻습니다.
3. 디코딩: `createImageBitmap(file, { imageOrientation: 'from-image' })`. 실패하면(HEIC를 열지 못하는 브라우저 등) 그 파일만 건너뛰고 "{파일 이름}을 열 수 없습니다. JPEG로 바꿔 올려 주세요."를 보여 줍니다. 변환 라이브러리는 넣지 않습니다.
4. 크기: 긴 변 기준 원본 2000px, 썸네일 800px로 줄입니다. 더 작은 사진은 키우지 않습니다.
5. 인코딩: 캔버스 `toBlob(cb, 'image/webp', 0.82)`. 결과의 `type`이 `image/webp`가 아니면(WebP 인코딩을 지원하지 않는 브라우저는 PNG를 돌려줍니다) `image/jpeg`, 0.85로 다시 인코딩합니다.
6. 업로드: 한 장씩 차례로 `POST /api/photos`(multipart: activityId, file, thumb, takenAt). 진행은 장 단위로 보여 주고, 실패한 장에만 '다시 올리기'를 둡니다.

```ts
// apps/admin/src/pages/api/photos/index.ts (요지)
const ext = file.type === 'image/webp' ? 'webp' : 'jpg';
const key = `a/${activityId}/${crypto.randomUUID()}.${ext}`;
const thumbKey = key.replace(/\.(webp|jpg)$/, '_t.$1');
const meta = (type: string) => ({
  httpMetadata: { contentType: type, cacheControl: 'public, max-age=31536000, immutable' },
});
await env.PHOTOS.put(key, file, meta(file.type));
await env.PHOTOS.put(thumbKey, thumb, meta(thumb.type));
// INSERT INTO photo (activity_id, object_key, taken_at, is_cover, ord)
// VALUES (?1, ?2, ?3,
//   NOT EXISTS (SELECT 1 FROM photo WHERE activity_id = ?1),
//   (SELECT COALESCE(MAX(ord), 0) + 1 FROM photo WHERE activity_id = ?1))
```

- 받는 파일은 `image/webp`·`image/jpeg`만, 원본 5MB·썸네일 1MB 이하로 검사하고 아니면 400을 돌려줍니다.
- 키에 임의 UUID를 쓰는 것은 R2 공개 주소가 키만 알면 열리기 때문입니다. 게시 전 회차의 사진 주소를 추측할 수 없게 합니다. 한 번 올린 키는 바뀌지 않으므로 1년 캐시를 걸어도 됩니다.
- 대표 지정(`PATCH /api/photos/:id`)은 같은 배치에서 그 회차의 다른 사진을 `is_cover = 0`으로 바꿉니다.
- 삭제(`DELETE /api/photos/:id`)는 행을 먼저 지우고 R2의 원본과 썸네일을 지웁니다. R2 삭제가 실패해도 남는 것은 보이지 않는 파일뿐입니다. 대표 사진을 지우면 `ord`가 가장 작은 사진을 대표로 올립니다. 초안 회차를 지울 때는 `a/{id}/` 접두어로 R2 객체를 찾아 함께 지웁니다.
- 그래도 CPU 한도 오류가 나면 multipart 대신 원본과 썸네일을 바이트 그대로 보내는 두 요청으로 바꿉니다.
- 얼굴 자동 모자이크는 3단계입니다. 지금은 편집기 사진 단계에 "얼굴이 알아볼 수 있게 나온 사진은 가린 뒤 올려 주세요." 한 줄을 둡니다.

## 게시·빌드·배포와 도메인 연결

### 게시 흐름

게시는 '공개 사이트에 반영할 확정'입니다. 기획서의 정의대로 무산 회차도 확정은 하지만, 공개 페이지에는 done 회차만 나옵니다(`public_activity`). 무산 회차의 확정은 결산의 무산 수에 들어갑니다.

1. 편집기에서 '게시'를 누르면 먼저 저장(`PUT`)하고 `POST /api/activities/:id/publish`를 보냅니다.
2. 서버는 done 회차라면 주제(`theme`)와 장소 1곳 이상이 있는지 확인합니다. 없으면 400과 "주제를 적어 주세요." 같은 문장을 돌려줍니다. 사진이나 선곡이 없는 것은 막지 않고 경고만 합니다.
3. `UPDATE activity SET published_at = ? WHERE id = ?`로 확정합니다. 기수 마지막 회차라면 트리거가 기수를 마감합니다(4장).
4. GitHub API로 deploy-site 워크플로 실행을 요청합니다. 응답이 204면 `rebuildRequested: true`입니다. 요청이 실패해도 게시는 되돌리지 않고, 대시보드에 '공개 사이트 다시 만들기' 버튼을 띄웁니다.
5. 빌드가 끝나야 공개 사이트에 보입니다. 대시보드는 열릴 때 GitHub API로 마지막 실행의 상태(대기·진행·성공·실패)를 한 번 읽어 보여 줍니다. 자동 새로 고침은 하지 않습니다.

게시 취소는 `published_at`을 NULL로 되돌린 뒤 같은 재빌드를 요청합니다. 마지막 회차의 게시를 취소해도 기수 마감은 자동으로 풀리지 않으므로 `/cohorts`의 되돌리기를 씁니다. 게시된 회차를 고쳐 저장할 때도 재빌드를 요청합니다.

```ts
// apps/admin/src/lib/github.ts
export async function requestSiteBuild(env: Env): Promise<boolean> {
  const res = await fetch(
    `https://api.github.com/repos/${env.GITHUB_REPO}/actions/workflows/deploy-site.yml/dispatches`,
    {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${env.GITHUB_DISPATCH_TOKEN}`,
        Accept: 'application/vnd.github+json',
        'User-Agent': 'sounddive-admin', // GitHub API는 User-Agent 없는 요청을 거절
      },
      body: JSON.stringify({ ref: 'main' }),
    },
  );
  return res.status === 204;
}
```

### 워크플로

| 파일 | 실행 조건 | 하는 일 |
| --- | --- | --- |
| `deploy-site.yml` | 관리자의 요청(workflow\_dispatch), main에 `apps/site`·`packages` 변경 | 설치 → 테스트 → D1 공개 데이터 내보내기(원격) → Astro 빌드 → site Worker 배포 |
| `deploy-admin.yml` | main에 `apps/admin`·`packages` 변경 | 설치 → 테스트 → 빌드 → admin Worker 배포 |

```yaml
# .github/workflows/deploy-site.yml (요지)
name: deploy-site
on:
  workflow_dispatch:
  push:
    branches: [main]
    paths: ['apps/site/**', 'packages/**', 'pnpm-lock.yaml']
concurrency:
  group: deploy-site
  cancel-in-progress: false # 진행 중인 배포는 끝까지, 대기는 가장 최근 한 건만
jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
      PUBLIC_PHOTO_BASE: ${{ vars.PUBLIC_PHOTO_BASE }}
      SITE_URL: ${{ vars.SITE_URL }}
    steps:
      - uses: actions/checkout@vN # vN: 만들 때의 최신 주 버전
      - uses: pnpm/action-setup@vN
      - uses: actions/setup-node@vN
        with: { node-version-file: .nvmrc, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm test
      - run: pnpm --filter site export-data:remote
      - run: pnpm --filter site build
      - run: pnpm --filter site exec wrangler deploy
```

- 빌드는 매번 D1의 최신 상태를 읽으므로, 게시가 몰려 대기가 가장 최근 한 건으로 합쳐져도 빠지는 회차가 없습니다.
- deploy-admin은 같은 모양에서 내보내기 단계가 없고 `--filter admin`을 씁니다. 원격 마이그레이션은 CI에서 돌리지 않습니다(4장). 스키마를 바꾸는 변경은 사람이 원격 마이그레이션을 먼저 적용한 뒤 코드를 올립니다. 순서가 바뀌면 그 사이의 요청이 실패합니다.
- 혼자 개발하는 동안은 main에 바로 올리되, 저장소 설정에서 강제 푸시와 브랜치 삭제만 막습니다. 공동 개발로 바꾸면 PR 필수로 바꿉니다.
- 되돌리기: 코드는 Cloudflare 대시보드의 배포 목록이나 `wrangler rollback`으로 이전 버전에, 데이터는 D1 Time Travel로 되돌립니다.

### 도메인 연결(12/4)

1. 원하는 TLD를 Cloudflare Registrar가 판매하면 거기서 삽니다. 처음부터 Cloudflare DNS에 올라가 바로 쓸 수 있습니다. 판매하지 않는 TLD는 국내 등록업체 등에서 사고 네임서버를 Cloudflare가 알려 주는 두 개로 바꿉니다. 이 반영에는 몇 시간에서 하루가량 걸릴 수 있어, 12/4 저녁에 사면 OT 전에 끝나지 않을 수 있습니다. 그래서 OT 자료의 링크와 QR은 workers.dev 주소로 준비해 두고, 도메인이 활성화되면 바꿉니다.
2. 공개 사이트 Worker의 Settings → Domains & Routes에서 Custom Domain으로 `도메인`(원하면 `www.도메인`도)을 붙입니다.
3. 관리자 Worker에 `admin.도메인`을 붙입니다. Worker 단위로 켠 Access는 새 Custom Domain까지 자동으로 보호합니다. 시크릿 창에서 열어 로그인 화면이 뜨는지 확인합니다.
4. R2 버킷의 Custom Domains에 `photos.도메인`을 붙입니다.
5. admin의 `PUBLIC_PHOTO_BASE`와 GitHub 변수 `PUBLIC_PHOTO_BASE`·`SITE_URL`을 새 주소로 바꾸고, admin을 다시 배포한 뒤 deploy-site를 수동으로 실행합니다.
6. 공개 페이지의 사진 주소, `og:image`와 sitemap의 절대 주소, 카카오톡 공유 미리보기를 확인한 뒤 R2의 r2.dev 공개 주소를 끕니다.
7. 공개 사이트의 workers.dev 주소는 OT가 끝날 때까지 예비로 두었다가 끕니다. 두 주소가 함께 검색에 잡히지 않게 하기 위해서입니다. 관리자 Worker의 workers.dev 주소는 Access가 막고 있으므로 켜 두어도 됩니다.

## 인증·보안 규칙

관리자 화면은 두 겹으로 막습니다. 바깥은 관리자 Worker 전체에 건 Cloudflare Access로, workers.dev·미리보기·Custom Domain을 가리지 않고 로그인하지 않은 요청을 Worker에 닿기 전에 돌려보냅니다. 안쪽은 미들웨어의 JWT 재검증으로, Access 설정을 실수로 끄거나 바꿔도 코드가 요청을 막습니다. 설정값이 비어 있을 때도 통과시키지 않고 막습니다.

### 미들웨어

```ts
// apps/admin/src/middleware.ts
import { defineMiddleware } from 'astro:middleware';
import { env } from 'cloudflare:workers';
import { createRemoteJWKSet } from 'jose';
import { verifyAccess } from './lib/access'; // jose의 jwtVerify를 감싼 순수 함수

let jwks: ReturnType<typeof createRemoteJWKSet> | undefined;

export const onRequest = defineMiddleware(async (ctx, next) => {
  const url = new URL(ctx.request.url);

  // 로컬 개발에서만: 개발 서버이면서 localhost일 때
  if (import.meta.env.DEV && ['localhost', '127.0.0.1'].includes(url.hostname)) {
    ctx.locals.admin = { email: 'dev@localhost' };
    return withHeaders(await next());
  }

  const token = ctx.request.headers.get('cf-access-jwt-assertion');
  if (!token || !env.ACCESS_TEAM_DOMAIN || !env.ACCESS_AUD || !env.ADMIN_EMAILS) return deny(url);

  jwks ??= createRemoteJWKSet(new URL(`https://${env.ACCESS_TEAM_DOMAIN}/cdn-cgi/access/certs`));
  const email = await verifyAccess(token, jwks, {
    issuer: `https://${env.ACCESS_TEAM_DOMAIN}`,
    audience: env.ACCESS_AUD,
    allowed: env.ADMIN_EMAILS.split(',').map((s) => s.trim().toLowerCase()),
  }); // 실패하면 null
  if (!email) return deny(url);
  ctx.locals.admin = { email };

  // 상태를 바꾸는 요청은 같은 출처에서 온 것만
  if (!['GET', 'HEAD'].includes(ctx.request.method) && ctx.request.headers.get('origin') !== url.origin) {
    return deny(url);
  }
  return withHeaders(await next());
});
```

- `deny`는 `/api` 아래면 `{ error: { code: 'forbidden', message: '권한이 없습니다.' } }`와 403을, 그 밖에는 짧은 안내 문장과 403을 돌려줍니다. `withHeaders`는 응답을 새로 감싸 `X-Frame-Options: DENY`, `Referrer-Policy: same-origin`, `X-Content-Type-Options: nosniff`, `Cache-Control: no-store`를 붙입니다.
- `import.meta.env.DEV`는 배포 빌드에서 false이므로 우회 코드는 운영에서 동작하지 않습니다. 호스트 이름 검사는 그 위에 한 겹 더 거는 것입니다.
- `ACCESS_AUD`는 관리자 Worker의 Access 애플리케이션 화면에 있는 AUD 태그이고, 팀 도메인은 `팀이름.cloudflareaccess.com` 형식입니다.
- JWKS는 모듈 범위에 한 번 만들어 재사용합니다. 요청마다 만들면 공개키를 매번 다시 받아옵니다.
- 테스트(Vitest): `verifyAccess`에 jose의 `createLocalJWKSet`으로 만든 키를 넣고, ① 토큰 없음 ② 서명 위조 ③ 다른 AUD ④ 허용 목록에 없는 이메일이 모두 거절되고 정상 토큰만 통과하는지 확인합니다. 다른 출처의 POST가 403인지는 로컬에서 요청을 보내 확인합니다.

### 비밀값

- 키·토큰은 코드, 커밋, AI 대화, 이슈, 스크린샷 어디에도 넣지 않습니다. 운영 값은 `wrangler secret put 이름`으로, 로컬 값은 `apps/admin/.dev.vars`(커밋 안 함)에 사람이 직접 넣습니다. 저장소에는 이름만 적은 `.dev.vars.example`을 둡니다.
- 운영진 이메일(`ADMIN_EMAILS`)도 시크릿으로 넣습니다(3장). `wrangler.jsonc`의 vars에 두면 공개 저장소에 그대로 드러납니다.
- 저장소 설정에서 Secret scanning과 Push protection이 켜져 있는지 확인합니다.
- 새어 나갔다면 기록을 지우는 것보다 키를 폐기하고 새로 발급하는 것이 먼저입니다. 공개 저장소는 이미 복제됐을 수 있습니다.

### 데이터 최소화

- DB에는 실명·연락처·학교를 두지 않습니다. 멤버는 닉네임과 별칭뿐이고, 감상에는 작성자 칸이 없습니다(4장).
- 공개 사이트에 나가는 데이터는 `public_activity` 뷰를 거친 것뿐입니다(5장).
- 사진은 브라우저에서 다시 인코딩해 EXIF를 지우고, 추측할 수 없는 키로 저장합니다(8장).
- 로그에는 요청 본문이나 정리본 내용을 남기지 않습니다. `console.log`에는 회차 ID와 오류 코드만 씁니다.
- 릴레이션 표 보기는 읽기 전용이고, 테이블 이름은 `sqlite_master` 목록과 대조한 뒤에만 SQL에 넣습니다(4장).

### 의존성

꼭 필요한 패키지만 씁니다: astro, @astrojs/cloudflare, @astrojs/react, react, react-dom, zod, yaml, jose, exifr, lucide-static, wrangler, typescript, tsx, vitest. AI가 새 패키지를 더하려면 이유를 먼저 묻고, 잠금 파일(`pnpm-lock.yaml`)은 항상 커밋합니다.

### 보안 점검 목록(12/3)

1. 커밋 기록에 비밀값이 없습니다. `git log -p`에서 `AIza`(구글 API 키 형식), `github_pat_`(GitHub 토큰 형식), 카카오 키의 앞 여섯 글자를 검색해 아무것도 나오지 않습니다.
2. 시크릿 창에서 관리자 주소(workers.dev, 나중에 admin.도메인)를 열면 Access 로그인 화면이 뜹니다. 허용하지 않은 이메일로는 들어가지 못합니다.
3. `verifyAccess` 테스트가 통과하고, 다른 출처의 POST는 403입니다.
4. 공개 사이트 빌드 결과(`apps/site/dist`)를 검색해 초안·무산 회차와 닉네임 외의 사람 정보가 없습니다.
5. 공개 사진 하나를 내려받아 exifr로 읽었을 때 GPS와 기기 정보가 없습니다.
6. `wrangler.jsonc`, `.github`, `docs`에 이메일과 키가 없습니다.

## 디자인 적용

화면을 만들기 전에 디자인 시스템의 `README.md`와 `02-web.md`를 읽고, 쓰려는 컴포넌트의 `README.md`와 `preview.html`을 봅니다. 이 장은 그 규칙을 이 사이트에 연결하는 방법과, 디자인 시스템과 다른 문서가 어긋나던 곳의 결정을 적습니다.

### 파일과 연결

- `packages/ui/tokens.css`: `packages/ui/scripts/build-tokens.ts`가 `tokens.json`을 읽어 만드는 CSS 변수 파일입니다. 이 스크립트는 `bundle.css`가 쓰는 `var(--…)`가 모두 정의됐는지 확인하고, 빠진 것이 있으면 실패합니다.
- `packages/ui/bundle.css`: 디자인 시스템의 `components/bundle.css`를 고치지 않고 그대로 복사합니다. 사이트에만 필요한 스타일은 각 앱의 `src/styles/`에 두고, 값은 토큰 변수만 씁니다(HEX 금지).
- `packages/ui/src/*.astro`: 컴포넌트마다 `preview.html`의 마크업과 `sd-` 클래스 구조를 그대로 옮긴 Astro 컴포넌트입니다(SiteHeader, SiteFooter, Hero, SectionHeading, ActivityEntry, PhotoFrame, Card, Tag, Button, Timeline). 클래스 이름을 새로 짓지 않습니다.
- 페이지 기본값은 `02-web.md`의 '기본 설정' 그대로입니다. `<html lang="ko">`, `theme-color` #221B16, Google Fonts 링크 하나, `<body class="sd-root">`, 다크 단일 테마.
- 이미지 파일: 로고(Logos)와 파비콘(Icons) 파일은 디자인 시스템 페이지의 자산 그룹에 올라가 있습니다. 거기서 내려받아 `apps/site/public/`에 사람이 직접 넣습니다. AI는 마크나 파비콘을 다시 그리거나 비슷하게 만들지 않습니다. 공유 이미지(1200×630)가 준비되기 전까지는 `sounddive-profile-1080.png`를 씁니다(5장).

### 어긋나던 곳과 결정

| 항목 | 디자인 시스템 | 결정 |
| --- | --- | --- |
| 메뉴 | 3\~4개(소개·활동 기록·함께하기) | 소개(/#about)·활동 기록·함께하기 세 개. 회차·장소·선곡은 활동 기록 아래 하위 탭 |
| 활동 종류 | LP카페·리스닝룸·음감회·탐방 넷 | OT의 세 가지 구분을 따라 셋: `visit` 'LP카페·리스닝룸', `rental` '음감회', `tour` '탐방'. 이름은 `packages/shared/src/labels.ts` 한 곳에만 둠 |
| 장소 표기 | (예전) 대관 공간은 동네까지 → (지금) 모든 장소를 이름으로 | 대관 장소도 장소 이름과 지도 링크를 씁니다(디자인 시스템 02-web.md도 같게 고침) |
| 모집 페이지 | 함께하기 | `/join`의 화면 이름은 '함께하기', 모집 안내는 그 안의 섹션 |

### 문구와 표기

- 이름은 `SoundDive`(S·D만 대문자, 붙여 씀)입니다. 사람은 '멤버', 꾸리는 사람은 '운영진'이고, 문장은 '\~습니다'로 짧게 쓰며 느낌표는 아낍니다.
- 이모지와 음표·헤드폰·스피커 같은 장식 아이콘을 쓰지 않습니다. 기능 아이콘은 Lucide(`lucide-static@1.71.0`)의 SVG를 2px 선, 20·24px, `ink-muted`로 쓰고, 아이콘만 있는 버튼에는 `aria-label`을 답니다.
- 날짜는 화면에서 `YYYY.MM.DD`로 씁니다. DB의 `YYYY-MM-DD`를 바꾸는 함수는 `packages/shared`에 하나만 둡니다.
- 기수 묶음 이름은 '1기 2026 겨울'처럼 씁니다. 계절은 기수 시작일의 달로 정합니다(12\~2월 겨울, 3\~5월 봄, 6\~8월 여름, 9\~11월 가을).
- 회차 카드(ActivityEntry)는 사진, 종류, 날짜, '장소 · 인원'(예: '장소 이름 · 5명'), 제목(주제), 곡 '아티스트 — 곡명' 최대 5줄(넘으면 '외 n곡'), 한 줄 감상(첫 회차 감상, 없으면 생략) 순서입니다.
- 오류와 경고는 상태 색 없이 `ink` 글자와 `line` 테두리 상자에 문장으로 씁니다. 빨강·초록을 새로 만들지 않고, 오렌지도 쓰지 않습니다.

### 화면마다 오렌지 한 곳

| 화면 | 오렌지를 쓰는 곳 |
| --- | --- |
| 홈 | Hero의 주 버튼 하나. 모집 중이면 '지원하기', 아니면 '활동 기록 보기' |
| 함께하기 | 모집 중이면 '지원하기', 아니면 없음 |
| 활동 기록·회차 상세·장소·선곡·결산 | 없음 |
| 관리자 회차 편집기 | '게시' 버튼 |
| 그 밖의 관리자 화면 | 없음(주 행동도 보조 버튼 모양) |

### 상태와 접근성

- 현재 메뉴는 `ink` 글자와 `line` 밑줄, 선택된 필터·하위 탭은 `surface-raised` 면과 `ink` 테두리로 표시합니다. 오렌지로 표시하지 않습니다.
- 누를 수 있는 요소는 높이 44px 이상(버튼 48px)이고, 모두 `focus-ring`이 보여야 합니다.
- 등장 효과(`sd-rise`)는 홈 첫 화면·섹션 머리·카드에만 씁니다. `prefers-reduced-motion` 처리는 bundle.css에 들어 있습니다.
- 관리자 화면도 같은 토큰과 bundle.css를 씁니다. 표가 많아 간격은 촘촘하게(`space-8`\~`space-16`) 두어도 되지만, 색과 오렌지 규칙은 같습니다.

## 작업별 완료 조건

모든 작업에 공통으로 `pnpm test`와 `pnpm build`가 통과하고, 새 화면 문구가 11장 규칙을 지키고, 커밋에 비밀값이 없어야 끝난 것으로 봅니다. 그 위에 작업마다 아래를 확인합니다. 순서는 마일스톤의 작업 목록과 같습니다.

| 작업 | 끝났다고 보는 기준 |
| --- | --- |
| 계정·키 준비 | `wrangler whoami`가 동아리 계정을 보여 주고, `wrangler d1 list`에 sounddive가, 대시보드에 R2 버킷이 있으며, `wrangler secret list`에 키 이름이 보입니다(값은 안 보임). |
| 저장소 뼈대·스키마·샘플 | 새로 복제한 폴더에서 `pnpm install`, `pnpm db:migrate:local`, `pnpm db:seed:local`, `pnpm dev:admin`, `pnpm dev:site`가 차례로 됩니다. 관리자에서 고친 시드 데이터가 공개 사이트 미리보기에 나옵니다. |
| Claude Code 준비 | CLAUDE.md, `docs/`, `.claude/settings.json`이 저장소에 있고, 새 세션에서 "4장의 트리거 두 개는 각각 언제 동작하나요?"에 문서대로 답합니다. `.dev.vars`를 읽어 달라고 하면 거부됩니다. |
| 첫 배포·Access | 두 Worker의 workers.dev 주소가 열립니다. 시크릿 창에서 관리자 주소는 Access 로그인 화면이 뜨고, 허용하지 않은 이메일은 들어가지 못합니다. |
| 공개 페이지 | 5장 표의 모든 라우트가 시드 데이터로 그려지고 페이지마다 OG 태그가 있습니다. 초안·무산 회차는 빌드 결과 어디에도 없습니다. 폭 360px에서 가로 스크롤이 없고, 키보드만으로 메뉴와 링크를 오갈 수 있습니다. |
| 관리자: 표 보기·멤버·기수 | `/tables`에 테이블·뷰가 모두 나오고 50행 넘김과 정렬이 됩니다. 멤버 추가·별칭 수정, 기수 추가가 저장됩니다. |
| 관리자: 회차 편집기·사진 | 새 회차를 다섯 단계로 채워 저장하고 다시 열면 그대로입니다. 사진 3장 올리기·순서 바꾸기·대표 지정·삭제가 되고, 지운 사진은 R2에서도 사라집니다. |
| 공동 개발 전환 판단(11/29 밤) | 바로 위 행이 끝났으면 혼자 계속합니다. 아니면 마일스톤의 전환 절차를 시작합니다. |
| 정리본 파서 | 7장의 테스트 11개가 통과하고, 편집기에서 붙여넣기 → 미리보기 → 적용이 됩니다. |
| 카카오·유튜브 | 장소를 검색해 고르면 `place`에 저장되고, '유튜브 찾기'가 빈 곡마다 후보 3개를 보여 줍니다. 링크 네 가지 형식이 모두 인식됩니다. |
| 게시·재빌드·결산 집계 | 게시하면 GitHub Actions가 돌고 공개 사이트에 회차가 나타나며, 게시를 취소하면 사라집니다. 마지막 회차를 게시하면 `cohort.closed_at`이 채워지고, 관리자 `/recap`은 마감 전에도 숫자를 보여 줍니다. |
| 함께하기·결산 페이지 | `recruit.status`를 바꾸고 재빌드하면 '지원하기' 버튼이 나타나고 사라집니다. 결산 페이지는 마감된 기수에만 생깁니다. |
| 보안 점검 | 10장의 점검 목록 여섯 가지가 모두 통과합니다. |
| 처음부터 게시까지(공동 운영자) | 공동 운영자가 관리자 `/guide`만 보고 모의 회차 하나를 게시까지 마칩니다. 막힌 곳은 `/guide` 문구로 고칩니다. |
| 실기기 점검 | 안드로이드와 iOS 휴대폰에서 공개 페이지를 보고, 관리자에서 휴대폰 사진을 올릴 수 있습니다. |
| 도메인 | 9장의 도메인 연결 항목이 끝났고, OT 자료의 링크가 실제로 열립니다. |

## Claude Code 운영

개발은 Claude Code로 합니다. Claude Code는 저장소 안에서 파일을 직접 고치고 테스트와 개발 서버를 돌리며, 저장소 루트의 CLAUDE.md를 세션마다 자동으로 읽습니다. 그래서 문서와 규칙을 모두 저장소 안에 둡니다.

### 저장소에 넣을 문서

| 경로 | 내용 | 갱신 |
| --- | --- | --- |
| `CLAUDE.md` | 아래 초안 | 규칙이 바뀔 때 |
| `docs/spec.md` | 이 개발 명세 | 바뀔 때마다 Markdown으로 내보내 덮어쓰고 커밋 |
| `docs/plan.md` | 웹사이트 기획서(스키마 원본) | 같음 |
| `docs/milestones.md` | 개발 마일스톤 | 같음 |
| `docs/design/` | 디자인 시스템의 README.md, 01-philosophy.md, 02-web.md, tokens.json, components/(bundle.css, 컴포넌트별 README.md와 preview.html) | 디자인이 바뀔 때 |
| `.claude/settings.json` | 아래 권한 설정 | 처음 한 번 |

- 문서를 고치면 Markdown으로 내보내 `docs/`를 덮어쓰고 커밋합니다. Claude Code는 `docs/`만 읽으므로 원본만 고치고 끝내면 예전 기준으로 코드를 씁니다.
- 디자인 시스템과 이 명세가 다르면 이 명세(11장)가 우선입니다. 디자인 시스템을 고치면 docs/design/도 다시 복사합니다.

### 권한 설정

CLAUDE.md는 지시일 뿐 강제가 아닙니다. 어겨서는 안 되는 규칙은 `.claude/settings.json`의 deny 규칙으로 막고, 배포로 이어지는 push는 매번 묻게 합니다.

```json
{
  "permissions": {
    "allow": ["Bash(pnpm test)", "Bash(pnpm build)", "Bash(pnpm db:migrate:local)", "Bash(pnpm db:seed:local)"],
    "ask": ["Bash(git push *)"],
    "deny": [
      "Read(./apps/admin/.dev.vars)",
      "Edit(./apps/admin/.dev.vars)",
      "Bash(wrangler *)",
      "Bash(npx wrangler *)",
      "Bash(pnpm exec wrangler *)"
    ]
  }
}
```

- 로컬 D1은 루트 명령(`pnpm db:*`)으로만 다루게 하고, wrangler를 직접 부르는 것은 막습니다. 배포·원격 마이그레이션·시크릿 입력은 내가 따로 연 터미널에서만 합니다.
- Bash 규칙은 Claude가 쓴 명령 문자열에 맞춰 막으므로, 다른 형태로 부르면 빠져나갈 수 있습니다. 권한 요청 창에 wrangler나 `--remote`가 보이면 거절합니다.
- 허가 창에서 '다시 묻지 않기'를 고르면 개인 설정에 저장되므로, 원격에 영향을 주는 명령에는 고르지 않습니다.

### 작업 하나에 세션 하나

1. 마일스톤에서 작업 하나를 고르고 새 세션을 엽니다(이전 작업의 대화를 이어 쓰지 않음).
2. 계획 모드로 아래 요청을 보내 바꿀 파일과 순서를 먼저 받습니다.
3. 계획이 명세와 맞으면 승인하고 구현을 맡깁니다. 테스트와 빌드 오류는 Claude Code가 직접 돌려 보고 고칩니다.
4. 내가 개발 서버로 화면을 보고 12장의 완료 조건을 확인합니다.
5. 작업 단위로 커밋하고, push(자동 배포)는 확인한 뒤에 합니다.

```text
작업: 마일스톤의 '관리자: 회차 편집기·사진'
관련 명세: docs/spec.md 6장(회차 편집기), 8장(사진 처리)
완료 조건: docs/spec.md 12장의 해당 행
바꿀 파일과 순서를 계획으로 먼저 보여 주세요. 명세에 없는 결정이 필요하면 계획에서 질문으로 남겨 주세요.
구현이 끝나면 pnpm test와 pnpm build를 돌리고, 바뀐 파일과 내가 확인할 완료 조건을 정리해 주세요.
```

### CLAUDE.md 초안

```md
# SoundDive

음악감상 연합동아리 SoundDive의 기록·소개 사이트. 공개 사이트(apps/site, 정적 빌드)와 관리자(apps/admin, 서버 렌더링, Cloudflare Access 뒤)는 별도 Worker다.

## 먼저 읽을 문서
- docs/spec.md: 구현 기준(개발 명세). 명세와 충돌하거나 명세에 없는 결정은 코드를 쓰기 전에 묻는다.
- docs/plan.md(기획서), docs/milestones.md(마일스톤), docs/design/(디자인 시스템. README.md와 02-web.md 먼저)
- 디자인 시스템과 docs/spec.md가 다르면 spec.md 11장을 따른다.

## 명령
- pnpm install
- pnpm db:migrate:local && pnpm db:seed:local  # 로컬 D1 준비(apps/admin/.wrangler/state)
- pnpm dev:admin / pnpm dev:site
- pnpm test  # Vitest
- pnpm build
- 작업을 끝내기 전에 pnpm test와 pnpm build를 통과시킨다.

## 규칙
- 비밀값은 코드·커밋·대화에 넣지 않는다. 이름만 쓰고, 값은 사람이 넣는다. .dev.vars는 읽지 않는다.
- 배포, 원격 D1 명령(--remote), wrangler secret은 실행하지 않는다. 필요하면 명령을 적어서 사람에게 넘긴다.
- SQL은 prepared statement만. 요청 하나에 쿼리 50개 이하, 쓰기는 batch 하나. 공개 데이터는 public_activity 뷰로만.
- 마이그레이션은 packages/db/migrations에 새 번호 파일로만 더한다. 이미 있는 파일은 고치지 않는다.
- 실명과 감상 작성자는 저장하지 않는다.
- 화면은 디자인 시스템 토큰 변수와 sd- 클래스만(HEX 금지). 문구는 '~습니다', '멤버'·'운영진', 이모지 없음, 화면마다 오렌지 한 곳.
- 새 패키지는 이유를 먼저 말하고 더한다.
- 설명은 한국어 존댓말로 한다.
```

### 공동 개발로 바꿀 때

공동 운영자도 저장소를 복제해 Claude Code로 작업합니다. CLAUDE.md, `docs/`, `.claude/settings.json`이 모두 저장소에 있어 두 사람의 Claude Code가 같은 규칙을 읽습니다. 필요한 것은 Node.js·pnpm 설치와 루트 명령 네 개(`pnpm install` → `pnpm db:migrate:local` → `pnpm db:seed:local` → `pnpm dev:site`)뿐이고, Cloudflare 계정 권한은 필요하지 않습니다.

## 출처

- Cloudflare, [Workers에 Access 적용하기](https://developers.cloudflare.com/workers/configuration/cloudflare-access/): Worker 하나에 켜면 Custom Domain·workers.dev·미리보기까지 보호
- Cloudflare, [Workers 정적 자산의 \_headers](https://developers.cloudflare.com/workers/static-assets/headers/)
- Google, [YouTube Data API 할당량 계산](https://developers.google.com/youtube/v3/determine_quota_cost): 검색 한도와 태평양 시간 자정 초기화
- Claude Code 문서, [CLAUDE.md와 메모리](https://code.claude.com/docs/en/memory)
- Claude Code 문서, [권한 설정](https://code.claude.com/docs/en/permissions)
- D1·R2·Workers 무료 한도와 카카오 로컬 API는 기획서의 출처를 따릅니다.
