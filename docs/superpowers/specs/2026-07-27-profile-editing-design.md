# 프로필 수정(닉네임·아바타) — 설계

## 배경
로그인 시스템(`docs/superpowers/specs/2026-07-26-login-system-design.md`)에서 Google 로그인·"내 스테이지" 관리가 구현됐지만, 로그인 상태는 텍스트 링크("OO님 · 로그아웃")로만 표시되고 사용자가 앱 안에서 직접 바꿀 수 있는 프로필 정보가 없다. 로그인 설계 문서에도 "닉네임 등 별도 프로필 문서는 범위 밖(백로그)"으로 명시돼 있었다 — 이번이 그 백로그를 착수하는 작업이다.

## 범위
- 로그인 사용자가 닉네임과 아바타 이미지를 앱 안에서 직접 설정/변경
- 새 `users/{uid}` Firestore 컬렉션 신설
- 홈 화면 계정 영역을 텍스트 링크에서 아바타+닉네임 표시로 교체

**범위 밖:**
- 프로필 정보의 타 화면 노출(공개 갤러리 카드, 결과 화면 등에 제작자 닉네임 표시) — 지금은 홈 계정 영역에만 표시
- 다른 사용자의 프로필 조회(공개 프로필 개념 자체가 없음)
- 닉네임 중복 검사, 부적절 단어 필터링

## 1. 데이터 모델

새 컬렉션 `users/{uid}` (문서 ID = Firebase Auth uid):

```
{
  nickname: string,          // 2~12자
  avatarB64: string | null,  // data URI (JPEG), 없으면 null
  updatedAt: serverTimestamp
}
```

문서는 로그인 시점엔 생성하지 않는다. 사용자가 프로필 편집 화면에서 처음 저장할 때만 `set(..., {merge:true})`로 생성된다(빈 문서 남발 방지). 문서가 없는 동안 UI는 Google 계정의 `displayName`을 기본 닉네임으로, 아바타는 이니셜 원으로 대체 표시한다.

## 2. 아바타 이미지 처리

기존 사진 업로드의 `PHOTO_ENCODE_STEPS` 적응형 압축 패턴을 재사용하되, 아바타는 훨씬 작은 목표치로 축소한다.

- `CONFIG`에 상수 추가: `AVATAR_SIZE: 160`(정사각형 한 변 px), `AVATAR_B64_MAX: 40000`(바이트)
- 파일 선택 → 이미지 로드 → 중앙 기준 정사각 크롭(짧은 변 기준) → 160×160로 리사이즈 → JPEG 인코딩
- 목표 크기(`AVATAR_B64_MAX`) 초과 시 화질을 단계적으로 낮춰 재인코딩(예: q0.8 → 0.6 → 0.4, 크기는 160px 고정 — 이미 작아서 크기까지 줄일 필요는 없음)
- 별도 수동 크롭 UI 없음 — 자동 크롭 결과를 미리보기로 즉시 보여주고, 마음에 안 들면 "사진 바꾸기"로 다시 선택

## 3. UI · 화면 흐름

### 홈 화면 계정 영역 (`home-account-row`)
- **로그아웃 상태**: 기존과 동일하게 "로그인" 텍스트 버튼
- **로그인 상태**: 텍스트 링크 대신 **원형 아바타(24~28px) + 닉네임**으로 교체. 아바타 없으면 닉네임 첫 글자를 넣은 이니셜 원으로 대체.
- 아바타/닉네임 영역 클릭 → 프로필 편집 패널(`#profile-panel`)로 이동. 로그아웃 버튼은 더 이상 이 자리에 없음(아래로 이동).
- "내 스테이지" 링크는 기존 위치·동작 그대로 유지.

### 신규 `#profile-panel`
- 아바타 미리보기(원형, 편집 중인 이미지를 즉시 반영)
- "사진 바꾸기" 버튼 → 파일 선택 → 자동 크롭·리사이즈 → 미리보기 갱신(이 시점엔 아직 저장 안 됨, 로컬 미리보기만)
- 닉네임 입력 필드(초기값: 기존 저장된 닉네임, 없으면 Google `displayName`)
- "저장" / "취소" 버튼(`.btn.btn-primary` / `.btn.btn-ghost`, 기존 패턴 재사용)
- 하단에 작은 "로그아웃" 링크(기존 로그인 버튼이 겸하던 로그아웃 기능은 여기로 이동)

### 저장 동작
1. 닉네임 공백/길이(2~12자) 검증 — 실패 시 인라인 에러 메시지, 저장 막지 않고 재입력 유도
2. `users/{uid}` 문서를 `set({nickname, avatarB64, updatedAt: serverTimestamp()}, {merge:true})`로 갱신
3. 성공 → `updateAccountUI()`로 홈 화면 아바타·닉네임 갱신 → 이전 화면(홈)으로 복귀
4. 실패 → `alert`로 안내, 편집 화면 유지(입력 내용 보존)

## 4. Firestore 규칙 추가 (`firestore.rules`)

```
match /users/{uid} {
  // 본인만 자기 프로필을 읽는다 — 지금은 프로필이 어디에도 공개 노출되지 않으므로 타인 조회 불허
  allow get: if request.auth != null && request.auth.uid == uid;
  allow list: if false;

  allow create, update: if request.auth != null && request.auth.uid == uid
    && request.resource.data.nickname is string
    && request.resource.data.nickname.size() >= 2
    && request.resource.data.nickname.size() <= 12
    && (!('avatarB64' in request.resource.data)
        || request.resource.data.avatarB64 == null
        || (request.resource.data.avatarB64 is string
            && request.resource.data.avatarB64.size() < 50000));

  allow delete: if false;
}
```

## 5. 에러 처리
- 프로필 문서 읽기 실패(오프라인, 권한 오류 등) → `console.warn`만 하고 기본값(Google `displayName`, 이니셜 아바타)으로 폴백. 기존 방침("비동기 실패는 UX를 막지 않는다")과 동일.
- 저장 실패 → alert 안내, 입력 내용 보존.
- 이미지 로드/크롭 실패(손상된 파일 등) → alert 안내, 기존 아바타 유지.

## 확인된 제약
- 아바타는 최대 40KB 목표(문서 크기 예산 여유 큼 — `stages` 문서의 700KB/150KB 예산과 무관한 별도의 작은 문서라 Firestore 무료 한도에 미치는 영향 미미).
- 익명 사용자에게는 이 기능이 전혀 노출되지 않는다(로그인 상태에서만 접근 가능한 화면).

## 영향받지 않는 것
- 기존 로그인/병합/내 스테이지 로직 전체
- `stages` 컬렉션 및 그 규칙
- 저마찰 제작 플로우("사진 고르기 → 바로 제작")
