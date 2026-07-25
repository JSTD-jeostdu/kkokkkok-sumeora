# 파비콘 & 오픈그래프 이미지 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `index.html`에 파비콘(탭 아이콘 + iOS 홈 화면 아이콘)과 오픈그래프/트위터 카드 이미지를 추가한다.

**Architecture:** 기존 홈 히어로의 종이인형 마크(circle+rounded-rect 6개 도형, viewBox 0 0 256 256, 흰 채움 + `#4A4A4A` 윤곽선)를 모든 에셋의 시각적 기반으로 재사용한다.
- 텍스트가 없는 순수 도형 에셋(`favicon.svg`, `favicon-32.png`, `favicon-16.png`, `apple-touch-icon.png`)은 PowerShell `System.Drawing`(GDI+)으로 결정적으로 렌더링한다 — 브라우저 창 크기/DPI 스케일링에 좌우되지 않고 정확한 픽셀 치수를 보장하기 위함.
- 한글 텍스트 + Gaegu 폰트가 필요한 `og-image.png`만 Chrome으로 실제 CDN 폰트를 로드해 렌더링한 뒤, 캡처된 원본을 PowerShell로 정확히 1200×630으로 리샘플링해 최종 픽셀 치수를 보장한다.
- 모든 좌표는 기존 히어로 SVG(`index.html`)와 동일한 256×256 좌표계를 그대로 재사용해 브랜드 일관성을 유지한다.

**Tech Stack:** PowerShell + .NET `System.Drawing`(GDI+), Chrome 브라우저 자동화(`mcp__claude-in-chrome__*`), 순수 HTML/CSS(빌드 도구 없음)

## Global Constraints
- 단일 파일 `index.html` 원칙: 빌드 도구·프레임워크 없음. 파비콘/OG 이미지는 브라우저·SNS 크롤러 규격상 실제 정적 파일(svg/png)로 존재해야 하며, 빌드 단계 없이 그대로 커밋되므로 이 원칙에 위배되지 않는다.
- 디자인 토큰: 도화지 `#FAF7F0` · 연필 `#4A4A4A` · 연필-연함 `#9B958A` · 포인트 레드 `#E45B5B` · Gaegu 폰트
- 모든 신규 에셋 파일은 `index.html`과 같은 레포 루트에 위치한다.
- 커밋 메시지 등 사용자向 문구는 한국어.
- `og:image`/`twitter:image`는 절대 URL이어야 하므로 `https://jstd-jeostdu.github.io/kkokkkok-sumeora/` 도메인을 하드코딩한다.
- Firebase 데이터 모델·게임 로직·기존 화면 스타일은 변경하지 않는다.

---

### Task 1: `favicon.svg` 작성 (벡터 소스)

**Files:**
- Create: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\favicon.svg`

**Interfaces:**
- Consumes: 없음 (독립 정적 파일)
- Produces: `favicon.svg` — 이후 Task 4에서 `index.html`의 `<link rel="icon" type="image/svg+xml">`가 참조. Task 2의 GDI+ 래스터화 시 동일 좌표계·색상을 참고

- [ ] **Step 1: `favicon.svg` 작성**

기존 히어로 인형과 동일한 256×256 좌표계를 사용하되, 16~32px의 작은 탭 아이콘 크기에서도 윤곽선이 뭉개지지 않도록 `stroke-width`를 5 → 16으로 굵게 조정한다.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 256 256">
  <g fill="#FFFFFF" stroke="#4A4A4A" stroke-width="16" stroke-linejoin="round">
    <circle cx="128" cy="52" r="34"/>
    <rect x="92"  y="82"  width="72" height="86" rx="30"/>
    <rect x="58"  y="92"  width="26" height="62" rx="13"/>
    <rect x="172" y="92"  width="26" height="62" rx="13"/>
    <rect x="97"  y="158" width="27" height="74" rx="13"/>
    <rect x="132" y="158" width="27" height="74" rx="13"/>
  </g>
</svg>
```

- [ ] **Step 2: 브라우저에서 렌더링 확인**

