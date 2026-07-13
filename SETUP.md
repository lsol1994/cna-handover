# 업무인수인계 웹사이트 설정 가이드

## 1. Firebase 프로젝트 준비
1. https://console.firebase.google.com 에서 새 프로젝트 생성 (또는 기존 프로젝트 사용)
2. **Authentication** → 시작하기 → **이메일/비밀번호** 로그인 방식 사용 설정
3. **Firestore Database** → 데이터베이스 만들기 (프로덕션 모드)
4. **Storage** → 시작하기 (첨부 양식 파일 저장용)
5. 프로젝트 설정 → 일반 → "내 앱" → 웹 앱 추가 → SDK 설정 및 구성 값 복사

## 2. index.html에 설정값 입력
`index.html` 상단 `<script type="module">` 안의 아래 부분을 실제 값으로 교체하세요.

```js
const firebaseConfig = {
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  storageBucket: "...",
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
  }
}
```

## 4. Storage 보안 규칙 (Storage → 규칙)

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /forms/{allPaths=**} {
      allow read: if request.auth != null;
      allow write: if request.auth != null &&
        firestore.get(/databases/(default)/documents/users/$(request.auth.uid)).data.role == 'admin';
    }
  }
}
```

## 5. 최초 관리자 계정 만들기 (최초 1회만)
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

## 6. GitHub Pages 배포
1. GitHub 저장소 생성 후 `index.html`을 루트에 업로드
2. 저장소 Settings → Pages → Branch를 `main` (또는 사용하는 브랜치) / `/ (root)` 로 설정
3. 발급된 URL로 접속해 로그인 테스트

## 참고 / 알아두면 좋은 점
- 계정을 **비활성화**하면 로그인은 되어도 사이트 접근은 막힙니다. 완전 삭제는 Firebase 콘솔 Authentication 탭에서 직접 해주세요.
- GitHub Pages는 링크를 아는 사람은 누구나 접속할 수 있는 구조라, 로그인 화면이 1차 방어선입니다. 완전 비공개가 필요하면 저장소를 Private로 두고 Pages 접근을 GitHub 계정으로 제한하는 방법도 있어요(별도 설정 필요).
- 재무/회계 업무의 "마감 일정"은 관리자 모드 → 업무 수정에서 날짜를 등록하면 상단 타임라인에 자동으로 D-day로 표시돼요. "매월"을 선택하면 매달 반복, 특정 월을 선택하면 그 달에 연 1회만 표시됩니다(부가세처럼 1월/7월 두 번이면 날짜를 두 줄 추가하면 돼요).
