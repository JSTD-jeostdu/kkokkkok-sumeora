# 스포이드 루페 + 결과 화면 인형 원본 공개 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** (A) 스포이드 사용 시 픽셀 단위 확대 루페로 미리보기 후 확정/취소하는 인터랙션을 추가하고, (B) 플레이 종료 결과 화면에 제작자가 만든 종이인형 원본을 카드로 공개한다.

**Architecture:** 두 기능 모두 `index.html` 단일 파일 안에서 CSS/HTML 마크업 추가 + 기존 함수 재사용(무수정) + 신규 함수 소량 추가로 구현한다. 데이터 모델·Firestore 규칙 변경 없음. 자동 테스트 프레임워크가 없는 순수 정적 페이지이므로, 각 작업의 "테스트"는 브라우저(파일을 직접 열거나 로컬 정적 서버)에서 실제 동작을 확인하거나 devtools 콘솔에서 함수를 직접 호출해 검증하는 방식으로 대체한다.

**Tech Stack:** 순수 HTML/CSS/JS, Canvas 2D API, Firebase compat SDK(CDN, 기능 B 콘솔 검증 단계에선 미사용). 빌드 도구 없음.

## Global Constraints

- 단일 파일 `index.html`만 수정한다. 새 파일 생성 없음.
- 주석은 한국어로 작성한다.
- 외부 의존성 추가 금지(CDN 포함) — 이번 작업은 기존 의존성만 사용.
- 새 상수는 `CONFIG` 객체에 넣지 않아도 되는 로컬 상수(`LOUPE_ZOOM`, `LOUPE_SRC` 등)는 해당 기능 코드 블록 상단에 선언 — 기존 코드도 `DOLL_VIEW_SCALE` 같은 로컬 상수를 `CONFIG` 밖에 두는 패턴을 따른다.
- 터치 영역 44px 이상 원칙은 이번 작업 대상(루페, 인형 카드)에는 클릭 가능한 인터랙티브 요소가 없으므로 해당 없음(루페는 `pointer-events:none`, 인형 카드는 정보 표시 전용).
- 사진과 인형 레이어를 합성(굽기)하지 않는다 — 인형 원본 공개 기능은 이미 분리 저장된 `dollB64` 이미지를 그대로 노출하는 것이라 이 원칙을 위반하지 않는다.
- 공유 카드(`buildShareCard`, `index.html` 내 별도 함수)는 이번 작업에서 손대지 않는다.
- Firestore 문서 스키마·`firestore.rules` 변경 없음.
- 스펙 문서: `docs/superpowers/specs/2026-07-23-loupe-and-doll-reveal-design.md` (모든 세부 설계 근거는 이 문서를 따른다)
- **줄 번호 주의**: 각 Task의 `Files:` 항목에 적힌 줄 번호는 이 계획을 작성한 시점(수정 전) 기준이다. 앞선 Task가 코드를 추가하면 그 아래 모든 줄 번호가 밀린다. 실제 위치는 항상 각 Step에 인용된 **정확한 코드 스니펫**(old_string)으로 파일 안에서 찾아서 수정할 것 — 줄 번호는 대략적인 참고용일 뿐이다.

---

## 기능 A — 스포이드 확대 루페

### Task 1: 루페 DOM/CSS 뼈대 추가

**Files:**
- Modify: `index.html:293` (CSS, `.eyedropper-badge` 규칙 뒤에 추가)
- Modify: `index.html:547` (HTML, `zoom-modal` 블록 뒤에 추가)

**Interfaces:**
- Produces: `#loupe`(엘리먼트), `#loupe-canvas`(112×112 캔버스), `#loupe-swatch`(색 스와치 div), `#loupe-hex`(HEX 텍스트 span) — Task 2에서 JS가 이 id들을 참조한다.

- [ ] **Step 1: CSS 추가**

아래 줄(`index.html:293`)을 찾아:

```css
.eyedropper-badge{ text-align:center; font-size:15px; color:var(--pencil-soft); margin:0 0 6px; }
```

그 바로 뒤에 삽입:

```css
.loupe{ position:fixed; z-index:60; pointer-events:none; transform:translate(-50%,-100%); }
.loupe canvas{ border-radius:50%; border:3px solid var(--pencil); box-shadow:var(--shadow); image-rendering:pixelated; display:block; }
.loupe .swatch-row{ display:flex; align-items:center; justify-content:center; gap:6px; margin-top:4px; }
.loupe .swatch-dot{ width:18px; height:18px; border-radius:50%; border:2px solid #fff; box-shadow:0 0 0 1px var(--pencil-soft); }
.loupe .hex-label{ font-size:13px; color:var(--pencil); background:#fff; padding:1px 6px; border-radius:8px; }
```

- [ ] **Step 2: HTML 추가**

아래 블록(`zoom-modal` 마크업, `index.html:542-547`)을 찾아:

```html
  <div id="zoom-modal" class="zoom-modal hidden" role="dialog" aria-modal="true" aria-label="확대 색칠">
    <button id="btn-zoom-close" aria-label="확대 닫기">✕</button>
    <div id="zoom-canvas-slot"></div>
    <div id="zoom-tools-slot"></div>
  </div>
```

그 바로 뒤에 삽입:

```html
<!-- 스포이드 확대 루페: 색칠 단계에서 스포이드 드래그 중에만 표시 -->
<div id="loupe" class="loupe hidden" aria-hidden="true">
  <canvas id="loupe-canvas" width="112" height="112"></canvas>
  <div class="swatch-row">
    <div class="swatch-dot" id="loupe-swatch"></div>
    <span class="hex-label" id="loupe-hex"></span>
  </div>
</div>
```

- [ ] **Step 3: 브라우저에서 확인**

`index.html`을 브라우저로 열고 devtools 콘솔에서:

```js
document.getElementById('loupe').classList.remove('hidden');
```

