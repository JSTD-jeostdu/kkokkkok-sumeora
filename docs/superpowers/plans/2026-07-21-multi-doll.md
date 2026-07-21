# 인형 최대 3개 배치 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 제작자가 인형을 1~3개까지 자유롭게 배치하고, 도전자가 전부 찾아야 클리어되는 난이도 조절 기능을 추가한다.

**Architecture:** 편집 상태를 단일 `state.doll` → 배열 `state.dolls`(+ `state.activeDollIndex`)로 리팩터링한다. 인형마다 독립된 오프스크린 캔버스 세트(`paintCanvas`/`dollComposite`/`bgCropCanvas`/`undoStack`)를 배열로 관리하되, "지금 활성 인형" 세트를 가리키는 모듈 전역 `let` 별칭(`paintCanvas`, `paintCtx`, `dollComposite`, `compCtx`, `bgCropCanvas`, `bgCropCtx`, `undoStack`)을 두어 기존 붓질·되돌리기·스포이드 함수 내부 코드를 거의 그대로 재사용한다(탭 전환 시 별칭만 재할당). 편집기(배치·색칠·로컬 테스트 플레이)는 언제나 배열 기반으로 동작하도록 완전히 새로 짠다 — 편집기는 앞으로 오직 신규 `dolls` 배열 스키마만 생성하므로 "예전 편집기"를 남겨둘 필요가 없다. 반면 **이미 저장된 기존 단일 인형 스테이지를 읽는 코드**(`activeDoll()`, `hitTest()`, 힌트 3종, `spawnFoundDoll()`, `spawnRevealRing()`)는 원격 플레이(`ready`/`play`/`end`/`view` 단계에서 `playStage.doll`이 있는 경우) 전용으로 완전히 그대로 남기고, 신규 다중 인형 원격 플레이는 이름이 `md`로 시작하는 병렬 함수 세트로 새로 추가한다. 각 호출 지점(포인터 핸들러, `renderStage()`, `finishPlay()` 등)은 `playStage.dolls` 존재 여부로 새 함수와 기존 함수를 분기 호출한다.

**Tech Stack:** 순수 HTML/CSS/JS (`index.html` 단일 파일), 빌드 도구·프레임워크·외부 JS 라이브러리 없음. Firebase Firestore(문서에 base64 직접 저장, Storage 미사용).

## Global Constraints

- 단일 파일 `index.html`만 수정 — 새 파일 생성 금지 (단, `firestore.rules`는 별도 파일로 이미 존재하며 이번 작업에서 함께 수정)
- 주석은 한국어
- 탭 영역(버튼) 44px 이상 유지
- `prefers-reduced-motion` 존중 (이 기능은 새 애니메이션을 추가하지 않음)
- **프로젝트에 자동 테스트 프레임워크가 없음** — 검증은 `.superpowers/sdd/qa/drive.js`(git-ignore된 로컬 스크래치 디렉터리, Playwright+Chromium 기반, 이전 세션에서 구축해 즉시 실행 가능)를 사용한다. 기존 시나리오(`task1`~`task3`, `outline`, `colorpicker-*`)의 구조를 참고해 이 기능 전용 새 시나리오를 추가한다. 만약 `.superpowers/sdd/qa/`가 없다면 `npx playwright install chromium` 후 `npm install playwright@latest --no-save`로 새로 구성한다(자세한 복구 절차는 각 태스크의 QA 절에 있음)
- QA 스크린샷은 Read 도구로 직접 열어 눈으로 확인한다(콘솔 출력만 믿지 않는다)
- **이 플랜의 각 태스크는 앞선 태스크가 이미 파일을 수정한 상태를 전제로 한다** — 뒤 태스크로 갈수록 실제 줄 번호는 이 문서에 적힌 숫자와 달라진다. 각 Step에 적힌 줄 번호는 "이 플랜을 작성한 시점" 기준이므로, 실제 편집 시에는 반드시 Step에 함께 적힌 주변 코드(앵커 텍스트)를 파일에서 검색해 위치를 찾아 편집한다
- **핵심 설계 원칙**: 기존에 저장된 단일 인형 스테이지(Firestore 문서에 `doll`/`dollB64`/`eyedropperUses` 단일 필드가 있고 `dolls` 배열이 없는 경우)는 절대 깨지면 안 된다. 원격 플레이·결과 보기·갤러리에서 이 문서들을 다룰 때는 기존 함수(`activeDoll`, `hitTest`, `spawnFoundDoll`, `spawnRevealRing`, `hintQuadrant`, `hintTemp`, `tempFeedback`, `hintFlash`, `computeGrade`, `recordResult`)를 호출 지점에서 그대로 호출하고, 새 코드는 `playStage.dolls`가 있을 때만 타는 `md`-접두사 함수로 분리한다
- `CONFIG.MAX_DOLLS = 3` 추가. 그 외 기존 `CONFIG` 값(등급 구간, 힌트 텍스트 등)은 무변경, 그대로 재사용

참고 스펙: `docs/superpowers/specs/2026-07-21-multi-doll-design.md`

---

### Task 1: 데이터 모델 리팩터 — `state.dolls` 배열 + 캔버스 세트 별칭

**Files:**
- Modify: `index.html` — `CONFIG` 객체(`MAX_DOLLS` 추가)
- Modify: `index.html` — `state` 객체 리터럴(`doll` → `dolls`/`activeDollIndex`, `eyedropperUses` 제거)
- Modify: `index.html` — 오프스크린 캔버스 선언부(단일 상수 → 배열 + 별칭 함수)
- Modify: `index.html` — `undoStack` 선언(단일 상수 제거, 별칭으로 대체)
- Modify: `index.html` — `captureBgCrop()`, `renderDollView()`, `nearDoll()`, `dollPos()`, `updateEyedropperUI()`, `pickColorFromCrop()`
- Modify: `index.html` — 배치 컨트롤 배선(`sc-scale`/`sc-rot`/`btn-flip`)
- Modify: `index.html` — `renderStage()` (배치/테스트 단계 분기 추가)
- Modify: `index.html` — `resetAll()`

**Interfaces:**
- Produces: `activeEditDoll()`, `setActiveDollIndex(i)`, `dollSets`(배열), `makeDollCanvasSet()`, `mdDollSize(d)`, `mdDrawDoll(d, source)`, `mdRenderEditDolls()` — Task 2~4가 이들을 확장·사용
- Consumes: 기존 `CONFIG.DOLL_SIZE`, `dollPath`, `minSide()`

- [ ] **Step 1: `CONFIG`에 `MAX_DOLLS` 추가**

`CONFIG` 객체에서 `KEY_LEN: 8,` 다음 줄에 추가:
```js
  MAX_DOLLS: 3,
```

- [ ] **Step 2: `state` 객체를 배열 기반으로 변경**

현재:
```js
const state = {
  phase: 'home', // home|upload|paint|place|test|save|share|load|ready|play|end|view|error|delete
  photoW: 0, photoH: 0,
  photoB64: null,           // 업로드용 사진 dataURL 캐시 (적응형 압축 결과)
  doll: { x: 0.5, y: 0.5, scale: 0.10, rotation: 0, flip: false }, // 정규화 좌표(0~1)
  color: CONFIG.DEFAULT_COLOR,
  brush: 12,
  eraser: false,
  eyedropperUses: 0,        // 이번 스테이지에서 스포이드를 쓴 횟수 (최대 EYEDROPPER_MAX_USES)
  found: false,
  reveal: false,
  savedId: null,
  deleteTargetId: null,
  deleteReturnPhase: 'home',
};
```
아래로 교체:
```js
const state = {
  phase: 'home', // home|upload|paint|place|test|save|share|load|ready|play|end|view|error|delete
  photoW: 0, photoH: 0,
  photoB64: null,           // 업로드용 사진 dataURL 캐시 (적응형 압축 결과)
  dolls: [{ x: 0.5, y: 0.5, scale: 0.10, rotation: 0, flip: false, eyedropperUses: 0 }], // 1~3개, 정규화 좌표(0~1)
  activeDollIndex: 0,       // 지금 배치·색칠 중인 인형의 인덱스
  color: CONFIG.DEFAULT_COLOR,
  brush: 12,
  eraser: false,
  found: false,
  reveal: false,
  savedId: null,
  deleteTargetId: null,
  deleteReturnPhase: 'home',
};
```

- [ ] **Step 3: 오프스크린 캔버스를 인형별 배열 + 별칭으로 변경**

현재:
```js
/* ---------- 오프스크린 캔버스 ---------- */
const photoBuf = document.createElement('canvas');
const bufCtx = photoBuf.getContext('2d', { willReadFrequently: true });
const paintCanvas = document.createElement('canvas');
paintCanvas.width = paintCanvas.height = CONFIG.DOLL_SIZE;
const paintCtx = paintCanvas.getContext('2d', { willReadFrequently: true });
const dollComposite = document.createElement('canvas');
dollComposite.width = dollComposite.height = CONFIG.DOLL_SIZE;
const compCtx = dollComposite.getContext('2d');
// 칠하기 화면에서 인형 뒤에 보여줄 배경 크롭 (정적 스냅샷 — 저장되지 않음, 화면 표시 전용)
const bgCropCanvas = document.createElement('canvas');
bgCropCanvas.width = bgCropCanvas.height = CONFIG.DOLL_SIZE;
const bgCropCtx = bgCropCanvas.getContext('2d', { willReadFrequently: true });
bgCropCtx.fillStyle = '#FAF7F0';
bgCropCtx.fillRect(0, 0, CONFIG.DOLL_SIZE, CONFIG.DOLL_SIZE);
// 칠하기 화면에서 인형은 크롭 배율의 역수만큼 작게 그려 주변 배경이 보이도록 한다
const DOLL_VIEW_SCALE = 1 / CONFIG.PAINT_BG_CROP_RATIO;
```
아래로 교체:
```js
/* ---------- 오프스크린 캔버스 ---------- */
const photoBuf = document.createElement('canvas');
const bufCtx = photoBuf.getContext('2d', { willReadFrequently: true });
// 칠하기 화면에서 인형은 크롭 배율의 역수만큼 작게 그려 주변 배경이 보이도록 한다
const DOLL_VIEW_SCALE = 1 / CONFIG.PAINT_BG_CROP_RATIO;
/* 인형마다 독립된 캔버스 세트(붓칠·합성·배경크롭·되돌리기 이력)를 만든다.
   paintCanvas/paintCtx/dollComposite/compCtx/bgCropCanvas/bgCropCtx/undoStack은
   아래에서 "지금 활성 인형"의 세트를 가리키는 별칭(let)으로 선언되어,
   탭을 바꿀 때 setActiveDollIndex()만 호출하면 기존 붓질·되돌리기·스포이드
   함수들은 코드 수정 없이 그 인형의 캔버스에 그대로 동작한다. */
function makeDollCanvasSet(){
  const pc = document.createElement('canvas');
  pc.width = pc.height = CONFIG.DOLL_SIZE;
  const dc = document.createElement('canvas');
  dc.width = dc.height = CONFIG.DOLL_SIZE;
  const bc = document.createElement('canvas');
  bc.width = bc.height = CONFIG.DOLL_SIZE;
  const bctx = bc.getContext('2d', { willReadFrequently: true });
  bctx.fillStyle = '#FAF7F0';
  bctx.fillRect(0, 0, CONFIG.DOLL_SIZE, CONFIG.DOLL_SIZE);
  return {
    paintCanvas: pc, paintCtx: pc.getContext('2d', { willReadFrequently: true }),
    dollComposite: dc, compCtx: dc.getContext('2d'),
    bgCropCanvas: bc, bgCropCtx: bctx,
    undoStack: [],
  };
}
let dollSets = [makeDollCanvasSet()];
let paintCanvas, paintCtx, dollComposite, compCtx, bgCropCanvas, bgCropCtx, undoStack;
function setActiveDollIndex(i){
  state.activeDollIndex = i;
  const s = dollSets[i];
  paintCanvas = s.paintCanvas; paintCtx = s.paintCtx;
  dollComposite = s.dollComposite; compCtx = s.compCtx;
  bgCropCanvas = s.bgCropCanvas; bgCropCtx = s.bgCropCtx;
  undoStack = s.undoStack;
}
setActiveDollIndex(0);
function activeEditDoll(){ return state.dolls[state.activeDollIndex]; }
```

- [ ] **Step 4: 기존 `const undoStack = [];` 제거**

`/* ---------- 색칠: 실행취소 ---------- */` 주석 다음 줄의 다음 코드를 삭제(Step 3에서 별칭으로 이미 선언했으므로 중복 선언 제거):
```js
const undoStack = [];
```
(`pushUndo()`/`undo()` 함수 본문은 그대로 둔다 — `undoStack`을 이름으로 참조하므로 별칭이 가리키는 세트에 자동으로 맞게 동작한다)

