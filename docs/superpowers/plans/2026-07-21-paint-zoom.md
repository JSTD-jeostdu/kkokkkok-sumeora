# 색칠 화면 확대 기능 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 색칠 화면(`paint-panel`)에서 종이인형 캔버스가 172px로 작게 표시되어 섬세한 색칠이 어려운 문제를, 전체화면 확대 모달로 해결한다.

**Architecture:** 기존 `dollView` 캔버스와 도구 모음(`tools` div)을 복제하지 않고 DOM에서 전체화면 모달 안으로 이동(`appendChild`)시켰다가 닫을 때 원래 자리로 되돌린다. 그림 데이터(`paintCtx`)·undo 스택·이벤트 리스너·버튼 상태가 전부 그대로 유지되므로 별도 동기화 코드가 필요 없다. CSS는 컨텍스트 선택자(`.zoom-modal .doll-view-wrap`)로 캔버스 표시 크기만 오버라이드한다.

**Tech Stack:** 순수 HTML/CSS/JS (`index.html` 단일 파일), 빌드 도구·프레임워크·외부 JS 라이브러리 없음.

## Global Constraints

- 단일 파일 `index.html`만 수정 — 새 파일 생성 금지
- 주석은 한국어
- 탭 영역(버튼) 44px 이상 유지
- `prefers-reduced-motion` 존중 (이 기능은 애니메이션을 추가하지 않아 자동으로 충족됨)
- 사진과 인형은 합성하지 않는다는 기존 원칙과 무관 — 이 기능은 순수 UI 표시 크기 변경
- **프로젝트에 자동 테스트 프레임워크가 없음** — 각 태스크의 검증은 브라우저에서 `index.html`을 직접 열어 수동으로 확인한다 (더블클릭으로 열거나 로컬 정적 서버 사용, Firebase 설정 없이도 제작·색칠 기능은 오프라인으로 동작함)
- `CONFIG` 객체 변경 없음 — 이 기능은 CSS 레이아웃 수치이며, 기존에도 레이아웃 수치는 `<style>`에 직접 기술하는 패턴을 따름 (예: `.doll-view-wrap{width:172px}`)

참고 스펙: `docs/superpowers/specs/2026-07-21-paint-zoom-design.md`

---

### Task 1: 확대 트리거 버튼 + 모달 마크업 + CSS

**Files:**
- Modify: `index.html:176-178` (CSS — 색칠 도구 관련 스타일 블록 뒤에 새 규칙 추가)
- Modify: `index.html:317` (HTML — `paint-panel` 안 `hint-line` 다음에 트리거 버튼 추가)
- Modify: `index.html:468-469` (HTML — `</section>` 다음, `</main>` 앞에 모달 마크업 추가)

**Interfaces:**
- Produces: `#btn-zoom-paint`(트리거 버튼), `#zoom-modal`(모달 컨테이너, 기본 `hidden`), `#btn-zoom-close`(닫기 버튼), `#zoom-canvas-slot`/`#zoom-tools-slot`(Task 2에서 JS가 엘리먼트를 옮겨 넣을 빈 슬롯) — 이후 태스크가 이 ID들을 그대로 사용한다.

- [ ] **Step 1: CSS 규칙 추가**

`index.html`에서 `.brush-dot{ display:inline-block; ... }` 규칙(현재 176번째 줄) 바로 다음, `.panel-actions{ display:flex; ... }` 규칙 앞에 아래 CSS를 추가한다:

```css
/* ---------- 색칠 확대 모달 ---------- */
.zoom-modal{
  position:fixed; inset:0; z-index:50;
  background:var(--paper);
  display:flex; flex-direction:column; align-items:center;
  padding:16px; gap:12px;
}
#btn-zoom-close{
  align-self:flex-end;
  font-family:inherit; font-size:22px; font-weight:700;
  border:2.5px solid var(--pencil); border-radius:50%;
  background:#fff; color:var(--pencil);
  width:44px; height:44px; min-width:44px; min-height:44px;
  cursor:pointer; box-shadow:var(--shadow);
}
#btn-zoom-close:focus-visible{ outline:3px solid var(--accent); outline-offset:2px; }
#zoom-canvas-slot{ display:flex; justify-content:center; }
.zoom-modal .doll-view-wrap{ width:min(90vw, 58vh); height:min(90vw, 58vh); }
#zoom-tools-slot{ width:100%; max-width:360px; }
```