실행: 화면 좌상단(기본 `left:0; top:0`이라 화면 모서리에 걸쳐 보임)에 검은 테두리 원과 그 아래 스와치+라벨이 보이면 성공. 확인 후 다시 `classList.add('hidden')`으로 숨긴다.

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "add: 스포이드 루페 DOM/CSS 뼈대 추가"
```

---

### Task 2: 루페 상태 변수 + `updateLoupe()` 렌더링 함수 구현

**Files:**
- Modify: `index.html:1070` (기존 `let eyedropperOn = false;` 다음)

**Interfaces:**
- Consumes: `bgCropCanvas`, `bgCropCtx`(`index.html:731` alias), `CONFIG.DOLL_SIZE`(=256), `toHex()`(`index.html:1181`)
- Produces: `updateLoupe(v, e)` — `v`는 `viewPos(e)` 결과(`{x,y,cssX,cssY}`), `e`는 원본 포인터 이벤트. 호출 시 `lastLoupeView`(전역)에 `v`를 저장하고 `eyedropperLastInside`(전역 불리언)를 갱신한다. Task 3에서 이 세 값을 그대로 사용한다.

- [ ] **Step 1: 상태 변수 + 함수 추가**

`index.html:1070`(`let eyedropperOn = false;` 줄) 바로 뒤에 삽입:

```js
let eyedropperDragging = false;
let eyedropperLastInside = false;
let lastLoupeView = null; // 마지막 포인터 이벤트의 viewPos(e) 결과 — 확정 시 pickColorFromCrop에 그대로 전달
const LOUPE_ZOOM = 5;   // 픽셀 1개를 몇 배로 그릴지
const LOUPE_SRC = 22;   // bgCropCanvas에서 읽어올 정사각형 한 변 길이(px)
const loupeEl = $('loupe'), loupeCanvas = $('loupe-canvas'), loupeCtx = loupeCanvas.getContext('2d');
loupeCtx.imageSmoothingEnabled = false;
/* v: viewPos(e) 결과(dollView 0~256 좌표계). e: 원본 포인터 이벤트(clientX/Y는 뷰포트 절대 좌표 — 루페 위치 계산용). */
function updateLoupe(v, e){
  lastLoupeView = v;
  const { x: vx, y: vy } = v;
  const inside = vx >= 0 && vx < CONFIG.DOLL_SIZE && vy >= 0 && vy < CONFIG.DOLL_SIZE;
  eyedropperLastInside = inside;
  const px = Math.min(CONFIG.DOLL_SIZE - 1, Math.max(0, Math.floor(vx)));
  const py = Math.min(CONFIG.DOLL_SIZE - 1, Math.max(0, Math.floor(vy)));
  const half = LOUPE_SRC / 2;
  const size = loupeCanvas.width; // 112
  loupeCtx.clearRect(0, 0, size, size);
  loupeCtx.save();
  loupeCtx.beginPath(); loupeCtx.arc(size / 2, size / 2, size / 2, 0, Math.PI * 2); loupeCtx.clip();
  loupeCtx.drawImage(bgCropCanvas, px - half, py - half, LOUPE_SRC, LOUPE_SRC, 0, 0, size, size);
  loupeCtx.restore();
  // 중앙 크로스헤어(캔버스 정중앙 1픽셀 셀 강조) — 캔버스 밖이면 빨간색으로 취소 표시
  const cell = size / LOUPE_SRC; // 확대된 픽셀 1개 크기
  loupeCtx.strokeStyle = inside ? '#4A4A4A' : '#E45B5B';
  loupeCtx.lineWidth = 2;
  loupeCtx.strokeRect(size / 2 - cell / 2, size / 2 - cell / 2, cell, cell);
  const d = bgCropCtx.getImageData(px, py, 1, 1).data;
  const hex = '#' + toHex(d[0]) + toHex(d[1]) + toHex(d[2]);
  $('loupe-swatch').style.background = hex;
  $('loupe-hex').textContent = inside ? hex : '취소';
  loupeEl.style.left = e.clientX + 'px';
  loupeEl.style.top = (e.clientY - 90) + 'px';
}
```

- [ ] **Step 2: 브라우저 콘솔에서 함수 단독 호출로 확인**

`index.html`을 열고 제작 화면 → 사진 업로드 → 색칠 단계까지 진입(오프라인 모드로 충분, Firebase 불필요). devtools 콘솔에서:

```js
loupeEl.classList.remove('hidden');
updateLoupe({ x: 128, y: 128 }, { clientX: 300, clientY: 300 });
```

실행 후 화면 `(300, 210)` 부근에 루페가 뜨고, 안에 사진 크롭 중앙 부분이 확대되어 보이고 크로스헤어가 회색이며, 스와치/HEX 라벨이 실제 색으로 채워지는지 확인. 이어서:

```js
updateLoupe({ x: -10, y: 128 }, { clientX: 300, clientY: 300 });
```

크로스헤어가 빨간색으로 바뀌고 라벨이 "취소"로 바뀌는지 확인. 확인 후 `loupeEl.classList.add('hidden')`.

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "add: updateLoupe 렌더링 함수 구현"
```

---

### Task 3: 스포이드 이벤트 리스너를 press-drag-확정 방식으로 교체

**Files:**
- Modify: `index.html:1149-1174` (기존 `dollView`의 `pointerdown`/`pointermove`/`pointerup`/`pointercancel` 리스너 4개 전체 교체)

**Interfaces:**
- Consumes: `updateLoupe(v, e)`(Task 2), `pickColorFromCrop(vx, vy, cssX, cssY)`(`index.html:1103`, 기존 함수 무수정 재사용), `viewPos(e)`(`index.html:1072`, 기존), `endStroke()`(`index.html:1142`, 기존), `eyedropperOn`(`index.html:1070`, 기존 전역), `eyedropperDragging`/`eyedropperLastInside`/`lastLoupeView`/`loupeEl`(Task 2에서 선언)

기존 코드(`index.html:1149-1174`):

```js
dollView.addEventListener('pointerdown', (e) => {
  if (state.phase !== 'paint') return;
  e.preventDefault();
  if (eyedropperOn){
    const v = viewPos(e);
    pickColorFromCrop(v.x, v.y, v.cssX, v.cssY);
    return;
  }
  dollView.setPointerCapture(e.pointerId);
  pushUndo();
  painting = true;
  strokeSetup();
  lastPt = dollPos(e);
  strokeDot(lastPt);
  renderDollView();
});
dollView.addEventListener('pointermove', (e) => {
  if (!painting) return;
  const pt = dollPos(e);
  strokeSegment(lastPt, pt);
  lastPt = pt;
  renderDollView();
});
dollView.addEventListener('pointerup', endStroke);
dollView.addEventListener('pointercancel', endStroke);
```

