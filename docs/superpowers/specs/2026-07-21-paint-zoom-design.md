# 색칠 화면 확대 기능 설계

> 작성일: 2026-07-21
> 관련 백로그: 없음(신규). Stage 3 마감 이후 사용성 개선 항목.

## 배경 / 문제

색칠 화면(`paint-panel`)의 `dollView` 캔버스는 내부 해상도가 256×256이지만, 실제 화면 표시 크기는 CSS로 **172×172px**에 불과하다. 게다가 칠하기 화면 배경 크롭 맥락을 보여주기 위해 인형 자체는 `DOLL_VIEW_SCALE`(`1 / CONFIG.PAINT_BG_CROP_RATIO` = 1/1.6 ≈ 0.625)만큼 더 축소되어 그려진다. 결과적으로 인형이 실제로 화면에 차지하는 크기는 약 107px에 불과해, 모바일 터치 입력으로 섬세한 색칠 작업(줄무늬, 무늬 등 디테일)을 하기가 매우 어렵다.

내부 해상도(256×256)는 이미 충분히 세밀하고, 포인터 좌표 계산(`viewPos()`)은 `getBoundingClientRect()` 기반이라 표시 크기와 무관하게 정확하다. 즉 **문제는 화면 표시 크기이지 해상도가 아니다** — 캔버스를 화면에 더 크게 보여주기만 해도 터치 정밀도 문제가 해결된다.

## 목표

- 인형을 화면 대부분을 채우는 크기로 확대해서 볼 수 있는 전체화면 모달 제공
- 배경(사진 크롭) 맥락도 함께 확대되어 보여, 위장색을 감으로 고르는 기존 흐름을 그대로 유지
- 기존 도구(붓 크기 3종, 지우개, 되돌리기, 전부지우기, 색상 선택, 스포이드)를 확대 모드에서도 전부 사용 가능
- 별도 캔버스 복제 없이 기존 `dollView`와 그림 데이터(`paintCtx`)·undo 스택을 그대로 공유

## 비목표

- 핀치 줌/팬 등 자유 배율 조정 (고정 확대 크기로 충분하다고 판단)
- 확대 모드 전용 브러시 크기 추가
- 배경 탭으로 모달 닫기 (오조작 방지 위해 X 버튼 전용)

## 설계

### 1. 인터랙션 흐름

1. 색칠 패널의 `hint-line` 바로 아래에 확대 진입 버튼 배치
2. 탭 → 전체화면 모달이 열리고, 기존 `dollView`(및 부모 wrap)와 도구 모음(`tools` div)이 모달 안으로 **이동**(복제 아님)
3. 모달 안 캔버스는 `min(90vw, 58vh)` 정사각형으로 확대 표시. 내부 해상도(256×256)는 변경 없음
4. 배경 크롭도 함께 확대되어 보이며, 스포이드는 그대로 동작
5. 우상단 X 버튼 또는 `Esc` 키로만 닫힘 (배경 탭으로는 닫히지 않음)
6. 닫으면 캔버스와 도구 모음이 원래 `paint-row` 자리로 복귀
7. 모달이 열려 있는 동안 `body` 스크롤 잠금

### 2. 화면/DOM 구성

**진입 버튼** (`paint-panel` 내, `hint-line` 다음):
```html
<button class="btn btn-ghost btn-wide" id="btn-zoom-paint">🔍 확대해서 칠하기</button>
```
기존 `.btn-ghost`(점선 테두리) 스타일을 재사용해 `btn-save`(주 액션)와 시각적으로 경쟁하지 않는 보조 버튼 톤 유지.

**모달 마크업** (패널 컨테이너 레벨, `paint-panel` 형제 요소):
```html
<div id="zoom-modal" class="zoom-modal hidden" role="dialog" aria-modal="true" aria-label="확대 색칠">
  <button id="btn-zoom-close" aria-label="확대 닫기">✕</button>
  <div id="zoom-canvas-slot"></div>
  <div id="zoom-tools-slot"></div>
</div>
```

