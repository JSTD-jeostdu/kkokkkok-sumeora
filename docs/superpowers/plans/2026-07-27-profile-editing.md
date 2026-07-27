# 프로필 수정(닉네임·아바타) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 로그인 사용자가 앱 안에서 닉네임과 아바타 이미지를 설정/변경할 수 있게 한다.

**Architecture:** 새 Firestore 컬렉션 `users/{uid}`에 `{nickname, avatarB64, updatedAt}`을 저장한다. 홈 화면 계정 영역의 "로그인" 버튼은 로그인 상태에서 아바타+닉네임을 보여주는 진입점으로 바뀌고, 클릭 시 새 `#profile-panel`로 이동한다. 로그아웃은 프로필 패널 안으로 옮긴다. 아바타 이미지는 기존 사진 업로드의 적응형 압축 패턴(`PHOTO_ENCODE_STEPS`)을 훨씬 작은 목표치로 재사용해 중앙 정사각 크롭 → 160px 고정 → 품질 단계적 하향으로 인코딩한다.

**Tech Stack:** 단일 파일 `index.html`(순수 JS, 빌드 없음), Firebase compat SDK(Firestore, Auth), `firestore.rules`.

## Global Constraints

- 단일 파일 `index.html` — 빌드 도구·프레임워크 없음
- 모든 상수는 `CONFIG` 객체에 모은다
- 주석은 한국어
- 모바일 퍼스트: 탭 영역 44px 이상
- 사진(아바타)과 텍스트(닉네임)는 신뢰할 수 없는 사용자 입력이다 — DOM 삽입 시 `innerHTML` 문자열 조합 금지, `textContent`/`createElement`만 사용 (XSS 방지)
- 비동기 실패는 `console.warn` 후 안전한 기본값으로 폴백 — UX를 막지 않는다(기존 방침과 동일)
- 익명 사용자에게는 이 기능이 전혀 노출되지 않는다
- 테스트 프레임워크 없음 — 각 태스크 검증은 `python -m http.server`로 로컬 구동 후 브라우저(Chrome 자동화 또는 수동)로 확인한다. 실제 Firebase 프로젝트 왕복이 필요한 검증은 Task 6에서 한 번에 모아 수행한다(크레딧 절약 방침).

---

## 파일 구조

전부 기존 세 파일을 수정한다(신규 파일 없음):
- `index.html` — HTML(홈 계정 영역, 신규 `#profile-panel`), CSS(신규 `.mini-avatar`, `.profile-avatar-*`, `#profile-panel input`), JS(`CONFIG` 상수, 아바타 인코딩 유틸, 프로필 read/write 헬퍼, `updateAccountUI`/`renderAccountLink` 리팩터, 프로필 패널 배선)
- `firestore.rules` — 신규 `users/{uid}` match 블록

---

### Task 1: 아바타 인코딩 유틸

**Files:**
- Modify: `index.html:702-707` (CONFIG)
- Modify: `index.html` (`encodeDollB64ForIndex` 함수 뒤, 원본 기준 1239~1250줄 부근)

**Interfaces:**
- Produces: `CONFIG.AVATAR_SIZE`(160), `CONFIG.AVATAR_B64_MAX`(40000), `CONFIG.AVATAR_QUALITY_STEPS`([0.8,0.6,0.4]); `async function fileToAvatarB64(file): Promise<string|null>` — 성공 시 JPEG data URI, 이미지를 열 수 없으면 `loadOrientedBitmap`이 예외를 던짐(호출자가 catch), 압축 후에도 예산 초과 시에만 `null` 반환
- Consumes: 기존 `loadOrientedBitmap(file)`(index.html:1186), `document.createElement('canvas')` 패턴(`encodeDollB64ForIndex`와 동일)

- [ ] **Step 1: CONFIG에 아바타 상수 추가**

`index.html`의 기존 코드:
```js
  PHOTO_ENCODE_STEPS: [
    { max: 1280, q: 0.8 }, { max: 1280, q: 0.65 },
    { max: 1024, q: 0.7 }, { max: 1024, q: 0.55 },
    { max: 900,  q: 0.5 },
  ],
  DEFAULT_COLOR: '#E45B5B',
```

