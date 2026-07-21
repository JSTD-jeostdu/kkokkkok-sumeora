# 커스텀 컬러피커(SV박스+Hue막대) 설계

> 작성일: 2026-07-21

## 배경 / 목표

현재 색칠 화면의 색상 선택은 브라우저 네이티브 `<input type="color">`(index.html:324)에 의존한다. OS/브라우저마다 UI가 제각각이고, 세밀한 채도·명도 조절이 불편하다. 포토샵 스타일의 정사각형 SV(채도×명도)박스 + Hue(색상) 막대로 구성된 커스텀 컬러피커로 완전히 대체해, 모든 기기에서 동일한 UI로 정교하게 색을 고를 수 있게 한다.

## 목표

- 네이티브 `<input type="color">`를 커스텀 피커로 완전히 대체
- 정사각형 SV박스(드래그로 채도·명도 선택) + 가로 Hue막대(드래그로 색상 선택)
- HEX 코드 직접 입력칸 포함, SV박스/Hue막대와 양방향 동기화
- 기존 `#swatch`(현재 색 표시 동그라미)를 탭 트리거로 사용해 모달로 엶

## 비목표

- 색상 히스토리/최근 사용 색 목록
- RGB 개별 슬라이더(HEX 입력으로 대체)
- 팔레트 프리셋(자주 쓰는 색 저장)

## 설계

### 1. 인터랙션 흐름

1. 색칠 화면의 `#swatch`를 탭 가능하게 만듦(버튼 역할 부여, 44px 유지) — 탭하면 `#colorpicker-modal`이 열림
2. 모달이 열리면 현재 `state.color`(HEX)를 HSV로 변환해 SV박스 인디케이터 위치, Hue막대 인디케이터 위치, HEX 입력칸을 그 값으로 초기화
3. SV박스를 드래그 → 채도·명도가 실시간으로 바뀌며 미리보기 스와치·HEX 입력칸 갱신
4. Hue막대를 드래그 → 색상(Hue)이 바뀌며 SV박스의 기준색(오른쪽 위 꼭짓점 색)이 다시 그려지고, 미리보기 스와치·HEX 입력칸도 갱신
5. HEX 입력칸에 직접 입력(6자리 HEX 완성 시) → SV박스/Hue막대 인디케이터 위치가 그 색에 맞게 재계산되어 이동
6. "확인" 버튼 → `state.color`에 반영, 지우개 해제(`state.eraser = false`, 기존 `#color-picker` input 핸들러와 동일 동작), `#swatch` 배경 갱신, 모달 닫힘
7. 우상단 ✕ 또는 배경 탭 → 변경 취소하고 모달만 닫힘(모달을 여는 순간의 `state.color` 그대로 유지)

### 2. 마크업/DOM 구성

`#paint-panel` 밖(패널 컨테이너 레벨, 기존 `#zoom-modal`과 형제 요소)에 추가:

```html
<div id="colorpicker-modal" class="cp-modal hidden" role="dialog" aria-modal="true" aria-label="색상 선택">
  <div class="cp-card">
    <div class="cp-header">
      <span>색상 선택</span>
      <button id="btn-cp-close" aria-label="색상 선택 닫기">✕</button>
    </div>
    <canvas id="cp-svbox" width="240" height="240" aria-label="채도·명도 선택"></canvas>
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

`#cp-svbox` 위에는 드래그 인디케이터용 원형 `<div id="cp-sv-thumb">`를 절대 위치로 겹쳐 그린다(SV박스 자체는 캔버스 그림, 인디케이터만 DOM 엘리먼트로 분리 — 매 프레임 캔버스를 다시 그릴 필요 없음).

### 3. 렌더링

**SV박스** (`#cp-svbox`, 2D 캔버스):
```js
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
```
가로축 = 채도(왼쪽 0% → 오른쪽 100%), 세로축 = 명도(위쪽 100% → 아래쪽 0%). Hue막대를 움직일 때마다 `renderSVBox(newHue)`를 다시 호출해 기준색만 갱신한다.

**Hue막대** (`#cp-hue-track`, CSS 배경, DOM 엘리먼트라 매번 다시 그릴 필요 없음):
```css
#cp-hue-track{
  position:relative; height:28px; border-radius:14px; margin:10px 0;
  background:linear-gradient(to right, #f00 0%, #ff0 17%, #0f0 33%, #0ff 50%, #00f 67%, #f0f 83%, #f00 100%);
}
#cp-hue-thumb{
  position:absolute; top:-3px; width:6px; height:34px; border-radius:3px;
  background:#fff; border:2px solid var(--pencil); box-shadow:var(--shadow);
  transform:translateX(-3px); pointer-events:none;
}
```

