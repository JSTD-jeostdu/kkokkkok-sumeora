# CLAUDE.md — 「꼭꼭 숨어라」

> 모든 작업 전에 이 파일을 먼저 읽는다.

## 프로젝트 한 줄 정의
"내 사진에서 색을 훔쳐 종이인형을 위장시키고, 친구가 그걸 찾아내는 비동기 숨바꼭질 웹앱"

## 개발 규약 (절대 준수)
- **단일 파일 `index.html`** — 빌드 도구·프레임워크 없음, 순수 HTML/CSS/JS
- 외부 의존성은 CDN만 허용 (현재: Google Fonts Gaegu)
- 모든 상수는 파일 최상단 `CONFIG` 객체에 모은다
- 주석은 한국어
- 배포: GitHub Pages / 화면 전환은 JS 섹션 show-hide, 라우팅은 Stage 2에서 `?s=` 쿼리 파라미터
- 디자인 토큰: 도화지 `#FAF7F0` · 연필 `#4A4A4A` · 포인트 레드 `#E45B5B` · Gaegu / 가위 점선 카드 + 종이인형 시그니처
- **모든 화면 최하단 공통 푸터 `made by JSTD`** (작은 글씨, 링크 없음)
- 모바일 퍼스트: 터치 우선, 탭 영역 44px 이상
- 사진과 인형은 절대 합성(굽기)하지 않는다 — 항상 별도 레이어
- stats류 서버 수치 갱신은 `increment()` 전용 (Stage 2부터 적용)

## 3단계 로드맵
- [x] **Stage 1** — 에디터 + 배치 + 로컬 테스트 플레이
- [x] **Stage 2** — Firebase 연동(익명 인증, 리사이즈 업로드, `?s=` URL 로드, 삭제키, 보안 규칙) + 정식 플레이 화면(카운트다운·타이머·발견 연출·stats `increment`)
- [x] **Stage 3** — 결과 화면·위장 등급·탭 히트맵·힌트 3종·공유 카드·예제 스테이지 5개·마감

## 현재 진행 상태 (Stage 1 완료 — 2026-07-18)
구현 완료 항목:
- 화면: 홈(히어로 인형 SVG + 갤러리 placeholder) / 제작(업로드→색칠→배치) / 테스트 플레이 / 공통 푸터
- 사진 처리: `createImageBitmap({imageOrientation:'from-image'})` + `<img>` 폴백, 긴 변 1280px 리사이즈, JPEG q0.8 재인코딩(EXIF·GPS 제거), Blob은 `state.photoBlob`에 보관 (Stage 2 업로드용)
- 색칠 에디터: 256×256 관절 종이인형(Path2D, 홈 SVG와 동일 좌표), 실루엣 클리핑, 브러시 3종(24/12/5px), 지우개(destination-out), 실행취소 10단계(ImageData 스택), 전체 지우기
- 스포이드: **색칠 단계에서 사진 탭 = 즉시 색 추출** (getImageData, 색 팝 피드백, 스와치 갱신)
- 배치: 인형 근처 드래그 이동(오조작 방지용 근접 판정), 크기 4~20% 슬라이더, 회전 0~360°, 좌우반전 / 상태는 정규화 좌표 `doll:{x,y,scale,rotation,flip}`
- 테스트 플레이: 원 판정(바운딩 반경×1.15, 최소 반경 짧은 변 3%), 오답 잔물결, 정답 팔랑 낙하 연출(+진동), 다시 편집 복귀
- 접근성·품질: aria-pressed/aria-label, prefers-reduced-motion 존중, 탭 영역 44px+