- [ ] **Step 1: 4개 리스너를 아래 코드로 교체**

```js
dollView.addEventListener('pointerdown', (e) => {
  if (state.phase !== 'paint') return;
  e.preventDefault();
  if (eyedropperOn){
    dollView.setPointerCapture(e.pointerId);
    eyedropperDragging = true;
    loupeEl.classList.remove('hidden');
    updateLoupe(viewPos(e), e);
    return;
  }
  dollView.setPointerCapture(e.pointerId);
  pushUndo();
  painting = true;
  strokeSetup();
  lastPt = dollPos(e);
  strokeDot(lastPt);
  renderDollView();
});
dollView.addEventListener('pointermove', (e) => {
  if (eyedropperDragging){
    updateLoupe(viewPos(e), e);
    return;
  }
  if (!painting) return;
  const pt = dollPos(e);
  strokeSegment(lastPt, pt);
  lastPt = pt;
  renderDollView();
});
function endEyedropperDrag(commit){
  eyedropperDragging = false;
  loupeEl.classList.add('hidden');
  if (commit && eyedropperLastInside){
    const v = lastLoupeView;
    pickColorFromCrop(v.x, v.y, v.cssX, v.cssY);
  }
}
dollView.addEventListener('pointerup', (e) => {
  if (eyedropperDragging){ endEyedropperDrag(true); return; }
  endStroke();
});
dollView.addEventListener('pointercancel', (e) => {
  if (eyedropperDragging){ endEyedropperDrag(false); return; }
  endStroke();
});
```

- [ ] **Step 2: 브라우저 수동 검증 — 캔버스 안에서 확정**

로컬에서 `index.html` 열기(또는 `python -m http.server`로 정적 서빙 후 `localhost`) → 홈 → 만들기 → 사진 업로드 → 색칠 단계 → 스포이드 버튼 켜기(`aria-pressed="true"`가 되는지 확인) → `dollView` 캔버스 위에서 마우스/터치 누른 채 드래그.

기대 결과: 누르는 즉시 루페가 손가락 위쪽에 나타나고, 드래그하는 동안 루페 안 색이 실시간으로 바뀐다. 캔버스 안에서 손을 떼면: 루페가 사라지고, 스와치 색이 확정한 색으로 바뀌고, 스포이드 남은 횟수(`#eyedropper-count`)가 1 줄고, 스포이드 버튼이 자동으로 꺼진다(`aria-pressed="false"`).

- [ ] **Step 3: 브라우저 수동 검증 — 캔버스 밖에서 취소**

스포이드를 다시 켜고, `dollView` 위에서 누른 채 캔버스 바깥까지 드래그한 뒤 그 자리에서 손을 뗀다.

기대 결과: 루페가 사라지지만 스와치 색·남은 횟수는 변화 없고, 스포이드 버튼은 계속 켜진 상태(`aria-pressed="true"`)로 남아 바로 재시도할 수 있다.

- [ ] **Step 4: 붓칠 회귀 확인**

스포이드를 끄고 일반 붓질 → 색칠이 정상 동작하는지, 되돌리기·전체지우기가 그대로 동작하는지 확인(루페 관련 변경이 붓칠 경로를 건드리지 않았는지 재확인).

- [ ] **Step 5: 확대 모달(`btn-zoom-paint`)에서도 동일하게 검증**

색칠 단계에서 "🔍 확대해서 칠하기" 버튼으로 확대 모달을 연 뒤, Step 2~4를 모달 안에서 반복. 동일하게 동작해야 한다(설계상 `dollView` 엘리먼트가 그대로 이동할 뿐이라 별도 처리 없이 동작해야 함).

- [ ] **Step 6: 커밋**

```bash
git add index.html
git commit -m "change: 스포이드를 탭 즉시확정에서 드래그-루페-확정 방식으로 변경"
```

---

## 기능 B — 결과 화면에 제작자 종이인형 원본 공개

### Task 4: 인형 원본 공개 DOM/CSS 추가

**Files:**
- Modify: `index.html` (CSS, Task 1에서 추가한 `.loupe .hex-label{...}` 규칙 뒤에 추가)
- Modify: `index.html:492` (HTML, `playend-panel` 안 `eyedropper-badge` 줄 바로 앞에 삽입)

