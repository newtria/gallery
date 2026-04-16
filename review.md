# gallery — review

조사 일자: 2026-04-11
대상 커밋: `f99457f`
스택: Astro 6.1 (정적 사이트) · 순수 HTML/CSS/JS · 인라인 script · Vercel 배포
도메인: 일러스트레이터(`heznpc`) 개인 갤러리 + 커미션 안내 사이트

---

## 1. 원격 상태 (newtria/gallery)

- 미해결 이슈: **0건**
- 미해결 PR: **0건**
- 커밋 5개: init → pixiv-style 리디자인 → innerHTML XSS fix → profile/permalink → Astro 마이그레이션
- CI: 없음
- 배포: `vercel.json` 존재 (Vercel)

→ 외부 리포트 없음. 1인 personal portfolio.

---

## 2. 코드 품질 종합

### 강점

- **dependency 1개**: `astro` 만. 보안 surface 최소.
- **XSS-safe DOM 구축**: `Gallery.astro:78-126` 가 `createElement` + `setAttribute` + `textContent` 만 사용. `innerHTML`는 `gallery.innerHTML = ''` reset 1회 + `lbTagsEl.innerHTML = ''` reset 1회만. 사용자 데이터는 모두 `textContent` 로 들어감 → XSS 무방.
- **Lightbox + 키보드 nav + permalink**: `#work-N` 형식의 hash routing. ESC/←/→ 키 지원. 외부 라이브러리 없이 vanilla로.
- **테마 토글이 inline script로 layout에 박힘**: `Base.astro:28-46` — FOUC 방지의 정석적인 패턴. localStorage를 SSR 전에 read해서 `data-theme` set.
- **메이슨리 레이아웃 직접 구현**: `colH` 추적, 가장 짧은 column에 push. 라이브러리 없이 60줄.
- **반응형 col 수**: 900px 이하 2칸, 이상 3칸. resize 디바운스 200ms.
- **commission.json**: 가격/일정/FAQ 모두 데이터로 분리. 디자인 변경 없이 commission 정책만 업데이트 가능.

### Fix TODO (우선순위순)

**[P0] gallery.json 의 works 배열이 비어 있음 — 사이트 콘텐츠 0**
- 위치: `src/data/gallery.json:2-3`
  ```json
  { "works": [] }
  ```
- 증상: 사이트가 빈 상태. 방문자에게 "아직 작품이 없습니다" 만 보임. **갤러리 사이트의 핵심 콘텐츠 자체가 없음.**
- Fix: 작품을 채우거나, 채우기 전까지는 `commission.html` 만 노출하고 `/` 를 commission으로 redirect.

**[P1] 이미지 경로 컨벤션 / 호스팅 결정 부재**
- 위치: `Gallery.astro:110` `img.setAttribute('src', w.image)` — 데이터 형식이 path string. 그러나 `public/` 안에 어떤 폴더 구조로 둘지, CDN 사용할지, 워터마크 처리 어떻게 할지 결정 흔적 없음.
- Fix:
  - `public/works/{id}/{full,thumb,og}.webp` 같은 컨벤션.
  - thumb는 별도 size로 저장 (현 코드는 자연 크기를 그대로 사용 → mobile 데이터 낭비).
  - Astro의 `getImage()` API + `<Image />` 컴포넌트로 전환하면 자동 webp/avif 변환 + lazy + responsive srcset 까지 무료. **이게 가장 큰 ROI.**

**[P1] `colH` 가 image load 전에 +300 으로 추정**
- 위치: `Gallery.astro:118-126`
  ```js
  colH[minIdx] += 300;
  img.addEventListener('load', function() {
    colH[ci] += this.naturalHeight / this.naturalWidth * el.offsetWidth - 300;
  });
  ```
- 증상: 이미지 load 후 height 보정. 그러나 보정은 `colH` array 만 갱신할 뿐 이미 placement된 element들의 column은 다시 안 바뀜. → 첫 페이지 로드 시 column 높이가 시각적으로 어긋날 수 있음. resize 시에만 reflow.
- Fix: Astro의 `Image` 가 width/height 메타 알려주므로 빌드 시점에 미리 알 수 있음 → `colH` 추정 자체 불필요. 또는 CSS columns로 전환 (`column-count: 3; column-gap: …`).

**[P1] permalink가 `WORKS.indexOf` 기반**
- 위치: `Gallery.astro:152` `var globalIdx = WORKS.indexOf(w);`
- 증상: 작품 추가/삭제/순서 변경 시 모든 기존 permalink가 깨짐. 외부에서 트위터에 `#work-3` 링크를 공유했는데 작품 1개만 삭제되어도 다른 작품으로 매핑됨.
- Fix: 각 work에 `id` (slug) 부여 + `#work-{slug}` 형식. 데이터 마이그레이션 시 기존 정수 id 함께 보존.

**[P2] 헤더 링크가 `https://heznpc.github.io` 외부 페이지로**
- 위치: `Header.astro:14`
- 증상: 같은 `heznpc` 의 또 다른 사이트로 가는 링크. 그러나 본 사이트와의 관계가 헤더에 explicit하지 않음. 방문자가 두 사이트를 별개 인격으로 오해할 수 있음.
- Fix: aria-label 또는 hover tooltip으로 "메인 홈페이지" 같은 설명. 또는 favicon/타이틀 통일.

