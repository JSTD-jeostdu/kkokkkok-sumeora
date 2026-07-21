# 인형 외곽선 제거 + 커스텀 컬러피커 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** (1) 게임플레이용 인형 렌더링에서 외곽선을 제거해 위장 효과를 높이고, (2) 네이티브 `<input type="color">`를 포토샵 스타일 SV박스+Hue막대+HEX입력 커스텀 컬러피커로 완전히 대체한다.

**Architecture:** 외곽선 제거는 `renderDollComposite()`에서 stroke 호출 4줄을 삭제하는 단일 지점 수정. 컬러피커는 순수 JS HSV↔RGB↔HEX 변환 함수 + `<canvas>` 기반 SV박스 + CSS 그라디언트 Hue막대로 구성된 새 모달을 만들어, 기존 `#swatch`를 트리거로 삼아 열고 "확인" 시 `state.color`에 반영한 뒤 기존 네이티브 `<input type="color">`와 관련 코드를 제거한다.

**Tech Stack:** 순수 HTML/CSS/JS (`index.html` 단일 파일), 빌드 도구·프레임워크·외부 JS 라이브러리 없음.

## Global Constraints

- 단일 파일 `index.html`만 수정 — 새 파일 생성 금지
- 주석은 한국어
- 탭 영역(버튼) 44px 이상 유지 (SV박스/Hue막대는 드래그형 컨트롤이라 확대 모달의 색칠 캔버스와 같은 예외 범주 — 인디케이터 두께로 보완)
- `prefers-reduced-motion` 존중 (이 작업들은 애니메이션을 추가하지 않아 자동으로 충족됨)
- 외부 CDN·라이브러리 추가 없음 — 색 변환은 직접 구현
- `CONFIG` 객체: 컬러피커 작업은 새 상수를 추가하지 않음. 외곽선 제거 작업은 더 이상 쓰이지 않는 `OUTLINE_COLOR`/`OUTLINE_WIDTH`를 제거함
- **프로젝트에 자동 테스트 프레임워크가 없음** — 검증은 `.superpowers/sdd/qa/drive.js`(git-ignore된 로컬 스크래치 디렉터리, 이전 세션에서 구축한 Playwright+Chromium 기반 QA 드라이버, 즉시 실행 가능한 상태)를 사용한다. 이미 `task1`/`task2`/`task3`(확대 모달용) 시나리오가 있으니 그 패턴을 참고해 새 시나리오(`outline`, `colorpicker`)를 추가한다. 만약 `.superpowers/sdd/qa/`가 없다면(새 워크스페이스 등) `npx playwright install chromium` 후 동일한 구조로 새로 만든다 — `node_modules`/`playwright`가 로컬 설치되어 있어야 함
- QA 스크린샷은 Read 도구로 직접 열어 눈으로 확인한다(콘솔 출력만 믿지 않는다)
- **이 플랜의 각 태스크는 앞선 태스크가 이미 파일을 수정한 상태를 전제로 한다** — Task 1이 줄 수를 줄이고, Task 2가 늘리는 식으로 뒤 태스크로 갈수록 실제 줄 번호는 이 문서에 적힌 숫자와 달라진다. 각 Step에 적힌 줄 번호는 "이 플랜을 작성한 시점" 기준이므로, 실제 편집 시에는 반드시 Step에 함께 적힌 주변 코드(앵커 텍스트 — 예: 특정 함수명, 이전 줄의 정확한 코드)를 파일에서 검색해 그 위치를 찾아 편집한다. 줄 번호는 대략적인 위치 참고용으로만 쓴다

참고 스펙:
- `docs/superpowers/specs/2026-07-21-remove-doll-outline-design.md`
- `docs/superpowers/specs/2026-07-21-color-picker-design.md`

---

### Task 1: 인형 외곽선 제거

**Files:**
- Modify: `index.html:668-680` (`renderDollComposite()` — 외곽선 stroke 제거)
- Modify: `index.html:537-539` (`CONFIG` — 미사용 상수 제거)

**Interfaces:**
- Consumes: 없음 (기존 함수 내부 수정)
- Produces: 없음 (다른 태스크가 의존하지 않음)

- [ ] **Step 1: `renderDollComposite()`에서 외곽선 제거**