다음으로 교체:
```js
  PHOTO_ENCODE_STEPS: [
    { max: 1280, q: 0.8 }, { max: 1280, q: 0.65 },
    { max: 1024, q: 0.7 }, { max: 1024, q: 0.55 },
    { max: 900,  q: 0.5 },
  ],
  // 프로필 아바타: 정사각 고정 크기(160px) + 품질만 단계적으로 낮추는 훨씬 작은 목표 용량
  AVATAR_SIZE: 160,
  AVATAR_B64_MAX: 40000,
  AVATAR_QUALITY_STEPS: [0.8, 0.6, 0.4],
  DEFAULT_COLOR: '#E45B5B',
```

- [ ] **Step 2: 아바타 인코딩 함수 추가**

`index.html`의 기존 코드(`encodeDollB64ForIndex` 끝부분):
```js
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

다음 함수를 그 바로 뒤에 추가:
```js
/* 프로필 아바타 인코딩 — 원본 사진 처리(loadOrientedBitmap)와 별개 경로.
   중앙 기준 정사각 크롭 → CONFIG.AVATAR_SIZE 고정 리사이즈 → 품질 단계적 하향 JPEG. */
async function fileToAvatarB64(file){
  const bmp = await loadOrientedBitmap(file);
  const srcW = bmp.width || bmp.naturalWidth, srcH = bmp.height || bmp.naturalHeight;
  const side = Math.min(srcW, srcH);
  const sx = (srcW - side) / 2, sy = (srcH - side) / 2;
  const c = document.createElement('canvas');
  c.width = c.height = CONFIG.AVATAR_SIZE;
  c.getContext('2d').drawImage(bmp, sx, sy, side, side, 0, 0, CONFIG.AVATAR_SIZE, CONFIG.AVATAR_SIZE);
  if (bmp.close) bmp.close();
  for (const q of CONFIG.AVATAR_QUALITY_STEPS){
    const url = c.toDataURL('image/jpeg', q);
    if (url.length <= CONFIG.AVATAR_B64_MAX) return url;
  }
  return null; // 극히 예외적인 경우(거의 발생하지 않음)
}
```

- [ ] **Step 3: 로컬 확인**

`python -m http.server`로 로컬 구동 후 브라우저 콘솔에서:
```js
console.log(CONFIG.AVATAR_SIZE, CONFIG.AVATAR_B64_MAX, typeof fileToAvatarB64);
```
기대 출력: `160 40000 "function"`

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "프로필 아바타 인코딩 유틸 추가 (CONFIG 상수 + fileToAvatarB64)"
```

---

### Task 2: 프로필 Firestore 헬퍼 + 보안 규칙

**Files:**
- Modify: `index.html` (`mergeAndSignIn` 함수 뒤, `updateAccountUI` 함수 앞 — 원본 기준 855줄 부근)
- Modify: `firestore.rules`

**Interfaces:**
- Produces: `async function loadProfile(uid): Promise<{nickname:string, avatarB64:string|null}|null>` (문서 없음/읽기 실패 시 `null`), `async function saveProfile(uid, nickname, avatarB64): Promise<void>` (실패 시 예외를 던짐 — 호출자가 처리)
- Consumes: `fb.db`(이미 정의됨), `firebase.firestore.FieldValue.serverTimestamp()`(기존 `saveStage`가 쓰는 것과 동일 패턴, index.html:1851)

- [ ] **Step 1: read/write 헬퍼 추가**

`index.html`의 기존 코드:
```js
// 로그인 상태에 따라 홈 화면 링크 텍스트·"내 스테이지" 링크 표시 갱신
function updateAccountUI(){
```

다음으로 교체(헬퍼 함수를 앞에 삽입):
```js
/* ---------- 프로필(닉네임·아바타) ---------- */
async function loadProfile(uid){
  try {
    const doc = await fb.db.collection('users').doc(uid).get();
    return doc.exists ? doc.data() : null;
  } catch(e){
    console.warn('프로필 로드 실패:', e);
    return null;
  }
}
async function saveProfile(uid, nickname, avatarB64){
  await fb.db.collection('users').doc(uid).set({
    nickname,
    avatarB64: avatarB64 || null,
    updatedAt: firebase.firestore.FieldValue.serverTimestamp(),
  }, { merge: true });
}

// 로그인 상태에 따라 홈 화면 링크 텍스트·"내 스테이지" 링크 표시 갱신
function updateAccountUI(){
```

(Task 3에서 `updateAccountUI` 자체를 async로 리팩터한다 — 이 스텝에서는 헬퍼만 추가)