- [ ] **Step 5: `captureBgCrop()`의 `state.doll` 참조 교체**

현재:
```js
function captureBgCrop(){
  bgCropCtx.fillStyle = '#FAF7F0';
  bgCropCtx.fillRect(0, 0, CONFIG.DOLL_SIZE, CONFIG.DOLL_SIZE);
  if (!state.photoW) return;
  const cropSide = dollSizePx() * CONFIG.PAINT_BG_CROP_RATIO;
  const cx = state.doll.x * state.photoW, cy = state.doll.y * state.photoH;
```
아래로 교체(`dollSizePx()`는 원격 플레이 전용 `activeDoll()`을 내부에서 쓰므로 편집 단계에서는 더 이상 호출하지 않는다):
```js
function captureBgCrop(){
  bgCropCtx.fillStyle = '#FAF7F0';
  bgCropCtx.fillRect(0, 0, CONFIG.DOLL_SIZE, CONFIG.DOLL_SIZE);
  if (!state.photoW) return;
  const d = activeEditDoll();
  const cropSide = d.scale * minSide() * CONFIG.PAINT_BG_CROP_RATIO;
  const cx = d.x * state.photoW, cy = d.y * state.photoH;
```
(나머지 `bgCropCtx.drawImage(...)` 이하는 그대로 둔다)

- [ ] **Step 6: `renderDollView()`의 `state.doll` 참조 교체**

현재:
```js
  viewCtx.rotate(state.doll.rotation * Math.PI / 180);
  if (state.doll.flip) viewCtx.scale(-1, 1);
```
아래로 교체:
```js
  viewCtx.rotate(activeEditDoll().rotation * Math.PI / 180);
  if (activeEditDoll().flip) viewCtx.scale(-1, 1);
```

- [ ] **Step 7: `nearDoll()`의 `state.doll`/`dollSizePx()` 참조 교체**

현재:
```js
function nearDoll(nx, ny){
  const dx = (nx - state.doll.x) * state.photoW;
  const dy = (ny - state.doll.y) * state.photoH;
  const grabR = Math.max(dollSizePx() * 0.9, minSide() * 0.09);
  return Math.hypot(dx, dy) <= grabR;
}
```
아래로 교체:
```js
function nearDoll(nx, ny){
  const d = activeEditDoll();
  const dx = (nx - d.x) * state.photoW;
  const dy = (ny - d.y) * state.photoH;
  const grabR = Math.max(d.scale * minSide() * 0.9, minSide() * 0.09);
  return Math.hypot(dx, dy) <= grabR;
}
```

- [ ] **Step 8: `dollPos()`의 `state.doll` 참조 교체**

현재:
```js
function dollPos(e){
  const v = viewPos(e);
  const c = CONFIG.DOLL_SIZE / 2;
  const dx = (v.x - c) / DOLL_VIEW_SCALE;
  const dy = (v.y - c) / DOLL_VIEW_SCALE;
  const rad = -state.doll.rotation * Math.PI / 180;
  const cos = Math.cos(rad), sin = Math.sin(rad);
  let lx = dx * cos - dy * sin;
  let ly = dx * sin + dy * cos;
  if (state.doll.flip) lx = -lx;
  return { x: lx + c, y: ly + c };
}
```
아래로 교체:
```js
function dollPos(e){
  const v = viewPos(e);
  const c = CONFIG.DOLL_SIZE / 2;
  const dx = (v.x - c) / DOLL_VIEW_SCALE;
  const dy = (v.y - c) / DOLL_VIEW_SCALE;
  const d = activeEditDoll();
  const rad = -d.rotation * Math.PI / 180;
  const cos = Math.cos(rad), sin = Math.sin(rad);
  let lx = dx * cos - dy * sin;
  let ly = dx * sin + dy * cos;
  if (d.flip) lx = -lx;
  return { x: lx + c, y: ly + c };
}
```

- [ ] **Step 9: `updateEyedropperUI()`/`pickColorFromCrop()`의 `state.eyedropperUses` 참조 교체**

현재:
```js
function updateEyedropperUI(){
  const remaining = Math.max(0, CONFIG.EYEDROPPER_MAX_USES - state.eyedropperUses);
```
아래로 교체:
```js
function updateEyedropperUI(){
  const remaining = Math.max(0, CONFIG.EYEDROPPER_MAX_USES - activeEditDoll().eyedropperUses);
```

현재(`pickColorFromCrop()` 안):
```js
  state.eyedropperUses++;
```
아래로 교체:
```js
  activeEditDoll().eyedropperUses++;
```

- [ ] **Step 10: 배치 컨트롤 배선의 `state.doll` 참조 교체**

현재:
```js
/* ---------- 배치 컨트롤 배선 ---------- */
$('sc-scale').addEventListener('input', (e) => {
  state.doll.scale = Number(e.target.value) / 100;
  $('out-scale').textContent = e.target.value + '%';
  renderStage();
});
$('sc-rot').addEventListener('input', (e) => {
  state.doll.rotation = Number(e.target.value);
  $('out-rot').textContent = e.target.value + '°';
  renderStage();
});
$('btn-flip').addEventListener('click', () => {
  state.doll.flip = !state.doll.flip;
  $('btn-flip').setAttribute('aria-pressed', String(state.doll.flip));
  renderStage();
});
```
아래로 교체:
```js
/* ---------- 배치 컨트롤 배선 ---------- */
$('sc-scale').addEventListener('input', (e) => {
  activeEditDoll().scale = Number(e.target.value) / 100;
  $('out-scale').textContent = e.target.value + '%';
  renderStage();
});
$('sc-rot').addEventListener('input', (e) => {
  activeEditDoll().rotation = Number(e.target.value);
  $('out-rot').textContent = e.target.value + '°';
  renderStage();
});
$('btn-flip').addEventListener('click', () => {
  activeEditDoll().flip = !activeEditDoll().flip;
  $('btn-flip').setAttribute('aria-pressed', String(activeEditDoll().flip));
  renderStage();
});
```

- [ ] **Step 11: 배치 드래그 핸들러(`photoCanvas` `pointermove`)의 `state.doll` 참조 교체**

현재:
```js
photoCanvas.addEventListener('pointermove', (e) => {
  if (!dragging || state.phase !== 'place') return;
  const p = stagePos(e);
  state.doll.x = Math.min(0.98, Math.max(0.02, p.nx));
  state.doll.y = Math.min(0.98, Math.max(0.02, p.ny));
  renderStage();
});
```
아래로 교체:
```js
photoCanvas.addEventListener('pointermove', (e) => {
  if (!dragging || state.phase !== 'place') return;
  const p = stagePos(e);
  const d = activeEditDoll();
  d.x = Math.min(0.98, Math.max(0.02, p.nx));
  d.y = Math.min(0.98, Math.max(0.02, p.ny));
  renderStage();
});
```

- [ ] **Step 12: `renderStage()`에 배치/테스트 단계용 다중 인형 렌더링 분기 추가**

`/* ---------- 인형 렌더링 ---------- */` 섹션(`renderDollComposite()` 다음, `captureBgCrop()` 앞)에 새 함수들을 추가:
```js
/* ---------- 다중 인형(md) 공용 렌더링 헬퍼 ---------- */
function mdDollSize(d){ return d.scale * minSide(); }
function mdDrawDoll(d, source){
  const s = mdDollSize(d);
  pctx2d.save();
  pctx2d.translate(d.x * state.photoW, d.y * state.photoH);
  pctx2d.rotate(d.rotation * Math.PI / 180);
  if (d.flip) pctx2d.scale(-1, 1);
  pctx2d.drawImage(source, -s / 2, -s / 2, s, s);
  pctx2d.restore();
}
/* 배치·테스트 단계에서 편집 중인 인형 전부를 그린다.
   테스트 단계에서 이미 찾은 인형은 건너뛴다(Task 4에서 testFoundFlags를 채움 —
   그 전까지는 항상 undefined라 이 조건은 항상 통과해 전부 그려진다). */
function mdRenderEditDolls(){
  state.dolls.forEach((d, i) => {
    if (state.phase === 'test' && typeof testFoundFlags !== 'undefined' && testFoundFlags && testFoundFlags[i]) return;
    mdDrawDoll(d, dollSets[i].dollComposite);
  });
}
```

`renderStage()`의 현재 코드:
```js
function renderStage(){
  if (!state.photoW) return;
  pctx2d.clearRect(0, 0, state.photoW, state.photoH);
  pctx2d.drawImage(photoBuf, 0, 0);
  // 인형 표시 조건: 배치 / (테스트·실제 플레이 & 미발견) / (플레이 실패 공개) / 제작자 결과 보기
  const show = (state.phase === 'place')
            || (state.phase === 'test' && !state.found)
            || (state.phase === 'play' && !state.found)
            || (state.phase === 'end' && state.reveal)
            || (state.phase === 'view');
  if (show){
    const d = activeDoll(), s = dollSizePx();
    pctx2d.save();
    pctx2d.translate(d.x * state.photoW, d.y * state.photoH);
    pctx2d.rotate(d.rotation * Math.PI / 180);
    if (d.flip) pctx2d.scale(-1, 1);
    pctx2d.drawImage(activeDollSource(), -s/2, -s/2, s, s);
    pctx2d.restore();
  }
```
아래로 교체(히트맵 블록은 바로 아래 그대로 이어짐, 손대지 않음):
```js
function renderStage(){
  if (!state.photoW) return;
  pctx2d.clearRect(0, 0, state.photoW, state.photoH);
  pctx2d.drawImage(photoBuf, 0, 0);
  if (state.phase === 'place' || state.phase === 'test'){
    // 배치·테스트 단계는 항상 다중 인형 배열(state.dolls) 기반으로 그린다 — 편집기는 앞으로 이 경로만 쓴다
    mdRenderEditDolls();
  } else {
    // ---- 기존 단일 인형 경로 (레거시 원격 플레이 전용, 무변경 — Task 6에서 dolls 배열 분기 추가 예정) ----
    const show = (state.phase === 'play' && !state.found)
              || (state.phase === 'end' && state.reveal)
              || (state.phase === 'view');
    if (show){
      const d = activeDoll(), s = dollSizePx();
      pctx2d.save();
      pctx2d.translate(d.x * state.photoW, d.y * state.photoH);
      pctx2d.rotate(d.rotation * Math.PI / 180);
      if (d.flip) pctx2d.scale(-1, 1);
      pctx2d.drawImage(activeDollSource(), -s/2, -s/2, s, s);
      pctx2d.restore();
    }
  }
```
(`state.phase === 'test'`가 기존 `show` 목록에서 사라진 것은 의도된 변경이다 — 테스트 단계는 항상 위쪽의 새 분기로 처리되어 이 레거시 블록에 절대 도달하지 않으므로, 도달 불가능한 조건을 남겨두지 않고 정리했다. `activeDoll()`/`dollSizePx()`/`activeDollSource()` 함수 자체는 이 태스크에서 전혀 수정하지 않는다.)

- [ ] **Step 13: `resetAll()` 갱신**

현재:
```js
function resetAll(){
  state.photoW = state.photoH = 0;
  state.photoB64 = null;
  state.doll = { x: 0.5, y: 0.5, scale: 0.10, rotation: 0, flip: false };
  state.found = false;
  state.reveal = false;
  state.savedId = null;
  state.eyedropperUses = 0;
  eyedropperOn = false;
  playStage = null;
  remoteDollImg = null;
  playRun = null;
  resultTaps = null;
  undoStack.length = 0;
  paintCtx.clearRect(0, 0, CONFIG.DOLL_SIZE, CONFIG.DOLL_SIZE);
  $('sc-scale').value = 10; $('out-scale').textContent = '10%';
  $('sc-rot').value = 0;   $('out-rot').textContent = '0°';
  $('btn-flip').setAttribute('aria-pressed', 'false');
  updateEyedropperUI();
  clearOverlay();
  renderDollView();
}
```
아래로 교체:
```js
function resetAll(){
  state.photoW = state.photoH = 0;
  state.photoB64 = null;
  state.dolls = [{ x: 0.5, y: 0.5, scale: 0.10, rotation: 0, flip: false, eyedropperUses: 0 }];
  dollSets = [makeDollCanvasSet()];
  setActiveDollIndex(0);
  state.found = false;
  state.reveal = false;
  state.savedId = null;
  eyedropperOn = false;
  playStage = null;
  remoteDollImg = null;
  playRun = null;
  resultTaps = null;
  $('sc-scale').value = 10; $('out-scale').textContent = '10%';
  $('sc-rot').value = 0;   $('out-rot').textContent = '0°';
  $('btn-flip').setAttribute('aria-pressed', 'false');
  updateEyedropperUI();
  clearOverlay();
  renderDollView();
}
```
(`undoStack.length = 0; paintCtx.clearRect(...)` 두 줄은 `dollSets = [makeDollCanvasSet()]`가 이미 완전히 새 빈 캔버스 세트를 만들어 대체하므로 제거한다 — 지우려는 대상 자체가 새로 생성되어 중복이 된다)

