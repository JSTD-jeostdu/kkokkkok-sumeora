# 공개 갤러리 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 스테이지 제작 시 공개/비공개를 선택할 수 있게 하고, 공개로 설정된 스테이지를 홈 화면의 별도 갤러리 섹션에서 발견할 수 있게 한다.

**Architecture:** `stages` 문서에 `isPublic: boolean`(기본 `false`, 생성 시점에만 설정) 필드를 추가한다. Firestore 보안 규칙을 `get`(개별 조회, 전체 허용 유지)과 `list`(컬렉션 쿼리, `isPublic==true`만 허용)로 분리해 신규 목록 조회 기능을 안전하게 도입한다. `isPublic + stats.plays` 복합 쿼리를 위한 Firestore 인덱스를 `firestore.indexes.json`으로 코드화해 CLI로 배포한다. UI는 기존 `.tool-btn`(aria-pressed 토글 버튼) 패턴과 `.g-card` 갤러리 카드 스타일을 재사용한다.

**Tech Stack:** 순수 HTML/CSS/JS(빌드 도구 없음), Firebase Firestore(compat SDK), Firebase CLI(규칙·인덱스 배포)

## Global Constraints
- 단일 파일 `index.html` 원칙 — UI/로직 변경은 `index.html` 내부에서만 발생
- 기존에 배포된 `stages` 문서는 `isPublic` 필드가 없으며, 이 필드가 없는 문서는 신규 `list` 쿼리에서 자동으로 제외되어야 한다(마이그레이션 없이 안전하게 비공개 취급)
- `allow update`는 `stats`/`taps` 필드만 허용 — `isPublic`은 생성 시점에만 설정 가능, 이후 변경 불가(이번 범위 밖)
- 디자인 토큰: 도화지 `#FAF7F0` · 연필 `#4A4A4A` · 연필-연함 `#9B958A` · 포인트 레드 `#E45B5B`
- **주의**: 설계 문서(`docs/superpowers/specs/2026-07-25-public-gallery-design.md`)는 토글 버튼 위치를 "place-panel"이라 적었으나, 실제 코드에서 "저장하고 공유하기"(`btn-save`) 버튼은 `paint-panel`(`index.html:432-469`) 안에 있다. 아래 태스크는 실제 코드 구조(`paint-panel`)를 기준으로 한다 — 설계 의도(버튼 바로 위에 토글 배치)는 동일하게 유지된다.

---

### Task 1: `firestore.rules` — get/list 분리

**Files:**
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\firestore.rules`

**Interfaces:**
- Consumes: 없음
- Produces: `allow list`이 `resource.data.isPublic == true`를 요구 — Task 4의 `renderPublicGallery()` 쿼리가 이 규칙을 통과해야 함

- [ ] **Step 1: 규칙 수정**

`firestore.rules`의 다음 줄:

```
      // 읽기: URL을 아는 누구나 플레이할 수 있어야 하므로 전체 허용
      allow read: if true;
```

다음으로 교체:

```
      // 개별 조회: 링크(?s=)로 플레이, 예제 갤러리 — ID를 아는 사람은 전체 허용 (기존과 동일)
      allow get: if true;
      // 목록 조회: 공개 갤러리 전용 — isPublic == true인 문서만 쿼리 결과에 포함.
      // 기존 문서는 isPublic 필드가 없어(undefined != true) 자동으로 제외된다.
      allow list: if resource.data.isPublic == true;