### 구현 중 내린 판단 (Stage 2 담당자 참고)
1. **EXIF 수동 파싱을 하지 않음**: 최신 브라우저(iOS Safari 13.4+/Chrome 81+/Firefox 77+)는 디코딩 시 방향을 자동 보정하므로, 수동 파싱을 중첩하면 이중 회전 버그가 난다. `imageOrientation:'from-image'` 명시 + 재인코딩으로 EXIF 제거를 달성.
2. **스포이드를 별도 토글이 아닌 기본 동작으로**: 색칠 단계에서 사진 탭 = 항상 색 추출. "사진 탭 → 색 추출 → 바로 칠하기" 흐름이 끊기지 않는다.
3. **판정을 사각 바운딩 대신 원 판정으로**: 회전한 인형의 축정렬 박스 계산보다 단순하고, 여유 반경(×1.15)·최소 반경 규칙을 그대로 만족한다.
4. `photoCanvas` 내부 해상도 = 리사이즈된 사진 픽셀. CSS로만 축소 표시하므로 정규화 좌표 변환이 단순하다(창 크기 무관).

## Stage 2 완료 (2026-07-18)
구현 완료 항목:
- **Firebase 기반**: compat SDK 10.12.2 CDN, `CONFIG.firebase` placeholder(비면 오프라인 모드 — 제작·테스트는 계속 동작), `signInAnonymously`
- **저장·공유**: auto-ID → Storage 업로드(photo.jpg ≤300KB, doll.png) → 삭제키 8자리(혼동 문자 제외) 생성·SHA-256 해시만 저장·원문 1회 표시 → Firestore 문서(정규화 doll, timeLimit 60, stats 0, taps []) → 공유 URL + 클립보드 복사(execCommand 폴백), 진행 스피너·실패 재시도
- **`?s=` 라우팅**: 로드 → 플레이 직행, 문서 없음/실패 시 에러 화면, 홈 이동 시 `history.replaceState`로 파라미터 정리
- **정식 플레이**: 블러 + 3-2-1 카운트다운, 타이머 바(잔여 10초 레드+맥박), 모든 탭 정규화 기록, 정답 팔랑 낙하(+진동 패턴), 시간 초과 시 위치 공개(점선 링 3회 펄스), 임시 결과(시간+나도 숨기기/홈)
- **기록**: 단일 트랜잭션 — taps 읽기→concat→최근 500개 슬라이스 + `stats.plays/clears/totalClearMs`를 `increment()`로 동시 기록 (원자적, last-write-wins 없음)
- **삭제**: 키 입력 → SHA-256 비교 → Storage 2파일(개별 try) + 문서 삭제
- **보안 규칙**: `firestore.rules`(update는 stats·taps 필드만, delete는 클라 검증 한계 주석), `storage.rules`(크기·contentType 제한, 그 외 경로 전면 금지)

### Stage 2에서 내린 판단 (Stage 3 담당자 참고)
1. **stats와 taps를 하나의 트랜잭션으로 통합**: 프롬프트는 update+트랜잭션 2회였지만, 트랜잭션 안에서 increment를 함께 쓰면 1회 원자적 기록이 되어 더 안전하다.
2. **플레이 시작 전 사진 블러**: 카운트다운 중 미리 훑어보는 편법 방지 + 시작 연출을 겸한다.
3. **`btn-play-make`(나도 숨기기)는 URL 파라미터를 정리하고 업로드 화면으로 직행** — Stage 3 이어달리기 체인의 접점.
4. 플레이 종료 기록 실패는 console.warn만 하고 UX를 막지 않는다.