- [ ] **Step 14: 브라우저 QA — 단일 인형 배치·색칠이 그대로 동작하는지 확인**

`.superpowers/sdd/qa/drive.js`에 새 시나리오 `md-task1`을 추가(기존 `toPaintScreen(page)` 헬퍼와 `outline`/`colorpicker-*` 시나리오 구조를 참고):
```js
    } else if (scenario === 'md-task1'){
      await toPaintScreen(page);
      await page.screenshot({ path: shotPath('md-task1-paint-screen') });
      // 붓질이 정상 동작하는지(별칭 리팩터 후 회귀 확인)
      await page.click('.brush-btn[data-size="24"]');
      const canvas = page.locator('#dollView');
      const box = await canvas.boundingBox();
      const cx = box.x + box.width / 2, cy = box.y + box.height / 2;
      await page.mouse.move(cx - 30, cy);
      await page.mouse.down();
      await page.mouse.move(cx + 30, cy, { steps: 4 });
      await page.mouse.up();
      await page.screenshot({ path: shotPath('md-task1-after-stroke') });
      // 배치 화면으로 돌아가서 인형이 사진 위에 정상 표시되는지(mdRenderEditDolls 확인)
      await page.click('#btn-to-place');
      await page.waitForSelector('#place-panel:not(.hidden)', { timeout: 5000 });
      await page.screenshot({ path: shotPath('md-task1-place-screen') });
      const errors2 = [];
      page.on('pageerror', (e) => errors2.push(e.message));
      console.log('extra errors after place nav:', errors2.length ? JSON.stringify(errors2) : 'none');
    }
```
실행:
```bash
cd .superpowers/sdd/qa && node drive.js md-task1
```
확인 사항: `md-task1-paint-screen.png`가 정상적으로 색칠 화면을 보여주는지(회귀 없음), `md-task1-after-stroke.png`에 그은 획이 인형 위에 보이는지, `md-task1-place-screen.png`에서 배치 화면에 인형이 사진 위에 정상적으로 보이는지(이제 `mdRenderEditDolls()` 경로를 타지만 시각적으로는 이전과 동일해야 함). 콘솔에 에러가 없어야 한다.

이어서 기존 확대 모달·컬러피커 회귀 확인: `node drive.js task1`, `node drive.js task2`, `node drive.js colorpicker-drag`도 재실행해 정상 출력을 확인한다.

**주의**: 이 태스크 이후 "테스트 플레이" 버튼(`btn-to-test`)을 눌러 인형을 탭하면 아직 깨진다(레거시 `hitTest()`가 이제 없는 `state.doll`을 참조) — 이건 Task 4에서 고친다. 이번 QA에서는 테스트 플레이 화면에 진입하지 않는다(진입 자체는 안전하지만, **탭하지는 않는다**).

- [ ] **Step 15: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
인형 배열(state.dolls) 데이터 모델과 인형별 캔버스 세트 별칭 도입

state.doll(단일)을 state.dolls(배열)+activeDollIndex로 리팩터링하고,
인형마다 독립된 오프스크린 캔버스 세트를 배열로 관리하되 활성 인형
세트를 가리키는 전역 별칭으로 기존 붓질·되돌리기·스포이드 코드를
그대로 재사용한다. 배치/테스트 단계 렌더링을 다중 인형 경로로 전환하고,
원격 플레이(레거시 단일 인형) 경로는 완전히 그대로 둔다.
EOF
)"
```

---

### Task 2: 배치 화면 — 다중 인형 탭 UI (선택·추가·삭제)

**Files:**
- Modify: `index.html` — `place-panel` HTML(탭 컨테이너 추가)
- Modify: `index.html` — CSS(탭 스타일 추가)
- Modify: `index.html` — JS(탭 렌더링·선택·추가·삭제 함수, `setPhase()`의 `place` 진입 훅)

**Interfaces:**
- Consumes: Task 1의 `activeEditDoll()`, `setActiveDollIndex()`, `dollSets`, `makeDollCanvasSet()`, `state.dolls`
- Produces: `renderDollTabs()`, `selectDollTab(i)`, `syncPlaceControlsToActiveDoll()`, `addDoll()`, `deleteDollTab(i)` — Task 3(색칠 화면의 "+인형 추가")이 `addDoll()`을 사용

- [ ] **Step 1: `place-panel`에 탭 컨테이너 HTML 추가**

`place-panel`의 현재 마크업:
```html
    <div id="place-panel" class="panel hidden">
      <h3>인형을 몰래 숨겨두자 🤫</h3>
      <p class="hint-line">인형을 <b>손가락으로 끌어서</b> 옮길 수 있어요</p>
```
아래로 교체:
```html
    <div id="place-panel" class="panel hidden">
      <h3>인형을 몰래 숨겨두자 🤫</h3>
      <div class="doll-tab-row" id="doll-tabs"></div>
      <p class="hint-line">인형을 <b>손가락으로 끌어서</b> 옮길 수 있어요</p>
```

- [ ] **Step 2: CSS 추가**

`.slider-row output{...}` 규칙 다음 줄에 추가:
```css
.doll-tab-row{ display:flex; gap:6px; flex-wrap:wrap; margin-bottom:10px; }
.doll-tab-item{ display:flex; gap:2px; }
.doll-tab-del{ font-size:14px; padding:4px 8px; }
```

- [ ] **Step 3: 탭 렌더링·선택·추가·삭제 함수 추가**

`/* ---------- 배치 드래그 ---------- */` 주석 바로 앞(`nearDoll()` 함수 앞)에 추가:
```js
/* ---------- 배치: 인형 탭(선택·추가·삭제) ---------- */
function syncPlaceControlsToActiveDoll(){
  const d = activeEditDoll();
  $('sc-scale').value = Math.round(d.scale * 100 * 2) / 2;
  $('out-scale').textContent = $('sc-scale').value + '%';
  $('sc-rot').value = d.rotation;
  $('out-rot').textContent = d.rotation + '°';
  $('btn-flip').setAttribute('aria-pressed', String(d.flip));
  updateEyedropperUI();
}
function renderDollTabs(){
  const wrap = $('doll-tabs');
  wrap.innerHTML = '';
  const circled = ['①', '②', '③'];
  state.dolls.forEach((d, i) => {
    const item = document.createElement('div');
    item.className = 'doll-tab-item';
    const tab = document.createElement('button');
    tab.className = 'tool-btn doll-tab';
    tab.textContent = circled[i] || String(i + 1);
    tab.setAttribute('aria-pressed', String(i === state.activeDollIndex));
    tab.setAttribute('aria-label', '인형 ' + (i + 1) + ' 선택');
    tab.addEventListener('click', () => selectDollTab(i));
    item.appendChild(tab);
    if (state.dolls.length > 1){
      const del = document.createElement('button');
      del.className = 'tool-btn doll-tab-del';
      del.textContent = '✕';
      del.setAttribute('aria-label', '인형 ' + (i + 1) + ' 삭제');
      del.addEventListener('click', (e) => { e.stopPropagation(); deleteDollTab(i); });
      item.appendChild(del);
    }
    wrap.appendChild(item);
  });
  if (state.dolls.length < CONFIG.MAX_DOLLS){
    const add = document.createElement('button');
    add.className = 'tool-btn';
    add.id = 'btn-add-doll';
    add.textContent = '+ 인형';
    add.setAttribute('aria-label', '인형 추가');
    add.addEventListener('click', addDoll);
    wrap.appendChild(add);
  }
}
function selectDollTab(i){
  setActiveDollIndex(i);
  syncPlaceControlsToActiveDoll();
  renderDollTabs();
  renderDollView();
  renderStage();
}
function addDoll(){
  if (state.dolls.length >= CONFIG.MAX_DOLLS) return;
  state.dolls.push({ x: 0.5, y: 0.5, scale: 0.10, rotation: 0, flip: false, eyedropperUses: 0 });
  dollSets.push(makeDollCanvasSet());
  setPhase('place');
  selectDollTab(state.dolls.length - 1);
}
function deleteDollTab(i){
  if (state.dolls.length <= 1) return;
  if (!confirm('인형 ' + (i + 1) + '을(를) 삭제할까요?')) return;
  state.dolls.splice(i, 1);
  dollSets.splice(i, 1);
  const newIndex = Math.min(i, state.dolls.length - 1);
  selectDollTab(newIndex);
}
```

- [ ] **Step 4: `setPhase()`에 배치 단계 진입 훅 추가**

`setPhase(phase)` 함수의 현재 코드:
```js
  if (phase === 'test'){
    state.found = false;
    testStatus.textContent = '인형이 어디 숨었을까? 사진을 눌러 찾아보세요!';
    testStatus.classList.remove('win');
    $('btn-back-edit').style.display = 'none';
    $('btn-giveup').style.display = '';
    clearOverlay();
  }
```
바로 앞에 추가:
```js
  if (phase === 'place'){
    renderDollTabs();
    syncPlaceControlsToActiveDoll();
  }
```

- [ ] **Step 5: 브라우저 QA — 탭 추가·삭제·전환 확인**

`drive.js`에 `md-task2` 시나리오 추가:
```js
    } else if (scenario === 'md-task2'){
      await toPaintScreen(page); // 배치 → 칠하기까지 진행된 상태
      await page.click('#btn-to-place');
      await page.waitForSelector('#place-panel:not(.hidden)', { timeout: 5000 });
      await page.screenshot({ path: shotPath('md-task2-single-doll') });
      // 인형 추가
      await page.click('#btn-add-doll');
      await page.waitForTimeout(200);
      await page.screenshot({ path: shotPath('md-task2-after-add-1') });
      const tabCount1 = await page.locator('.doll-tab-item').count();
      console.log('tab count after 1 add:', tabCount1);
      // 두 번째 인형 위치를 다르게 드래그
      const stageBox = await page.locator('#photoCanvas').boundingBox();
      await page.mouse.move(stageBox.x + stageBox.width * 0.5, stageBox.y + stageBox.height * 0.5);
      await page.mouse.down();
      await page.mouse.move(stageBox.x + stageBox.width * 0.75, stageBox.y + stageBox.height * 0.25, { steps: 5 });
      await page.mouse.up();
      await page.screenshot({ path: shotPath('md-task2-after-drag-2nd') });
      // 인형 하나 더 추가 (최대 3개 확인)
      await page.click('#btn-add-doll');
      await page.waitForTimeout(200);
      const addBtnAfter3 = await page.$('#btn-add-doll');
      console.log('add button hidden at 3 dolls:', !addBtnAfter3);
      // 1번 탭으로 돌아가서 슬라이더 값이 그 인형 것으로 복원되는지
      await page.click('.doll-tab >> nth=0');
      await page.waitForTimeout(100);
      const scaleVal = await page.inputValue('#sc-scale');
      console.log('scale value after switching to tab 1:', scaleVal);
      // 인형 삭제
      await page.click('.doll-tab-del >> nth=0').catch(async () => {
        // 첫 탭은 아직 삭제 버튼이 없을 수 있으므로(1개였다면), 마지막 탭 삭제로 대체 확인
      });
      page.once('dialog', (d) => d.accept());
      const delBtns = await page.$$('.doll-tab-del');
      if (delBtns.length) await delBtns[delBtns.length - 1].click();
      await page.waitForTimeout(200);
      const tabCountAfterDel = await page.locator('.doll-tab-item').count();
      console.log('tab count after delete:', tabCountAfterDel);
      await page.screenshot({ path: shotPath('md-task2-after-delete') });
    }
```
실행: `node drive.js md-task2`

확인 사항: `md-task2-single-doll.png`(탭 1개, 삭제 버튼 없음), `md-task2-after-add-1.png`(탭 2개, 두 번째 인형이 사진 중앙에 추가됨), `md-task2-after-drag-2nd.png`(두 번째 인형이 오른쪽 위로 이동), `add button hidden at 3 dolls: true`, `scale value after switching to tab 1`이 기존 인형1의 값(기본 10)과 일치, `tab count after delete`가 삭제 전보다 1 적음, `md-task2-after-delete.png`에서 남은 인형들이 정상 표시.

`node drive.js md-task1`도 재실행해 회귀 없는지 확인.

- [ ] **Step 6: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
배치 화면에 다중 인형 탭 UI(선택·추가·삭제) 추가

인형 개수만큼 번호 탭을 표시하고, 탭 클릭으로 활성 인형을 전환하며
슬라이더가 그 인형 값으로 동기화된다. 3개 미만일 때 "+ 인형" 버튼으로
추가, 2개 이상일 때 각 탭에 삭제 버튼이 노출된다(최소 1개 유지).
EOF
)"
```

