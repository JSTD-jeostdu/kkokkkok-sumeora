# 스포이드 루페 + 결과 화면 인형 원본 공개 설계

> 작성일: 2026-07-23
> 관련 백로그: 없음(신규). Stage 3 마감 이후 사용성/재미 개선 항목 2건.

이 문서는 독립적인 두 기능을 함께 다룬다. 서로 코드 영역이 겹치지 않으므로(색칠 화면 vs 결과 화면) 구현은 순서대로 하나씩 진행해도 무방하다.

---

## 기능 A — 스포이드 확대 루페

### 배경 / 문제

현재 스포이드(`pickColorFromCrop`)는 `dollView` 캔버스를 `pointerdown`하는 즉시 `bgCropCtx.getImageData(px, py, 1, 1)`로 색을 읽어 바로 확정한다. `dollView`는 내부 해상도 256×256이지만 화면 표시 크기는 172px(확대 모달에서는 더 크지만 여전히 손가락보다 작은 픽셀 단위)라, 사용자가 정확히 원하는 픽셀을 짚었는지 확인할 방법이 없다. 스포이드는 인형당 3회로 제한된 자원이라(`CONFIG.EYEDROPPER_MAX_USES`), 잘못 짚으면 되돌릴 수 없이 소모된다.

### 목표

- 스포이드 사용 중 어떤 색이 뽑힐지 손가락으로 가리는 부분 없이 실시간으로 미리보기
- 마음에 안 들면 확정 전에 취소 가능 (사용 횟수 소모 없이)
- 기존 붓칠·되돌리기·확대 모달(`zoom-modal`) 로직과 충돌 없이 동작

### 비목표

- 핀치줌 등 자유 배율 조정
- 루페에 인형 실루엣 바깥(사진 전체) 탐색 기능 추가 — 스포이드는 여전히 `bgCropCanvas`(인형 주변 크롭)로 범위 한정
- 데스크톱 마우스 호버만으로 루페 표시(요구사항 아님, pointerdown 이후만 표시)

### 설계

#### 1. 인터랙션 흐름 변경

기존: `pointerdown` → 즉시 확정.
변경 후:

1. `eyedropperOn`이 `true`인 상태에서 `dollView`에 `pointerdown` → 드래그 모드 시작, 루페 표시, `pointerCapture` 설정
2. `pointermove`마다 현재 좌표의 픽셀색을 읽어 루페 내용 갱신 (아직 `state.color`/`eyedropperUses` 변경 없음)
3. `pointerup` 시점 좌표가 `dollView` 영역 **안**이면 확정(기존 `pickColorFromCrop`과 동일한 부수효과 적용) → 루페 숨김 → `eyedropperOn = false`
4. `pointerup` 시점 좌표가 `dollView` 영역 **밖**, 또는 `pointercancel` → 아무 것도 확정하지 않고 루페만 숨김. `eyedropperOn`은 유지(재시도 가능)

"영역 안/밖" 판정은 `viewPos()`로 구한 정규화 좌표(0~256 스케일)를 0~256 범위와 비교해 판단한다(범위 밖이면 밖).

#### 2. 화면/DOM 구성

**루페 마크업** (`stage`/`overlay` 레벨이 아니라 `body` 최상단, 다른 모달과 독립적으로 뜨도록 `paint-panel` 밖에 배치):

```html
<div id="loupe" class="loupe hidden" aria-hidden="true">
  <canvas id="loupe-canvas" width="112" height="112"></canvas>
  <div id="loupe-swatch"></div>
  <span id="loupe-hex"></span>
</div>
```

**CSS** (신규):
- `.loupe`: `position:fixed; z-index:60; pointer-events:none;` (터치 이벤트를 가리지 않도록) `transform:translate(-50%,-100%)`로 기준점이 원 하단 중앙이 되게 함
- `#loupe-canvas`: `border-radius:50%; border:3px solid var(--pencil); box-shadow:var(--shadow); image-rendering:pixelated;`
- 크로스헤어는 캔버스에 직접 그림(아래 로직 참고)
- `#loupe-swatch`: 작은 원(지름 20px), 캔버스 하단 중앙에 겹쳐 배치, 현재 색으로 배경 채움
- `#loupe-hex`: 스와치 옆 작은 라벨, HEX 코드 텍스트