**Interfaces:**
- Produces: `#doll-reveal-row`(빈 컨테이너 div) — Task 5에서 JS가 이 id로 카드를 채운다.

- [ ] **Step 1: CSS 추가**

Task 1에서 추가한 아래 줄을 찾아:

```css
.loupe .hex-label{ font-size:13px; color:var(--pencil); background:#fff; padding:1px 6px; border-radius:8px; }
```

그 바로 뒤에 삽입:

```css
.doll-reveal-row{ display:flex; gap:10px; overflow-x:auto; padding:6px 2px 10px; }
.doll-reveal-card{ flex:0 0 auto; width:88px; height:88px; background:var(--paper);
  border:2px dashed var(--pencil-soft); border-radius:12px;
  display:flex; align-items:center; justify-content:center; }
.doll-reveal-card img{ width:72px; height:72px; object-fit:contain; }
```

- [ ] **Step 2: HTML 추가**

아래 줄(`index.html:492`)을 찾아:

```html
      <p class="eyedropper-badge hidden" id="eyedropper-badge">🧪 스포이드 사용됨</p>
```

그 바로 앞에 삽입(주의: `result-panel` 안에도 `id="rv-eyedropper-badge"`로 텍스트는 같지만 `id`가 다른 비슷한 줄이 있으니 `id="eyedropper-badge"` 쪽인지 확인할 것):

```html
      <p class="hint-line">제작자가 만든 종이인형</p>
      <div class="doll-reveal-row" id="doll-reveal-row"></div>
```

- [ ] **Step 3: 브라우저에서 확인**

`index.html`을 열고 devtools 콘솔에서:

```js
const row = document.getElementById('doll-reveal-row');
const img = document.createElement('img');
img.src = 'data:image/svg+xml;base64,' + btoa('<svg xmlns="http://www.w3.org/2000/svg" width="72" height="72"><circle cx="36" cy="36" r="30" fill="red"/></svg>');
const card = document.createElement('div'); card.className = 'doll-reveal-card';
card.appendChild(img); row.appendChild(card);
document.getElementById('playend-panel').classList.remove('hidden');
```

빨간 원이 담긴 점선 카드가 도화지색 배경에 표시되면 성공. 확인 후 새로고침(상태 원복).

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "add: 결과 화면 인형 원본 공개 DOM/CSS 뼈대 추가"
```

---

### Task 5: `renderDollReveal()` 구현 + `finishPlay()` 연결

**Files:**
- Modify: `index.html:741` (기존 `function activeEditDoll(){...}` 다음— 새 함수 추가 위치)
- Modify: `index.html:1767-1810` (`finishPlay(cleared)` 함수, `setPhase('end')` 호출 직전에 함수 호출 1줄 추가)

**Interfaces:**
- Consumes: `playStage`(전역, `{dolls}` 또는 `{doll}` 포맷 — `index.html:1690`/`1696`에서 채워짐), `remoteDollImgs`(전역 배열, `index.html:665`), `remoteDollImg`(전역 단일, 레거시 포맷용, `index.html:1675`)
- Produces: `renderDollReveal()` — 인자 없음, `#doll-reveal-row`를 현재 `playStage`의 인형 이미지들로 채운다.

- [ ] **Step 1: `renderDollReveal()` 함수 추가**

`index.html:741`(`function activeEditDoll(){ return state.dolls[state.activeDollIndex]; }` 줄) 바로 뒤에 삽입:

```js
/* 플레이 종료 후 결과 화면에 제작자가 만든 종이인형 원본을 그대로 노출한다.
   dollB64/remoteDollImg(s)는 투명 배경의 인형만 담고 있어(사진과 별도 레이어) 그대로 표시해도 무방하다. */
function renderDollReveal(){
  const imgs = playStage.dolls ? remoteDollImgs : [remoteDollImg];
  const row = $('doll-reveal-row');
  row.innerHTML = '';
  imgs.forEach((img) => {
    const card = document.createElement('div');
    card.className = 'doll-reveal-card';
    const el = document.createElement('img');
    el.src = img.src;
    el.alt = '제작자가 만든 종이인형';
    card.appendChild(el);
    row.appendChild(card);
  });
}
```