- [ ] **Step 2: `firestore.rules`에 `users/{uid}` 규칙 추가**

`firestore.rules`의 기존 코드:
```
      // 삭제: 기존엔 "인증만 되면 누구나" 삭제 가능했던 허점을 소유자 본인으로 강화.
      // 지금까지도 삭제는 항상 스테이지를 만든 바로 그 세션(창작자 본인)에서만 이뤄져 왔으므로
      // 기존 삭제키 UX에는 영향이 없고, ID만 알아도 삭제할 수 있었던 허점만 막는다.
      allow delete: if request.auth != null && request.auth.uid == resource.data.creatorUid;
    }
  }
}
```

다음으로 교체:
```
      // 삭제: 기존엔 "인증만 되면 누구나" 삭제 가능했던 허점을 소유자 본인으로 강화.
      // 지금까지도 삭제는 항상 스테이지를 만든 바로 그 세션(창작자 본인)에서만 이뤄져 왔으므로
      // 기존 삭제키 UX에는 영향이 없고, ID만 알아도 삭제할 수 있었던 허점만 막는다.
      allow delete: if request.auth != null && request.auth.uid == resource.data.creatorUid;
    }

    // 프로필(닉네임·아바타) — 로그인 사용자 전용, 본인만 읽고 쓸 수 있다.
    // 지금은 이 정보가 어디에도 공개 노출되지 않으므로 목록 조회·타인 조회는 전면 금지.
    match /users/{uid} {
      allow get: if request.auth != null && request.auth.uid == uid;
      allow list: if false;

      allow create, update: if request.auth != null && request.auth.uid == uid
        && request.resource.data.keys().hasAll(['nickname'])
        && request.resource.data.nickname is string
        && request.resource.data.nickname.size() >= 2
        && request.resource.data.nickname.size() <= 12
        && (!('avatarB64' in request.resource.data)
            || request.resource.data.avatarB64 == null
            || (request.resource.data.avatarB64 is string
                && request.resource.data.avatarB64.size() < 50000
                // data: URI만 허용 — 임의 원격 URL을 넣어 추적 픽셀 등으로 악용하는 것을 방지
                && request.resource.data.avatarB64.matches('^data:image/.*')));

      allow delete: if false;
    }
  }
}
```

- [ ] **Step 3: 로컬 확인**

브라우저 콘솔에서 함수가 정의됐는지만 확인(실제 Firestore 왕복은 Task 6):
```js
console.log(typeof loadProfile, typeof saveProfile);
```
기대 출력: `"function" "function"`

`firestore.rules`는 문법을 눈으로 재확인 — 기존 `stages` 블록과 중괄호 짝이 맞는지, `match /users/{uid}`가 `match /databases/{database}/documents {` 블록 안에 있는지 확인.

- [ ] **Step 4: 커밋**

```bash
git add index.html firestore.rules
git commit -m "프로필 Firestore 헬퍼(loadProfile/saveProfile) + users/{uid} 보안 규칙 추가"
```

---

### Task 3: 홈 계정 영역 — 아바타 표시로 전환

**Files:**
- Modify: `index.html:104-107` (CSS — `.account-link`에 gap 추가, `.mini-avatar` 신설)
- Modify: `index.html` (`updateAccountUI` → `renderAccountLink` 분리 + async 버전, 원본 기준 856~871줄)
- Modify: `index.html:2587-2598` (`link-login` 클릭 핸들러)

**Interfaces:**
- Produces: 전역 `let profileCache`(`{nickname,avatarB64}|null`), `function renderAccountLink()`(재조회 없이 `profileCache`+현재 `firebase.auth().currentUser` 기준으로 DOM만 갱신), `async function updateAccountUI()`(인증 상태 변화 시 `loadProfile`로 재조회 후 `renderAccountLink()` 호출) — 기존 시그니처 이름은 유지하되 이제 `await`가 필요해짐
- Consumes: Task 2의 `loadProfile`

- [ ] **Step 1: CSS 추가**

`index.html`의 기존 코드:
```css
.account-link{ background:none; border:none; font-family:inherit; font-size:14px;
  color:var(--pencil-soft); text-decoration:underline; cursor:pointer;
  padding:0 4px; min-height:44px; display:inline-flex; align-items:center; }
```