**[P2] 다국어 부재**
- 위치: 모든 UI가 한국어 하드코딩 (`전체`, `아직 작품이 없습니다`, `갤러리`, `커미션`, `상담`).
- 증상: pixiv 이주자(일본어), Twitter 외국 팔로워(영어) 접근 불가.
- Fix: Astro의 i18n routing (`/`, `/en/`, `/ja/`) + 작품 description의 locale 분리.

**[P2] OG 이미지 메타 부재**
- 위치: `Base.astro:18-19`
- 증상: `og:image` 누락. 트위터/카카오톡에 공유 시 이미지 미리보기 없음.
- Fix: 정적 OG 이미지 1장 (`/og.png`) 또는 작품 페이지마다 동적 OG (Astro의 `og-image-generator`).

**[P3] 작품 description / tag 검색 없음**
- 위치: filter는 tag만. text search 없음.
- 증상: 작품 수가 50개 넘어가면 발견성 ↓.
- Fix: client-side fuzzy search (`fuse.js` 또는 직접 구현). 정적 사이트라 server 안 필요.

**[P3] commission.json 의 `lastUpdated` 가 수동**
- 위치: `commission.json:3` `"2026-03"`
- 증상: 사람이 손으로 업데이트해야 함. 잊으면 stale.
- Fix: 빌드 시 `git log -1 --format=%cs commission.json` 로 자동 추출.

**[P3] commission FAQ에 "상업적 사용 별도 요금" 만 있고 가격은 안내 없음**
- 위치: `commission.json:41`
- 증상: 의뢰자가 매번 DM으로 물어봐야 함 → 일러스트레이터의 시간 낭비.
- Fix: 상업적 사용 라이선스 표 추가 (e.g. SNS 광고 +50%, 인쇄물 +100% 등).

**[P3] commission `priceMin/priceMax` 의 의미 불명확**
- 위치: `commission.json:6-7`
- 증상: `30000–50000` 의 범위가 무엇에 의해 결정되는지 (복잡도? 캐릭터 수?) UI에 없음.
- Fix: tier 안에 `priceFactors` 필드 추가 ("캐릭터 수, 배경 디테일, 의상 복잡도").

---

## 3. 테스트 상태

- **테스트 0개.** 정적 갤러리 사이트 + Astro 빌드 + JSON 데이터 구조이므로 코드량 자체가 매우 적고 (script 약 200줄), 단위 테스트가 필요한 함수도 명확하지 않음.
- 회귀 가드가 필요한 부분: **`gallery.json` 스키마 검증** (works 배열의 각 항목이 `image/title/date/tags` 필드를 갖는지) — JSON Schema 또는 Astro content collections로 정의 가능.
- 엉터리 테스트는 없음. 단순히 아예 없음.

---

## 4. 시장 가치 (2026-04-11 기준, 글로벌 관점)

**한 줄 평**: 시장 가치는 N/A — 본질적으로 personal portfolio. 의도된 audience 가 일러스트레이터 본인의 잠재 의뢰자이므로, "시장 가치"가 아니라 **"개인 브랜드 자산 가치"** 로 봐야 함.

**관점 전환: 일러스트레이터 portfolio + commission 페이지의 ROI**

- 한국 일러스트레이터 commission 시장은 X(Twitter) DM 상담 + Patreon/일러스트북 + 외주 사이트 (크몽, 라우드소싱) 가 주류.
- 개인 도메인 portfolio 가치:
  - 의뢰자가 신뢰할 수 있는 1차 reference. X/pixiv 만으로는 가격/일정/FAQ가 휘발됨.
  - SEO를 거의 못 받지만, X bio 의 link → portfolio → commission 페이지로 이어지는 funnel 이 자연스러움.
  - 본 사이트는 이 funnel의 마지막 페이지로서 잘 설계됨 (status badge, 가격 표, process 5단계, FAQ).
- **현재 가치 ★☆☆☆☆ (works 가 비어 있어서)** → works 채우면 ★★★☆☆.
- **국내 일러스트 commission 시장**: 정확한 통계 부재이나 국내 일러스트 외주 거래액은 연 수백억 원대로 추정되며, 1인 일러스트레이터 평균 의뢰 단가는 5–30만 원 (본 사이트의 가격 표와 일치).
- **글로벌 진출 가능성**: pixiv 활동 흔적이 있다면 일본어, X 활동이 있다면 영어 audience 가 있을 수 있음. i18n 추가 + 영어 commission 페이지가 ROI 1순위.

**경쟁/대안**

- ArtStation, DeviantArt, pixiv: 작품 게시는 가능하나 commission 정책 페이지 표현력 떨어짐 → 본 사이트의 존재 이유.
- Carrd / Big Cartel: SaaS 솔루션. 월 사용료 발생. 본 사이트는 코드 직접 작성 → 운영비 0, 자유도 ★★★★★.
- **결론**: SaaS 대신 자체 호스팅을 선택한 것은 합리적. 단 코드 품질의 실익은 **콘텐츠 (작품 + 의뢰 후기)** 에 비해 작음. P0의 "works 배열을 채우는 것" 이 가장 큰 작업.

---

## 5. 한 줄 요약

> 코드는 작고 깔끔하며 commission 페이지가 잘 구성됐지만 **갤러리 본체가 비어 있어서** 사이트가 동작 자체를 못 함. P0는 작품 등록 → P1는 Astro `Image` 컴포넌트 도입(자동 webp/srcset)과 permalink slug화 두 가지.

## Sources

(N/A — 개인 사이트로 글로벌 시장 비교 대상이 의미 없어 일반 검색 생략)