`mcp__claude-in-chrome__tabs_create_mcp`로 새 탭을 만들고 `mcp__claude-in-chrome__navigate`로 `file:///C:/Users/jeodu/Desktop/VibeCoding/kkokkkok-sumeora/favicon.svg`를 연 뒤, `mcp__claude-in-chrome__computer`(action: `screenshot`)로 확인한다.

Expected: 파싱 에러 없이 흰 채움 + 진한 회색 윤곽선의 종이인형 실루엣이 표시됨. 콘솔 에러 없음.

- [ ] **Step 3: Commit**

```bash
git add favicon.svg
git commit -m "favicon.svg 추가 — 히어로 인형 마크 재사용, 작은 크기용 굵은 획"
```

---

### Task 2: 텍스트 없는 PNG 3종 생성 (favicon-32/16, apple-touch-icon)

**Files:**
- Create: `C:\Users\jeodu\AppData\Local\Temp\claude\C--Users-jeodu-Desktop-VibeCoding-kkokkkok-sumeora\c9ef31d1-d251-4951-b70b-1bd45e59fc2a\scratchpad\generate-icons.ps1` (스크래치패드, 커밋 대상 아님 — 1회성 생성 스크립트)
- Create: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\favicon-32.png`
- Create: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\favicon-16.png`
- Create: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\apple-touch-icon.png`

**Interfaces:**
- Consumes: Task 1의 `favicon.svg`와 동일한 256×256 좌표계·색상(`#FFFFFF` 채움, `#4A4A4A` 윤곽선, 도화지 `#FAF7F0`)
- Produces: `favicon-32.png`(32×32), `favicon-16.png`(16×16), `apple-touch-icon.png`(180×180, 도화지색 불투명 배경) — Task 4에서 `<link>` 태그가 참조

- [ ] **Step 1: 생성 스크립트 작성**

`generate-icons.ps1`:

```powershell
Add-Type -AssemblyName System.Drawing

function New-RoundedRectPath {
    param([float]$x, [float]$y, [float]$w, [float]$h, [float]$r)
    $path = New-Object System.Drawing.Drawing2D.GraphicsPath
    $d = $r * 2
    $path.AddArc($x, $y, $d, $d, 180, 90)
    $path.AddArc($x + $w - $d, $y, $d, $d, 270, 90)
    $path.AddArc($x + $w - $d, $y + $h - $d, $d, $d, 0, 90)
    $path.AddArc($x, $y + $h - $d, $d, $d, 90, 90)
    $path.CloseFigure()
    return $path
}

function Draw-DollMark {
    param(
        [System.Drawing.Graphics]$g,
        [float]$scale,
        [float]$strokeWidthUnits,
        [System.Drawing.Color]$fillColor,
        [System.Drawing.Color]$strokeColor
    )
    $g.SmoothingMode = [System.Drawing.Drawing2D.SmoothingMode]::AntiAlias
    $brush = New-Object System.Drawing.SolidBrush($fillColor)
    $pen = New-Object System.Drawing.Pen($strokeColor, [float]($strokeWidthUnits * $scale))
    $pen.LineJoin = [System.Drawing.Drawing2D.LineJoin]::Round

    # 머리 (원)
    $headD = 34 * 2 * $scale
    $g.FillEllipse($brush, (128 - 34) * $scale, (52 - 34) * $scale, $headD, $headD)
    $g.DrawEllipse($pen, (128 - 34) * $scale, (52 - 34) * $scale, $headD, $headD)

    # 몸통, 팔x2, 다리x2 (x, y, w, h, rx) — 히어로 SVG와 동일 좌표·순서
    $shapes = @(
        @(92, 82, 72, 86, 30),
        @(58, 92, 26, 62, 13),
        @(172, 92, 26, 62, 13),
        @(97, 158, 27, 74, 13),
        @(132, 158, 27, 74, 13)
    )
    foreach ($s in $shapes) {
        $path = New-RoundedRectPath -x ($s[0] * $scale) -y ($s[1] * $scale) -w ($s[2] * $scale) -h ($s[3] * $scale) -r ($s[4] * $scale)
        $g.FillPath($brush, $path)
        $g.DrawPath($pen, $path)
        $path.Dispose()
    }

    $pen.Dispose()
    $brush.Dispose()
}

function New-FaviconPng {
    param([int]$size, [float]$strokeWidthUnits, [string]$outPath)
    $bmp = New-Object System.Drawing.Bitmap($size, $size, [System.Drawing.Imaging.PixelFormat]::Format32bppArgb)
    $g = [System.Drawing.Graphics]::FromImage($bmp)
    $g.Clear([System.Drawing.Color]::Transparent)
    Draw-DollMark -g $g -scale ($size / 256) -strokeWidthUnits $strokeWidthUnits `
        -fillColor ([System.Drawing.Color]::White) -strokeColor ([System.Drawing.Color]::FromArgb(0x4A,0x4A,0x4A))
    $bmp.Save($outPath, [System.Drawing.Imaging.ImageFormat]::Png)
    $g.Dispose(); $bmp.Dispose()
}