`index.html:668-680`의 현재 코드:
```js
function renderDollComposite(){
  compCtx.clearRect(0, 0, CONFIG.DOLL_SIZE, CONFIG.DOLL_SIZE);
  compCtx.save();
  compCtx.fillStyle = '#FFFFFF';
  compCtx.fill(dollPath);
  compCtx.clip(dollPath);
  compCtx.drawImage(paintCanvas, 0, 0);
  compCtx.restore();
  compCtx.lineWidth = CONFIG.OUTLINE_WIDTH;
  compCtx.strokeStyle = CONFIG.OUTLINE_COLOR;
  compCtx.lineJoin = 'round';
  compCtx.stroke(dollPath);
}
```
아래로 교체:
```js
function renderDollComposite(){
  compCtx.clearRect(0, 0, CONFIG.DOLL_SIZE, CONFIG.DOLL_SIZE);
  compCtx.save();
  compCtx.fillStyle = '#FFFFFF';
  compCtx.fill(dollPath);
  compCtx.clip(dollPath);
  compCtx.drawImage(paintCanvas, 0, 0);
  compCtx.restore();
}
```

- [ ] **Step 2: `CONFIG`에서 미사용 상수 제거**

`index.html:537-539`의 현재 코드:
```js
  DEFAULT_COLOR: '#E45B5B',
  OUTLINE_COLOR: 'rgba(74,74,74,.85)',
  OUTLINE_WIDTH: 2.5,
```
아래로 교체:
```js
  DEFAULT_COLOR: '#E45B5B',
```

- [ ] **Step 3: 다른 곳에서 `OUTLINE_COLOR`/`OUTLINE_WIDTH`를 쓰지 않는지 확인**

```bash
grep -n "OUTLINE_COLOR\|OUTLINE_WIDTH" index.html
```
Expected: 결과 없음(0 matches). 있다면 그 사용처도 함께 검토 필요 — 이 플랜 작성 시점 기준으로는 `renderDollComposite()`에서만 쓰였음을 확인했음.

- [ ] **Step 4: 브라우저 QA — 외곽선이 실제로 사라졌는지 확인**

`.superpowers/sdd/qa/drive.js`의 기존 시나리오 구조(특히 `task1`의 `toPaintScreen(page)` 헬퍼)를 참고해 새 시나리오 `outline`을 추가한다:

```js
    } else if (scenario === 'outline'){
      await toPaintScreen(page);
      // 인형 전체를 굵은 붓으로 채워 칠한다(테두리가 남아있다면 도화지색과 대비되어 뚜렷이 보임)
      await page.click('.brush-btn[data-size="24"]');
      const canvas = page.locator('#dollView');
      const box = await canvas.boundingBox();
      const cx = box.x + box.width / 2, cy = box.y + box.height / 2;
      for (let dy = -60; dy <= 60; dy += 15){
        await page.mouse.move(cx - 50, cy + dy);
        await page.mouse.down();
        await page.mouse.move(cx + 50, cy + dy, { steps: 4 });
        await page.mouse.up();
      }
      await page.screenshot({ path: shotPath('outline-after-fill') });
    }
```
(이 블록을 기존 `if/else if` 체인의 마지막 분기 앞, `task3` 블록 다음에 추가)

실행:
```bash
cd .superpowers/sdd/qa && node drive.js outline
```
`outline-after-fill.png`을 Read 도구로 열어 인형 실루엣 가장자리에 진한 회색 테두리 선이 없는지 눈으로 확인한다(색칠한 색이 실루엣 경계까지 매끄럽게 이어지고, 별도의 어두운 윤곽선이 보이지 않아야 함).

이어서 `node drive.js task1`과 `node drive.js task2`를 다시 실행해 확대 모달 기능에 회귀가 없는지 확인한다(`btn-zoom-paint exists: true`, `canvas returned to paint-row: true`, `CONSOLE_ERRORS: none` 등 기존과 동일한 출력이 나와야 함).

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
게임플레이용 인형 렌더링에서 외곽선 제거

외곽선이 위장 효과를 떨어뜨려(색을 맞춰도 실루엣 가장자리가 뚜렷하게
드러남) renderDollComposite()의 stroke 호출을 제거. 반짝 힌트와
공유 카드의 별도 테두리는 의도된 연출이라 그대로 둔다.
EOF
)"
```

---

### Task 2: 컬러피커 — 색 변환 유틸 함수 + 마크업/CSS (정적 구조)

**Files:**
- Modify: `index.html:987` (JS — `toHex()` 다음에 HSV↔RGB↔HEX 변환 함수 추가)
- Modify: `index.html:495` (HTML — `#zoom-modal` 다음에 `#colorpicker-modal` 마크업 추가)
- Modify: `index.html:195` (CSS — `#zoom-tools-slot` 규칙 다음에 컬러피커 스타일 추가)