```

- [ ] **Step 2: 문법 검증**

Run: `firebase deploy --only firestore:rules --dry-run` (dry-run 미지원 시 `firebase deploy --only firestore:rules` 자체가 배포 전 컴파일 검증을 수행하므로 Task 6에서 실제 배포로 검증)

Expected: 규칙 문법 오류 없음 (중괄호·세미콜론 등)

- [ ] **Step 3: Commit**

```bash
git add firestore.rules
git commit -m "firestore.rules: read를 get/list로 분리, list는 공개 스테이지만 허용"
```

---

### Task 2: `firestore.indexes.json` 추가 — 복합 인덱스 코드화

**Files:**
- Create: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\firestore.indexes.json`
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\firebase.json`

**Interfaces:**
- Consumes: 없음
- Produces: `isPublic`(ASC) + `stats.plays`(DESC) 복합 인덱스 정의 — Task 6에서 `firebase deploy --only firestore:indexes`로 실제 생성

- [ ] **Step 1: `firestore.indexes.json` 작성**

```json
{
  "indexes": [
    {
      "collectionGroup": "stages",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "isPublic", "order": "ASCENDING" },
        { "fieldPath": "stats.plays", "order": "DESCENDING" }
      ]
    }
  ],
  "fieldOverrides": []
}
```

- [ ] **Step 2: `firebase.json`에 연결**

현재 내용:

```json
{
  "firestore": {
    "rules": "firestore.rules"
  }
}
```

다음으로 교체:

```json
{
  "firestore": {
    "rules": "firestore.rules",
    "indexes": "firestore.indexes.json"
  }
}
```

- [ ] **Step 3: JSON 문법 검증**

Run:
```powershell
Get-Content "C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\firestore.indexes.json" -Raw | ConvertFrom-Json | Out-Null
Get-Content "C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\firebase.json" -Raw | ConvertFrom-Json | Out-Null
"OK"
```
Expected: `OK` (파싱 에러 없음)

- [ ] **Step 4: Commit**

```bash
git add firestore.indexes.json firebase.json
git commit -m "firestore.indexes.json 추가 — 공개 갤러리 복합 인덱스(isPublic+stats.plays) 코드화"
```

---

### Task 3: 데이터 모델 + 제작 화면 공개/비공개 토글

**Files:**
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html:709-723` (`state` 객체)
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html:461-468` (`paint-panel` 저장 버튼 영역)
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html:2445-2460` (`resetAll()`)
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html:1732-1741` (`saveStage()`의 `ref.set({...})`)
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html:2376-2380` 부근 (이벤트 리스너 배선)
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html` CSS (`.tool-btn` 규칙 다음, 재사용이라 신규 CSS 불필요 — 스타일 변경 없음)

**Interfaces:**
- Consumes: 기존 `.tool-btn`/`.btn-wide` CSS 클래스, `state` 객체, `resetAll()`
- Produces: `state.isPublic: boolean`, 저장된 문서의 `isPublic` 필드 — Task 4의 공개 갤러리 쿼리가 이 필드를 읽음

- [ ] **Step 1: `state` 객체에 필드 추가**

`index.html:709-723`의 `state` 객체 선언에서, `deleteReturnPhase: 'home',` 다음 줄에 추가:

```js
  isPublic: false,          // 공개 갤러리 노출 여부 — 생성 시점에만 설정, 기본 비공개
```

- [ ] **Step 2: `paint-panel`에 토글 버튼 추가**

`index.html:461-468`의 현재 내용:

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

다음으로 교체 (토글 버튼을 저장 버튼 행 바로 위에 삽입):

```html
      <div class="panel-actions">
        <button class="btn btn-ghost" id="btn-repick">사진 바꾸기</button>
        <button class="btn btn-ghost" id="btn-to-place">다시 배치</button>
      </div>
      <button class="tool-btn btn-wide" id="btn-toggle-public" aria-pressed="false">🔒 이 스테이지를 공개 갤러리에 올리기</button>
      <div class="panel-actions">
        <button class="btn btn-ghost" id="btn-to-test">테스트 플레이</button>
        <button class="btn btn-primary" id="btn-save">저장하고 공유하기</button>
      </div>
```

- [ ] **Step 3: 토글 클릭 핸들러 추가**

`index.html:2376-2380`의 기존 `btn-flip` 핸들러(동일한 aria-pressed 토글 패턴) 바로 다음에 추가:

```js
$('btn-toggle-public').addEventListener('click', () => {
  state.isPublic = !state.isPublic;
  $('btn-toggle-public').setAttribute('aria-pressed', String(state.isPublic));
  $('btn-toggle-public').textContent = state.isPublic
    ? '🌍 이 스테이지를 공개 갤러리에 올리기'
    : '🔒 이 스테이지를 공개 갤러리에 올리기';
});
```

- [ ] **Step 4: `resetAll()`에서 초기화**

`index.html:2445-2460`의 `resetAll()` 함수에서, `state.savedId = null;` 다음 줄에 추가:

```js
  state.isPublic = false;
  $('btn-toggle-public').setAttribute('aria-pressed', 'false');
  $('btn-toggle-public').textContent = '🔒 이 스테이지를 공개 갤러리에 올리기';
