# 인형 외곽선 제거 설계

> 작성일: 2026-07-21

## 배경 / 문제

종이인형을 배경색으로 위장시켜도 `renderDollComposite()`에서 항상 그리는 외곽선(`CONFIG.OUTLINE_COLOR`, 진한 회색 `rgba(74,74,74,.85)`, 두께 `CONFIG.OUTLINE_WIDTH` 2.5px)이 인형의 실루엣 가장자리를 뚜렷하게 드러내, 색을 아무리 잘 맞춰도 인형 위치가 금방 티가 난다.

## 범위

- **제거 대상**: `renderDollComposite()`(index.html:673-680)의 외곽선 stroke — 배치·색칠·실제 플레이·저장되는 인형 PNG(`dollB64`) 전부에 영향
- **유지 대상** (이번 변경과 무관, 그대로 둠):
  - 반짝 힌트(`hintFlash()`, index.html:1085-1103) — 0.5초 빨간 테두리 깜빡임은 의도된 힌트 연출
  - 공유 카드(index.html:1428-1443) — 흰 실루엣 브랜딩 일러스트의 테두리, 위장색 자체는 원래도 노출 안 됨

## 설계

`renderDollComposite()`에서 외곽선을 그리는 다음 블록을 제거한다:

```js
compCtx.lineWidth = CONFIG.OUTLINE_WIDTH;
compCtx.strokeStyle = CONFIG.OUTLINE_COLOR;
compCtx.lineJoin = 'round';
compCtx.stroke(dollPath);
```

`CONFIG.OUTLINE_COLOR`/`CONFIG.OUTLINE_WIDTH` 상수는 다른 곳에서 쓰이지 않으므로 `CONFIG` 객체에서 함께 제거한다(사용처 전수 확인 후 삭제).

## 영향 범위

- `index.html` 단일 파일, 함수 하나에서 4줄 제거 + 상수 2개 제거
- 데이터 모델·Firestore 규칙·다른 화면 로직 변경 없음
- 기존에 저장된 스테이지의 `dollB64`(외곽선이 이미 구워진 PNG)는 재생성되지 않으므로 여전히 외곽선이 남아있음 — 새로 저장되는 스테이지부터 외곽선 없이 저장됨 (기존 스테이지 재저장/마이그레이션은 범위 밖)
