# DANDI 프론트엔드

단국대학교 분실물 관리 서비스 DANDI의 프론트엔드입니다.

## 기술 스택

- **Next.js 16** / React 19 / TypeScript
- **Tailwind CSS** / Radix UI
- **Firebase** Authentication (Google 로그인)

---

## 로컬 개발 환경 설정

### 1. 패키지 설치

```bash
npm install
```

### 2. 환경변수 설정

`.env.example`을 참고하여 `.env.local` 파일을 생성합니다.

```env
NEXT_PUBLIC_API_BASE_URL=http://localhost:8080
NEXT_PUBLIC_AUTH_DEMO_MODE=false
NEXT_PUBLIC_FIREBASE_API_KEY=...
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=myauth-26083.firebaseapp.com
NEXT_PUBLIC_FIREBASE_PROJECT_ID=myauth-26083
NEXT_PUBLIC_FIREBASE_APP_ID=...
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=...
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=...
NEXT_PUBLIC_FIREBASE_MEASUREMENT_ID=...
```

> `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN`은 반드시 `myauth-26083.firebaseapp.com` (하이픈 포함)으로 설정해야 Google 로그인이 정상 동작합니다.

### 3. 개발 서버 실행

```bash
npm run dev
```

브라우저에서 `http://localhost:3000` 접속

---

## `altered` 브랜치 수정 사항

로컬 백엔드 통합 테스트(2026-05-11) 중 발견된 버그들을 수정한 브랜치입니다.

### 1. `app/login/page.tsx` — 로그인 시 사용자 정보 세션 저장

로그인 성공 시 Firebase에서 받은 `displayName`, `email`을 세션에 저장합니다.

```ts
setAuthSession({
  accessToken,
  profileCompleted,
  provider: "firebase-google",
  name: credential.user.displayName ?? undefined,
  email: credential.user.email ?? undefined,
});
```

---

### 2. `lib/auth-session.ts` — AuthSession 타입 확장

`AuthSession` 타입에 `name`, `email` 필드를 추가했습니다.

```ts
export type AuthSession = {
  accessToken: string;
  profileCompleted: boolean;
  provider?: "firebase-google";
  name?: string;
  email?: string;
};
```

---

### 3. `app/mypage/page.tsx` — 마이페이지 실명 표시

이름/이메일이 하드코딩(`"홍길동"`, `"example@dankook.ac.kr"`)되어 있던 것을 세션 값으로 교체했습니다.

```tsx
<Input defaultValue={session?.name ?? "홍길동"} readOnly />
<Input defaultValue={session?.email ?? "example@dankook.ac.kr"} readOnly />
```

> 재로그인 후부터 실제 이름/이메일이 표시됩니다.

---

### 4. `app/admin/page.tsx` — Vision API 응답 필드 매핑 수정

백엔드 실제 응답 필드명과 프론트 기대값이 달라 결과가 전부 `-`로 표시되던 문제를 수정했습니다.

| 이전 (잘못됨) | 수정 후 (백엔드 실제 필드) |
|---|---|
| `labels` | `objectLabels` |
| `rgb: {r,g,b}` | `dominantColors[0]` (문자열) |
| `text` | `maskedText` |
| `category` | `documentType` |
| `id` (string) | `id` (number → `String()` 변환) |

`onAnalyzeVision`, `onFetchVisionResult` 두 함수 모두 동일하게 수정했습니다.

---

### 5. `lib/dandi-state.tsx` — Authorization 헤더 자동 첨부

`apiJson` 공통 함수에 `Authorization` 헤더를 자동으로 추가합니다.

```ts
const session = getAuthSession();
const authHeader = session?.accessToken
  ? { Authorization: `Bearer ${session.accessToken}` }
  : {};
```

수령 QR 발급 등 인증이 필요한 API 호출에서 토큰이 누락되어 `Missing or invalid Authorization header` 오류가 발생하던 문제를 수정했습니다.

---

### 6. `components/app-shell.tsx` — Hydration 오류 수정

`getAuthSession()`을 렌더링 중 직접 호출하던 것을 `useEffect` + `useState`로 변경했습니다.

SSR과 클라이언트 렌더링 간 `localStorage` 접근 차이로 발생하던 **Hydration 오류**를 수정합니다.

```tsx
const [session, setSession] = useState<AuthSession | null>(null);
const [mounted, setMounted] = useState(false);

useEffect(() => {
  const s = getAuthSession();
  setSession(s);
  setMounted(true);
}, [router]);
```

---

### 7. `app/lost/[id]/page.tsx` — Next.js 16 params Promise 대응

Next.js 16에서 `params`가 Promise로 변경됨에 따라 `React.use()`로 언래핑합니다.

```tsx
// 이전
export default function LostDetailPage({ params }: { params: { id: string } }) {
  const item = useMemo(() => [...].find((it) => it.id === params.id), [params.id]);

// 수정 후
import { use } from "react";
export default function LostDetailPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = use(params);
  const item = useMemo(() => [...].find((it) => it.id === id), [id]);
```

---

### 환경변수 수정사항 (`.env.local`)

| 항목 | 이전 값 | 수정 값 |
|---|---|---|
| `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` | `myauth26083.firebaseapp.com` | `myauth-26083.firebaseapp.com` |
| `NEXT_PUBLIC_AUTH_DEMO_MODE` | (없음) | `false` |