---

### Task 3: 색칠 화면 — 인형 추가 연결

**Files:**
- Modify: `index.html` — `paint-panel` HTML(인형 추가 버튼·진행 표시 추가)
- Modify: `index.html` — CSS
- Modify: `index.html` — JS(버튼 배선, 표시 갱신 함수, `setPhase()`의 `paint` 진입 훅 확장)

**Interfaces:**
- Consumes: Task 2의 `addDoll()`
- Produces: `updatePaintAddDollUI()` — 이후 태스크가 직접 쓰지는 않지만 유지보수 시 참고

- [ ] **Step 1: `paint-panel`에 인형 추가 버튼과 진행 표시 HTML 추가**

`paint-panel`의 현재 마크업(패널 액션 앞부분):
```html
      <div class="panel-actions">
        <button class="btn btn-ghost" id="btn-repick">사진 바꾸기</button>
        <button class="btn btn-ghost" id="btn-to-place">다시 배치</button>
      </div>
      <div class="panel-actions">
        <button class="btn btn-ghost" id="btn-to-test">테스트 플레이</button>
        <button class="btn btn-primary" id="btn-save">저장하고 공유하기</button>
      </div>
```
아래로 교체:
```html
      <p class="hint-line hidden" id="paint-doll-progress"></p>
      <div class="panel-actions hidden" id="paint-add-doll-row">
        <button class="btn btn-ghost btn-wide" id="btn-add-doll-paint">+ 인형 추가</button>
      </div>
      <div class="panel-actions">
        <button class="btn btn-ghost" id="btn-repick">사진 바꾸기</button>
        <button class="btn btn-ghost" id="btn-to-place">다시 배치</button>
      </div>
      <div class="panel-actions">
        <button class="btn btn-ghost" id="btn-to-test">테스트 플레이</button>
        <button class="btn btn-primary" id="btn-save">저장하고 공유하기</button>
      </div>
```

- [ ] **Step 2: 진행 표시·추가 버튼 갱신 함수 추가 + `setPhase()` 훅 확장**

Task 2에서 추가한 `/* ---------- 배치: 인형 탭(선택·추가·삭제) ---------- */` 섹션 끝(`deleteDollTab()` 함수 다음)에 추가:
```js
/* ---------- 색칠: 인형 추가 UI 갱신 ---------- */
function updatePaintAddDollUI(){
  const canAdd = state.dolls.length < CONFIG.MAX_DOLLS;
  $('paint-add-doll-row').classList.toggle('hidden', !canAdd);
  const progress = $('paint-doll-progress');
  if (state.dolls.length > 1){
    progress.textContent = '인형 ' + (state.activeDollIndex + 1) + '/' + state.dolls.length + ' 색칠 중';
    progress.classList.remove('hidden');
  } else {
    progress.classList.add('hidden');
  }
}
```

`setPhase(phase)`의 현재 코드:
```js
  if (phase === 'paint'){
    captureBgCrop();
    renderDollView();
  }
```
아래로 교체:
```js
  if (phase === 'paint'){
    captureBgCrop();
    renderDollView();
    updatePaintAddDollUI();
  }
```

Task 2의 `selectDollTab(i)` 함수 끝(`renderStage();` 다음 줄)에 한 줄 추가:
```js
  updatePaintAddDollUI();
```

- [ ] **Step 3: 인형 추가 버튼 배선**

`/* ---------- 도구 버튼 배선 ---------- */` 섹션의 `$('btn-eyedropper').addEventListener(...)` 블록 다음에 추가:
```js
$('btn-add-doll-paint').addEventListener('click', addDoll);
```

- [ ] **Step 4: 브라우저 QA — 색칠 화면에서 인형 추가·전환 확인**

`drive.js`에 `md-task3` 시나리오 추가:
```js
    } else if (scenario === 'md-task3'){
      await toPaintScreen(page);
      const addRowHiddenAt1 = await page.evaluate(() => document.getElementById('paint-add-doll-row').classList.contains('hidden'));
      console.log('add-doll row hidden at 1 doll:', addRowHiddenAt1);
      await page.click('#btn-add-doll-paint');
      await page.waitForSelector('#place-panel:not(.hidden)', { timeout: 5000 });
      console.log('navigated to place after add: OK');
      await page.click('#btn-to-paint');
      await page.waitForSelector('#paint-panel:not(.hidden)', { timeout: 5000 });
      const progressText = await page.textContent('#paint-doll-progress');
      console.log('progress text with 2 dolls:', progressText);
      await page.screenshot({ path: shotPath('md-task3-second-doll-paint') });
      // 두 번째 인형에 다른 색으로 칠하기
      await page.fill('#cp-hex', '#22AA44').catch(() => {});
    }
```
실행: `node drive.js md-task3`

확인: `add-doll row hidden at 1 doll: true`, "인형 추가" 클릭 후 배치 화면으로 이동, `progress text with 2 dolls`가 "인형 2/2 색칠 중" 형태, 스크린샷에서 두 번째 인형(아직 흰 실루엣)이 색칠 화면에 정상 표시.

`node drive.js md-task1`, `node drive.js md-task2` 회귀 재확인.

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
색칠 화면에 인형 추가 버튼과 진행 표시(N/M) 추가

색칠 완료 후 인형이 3개 미만이면 "+ 인형 추가" 버튼으로 배치 화면에
돌아가 새 인형을 배치할 수 있다. 인형이 2개 이상이면 몇 번째를
칠하는 중인지 표시한다.
EOF
)"
```

---

### Task 4: 로컬 테스트 플레이 다중화

**Files:**
- Modify: `index.html` — `setPhase()`의 `test` 진입 훅
- Modify: `index.html` — `photoCanvas`의 `pointerdown` 핸들러 (`test` 분기)
- Modify: `index.html` — `mdRenderEditDolls()` (Task 1에서 만든 미완성 조건 완성)

**Interfaces:**
- Consumes: Task 1의 `mdDollSize`, `mdDrawDoll`, `mdRenderEditDolls`, `activeEditDoll`
- Produces: `testFoundFlags`, `mdAllFound(flags)`, `mdEditHitTestIndex(nx, ny)`, `mdSpawnFoundDoll(d, source)` — Task 6이 `mdAllFound`/`mdSpawnFoundDoll`를 원격 플레이에도 재사용

- [ ] **Step 1: 테스트 발견 상태 배열과 공용 헬퍼 추가**

`let playStage = null;` 등 모듈 전역 변수 선언부(`let timerIv = null, countdownIv = null;` 다음 줄)에 추가:
```js
let testFoundFlags = null;   // 로컬 테스트 플레이 중 인형별 발견 여부(state.dolls와 같은 길이)
let playFoundFlags = null;   // 신규 포맷 원격 플레이 중 인형별 발견 여부 (Task 6에서 사용)
```

Task 1에서 추가한 `mdDrawDoll()` 함수 다음에 공용 헬퍼 추가:
```js
function mdAllFound(flags){ return !!flags && flags.length > 0 && flags.every(Boolean); }
function mdSpawnFoundDoll(d, source){
  const { kx, ky } = cssScale();
  const sizeCss = mdDollSize(d) * kx;
  const img = document.createElement('img');
  img.className = 'found-doll';
  img.src = (source instanceof HTMLImageElement) ? source.src : source.toDataURL();
  img.alt = '찾아낸 종이인형';
  img.style.width = sizeCss + 'px';
  img.style.height = sizeCss + 'px';
  img.style.left = (d.x * state.photoW * kx - sizeCss / 2) + 'px';
  img.style.top  = (d.y * state.photoH * ky - sizeCss / 2) + 'px';
  overlay.appendChild(img);
}
```
(`cssScale()`는 이미 위쪽에 정의되어 있으므로 여기서는 그대로 참조만 한다 — 만약 순서상 `mdSpawnFoundDoll`이 `cssScale` 정의보다 앞에 온다면, 함수 선언은 호이스팅되므로 실행 시점에는 문제없다)

- [ ] **Step 2: `mdRenderEditDolls()`의 발견 여부 조건 정리**

Task 1에서 만든 코드:
```js
function mdRenderEditDolls(){
  state.dolls.forEach((d, i) => {
    if (state.phase === 'test' && typeof testFoundFlags !== 'undefined' && testFoundFlags && testFoundFlags[i]) return;
    mdDrawDoll(d, dollSets[i].dollComposite);
  });
}
```
아래로 교체(이제 `testFoundFlags`가 항상 선언되어 있으므로 `typeof` 방어 코드가 필요 없어짐):
```js
function mdRenderEditDolls(){
  state.dolls.forEach((d, i) => {
    if (state.phase === 'test' && testFoundFlags && testFoundFlags[i]) return;
    mdDrawDoll(d, dollSets[i].dollComposite);
  });
}
```

- [ ] **Step 3: `setPhase()`의 `test` 진입 훅에 다중 인형 초기화 추가**

현재:
```js
  if (phase === 'test'){
    state.found = false;
    testStatus.textContent = '인형이 어디 숨었을까? 사진을 눌러 찾아보세요!';
    testStatus.classList.remove('win');
    $('btn-back-edit').style.display = 'none';
    $('btn-giveup').style.display = '';
    clearOverlay();
  }
```
아래로 교체:
```js
  if (phase === 'test'){
    state.found = false;
    testFoundFlags = state.dolls.map(() => false);
    testStatus.textContent = state.dolls.length > 1
      ? '인형들이 어디 숨었을까? 사진을 눌러 찾아보세요! (0/' + state.dolls.length + ')'
      : '인형이 어디 숨었을까? 사진을 눌러 찾아보세요!';
    testStatus.classList.remove('win');
    $('btn-back-edit').style.display = 'none';
    $('btn-giveup').style.display = '';
    clearOverlay();
  }