**Interfaces:**
- Produces: `hsvToRgb(h,s,v)→{r,g,b}`, `rgbToHex(r,g,b)→string`, `hsvToHex(h,s,v)→string`, `hexToRgb(hex)→{r,g,b}|null`, `rgbToHsv(r,g,b)→{h,s,v}` — Task 3이 이 함수들을 사용. HTML 요소 ID: `colorpicker-modal`, `btn-cp-close`, `cp-svbox`(캔버스), `cp-sv-thumb`, `cp-hue-track`, `cp-hue-thumb`, `cp-preview`, `cp-hex`, `btn-cp-confirm` — Task 3·4가 이 ID들을 그대로 사용.

- [ ] **Step 1: 색 변환 함수 추가**

`index.html:987`(`function toHex(n){ return n.toString(16).padStart(2, '0'); }` 다음 줄)에 추가:

```js
/* ---------- 컬러피커: HSV<->RGB<->HEX 변환 ---------- */
function hsvToRgb(h, s, v){
  s /= 100; v /= 100;
  const c = v * s, x = c * (1 - Math.abs((h / 60) % 2 - 1)), m = v - c;
  let r, g, b;
  if (h < 60) [r, g, b] = [c, x, 0];
  else if (h < 120) [r, g, b] = [x, c, 0];
  else if (h < 180) [r, g, b] = [0, c, x];
  else if (h < 240) [r, g, b] = [0, x, c];
  else if (h < 300) [r, g, b] = [x, 0, c];
  else [r, g, b] = [c, 0, x];
  return { r: Math.round((r + m) * 255), g: Math.round((g + m) * 255), b: Math.round((b + m) * 255) };
}
function rgbToHex(r, g, b){ return '#' + [r, g, b].map((n) => toHex(n)).join(''); }
function hsvToHex(h, s, v){ const { r, g, b } = hsvToRgb(h, s, v); return rgbToHex(r, g, b); }
function hexToRgb(hex){
  const m = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
  return m ? { r: parseInt(m[1], 16), g: parseInt(m[2], 16), b: parseInt(m[3], 16) } : null;
}
function rgbToHsv(r, g, b){
  r /= 255; g /= 255; b /= 255;
  const max = Math.max(r, g, b), min = Math.min(r, g, b), d = max - min;
  let h = 0;
  if (d !== 0){
    if (max === r) h = 60 * (((g - b) / d) % 6);
    else if (max === g) h = 60 * ((b - r) / d + 2);
    else h = 60 * ((r - g) / d + 4);
  }
  if (h < 0) h += 360;
  return { h, s: max === 0 ? 0 : (d / max) * 100, v: max * 100 };
}
```

- [ ] **Step 2: 모달 마크업 추가**

`index.html:495`(`</div>` — `#zoom-modal`의 닫는 태그 — 다음 줄, `</main>` 앞)에 추가:

```html
  <!-- 컬러피커 모달: SV박스 + Hue막대 + HEX 입력 -->
  <div id="colorpicker-modal" class="cp-modal hidden" role="dialog" aria-modal="true" aria-label="색상 선택">
    <div class="cp-card">
      <div class="cp-header">
        <span>색상 선택</span>
        <button id="btn-cp-close" aria-label="색상 선택 닫기">✕</button>
      </div>
      <div class="cp-svbox-wrap">
        <canvas id="cp-svbox" width="240" height="240" aria-label="채도·명도 선택"></canvas>
        <div id="cp-sv-thumb"></div>
      </div>
      <div id="cp-hue-track" aria-label="색상(Hue) 선택">
        <div id="cp-hue-thumb"></div>
      </div>
      <div class="cp-preview-row">
        <div id="cp-preview"></div>
        <input type="text" id="cp-hex" maxlength="7" aria-label="HEX 코드 입력" placeholder="#RRGGBB">
      </div>
      <button class="btn btn-primary btn-wide" id="btn-cp-confirm">확인</button>
    </div>
  </div>
```

- [ ] **Step 3: CSS 추가**

`index.html:195`(`#zoom-tools-slot{ width:100%; max-width:360px; }` 다음 줄, `.panel-actions{...}` 앞)에 추가:

```css
/* ---------- 컬러피커 모달 ---------- */
.cp-modal{
  position:fixed; inset:0; z-index:60;
  background:rgba(74,74,74,.45);
  display:flex; align-items:center; justify-content:center; padding:16px;
}
.cp-card{
  background:var(--paper); border:2.5px dashed var(--pencil-soft); border-radius:18px;
  box-shadow:var(--shadow); padding:16px; width:100%; max-width:300px;
}
.cp-header{ display:flex; justify-content:space-between; align-items:center; margin-bottom:10px; font-size:19px; font-weight:700; }
.cp-header button{
  font-family:inherit; font-size:18px; border:2px solid var(--pencil); border-radius:50%;
  width:36px; height:36px; background:#fff; cursor:pointer;
}
.cp-header button:focus-visible{ outline:3px solid var(--accent); outline-offset:2px; }
.cp-svbox-wrap{ position:relative; width:100%; aspect-ratio:1; touch-action:none; }
#cp-svbox{ width:100%; height:100%; display:block; border-radius:10px; }
#cp-sv-thumb{
  position:absolute; width:16px; height:16px; border-radius:50%;
  border:3px solid #fff; box-shadow:0 0 0 1.5px var(--pencil), var(--shadow);
  transform:translate(-50%,-50%); pointer-events:none;
}
#cp-hue-track{
  position:relative; height:28px; border-radius:14px; margin:14px 0 10px;
  background:linear-gradient(to right, #f00 0%, #ff0 17%, #0f0 33%, #0ff 50%, #00f 67%, #f0f 83%, #f00 100%);
  touch-action:none;
}
#cp-hue-thumb{
  position:absolute; top:-3px; width:6px; height:34px; border-radius:3px;
  background:#fff; border:2px solid var(--pencil); box-shadow:var(--shadow);
  transform:translateX(-3px); pointer-events:none;
}
.cp-preview-row{ display:flex; gap:10px; align-items:center; margin-bottom:12px; }
#cp-preview{
  width:44px; height:44px; border-radius:50%; border:3px solid #fff;
  box-shadow:0 0 0 2px var(--pencil), var(--shadow); flex:0 0 auto;
}
#cp-hex{
  flex:1; min-width:0; font-family:inherit; font-size:18px; text-transform:uppercase;
  border:2px solid var(--pencil-soft); border-radius:10px; padding:8px 10px; background:#fff; color:var(--pencil);
}
#cp-hex:focus-visible{ outline:3px solid var(--accent); outline-offset:2px; }
#swatch{ cursor:pointer; }
```
(마지막 `#swatch{ cursor:pointer; }`는 기존 `#swatch{...}` 규칙(`index.html:174-175`)과는 별개의 새 규칙으로 추가 — 기존 규칙을 수정하지 않고 뒤에 추가해 클릭 가능함을 시각적으로 알림)

- [ ] **Step 4: 브라우저 QA — 정적 구조 확인 + 변환 함수 정확성 확인**

`node drive.js task1`을 먼저 실행해 색칠 화면까지 진입하는 기존 흐름이 깨지지 않았는지 확인(간접 확인 — 이 태스크는 아직 아무 것도 지우거나 바꾸지 않았으므로 100% 그대로 통과해야 함).

새 시나리오 `colorpicker-static`을 `drive.js`에 추가해 정적 렌더링과 변환 함수를 확인한다:

```js
    } else if (scenario === 'colorpicker-static'){
      await toPaintScreen(page);
      const checks = await page.evaluate(() => {
        const out = {};
        out.hsvToRgb_red = JSON.stringify(hsvToRgb(0, 100, 100));       // {r:255,g:0,b:0}
        out.rgbToHex_red = rgbToHex(255, 0, 0);                          // #ff0000
        out.hexToRgb_accent = JSON.stringify(hexToRgb('#E45B5B'));       // {r:228,g:91,b:91}
        const hsv = rgbToHsv(228, 91, 91);
        out.rgbToHsv_accent = JSON.stringify({ h: Math.round(hsv.h), s: Math.round(hsv.s), v: Math.round(hsv.v) });
        out.roundtrip = hsvToHex(210, 60, 75) === rgbToHex(hsvToRgb(210, 60, 75).r, hsvToRgb(210, 60, 75).g, hsvToRgb(210, 60, 75).b);
        return out;
      });
      console.log('conversion checks:', JSON.stringify(checks, null, 2));
      await page.evaluate(() => document.getElementById('colorpicker-modal').classList.remove('hidden'));
      await page.screenshot({ path: shotPath('colorpicker-static-open') });
      await page.evaluate(() => document.getElementById('colorpicker-modal').classList.add('hidden'));
    }
```