다음으로 교체:
```css
.account-link{ background:none; border:none; font-family:inherit; font-size:14px;
  color:var(--pencil-soft); text-decoration:underline; cursor:pointer;
  padding:0 4px; min-height:44px; display:inline-flex; align-items:center; gap:6px; }
.account-link:has(.mini-avatar){ text-decoration:none; }
.mini-avatar{ width:26px; height:26px; border-radius:50%; overflow:hidden; flex:0 0 auto;
  background:var(--pencil-soft); color:#fff; font-size:13px; font-weight:700;
  display:inline-flex; align-items:center; justify-content:center; }
.mini-avatar img{ width:100%; height:100%; object-fit:cover; }
```

(`:has()`는 대상 브라우저 기준 신뢰 가능 — 실패해도 밑줄이 남는 것뿐이라 저하가 안전함)

- [ ] **Step 2: `updateAccountUI`를 `renderAccountLink` + async `updateAccountUI`로 분리**

`index.html`의 기존 코드:
```js
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

다음으로 교체:
```js
// 로그인 상태일 때 현재 사용자의 프로필({nickname,avatarB64}) 캐시. 비로그인이면 null.
let profileCache = null;

// profileCache + 현재 인증 상태만으로 홈 화면 계정 영역 DOM을 다시 그린다(재조회 없음).
// 닉네임은 사용자 입력값이라 textContent만 사용한다 — innerHTML 문자열 조합 금지(XSS 방지).
function renderAccountLink(){
  const loginLink = $('link-login');
  const mystagesLink = $('link-mystages');
  if (!loginLink || !mystagesLink) return;
  const user = firebase.auth().currentUser;
  loginLink.innerHTML = '';
  if (user && !user.isAnonymous){
    const nickname = (profileCache && profileCache.nickname) || user.displayName || '회원';
    const avatar = document.createElement('span');
    avatar.className = 'mini-avatar';
    if (profileCache && profileCache.avatarB64){
      const img = document.createElement('img');
      img.src = profileCache.avatarB64;
      img.alt = '';
      avatar.appendChild(img);
    } else {
      avatar.textContent = nickname.charAt(0);
    }
    const nameSpan = document.createElement('span');
    nameSpan.textContent = nickname;
    loginLink.appendChild(avatar);
    loginLink.appendChild(nameSpan);
    loginLink.dataset.mode = 'account';
    mystagesLink.classList.remove('hidden');
  } else {
    loginLink.textContent = '로그인';
    loginLink.dataset.mode = 'login';
    mystagesLink.classList.add('hidden');
  }
}

// 로그인/로그아웃 등 인증 상태가 바뀔 때 프로필을 다시 읽어와 계정 영역을 갱신.
async function updateAccountUI(){
  const user = firebase.auth().currentUser;
  profileCache = (user && !user.isAnonymous) ? await loadProfile(user.uid) : null;
  renderAccountLink();
}
```

- [ ] **Step 3: `link-login` 클릭 핸들러 수정 — 로그인 상태에서는 로그아웃 대신 프로필 패널로 이동**

`index.html`의 기존 코드:
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
```

다음으로 교체(로그아웃 로직은 Task 5의 `logoutAccount()`로 이동):
```js
$('link-login').addEventListener('click', () => {
  if ($('link-login').dataset.mode === 'account'){
    openProfilePanel();
  } else {
    loginWithGoogle();
  }
});
```

`openProfilePanel`은 Task 5에서 정의된다 — 이 시점엔 아직 없으므로 Step 4의 로컬 확인에서는 호출 자체(클릭)까지는 하지 않는다.

- [ ] **Step 4: 로컬 확인**

`python -m http.server`로 로컬 구동 후(오프라인 모드 — `CONFIG.firebase`가 비어 있지 않다면 실제 익명 로그인이 시도됨):
- 페이지 로드 후 홈 화면에 "로그인" 텍스트가 그대로 보이는지 확인(비로그인 기본 상태 회귀 없음)
- 콘솔에서 `typeof renderAccountLink, typeof updateAccountUI` → `"function" "function"`

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "홈 계정 영역을 텍스트 링크에서 아바타+닉네임 표시로 전환"
```

---

### Task 4: 프로필 편집 패널 UI (HTML+CSS, 아직 미배선)

**Files:**
- Modify: `index.html` (mystages-panel 뒤, 원본 기준 619~626줄)
- Modify: `index.html` (`#delete-msg` CSS 뒤, 원본 기준 366~368줄)
- Modify: `index.html:1136-1158` (`PHASE_LABELS`, `panels` map)

