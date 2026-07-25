# 공개 갤러리 — 설계

## 배경
현재 홈 화면에는 `CONFIG.EXAMPLE_STAGES`(제작자가 직접 큐레이션한 ID 배열) 기반의 "예제 스테이지" 갤러리만 있다. 신규 유입을 늘리려면 사용자가 자유롭게 제출한(공개로 설정한) 스테이지도 링크 없이 홈에서 발견할 수 있어야 한다. 이번 작업은 스테이지 제작 시 공개/비공개를 선택하는 토글과, 공개로 설정된 스테이지를 모아 보여주는 별도 갤러리 섹션을 추가한다.

## 기존 제약 확인
- `firestore.rules`는 현재 `allow read: if true` — 개별 문서 조회(`get`)와 컬렉션 목록 조회(`list`)를 구분하지 않고 전부 허용한다. 지금까지는 앱 코드가 `doc(id).get()`(개별 조회)만 사용해서 문제가 드러나지 않았을 뿐, 컬렉션을 직접 쿼리하면 누구나 전체 문서를 열람할 수 있는 상태였다.
- 이번 기능으로 컬렉션 `list` 쿼리가 처음 도입되므로, 이 시점에 규칙을 `get`/`list`로 분리해 `list`는 공개 문서만 허용하도록 강화한다.
- `allow update`는 `stats`/`taps` 필드만 변경을 허용하므로, `isPublic`은 생성 시점에만 설정 가능하고 이후 수정할 수 없다(범위 밖).

## 데이터 모델
`stages/{stageId}` 문서에 필드 추가:
- `isPublic: boolean` — 기본값 `false`. 생성 시 토글 상태에 따라 결정, 이후 불변.

기존에 생성된 문서에는 이 필드가 없다. `resource.data.isPublic == true` 비교에서 `undefined == true`는 `false`이므로, **기존 문서는 마이그레이션 없이 자동으로 목록에서 제외**된다(안전한 기본 동작).

## 보안 규칙 변경 (`firestore.rules`)
`allow read: if true;` 한 줄을 아래 두 줄로 교체:

```
// 개별 조회: 링크(?s=)로 플레이, 예제 갤러리 — ID를 아는 사람은 계속 전체 허용 (기존과 동일)
allow get: if true;
// 목록 조회: 신규 공개 갤러리 전용 — isPublic == true인 문서만 쿼리 결과에 포함
allow list: if resource.data.isPublic == true;
```

`allow create`/`allow update`/`allow delete`는 변경 없음.

## 제작 화면 — 공개/비공개 토글
`place-panel`의 "저장하고 공유하기"(`btn-save`) 버튼 바로 위에 토글 버튼 추가. 이 코드베이스는 체크박스 대신 `.tool-btn` + `aria-pressed` 패턴을 전역에서 사용하므로(붓 크기, 좌우반전, 스포이드 등) 동일한 패턴을 따른다.

```html
<button class="tool-btn btn-wide" id="btn-toggle-public" aria-pressed="false">
  🔒 이 스테이지를 공개 갤러리에 올리기
</button>
```

- 클릭 시 `aria-pressed` 토글, 라벨 아이콘 🔒 ↔ 🌍 전환, `state.isPublic` 갱신
- 기본값 `false`(비공개) — 개인 사진을 다루는 앱 특성상 프라이버시 보호적 기본값
- `resetAll()`에서 `state.isPublic = false`로 초기화 (다음 제작 시 항상 비공개부터 시작)
- `saveStage()`의 `ref.set({...})` 객체에 `isPublic: state.isPublic` 추가

## 홈 화면 — 공개 갤러리 섹션
기존 `#gallery`(예제 스테이지, `CONFIG.EXAMPLE_STAGES` 기반)는 그대로 유지. 그 아래 새 섹션 `#public-gallery` 추가.

- 제목: "다른 사람들이 숨긴 스테이지 🌍"
- 쿼리: `fb.db.collection('stages').where('isPublic','==',true).orderBy('stats.plays','desc').limit(20).get()`
- 정렬: 인기순(도전 수 `stats.plays` 내림차순), 최대 20개, 더보기 없음 (Firestore 무료 읽기 예산 고려)
- 카드 UI: 기존 `.g-card`/`.g-thumb` 스타일 재사용 — 강한 블러 썸네일(정답 노출 방지), 도전 수, 위장 등급, 인형 수 배지. 예제 갤러리와 시각적 언어는 통일하되 섹션 제목으로 구분
- 결과 0개면 섹션 자체 숨김 (기존 예제 갤러리와 동일한 패턴)
- `renderGallery()`와 별도로 `renderPublicGallery()` 함수를 만들어 홈 진입 시 호출. Firebase 미설정/오프라인이면 조용히 건너뜀 (기존 `initFirebase()`/`fb.ready` 체크 패턴 재사용)

## 배포 절차 추가 (`CLAUDE.md`)
`isPublic`(등호 필터) + `stats.plays`(정렬)를 함께 쓰는 쿼리는 Firestore 복합 인덱스가 필요하다. 인덱스가 없으면 첫 쿼리 실행 시 에러가 발생하며, 에러 메시지에 포함된 콘솔 링크를 클릭하면 자동 생성된다. 기존 "배포 절차" 목록에 다음 단계를 추가:

> 공개 갤러리를 한 번 방문해 콘솔 에러의 인덱스 생성 링크를 클릭 (또는 Firestore 콘솔에서 수동으로 `stages` 컬렉션에 `isPublic` Ascending + `stats.plays` Descending 복합 인덱스 생성)

## 영향받지 않는 것
- 예제 갤러리(`CONFIG.EXAMPLE_STAGES`, `renderGallery()`) 동작 변경 없음
- `?s=` 링크 플레이, 제작자 결과 보기, 삭제 등 개별 문서 `get` 기반 기능은 규칙 변경의 영향을 받지 않음
- `isPublic`을 생성 후 나중에 바꾸는 기능(제작자 결과 보기 화면 등)은 이번 범위 밖 — 필요해지면 별도 스펙
- 신고·자동 숨김 등 콘텐츠 모더레이션은 CLAUDE.md 백로그에 남아있는 별도 항목으로, 이번 작업 범위 밖
