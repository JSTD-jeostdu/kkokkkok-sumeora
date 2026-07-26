# 로그인 시스템 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 선택적 Google 로그인을 추가해 제작자가 자기 스테이지를 목록으로 관리할 수 있게 하고, 향후 프로필 기반 기능의 토대를 마련한다.

**Architecture:** 기존 익명 인증(`signInAnonymously`)은 그대로 기본 동작으로 유지한다. 로그인 버튼을 누르면 현재 익명 계정에 Google credential을 **연결**(`linkWithRedirect`)해서 uid를 그대로 승격시킨다. 이미 다른 기기에서 그 Google 계정을 쓴 적이 있어 연결이 실패하면, 보조 Firebase 앱 인스턴스로 대상 계정의 uid만 조용히 확인한 뒤 — **아직 익명 세션으로 인증된 상태에서** 내 스테이지들의 `creatorUid`를 그 uid로 먼저 넘기고(규칙: 현재 소유자만 가능), 그 다음에야 계정을 전환한다. 이 순서 덕분에 어느 시점에도 데이터가 유실되지 않는다. Firestore 규칙은 "내 스테이지 목록 조회"와 "소유자 본인만 삭제"를 추가로 허용하도록 확장한다.

**Tech Stack:** 순수 HTML/CSS/JS(빌드 도구 없음), Firebase Auth(Google 제공자, compat SDK — 이미 로드된 `firebase-auth-compat.js` 재사용), Firebase Firestore

## Global Constraints
- 단일 파일 `index.html` 원칙 — 로직·UI 변경은 `index.html` 내부에서만 발생
- **제작 흐름 어디에도 로그인을 강제하지 않는다.** "사진 고르기 → 바로 제작 시작" 저마찰 플로우는 완전히 그대로 유지
- Google 로그인만 지원(카카오 등은 이번 범위 밖)
- 프로필 문서(`users/{uid}` 컬렉션 등)는 만들지 않는다 — Firebase Auth의 `displayName`을 그대로 사용
- 병합은 **덮어쓰기 금지, 반드시 먼저 소유권을 넘기고 나중에 계정을 전환하는 순서** — 사용자가 과거에 겪은 데이터 유실 사고를 반복하지 않기 위한 핵심 제약
- 모바일 인앱 브라우저(카카오톡 등) 호환을 위해 팝업이 아닌 `signInWithRedirect`/`linkWithRedirect` 사용
- 탭 영역 44px 이상(CLAUDE.md 모바일 퍼스트 원칙) — 신규 링크·버튼도 준수
- **실제 Google OAuth 완료 흐름은 자동화 테스트가 불가능하다.** Google이 원격 제어되는 브라우저의 로그인 시도를 차단하기 때문. Task 6에서 이 한계와 사용자가 직접 검증해야 할 부분을 명시한다.

---

### Task 1: `firestore.rules` — list/update/delete 확장

**Files:**
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\firestore.rules`

**Interfaces:**
- Consumes: 없음
- Produces: `allow list`가 `creatorUid == request.auth.uid`인 문서도 허용, `allow update`가 `creatorUid` 단독 변경(본인 소유일 때만)도 허용, `allow delete`가 본인 소유로 제한 — Task 2~4의 모든 Firestore 호출이 이 규칙을 통과해야 함

- [ ] **Step 1: 규칙 수정**

현재 (`firestore.rules`):

```
      // 개별 조회: 링크(?s=)로 플레이, 예제 갤러리 — ID를 아는 사람은 전체 허용 (기존과 동일)
      allow get: if true;
      // 목록 조회: 공개 갤러리 전용 — isPublic == true인 문서만 쿼리 결과에 포함.
      // 기존 문서는 isPublic 필드가 없어(undefined != true) 자동으로 제외된다.
      allow list: if resource.data.isPublic == true;

      // 생성: 익명 인증 + 본인 uid + 필수 필드 + 이미지 크기 예산 준수
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

      // 수정: 플레이 기록(stats·taps)만 변경 가능 — 사진·인형·삭제키 등 변경 거부
      allow update: if request.auth != null
        && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['stats','taps']);

      // 삭제: 삭제키(SHA-256 해시) 검증이 클라이언트에서 이뤄지는 구조적 한계.
      // 무료 플랜에서 Cloud Functions 없이 서버 검증이 불가하므로 인증 사용자에게 허용한다.
      // → 문제가 생기면 Functions 도입(삭제키 서버 검증)으로 전환할 것.
      allow delete: if request.auth != null;