**Interfaces:**
- Produces: DOM 요소 `#profile-panel`, `#profile-avatar-preview`, `#btn-profile-pick`, `#profile-avatar-input`, `#profile-nickname-input`, `#profile-msg`, `#btn-profile-cancel`, `#btn-profile-save`, `#btn-profile-logout`; `PHASE_LABELS.profile`, `panels.profile`(setPhase('profile')로 이 패널을 보여줄 수 있게 됨)
- Consumes: 없음(이 태스크는 아직 이벤트를 배선하지 않는다 — Task 5에서 배선)

- [ ] **Step 1: HTML 추가**

`index.html`의 기존 코드:
```html
    <!-- 내 스테이지 (로그인 사용자 전용) -->
    <div id="mystages-panel" class="panel hidden">
      <h3>내 스테이지 📂</h3>
      <p id="mystages-loading" class="hint-line">불러오는 중…</p>
      <p id="mystages-empty" class="hint-line hidden">아직 만든 스테이지가 없어요.</p>
      <div id="mystages-list"></div>
    </div>
  </section>
```

다음으로 교체:
```html
    <!-- 내 스테이지 (로그인 사용자 전용) -->
    <div id="mystages-panel" class="panel hidden">
      <h3>내 스테이지 📂</h3>
      <p id="mystages-loading" class="hint-line">불러오는 중…</p>
      <p id="mystages-empty" class="hint-line hidden">아직 만든 스테이지가 없어요.</p>
      <div id="mystages-list"></div>
    </div>

    <!-- 프로필 편집 (로그인 사용자 전용) -->
    <div id="profile-panel" class="panel solo hidden">
      <h3>프로필 편집 🖊️</h3>
      <div class="profile-avatar-row">
        <div class="profile-avatar-preview" id="profile-avatar-preview" aria-label="프로필 사진 미리보기"></div>
        <button class="btn btn-ghost" id="btn-profile-pick">사진 바꾸기</button>
        <input type="file" id="profile-avatar-input" accept="image/*" hidden>
      </div>
      <label class="hint-line" for="profile-nickname-input">닉네임</label>
      <input type="text" id="profile-nickname-input" maxlength="12" autocomplete="off" aria-label="닉네임">
      <p id="profile-msg" aria-live="polite"></p>
      <div class="panel-actions">
        <button class="btn btn-ghost" id="btn-profile-cancel">취소</button>
        <button class="btn btn-primary" id="btn-profile-save">저장</button>
      </div>
      <button class="link-btn" id="btn-profile-logout">로그아웃</button>
    </div>
  </section>
```

- [ ] **Step 2: CSS 추가**

`index.html`의 기존 코드:
```css
#delete-panel input{ width:100%; font-family:inherit; font-size:22px; letter-spacing:.2em; text-align:center;
  border:2.5px solid var(--pencil); border-radius:12px; padding:10px; background:#fff; text-transform:uppercase; }
#delete-msg{ text-align:center; font-size:17px; min-height:24px; margin:8px 0 0; color:var(--accent); }
```

다음으로 교체:
```css
#delete-panel input{ width:100%; font-family:inherit; font-size:22px; letter-spacing:.2em; text-align:center;
  border:2.5px solid var(--pencil); border-radius:12px; padding:10px; background:#fff; text-transform:uppercase; }
#delete-msg{ text-align:center; font-size:17px; min-height:24px; margin:8px 0 0; color:var(--accent); }

/* ---------- 프로필 편집 ---------- */
.profile-avatar-row{ display:flex; align-items:center; gap:14px; margin-bottom:14px; }
.profile-avatar-preview{ width:88px; height:88px; border-radius:50%; overflow:hidden; flex:0 0 auto;
  background:var(--pencil-soft); color:#fff; font-size:32px; font-weight:700;
  display:flex; align-items:center; justify-content:center; }
.profile-avatar-preview img{ width:100%; height:100%; object-fit:cover; }
#profile-panel input[type=text]{ width:100%; font-family:inherit; font-size:18px;
  border:2.5px solid var(--pencil); border-radius:10px; padding:10px; background:#fff; margin-bottom:2px; }
#profile-msg{ text-align:center; font-size:15px; min-height:22px; margin:4px 0 0; color:var(--accent); }
```

- [ ] **Step 3: `PHASE_LABELS`, `panels` map에 `profile` 추가**

