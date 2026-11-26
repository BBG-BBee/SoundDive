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