$repoRoot = "C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora"

New-FaviconPng -size 32 -strokeWidthUnits 16 -outPath "$repoRoot\favicon-32.png"
New-FaviconPng -size 16 -strokeWidthUnits 16 -outPath "$repoRoot\favicon-16.png"

# apple-touch-icon: 불투명 도화지 배경 180x180, 히어로와 동일한 얇은 획(5)
$size = 180
$bmp = New-Object System.Drawing.Bitmap($size, $size, [System.Drawing.Imaging.PixelFormat]::Format32bppArgb)
$g = [System.Drawing.Graphics]::FromImage($bmp)
$paper = [System.Drawing.Color]::FromArgb(0xFA,0xF7,0xF0)
$g.Clear($paper)
Draw-DollMark -g $g -scale ($size / 256) -strokeWidthUnits 5 `
    -fillColor ([System.Drawing.Color]::White) -strokeColor ([System.Drawing.Color]::FromArgb(0x4A,0x4A,0x4A))
$bmp.Save("$repoRoot\apple-touch-icon.png", [System.Drawing.Imaging.ImageFormat]::Png)
$g.Dispose(); $bmp.Dispose()

Write-Output "done"
```

- [ ] **Step 2: 스크립트 실행**

Run: `powershell -ExecutionPolicy Bypass -File "<scratchpad>\generate-icons.ps1"`
Expected output: `done`

- [ ] **Step 3: 픽셀 치수 검증**

Run:
```powershell
Add-Type -AssemblyName System.Drawing
foreach ($f in @("favicon-32.png","favicon-16.png","apple-touch-icon.png")) {
    $img = [System.Drawing.Image]::FromFile("C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\$f")
    "$f -> $($img.Width)x$($img.Height)"
    $img.Dispose()
}
```
Expected:
```
favicon-32.png -> 32x32
favicon-16.png -> 16x16
apple-touch-icon.png -> 180x180
```

- [ ] **Step 4: 육안 확인**

Read 도구로 `favicon-32.png`, `apple-touch-icon.png`를 열어 도형이 잘리지 않고 중앙에 배치되어 있는지, `apple-touch-icon.png`는 배경이 투명하지 않고 도화지색으로 채워져 있는지 확인.

- [ ] **Step 5: Commit**

```bash
git add favicon-32.png favicon-16.png apple-touch-icon.png
git commit -m "파비콘 PNG 3종 생성 (32/16px 탭 아이콘, 180px iOS 홈화면 아이콘)"
```

---

### Task 3: `og-image.png` 생성 (1200×630, 텍스트 포함)