`index.html`의 기존 코드:
```js
const PHASE_LABELS = {
  upload:'사진 고르기', paint:'위장 색칠', place:'숨길 곳 정하기', test:'테스트 플레이',
  save:'저장 중', share:'공유하기', load:'불러오는 중', ready:'준비!', play:'찾아라!',
  end:'결과', view:'결과 보기', error:'앗', 'delete':'스테이지 삭제', mystages:'내 스테이지',
};
```

다음으로 교체:
```js
const PHASE_LABELS = {
  upload:'사진 고르기', paint:'위장 색칠', place:'숨길 곳 정하기', test:'테스트 플레이',
  save:'저장 중', share:'공유하기', load:'불러오는 중', ready:'준비!', play:'찾아라!',
  end:'결과', view:'결과 보기', error:'앗', 'delete':'스테이지 삭제', mystages:'내 스테이지',
  profile:'프로필 편집',
};
```

`index.html`의 기존 코드:
```js
  const panels = {
    upload: uploadBox, paint: paintPanel, place: placePanel, test: testPanel,
    save: $('save-panel'), share: $('share-panel'), load: $('load-panel'),
    ready: $('play-panel'), play: $('play-panel'), end: $('playend-panel'),
    view: $('result-panel'), error: $('error-panel'), 'delete': $('delete-panel'),
    mystages: $('mystages-panel'),
  };
```

다음으로 교체:
```js
  const panels = {
    upload: uploadBox, paint: paintPanel, place: placePanel, test: testPanel,
    save: $('save-panel'), share: $('share-panel'), load: $('load-panel'),
    ready: $('play-panel'), play: $('play-panel'), end: $('playend-panel'),
    view: $('result-panel'), error: $('error-panel'), 'delete': $('delete-panel'),
    mystages: $('mystages-panel'), profile: $('profile-panel'),
  };
```

- [ ] **Step 4: 로컬 확인**

`python -m http.server`로 로컬 구동 후 브라우저 콘솔에서 강제로 화면 전환해 레이아웃만 눈으로 확인(아직 버튼 배선 전이라 클릭은 동작 안 함):
```js
setPhase('profile');
```
기대: 상단바 라벨이 "프로필 편집"으로 바뀌고, 아바타 원(빈 상태)·"사진 바꾸기" 버튼·닉네임 입력창·저장/취소 버튼·로그아웃 링크가 카드 안에 표시됨. 320px 폭에서도 버튼이 겹치지 않는지 확인.

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "프로필 편집 패널 UI 추가 (아직 미배선)"
```

---

### Task 5: 프로필 편집 동작 배선

**Files:**
- Modify: `index.html` (`updateAccountUI` 뒤 — Task 3에서 재배치된 위치 바로 뒤에 새 함수들 추가)
- Modify: `index.html:2587-2599` 부근 (이벤트 리스너)

**Interfaces:**
- Produces: `function openProfilePanel()`, `function renderProfileAvatarPreview()`, `async function handleProfileAvatarFile(file)`, `async function saveProfileEdit()`, `function cancelProfileEdit()`, `async function logoutAccount()`
- Consumes: Task 1의 `fileToAvatarB64`, Task 2의 `saveProfile`, Task 3의 `profileCache`/`renderAccountLink`/`updateAccountUI`, Task 4의 DOM 요소들

- [ ] **Step 1: 프로필 편집 함수 추가**

`renderAccountLink`/`updateAccountUI` 정의부 바로 뒤(Task 3에서 편집한 지점)에 추가:
```js
// 편집 중인 아바타(저장 전 임시값). null = 아바타 없음, 문자열 = data URI(기존값 또는 새로 고른 사진).
let profileEditAvatarB64 = null;

function openProfilePanel(){
  const user = firebase.auth().currentUser;
  if (!user || user.isAnonymous) return;
  $('profile-nickname-input').value = (profileCache && profileCache.nickname) || user.displayName || '';
  profileEditAvatarB64 = (profileCache && profileCache.avatarB64) || null;
  $('profile-msg').textContent = '';
  renderProfileAvatarPreview();
  setPhase('profile');
}

// profileEditAvatarB64(또는 없으면 닉네임 이니셜)로 편집 화면의 미리보기만 갱신 — 저장 전 로컬 상태.
function renderProfileAvatarPreview(){
  const el = $('profile-avatar-preview');
  el.innerHTML = '';
  if (profileEditAvatarB64){
    const img = document.createElement('img');
    img.src = profileEditAvatarB64;
    img.alt = '';
    el.appendChild(img);
  } else {
    const nickname = $('profile-nickname-input').value.trim();
    el.textContent = nickname ? nickname.charAt(0) : '?';
  }
}