```

다음으로 교체:

```
      // 개별 조회: 링크(?s=)로 플레이, 예제 갤러리 — ID를 아는 사람은 전체 허용 (기존과 동일)
      allow get: if true;
      // 목록 조회: 공개 갤러리(isPublic) + 내 스테이지 목록(creatorUid 본인) 둘 다 허용.
      // 기존 문서는 isPublic 필드가 없어(undefined != true) 공개 갤러리에서는 자동 제외된다.
      allow list: if resource.data.isPublic == true
        || (request.auth != null && resource.data.creatorUid == request.auth.uid);

      // 생성: 익명 인증 + 본인 uid + 필수 필드 + 이미지 크기 예산 준수
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

      // 수정: 기존 stats/taps 변경 + 신규 creatorUid 단독 이전(병합용, 반드시 현재 소유자 본인만).
      // creatorUid 이전은 "지금 소유자가 자기 콘텐츠를 자기 다른 계정으로 넘기는" 동작이라
      // 다른 사람 소유 문서를 가로챌 수 없다 (request.auth.uid == resource.data.creatorUid로 보호).
      allow update: if request.auth != null && (
        request.resource.data.diff(resource.data).affectedKeys().hasOnly(['stats','taps'])
        || (request.resource.data.diff(resource.data).affectedKeys().hasOnly(['creatorUid'])
            && request.auth.uid == resource.data.creatorUid)
      );

      // 삭제: 기존엔 "인증만 되면 누구나" 삭제 가능했던 허점을 소유자 본인으로 강화.
      // 지금까지도 삭제는 항상 스테이지를 만든 바로 그 세션(창작자 본인)에서만 이뤄져 왔으므로
      // 기존 삭제키 UX에는 영향이 없고, ID만 알아도 삭제할 수 있었던 허점만 막는다.
      allow delete: if request.auth != null && request.auth.uid == resource.data.creatorUid;
```

- [ ] **Step 2: 문법 검증**

Run: `firebase deploy --only firestore:rules --dry-run`
Expected: `rules file firestore.rules compiled successfully`

- [ ] **Step 3: Commit**

```bash
git add firestore.rules
git commit -m "firestore.rules: 내 스테이지 목록 조회, 소유권 이전(병합), 삭제 소유자 제한 추가"
```

---

### Task 2: Firebase Auth 코어 로직 (익명 → 구글 연결, 안전 병합)

**Files:**
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html:746-766` (`fb` 상태 + `initFirebase()`)

**Interfaces:**
- Consumes: `CONFIG.firebase`, 전역 `fb` 객체
- Produces: `loginWithGoogle()`, `completeAuthRedirect()`, `mergeAndSignIn(credential)`, `updateAccountUI()` — Task 3(홈 UI)이 `loginWithGoogle`/`updateAccountUI`를 호출, Task 4(내 스테이지)가 로그인 상태 확인에 `firebase.auth().currentUser`를 사용

- [ ] **Step 1: `initFirebase()` 재구성 + 신규 함수 추가**

`index.html:746-766`의 현재 내용:

```js
/* ---------- Firebase (익명 인증) ---------- */
const fb = { db: null, uid: null, ready: false };
let fbInitPromise = null;
function initFirebase(){
  if (fbInitPromise) return fbInitPromise;
  fbInitPromise = (async () => {
    if (!window.firebase || !CONFIG.firebase.apiKey) return false;
    try {
      firebase.initializeApp(CONFIG.firebase);
      fb.db = firebase.firestore();
      const cred = await firebase.auth().signInAnonymously();
      fb.uid = cred.user.uid;
      fb.ready = true;
    } catch(e){
      console.warn('Firebase 초기화 실패 — 오프라인 모드로 동작:', e);
      fb.ready = false;
    }
    return fb.ready;
  })();
  return fbInitPromise;
}
```

다음으로 교체:

```js
/* ---------- Firebase (익명 기본 + 선택적 구글 로그인) ---------- */
const fb = { db: null, uid: null, ready: false };
let fbInitPromise = null;
function initFirebase(){
  if (fbInitPromise) return fbInitPromise;
  fbInitPromise = (async () => {
    if (!window.firebase || !CONFIG.firebase.apiKey) return false;
    try {
      firebase.initializeApp(CONFIG.firebase);
      fb.db = firebase.firestore();
      await completeAuthRedirect(); // 구글 로그인 리다이렉트 복귀 처리 (없으면 즉시 반환)
      if (!firebase.auth().currentUser){
        const cred = await firebase.auth().signInAnonymously();
        fb.uid = cred.user.uid;
      } else {
        fb.uid = firebase.auth().currentUser.uid;
      }
      fb.ready = true;
      updateAccountUI();
    } catch(e){
      console.warn('Firebase 초기화 실패 — 오프라인 모드로 동작:', e);
      fb.ready = false;
    }
    return fb.ready;
  })();
  return fbInitPromise;
}

// 로그인 버튼 클릭 → 현재 익명 계정에 구글 credential 연결 시도 (리다이렉트로 이동)
async function loginWithGoogle(){
  await initFirebase();
  if (!fb.ready){ alert('지금은 연결할 수 없어요.'); return; }
  const provider = new firebase.auth.GoogleAuthProvider();
  try {
    await firebase.auth().currentUser.linkWithRedirect(provider);
    // 여기서 구글 로그인 페이지로 이동하므로 이후 코드는 실행되지 않는다.
  } catch(e){
    console.warn('로그인 시작 실패:', e);
    alert('로그인을 시작할 수 없어요.');
  }
}

// 페이지 로드 시 구글 리다이렉트 복귀 결과 처리 (리다이렉트가 없었으면 조용히 반환)
async function completeAuthRedirect(){
  try {
    const result = await firebase.auth().getRedirectResult();
    if (result && result.user){
      fb.uid = result.user.uid; // 연결 성공 — uid는 그대로 유지되지만 갱신해 둠
    }
  } catch(e){
    if (e.code === 'auth/credential-already-in-use'){
      await mergeAndSignIn(e.credential);
    } else if (e.code){
      console.warn('로그인 실패:', e);
    }
  }
}

// "이미 다른 기기에서 이 구글 계정을 쓴 적 있음" 상황의 안전한 병합.
// 순서가 핵심: 아직 익명 세션인 상태에서 소유권을 먼저 넘긴 뒤에야 계정을 전환한다.
// 어느 시점에도 규칙(creatorUid == 현재 로그인 uid)을 어기지 않고, 데이터가 유실되지 않는다.
async function mergeAndSignIn(credential){
  const oldUid = fb.uid; // 아직 익명 uid로 인증된 상태
  // 보조 Firebase 앱 인스턴스로 대상 계정의 uid만 조용히 확인 — 기본 세션은 익명으로 유지
  const secondary = firebase.initializeApp(CONFIG.firebase, 'merge-probe-' + Date.now());
  let targetUid;
  try {
    const cred = await secondary.auth().signInWithCredential(credential);
    targetUid = cred.user.uid;
  } finally {
    await secondary.auth().signOut().catch(() => {});
    await secondary.delete().catch(() => {});
  }
  // 여전히 기본 세션은 익명(oldUid) — 이 상태에서 소유권 이전 (규칙: 현재 소유자 본인만 허용)
  const snap = await fb.db.collection('stages').where('creatorUid', '==', oldUid).get();
  if (!snap.empty){
    const batch = fb.db.batch();
    snap.forEach((doc) => batch.update(doc.ref, { creatorUid: targetUid }));
    await batch.commit();
  }
  // 이관이 끝난 뒤에야 기본 세션을 대상 계정으로 전환
  const finalCred = await firebase.auth().signInWithCredential(credential);
  fb.uid = finalCred.user.uid;
}

// 로그인 상태에 따라 홈 화면 링크 텍스트·"내 스테이지" 링크 표시 갱신
function updateAccountUI(){
  const loginLink = $('link-login');
  const mystagesLink = $('link-mystages');
  if (!loginLink || !mystagesLink) return;
  const user = firebase.auth().currentUser;
  if (user && !user.isAnonymous){
    loginLink.textContent = (user.displayName || '회원') + '님 · 로그아웃';
    loginLink.dataset.mode = 'logout';
    mystagesLink.classList.remove('hidden');
  } else {
    loginLink.textContent = '로그인';
    loginLink.dataset.mode = 'login';
    mystagesLink.classList.add('hidden');
  }
}
```