**CSS**:
- `.zoom-modal`: `position:fixed; inset:0; z-index:50;` 도화지색(`--paper`, #FAF7F0) 배경, 세로 flex 배치(캔버스 상단, 도구 하단), `hidden`이면 `display:none`
- `.zoom-modal .doll-view-wrap{ width:min(90vw,58vh); height:min(90vw,58vh); }` — 기존 `.doll-view-wrap{width:172px;height:172px;}`를 컨텍스트 선택자로 오버라이드. 인라인 스타일 조작 없이 CSS 캐스케이딩만으로 크기 전환
- 열림/닫힘 트랜지션(예: opacity/scale)은 `prefers-reduced-motion: reduce`에서 생략

### 3. 동작 로직 (JS)

```js
const zoomModal = $('zoom-modal');
const zoomCanvasSlot = $('zoom-canvas-slot'), zoomToolsSlot = $('zoom-tools-slot');
const dollViewWrap = dollView.parentElement; // .doll-view-wrap
const toolsDiv = document.querySelector('#paint-panel .tools');
const paintRow = document.querySelector('#paint-panel .paint-row'); // 원위치 복귀용 부모
let zoomOpen = false;

function openZoom(){
  zoomCanvasSlot.appendChild(dollViewWrap);
  zoomToolsSlot.appendChild(toolsDiv);
  zoomModal.classList.remove('hidden');
  document.body.style.overflow = 'hidden';
  zoomOpen = true;
  $('btn-zoom-close').focus();
}
function closeZoom(){
  // paint-row 안 원래 위치로 복귀 (도구 slot이 canvas slot 뒤에 오도록 순서 보존)
  paintRow.appendChild(dollViewWrap);
  paintRow.appendChild(toolsDiv);
  zoomModal.classList.add('hidden');
  document.body.style.overflow = '';
  zoomOpen = false;
  $('btn-zoom-paint').focus();
}
$('btn-zoom-paint').addEventListener('click', openZoom);
$('btn-zoom-close').addEventListener('click', closeZoom);
document.addEventListener('keydown', (e) => { if (zoomOpen && e.key === 'Escape') closeZoom(); });
```

엘리먼트를 그대로 옮기므로(복제 아님) `paintCtx`에 그려지는 그림 데이터, `undoStack`, 브러시/지우개/색상 버튼의 `aria-pressed` 상태, 스포이드 남은 횟수 표시 등 **모든 상태와 이벤트 리스너가 자동으로 공유**된다. 별도 동기화 코드가 필요 없다.

기존 `renderDollView()`는 `dollView`/`viewCtx`를 직접 참조하므로 캔버스가 DOM 어디에 있든(모달 안이든 밖이든) 그대로 동작한다.

### 4. 예외 처리

- **화면 회전/리사이즈**: `vw`/`vh` 기반 CSS라 별도 JS 리스너 없이 자동 대응
- **색 팝업 위치**: 스포이드 사용 시 뜨는 `color-pop`은 `dollView.parentElement.appendChild(pop)`로 실시간 부모 기준 배치되므로, 모달로 이동된 상태에서도 위치가 올바르게 계산됨
- **phase 전환 중 모달이 열려있는 경우**: 정상 흐름상 발생하지 않지만(모달은 `paint` phase 내에서만 열림) 안전장치로 화면 전환 함수(`goPhase` 계열) 진입 시 `zoomOpen`이면 `closeZoom()`을 먼저 호출해 엘리먼트가 잘못된 위치에 낀 채로 남지 않도록 함
- **포커스 관리**: 열 때 닫기 버튼으로, 닫을 때 트리거 버튼으로 포커스 이동 (스크린리더/키보드 사용자 대응)

### 5. 테스트 계획 (수동 — 프로젝트에 자동 테스트 프레임워크 없음)

- 확대 열기 → 붓질 → 닫기 → 축소 화면에 반영 확인
- 확대 상태에서 되돌리기 / 전부지우기 / 스포이드 / 색상변경 / 브러시크기 전환 모두 정상 동작
- 확대 → 축소 → 확대 반복 시 그림 유실 없음
- 모바일 세로/가로, 데스크톱 폭(320~1440px)에서 레이아웃 깨짐 없음
- `prefers-reduced-motion` 켠 상태에서 트랜지션 생략 확인
- 스포이드로 색 추출 시 팝업이 확대된 캔버스 위 올바른 위치에 표시되는지 확인
- 키보드로 `Esc` 눌러 닫히는지, 포커스가 올바르게 이동하는지 확인

## 영향 범위

- `index.html` 단일 파일 내 CSS 추가(모달 스타일) + HTML 마크업 추가(버튼·모달) + JS 추가(열기/닫기 로직)
- 기존 색칠 로직(`renderDollView`, `strokeSetup`, `pickColorFromCrop`, undo 등)은 **수정 없음** — 엘리먼트 위치만 이동하므로 무변경으로 재사용
- `CONFIG` 객체 변경 없음 (순수 UI/CSS 사안이며 기존에도 레이아웃 수치는 CSS에 직접 기술하는 패턴을 따름)
- 데이터 모델(Firestore 문서 스키마) 변경 없음
