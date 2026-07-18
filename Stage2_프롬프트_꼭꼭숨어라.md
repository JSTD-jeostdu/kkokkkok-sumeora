# 「꼭꼭 숨어라」 Stage 2 — Claude Code 프롬프트

> 전제: Stage 1 완료 (에디터+배치+로컬 테스트 플레이, `CLAUDE.md` 존재)
> 아래 구분선 이하 전체를 Claude Code에 붙여넣으세요.

---

너는 시니어 프론트엔드 개발자야. 「꼭꼭 숨어라」의 Stage 2를 구현한다.

## 0. 시작 전 필수 작업

1. **`CLAUDE.md`를 먼저 읽고** 프로젝트 정의·규약·Stage 1 완료 상태를 파악한다.
2. `index.html`의 현재 구조(특히 `doll` 상태 객체, 리사이즈 파이프라인, 테스트 플레이 판정 로직)를 확인한 뒤 작업을 시작한다.
3. 작업 완료 시 `CLAUDE.md`의 진행 상태를 Stage 2 완료로 갱신한다.

## 1. 규약 재확인 (Stage 1과 동일, 절대 준수)

- 단일 파일 `index.html`, 순수 HTML/CSS/JS, CDN만 허용
- 상수는 `CONFIG` 객체, 주석은 한국어, 모바일 퍼스트
- 모든 화면 최하단 공통 푸터 `made by JSTD` 유지
- 사진과 인형은 절대 합성(굽기)하지 않는다 — 별도 레이어 렌더 유지

## 2. Stage 2 구현 범위

### A. Firebase 연동 기반
- Firebase **compat SDK** (app, auth, firestore, storage)를 CDN `<script>`로 추가
- `CONFIG.firebase`에 firebaseConfig placeholder를 두고, 값 채우는 위치를 한국어 주석으로 안내
- 앱 시작 시 `signInAnonymously()` — 실패 시 "저장 기능을 사용할 수 없어요" 안내 후 로컬 기능(제작·테스트 플레이)은 계속 동작 (오프라인 폴백)

### B. 스테이지 저장 (제작 화면에 "저장하고 공유" 버튼 추가)
저장 버튼 클릭 시 순서:
1. `stageId` 생성: Firestore `stages` 컬렉션의 auto-ID 사용
2. Storage 업로드:
   - `stages/{stageId}/photo.jpg` — Stage 1에서 리사이즈된 JPEG Blob (긴 변 1280px, q0.8)
   - `stages/{stageId}/doll.png` — 색칠된 인형 캔버스를 256×256 투명 PNG로 export
3. 삭제키 발급: 8자리 영숫자 랜덤 문자열 생성 → **SHA-256 해시**(`crypto.subtle.digest`)만 Firestore에 저장, 원문은 사용자에게 1회 표시
4. Firestore `stages/{stageId}` 문서 생성:
   ```js
   {
     createdAt: serverTimestamp(),
     creatorUid,                       // 익명 uid
     photoPath, dollPath,              // Storage 경로
     doll: { x, y, scale, rotation, flip },  // Stage 1 정규화 좌표 그대로
     timeLimit: 60,
     deleteKeyHash,
     stats: { plays: 0, clears: 0, totalClearMs: 0 },
     taps: []
   }
   ```
5. 완료 화면 표시: 공유 URL(`{현재 origin+path}?s={stageId}`) + 복사 버튼(`navigator.clipboard`, 실패 시 선택 가능한 텍스트 폴백) + 삭제키 안내("이 키는 다시 볼 수 없어요, 꼭 저장하세요")
- 업로드 중 진행 상태 표시(스피너 + 단계 문구), 실패 시 재시도 버튼

### C. 스테이지 로드 (`?s=` 라우팅)
- 페이지 로드 시 `URLSearchParams`로 `s` 파라미터 확인
- 있으면: Firestore 문서 조회 → Storage에서 photo.jpg / doll.png 다운로드 URL 획득 → 플레이 화면으로 직행
- 문서가 없거나 로드 실패 시: "이 스테이지는 사라졌어요" 안내 + 홈으로 이동 버튼
- 없으면: 기존 홈 화면 표시