- [ ] **Step 2: 문법 확인**

브라우저에서 열었을 때 JS 파싱 에러가 없는지 확인(Step은 Task 3에서 UI와 함께 로컬 검증).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Firebase Auth 코어 로직 추가 — 구글 로그인 연결, 안전한 소유권 병합(먼저 이전 후 전환)"
```

---

### Task 3: 홈 화면 로그인 링크 UI

**Files:**
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html:369-370` (`#screen-home` 시작 부분)
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html:103` (CSS, `#screen-home` 규칙 다음)
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html:2419-2423` 부근 (이벤트 리스너 배선)

**Interfaces:**
- Consumes: Task 2의 `loginWithGoogle()`, `updateAccountUI()`, `firebase.auth()`
- Produces: `#link-login`, `#link-mystages` DOM 요소 — Task 4가 `#link-mystages` 클릭 핸들러에서 `renderMyStages()` 호출

- [ ] **Step 1: HTML에 계정 링크 행 추가**

`index.html:369-370`의 현재 내용:

```html
  <section id="screen-home">
    <svg class="hero-doll" viewBox="0 0 256 256" aria-hidden="true">
```

다음으로 교체:

```html
  <section id="screen-home">
    <div class="home-account-row">
      <button class="account-link" id="link-login">로그인</button>
      <button class="account-link hidden" id="link-mystages">내 스테이지</button>
    </div>
    <svg class="hero-doll" viewBox="0 0 256 256" aria-hidden="true">
```

- [ ] **Step 2: CSS 추가**

`index.html:103`(`#screen-home{...}` 규칙) 바로 다음에 추가:

```css
.home-account-row{ width:100%; display:flex; justify-content:flex-end; gap:14px; }
.account-link{ background:none; border:none; font-family:inherit; font-size:14px;
  color:var(--pencil-soft); text-decoration:underline; cursor:pointer;
  padding:0 4px; min-height:44px; display:inline-flex; align-items:center; }
```

- [ ] **Step 3: 이벤트 리스너 배선**

`index.html:2419-2423`의 `btn-flip` 핸들러 다음(기존 `btn-toggle-public` 핸들러 앞)에 추가:

```js
$('link-login').addEventListener('click', async () => {
  if ($('link-login').dataset.mode === 'logout'){
    await firebase.auth().signOut();
    const cred = await firebase.auth().signInAnonymously();
    fb.uid = cred.user.uid;
    updateAccountUI();
    renderGallery();
    renderPublicGallery();
  } else {
    loginWithGoogle();
  }
});
$('link-mystages').addEventListener('click', () => renderMyStages());
```

- [ ] **Step 4: 로컬 서버로 렌더링·에러 확인**

1. 저장소 루트에서 `python -m http.server 8765` 실행
2. Chrome으로 `http://localhost:8765/index.html` 접속
3. 콘솔에 JS 파싱/런타임 에러가 없는지 확인 (`renderMyStages`가 아직 없는 Task 3 시점에는 `link-mystages` 클릭 시에만 `ReferenceError`가 나는 게 정상 — 클릭하지 않고 확인)
4. 스크린샷으로 홈 화면 우측 상단에 "로그인" 링크가 표시되는지 확인
5. 로컬 서버 종료

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "홈 화면에 로그인/내 스테이지 링크 UI 추가"
```

---

### Task 4: 내 스테이지 화면

**Files:**
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html:603-604` (`delete-panel` 다음, `screen-maker` 닫기 전)
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html` CSS (`.g-count{...}` 규칙 다음 등 갤러리 CSS 블록 근처)
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html:1031-1049` (`PHASE_LABELS`, `panels` in `setPhase()`)
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\index.html` (renderPublicGallery 함수 다음에 신규 함수 추가)

**Interfaces:**
- Consumes: Task 1의 `allow list`(creatorUid 본인)·`allow delete`(소유자 본인) 규칙, Task 2의 `fb`/`initFirebase`, 기존 `loadResultView(id)` 함수(재사용)
- Produces: 없음 (최종 UI)

- [ ] **Step 1: HTML에 패널 추가**

`index.html:603-604`의 현재 내용:

```html
    </div>
  </section>