```

- [ ] **Step 5: `saveStage()`에 필드 추가**

`index.html:1732-1741`의 `ref.set({...})` 호출에서, `taps: [],` 다음 줄에 추가:

```js
      isPublic: state.isPublic,
```

- [ ] **Step 6: 로컬 서버로 토글 UI 확인**

1. 저장소 루트에서 `python -m http.server 8765` 실행
2. Chrome으로 `http://localhost:8765/index.html` 접속
3. `mcp__claude-in-chrome__javascript_tool`로 다음을 실행해 paint-panel을 강제 노출:

```javascript
document.querySelectorAll('main > section').forEach(s => s.classList.add('hidden'));
document.getElementById('screen-maker').classList.remove('hidden');
document.querySelectorAll('#screen-maker .panel').forEach(p => p.classList.add('hidden'));
document.getElementById('paint-panel').classList.remove('hidden');
```

4. 스크린샷으로 "🔒 이 스테이지를 공개 갤러리에 올리기" 버튼이 "저장하고 공유하기" 위에 표시되는지 확인
5. `mcp__claude-in-chrome__computer`(action: `left_click`)로 버튼 클릭 후 다시 스크린샷 — "🌍"로 아이콘이 바뀌고 버튼이 눌린 상태(배경색 반전)로 보이는지 확인
6. 로컬 서버 프로세스 종료

Expected: 클릭할 때마다 🔒 ↔ 🌍 전환, 버튼 배경색이 `aria-pressed` 상태에 따라 반전됨. 레이아웃 깨짐 없음.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "공개/비공개 토글 추가 — state.isPublic, paint-panel UI, saveStage() 필드 저장"
```

---

### Task 4: 홈 화면 공개 갤러리 섹션

**Files:**
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html:380-384` (홈 화면, `#gallery` 다음)
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html:111-122` (CSS, 예제 갤러리 규칙 다음)
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html:2108-2149` (`renderGallery()` — 카드 생성 로직을 `buildStageCard()`로 추출)
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html:2397, 2480` (`renderGallery()` 호출 지점 — `renderPublicGallery()` 추가 호출)

**Interfaces:**
- Consumes: Task 3에서 저장된 `isPublic` 필드, `firestore.rules`의 `allow list` 규칙(Task 1), 복합 인덱스(Task 2)
- Produces: 없음 (최종 UI, 다른 태스크가 참조하지 않음)

- [ ] **Step 1: 홈 화면에 섹션 추가**

`index.html:380-384`의 현재 내용:

```html
    <div id="gallery" class="cut-card hidden">
      <h3>예제 스테이지 🎯</h3>
      <p id="gallery-loading">불러오는 중…</p>
      <div id="gallery-grid"></div>
    </div>
```

다음으로 교체 (새 섹션 추가):

```html
    <div id="gallery" class="cut-card hidden">
      <h3>예제 스테이지 🎯</h3>
      <p id="gallery-loading">불러오는 중…</p>
      <div id="gallery-grid"></div>
    </div>
    <!-- 공개 갤러리: 결과 0개면 통째로 숨겨짐 -->
    <div id="public-gallery" class="cut-card hidden">
      <h3>다른 사람들이 숨긴 스테이지 🌍</h3>
      <p id="public-gallery-loading">불러오는 중…</p>
      <div id="public-gallery-grid"></div>
    </div>
```

- [ ] **Step 2: CSS 추가**

`index.html:111-122`의 예제 갤러리 CSS 블록(`#gallery-loading{...}` 규칙) 바로 다음에 추가:

```css
#public-gallery{ width:100%; text-align:left; margin-top:16px; }
#public-gallery h3{ margin-bottom:10px; }
#public-gallery-grid{ display:grid; grid-template-columns:1fr 1fr; gap:10px; }
#public-gallery-loading{ text-align:center; color:var(--pencil-soft); font-size:16px; }
```

- [ ] **Step 3: `buildStageCard()` 추출 + `renderGallery()`/`renderPublicGallery()` 구현**

`index.html:2108-2149`의 현재 `renderGallery()` 전체:

```js
async function renderGallery(){
  const ids = CONFIG.EXAMPLE_STAGES;
  if (!ids || !ids.length) return;         // 비어 있으면 갤러리 숨김 유지
  await initFirebase();
  if (!fb.ready) return;
  $('gallery').classList.remove('hidden');
  $('gallery-loading').classList.remove('hidden'); // 재진입 시 로딩 문구 복원
  const grid = $('gallery-grid');
  grid.innerHTML = '';
  let loaded = 0;
  // 카드 단위 지연 로드: 하나 실패해도 나머지는 표시
  for (const id of ids){
    try {
      const snap = await fb.db.collection('stages').doc(id).get();
      if (!snap.exists) continue;
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
      card.querySelector('img').src = d.photoB64;
      card.addEventListener('click', () => {
        // 새 URL로 이동해 플레이 (라우팅 로직 재사용, 뒤로 가기도 자연스럽다)
        location.href = location.pathname + '?s=' + id;
      });
      grid.appendChild(card);
      loaded++;
    } catch(e){
      console.warn('예제 스테이지 로드 실패:', id, e);
    }
  }
  $('gallery-loading').classList.add('hidden');
  if (!loaded) $('gallery').classList.add('hidden');
}
```

다음으로 교체 (카드 생성 로직을 `buildStageCard()`로 추출, `renderPublicGallery()` 신규 추가):

```js
// 갤러리 카드 하나 생성 (예제 갤러리·공개 갤러리 공용)
function buildStageCard(id, d){
  const eyedropperUsed = d.dolls ? mdAnyEyedropperUsed(d.dolls) : d.eyedropperUses > 0;
  const grade = computeGrade(d.stats, d.timeLimit || CONFIG.TIME_LIMIT, eyedropperUsed ? 1 : 0);
  const dollCount = d.dolls ? d.dolls.length : 1;
  const card = document.createElement('button');
  card.className = 'g-card';
  card.setAttribute('aria-label', '스테이지 플레이');
  // 썸네일은 강한 블러 처리 — 정답 위치·사진 내용 노출 방지
  card.innerHTML =
    '<div class="g-thumb"><img alt="" loading="lazy"></div>' +
    '<div class="g-meta"><span>도전 ' + (d.stats ? d.stats.plays : 0) + '</span>' +
    (dollCount > 1 ? '<span class="g-count">🧍×' + dollCount + '</span>' : '') +
    '<span class="g-grade">' + grade.g + '</span></div>';
  card.querySelector('img').src = d.photoB64;
  card.addEventListener('click', () => {
    // 새 URL로 이동해 플레이 (라우팅 로직 재사용, 뒤로 가기도 자연스럽다)
    location.href = location.pathname + '?s=' + id;
  });
  return card;
}

async function renderGallery(){
  const ids = CONFIG.EXAMPLE_STAGES;
  if (!ids || !ids.length) return;         // 비어 있으면 갤러리 숨김 유지
  await initFirebase();
  if (!fb.ready) return;
  $('gallery').classList.remove('hidden');
  $('gallery-loading').classList.remove('hidden'); // 재진입 시 로딩 문구 복원
  const grid = $('gallery-grid');
  grid.innerHTML = '';
  let loaded = 0;
  // 카드 단위 지연 로드: 하나 실패해도 나머지는 표시
  for (const id of ids){
    try {
      const snap = await fb.db.collection('stages').doc(id).get();
      if (!snap.exists) continue;
      grid.appendChild(buildStageCard(id, snap.data()));
      loaded++;
    } catch(e){
      console.warn('예제 스테이지 로드 실패:', id, e);
    }
  }
  $('gallery-loading').classList.add('hidden');
  if (!loaded) $('gallery').classList.add('hidden');
}

async function renderPublicGallery(){
  await initFirebase();
  if (!fb.ready) return;
  $('public-gallery').classList.remove('hidden');
  $('public-gallery-loading').classList.remove('hidden');
  const grid = $('public-gallery-grid');
  grid.innerHTML = '';
  let loaded = 0;
  try {
    const snap = await fb.db.collection('stages')
      .where('isPublic', '==', true)
      .orderBy('stats.plays', 'desc')
      .limit(20)
      .get();
    snap.forEach((doc) => {
      grid.appendChild(buildStageCard(doc.id, doc.data()));
      loaded++;
    });
  } catch(e){
    // 인덱스 미생성 등으로 쿼리가 실패해도 섹션을 조용히 숨긴다
    console.warn('공개 갤러리 로드 실패:', e);
  }
  $('public-gallery-loading').classList.add('hidden');
  if (!loaded) $('public-gallery').classList.add('hidden');
}
```