`.zoom-modal`은 `class="zoom-modal hidden"`으로 시작하므로, 이미 파일 최상단에 있는 전역 규칙 `.hidden{ display:none !important; }`(45번째 줄)이 `display:flex`를 덮어써 기본적으로 숨겨진다. 별도의 `.zoom-modal.hidden` 규칙은 필요 없다.

- [ ] **Step 2: 트리거 버튼 마크업 추가**

`index.html`에서 색칠 패널의 `hint-line` 다음 줄(현재 317번째 줄, `<div class="paint-row">` 바로 위)에 버튼을 추가한다:

```html
      <p class="hint-line">붓으로 <b>감으로 색을 골라</b> 칠해보세요</p>
      <button class="btn btn-ghost btn-wide" id="btn-zoom-paint">🔍 확대해서 칠하기</button>
      <div class="paint-row">
```

- [ ] **Step 3: 모달 마크업 추가**

`index.html`에서 `screen-maker`의 `</section>`(현재 468번째 줄) 다음, `</main>`(현재 469번째 줄) 앞에 추가한다:

```html
  </section>

  <!-- 색칠 확대 모달: dollView 캔버스와 도구 모음을 JS로 이 안에 옮겨 넣는다 (Task 2) -->
  <div id="zoom-modal" class="zoom-modal hidden" role="dialog" aria-modal="true" aria-label="확대 색칠">
    <button id="btn-zoom-close" aria-label="확대 닫기">✕</button>
    <div id="zoom-canvas-slot"></div>
    <div id="zoom-tools-slot"></div>
  </div>
</main>
```

- [ ] **Step 4: 브라우저에서 정적 렌더링 확인**

`index.html`을 브라우저로 연다(더블클릭). 개발자 도구 콘솔에서 다음을 실행한다:

```js
document.getElementById('zoom-modal').classList.remove('hidden')
```

Expected: 화면 전체를 도화지색(#FAF7F0)으로 덮는 오버레이가 나타나고, 우상단에 원형 ✕ 버튼이 보인다. (이 시점엔 캔버스가 아직 안에 없으므로 빈 화면이어도 정상 — Task 2에서 채워짐)

콘솔에서 다시 숨긴다:

```js
document.getElementById('zoom-modal').classList.add('hidden')
```

Expected: 오버레이가 사라지고 원래 화면으로 돌아온다.

이어서 홈 → 사진 고르기 → 배치 → "칠하러 가기"로 색칠 화면까지 이동해, `hint-line` 아래에 "🔍 확대해서 칠하기" 버튼이 정상 표시되는지(탭하면 아직 아무 반응 없어도 정상) 확인한다.

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
색칠 확대 모달 마크업·CSS 추가

전체화면 확대 모달의 정적 구조(트리거 버튼, 모달 컨테이너, 닫기 버튼,
캔버스/도구용 빈 슬롯)와 스타일을 추가. JS 동작은 다음 커밋에서 연결.
EOF
)"
```

---

### Task 2: 열기/닫기 핵심 로직 (캔버스·도구 이동)

**Files:**
- Modify: `index.html:1619-1621` (JS — 기존 "도구 버튼 배선" 블록과 "배치 컨트롤 배선" 블록 사이에 새 섹션 추가)

**Interfaces:**
- Consumes: Task 1이 만든 `#btn-zoom-paint`, `#zoom-modal`, `#btn-zoom-close`, `#zoom-canvas-slot`, `#zoom-tools-slot`. 기존 전역 `$(id)` 헬퍼(`index.html:592`), 기존 `dollView` 캔버스 상수(`index.html:597`).
- Produces: `zoomOpen`(boolean 상태), `openZoom()`, `closeZoom()` — Task 3이 이 함수/변수를 확장한다.