```

(이 `</div>`는 `delete-panel`을 닫는 태그) 다음으로 교체:

```html
    </div>

    <!-- 내 스테이지 (로그인 사용자 전용) -->
    <div id="mystages-panel" class="panel hidden">
      <h3>내 스테이지 📂</h3>
      <p id="mystages-loading" class="hint-line">불러오는 중…</p>
      <p id="mystages-empty" class="hint-line hidden">아직 만든 스테이지가 없어요.</p>
      <div id="mystages-list"></div>
    </div>
  </section>
```

- [ ] **Step 2: CSS 추가**

`.g-count{ font-size:13px; color:var(--pencil-soft); }` 규칙 다음에 추가:

```css
.mystage-item{ margin-bottom:14px; }
.mystage-row{ display:flex; align-items:center; gap:10px; margin-bottom:8px; }
.mystage-thumb{ width:64px; height:64px; border-radius:10px; overflow:hidden; background:#e9e5dc; flex:0 0 auto; }
.mystage-thumb img{ width:100%; height:100%; object-fit:cover; }
.mystage-plays{ font-size:16px; color:var(--pencil-soft); }
.mystage-item .btn{ margin-top:6px; }
```

- [ ] **Step 3: `PHASE_LABELS`·`panels`에 등록**

`index.html:1031-1035`의 현재 내용:

```js
const PHASE_LABELS = {
  upload:'사진 고르기', paint:'위장 색칠', place:'숨길 곳 정하기', test:'테스트 플레이',
  save:'저장 중', share:'공유하기', load:'불러오는 중', ready:'준비!', play:'찾아라!',
  end:'결과', view:'결과 보기', error:'앗', 'delete':'스테이지 삭제',
};
```

다음으로 교체:

```js
const PHASE_LABELS = {
  upload:'사진 고르기', paint:'위장 색칠', place:'숨길 곳 정하기', test:'테스트 플레이',
  save:'저장 중', share:'공유하기', load:'불러오는 중', ready:'준비!', play:'찾아라!',
  end:'결과', view:'결과 보기', error:'앗', 'delete':'스테이지 삭제', mystages:'내 스테이지',
};
```

`index.html:1044-1049`의 현재 내용:

```js
  const panels = {
    upload: uploadBox, paint: paintPanel, place: placePanel, test: testPanel,
    save: $('save-panel'), share: $('share-panel'), load: $('load-panel'),
    ready: $('play-panel'), play: $('play-panel'), end: $('playend-panel'),
    view: $('result-panel'), error: $('error-panel'), 'delete': $('delete-panel'),
  };
```

다음으로 교체:

```js
  const panels = {
    upload: uploadBox, paint: paintPanel, place: placePanel, test: testPanel,
    save: $('save-panel'), share: $('share-panel'), load: $('load-panel'),
    ready: $('play-panel'), play: $('play-panel'), end: $('playend-panel'),
    view: $('result-panel'), error: $('error-panel'), 'delete': $('delete-panel'),
    mystages: $('mystages-panel'),
  };
```

- [ ] **Step 4: `renderMyStages()`·`buildMyStageItem()`·`deleteOwnStage()` 추가**

`renderPublicGallery()` 함수(공개 갤러리 태스크에서 작성됨) 바로 다음에 추가:

```js
async function renderMyStages(){
  await initFirebase();
  if (!fb.ready) return;
  const user = firebase.auth().currentUser;
  if (!user || user.isAnonymous) return; // 로그인하지 않았으면 진입하지 않음
  setPhase('mystages');
  $('mystages-loading').classList.remove('hidden');
  $('mystages-empty').classList.add('hidden');
  const list = $('mystages-list');
  list.innerHTML = '';
  try {
    const snap = await fb.db.collection('stages').where('creatorUid', '==', user.uid).get();
    if (snap.empty){
      $('mystages-empty').classList.remove('hidden');
    } else {
      snap.forEach((doc) => list.appendChild(buildMyStageItem(doc.id, doc.data())));
    }
  } catch(e){
    console.warn('내 스테이지 로드 실패:', e);
    $('mystages-empty').textContent = '불러오기에 실패했어요.';
    $('mystages-empty').classList.remove('hidden');
  }
  $('mystages-loading').classList.add('hidden');
}

function buildMyStageItem(id, d){
  const item = document.createElement('div');
  item.className = 'mystage-item cut-card';
  item.innerHTML =
    '<div class="mystage-row">' +
      '<div class="mystage-thumb"><img alt=""></div>' +
      '<span class="mystage-plays">도전 ' + (d.stats ? d.stats.plays : 0) + '</span>' +
    '</div>' +
    '<button class="btn btn-ghost btn-wide" data-action="view">보기</button>' +
    '<button class="btn btn-ghost btn-wide" data-action="link">공유 링크 다시 보기</button>' +
    '<button class="btn btn-ghost btn-wide" data-action="delete">삭제</button>';
  item.querySelector('img').src = d.photoB64;
  item.querySelector('[data-action="view"]').addEventListener('click', () => loadResultView(id));
  item.querySelector('[data-action="link"]').addEventListener('click', () => {
    prompt('공유 링크 (Ctrl+C로 복사하세요)', location.origin + location.pathname + '?s=' + id);
  });
  item.querySelector('[data-action="delete"]').addEventListener('click', () => deleteOwnStage(id, item));
  return item;
}

async function deleteOwnStage(id, itemEl){
  if (!confirm('이 스테이지를 삭제할까요? 되돌릴 수 없어요.')) return;
  try {
    await fb.db.collection('stages').doc(id).delete();
    itemEl.remove();
    if (!$('mystages-list').children.length) $('mystages-empty').classList.remove('hidden');
  } catch(e){
    console.warn('삭제 실패:', e);
    alert('삭제에 실패했어요.');
  }
}
```

- [ ] **Step 5: 로컬 서버로 화면 강제 노출 확인**

1. `python -m http.server 8765` 실행, Chrome으로 접속
2. `javascript_tool`로 다음을 실행해 mystages-panel을 강제 노출(로그인 없이 레이아웃만 확인):

```javascript
document.querySelectorAll('main > section').forEach(s => s.classList.add('hidden'));
document.getElementById('screen-maker').classList.remove('hidden');
document.querySelectorAll('#screen-maker .panel').forEach(p => p.classList.add('hidden'));
document.getElementById('mystages-panel').classList.remove('hidden');
document.getElementById('mystages-empty').classList.remove('hidden');
```

3. 스크린샷으로 "아직 만든 스테이지가 없어요" 문구가 정상 표시되는지 확인
4. 로컬 서버 종료

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "내 스테이지 화면 추가 — 목록·보기·공유 링크 다시 보기·삭제"
```

---

### Task 5: `CLAUDE.md` 배포 절차 갱신

**Files:**
- Modify: `C:\Users\jeodu\Desktop\VibeCoding\kkokkkok-sumeora\CLAUDE.md`

**Interfaces:**
- Consumes: 없음
- Produces: 없음 (문서만)

- [ ] **Step 1: 배포 절차에 Google 로그인 활성화 단계 추가**

`CLAUDE.md:82-87`의 "## 배포 절차" 목록 현재 내용:

```markdown
1. ✅ Firebase 콘솔: 프로젝트 생성 → Authentication **익명 로그인 활성화** → 웹 앱 등록 → config를 `CONFIG.firebase`에 붙여넣기 (Spark 무료 플랜 그대로, 카드 불필요)
2. Firestore Database 생성(프로덕션 모드, 리전 asia-northeast3 권장) → `firestore.rules` 내용을 규칙 탭에 게시 (Storage는 만들지 않는다)
3. 공개 갤러리 복합 인덱스 배포: `firebase deploy --only firestore:rules,firestore:indexes` (firestore.indexes.json에 `isPublic`+`stats.plays` 인덱스가 코드화되어 있어 콘솔 수동 생성 불필요)
4. GitHub 저장소에 `index.html` 푸시 → Settings > Pages 활성화
5. 실기기 검증: 저장→시크릿 창 플레이→두 기기 동시 플레이(stats 유실 확인)→삭제키 삭제→결과 보기 히트맵
6. 예제 스테이지 5개 직접 제작 → ID를 `CONFIG.EXAMPLE_STAGES`에 기입 → 재푸시
```

다음으로 교체 (Google 로그인 활성화 단계를 2번으로 삽입하고 이후 번호를 순차적으로 밀기):

```markdown
1. ✅ Firebase 콘솔: 프로젝트 생성 → Authentication **익명 로그인 활성화** → 웹 앱 등록 → config를 `CONFIG.firebase`에 붙여넣기 (Spark 무료 플랜 그대로, 카드 불필요)
2. Firebase 콘솔: Authentication → Sign-in method → **Google** 활성화 (지원 이메일 설정 필요)
3. Firestore Database 생성(프로덕션 모드, 리전 asia-northeast3 권장) → `firestore.rules` 내용을 규칙 탭에 게시 (Storage는 만들지 않는다)
4. 공개 갤러리 복합 인덱스 배포: `firebase deploy --only firestore:rules,firestore:indexes` (firestore.indexes.json에 `isPublic`+`stats.plays` 인덱스가 코드화되어 있어 콘솔 수동 생성 불필요)
5. GitHub 저장소에 `index.html` 푸시 → Settings > Pages 활성화
6. 실기기 검증: 저장→시크릿 창 플레이→두 기기 동시 플레이(stats 유실 확인)→삭제키 삭제→결과 보기 히트맵→**구글 로그인·내 스테이지·다른 기기 병합 확인**
7. 예제 스테이지 5개 직접 제작 → ID를 `CONFIG.EXAMPLE_STAGES`에 기입 → 재푸시
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "CLAUDE.md 배포 절차에 구글 로그인 활성화 단계 추가"
```

---

### Task 6: 실제 배포 및 왕복 검증

> ⚠️ **실제 Google OAuth 완료 흐름(로그인 버튼 클릭 → 구글 계정 선택 → 동의 → 리다이렉트 복귀)은 이 세션에서 자동화 검증이 불가능하다.** Google이 CDP로 원격 제어되는 Chrome의 로그인 시도를 자동 차단하기 때문("이 브라우저 또는 앱이 안전하지 않을 수 있습니다" 오류). 이 태스크에서는 **Firestore 규칙과 병합 메커니즘의 핵심 로직**(익명 세션 2개로 시뮬레이션 가능)만 실제 프로덕션에서 검증하고, 실제 구글 버튼 클릭·동의 화면 확인은 배포 후 사용자가 실기기로 직접 확인해야 하는 항목으로 남긴다.

**Files:** 없음 (배포 및 검증)

**Interfaces:**
- Consumes: Task 1~5의 모든 변경사항
- Produces: 없음

- [ ] **Step 1: 규칙 배포**

Run: `firebase deploy --only firestore:rules`
Expected: `Deploy complete!`

- [ ] **Step 2: 규칙 시나리오 검증 (익명 세션 2개로 시뮬레이션)**

로컬 서버(`python -m http.server 8765`)로 앱을 열고, `javascript_tool`로 다음을 실행 — 익명 세션 A(현재 페이지의 기본 세션)와 보조 앱 인스턴스로 세션 B를 만들어 전체 규칙 표면을 검증한다:

```javascript
// 세션 A: 테스트 문서 하나 직접 생성 (UI를 거치지 않고 최소 필드로)
const uidA = fb.uid;
const testDoc = await fb.db.collection('stages').add({
  createdAt: firebase.firestore.FieldValue.serverTimestamp(),
  creatorUid: uidA,
  photoB64: 'data:image/jpeg;base64,AAAA',
  dolls: [{ x:0.5, y:0.5, scale:0.1, rotation:0, flip:false, eyedropperUses:0, dollB64:'data:image/png;base64,AAAA' }],
  timeLimit: 60,
  deleteKeyHash: 'test',
  stats: { plays: 0, clears: 0, totalClearMs: 0 },
  taps: [],
});
const id = testDoc.id;

// A: 내 스테이지 목록(list) 조회 — 성공해야 함
const listA = await fb.db.collection('stages').where('creatorUid','==',uidA).get();
const listAOk = listA.docs.some(d => d.id === id);

// 세션 B: 보조 앱 인스턴스로 별도 익명 계정 생성
const secondary = firebase.initializeApp(CONFIG.firebase, 'test-b-' + Date.now());
const credB = await secondary.auth().signInAnonymously();
const dbB = secondary.firestore();
const uidB = credB.user.uid;

// B가 A의 문서를 list로 조회 시도 — 거부되어야 함(permission-denied)
let listBDenied = false;
try { await dbB.collection('stages').where('creatorUid','==',uidA).get(); }
catch(e){ listBDenied = (e.code === 'permission-denied'); }

// B가 A의 문서를 삭제 시도 — 거부되어야 함
let deleteBDenied = false;
try { await dbB.collection('stages').doc(id).delete(); }
catch(e){ deleteBDenied = (e.code === 'permission-denied'); }

// A가 소유권을 B로 이전(병합 시뮬레이션) — 성공해야 함
await fb.db.collection('stages').doc(id).update({ creatorUid: uidB });

// 이제 B가 list/delete 가능해야 함
const listB2 = await dbB.collection('stages').where('creatorUid','==',uidB).get();
const listB2Ok = listB2.docs.some(d => d.id === id);
await dbB.collection('stages').doc(id).delete();
const stillExists = (await fb.db.collection('stages').doc(id).get()).exists;

await secondary.auth().signOut();
await secondary.delete();

JSON.stringify({ listAOk, listBDenied, deleteBDenied, listB2Ok, deletedByB: !stillExists });
```

Expected: `{"listAOk":true,"listBDenied":true,"deleteBDenied":true,"listB2Ok":true,"deletedByB":true}`

이 결과가 곧 Task 1의 규칙과 Task 2의 `mergeAndSignIn()`이 의존하는 핵심 메커니즘(소유자만 list·소유권 이전·삭제 가능)이 프로덕션에서 정확히 동작함을 증명한다.

- [ ] **Step 3: 로그인 버튼 동작(리다이렉트 시작) 확인**

`link-login` 클릭 시 `linkWithRedirect`가 호출되어 페이지가 `accounts.google.com`으로 이동을 시도하는지 확인 (완료까지는 가지 않아도 됨 — URL 이동 시도 자체만 확인). 콘솔 에러 없이 이동 시작되면 통과.

- [ ] **Step 4: 사용자 수동 검증 안내**

다음 항목은 GitHub Pages 배포 + Firebase 콘솔에서 Google 로그인 활성화 후, 실기기에서 사용자가 직접 확인해야 함을 안내한다 (자동화 불가):
- 실제 구글 계정으로 로그인 완료 → "OO님 · 로그아웃"으로 링크 텍스트 변경 확인
- "내 스테이지" 진입 → 방금 만든 스테이지가 목록에 보이는지 확인
- 다른 기기에서 같은 구글 계정으로 로그인 → 두 기기의 스테이지가 병합되어 모두 보이는지 확인 (핵심 시나리오)
- 로그아웃 → 다시 로그인 → 상태 정상 복원 확인

---

## Self-Review 결과
- **스펙 커버리지**: 설계 문서의 인증 흐름(Task 2), 규칙 확장(Task 1), 홈 UI(Task 3), 내 스테이지 화면(Task 4), 콘솔 설정(Task 5) 모두 대응. 설계 문서의 "확인된 제약"(동일 브라우저 세션 한정 병합)은 Global Constraints와 Task 6 안내에 반영.
- **플레이스홀더 스캔**: 없음.
- **타입/네이밍 일관성**: `loginWithGoogle`, `completeAuthRedirect`, `mergeAndSignIn`, `updateAccountUI`, `renderMyStages`, `buildMyStageItem`, `deleteOwnStage`, `link-login`, `link-mystages`, `mystages-panel` 이름이 모든 태스크에서 동일하게 사용됨.
- **실행 가능성 재검토**: 원래 브레인스토밍 논의에서는 "먼저 넘기고 나중에 전환"이라고만 합의했으나, 계획 작성 중 "대상 계정의 uid를 어떻게 알아내는가"라는 기술적 공백을 발견해 **보조 Firebase 앱 인스턴스로 조용히 확인하는 기법**을 구체화했다 — 설계 의도(먼저 이전, 나중에 전환)는 동일하게 지키면서 실제로 구현 가능하도록 보완함.
- **자동화 한계 명시**: 실제 Google OAuth 완료는 자동화 불가라는 점을 Global Constraints와 Task 6에 명확히 기재해, 검증 범위에 대한 오해가 없도록 함.