실행:
```bash
cd .superpowers/sdd/qa && node drive.js colorpicker-static
```
Expected 출력값: `hsvToRgb_red` → `{"r":255,"g":0,"b":0}`, `rgbToHex_red` → `#ff0000`, `hexToRgb_accent` → `{"r":228,"g":91,"b":91}`, `rgbToHsv_accent` → `{"h":0,"s":60,"v":89}`(반올림 오차로 h는 0 근처, s/v는 근사값이면 됨 — 정확히 이 숫자가 아니어도 상식적인 범위면 통과), `roundtrip` → `true`.

`colorpicker-static-open.png`을 Read로 열어: 모달이 화면 중앙에 카드 형태로 뜨고, SV박스(아직 아무 것도 안 그려서 흰 배경), Hue막대(무지개 그라디언트는 보임 — CSS 배경이라 이미 렌더링됨), 미리보기 동그라미, HEX 입력칸, 확인 버튼이 레이아웃대로 배치되어 있는지 확인. SV박스가 비어있는 건 정상(Task 3에서 채워짐).

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
컬러피커 색 변환 유틸 함수와 모달 정적 마크업·CSS 추가

HSV<->RGB<->HEX 변환 함수(외부 라이브러리 없이 직접 구현)와 SV박스+
Hue막대+HEX입력으로 구성된 모달의 뼈대를 추가. 아직 인터랙션은 없음.
EOF
)"
```

---

### Task 3: 컬러피커 — SV박스/Hue막대 렌더링 + 드래그 인터랙션 + 스와치 트리거

**Files:**
- Modify: `index.html:1647` (JS — 도구 버튼 배선 블록 끝, 색칠 확대 모달 섹션 앞에 새 섹션 추가)

**Interfaces:**
- Consumes: Task 2가 만든 `hsvToRgb`/`rgbToHex`/`hsvToHex`/`hexToRgb`/`rgbToHsv`, HTML 요소 ID들, 기존 전역 `$(id)`·`state`·`swatch` 상수
- Produces: `cpState`(`{h,s,v}`), `openColorPicker()`, `renderSVBox(hue)`, `updateThumbs()`, `updatePreview()` — Task 4가 이들을 확장/사용

- [ ] **Step 1: 컬러피커 상태·렌더링·드래그 로직 추가**

`index.html:1647`(`$('btn-eyedropper').addEventListener(...)` 블록이 끝나는 지점, `/* ---------- 색칠 확대 모달 ---------- */` 주석 앞)에 추가:

```js

/* ---------- 컬러피커 ---------- */
const cpState = { h: 0, s: 0, v: 0 };
const cpSvBox = $('cp-svbox'), cpSvCtx = cpSvBox.getContext('2d');
const cpSvThumb = $('cp-sv-thumb'), cpSvWrap = document.querySelector('.cp-svbox-wrap');
const cpHueTrack = $('cp-hue-track'), cpHueThumb = $('cp-hue-thumb');
const cpPreview = $('cp-preview'), cpHex = $('cp-hex');
const colorpickerModal = $('colorpicker-modal');