- [ ] **Step 1: 열기/닫기 함수와 배선 추가**

`index.html`에서 "도구 버튼 배선" 블록의 마지막 줄(`$('btn-eyedropper').addEventListener(...)` 블록이 끝나는 지점, 현재 1619번째 줄) 바로 다음, "배치 컨트롤 배선" 주석(현재 1621번째 줄) 앞에 추가한다:

```js

/* ---------- 색칠 확대 모달 ---------- */
const zoomModal = $('zoom-modal');
const zoomCanvasSlot = $('zoom-canvas-slot'), zoomToolsSlot = $('zoom-tools-slot');
const dollViewWrap = dollView.parentElement;
const toolsDiv = document.querySelector('#paint-panel .tools');
const paintRow = document.querySelector('#paint-panel .paint-row');
let zoomOpen = false;
function openZoom(){
  zoomCanvasSlot.appendChild(dollViewWrap);
  zoomToolsSlot.appendChild(toolsDiv);
  zoomModal.classList.remove('hidden');
  zoomOpen = true;
}
function closeZoom(){
  if (!zoomOpen) return;
  paintRow.appendChild(dollViewWrap);
  paintRow.appendChild(toolsDiv);
  zoomModal.classList.add('hidden');
  zoomOpen = false;
}
$('btn-zoom-paint').addEventListener('click', openZoom);
$('btn-zoom-close').addEventListener('click', closeZoom);
```

엘리먼트를 복제하지 않고 `appendChild`로 그대로 옮기므로, `paintCtx`에 그려진 그림·`undoStack`·브러시/지우개 버튼의 `aria-pressed` 상태·스포이드 남은 횟수 표시가 모두 그대로 유지된다. `renderDollView()`는 `dollView`/`viewCtx`를 직접 참조하므로 캔버스가 DOM 어디로 옮겨지든 그대로 동작해 수정할 필요가 없다.

- [ ] **Step 2: 브라우저에서 전체 흐름 수동 확인**

`index.html`을 열고: 홈 → 사진 고르기 → 인형 배치 → "칠하러 가기"로 색칠 화면 진입.

1. "🔍 확대해서 칠하기" 클릭 → 모달이 열리고 캔버스와 도구(색상 선택·붓 3종·지우개·되돌리기·전부지우기·스포이드)가 모달 안에 크게 표시되는지 확인
2. 확대된 캔버스에 붓으로 선을 그어본다 → 그려지는지 확인
3. 브러시 크기를 바꾸고, 색상을 바꾸고, 지우개를 켜서 지워본다 → 모두 정상 동작하는지 확인
4. 스포이드 버튼을 켜고 배경(사진 크롭) 위를 탭한다 → 확대된 캔버스 위 탭한 위치 근처에 색 팝업이 정확히 뜨는지, 색상이 스와치에 반영되는지 확인
5. 우상단 ✕ 클릭 → 모달이 닫히고 캔버스·도구가 원래 작은 화면 레이아웃으로 돌아오는지, 방금 그린 그림이 작은 캔버스에도 반영되어 있는지 확인
6. 다시 "🔍 확대해서 칠하기" 클릭 → 그림이 유지된 채 모달이 다시 열리는지, 이어서 그릴 수 있는지 확인
7. "되돌리기" 버튼이 확대 모드에서 그린 획도 정상적으로 취소하는지 확인

Expected: 위 7가지 모두 정상 동작. 콘솔에 에러 없음.

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
색칠 확대 모달 열기/닫기 핵심 로직 추가