**Files:**
- Create: `<scratchpad>\og-image.html` (스크래치패드, 커밋 대상 아님)
- Create: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\og-image.png`

**Interfaces:**
- Consumes: 히어로 인형 좌표계, `.cut-card` 가위 점선 스타일(`index.html:61-69`), 홈 타이틀/태그라인 문구(`index.html:340-341`)
- Produces: `og-image.png`(정확히 1200×630) — Task 4에서 `og:image`/`twitter:image` meta가 절대 URL로 참조

- [ ] **Step 1: `og-image.html` 작성**

`<scratchpad>\og-image.html`:

```html
<!DOCTYPE html>
<html><head>
<meta charset="UTF-8">
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Gaegu:wght@300;400;700&display=swap" rel="stylesheet">
<style>
html,body{margin:0;padding:0;}
#card{
  width:1200px; height:630px; box-sizing:border-box;
  background:#FAF7F0; font-family:'Gaegu',sans-serif; color:#4A4A4A;
  display:flex; flex-direction:column; align-items:center; justify-content:center;
  position:relative; overflow:hidden;
}
#card::after{
  content:''; position:absolute; inset:28px;
  border:4px dashed #9B958A; border-radius:36px; pointer-events:none;
}
.scissor{ position:absolute; top:12px; left:56px; background:#FAF7F0; padding:0 12px; color:#9B958A; font-size:30px; }
svg.doll{ width:220px; height:220px; filter:drop-shadow(0 8px 12px rgba(74,74,74,.18)); margin-bottom:8px; }
h1{ font-size:88px; margin:0; line-height:1.1; letter-spacing:.02em; }
h1 .dot{ color:#E45B5B; }
p.tag{ font-size:34px; margin:14px 0 0; text-align:center; line-height:1.5; }
</style>
</head>
<body>
<div id="card">
  <span class="scissor">&#9986;</span>
  <svg class="doll" viewBox="0 0 256 256">
    <g fill="#FFFFFF" stroke="#4A4A4A" stroke-width="6" stroke-linejoin="round">
      <circle cx="128" cy="52" r="34"/>
      <rect x="92"  y="82"  width="72" height="86" rx="30"/>
      <rect x="58"  y="92"  width="26" height="62" rx="13"/>
      <rect x="172" y="92"  width="26" height="62" rx="13"/>
      <rect x="97"  y="158" width="27" height="74" rx="13"/>
      <rect x="132" y="158" width="27" height="74" rx="13"/>
    </g>
  </svg>
  <h1>꼭꼭 숨어라<span class="dot">.</span></h1>
  <p class="tag">사진에서 색을 훔쳐 종이인형을 위장시키고,<br>친구가 그걸 찾아내는 숨바꼭질</p>
</div>
</body></html>
```

- [ ] **Step 2: Chrome으로 렌더링 후 캡처**

1. `mcp__claude-in-chrome__tabs_create_mcp` 호출 → tabId 확보
2. `mcp__claude-in-chrome__navigate`(tabId, url: `file:///<scratchpad-path-with-forward-slashes>/og-image.html`)
3. `mcp__claude-in-chrome__resize_window`(tabId, width: 1300, height: 900) — `#card`가 1200×630로 스크롤 없이 뷰포트에 온전히 들어오도록 여유 있게
4. `mcp__claude-in-chrome__computer`(action: `wait`, duration: 1, tabId) — Gaegu 웹폰트 로드 대기
5. `mcp__claude-in-chrome__computer`(action: `zoom`, region: [0,0,1200,630], tabId, save_to_disk: true) — 반환된 저장 경로를 `$rawCapturePath`로 기록

Expected: 저장된 이미지에 도화지 배경, 가위 점선 테두리, 인형 마크, 타이틀, 태그라인이 잘리지 않고 모두 보임 (Step 2 결과 스크린샷을 눈으로 확인).

- [ ] **Step 3: 정확히 1200×630으로 리샘플링**

Run (Step 2에서 반환된 실제 경로로 `$rawCapturePath` 대체):

```powershell
Add-Type -AssemblyName System.Drawing
$src = [System.Drawing.Image]::FromFile("$rawCapturePath")
$target = New-Object System.Drawing.Bitmap(1200, 630)
$g = [System.Drawing.Graphics]::FromImage($target)
$g.InterpolationMode = [System.Drawing.Drawing2D.InterpolationMode]::HighQualityBicubic
$g.DrawImage($src, 0, 0, 1200, 630)
$target.Save("C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\og-image.png", [System.Drawing.Imaging.ImageFormat]::Png)
$g.Dispose(); $src.Dispose(); $target.Dispose()
```

이 리샘플링 단계는 Chrome 캡처가 화면 DPI 배율(예: 2배 HiDPI)로 인해 1200×630이 아닌 다른 원본 해상도로 저장되더라도, 최종 파일은 항상 정확히 1200×630이 되도록 보장한다.

- [ ] **Step 4: 픽셀 치수 검증**

Run:
```powershell
Add-Type -AssemblyName System.Drawing
$img = [System.Drawing.Image]::FromFile("C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\og-image.png")
"$($img.Width)x$($img.Height)"
$img.Dispose()
```
Expected: `1200x630`

- [ ] **Step 5: 육안 확인**

Read 도구로 `og-image.png`를 열어 텍스트 잘림, 폰트 미적용(폴백 폰트로 보이는지), 점선 테두리 위치 등을 확인.

- [ ] **Step 6: Commit**

```bash
git add og-image.png
git commit -m "og-image.png 생성 — 1200x630 오픈그래프 미리보기 카드"
```

---

### Task 4: `index.html`에 파비콘·OG 메타 태그 연결

**Files:**
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html:6` (`<title>` 태그 바로 다음)

**Interfaces:**
- Consumes: Task 1~3에서 생성된 `favicon.svg`, `favicon-32.png`, `favicon-16.png`, `apple-touch-icon.png`, `og-image.png` (모두 `index.html`과 같은 경로)
- Produces: 없음 (최종 통합 단계)

- [ ] **Step 1: `<head>`에 태그 삽입**

`index.html`의 `<title>꼭꼭 숨어라 — 사진 속 종이인형 숨바꼭질</title>` 바로 다음 줄에 삽입:

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

- [ ] **Step 2: 로컬에서 404 여부 확인**

`mcp__claude-in-chrome__tabs_create_mcp` → `mcp__claude-in-chrome__navigate`(url: `file:///C:/Users/jeodu/Desktop/VibeCoding/kkokkkok-sumeora/index.html`) → `mcp__claude-in-chrome__read_console_messages`(pattern: `favicon|apple-touch|404|ERR_FILE`)

Expected: `favicon.svg`, `favicon-32.png`, `favicon-16.png`, `apple-touch-icon.png` 관련 404/로드 실패 로그 없음. (`og-image.png`는 페이지가 직접 로드하는 리소스가 아니라 SNS 크롤러 전용이므로 로컬 콘솔에는 애초에 나타나지 않음 — 정상)

- [ ] **Step 3: 브라우저 탭에서 아이콘 육안 확인**

`mcp__claude-in-chrome__computer`(action: `screenshot`)로 탭 자체를 캡처해 탭 아이콘이 표시되는지 확인 (브라우저 UI 크롬이 스크린샷에 포함되는 경우).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "index.html에 파비콘·오픈그래프·트위터 카드 메타 태그 연결"
```

---

## Self-Review 결과
- **스펙 커버리지**: 설계 문서의 에셋 표(favicon.svg/32/16/apple-touch-icon/og-image) 전부 Task 1~3에 대응, `<head>` 변경사항 전체 Task 4에 대응. 배포 후 실제 SNS 미리보기 검증은 설계 문서에서도 명시적으로 범위 밖으로 뒀으므로 이 플랜에도 포함하지 않음.
- **플레이스홀더 스캔**: 없음 — 모든 코드/명령 블록에 실제 값 포함.
- **타입/네이밍 일관성**: 파일명(`favicon.svg`, `favicon-32.png`, `favicon-16.png`, `apple-touch-icon.png`, `og-image.png`)이 설계 문서·모든 태스크에서 동일하게 사용됨. 좌표계(256 viewBox, 동일 6개 도형)가 Task 1/2/3에서 일관됨.
