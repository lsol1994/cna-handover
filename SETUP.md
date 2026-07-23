[SETUP.md](https://github.com/user-attachments/files/30292621/SETUP.md)
# 업무인수인계 웹사이트 설정 가이드

## 0. 첨부 양식 파일은 Firebase Storage를 쓰지 않아요
Firebase 정책 변경으로 Storage 기능은 결제 카드를 등록하는 Blaze(종량제) 요금제에서만 쓸 수 있게 되었어요(실사용료가 0원이어도 카드 등록은 필수). 완전 무료로 유지하기 위해 이 사이트는 첨부 양식을 **링크로만 연결**하도록 만들었습니다.

양식 파일은 아래 둘 중 편한 방법으로 준비하세요.
- **구글드라이브**: 파일 업로드 → 마우스 오른쪽 클릭 → 공유 → "링크가 있는 모든 사용자"로 설정 → 링크 복사
- **GitHub 저장소**: index.html과 같은 저장소에 `forms/사직서양식.docx` 처럼 파일을 올린 뒤, 파일 페이지에서 "Raw" 버튼을 눌러 나오는 URL 복사

이렇게 얻은 링크를 사이트의 관리자 모드 → 업무 수정 → "첨부 양식"에서 이름과 함께 붙여넣으면 됩니다.

## 1. Firebase 프로젝트 준비
1. https://console.firebase.google.com 에서 새 프로젝트 생성 (또는 기존 프로젝트 사용)
2. **Authentication** → 시작하기 → **이메일/비밀번호** 로그인 방식 사용 설정
3. **Firestore Database** → 데이터베이스 만들기 (프로덕션 모드, edition은 Standard)
4. 프로젝트 설정 → 일반 → "내 앱" → 웹 앱 추가 → SDK 설정 및 구성 값 복사

> Storage는 별도로 켤 필요가 없습니다 (요금제가 Spark로 유지되어 완전 무료).

## 2. index.html에 설정값 입력
`index.html` 상단 `<script type="module">` 안의 아래 부분을 실제 값으로 교체하세요.

```js
const firebaseConfig = {
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  messagingSenderId: "...",
  appId: "..."
};
```

## 3. Firestore 보안 규칙 (Firestore Database → 규칙)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function isAdmin() {
      return request.auth != null &&
        exists(/databases/$(database)/documents/users/$(request.auth.uid)) &&
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'admin';
    }
    match /users/{userId} {
      allow read: if request.auth != null;
      allow write: if isAdmin();
    }
    match /tasks/{taskId} {
      allow read: if request.auth != null;
      allow write: if isAdmin();
    }
    match /settings/{docId} {
      allow read: if request.auth != null;
      allow write: if isAdmin();
    }
  }
}
```

## 4. 최초 관리자 계정 만들기 (최초 1회만)
사이트에는 로그인만으로 관리자를 스스로 만들 수 없도록 되어 있어요(보안 목적). 최초 1명은 콘솔에서 직접 만들어야 합니다.

1. **Authentication** → 사용자 추가
   - 이메일: `아이디@handover.local` 형식 (예: 아이디가 `sol`이면 `sol@handover.local`)
   - 비밀번호: 원하는 값
2. 방금 만든 사용자의 **UID** 복사
3. **Firestore Database** → `users` 컬렉션 생성 → 문서 ID를 방금 복사한 UID로 지정 → 필드 추가
   - `username` (string): `sol`
   - `role` (string): `admin`
   - `active` (boolean): `true`
4. 사이트에서 아이디 `sol`, 비밀번호로 로그인

이후 추가 계정(관리자 포함)은 사이트 안의 **사용자 관리** 화면에서 만들 수 있어요.

## 5. GitHub Pages 배포
1. GitHub 저장소 생성 후 `index.html`을 루트에 업로드
2. 저장소 Settings → Pages → Branch를 `main` (또는 사용하는 브랜치) / `/ (root)` 로 설정
3. 발급된 URL로 접속해 로그인 테스트

## 참고 / 알아두면 좋은 점
- 계정을 **비활성화**하면 로그인은 되어도 사이트 접근은 막힙니다. 완전 삭제는 Firebase 콘솔 Authentication 탭에서 직접 해주세요.
- GitHub Pages는 링크를 아는 사람은 누구나 접속할 수 있는 구조라, 로그인 화면이 1차 방어선입니다. 완전 비공개가 필요하면 저장소를 Private로 두고 Pages 접근을 GitHub 계정으로 제한하는 방법도 있어요(별도 설정 필요).
- 재무/회계 업무의 "마감 일정"은 관리자 모드 → 업무 수정에서 날짜를 등록하면 상단 타임라인에 자동으로 D-day로 표시돼요. "매월"을 선택하면 매달 반복, 특정 월을 선택하면 그 달에 연 1회만 표시됩니다(부가세처럼 1월/7월 두 번이면 날짜를 두 줄 추가하면 돼요).
- 일정마다 "주말/공휴일 조정" 방식을 고를 수 있어요: **이전 영업일로**(급여 등 미리 지급해야 하는 항목), **다음 영업일로**(세금 신고납부 등), **조정 안 함**. 마감일이 토·일요일이나 공휴일과 겹치면 이 설정에 따라 자동으로 날짜가 당겨지거나 밀려요.
- 공휴일 목록은 헤더의 **공휴일 관리**(관리자 전용)에서 확인/추가/삭제할 수 있어요. 2026~2027년 공휴일(대체공휴일 포함)은 이미 등록되어 있고, 2028년 이후나 임시공휴일(선거일 등)은 매년 그때그때 추가해주시면 됩니다.
- 헤더의 **반복업무 가져오기**(관리자 전용)를 누르면, 지난 스케줄 로그에서 분석한 반복 업무 119건(재무/회계 80·노무 28·총무 11)이 업무 항목으로 한 번에 등록됩니다. 일부는 마감일이 자동으로 채워져 있고 "확인 후 확정해주세요"라고 표시되어 있으니, 실제와 다르면 나중에 수정해주세요. **딱 한 번만 실행**해야 중복 등록을 피할 수 있어요.