#### 3. 루페 렌더링 로직 (JS)

```js
const LOUPE_ZOOM = 5;          // 확대 배율
const LOUPE_SRC = 22;          // 원본에서 읽어올 정사각형 크기(px, bgCropCanvas 기준)
const loupeEl = $('loupe'), loupeCanvas = $('loupe-canvas'), loupeCtx = loupeCanvas.getContext('2d');
loupeCtx.imageSmoothingEnabled = false;
let eyedropperDragging = false;
let eyedropperLastInside = false;
let lastLoupeView = null; // 마지막 pointer 이벤트의 viewPos(e) 결과 — pointerup 확정 시 pickColorFromCrop에 그대로 전달

// v: viewPos(e) 결과({x,y,cssX,cssY}, dollView 기준 0~256/오프셋 좌표 — pickColorFromCrop 인자로 그대로 씀)
// e: 원본 포인터 이벤트 — clientX/Y(뷰포트 절대 좌표)만 루페 위치 계산에 사용
function updateLoupe(v, e){
  lastLoupeView = v;
  const { x: vx, y: vy } = v;
  const inside = vx >= 0 && vx < CONFIG.DOLL_SIZE && vy >= 0 && vy < CONFIG.DOLL_SIZE;
  eyedropperLastInside = inside;
  const px = Math.min(CONFIG.DOLL_SIZE - 1, Math.max(0, Math.floor(vx)));
  const py = Math.min(CONFIG.DOLL_SIZE - 1, Math.max(0, Math.floor(vy)));
  const half = LOUPE_SRC / 2;
  loupeCtx.clearRect(0, 0, 112, 112);
  loupeCtx.save();
  loupeCtx.beginPath(); loupeCtx.arc(56, 56, 56, 0, Math.PI * 2); loupeCtx.clip();
  loupeCtx.drawImage(bgCropCanvas, px - half, py - half, LOUPE_SRC, LOUPE_SRC, 0, 0, 112, 112);
  loupeCtx.restore();
  // 크로스헤어
  loupeCtx.strokeStyle = inside ? '#4A4A4A' : '#E45B5B';
  loupeCtx.lineWidth = 2;
  loupeCtx.strokeRect(56 - LOUPE_ZOOM/2*2, 56 - LOUPE_ZOOM/2*2, LOUPE_ZOOM*2, LOUPE_ZOOM*2);
  // 색 읽기 (경계 밖이면 클램프된 값 그대로 미리보기 용으로만 사용, 확정 안 함)
  const d = bgCropCtx.getImageData(px, py, 1, 1).data;
  const hex = '#' + toHex(d[0]) + toHex(d[1]) + toHex(d[2]);
  $('loupe-swatch').style.background = hex;
  $('loupe-hex').textContent = inside ? hex : '취소';
  loupeEl.style.left = e.clientX + 'px';       // 뷰포트 절대 좌표 (position:fixed 기준)
  loupeEl.style.top = (e.clientY - 90) + 'px';
}
```

(정확한 크로스헤어 셀 좌표 등 세부 상수는 구현 시 시각적으로 다듬는다 — 위 코드는 로직 골격.)

#### 4. 이벤트 리스너 재구성

기존 `dollView`의 `pointerdown`/`pointermove`/`pointerup`/`pointercancel` 리스너를 다음과 같이 분기한다.

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
  // ...기존 붓칠 시작 로직 그대로...
});

dollView.addEventListener('pointermove', (e) => {
  if (eyedropperDragging){
    updateLoupe(viewPos(e), e);
    return;
  }
  if (!painting) return;
  // ...기존 붓칠 로직 그대로...
});