기존 dollView 캔버스와 도구 모음을 복제 없이 모달 안팎으로 이동시켜
그림 데이터·undo 스택·버튼 상태를 그대로 공유하도록 구현.
EOF
)"
```

---

### Task 3: 폴리시 — Esc 닫기, 포커스 관리, 스크롤 잠금, 화면전환 안전장치

**Files:**
- Modify: `index.html` (Task 2에서 추가한 `openZoom`/`closeZoom` 함수 수정, 새 `keydown` 리스너 추가)
- Modify: `index.html:733-734` (JS — `setPhase(phase)` 함수 시작 부분에 안전장치 추가)

**Interfaces:**
- Consumes: Task 2의 `zoomOpen`, `openZoom()`, `closeZoom()`, `$('btn-zoom-close')`, `$('btn-zoom-paint')`.
- Produces: 없음 (최종 사용자 동작 폴리시)

- [ ] **Step 1: `openZoom`/`closeZoom`에 스크롤 잠금 + 포커스 이동 추가**

Task 2에서 추가한 `openZoom`/`closeZoom` 함수를 아래 내용으로 교체한다:

```js
function openZoom(){
  zoomCanvasSlot.appendChild(dollViewWrap);
  zoomToolsSlot.appendChild(toolsDiv);
  zoomModal.classList.remove('hidden');
  document.body.style.overflow = 'hidden';
  zoomOpen = true;
  $('btn-zoom-close').focus();
}
function closeZoom(){
  if (!zoomOpen) return;
  paintRow.appendChild(dollViewWrap);
  paintRow.appendChild(toolsDiv);
  zoomModal.classList.add('hidden');
  document.body.style.overflow = '';
  zoomOpen = false;
  $('btn-zoom-paint').focus();
}
```

- [ ] **Step 2: `Esc` 키로 닫기 추가**

같은 위치(`$('btn-zoom-close').addEventListener('click', closeZoom);` 다음 줄)에 추가한다:

```js
document.addEventListener('keydown', (e) => {
  if (zoomOpen && e.key === 'Escape') closeZoom();
});
```

- [ ] **Step 3: 화면 전환 안전장치 추가**

`index.html`의 `setPhase(phase)` 함수(현재 733번째 줄) 시작 부분을 아래처럼 수정한다:

```js
function setPhase(phase){
  if (zoomOpen) closeZoom();
  state.phase = phase;
```

`zoomOpen`과 `closeZoom`은 스크립트 하단(Task 2)에서 선언되지만, `setPhase`는 사용자 액션이나 `boot()`(스크립트 맨 마지막 줄에서 호출)을 통해서만 실제로 호출되므로 — 즉 전체 스크립트가 한 번 다 실행된 뒤에 처음 호출되므로 — 참조 시점에는 이미 두 값 모두 초기화되어 있어 문제없다.

- [ ] **Step 4: 브라우저에서 수동 확인**

`index.html`을 열고 색칠 화면까지 이동:

1. 확대 모달을 열고 `Esc` 키를 누른다 → 모달이 닫히는지 확인 (데스크톱 브라우저 필요)
2. 확대 모달을 열었을 때 포커스가 ✕ 버튼으로 이동하는지(Tab 키로 바로 다른 도구로 이동 가능한지) 확인
3. 모달을 닫았을 때 포커스가 "🔍 확대해서 칠하기" 버튼으로 돌아오는지 확인
4. 확대 모달이 열린 상태에서 배경 페이지가 스크롤되지 않는지 확인 (모바일 뷰포트 또는 브라우저 창을 작게 줄여서 스크롤 시도)
5. 확대 모달을 연 채로 "다시 배치" 버튼을 눌러 배치 화면으로 이동해본다 → 모달이 자동으로 닫히고 정상적으로 배치 화면이 보이는지, 캔버스/도구가 원래 자리로 복귀했는지 확인 (콘솔 에러 없어야 함)

Expected: 위 5가지 모두 정상 동작. 콘솔에 에러 없음.

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
색칠 확대 모달에 Esc 닫기·포커스 관리·스크롤 잠금·전환 안전장치 추가

Esc 키로도 닫히게 하고, 열고 닫을 때 포커스를 적절히 이동시키며,
모달이 열린 동안 배경 스크롤을 막는다. 색칠 화면을 벗어나는 전환이
발생하면 모달을 먼저 닫아 엘리먼트가 잘못된 위치에 남지 않게 한다.
EOF
)"
```