```

- [ ] **Step 4: `photoCanvas` `pointerdown`의 `test` 분기를 다중 인형 판정으로 교체**

현재:
```js
  } else if (state.phase === 'test'){
    if (state.found) return;
    if (hitTest(p.nx, p.ny)){
      state.found = true;
      renderStage();
      spawnFoundDoll();
      testStatus.textContent = '찾았다! 🎉 위장이 뚫렸네요';
      testStatus.classList.add('win');
      $('btn-back-edit').style.display = '';
      $('btn-giveup').style.display = 'none';
      if (navigator.vibrate) navigator.vibrate(80);
    } else {
      spawnRipple(p.cssX, p.cssY);
    }
  } else if (state.phase === 'play'){
```
아래로 교체(편집기의 테스트 플레이는 항상 `state.dolls` 배열 기반이므로, 기존 `hitTest()`/`activeDoll()`/`spawnFoundDoll()`를 더 이상 호출하지 않고 완전히 새 로직으로 대체한다 — 원격 플레이용 레거시 함수들은 그대로 남아 `play` 분기 이하에서 별개로 쓰인다):
```js
  } else if (state.phase === 'test'){
    if (mdAllFound(testFoundFlags)) return;
    const idx = mdEditHitTestIndex(p.nx, p.ny);
    if (idx !== -1){
      testFoundFlags[idx] = true;
      renderStage();
      mdSpawnFoundDoll(state.dolls[idx], dollSets[idx].dollComposite);
      if (navigator.vibrate) navigator.vibrate(80);
      if (mdAllFound(testFoundFlags)){
        testStatus.textContent = '찾았다! 🎉 위장이 뚫렸네요';
        testStatus.classList.add('win');
        $('btn-back-edit').style.display = '';
        $('btn-giveup').style.display = 'none';
      } else {
        const foundCount = testFoundFlags.filter(Boolean).length;
        testStatus.textContent = foundCount + '/' + state.dolls.length + ' 찾음 — 계속 찾아보세요!';
      }
    } else {
      spawnRipple(p.cssX, p.cssY);
    }
  } else if (state.phase === 'play'){
```

`nearDoll()` 함수 다음(`/* ---------- 판정·연출 ---------- */` 주석 앞)에 새 함수 추가:
```js
function mdEditHitTestIndex(nx, ny){
  for (let i = 0; i < state.dolls.length; i++){
    if (testFoundFlags[i]) continue;
    const d = state.dolls[i];
    const s = mdDollSize(d);
    const dx = (nx - d.x) * state.photoW, dy = (ny - d.y) * state.photoH;
    const r = Math.max(s * 0.5 * CONFIG.HIT_MARGIN, minSide() * CONFIG.HIT_MIN_RATIO);
    if (Math.hypot(dx, dy) <= r) return i;
  }
  return -1;
}
```

- [ ] **Step 5: 브라우저 QA — 인형 1개·2개 테스트 플레이 확인**

`drive.js`에 `md-task4` 시나리오 추가:
```js
    } else if (scenario === 'md-task4'){
      await toPaintScreen(page);
      // 인형 1개 상태에서 테스트 플레이: 정답 위치를 알고 있으므로(항상 중앙 0.5,0.5) 그 지점을 탭
      await page.click('#btn-to-test');
      await page.waitForSelector('#test-panel:not(.hidden)', { timeout: 5000 });
      const stageBox1 = await page.locator('#photoCanvas').boundingBox();
      await page.mouse.click(stageBox1.x + stageBox1.width * 0.5, stageBox1.y + stageBox1.height * 0.5);
      await page.waitForTimeout(200);
      const statusAfter1 = await page.textContent('#test-status');
      console.log('status after finding 1-doll stage:', statusAfter1);
      await page.screenshot({ path: shotPath('md-task4-single-found') });
      // 편집으로 복귀 후 인형 2개로 만들고 다시 테스트
      await page.click('#btn-back-edit');
      await page.waitForSelector('#place-panel:not(.hidden)', { timeout: 5000 });
      await page.click('#btn-add-doll');
      await page.waitForTimeout(200);
      const stageBox2 = await page.locator('#photoCanvas').boundingBox();
      // 두 번째 인형을 오른쪽 아래로 이동
      await page.mouse.move(stageBox2.x + stageBox2.width * 0.5, stageBox2.y + stageBox2.height * 0.5);
      await page.mouse.down();
      await page.mouse.move(stageBox2.x + stageBox2.width * 0.85, stageBox2.y + stageBox2.height * 0.85, { steps: 5 });
      await page.mouse.up();
      await page.click('#btn-to-test');
      await page.waitForSelector('#test-panel:not(.hidden)', { timeout: 5000 });
      const statusBefore = await page.textContent('#test-status');
      console.log('status with 2 dolls before finding any:', statusBefore);
      // 인형1(중앙) 먼저 찾기
      await page.mouse.click(stageBox2.x + stageBox2.width * 0.5, stageBox2.y + stageBox2.height * 0.5);
      await page.waitForTimeout(200);
      const statusAfterFirst = await page.textContent('#test-status');
      console.log('status after finding 1 of 2:', statusAfterFirst);
      await page.screenshot({ path: shotPath('md-task4-found-1-of-2') });
      // 인형2(오른쪽 아래) 찾기 → 전부 발견
      await page.mouse.click(stageBox2.x + stageBox2.width * 0.85, stageBox2.y + stageBox2.height * 0.85);
      await page.waitForTimeout(200);
      const statusAfterAll = await page.textContent('#test-status');
      console.log('status after finding all:', statusAfterAll);
      await page.screenshot({ path: shotPath('md-task4-found-all') });
    }
```
실행: `node drive.js md-task4`

확인: `status after finding 1-doll stage`가 "찾았다! 🎉 위장이 뚫렸네요"(단일 인형 기존 문구 그대로 — 회귀 확인), `status with 2 dolls before finding any`가 "...(0/2)" 형태, `status after finding 1 of 2`가 "1/2 찾음 — 계속 찾아보세요!", `status after finding all`이 "찾았다! 🎉 위장이 뚫렸네요". 스크린샷에서 각 단계의 발견 연출(낙하한 인형 이미지)이 올바른 위치에 보이는지 확인.

`md-task1`~`md-task3` 및 `task1`~`task3`, `colorpicker-drag` 회귀 재확인.

- [ ] **Step 6: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
로컬 테스트 플레이를 다중 인형 배열 기반으로 전환

편집 중인 state.dolls 전부에 대해 판정하도록 테스트 플레이 탭 핸들러를
새로 작성. 하나씩 찾을 때마다 발견 연출을 재생하고 "N/M 찾음"으로
갱신하며, 전부 찾으면 기존과 동일한 클리어 문구를 보여준다.
EOF
)"
```

---

### Task 5: 저장(`saveStage`) + Firestore 보안 규칙

**Files:**
- Modify: `index.html` — `saveStage()`, `encodeDollB64()`(제거·대체)
- Modify: `firestore.rules` — `create` 검증 규칙

**Interfaces:**
- Consumes: Task 1의 `state.dolls`, `dollSets`
- Produces: `encodeDollB64ForIndex(i)` — Firestore 문서에 `dolls` 배열 필드로 저장

- [ ] **Step 1: `encodeDollB64()`를 인덱스 기반으로 교체**

현재:
```js
/* 인형 PNG 인코딩 — 예산 초과 시 192px로 축소 재시도 (투명 배경 유지 위해 PNG 고정) */
function encodeDollB64(){
  renderDollComposite();
  let url = dollComposite.toDataURL('image/png');
  if (url.length <= CONFIG.DOLL_B64_MAX) return url;
  const c = document.createElement('canvas');
  c.width = c.height = 192;
  c.getContext('2d').drawImage(dollComposite, 0, 0, 192, 192);
  url = c.toDataURL('image/png');
  return url.length <= CONFIG.DOLL_B64_MAX ? url : null;
}
```
아래로 교체(각 인형의 `dollComposite`는 그 인형을 칠할 때마다 `renderDollView()`→`renderDollComposite()`가 이미 호출되어 항상 최신 상태이므로, 저장 시점에 다시 렌더링할 필요가 없다):
```js
/* 인형 PNG 인코딩 — 예산 초과 시 192px로 축소 재시도 (투명 배경 유지 위해 PNG 고정).
   각 인형의 dollComposite는 칠할 때마다(renderDollView 경유) 이미 최신 상태로 유지된다. */
function encodeDollB64ForIndex(i){
  const composite = dollSets[i].dollComposite;
  let url = composite.toDataURL('image/png');
  if (url.length <= CONFIG.DOLL_B64_MAX) return url;
  const c = document.createElement('canvas');
  c.width = c.height = 192;
  c.getContext('2d').drawImage(composite, 0, 0, 192, 192);
  url = c.toDataURL('image/png');
  return url.length <= CONFIG.DOLL_B64_MAX ? url : null;
}
```

> **해석 메모**: 스펙 문서의 "공유 카드 문구"는 문자 그대로 읽으면 `buildShareCard()`(플레이 종료 후 결과를 담은 1080×1080 PNG 카드)를 가리키는 것처럼 보이지만, 그 카드는 게임이 끝난 뒤 플레이어의 결과(시간·등급)를 보여주는 용도라 "N마리를 찾아라!" 같은 초대 문구가 맥락상 어색하다. 대신 저장 직후 제작자에게 보이는 `share-panel`(링크를 복사해 친구에게 보내는 화면)에 초대 문구를 추가하는 것이 더 자연스럽다고 판단해 아래처럼 구현한다. 이 해석이 의도와 다르면 `buildShareCard()`에 별도로 추가하는 것으로 쉽게 바꿀 수 있다.

- [ ] **Step 2: `saveStage()`를 `dolls` 배열 저장으로 교체**

현재:
```js
    saveMsg('사진을 꾹꾹 눌러 담는 중…', true);
    const photoB64 = encodePhotoB64();
    if (!photoB64) throw new Error('사진이 너무 복잡해서 담지 못했어요. 다른 사진으로 시도해 주세요.');
    const dollB64 = encodeDollB64();
    if (!dollB64) throw new Error('인형 인코딩에 실패했어요.');

    const ref = fb.db.collection('stages').doc();
    const id = ref.id;

    saveMsg('스테이지를 기록하는 중…', true);
    const deleteKey = genDeleteKey();
    const deleteKeyHash = await sha256Hex(deleteKey);
    await ref.set({
      createdAt: firebase.firestore.FieldValue.serverTimestamp(),
      creatorUid: fb.uid,
      photoB64, dollB64,   // Storage 미사용: 문서에 직접 저장
      doll: { ...state.doll },
      timeLimit: CONFIG.TIME_LIMIT,
      deleteKeyHash,
      stats: { plays: 0, clears: 0, totalClearMs: 0 },
      taps: [],
      eyedropperUses: state.eyedropperUses,
    });
```
아래로 교체:
```js
    saveMsg('사진을 꾹꾹 눌러 담는 중…', true);
    const photoB64 = encodePhotoB64();
    if (!photoB64) throw new Error('사진이 너무 복잡해서 담지 못했어요. 다른 사진으로 시도해 주세요.');
    const dolls = [];
    for (let i = 0; i < state.dolls.length; i++){
      const dollB64 = encodeDollB64ForIndex(i);
      if (!dollB64) throw new Error('인형 인코딩에 실패했어요.');
      const d = state.dolls[i];
      dolls.push({ x: d.x, y: d.y, scale: d.scale, rotation: d.rotation, flip: d.flip, dollB64, eyedropperUses: d.eyedropperUses });
    }

    const ref = fb.db.collection('stages').doc();
    const id = ref.id;

    saveMsg('스테이지를 기록하는 중…', true);
    const deleteKey = genDeleteKey();
    const deleteKeyHash = await sha256Hex(deleteKey);
    await ref.set({
      createdAt: firebase.firestore.FieldValue.serverTimestamp(),
      creatorUid: fb.uid,
      photoB64,            // Storage 미사용: 문서에 직접 저장
      dolls,
      timeLimit: CONFIG.TIME_LIMIT,
      deleteKeyHash,
      stats: { plays: 0, clears: 0, totalClearMs: 0 },
      taps: [],
    });
```

이어서(`state.savedId = id;` 다음, `const base = ...` 다음, `setPhase('share')` 앞) 공유 안내 문구에 인형 개수 반영 코드를 추가:
```js
    $('share-invite-msg').textContent = dolls.length > 1
      ? ('숨은 인형 ' + dolls.length + '마리를 찾아라!')
      : '';
```

`share-panel`의 현재 마크업(`<p class="hint-line">이 링크를 받은 친구가 인형 찾기에 도전해요</p>` 다음 줄)에 새 요소를 추가:
```html
      <p class="hint-line" id="share-invite-msg"></p>
```

- [ ] **Step 3: Firestore 보안 규칙 — `create` 검증을 `dolls` 배열 기반으로 교체**

`firestore.rules`의 현재 `create` 규칙:
```
      allow create: if request.auth != null
        && request.resource.data.creatorUid == request.auth.uid
        && request.resource.data.keys().hasAll(['createdAt','creatorUid','photoB64','dollB64','doll','timeLimit','deleteKeyHash','stats','taps','eyedropperUses'])
        && request.resource.data.photoB64 is string
        && request.resource.data.photoB64.size() < 800000   // 문서 1MB 제한 예산
        && request.resource.data.dollB64 is string
        && request.resource.data.dollB64.size() < 200000
        && request.resource.data.eyedropperUses is int
        && request.resource.data.eyedropperUses >= 0
        && request.resource.data.eyedropperUses <= 3;       // CONFIG.EYEDROPPER_MAX_USES와 동일
```
아래로 교체(신규 스테이지는 앞으로 항상 `dolls` 배열 스키마로만 생성되므로 — 기존 `doll`/`dollB64`/`eyedropperUses` 단일 필드 스키마는 이 클라이언트가 다시는 만들지 않는다 — `create` 검증을 배열 스키마 전용으로 바꾼다. 이미 저장된 기존 문서는 `update`/`read`/`delete` 규칙에만 영향을 받으며, 이 규칙들은 스키마와 무관하게 이미 동작하므로 무변경 유지한다):
```
      allow create: if request.auth != null
        && request.resource.data.creatorUid == request.auth.uid
        && request.resource.data.keys().hasAll(['createdAt','creatorUid','photoB64','dolls','timeLimit','deleteKeyHash','stats','taps'])
        && request.resource.data.photoB64 is string
        && request.resource.data.photoB64.size() < 800000   // 문서 1MB 제한 예산
        && request.resource.data.dolls is list
        && request.resource.data.dolls.size() >= 1
        && request.resource.data.dolls.size() <= 3          // CONFIG.MAX_DOLLS와 동일
        && request.resource.data.dolls[0].dollB64 is string
        && request.resource.data.dolls[0].dollB64.size() < 200000
        && request.resource.data.dolls[0].eyedropperUses is int
        && request.resource.data.dolls[0].eyedropperUses >= 0
        && request.resource.data.dolls[0].eyedropperUses <= 3
        && (request.resource.data.dolls.size() < 2 || (
             request.resource.data.dolls[1].dollB64 is string
          && request.resource.data.dolls[1].dollB64.size() < 200000
          && request.resource.data.dolls[1].eyedropperUses is int
          && request.resource.data.dolls[1].eyedropperUses >= 0
          && request.resource.data.dolls[1].eyedropperUses <= 3
        ))
        && (request.resource.data.dolls.size() < 3 || (
             request.resource.data.dolls[2].dollB64 is string
          && request.resource.data.dolls[2].dollB64.size() < 200000
          && request.resource.data.dolls[2].eyedropperUses is int
          && request.resource.data.dolls[2].eyedropperUses >= 0
          && request.resource.data.dolls[2].eyedropperUses <= 3
        ));
```
(`update`/`read`/`delete` 규칙은 스키마와 무관하게 이미 동작하므로 전혀 건드리지 않는다)