function renderSVBox(hue){
  const w = cpSvBox.width, h = cpSvBox.height;
  cpSvCtx.fillStyle = hsvToHex(hue, 100, 100);
  cpSvCtx.fillRect(0, 0, w, h);
  const gWhite = cpSvCtx.createLinearGradient(0, 0, w, 0);
  gWhite.addColorStop(0, 'rgba(255,255,255,1)');
  gWhite.addColorStop(1, 'rgba(255,255,255,0)');
  cpSvCtx.fillStyle = gWhite;
  cpSvCtx.fillRect(0, 0, w, h);
  const gBlack = cpSvCtx.createLinearGradient(0, 0, 0, h);
  gBlack.addColorStop(0, 'rgba(0,0,0,0)');
  gBlack.addColorStop(1, 'rgba(0,0,0,1)');
  cpSvCtx.fillStyle = gBlack;
  cpSvCtx.fillRect(0, 0, w, h);
}
function updateThumbs(){
  cpSvThumb.style.left = (cpState.s / 100 * 100) + '%';
  cpSvThumb.style.top = ((1 - cpState.v / 100) * 100) + '%';
  cpHueThumb.style.left = (cpState.h / 360 * 100) + '%';
}
function updatePreview(){
  const hex = hsvToHex(cpState.h, cpState.s, cpState.v);
  cpPreview.style.background = hex;
  if (document.activeElement !== cpHex) cpHex.value = hex;
}
function cpSvPosToState(e){
  const r = cpSvWrap.getBoundingClientRect();
  const x = Math.min(1, Math.max(0, (e.clientX - r.left) / r.width));
  const y = Math.min(1, Math.max(0, (e.clientY - r.top) / r.height));
  cpState.s = x * 100;
  cpState.v = (1 - y) * 100;
}
function cpHuePosToState(e){
  const r = cpHueTrack.getBoundingClientRect();
  const x = Math.min(1, Math.max(0, (e.clientX - r.left) / r.width));
  cpState.h = x * 360;
}
let cpSvDragging = false, cpHueDragging = false;
cpSvWrap.addEventListener('pointerdown', (e) => {
  cpSvDragging = true;
  cpSvWrap.setPointerCapture(e.pointerId);
  cpSvPosToState(e); updateThumbs(); updatePreview();
});
cpSvWrap.addEventListener('pointermove', (e) => {
  if (!cpSvDragging) return;
  cpSvPosToState(e); updateThumbs(); updatePreview();
});
cpSvWrap.addEventListener('pointerup', () => { cpSvDragging = false; });
cpSvWrap.addEventListener('pointercancel', () => { cpSvDragging = false; });
cpHueTrack.addEventListener('pointerdown', (e) => {
  cpHueDragging = true;
  cpHueTrack.setPointerCapture(e.pointerId);
  cpHuePosToState(e); renderSVBox(cpState.h); updateThumbs(); updatePreview();
});
cpHueTrack.addEventListener('pointermove', (e) => {
  if (!cpHueDragging) return;
  cpHuePosToState(e); renderSVBox(cpState.h); updateThumbs(); updatePreview();
});
cpHueTrack.addEventListener('pointerup', () => { cpHueDragging = false; });
cpHueTrack.addEventListener('pointercancel', () => { cpHueDragging = false; });

function openColorPicker(){
  const rgb = hexToRgb(state.color) || { r: 228, g: 91, b: 91 };
  const hsv = rgbToHsv(rgb.r, rgb.g, rgb.b);
  cpState.h = hsv.h; cpState.s = hsv.s; cpState.v = hsv.v;
  renderSVBox(cpState.h);
  updateThumbs();
  updatePreview();
  colorpickerModal.classList.remove('hidden');
}
swatch.setAttribute('role', 'button');
swatch.setAttribute('tabindex', '0');
swatch.setAttribute('aria-label', '색상 선택 열기');
swatch.addEventListener('click', openColorPicker);
swatch.addEventListener('keydown', (e) => {
  if (e.key === 'Enter' || e.key === ' '){ e.preventDefault(); openColorPicker(); }
});
```

이 시점에는 아직 "확인" 버튼(`#btn-cp-confirm`)과 닫기(`#btn-cp-close`)에 이벤트가 연결되지 않았고, `state.color`에도 반영되지 않는다 — Task 4에서 처리.

- [ ] **Step 2: 브라우저 QA — 드래그로 색이 실제로 바뀌는지 확인**

`drive.js`에 새 시나리오 `colorpicker-drag` 추가:

```js
    } else if (scenario === 'colorpicker-drag'){
      await toPaintScreen(page);
      await page.click('#swatch');
      await page.waitForSelector('#colorpicker-modal:not(.hidden)', { timeout: 5000 });
      await page.screenshot({ path: shotPath('colorpicker-opened') });
      // Hue막대를 파란색 쪽(오른쪽에서 1/3 지점, h≈240)으로 드래그
      const hueBox = await page.locator('#cp-hue-track').boundingBox();
      await page.mouse.move(hueBox.x + hueBox.width * 0.67, hueBox.y + hueBox.height / 2);
      await page.mouse.down();
      await page.mouse.up();
      await page.screenshot({ path: shotPath('colorpicker-after-hue-drag') });
      // SV박스를 좌상단(흰색 쪽)으로 드래그
      const svBox = await page.locator('.cp-svbox-wrap').boundingBox();
      await page.mouse.move(svBox.x + 10, svBox.y + 10);
      await page.mouse.down();
      await page.mouse.up();
      await page.screenshot({ path: shotPath('colorpicker-after-sv-drag') });
      const preview = await page.evaluate(() => document.getElementById('cp-preview').style.background);
      console.log('preview after drag:', preview);
      const hexVal = await page.evaluate(() => document.getElementById('cp-hex').value);
      console.log('hex input after drag:', hexVal);
    }
```