- [ ] **Step 2: `finishPlay()`에서 호출**

`index.html:1806`(`$('eyedropper-badge').classList.toggle('hidden', !eyedropperUsed);` 줄) 바로 뒤, `index.html:1807`(`playRun.grade = grade;` 줄) 앞에 삽입:

```js
  renderDollReveal();
```

- [ ] **Step 3: 콘솔 스텁으로 함수 단독 검증(네트워크·Firebase 불필요)**

`index.html`을 열고 devtools 콘솔에서 다중 인형 포맷을 흉내 낸다:

```js
window.playStage = { dolls: [{}, {}] };
window.remoteDollImgs = [
  Object.assign(new Image(), { src: 'data:image/svg+xml;base64,' + btoa('<svg xmlns="http://www.w3.org/2000/svg" width="72" height="72"><circle cx="36" cy="36" r="30" fill="teal"/></svg>') }),
  Object.assign(new Image(), { src: 'data:image/svg+xml;base64,' + btoa('<svg xmlns="http://www.w3.org/2000/svg" width="72" height="72"><rect width="72" height="72" fill="orange"/></svg>') }),
];
renderDollReveal();
document.getElementById('playend-panel').classList.remove('hidden');
```

`#doll-reveal-row`에 청록 원 카드 1개 + 주황 사각 카드 1개가 가로로 나열되는지 확인. 이어서 레거시 단일 포맷도 확인:

```js
window.playStage = {};
window.remoteDollImg = remoteDollImgs[0];
renderDollReveal();
```

카드가 1개로 교체되는지 확인.

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "add: renderDollReveal 구현 및 finishPlay 연결"
```

---

### Task 6: 기능 B 실제 플레이 스루 검증 (선택 — 비용 안내 포함)

이 태스크는 **실제 Firebase 프로젝트에 저장→로드→플레이 왕복**이 필요하다. CLAUDE.md의 크레딧 절약 방침상, 이런 실제 서버 왕복 QA는 진행 전 사용자에게 비용을 알리고 진행 여부를 확인해야 한다. Task 5의 콘솔 스텁 검증으로 렌더링 로직 자체는 이미 확인됐으므로, 이 태스크는 "실제 배포 환경에서도 이상 없는지"의 추가 확인용이며 필수는 아니다.

**Files:** 없음(코드 변경 없음, 수동 QA만)

- [ ] **Step 1: (선택) 사용자에게 실제 왕복 QA 진행 여부 확인**

"Task 5의 콘솔 검증으로 로직은 확인됐습니다. 실제 Firebase에 스테이지 하나를 저장하고 시크릿 창에서 로드→플레이까지 왕복하며 최종 확인하려면 실제 배포 환경이 필요합니다. 진행할까요, 아니면 콘솔 검증으로 충분한가요?"라고 묻고 승인 시에만 아래를 진행한다.

- [ ] **Step 2: (승인 시) 실제 왕복 검증**

인형 1개 스테이지 저장 → 공유 링크로 새 탭/시크릿 창에서 플레이 → 클리어 → 결과 화면에 인형 카드 1개가 실제 칠한 색 그대로 표시되는지 확인. 이어서 인형 2~3개 스테이지로 동일 확인(카드 개수 일치). 시간 초과로 끝나는 경우도 한 번 확인(카드가 동일하게 표시되는지).

---

## 최종 회귀 확인 (기능 A+B 전체)

- [ ] **Step 1: 전체 플로우 한 번 더**

홈 → 만들기(사진 업로드 → 색칠에서 스포이드 드래그로 색 확정 + 취소 모두 시도 → 배치) → 테스트 플레이(로컬, 기능 B 미노출 확인 — `test-panel`에는 인형 카드가 없어야 정상) → 저장(오프라인 모드면 저장 버튼이 비활성/에러 처리되는 기존 동작 그대로인지만 확인, Firebase 연결 시엔 Task 6 방식으로 확인).

- [ ] **Step 2: 최종 커밋 확인**

```bash
git log --oneline -8
git status
```

모든 변경이 커밋되어 있고 워킹트리가 clean한지 확인.