async function handleProfileAvatarFile(file){
  if (!file || !file.type.startsWith('image/')){ alert('이미지 파일을 골라주세요.'); return; }
  let b64;
  try { b64 = await fileToAvatarB64(file); }
  catch(e){ alert('사진을 불러오지 못했어요. 다른 사진으로 시도해 주세요.'); return; }
  if (!b64){ alert('이 사진은 처리할 수 없었어요. 다른 사진으로 시도해 주세요.'); return; }
  profileEditAvatarB64 = b64;
  renderProfileAvatarPreview();
}

async function saveProfileEdit(){
  const nickname = $('profile-nickname-input').value.trim();
  if (nickname.length < 2 || nickname.length > 12){
    $('profile-msg').textContent = '닉네임은 2~12자로 입력해 주세요.';
    return;
  }
  const user = firebase.auth().currentUser;
  if (!user || user.isAnonymous) return;
  $('btn-profile-save').disabled = true;
  try {
    await saveProfile(user.uid, nickname, profileEditAvatarB64);
    profileCache = { nickname, avatarB64: profileEditAvatarB64 };
    renderAccountLink(); // 방금 쓴 값을 이미 알고 있으므로 재조회 없이 바로 반영
    setPhase('home');
  } catch(e){
    console.warn('프로필 저장 실패:', e);
    $('profile-msg').textContent = '저장에 실패했어요. 다시 시도해 주세요.';
  }
  $('btn-profile-save').disabled = false;
}

function cancelProfileEdit(){
  setPhase('home');
}

async function logoutAccount(){
  await firebase.auth().signOut();
  const cred = await firebase.auth().signInAnonymously();
  fb.uid = cred.user.uid;
  await updateAccountUI();
  setPhase('home');
  renderGallery();
  renderPublicGallery();
}
```

- [ ] **Step 2: 이벤트 리스너 연결**

`index.html`의 기존 코드(Task 3에서 이미 수정된 상태):
```js
$('link-login').addEventListener('click', () => {
  if ($('link-login').dataset.mode === 'account'){
    openProfilePanel();
  } else {
    loginWithGoogle();
  }
});
$('link-mystages').addEventListener('click', () => renderMyStages());
```

다음으로 교체(뒤에 프로필 패널 리스너 추가):
```js
$('link-login').addEventListener('click', () => {
  if ($('link-login').dataset.mode === 'account'){
    openProfilePanel();
  } else {
    loginWithGoogle();
  }
});
$('link-mystages').addEventListener('click', () => renderMyStages());
$('btn-profile-pick').addEventListener('click', () => $('profile-avatar-input').click());
$('profile-avatar-input').addEventListener('change', (e) => {
  const f = e.target.files && e.target.files[0];
  e.target.value = '';
  if (f) handleProfileAvatarFile(f);
});
$('profile-nickname-input').addEventListener('input', () => {
  if (!profileEditAvatarB64) renderProfileAvatarPreview(); // 이니셜 표시 중이면 입력에 맞춰 갱신
});
$('btn-profile-save').addEventListener('click', saveProfileEdit);
$('btn-profile-cancel').addEventListener('click', cancelProfileEdit);
$('btn-profile-logout').addEventListener('click', logoutAccount);
```

- [ ] **Step 3: 로컬 확인**

`python -m http.server`로 로컬 구동 후 브라우저에서(오프라인 모드여도 UI 흐름은 확인 가능):
- 콘솔에서 `firebase.auth().currentUser.isAnonymous = false`를 흉내낼 수 없으므로, 대신 `openProfilePanel`·`saveProfileEdit`·`cancelProfileEdit`·`handleProfileAvatarFile`·`logoutAccount`가 모두 함수로 정의됐는지 확인:
```js
[openProfilePanel, renderProfileAvatarPreview, handleProfileAvatarFile, saveProfileEdit, cancelProfileEdit, logoutAccount].map(f => typeof f)
```
기대: 전부 `"function"`
- 닉네임 입력창에 타이핑하면(아바타 미설정 상태) 미리보기 원 안의 이니셜이 실시간으로 바뀌는지 눈으로 확인(`setPhase('profile')`로 진입 후 확인 — 로그인 가드 없이도 화면 자체는 렌더링됨)

실제 로그인 상태에서의 전체 흐름(로그인 → 아바타 클릭 → 사진 변경 → 저장 → 홈 화면 반영)은 Task 6에서 실기기로 검증한다.

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "프로필 편집 패널 동작 배선 (사진 변경·저장·취소·로그아웃)"
```