- [ ] **Step 4: 브라우저 QA — 저장이 `dolls` 배열로 기록되는지 확인**

이 프로젝트의 Firebase 프로젝트는 `CONFIG.firebase`에 실제 값이 채워져 있어 오프라인 모드가 아니다. 로컬 `index.html`(file:// 또는 정적 서버)에서 저장을 시도하면 실제 Firestore에 문서가 생성되므로, QA에서는 **실제 저장을 실행하지 않는다** — 대신 저장 버튼을 누르기 직전까지의 상태(인코딩된 데이터)를 콘솔에서 직접 검사해 형태만 확인한다:

`drive.js`에 `md-task5` 시나리오 추가:
```js
    } else if (scenario === 'md-task5'){
      await toPaintScreen(page);
      await page.click('#btn-add-doll-paint');
      await page.waitForSelector('#place-panel:not(.hidden)', { timeout: 5000 });
      await page.click('#btn-to-paint');
      await page.waitForSelector('#paint-panel:not(.hidden)', { timeout: 5000 });
      // 실제 Firestore에 쓰지 않고, saveStage()가 만들 데이터 형태만 검증
      const shape = await page.evaluate(() => {
        const dolls = state.dolls.map((d, i) => {
          const b64 = encodeDollB64ForIndex(i);
          return { hasB64: !!b64, b64Prefix: b64 ? b64.slice(0, 20) : null, eyedropperUses: d.eyedropperUses, scale: d.scale };
        });
        return JSON.stringify(dolls);
      });
      console.log('dolls array shape saveStage would write:', shape);
    }
```
실행: `node drive.js md-task5`

확인: 콘솔에 출력된 배열이 인형 2개 각각에 대해 `hasB64: true`, `b64Prefix`가 `"data:image/png;base64"`로 시작, `eyedropperUses`가 숫자(0 이상), `scale`이 0.1 근처인지 확인.

`firestore.rules` 파일 자체는 코드 정적 검토로 충분하다(Firebase 콘솔 규칙 시뮬레이터가 없는 이 환경에서 실제 배포 테스트는 사용자가 배포 시점에 직접 수행 — CLAUDE.md의 배포 절차 참고). 규칙 문법이 유효한지는 괄호 짝과 필드명이 위 코드와 정확히 일치하는지 육안으로 재확인한다.

`md-task1`~`md-task4` 회귀 재확인.

- [ ] **Step 5: 커밋**

```bash
git add index.html firestore.rules
git commit -m "$(cat <<'EOF'
저장을 dolls 배열 스키마로 전환하고 Firestore 생성 규칙 갱신

saveStage()가 인형마다 개별 dollB64·eyedropperUses를 포함한 dolls
배열을 Firestore에 기록하도록 변경. 이제 신규 저장은 항상 이 스키마만
쓰므로 create 규칙도 배열 검증으로 교체(update/read/delete는 스키마와
무관해 무변경). 공유 안내에 인형 개수(2개 이상 시)도 표시.
EOF
)"
```

---

### Task 6: 원격 플레이 — 로드·렌더링·판정·힌트 다중화

**Files:**
- Modify: `index.html` — `fetchStage()`, `loadStage()`, `startCountdown()`
- Modify: `index.html` — `renderStage()` (원격 플레이 분기 추가)
- Modify: `index.html` — `photoCanvas`의 `pointerdown` 핸들러 (`play` 분기)
- Modify: `index.html` — `finishPlay()`
- Modify: `index.html` — 힌트 3종(`useHint` 대체, `hint-quad`/`hint-temp`/`hint-flash` 배선)

**Interfaces:**
- Consumes: Task 4의 `mdDollSize`, `mdDrawDoll`, `mdAllFound`, `mdSpawnFoundDoll`, `playFoundFlags`
- Produces: `remoteDollImgs`, `mdHitTestIndex`, `mdRenderPlayDolls`, `mdSpawnRevealRings`, `mdNearestUnfoundIndex`, `mdUseHint`, `mdHintQuadrant`, `mdHintTemp`, `mdTempFeedback`, `mdHintFlash`

- [ ] **Step 1: 전역 변수에 `remoteDollImgs` 추가**

`let remoteDollImg = null;` 다음 줄에 추가:
```js
let remoteDollImgs = null;   // 신규 포맷(dolls 배열) 원격 플레이용 로드된 인형 이미지 배열
```

- [ ] **Step 2: `fetchStage()`에 다중 인형 이미지 로드 분기 추가**

현재:
```js
async function fetchStage(id){
  const ref = fb.db.collection('stages').doc(id);
  const snap = await ref.get();
  if (!snap.exists) return null;
  const d = snap.data();
  const [photoImg, dollImg] = await Promise.all([loadImg(d.photoB64), loadImg(d.dollB64)]);
  remoteDollImg = dollImg;
  const { w, h } = fitSize(photoImg.naturalWidth, photoImg.naturalHeight, CONFIG.PHOTO_MAX);
  photoBuf.width = w; photoBuf.height = h;
  bufCtx.drawImage(photoImg, 0, 0, w, h);
  state.photoW = w; state.photoH = h;
  photoCanvas.width = w; photoCanvas.height = h;
  return { ref, data: d };
}
```
아래로 교체(레거시 블록은 마지막 8줄 그대로 유지):
```js
async function fetchStage(id){
  const ref = fb.db.collection('stages').doc(id);
  const snap = await ref.get();
  if (!snap.exists) return null;
  const d = snap.data();
  if (d.dolls){
    const [photoImg, ...dollImgs] = await Promise.all(
      [loadImg(d.photoB64)].concat(d.dolls.map((dl) => loadImg(dl.dollB64)))
    );
    remoteDollImgs = dollImgs;
    const { w, h } = fitSize(photoImg.naturalWidth, photoImg.naturalHeight, CONFIG.PHOTO_MAX);
    photoBuf.width = w; photoBuf.height = h;
    bufCtx.drawImage(photoImg, 0, 0, w, h);
    state.photoW = w; state.photoH = h;
    photoCanvas.width = w; photoCanvas.height = h;
    return { ref, data: d };
  }
  const [photoImg, dollImg] = await Promise.all([loadImg(d.photoB64), loadImg(d.dollB64)]);
  remoteDollImg = dollImg;
  const { w, h } = fitSize(photoImg.naturalWidth, photoImg.naturalHeight, CONFIG.PHOTO_MAX);
  photoBuf.width = w; photoBuf.height = h;
  bufCtx.drawImage(photoImg, 0, 0, w, h);
  state.photoW = w; state.photoH = h;
  photoCanvas.width = w; photoCanvas.height = h;
  return { ref, data: d };
}
```

- [ ] **Step 3: `loadStage()`에 `dolls`/`doll` 분기 추가**

현재:
```js
    playStage = {
      id, ref: got.ref, doll: got.data.doll,
      timeLimit: got.data.timeLimit || CONFIG.TIME_LIMIT,
      stats: got.data.stats || { plays: 0, clears: 0, totalClearMs: 0 },
      eyedropperUses: got.data.eyedropperUses || 0,
    };
    startCountdown();
```
아래로 교체:
```js
    if (got.data.dolls){
      playStage = {
        id, ref: got.ref, dolls: got.data.dolls,
        timeLimit: got.data.timeLimit || CONFIG.TIME_LIMIT,
        stats: got.data.stats || { plays: 0, clears: 0, totalClearMs: 0 },
      };
    } else {
      playStage = {
        id, ref: got.ref, doll: got.data.doll,
        timeLimit: got.data.timeLimit || CONFIG.TIME_LIMIT,
        stats: got.data.stats || { plays: 0, clears: 0, totalClearMs: 0 },
        eyedropperUses: got.data.eyedropperUses || 0,
      };
    }
    startCountdown();
```

- [ ] **Step 4: `startCountdown()`에 발견 상태 배열 초기화 추가**

현재:
```js
function startCountdown(){
  setPhase('ready');
  state.found = false; state.reveal = false;
  clearOverlay();
```
아래로 교체:
```js
function startCountdown(){
  setPhase('ready');
  state.found = false; state.reveal = false;
  playFoundFlags = playStage.dolls ? playStage.dolls.map(() => false) : null;
  clearOverlay();
```

- [ ] **Step 5: 원격 플레이용 다중 인형 렌더링·판정·연출 헬�어 추가**

Task 4에서 추가한 `mdSpawnFoundDoll()` 함수 다음에 추가:
```js
/* ---------- 원격 플레이(신규 dolls 배열) 전용 헬퍼 ---------- */
function mdHitTestIndex(dolls, foundFlags, nx, ny){
  for (let i = 0; i < dolls.length; i++){
    if (foundFlags[i]) continue;
    const d = dolls[i];
    const s = mdDollSize(d);
    const dx = (nx - d.x) * state.photoW, dy = (ny - d.y) * state.photoH;
    const r = Math.max(s * 0.5 * CONFIG.HIT_MARGIN, minSide() * CONFIG.HIT_MIN_RATIO);
    if (Math.hypot(dx, dy) <= r) return i;
  }
  return -1;
}
function mdRenderPlayDolls(){
  playStage.dolls.forEach((d, i) => {
    if (playFoundFlags && playFoundFlags[i]) return;
    mdDrawDoll(d, remoteDollImgs[i]);
  });
}
function mdSpawnRevealRings(){
  playStage.dolls.forEach((d, i) => {
    if (playFoundFlags[i]) return;
    const { kx, ky } = cssScale();
    const sizeCss = Math.max(mdDollSize(d) * kx * 1.6, 56);
    const ring = document.createElement('div');
    ring.className = 'reveal-ring';
    ring.style.width = sizeCss + 'px';
    ring.style.height = sizeCss + 'px';
    ring.style.left = (d.x * state.photoW * kx - sizeCss / 2) + 'px';
    ring.style.top  = (d.y * state.photoH * ky - sizeCss / 2) + 'px';
    overlay.appendChild(ring);
  });
}
function mdAnyEyedropperUsed(dolls){ return dolls.some((d) => d.eyedropperUses > 0); }
/* "가장 가까운 미발견 인형" = 가장 최근 탭 위치(없으면 사진 중앙) 기준 최근접 */
function mdNearestUnfoundIndex(){
  const taps = playRun.taps;
  const last = taps.length ? taps[taps.length - 1] : { x: 0.5, y: 0.5 };
  let best = -1, bestDist = Infinity;
  playStage.dolls.forEach((d, i) => {
    if (playFoundFlags[i]) return;
    const dist = Math.hypot((d.x - last.x) * state.photoW, (d.y - last.y) * state.photoH);
    if (dist < bestDist){ bestDist = dist; best = i; }
  });
  return best;
}
```

- [ ] **Step 6: `renderStage()`에 원격 다중 인형 분기 추가**

Task 1에서 만든 `else` 블록의 현재 코드:
```js
  } else {
    // ---- 기존 단일 인형 경로 (레거시 원격 플레이 전용, 무변경 — Task 6에서 dolls 배열 분기 추가 예정) ----
    const show = (state.phase === 'play' && !state.found)
              || (state.phase === 'end' && state.reveal)
              || (state.phase === 'view');
    if (show){
      const d = activeDoll(), s = dollSizePx();
      pctx2d.save();
      pctx2d.translate(d.x * state.photoW, d.y * state.photoH);
      pctx2d.rotate(d.rotation * Math.PI / 180);
      if (d.flip) pctx2d.scale(-1, 1);
      pctx2d.drawImage(activeDollSource(), -s/2, -s/2, s, s);
      pctx2d.restore();
    }
  }
```
아래로 교체:
```js
  } else if (playStage && playStage.dolls && (
       (state.phase === 'play' && !mdAllFound(playFoundFlags))
    || (state.phase === 'end' && state.reveal)
    || (state.phase === 'view')
  )){
    mdRenderPlayDolls();
  } else {
    // ---- 기존 단일 인형 경로 (레거시 원격 플레이 전용, 무변경) ----
    const show = (state.phase === 'play' && !state.found)
              || (state.phase === 'end' && state.reveal)
              || (state.phase === 'view');
    if (show){
      const d = activeDoll(), s = dollSizePx();
      pctx2d.save();
      pctx2d.translate(d.x * state.photoW, d.y * state.photoH);
      pctx2d.rotate(d.rotation * Math.PI / 180);
      if (d.flip) pctx2d.scale(-1, 1);
      pctx2d.drawImage(activeDollSource(), -s/2, -s/2, s, s);
      pctx2d.restore();
    }
  }
```

- [ ] **Step 7: `photoCanvas` `pointerdown`의 `play` 분기에 다중 인형 판정 추가**

현재:
```js
  } else if (state.phase === 'play'){
    if (!playRun || playRun.ended) return;
    playRun.taps.push({ x: Math.round(p.nx * 1000) / 1000, y: Math.round(p.ny * 1000) / 1000 });
    if (hitTest(p.nx, p.ny)){
      finishPlay(true);
    } else {
      spawnRipple(p.cssX, p.cssY);
      if (playRun.tempOn) tempFeedback(p.nx, p.ny);
    }
  }
```
아래로 교체:
```js
  } else if (state.phase === 'play'){
    if (!playRun || playRun.ended) return;
    playRun.taps.push({ x: Math.round(p.nx * 1000) / 1000, y: Math.round(p.ny * 1000) / 1000 });
    if (playStage.dolls){
      const idx = mdHitTestIndex(playStage.dolls, playFoundFlags, p.nx, p.ny);
      if (idx !== -1){
        playFoundFlags[idx] = true;
        renderStage();
        mdSpawnFoundDoll(playStage.dolls[idx], remoteDollImgs[idx]);
        if (navigator.vibrate) navigator.vibrate(80);
        const foundCount = playFoundFlags.filter(Boolean).length;
        $('play-status').textContent = foundCount + '/' + playStage.dolls.length + ' 찾음';
        if (mdAllFound(playFoundFlags)) finishPlay(true);
      } else {
        spawnRipple(p.cssX, p.cssY);
        if (playRun.tempOn) mdTempFeedback(p.nx, p.ny);
      }
    } else if (hitTest(p.nx, p.ny)){
      finishPlay(true);
    } else {
      spawnRipple(p.cssX, p.cssY);
      if (playRun.tempOn) tempFeedback(p.nx, p.ny);
    }
  }
```

- [ ] **Step 8: `finishPlay()`에 다중 인형 분기 추가**

현재:
```js
function finishPlay(cleared){
  if (!playRun || playRun.ended) return;
  playRun.ended = true;
  playRun.cleared = cleared;
  clearInterval(timerIv); timerIv = null;
  const tl = playStage.timeLimit;
  // 이번 플레이를 반영한 스테이지 stats (등급 표시용 — 서버 기록과 동일 계산)
  const statsNow = {
    plays: playStage.stats.plays + 1,
    clears: playStage.stats.clears + (cleared ? 1 : 0),
    totalClearMs: playStage.stats.totalClearMs + (cleared ? Math.round(performance.now() - playRun.start) : 0),
  };
  if (cleared){
    playRun.clearMs = Math.round(performance.now() - playRun.start);
    state.found = true;
    renderStage();
    spawnFoundDoll();
    if (navigator.vibrate) navigator.vibrate([60, 40, 120]);
    $('playend-title').textContent = '찾았다! 🎉';
    $('playend-sub').textContent = '';
    $('result-time').textContent = (playRun.clearMs / 1000).toFixed(1) + '초 만에 발견!';
    const remainRatio = Math.max(0, 1 - playRun.clearMs / (tl * 1000));
    $('gauge').style.width = (remainRatio * 100) + '%';
  } else {
    state.reveal = true;
    renderStage();
    spawnRevealRing();
    $('playend-title').textContent = '시간 종료! ⏰';
    $('playend-sub').textContent = '인형은 반짝이는 곳에 숨어 있었어요. 다음엔 찾을 수 있어!';
    $('result-time').textContent = '찾지 못했다…';
    $('gauge').style.width = '0%';
  }
  $('result-hints').textContent = playRun.hints
    ? '사용한 힌트 ' + '★'.repeat(playRun.hints)
    : '힌트 없이 도전!';
  const grade = computeGrade(statsNow, tl, playStage.eyedropperUses);
  $('grade-letter').textContent = grade.g;
  $('grade-msg').textContent = grade.msg;
  $('eyedropper-badge').classList.toggle('hidden', !(playStage.eyedropperUses > 0));
  playRun.grade = grade;
  setPhase('end');
  recordResult();
}
```
아래로 교체(`computeGrade()`·`recordResult()` 자체는 손대지 않고, 인형 수와 무관한 그대로 재사용한다):
```js
function finishPlay(cleared){
  if (!playRun || playRun.ended) return;
  playRun.ended = true;
  playRun.cleared = cleared;
  clearInterval(timerIv); timerIv = null;
  const tl = playStage.timeLimit;
  // 이번 플레이를 반영한 스테이지 stats (등급 표시용 — 서버 기록과 동일 계산)
  const statsNow = {
    plays: playStage.stats.plays + 1,
    clears: playStage.stats.clears + (cleared ? 1 : 0),
    totalClearMs: playStage.stats.totalClearMs + (cleared ? Math.round(performance.now() - playRun.start) : 0),
  };
  const eyedropperUsed = playStage.dolls ? mdAnyEyedropperUsed(playStage.dolls) : playStage.eyedropperUses > 0;
  if (cleared){
    playRun.clearMs = Math.round(performance.now() - playRun.start);
    state.found = true;
    renderStage();
    if (!playStage.dolls) spawnFoundDoll(); // 다중 인형은 각자 찾을 때 이미 연출됨
    if (navigator.vibrate) navigator.vibrate([60, 40, 120]);
    $('playend-title').textContent = '찾았다! 🎉';
    $('playend-sub').textContent = '';
    $('result-time').textContent = (playRun.clearMs / 1000).toFixed(1) + '초 만에 발견!';
    const remainRatio = Math.max(0, 1 - playRun.clearMs / (tl * 1000));
    $('gauge').style.width = (remainRatio * 100) + '%';
  } else {
    state.reveal = true;
    renderStage();
    if (playStage.dolls) mdSpawnRevealRings(); else spawnRevealRing();
    $('playend-title').textContent = '시간 종료! ⏰';
    $('playend-sub').textContent = '인형은 반짝이는 곳에 숨어 있었어요. 다음엔 찾을 수 있어!';
    $('result-time').textContent = '찾지 못했다…';
    $('gauge').style.width = '0%';
  }
  $('result-hints').textContent = playRun.hints
    ? '사용한 힌트 ' + '★'.repeat(playRun.hints)
    : '힌트 없이 도전!';
  const grade = computeGrade(statsNow, tl, eyedropperUsed ? 1 : 0);
  $('grade-letter').textContent = grade.g;
  $('grade-msg').textContent = grade.msg;
  $('eyedropper-badge').classList.toggle('hidden', !eyedropperUsed);
  playRun.grade = grade;
  setPhase('end');
  recordResult();
}
```

- [ ] **Step 9: 힌트 3종을 인형 개수만큼 사용 가능하도록 교체 + 다중 타겟팅 함수 추가**

현재:
```js
function useHint(btn){
  if (!playRun || playRun.ended) return false;
  if (btn.disabled) return false;
  btn.disabled = true;
  playRun.hints++;
  return true;
}
```
바로 다음에 다중 인형용 버전 추가:
```js
/* 다중 인형용: 힌트 종류당 최대 사용 횟수 = 인형 개수 (다 쓰면 버튼 비활성화) */
function mdUseHint(btn, key){
  if (!playRun || playRun.ended) return false;
  playRun.mdHintCounts = playRun.mdHintCounts || { quad: 0, temp: 0, flash: 0 };
  const max = playStage.dolls.length;
  if (playRun.mdHintCounts[key] >= max) return false;
  playRun.mdHintCounts[key]++;
  playRun.hints++;
  if (playRun.mdHintCounts[key] >= max) btn.disabled = true;
  return true;
}
```

`hintFlash()` 함수(힌트 3종 중 마지막) 다음에 다중 버전 3개 추가:
```js
function mdHintQuadrant(){
  if (!mdUseHint($('hint-quad'), 'quad')) return;
  const i = mdNearestUnfoundIndex();
  if (i === -1) return;
  const d = playStage.dolls[i];
  const v = document.createElement('div'); v.className = 'quad-line-v';
  const h = document.createElement('div'); h.className = 'quad-line-h';
  const hl = document.createElement('div'); hl.className = 'quad-hl';
  hl.style.left = (d.x < 0.5 ? '0' : '50%');
  hl.style.top  = (d.y < 0.5 ? '0' : '50%');
  overlay.appendChild(v); overlay.appendChild(h); overlay.appendChild(hl);
  setTimeout(() => { v.remove(); h.remove(); hl.remove(); }, 1500);
}
function mdHintTemp(){
  if (!mdUseHint($('hint-temp'), 'temp')) return;
  playRun.tempOn = true;
  $('play-status').textContent = '온도계 켜짐! 이제 누를 때마다 온도를 알려줘요';
}
function mdTempFeedback(nx, ny){
  const i = mdNearestUnfoundIndex();
  if (i === -1) return;
  const d = playStage.dolls[i];
  const dist = Math.hypot((nx - d.x) * state.photoW, (ny - d.y) * state.photoH) / minSide();
  for (const step of CONFIG.TEMP_STEPS){
    if (dist <= step.max){
      $('play-status').textContent = step.msg;
      if (navigator.vibrate) navigator.vibrate(step.vib);
      return;
    }
  }
}
function mdHintFlash(){
  if (!mdUseHint($('hint-flash'), 'flash')) return;
  const i = mdNearestUnfoundIndex();
  if (i === -1) return;
  const d = playStage.dolls[i];
  const s = mdDollSize(d);
  const k = s / CONFIG.DOLL_SIZE;
  pctx2d.save();
  pctx2d.translate(d.x * state.photoW, d.y * state.photoH);
  pctx2d.rotate(d.rotation * Math.PI / 180);
  if (d.flip) pctx2d.scale(-1, 1);
  pctx2d.scale(k, k);
  pctx2d.translate(-CONFIG.DOLL_SIZE / 2, -CONFIG.DOLL_SIZE / 2);
  pctx2d.lineWidth = 7 / k;
  pctx2d.strokeStyle = '#E45B5B';
  pctx2d.lineJoin = 'round';
  pctx2d.stroke(dollPath);
  pctx2d.restore();
  setTimeout(renderStage, CONFIG.FLASH_MS);
}
```

힌트 버튼 배선의 현재 코드:
```js
$('hint-quad').addEventListener('click', hintQuadrant);
$('hint-temp').addEventListener('click', hintTemp);
$('hint-flash').addEventListener('click', hintFlash);
```
아래로 교체:
```js
$('hint-quad').addEventListener('click', () => { (playStage && playStage.dolls ? mdHintQuadrant : hintQuadrant)(); });
$('hint-temp').addEventListener('click', () => { (playStage && playStage.dolls ? mdHintTemp : hintTemp)(); });
$('hint-flash').addEventListener('click', () => { (playStage && playStage.dolls ? mdHintFlash : hintFlash)(); });
```

- [ ] **Step 10: 브라우저 QA — 실제 저장 후 원격 플레이 전체 흐름 확인**

이 태스크는 실제 Firestore 저장이 필요하다(원격 로드 경로 자체를 검증해야 하므로). QA 스크립트에서 실제로 스테이지를 저장하고 그 `?s=` 링크로 재방문해 플레이한 뒤, **테스트가 끝나면 삭제키로 정리**한다(운영 Firestore에 QA용 문서가 남지 않도록):

`drive.js`에 `md-task6` 시나리오 추가:
```js
    } else if (scenario === 'md-task6'){
      await toPaintScreen(page);
      await page.click('#btn-add-doll-paint');
      await page.waitForSelector('#place-panel:not(.hidden)', { timeout: 5000 });
      const stageBox = await page.locator('#photoCanvas').boundingBox();
      await page.mouse.move(stageBox.x + stageBox.width * 0.5, stageBox.y + stageBox.height * 0.5);
      await page.mouse.down();
      await page.mouse.move(stageBox.x + stageBox.width * 0.8, stageBox.y + stageBox.height * 0.2, { steps: 5 });
      await page.mouse.up();
      await page.click('#btn-to-paint');
      await page.waitForSelector('#paint-panel:not(.hidden)', { timeout: 5000 });
      await page.click('#btn-save');
      await page.waitForSelector('#share-panel:not(.hidden)', { timeout: 20000 });
      const shareUrl = await page.inputValue('#share-url');
      const deleteKey = await page.textContent('#delete-key');
      const inviteMsg = await page.textContent('#share-invite-msg');
      console.log('share url:', shareUrl);
      console.log('invite message (2 dolls):', inviteMsg);
      await page.screenshot({ path: shotPath('md-task6-share-panel') });

      // 새 페이지로 그 링크를 열어 실제 원격 플레이 진행
      const page2 = await browser.newPage({ viewport: { width: 390, height: 844 } });
      const errors2 = [];
      page2.on('pageerror', (e) => errors2.push(e.message));
      await page2.goto(shareUrl);
      await page2.waitForSelector('#play-panel:not(.hidden)', { timeout: 15000 });
      await page2.waitForTimeout(3500); // 카운트다운(3-2-1) 대기
      await page2.screenshot({ path: shotPath('md-task6-play-started') });
      const photoBox2 = await page2.locator('#photoCanvas').boundingBox();
      // 인형1(중앙) 탭
      await page2.mouse.click(photoBox2.x + photoBox2.width * 0.5, photoBox2.y + photoBox2.height * 0.5);
      await page2.waitForTimeout(300);
      const statusAfter1 = await page2.textContent('#play-status');
      console.log('play status after finding 1 of 2:', statusAfter1);
      // 힌트(사분면) 사용 — 남은 인형 위치 힌트
      await page2.click('#hint-quad');
      await page2.waitForTimeout(300);
      await page2.screenshot({ path: shotPath('md-task6-hint-quad') });
      // 인형2(오른쪽 위) 탭 → 클리어
      await page2.mouse.click(photoBox2.x + photoBox2.width * 0.8, photoBox2.y + photoBox2.height * 0.2);
      await page2.waitForTimeout(500);
      await page2.waitForSelector('#playend-panel:not(.hidden)', { timeout: 5000 });
      const grade = await page2.textContent('#grade-letter');
      const resultTime = await page2.textContent('#result-time');
      console.log('grade letter:', grade, '| result time:', resultTime);
      await page2.screenshot({ path: shotPath('md-task6-cleared') });
      console.log('page2 errors:', errors2.length ? JSON.stringify(errors2) : 'none');
      await page2.close();

      // 정리: 방금 만든 QA용 스테이지 삭제
      await page.click('#btn-open-delete-share');
      await page.waitForSelector('#delete-panel:not(.hidden)', { timeout: 5000 });
      await page.fill('#delete-input', deleteKey.trim());
      page.once('dialog', (d) => d.accept());
      await page.click('#btn-delete-confirm');
      await page.waitForTimeout(1000);
      console.log('cleanup delete attempted for QA stage');
    }
```
실행: `node drive.js md-task6` (실제 네트워크·Firestore를 쓰므로 다른 시나리오보다 오래 걸릴 수 있다. 타임아웃이 걸리면 재시도한다)

확인 사항:
- `invite message (2 dolls)`가 "숨은 인형 2마리를 찾아라!"
- `md-task6-play-started.png`: 원격 플레이 화면에 인형 2개 모두 보임(둘 다 아직 미발견)
- `play status after finding 1 of 2`가 "1/2 찾음"
- `md-task6-hint-quad.png`: 남은 인형이 있는 사분면이 하이라이트됨
- `md-task6-cleared.png`: 클리어 화면, `grade letter`가 알파벳 한 글자(`?`가 아니어야 함 — 이번 플레이 자체가 `stats.clears`를 1로 만들었으므로), `result time`에 초 단위 시간 표시
- `page2 errors: none`
- 콘솔에 "cleanup delete attempted for QA stage" 출력(삭제 요청이 실제로 전송됐는지 — 실패해도 QA 자체 목적에는 지장 없으나, 운영 데이터 정리를 위해 반드시 시도한다)

이어서 `md-task1`~`md-task5`, `task1`~`task3`, `outline`, `colorpicker-*` 전체 회귀 재확인.

- [ ] **Step 11: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
원격 플레이를 dolls 배열 스키마로 확장(로드·판정·힌트·연출)

fetchStage/loadStage가 문서의 dolls 배열 유무로 분기해 다중 인형
이미지를 로드한다. 재생 중 탭마다 미발견 인형 전부를 판정하고, 힌트
3종은 가장 최근 탭에서 가장 가까운 미발견 인형을 대상으로 삼으며
종류당 인형 수만큼 쓸 수 있다. 기존 단일 인형(doll 필드) 원격 플레이
경로는 완전히 그대로 남겨 레거시 공유 링크가 계속 동작한다.
EOF
)"
```

---

### Task 7: 결과 화면·제작자 결과 보기·갤러리 마무리

**Files:**
- Modify: `index.html` — `loadResultView()`
- Modify: `index.html` — `renderGallery()`
- Modify: `index.html` — CSS(갤러리 인형 개수 뱃지)

**Interfaces:**
- Consumes: Task 6의 `mdAnyEyedropperUsed`, 기존 `computeGrade`

- [ ] **Step 1: `loadResultView()`에 `dolls`/`doll` 분기 추가**

현재:
```js
    const d = got.data;
    playStage = { id, ref: got.ref, doll: d.doll, timeLimit: d.timeLimit || CONFIG.TIME_LIMIT, stats: d.stats, eyedropperUses: d.eyedropperUses || 0 };
    resultTaps = d.taps || [];
    const st = d.stats || { plays: 0, clears: 0, totalClearMs: 0 };
    $('rv-plays').textContent = st.plays;
    $('rv-clears').textContent = st.clears;
    $('rv-avg').textContent = st.clears ? (st.totalClearMs / st.clears / 1000).toFixed(1) + '초' : '--';
    const grade = computeGrade(st, playStage.timeLimit, playStage.eyedropperUses);
    $('rv-grade').textContent = grade.g;
    $('rv-grade-msg').textContent = grade.msg;
    $('rv-eyedropper-badge').classList.toggle('hidden', !(playStage.eyedropperUses > 0));
    setPhase('view');