- [ ] **Step 4: 호출 지점 추가**

`index.html:2397`(`btn-home` 핸들러 내부의 `renderGallery();`) 다음 줄에 추가:

```js
  renderPublicGallery();
```

`index.html:2480`(`boot()` 함수 내부의 `renderGallery();`) 다음 줄에도 동일하게 추가:

```js
  renderPublicGallery();
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "홈 화면에 공개 갤러리 섹션 추가, buildStageCard() 공용 함수로 추출"
```

---

### Task 5: `CLAUDE.md` 배포 절차 갱신

**Files:**
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\CLAUDE.md`

**Interfaces:**
- Consumes: 없음
- Produces: 없음 (문서만)

- [ ] **Step 1: 배포 절차에 인덱스 배포 단계 추가**

`CLAUDE.md:82-86`의 "## 배포 절차" 목록 현재 내용:

```markdown
1. ✅ Firebase 콘솔: 프로젝트 생성 → Authentication **익명 로그인 활성화** → 웹 앱 등록 → config를 `CONFIG.firebase`에 붙여넣기 (Spark 무료 플랜 그대로, 카드 불필요)
2. Firestore Database 생성(프로덕션 모드, 리전 asia-northeast3 권장) → `firestore.rules` 내용을 규칙 탭에 게시 (Storage는 만들지 않는다)
3. GitHub 저장소에 `index.html` 푸시 → Settings > Pages 활성화
4. 실기기 검증: 저장→시크릿 창 플레이→두 기기 동시 플레이(stats 유실 확인)→삭제키 삭제→결과 보기 히트맵
5. 예제 스테이지 5개 직접 제작 → ID를 `CONFIG.EXAMPLE_STAGES`에 기입 → 재푸시
```

다음으로 교체 (인덱스 배포 단계를 3번으로 삽입하고 이후 번호를 순차적으로 밀기):

```markdown
1. ✅ Firebase 콘솔: 프로젝트 생성 → Authentication **익명 로그인 활성화** → 웹 앱 등록 → config를 `CONFIG.firebase`에 붙여넣기 (Spark 무료 플랜 그대로, 카드 불필요)
2. Firestore Database 생성(프로덕션 모드, 리전 asia-northeast3 권장) → `firestore.rules` 내용을 규칙 탭에 게시 (Storage는 만들지 않는다)
3. 공개 갤러리 복합 인덱스 배포: `firebase deploy --only firestore:rules,firestore:indexes` (firestore.indexes.json에 `isPublic`+`stats.plays` 인덱스가 코드화되어 있어 콘솔 수동 생성 불필요)
4. GitHub 저장소에 `index.html` 푸시 → Settings > Pages 활성화
5. 실기기 검증: 저장→시크릿 창 플레이→두 기기 동시 플레이(stats 유실 확인)→삭제키 삭제→결과 보기 히트맵
6. 예제 스테이지 5개 직접 제작 → ID를 `CONFIG.EXAMPLE_STAGES`에 기입 → 재푸시
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "CLAUDE.md 배포 절차에 공개 갤러리 인덱스 배포 단계 추가"
```

---

### Task 6: 실제 Firestore 프로젝트에 배포 및 왕복 검증

> ⚠️ 이 태스크는 로컬 에뮬레이터가 아닌 **실제 배포된 Firebase 프로젝트(`kkokkkok-sumeora`)**에 규칙·인덱스를 배포하고, 실제 문서를 생성·삭제한다. (로컬 환경에 Java가 없어 Firestore 에뮬레이터를 실행할 수 없음을 확인함 — 에뮬레이터가 있었다면 그쪽을 우선했을 것)

**Files:** 없음 (배포 및 수동 검증)

**Interfaces:**
- Consumes: Task 1~5의 모든 변경사항
- Produces: 없음

- [ ] **Step 1: 규칙 + 인덱스 배포**

Run: `firebase deploy --only firestore:rules,firestore:indexes`
Expected: `Deploy complete!` — 에러 없음

- [ ] **Step 2: 로컬 서버로 실제 앱 구동**

`python -m http.server 8765`로 저장소 루트를 서빙하고 Chrome으로 `http://localhost:8765/index.html` 접속. (이 앱은 `CONFIG.firebase`가 실제 프로젝트를 가리키므로 로컬에서 열어도 저장은 실제 프로덕션 Firestore에 기록된다.)