---

### Task 6: 규칙 배포 + 실기기 통합 검증

**Files:** 없음(배포 명령 + 수동 검증)

이 태스크는 실제 Firebase 프로젝트에 규칙을 배포하고, 실제 Google 로그인으로 전체 흐름을 검증한다. 여러 번 나눠 왕복하지 않고 아래 체크리스트를 한 세션에서 모아 확인한다(크레딧 절약 방침).

- [ ] **Step 1: 규칙 배포**

```bash
firebase deploy --only firestore:rules
```

- [ ] **Step 2: 실기기 체크리스트**

1. 실제 배포 도메인에서 Google 로그인 → 홈 화면 계정 영역이 "OO님 · 로그아웃" 텍스트가 아니라 **이니셜 원 아바타 + 닉네임**으로 바뀌는지 확인
2. 아바타/닉네임 클릭 → 프로필 편집 패널 진입 확인
3. 닉네임을 1글자로 바꾸고 저장 → "닉네임은 2~12자로 입력해 주세요." 인라인 에러 확인(화면 유지, alert 아님)
4. 닉네임을 정상 범위(예: 2~12자)로 바꾸고 저장 → 홈 화면으로 돌아오며 아바타 옆 닉네임이 즉시 반영되는지 확인
5. "사진 바꾸기"로 실제 사진 선택 → 미리보기가 정사각으로 크롭된 아바타로 즉시 바뀌는지 확인 → 저장 → 홈 화면 아바타에 반영되는지 확인
6. 페이지 새로고침 후 다시 로그인 상태 확인 시 저장한 닉네임·아바타가 유지되는지 확인(Firestore에서 다시 읽어옴)
7. 개발자 도구 콘솔에서 직접 `fb.db.collection('users').doc(firebase.auth().currentUser.uid).set({nickname:'ab', avatarB64:'x'.repeat(60000)})` 시도 → 규칙에 의해 거부(permission-denied)되는지 확인(용량 초과 방어 확인)
8. 프로필 패널의 "로그아웃" 클릭 → 홈 화면으로 돌아가고 계정 영역이 다시 "로그인" 텍스트로 바뀌는지 확인
9. 로그아웃 상태(익명)에서는 계정 영역에 아바타/프로필 진입점이 전혀 보이지 않는지 재확인(기존 저마찰 제작 플로우 회귀 없음)

- [ ] **Step 3: 커밋 (필요 시)**

체크리스트 통과 중 발견된 사소한 수정이 있었다면:
```bash
git add index.html firestore.rules
git commit -m "실기기 검증 중 발견된 프로필 기능 수정"
```

---

## Self-Review 결과 (계획 작성자 자체 점검)

- **스펙 커버리지**: 설계 문서의 데이터 모델(§1)→Task 2, 아바타 처리(§2)→Task 1, UI/흐름(§3)→Task 3·4·5, 규칙(§4)→Task 2, 에러 처리(§5)→Task 5(저장 실패 alert, 이미지 로드 실패 alert)·Task 3(프로필 읽기 실패는 `loadProfile` 내부에서 이미 폴백) 전부 반영됨.
- **플레이스홀더 스캔**: 없음 — 모든 스텝에 실제 코드 포함.
- **타입/이름 일관성**: `profileCache`, `renderAccountLink`, `updateAccountUI`, `fileToAvatarB64`, `loadProfile`, `saveProfile`, `openProfilePanel`, `handleProfileAvatarFile`, `saveProfileEdit`, `cancelProfileEdit`, `logoutAccount` — 전 태스크에서 동일한 이름·시그니처로 참조됨을 재확인.
- **스펙 대비 추가된 사항**: `firestore.rules`의 `avatarB64` 검증에 `matches('^data:image/.*')`를 스펙 초안보다 추가함 — data URI가 아닌 임의 원격 URL을 넣어 추적 픽셀 등으로 악용하는 것을 막기 위한 방어적 강화이며, 스펙의 의도(본인만 쓰기 가능한 작은 이미지 필드)와 충돌하지 않음.