```
아래로 교체:
```js
    const d = got.data;
    if (d.dolls){
      playStage = { id, ref: got.ref, dolls: d.dolls, timeLimit: d.timeLimit || CONFIG.TIME_LIMIT, stats: d.stats };
    } else {
      playStage = { id, ref: got.ref, doll: d.doll, timeLimit: d.timeLimit || CONFIG.TIME_LIMIT, stats: d.stats, eyedropperUses: d.eyedropperUses || 0 };
    }
    resultTaps = d.taps || [];
    const st = d.stats || { plays: 0, clears: 0, totalClearMs: 0 };
    $('rv-plays').textContent = st.plays;
    $('rv-clears').textContent = st.clears;
    $('rv-avg').textContent = st.clears ? (st.totalClearMs / st.clears / 1000).toFixed(1) + '초' : '--';
    const eyedropperUsed = playStage.dolls ? mdAnyEyedropperUsed(playStage.dolls) : playStage.eyedropperUses > 0;
    const grade = computeGrade(st, playStage.timeLimit, eyedropperUsed ? 1 : 0);
    $('rv-grade').textContent = grade.g;
    $('rv-grade-msg').textContent = grade.msg;
    $('rv-eyedropper-badge').classList.toggle('hidden', !eyedropperUsed);
    setPhase('view');