실행: `node drive.js colorpicker-drag`

확인 사항:
- `colorpicker-opened.png`: 스와치 탭으로 모달이 정상적으로 열림
- `colorpicker-after-hue-drag.png`: SV박스의 기준색이 파란 계열로 바뀌어 있어야 함(오른쪽 위가 파란색)
- `colorpicker-after-sv-drag.png`: 미리보기 동그라미가 흰색에 가까운 옅은 색으로 바뀌어 있어야 함(좌상단=채도 낮음+명도 높음)
- 콘솔 출력 `preview after drag`/`hex input after drag`가 서로 상응하는 값인지(둘 다 같은 옅은 색 계열의 hex) 확인

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
컬러피커 SV박스/Hue막대 렌더링과 드래그 인터랙션 추가

스와치를 탭하면 모달이 열리고, SV박스·Hue막대를 드래그하면 실시간으로
미리보기와 HEX 입력칸이 갱신된다. 아직 실제 붓 색상(state.color)에는
반영되지 않음 — 확인 버튼 연결은 다음 태스크에서 처리.
EOF
)"
```

---

### Task 4: 컬러피커 — HEX 입력 동기화 + 확인/닫기 + 기존 네이티브 피커 제거

**Files:**
- Modify: `index.html:1647`(Task 3에서 추가한 섹션 끝에 이어서) — HEX 입력 리스너, 닫기·확인 로직 추가
- Modify: `index.html:344` (HTML — 네이티브 `<input type="color">` 제거)
- Modify: `index.html:172-173` (CSS — `#color-picker{...}` 규칙 제거)
- Modify: `index.html:1626-1631` (JS — 네이티브 `#color-picker` `input` 리스너 제거)
- Modify: `index.html:916` (JS — `pickColorFromCrop()`의 죽은 참조 제거)

**Interfaces:**
- Consumes: Task 3의 `cpState`, `renderSVBox`, `updateThumbs`, `cpHex`, `cpPreview`, `colorpickerModal`, `state.color`
- Produces: 없음 (기능 완성 지점)

- [ ] **Step 1: HEX 입력 동기화 + 닫기 + 확인 로직 추가**

Task 3에서 추가한 컬러피커 섹션의 마지막 줄(`swatch.addEventListener('keydown', ...)` 블록) 다음에 이어서 추가:

```js
cpHex.addEventListener('input', () => {
  const rgb = hexToRgb(cpHex.value);
  if (!rgb) return; // 완성되지 않은 입력은 무시
  const hsv = rgbToHsv(rgb.r, rgb.g, rgb.b);
  cpState.h = hsv.h; cpState.s = hsv.s; cpState.v = hsv.v;
  renderSVBox(cpState.h);
  updateThumbs();
  cpPreview.style.background = hsvToHex(cpState.h, cpState.s, cpState.v);
});
function closeColorPicker(){
  colorpickerModal.classList.add('hidden');
}
$('btn-cp-close').addEventListener('click', closeColorPicker);
colorpickerModal.addEventListener('click', (e) => {
  if (e.target === colorpickerModal) closeColorPicker();
});
$('btn-cp-confirm').addEventListener('click', () => {
  state.color = hsvToHex(cpState.h, cpState.s, cpState.v);
  state.eraser = false;
  $('btn-eraser').setAttribute('aria-pressed', 'false');
  swatch.style.background = state.color;
  closeColorPicker();
});
```

- [ ] **Step 2: 네이티브 `<input type="color">` 제거**

`index.html:344`의 다음 줄을 삭제:
```html
            <input type="color" id="color-picker" value="#E45B5B" aria-label="색상 선택">
```

- [ ] **Step 3: 관련 CSS 규칙 제거**

`index.html:172-173`의 다음 두 줄을 삭제:
```css
#color-picker{ width:40px; height:40px; padding:2px; border:2px solid var(--pencil);
  border-radius:10px; background:#fff; cursor:pointer; flex:0 0 auto; }
```

- [ ] **Step 4: 네이티브 입력 리스너 제거**

`index.html:1626-1631`의 다음 블록을 삭제:
```js
$('color-picker').addEventListener('input', (e) => {
  state.color = e.target.value;
  state.eraser = false;
  $('btn-eraser').setAttribute('aria-pressed', 'false');
  swatch.style.background = state.color;
});
```

- [ ] **Step 5: `pickColorFromCrop()`의 죽은 참조 제거**