- [ ] **Step 3: 공개 테스트 스테이지 생성**

실제 업로드→색칠→배치 플로우를 최소한으로 거쳐(작은 테스트 이미지 사용) "🌍 공개" 토글을 켠 채로 저장. 생성된 스테이지 ID를 기록한다(공유 URL의 `?s=` 값).

- [ ] **Step 4: 비공개 테스트 스테이지 생성**

같은 방식으로 토글을 끈 채(기본값) 스테이지를 하나 더 생성. ID를 기록한다.

- [ ] **Step 5: 공개 갤러리 노출 검증**

홈으로 돌아가(`btn-home`) 공개 갤러리 섹션을 확인.

Expected: Step 3에서 만든 공개 스테이지 카드가 보임. Step 4의 비공개 스테이지 카드는 **보이지 않음**.

- [ ] **Step 6: 기존 문서(마이그레이션 없음) 검증**

이 프로젝트에 이미 배포되어 있던(오늘 이전에 만들어진) 스테이지가 있다면, 그 문서에는 `isPublic` 필드가 없으므로 공개 갤러리에 노출되지 않아야 한다. 콘솔에서 `console.warn`/에러 없이 조용히 제외되는지 확인.

- [ ] **Step 7: 테스트 데이터 정리**

Step 3·4에서 만든 두 테스트 스테이지를 각각 삭제키로 삭제(`btn-open-delete-play` 또는 `btn-open-delete-share` 흐름 사용). 프로덕션 DB에 테스트 데이터를 남기지 않는다.

- [ ] **Step 8: 로컬 서버 종료**

Step 2에서 실행한 `python -m http.server 8765` 프로세스를 종료한다.

---

## Self-Review 결과
- **스펙 커버리지**: 설계 문서의 데이터 모델(Task 3), 보안 규칙(Task 1), 제작 화면 토글(Task 3), 홈 화면 갤러리 섹션(Task 4), 배포 절차 문서화(Task 5) 모두 대응하는 태스크 있음. 복합 인덱스는 설계 문서가 "콘솔 수동 생성"으로 적었으나 계획에서는 `firestore.indexes.json` 코드화 + CLI 배포로 자동화 — 더 안전하고 재현 가능한 방법이라 개선해 반영함(설계 의도인 "인덱스 필요성 인지 및 해결"은 동일하게 충족).
- **플레이스홀더 스캔**: 없음.
- **타입/네이밍 일관성**: `isPublic`, `buildStageCard()`, `renderPublicGallery()`, `btn-toggle-public` 이름이 모든 태스크에서 동일하게 사용됨.
- **설계 문서와의 불일치 수정**: 토글 버튼 위치가 설계 문서엔 "place-panel"로 적혀 있었으나 실제 코드 확인 결과 `paint-panel`이 맞음 — Global Constraints에 명시하고 실제 위치 기준으로 태스크 작성함.