### 4. 색 변환 함수 (외부 라이브러리 없이 직접 구현)

```js
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
function rgbToHex(r, g, b){ return '#' + [r, g, b].map((n) => n.toString(16).padStart(2, '0')).join(''); }
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

### 5. 포인터 좌표 처리

기존 확대 모달(`viewPos()`)과 동일한 패턴 재사용 — `getBoundingClientRect()`로 정규화:

```js
function cpSvPos(e){
  const r = cpSvBox.getBoundingClientRect();
  const x = Math.min(1, Math.max(0, (e.clientX - r.left) / r.width));
  const y = Math.min(1, Math.max(0, (e.clientY - r.top) / r.height));
  return { s: x * 100, v: (1 - y) * 100 };
}
function cpHuePos(e){
  const r = cpHueTrack.getBoundingClientRect();
  const x = Math.min(1, Math.max(0, (e.clientX - r.left) / r.width));
  return x * 360;
}
```
`pointerdown`/`pointermove`(캡처 중)/`pointerup`으로 드래그를 구현하며, 기존 색칠 캔버스 드래그(`dollView`의 `pointerdown`/`pointermove`/`pointerup` 패턴, index.html:927-951)와 동일한 이벤트 구조를 따른다.

### 6. HEX 입력 동기화

```js
cpHex.addEventListener('input', () => {
  const rgb = hexToRgb(cpHex.value);
  if (!rgb) return; // 완성되지 않은 입력은 무시, 다음 keystroke 대기
  const { h, s, v } = rgbToHsv(rgb.r, rgb.g, rgb.b);
  cpState.h = h; cpState.s = s; cpState.v = v;
  updateThumbs(); renderSVBox(h); updatePreview();
});
```
SV박스/Hue막대 드래그 쪽에서는 매 이동마다 `cpHex.value`도 함께 갱신한다(단, 입력칸에 포커스가 가 있는 동안은 덮어쓰지 않음 — 타이핑 중 커서 위치가 튀는 것을 방지).

### 7. 기존 코드와의 연결

- 색칠 도구 HTML에서 `<input type="color" id="color-picker" ...>` 제거
- `$('swatch')`에 `role="button" tabindex="0" aria-label="색상 선택 열기"` 추가, `click`(및 `keydown` Enter/Space) 리스너로 `openColorPicker()` 호출
- `openColorPicker()`: 현재 `state.color`를 `rgbToHsv(hexToRgb(...))`로 변환해 `cpState`에 저장 → `renderSVBox()`, 인디케이터 위치, HEX 입력칸, 미리보기 스와치 초기화 → 모달 표시
- `btn-cp-confirm` 클릭 → `state.color = cpState 기반 HEX`, `state.eraser = false`, `$('btn-eraser').setAttribute('aria-pressed','false')`(기존 `#color-picker` input 핸들러와 동일 부수효과), `swatch.style.background = state.color`, 모달 닫힘
- `pickColorFromCrop()`(스포이드)는 기존처럼 `state.color`/`swatch.style.background`만 갱신하고 끝 — 더 이상 `$('color-picker').value = state.color` 줄은 필요 없으므로 제거(네이티브 input이 사라지므로)

## 접근성 · 모바일

- SV박스/Hue막대 드래그 인디케이터는 시각적 위치 표시일 뿐, 실제 값은 `cpState`(JS 변수)가 단일 진실 소스
- 모달 자체는 `role="dialog" aria-modal="true"`, 닫기 버튼 `aria-label`
- SV박스 240×240은 손가락 드래그에 충분히 크고, Hue막대 높이 28px는 44px 규칙(버튼 탭 영역) 대상은 아니지만(드래그형 컨트롤이라 이미 확대 모달의 색칠 캔버스와 같은 예외 범주) 두께를 34px 썸(인디케이터)로 보완

## 영향 범위

- `index.html` 단일 파일: 새 마크업(모달) + CSS(모달·SV박스·Hue막대 스타일) + JS(색 변환 함수, 렌더링, 포인터 핸들링, 기존 `#color-picker` 관련 코드 제거/교체)
- `CONFIG` 객체 변경 없음
- 데이터 모델·Firestore 규칙 변경 없음(선택된 색은 여전히 `state.color`의 HEX 문자열 하나)
- 외부 CDN 의존성 추가 없음(직접 구현)