`index.html:916`의 다음 줄을 삭제(네이티브 `#color-picker`가 더 이상 없으므로 `$('color-picker')`가 `null`을 반환해 `.value = ...`에서 `TypeError`가 발생함 — 스포이드 기능이 깨지는 실제 버그이므로 반드시 제거):
```js
  $('color-picker').value = state.color;
```

- [ ] **Step 6: `grep`으로 네이티브 피커 잔여 참조가 없는지 확인**

```bash
grep -n "color-picker" index.html
```
Expected: 결과 없음(0 matches)

- [ ] **Step 7: 브라우저 QA — 전체 흐름 + 회귀 확인**

`drive.js`에 새 시나리오 `colorpicker-full` 추가:

```js
    } else if (scenario === 'colorpicker-full'){
      await toPaintScreen(page);
      await page.click('#swatch');
      await page.waitForSelector('#colorpicker-modal:not(.hidden)', { timeout: 5000 });
      // HEX 입력으로 직접 진한 파란색 지정
      await page.fill('#cp-hex', '#2244AA');
      await page.locator('#cp-hex').dispatchEvent('input');
      await page.screenshot({ path: shotPath('colorpicker-hex-typed') });
      await page.click('#btn-cp-confirm');
      await page.waitForSelector('#colorpicker-modal.hidden', { state: 'attached', timeout: 5000 });
      const swatchBg = await page.evaluate(() => getComputedStyle(document.getElementById('swatch')).backgroundColor);
      console.log('swatch bg after confirm:', swatchBg); // rgb(34, 68, 170) 근처여야 함
      // 이 색으로 실제로 칠해지는지 확인
      const canvas = page.locator('#dollView');
      const box = await canvas.boundingBox();
      const cx = box.x + box.width / 2, cy = box.y + box.height / 2;
      await page.mouse.move(cx - 15, cy);
      await page.mouse.down();
      await page.mouse.move(cx + 15, cy, { steps: 4 });
      await page.mouse.up();
      await page.screenshot({ path: shotPath('colorpicker-stroke-with-picked-color') });
      // 배경 탭으로 닫히는지(취소) 확인 — 다시 열고 색을 바꾼 뒤 배경 탭
      await page.click('#swatch');
      await page.waitForSelector('#colorpicker-modal:not(.hidden)', { timeout: 5000 });
      await page.mouse.click(20, 20); // 모달 바깥(배경) 클릭
      await page.waitForSelector('#colorpicker-modal.hidden', { state: 'attached', timeout: 5000 });
      console.log('closed via backdrop click: OK');
      // 스포이드가 여전히 정상 동작하는지(죽은 참조 제거 확인 겸)
      const errors = [];
      page.on('pageerror', (e) => errors.push(e.message));
      await page.click('#btn-eyedropper');
      const photoBox = await page.locator('#photoCanvas').boundingBox().catch(() => null);
      if (photoBox) await page.mouse.click(photoBox.x + 10, photoBox.y + 10);
      console.log('eyedropper errors:', errors.length ? JSON.stringify(errors) : 'none');
    }
```

실행: `node drive.js colorpicker-full`

확인 사항:
- `colorpicker-hex-typed.png`: HEX 입력 후 SV박스/Hue막대 인디케이터가 진한 파란색 위치로 이동, 미리보기도 그 색
- 콘솔 `swatch bg after confirm`이 `rgb(34, 68, 170)` 근처(정확한 반올림 오차는 허용)
- `colorpicker-stroke-with-picked-color.png`: 인형에 그린 획이 그 파란색으로 보임
- `closed via backdrop click: OK` 출력 확인(취소 동작)
- `eyedropper errors: none` — 스포이드 클릭 시 에러 없음(Step 5에서 제거한 죽은 참조 확인)

마지막으로 회귀 확인 — `node drive.js task1`, `node drive.js task2`, `node drive.js task3`을 순서대로 실행해 확대 모달 기능 전체가 여전히 정상 동작하는지 확인(모두 기존과 동일한 출력이어야 함).

- [ ] **Step 8: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
컬러피커 HEX 동기화·확인/닫기 연결하고 네이티브 피커 완전히 대체

HEX 입력↔SV박스/Hue막대 양방향 동기화, 확인 버튼으로 state.color에
반영, 배경 탭/✕로 취소. 기존 input type=color와 관련 CSS·JS를 모두
제거하고, pickColorFromCrop()에 남아있던 죽은 참조(TypeError 유발)도
함께 정리.
EOF
)"
```