### D. 정식 플레이 화면 (Stage 1 테스트 플레이를 확장·분리)
- 시작 전: 사진을 블러 처리한 상태에서 "3, 2, 1" 카운트다운 → 블러 해제와 동시에 타이머 시작
- 상단 타이머 바: `timeLimit`(기본 60초) 카운트다운, 남은 10초부터 색이 포인트 레드로 변하고 맥박 애니메이션
- 탭 판정: Stage 1 로직 재사용 (바운딩 박스 + 15% 여유, 최소 반경 보장)
- **모든 탭을 정규화 좌표로 로컬 배열에 기록** (정답·오답 불문) — 게임 종료 시 한 번에 서버 반영
- 오답: 잔물결 애니메이션 / 정답: 발견 연출(Stage 1 애니메이션 재사용) + 소요 시간 표시
- 시간 초과(실패): "시간 종료!" 후 인형 위치를 아웃라인 반짝임으로 공개
- 종료 시 서버 기록 (성공/실패 공통, **반드시 단일 `update` 호출로**):
  ```js
  stats.plays: increment(1)
  // 성공 시에만 추가:
  stats.clears: increment(1)
  stats.totalClearMs: increment(소요ms)
  // 탭 기록: arrayUnion이 아닌 트랜잭션으로 읽기 → 기존 taps + 이번 탭들 → 최근 500개 슬라이스 후 저장
  ```
  - last-write-wins로 stats를 덮어쓰는 코드 절대 금지 — 모든 수치는 `increment()`만 사용
- 종료 후 임시 결과 표시: 성공 여부 + 소요 시간 + "나도 숨기기" 버튼(제작 화면으로) — 등급·히트맵 등 정식 결과 화면은 Stage 3 범위이므로 만들지 않는다

### E. 스테이지 삭제
- 완료 화면과 플레이 화면 하단에 작은 "스테이지 삭제" 링크
- 삭제키 입력 → SHA-256 해시 비교 → 일치 시 Firestore 문서 삭제 + Storage 파일 2개 삭제
- 불일치 시 "키가 맞지 않아요" (시도 횟수 제한 없음, 민감 데이터 아님)

### F. 보안 규칙 (별도 파일로 생성)
프로젝트 루트에 `firestore.rules`와 `storage.rules` 파일을 생성하고, Firebase 콘솔에 붙여넣는 방법을 한국어 주석으로 안내:
- Firestore `stages/{id}`: read 전체 허용 / create는 인증된 사용자 + `creatorUid == request.auth.uid` / update는 `stats`·`taps` 필드만 허용(다른 필드 변경 거부) / delete는 클라이언트 금지 대신 — 삭제키 검증이 클라이언트에서 이뤄지므로 delete는 인증 사용자에게 허용하되 한계를 주석으로 명시
- Storage: read 전체 허용 / write는 인증 사용자 + 파일 크기 상한(photo 500KB, doll 100KB) + contentType 이미지 제한

## 3. 완료 기준 (모두 충족 시 Stage 2 종료)

1. 사진 업로드 → 색칠 → 배치 → 저장 → URL 복사까지 끊김 없이 동작한다
2. 발급된 URL을 새 시크릿 창에서 열면 카운트다운 후 플레이가 시작된다
3. 성공/실패 각각 종료 후 Firestore의 stats 수치가 `increment`로 정확히 누적된다 (두 기기에서 동시에 플레이해도 유실 없음)
4. 삭제키로 스테이지가 삭제되고, 삭제된 URL 접속 시 안내 화면이 뜬다
5. Firebase 미설정/오프라인 상태에서도 제작·테스트 플레이는 동작한다
6. 푸터 `made by JSTD`가 새로 추가된 화면(완료·플레이·삭제)에도 모두 표시된다
7. `firestore.rules`, `storage.rules` 파일이 존재하고 `CLAUDE.md`가 Stage 2 완료로 갱신되어 있다

## 4. 하지 말 것

- 위장 등급 산정, 히트맵 시각화, 힌트 3종, 공유 카드, 예제 스테이지 구현 금지 (Stage 3 범위)
- stats를 읽어와 더한 뒤 `set`으로 덮어쓰는 패턴 금지 — `increment()` 전용
- 사진·인형 합성 금지, 외부 그림 라이브러리 금지 (Stage 1 규약 유지)
- 이메일/소셜 로그인 추가 금지 — 익명 인증만 사용
