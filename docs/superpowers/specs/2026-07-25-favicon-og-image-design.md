# 파비콘 & 오픈그래프 이미지 — 설계

## 배경
현재 `index.html`에는 파비콘도 오픈그래프(OG) 메타 태그도 없다. 브라우저 탭, iOS 홈 화면 추가, 카카오톡/트위터 등 링크 공유 미리보기 모두에서 브랜드 없이 노출된다.

홈 화면 히어로에는 이미 브랜드 마크로 쓰이는 종이인형 SVG가 있다(`index.html` 홈 섹션, `viewBox="0 0 256 256"`, 흰 채움 + `#4A4A4A` 연필색 윤곽선). 이 마크를 파비콘·OG 이미지의 시각적 기반으로 재사용한다.

## 범위
- 브라우저 탭 파비콘 (SVG 우선, PNG 폴백)
- iOS 홈 화면 추가 아이콘 (apple-touch-icon)
- 오픈그래프 + 트위터 카드 이미지 및 메타 태그

Storage 미사용, Firebase 데이터 모델, 게임 로직에는 영향 없음. 순수 정적 에셋 + `<head>` 메타 태그 추가.

## 단일 파일 원칙과의 관계
CLAUDE.md의 "단일 파일 `index.html`" 규약은 빌드 도구·프레임워크를 쓰지 않는다는 뜻으로 해석한다. 파비콘/OG 이미지는 브라우저·SNS 크롤러 규격상 실제 파일(png/ico/svg)로 존재해야 하며 (특히 `og:image`는 data URI를 지원하지 않음), 빌드 단계 없이 정적 파일로 커밋되므로 이 원칙에 위배되지 않는다.

## 생성할 에셋

| 파일 | 크기 | 용도 | 디자인 |
|---|---|---|---|
| `favicon.svg` | 벡터 | 브라우저 탭 아이콘 (기본) | 히어로 인형 마크를 단순화. 작은 크기(16~32px)에서 뭉개지지 않도록 세부 획 두께를 상대적으로 굵게 조정. 배경 투명 |
| `favicon-32.png` | 32×32 | SVG 파비콘 미지원 브라우저 폴백 | `favicon.svg` 래스터화 |
| `favicon-16.png` | 16×16 | 상동 | 상동 |
| `apple-touch-icon.png` | 180×180 | iOS 홈 화면 추가 아이콘 | 도화지색(`#FAF7F0`) 불투명 배경 정사각형 위에 인형 마크. iOS는 투명 배경을 흰 사각형으로 강제 대체하므로 반드시 불투명 배경 필요 |
| `og-image.png` | 1200×630 | 카카오톡/트위터 등 링크 공유 미리보기 카드 | 도화지 배경 + 가위 점선 테두리(기존 `.cut-card` 스타일 재사용) + 인형 마크(크게) + 타이틀 "꼭꼭 숨어라." + 태그라인 2줄. Gaegu 폰트 |

파비콘·홈화면 아이콘은 마크만 사용(텍스트 없음, 가독성 우선). OG 이미지만 텍스트 포함.

모든 파일은 `index.html`과 같은 레포 루트에 정적 파일로 위치.

## `index.html` `<head>` 변경

```html
<!-- 파비콘 -->
<link rel="icon" type="image/svg+xml" href="favicon.svg">
<link rel="icon" type="image/png" sizes="32x32" href="favicon-32.png">
<link rel="icon" type="image/png" sizes="16x16" href="favicon-16.png">
<link rel="apple-touch-icon" href="apple-touch-icon.png">
<meta name="theme-color" content="#FAF7F0">

<!-- Open Graph -->
<meta property="og:type" content="website">
<meta property="og:site_name" content="꼭꼭 숨어라">
<meta property="og:locale" content="ko_KR">
<meta property="og:url" content="https://jstd-jeostdu.github.io/kkokkkok-sumeora/">
<meta property="og:title" content="꼭꼭 숨어라">
<meta property="og:description" content="사진에서 색을 훔쳐 종이인형을 위장시키고, 친구가 그걸 찾아내는 숨바꼭질">
<meta property="og:image" content="https://jstd-jeostdu.github.io/kkokkkok-sumeora/og-image.png">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">

<!-- Twitter Card -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="꼭꼭 숨어라">
<meta name="twitter:description" content="사진에서 색을 훔쳐 종이인형을 위장시키고, 친구가 그걸 찾아내는 숨바꼭질">
<meta name="twitter:image" content="https://jstd-jeostdu.github.io/kkokkkok-sumeora/og-image.png">
```

- `og:description`/`twitter:description`은 홈 화면 태그라인(`.home-tag`)과 동일 문구 사용 — 문구 갱신 시 두 곳을 함께 확인
- `og:image`/`twitter:image`는 OG 스펙상 절대 URL 필수 → 배포 도메인(`https://jstd-jeostdu.github.io/kkokkkok-sumeora/`)을 하드코딩. 도메인이 바뀌면 이 값들만 수정
- `theme-color`는 도화지색(`#FAF7F0`)으로, 모바일 브라우저 주소창 색상에도 브랜드 반영

## 생성 & 검증 프로세스

1. **`favicon.svg`**: 래스터화 없이 직접 벡터 코드 작성. 기존 히어로 인형 경로(circle+rect 조합)를 참고해 작은 크기용으로 획 두께·디테일 조정
2. **PNG 3종** (`favicon-32.png`, `favicon-16.png`, `apple-touch-icon.png`, `og-image.png`): 스크래치패드에 각 타깃과 동일한 픽셀 크기의 임시 HTML(사이트와 동일한 CSS 디자인 토큰 — 도화지/연필/포인트레드/Gaegu — 사용)을 작성 → Chrome 브라우저 자동화 도구로 창 크기를 타깃 픽셀에 정확히 맞춘 뒤 스크린샷 → PNG로 저장
3. **검증**:
   - 각 임시 HTML을 브라우저로 먼저 육안 확인(레이아웃 깨짐, 텍스트 잘림 여부)
   - 최종 PNG 파일의 픽셀 치수가 의도한 값과 일치하는지 확인
   - `index.html`을 로컬에서 열어 새로 추가한 `<link>`/`<meta>` 태그의 리소스가 404 없이 로드되는지 확인
   - 실제 카카오톡/트위터 미리보기 검증은 GitHub Pages 배포 후에만 가능하므로 이번 작업 범위 밖. 배포 후 수동 확인 항목으로 남김(기존 CLAUDE.md 배포 절차의 실기기 검증 단계에 포함해서 진행 권장)

## 영향받지 않는 것
- Firebase 데이터 모델, 게임 로직, 기존 화면 전환·스타일 전혀 변경 없음
- 공유 카드(1080×1080, 플레이 결과별 canvas 생성) 기능과는 별개 — OG 이미지는 사이트 전체를 대표하는 고정 이미지 1장이며, 공유 카드는 스테이지별 동적 생성 이미지로 서로 대체 관계가 아님