## Stage 3 완료 — 프로젝트 마감 (2026-07-18)
구현 완료 항목:
- **힌트 3종** (각 1회, 개인 기록 전용 — stats 미저장): ① 사분면(점선 십자+해당 사분면 1.5초 하이라이트) ② 온도계(이후 매 탭 거리 피드백 — 색이 아닌 텍스트+거리별 진동, 색약 접근성) ③ 반짝(dollPath를 인형 위치·회전·크기로 변환해 아웃라인 0.5초 스트로크)
- **정식 결과 화면**: 소요 시간(0.1초 단위)·힌트 ★·남은 시간 게이지·위장 등급 뱃지+한 줄 평(S~D, clears 0이면 "미개봉 위장"). 등급은 이번 플레이를 로컬 합산한 stats로 계산해 서버 기록과 일치
- **공유 카드**: 1080×1080 canvas — 도화지 결+가위 점선 테두리+타이틀+흰 실루엣 인형(위장색 노출 금지: 색이 곧 힌트)+기록+등급 뱃지+`made by JSTD`. Web Share API(files) → PNG 다운로드 폴백. **사진·인형 위치·위장색 미포함**
- **제작자 결과 보기** `?s={id}&view=result`: 정답 위치에 인형 표시 + taps 히트맵(반투명 포인트 레드 원, 라이브러리 없이 캔버스) + 도전/발견/평균 시간/등급 요약 + 삭제 링크. 공유 화면에서 플레이 링크와 별도로 발급하며 "혼자 간직하세요" 경고 표기
- **예제 갤러리**: `CONFIG.EXAMPLE_STAGES` 배열 기반. 강한 블러 썸네일(정답·내용 노출 방지)+도전 수+등급, 탭 시 `?s=` 이동. 배열이 비었거나 전부 로드 실패 시 갤러리 자체 숨김
- **마감 점검**: 320~1440px(본문 max-width 520 중앙), aria-label/aria-live, 비동기 전 구간 스피너·재시도, 썸네일 lazy load, 전 화면 푸터, prefers-reduced-motion

## 아키텍처 변경 — Storage 제거 (2026-07-18, 배포 준비 중)
**배경**: 2024년 10월부터 신규 프로젝트의 Cloud Storage는 Blaze 요금제(카드 등록) 필수. 사용자가 "진짜 무료" 유지를 선택.
**변경**: 사진·인형을 base64 dataURL로 Firestore 문서에 직접 저장 (`photoB64`, `dollB64` 필드 — `photoPath`/`dollPath` 대체). Storage SDK·업로드·다운로드 URL·storage.rules 전부 제거.
**핵심 안전장치**: 문서 1MB 제한 대응 — 사진은 `PHOTO_ENCODE_STEPS`(1280px q0.8 → 900px q0.5) 적응형 압축으로 700KB 이하 보장, 인형 PNG는 150KB 초과 시 192px 축소 재시도, 규칙에서도 `photoB64.size() < 800000` 이중 검증. 예산: photo 700KB + doll 150KB + taps ~20KB + 메타 < 1MB.
**부수 효과**: dataURL은 CORS·캔버스 오염 문제가 없어 loadImg가 단순해졌고, 스테이지 삭제가 문서 삭제 1회로 끝난다. 갤러리 썸네일도 문서에서 바로 읽는다. Firestore Spark 무료 한도(저장 1GB ≈ 스테이지 약 1,200개, 읽기 5만/일) 내 운영.

## 배포 절차 (GitHub Pages — 사용자 직접 수행)
1. ✅ Firebase 콘솔: 프로젝트 생성 → Authentication **익명 로그인 활성화** → 웹 앱 등록 → config를 `CONFIG.firebase`에 붙여넣기 (Spark 무료 플랜 그대로, 카드 불필요)
2. Firestore Database 생성(프로덕션 모드, 리전 asia-northeast3 권장) → `firestore.rules` 내용을 규칙 탭에 게시 (Storage는 만들지 않는다)
3. GitHub 저장소에 `index.html` 푸시 → Settings > Pages 활성화
4. 실기기 검증: 저장→시크릿 창 플레이→두 기기 동시 플레이(stats 유실 확인)→삭제키 삭제→결과 보기 히트맵
5. 예제 스테이지 5개 직접 제작 → ID를 `CONFIG.EXAMPLE_STAGES`에 기입 → 재푸시

## 백로그 (구현하지 않음 — 기록만)
- 미끼 인형 2개 (가짜 실루엣, 누르면 "속았지?" 연출)
- 인형 포즈 3종 (서있기/앉기/매달리기)
- 타이머 옵션 (30·60·120초 — 데이터 모델의 timeLimit은 이미 대응)
- 이어달리기 체인 (클리어 → 제작 직행 연결 강화)
- 공개 갤러리 + 신고 누적 3회 자동 숨김
- Cloud Functions 도입 시: 삭제키 서버 검증으로 전환 (firestore.rules 주석 참고)