```

(`renderStage()`의 `state.phase === 'view'` 처리는 Task 6에서 이미 `playStage.dolls` 유무로 분기되어 있으므로, 여기서는 `playStage` 객체 구성만 맞추면 나머지는 자동으로 맞물린다. `?s={id}&view=result`로 진입 시 인형 위치가 화면에 표시되려면 `photoCanvas`가 그려질 수 있도록 `renderStage()`가 호출돼야 하는데, 이는 기존에도 `setPhase('view')` 안에서 `renderStage()`가 호출되므로 무변경으로 계속 동작한다)

- [ ] **Step 2: `renderGallery()`에 다중 인형 뱃지·등급 계산 분기 추가**

현재:
```js
      const d = snap.data();
      const grade = computeGrade(d.stats, d.timeLimit || CONFIG.TIME_LIMIT, d.eyedropperUses);
      const card = document.createElement('button');
      card.className = 'g-card';
      card.setAttribute('aria-label', '예제 스테이지 플레이');
      // 썸네일은 강한 블러 처리 — 정답 위치·사진 내용 노출 방지
      card.innerHTML =
        '<div class="g-thumb"><img alt="" loading="lazy"></div>' +
        '<div class="g-meta"><span>도전 ' + (d.stats ? d.stats.plays : 0) + '</span>' +
        '<span class="g-grade">' + grade.g + '</span></div>';
```
아래로 교체:
```js
      const d = snap.data();
      const eyedropperUsed = d.dolls ? mdAnyEyedropperUsed(d.dolls) : d.eyedropperUses > 0;
      const grade = computeGrade(d.stats, d.timeLimit || CONFIG.TIME_LIMIT, eyedropperUsed ? 1 : 0);
      const dollCount = d.dolls ? d.dolls.length : 1;
      const card = document.createElement('button');
      card.className = 'g-card';
      card.setAttribute('aria-label', '예제 스테이지 플레이');
      // 썸네일은 강한 블러 처리 — 정답 위치·사진 내용 노출 방지
      card.innerHTML =
        '<div class="g-thumb"><img alt="" loading="lazy"></div>' +
        '<div class="g-meta"><span>도전 ' + (d.stats ? d.stats.plays : 0) + '</span>' +
        (dollCount > 1 ? '<span class="g-count">🧍×' + dollCount + '</span>' : '') +
        '<span class="g-grade">' + grade.g + '</span></div>';
```

CSS의 `.g-grade{...}` 규칙 다음 줄에 추가:
```css
.g-count{ font-size:13px; color:var(--pencil-soft); }
```

- [ ] **Step 3: 브라우저 QA — 결과 보기·갤러리 표시 확인**

`drive.js`에 `md-task7` 시나리오 추가(Task 6과 유사하게 실제 저장 후 정리하되, 이번엔 결과 보기 링크를 확인):
```js
    } else if (scenario === 'md-task7'){
      await toPaintScreen(page);
      await page.click('#btn-add-doll-paint');
      await page.waitForSelector('#place-panel:not(.hidden)', { timeout: 5000 });
      await page.click('#btn-to-paint');
      await page.waitForSelector('#paint-panel:not(.hidden)', { timeout: 5000 });
      await page.click('#btn-save');
      await page.waitForSelector('#share-panel:not(.hidden)', { timeout: 20000 });
      const resultUrl = await page.inputValue('#result-url');
      const deleteKey = await page.textContent('#delete-key');

      const page2 = await browser.newPage({ viewport: { width: 390, height: 844 } });
      const errors2 = [];
      page2.on('pageerror', (e) => errors2.push(e.message));
      await page2.goto(resultUrl);
      await page2.waitForSelector('#result-panel:not(.hidden)', { timeout: 15000 });
      const rvGrade = await page2.textContent('#rv-grade');
      const rvClears = await page2.textContent('#rv-clears');
      console.log('result-view grade (no plays yet):', rvGrade, '| clears:', rvClears);
      await page2.screenshot({ path: shotPath('md-task7-result-view') });
      console.log('page2 errors:', errors2.length ? JSON.stringify(errors2) : 'none');
      await page2.close();

      await page.click('#btn-open-delete-share');
      await page.waitForSelector('#delete-panel:not(.hidden)', { timeout: 5000 });
      await page.fill('#delete-input', deleteKey.trim());
      page.once('dialog', (d) => d.accept());
      await page.click('#btn-delete-confirm');
      await page.waitForTimeout(1000);
      console.log('cleanup delete attempted for QA stage');
    }
```
실행: `node drive.js md-task7`

확인: `rv-grade`가 `?`(미개봉 — 아직 아무도 플레이 안 함), `rv-clears`가 `0`, `md-task7-result-view.png`에서 인형 2개의 정답 위치가 사진 위에 모두 표시되는지, `page2 errors: none`.

이어서 전체 회귀: `md-task1`~`md-task6`, `task1`~`task3`, `outline`, `colorpicker-*`를 모두 재실행해 최종 확인한다.

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
제작자 결과 보기·홈 갤러리를 dolls 배열 스키마까지 지원

결과 보기 로드가 dolls/doll 스키마를 분기 처리하고, 정직 배지·등급은
인형 배열 전체의 스포이드 사용 여부로 판단한다. 홈 예제 갤러리 카드에
인형이 2개 이상인 스테이지의 개수 뱃지를 추가.
EOF
)"
```