function endEyedropperDrag(commit){
  eyedropperDragging = false;
  loupeEl.classList.add('hidden');
  if (commit && eyedropperLastInside){
    const v = lastLoupeView;
    pickColorFromCrop(v.x, v.y, v.cssX, v.cssY); // 확정 + eyedropperOn=false는 함수 내부에서 처리
  }
}
dollView.addEventListener('pointerup', (e) => {
  if (eyedropperDragging){ endEyedropperDrag(true); return; }
  endStroke(); // 기존
});
dollView.addEventListener('pointercancel', (e) => {
  if (eyedropperDragging){ endEyedropperDrag(false); return; }
  endStroke(); // 기존
});
```

`pointerup`은 자체 좌표로 `viewPos(e)`를 다시 계산해도 되지만, 마지막 `pointermove`가 이미 `lastLoupeView`에 저장해 둔 값과 동일하므로(캡처된 포인터라 `pointerup` 직전에 `pointermove`가 항상 먼저 온다) 재계산 없이 재사용한다.

기존 `pickColorFromCrop` 함수는 **수정 없이 그대로 재사용**(이미 확정 시 필요한 모든 부수효과—색 적용, 사용횟수 증가, 스와치 갱신, 팝업 애니메이션, `eyedropperOn=false`—를 포함하고 있음).

#### 5. 확대 모달과의 관계

`zoom-modal`은 `dollView` 엘리먼트 자체를 DOM 안에서 이동시킬 뿐이므로, `pointerdown` 등 이벤트 리스너와 `viewPos()`(`getBoundingClientRect()` 기반)는 모달 내부에서도 그대로 정확히 동작한다. 루페는 `position:fixed` + `clientX/Y` 기준이라 모달 열림 여부와 무관하게 화면 어디서든 올바른 위치에 뜬다. 별도 처리 불필요.

### 예외 처리

- 캔버스 밖으로 나간 상태에서 루페: 크로스헤어를 빨간색으로 바꾸고 하단 라벨을 "취소"로 표시해 확정되지 않을 것임을 시각적으로 알림
- `EYEDROPPER_MAX_USES` 소진 상태(`updateEyedropperUI`가 `eyedropperOn=false`로 강제)에서는 애초에 `eyedropperOn`이 켜지지 않으므로 루페 로직 자체가 트리거되지 않음 — 기존 가드 그대로 유효
- `prefers-reduced-motion`: 루페는 트랜지션 애니메이션이 없으므로(즉시 표시/숨김) 별도 처리 불필요

### 테스트 계획 (수동)

- 스포이드 켜고 드래그 → 루페가 손가락 위쪽에 따라다니며 색이 실시간으로 바뀌는지 확인
- 캔버스 안에서 손 떼기 → 색 확정, 스와치 갱신, 사용 횟수 -1, 스포이드 자동 off
- 캔버스 밖으로 드래그해서 손 떼기 → 아무 것도 확정 안 됨, 사용 횟수 그대로, 스포이드 계속 켜진 상태로 재시도 가능
- 확대 모달 안에서도 동일하게 동작하는지 확인
- 스포이드 3회 소진 후 버튼 비활성화 상태에서 루페가 뜨지 않는지 확인
- 모바일 실기기(터치)에서 루페가 손가락에 가려지지 않는지 확인

---

## 기능 B — 결과 화면에 제작자 종이인형 원본 공개

### 배경 / 문제

플레이어가 스테이지를 플레이하면 인형을 찾을 때마다 사진 위에서 위장색이 입혀진 인형을 잠깐 보지만, 결과 화면(`playend-panel`)에는 인형 자체가 다시 표시되지 않는다. 제작자가 정성껏 고른 위장 색·무늬를 감상할 기회가 없다.

### 목표

- 플레이가 끝나면(클리어 성공/시간 종료 모두) `playend-panel`에 제작자가 만든 종이인형 원본(들)을 도화지색 카드에 담아 가로 스크롤로 보여준다
- 인형 1개든 다중 인형(최대 3개)이든 동일하게 동작
- 이미 로드되어 있는 인형 이미지(`remoteDollImgs`/`remoteDollImg`)를 그대로 재사용, 추가 네트워크 요청·데이터 모델 변경 없음

### 비목표

- 공유 카드(`buildShareCard`)에 인형 색 노출 — 기존 "위장색 미노출" 원칙 유지, 이번 변경 대상 아님
- 제작자 결과 보기 화면(`result-panel`)에 추가 — 제작자는 이미 자신의 인형을 알고 있으므로 불필요
- 로컬 테스트 플레이(`test-panel`) 대응 — 제작자 본인이 바로 만든 것을 다시 보여줄 이유 없음, 애초에 이 화면은 별도 플로우

### 설계

#### 1. HTML 마크업 추가

`playend-panel` 내 `#gauge-wrap`/`grade-msg` 다음, `eyedropper-badge` 앞에 삽입:

```html
<p class="hint-line">제작자가 만든 종이인형</p>
<div class="doll-reveal-row" id="doll-reveal-row"></div>
```

플레이가 끝나면 항상 렌더링되는 영역이라(클리어/시간종료 모두 표시) 별도 hidden 토글은 필요 없다.

#### 2. CSS 추가

```css
.doll-reveal-row{ display:flex; gap:10px; overflow-x:auto; padding:6px 2px 10px; }
.doll-reveal-card{ flex:0 0 auto; width:88px; height:88px; background:var(--paper);
  border:2px dashed var(--pencil-soft); border-radius:12px;
  display:flex; align-items:center; justify-content:center; }
.doll-reveal-card img{ width:72px; height:72px; object-fit:contain; }
```

`.cut-card`가 아닌 `.stat-cell`과 유사한 점선 상자 톤을 재사용해 결과 화면의 다른 카드류(`.stat-cell`)와 시각적으로 통일한다.

#### 3. 렌더링 로직 (JS)

```js
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

`finishPlay(cleared)` 함수 끝부분(`setPhase('end')` 호출 전 어디든, `cleared`/미클리어 분기 밖 공통 지점)에서 `renderDollReveal()`을 호출한다 — 클리어·시간종료 양쪽 모두 같은 지점을 지나므로 분기 처리 불필요.

#### 4. 기존 코드와의 관계

- `remoteDollImgs`/`remoteDollImg`는 `fetchStage()`에서 이미 로드되어 있으므로 새 네트워크 요청이나 Firestore 읽기가 추가되지 않는다
- 인형 이미지는 배경(사진)과 별도 레이어로 저장돼 있으므로(`dollB64`는 클리핑된 인형만, 투명 배경) 합성 없이 그대로 노출해도 "사진과 인형을 합성하지 않는다" 원칙에 위배되지 않는다
- 로컬 테스트 플레이 경로(`endStroke`, `testStatus` 등)는 전혀 건드리지 않는다

### 예외 처리

- `playStage`가 없거나(비정상 상태) `remoteDollImgs`/`remoteDollImg`가 비어있는 경우는 정상 흐름상 발생하지 않음(플레이가 시작됐다는 것 자체가 로드 완료를 의미) — 별도 방어 코드 불필요
- 레거시 단일 인형 포맷(`playStage.doll`)도 배열로 감싸(`[remoteDollImg]`) 동일 렌더링 경로를 타므로 포맷 분기 코드가 최소화됨

### 테스트 계획 (수동)

- 인형 1개 스테이지 클리어 → 카드 1개 표시, 이미지가 실제 칠한 색과 일치하는지 확인
- 인형 2~3개 스테이지 모두 찾고 클리어 → 카드가 개수만큼 가로로 나열, 스크롤 가능 여부(좁은 화면) 확인
- 시간 종료(못 찾음)로 끝난 경우에도 동일하게 카드가 표시되는지 확인
- 구버전(단일 인형 포맷) 스테이지에서도 카드 1개가 정상 표시되는지 확인
- 모바일 320px 폭에서 가로 스크롤 카드 목록이 레이아웃을 깨지 않는지 확인

## 영향 범위 (공통)

- `index.html` 단일 파일 내 CSS/HTML/JS 추가만 발생. 빌드 도구·의존성 변경 없음
- Firestore 데이터 모델, `firestore.rules` 변경 없음 — 두 기능 모두 기존에 로드/보유하고 있던 데이터만 다른 방식으로 표시하는 순수 클라이언트 UI 변경
- 기존 기능(붓칠, 되돌리기, 확대 모달, 공유 카드, 제작자 결과 보기, stats 기록)에 대한 회귀 없음 — 각 기능은 독립된 코드 경로에 추가되며 기존 함수는 그대로 재사용(`pickColorFromCrop`, `renderDollComposite` 등 무수정)
