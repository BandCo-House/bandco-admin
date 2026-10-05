# BandCo 어드민 콘솔

BandCo(JamPlay) 운영용 데스크톱 웹 앱이다. 서비스 앱·백엔드가 있는 본 저장소 [BandCo-House/BandCo](https://github.com/BandCo-House/BandCo)와 분리된 저장소이며, Vercel에도 별도 프로젝트로 배포하고 서브도메인 `admin.<서비스 도메인>`으로 연다.

- 어드민 API(`/admin/*`)는 본 저장소의 백엔드(`backend/src/modules/admin`)에 있다. API를 바꾸는 작업은 본 저장소 PR과 이 저장소 PR이 함께 나간다.
- 처음 코드는 본 저장소 PR #251의 `admin/` 폴더에서 이력째 옮겨 왔다.

- 스택: Vite + React 19 + TypeScript, TanStack Router(코드 기반 라우트 트리), TanStack Query, axios, Tailwind CSS v4, lucide-react, sonner
- API 계약: [backend/docs/backend/api-docs/admin.md](https://github.com/BandCo-House/BandCo/blob/dev/backend/docs/backend/api-docs/admin.md) (본 저장소)
- 운영 문서: [backend/docs/backend/operations/admin-console.md](https://github.com/BandCo-House/BandCo/blob/dev/backend/docs/backend/operations/admin-console.md) (시크릿·첫 계정·로컬 확인)

## 어드민 계정

어드민 계정은 서비스 회원(`users`)과 **완전히 분리된** `admin_users` 계정이다. 서비스 회원 계정으로는 로그인할 수 없고, 회원가입도 없다.

- 첫 계정(SUPER_ADMIN)은 본 저장소 백엔드에서 CLI로 만든다. 대상 DB를 셸에서 직접 지정해야 하며 자세한 방법은 위 운영 문서를 본다.

  ```bash
  # BandCo 저장소에서
  cd backend
  pnpm run admin:create
  ```

- 이후 계정은 SUPER_ADMIN이 콘솔의 **어드민 계정** 메뉴에서 만든다.
- 역할
  - `SUPER_ADMIN`(최고 관리자): 모든 기능
  - `OPERATOR`(운영자): 어드민 계정 관리, 서비스 설정 변경, 전체 알림 발송을 제외한 모든 기능

## 접근 경로와 배포

어드민 콘솔은 서비스와 다른 서브도메인으로 연다(예: `https://admin.<서비스 도메인>`).

- **서비스 도메인의 `/admin` 경로로 열지 않는 이유**: 같은 출처(origin)가 되면 서비스 앱에서 실행되는 스크립트(XSS나 오염된 npm 의존성)가 어드민 토큰이 든 `localStorage`를 그대로 읽을 수 있다. 서브도메인은 출처가 달라 저장소가 분리된다.
- Vercel 프로젝트: 이 저장소를 Import, Root Directory는 기본값(저장소 루트), Framework Vite, Output Directory `dist`(기본값), 환경 변수 `VITE_API_BASE_URL`=운영 백엔드 주소. 공개 저장소라 개인 Hobby 계정에서도 가져올 수 있다.
- 패키지 매니저는 `package.json`의 `packageManager`(pnpm 10)로 고정한다. 본 저장소 루트와 같은 버전이다.
- 도메인: Vercel 프로젝트 설정 → Domains에서 `admin.<서비스 도메인>`을 추가하고, 안내대로 DNS에 CNAME을 등록한다. 서비스 앱과 다른 Vercel 계정이면 소유 확인용 TXT 레코드를 함께 요구할 수 있다.
- 백엔드 CORS는 출처를 제한하지 않아 서브도메인에서도 API를 바로 호출할 수 있다.
- 서비스 앱 로그인 화면에는 어드민으로 가는 링크를 두지 않았다. 운영자는 주소로 직접 들어간다.

## 환경 변수

| 이름                | 설명            | 기본값                  |
| ------------------- | --------------- | ----------------------- |
| `VITE_API_BASE_URL` | 백엔드 API 주소 | `http://localhost:3000` |

로컬에서 다른 주소를 쓰려면 `admin/.env.local`(git 미추적)에 적는다.

```bash
VITE_API_BASE_URL=http://localhost:3000
```

## 실행

```bash
pnpm install
pnpm dev          # http://localhost:5174
```

백엔드는 본 저장소에서 따로 띄운다(기본 주소 `http://localhost:3000`).

## 빌드·검증

```bash
pnpm run build    # tsc -b && vite build → dist/
pnpm run lint     # eslint + prettier --check
pnpm test -- --run
```

## 구조

```
src/
  app/        라우터, provider, 레이아웃(사이드바·상단 바), URL 검색 파라미터 검증
  pages/      화면 단위 컴포넌트
  features/   영역별 API 함수·React Query 훅·타입·라벨·전용 컴포넌트
  shared/     API 클라이언트(토큰 재발급), 토큰 저장소, 포맷 유틸, 공용 UI
```

## 인증 동작

- 로그인 성공 시 access/refresh 토큰을 `localStorage`의 `bandco-admin.accessToken`, `bandco-admin.refreshToken`에 저장한다.
- 401을 받으면 refresh 토큰으로 `POST /admin/auth/token/access`를 한 번 호출해 재시도한다. 동시에 여러 요청이 401을 받아도 재발급은 한 번만 일어난다.
- 재발급에 실패하면 토큰을 지우고 `/login`으로 보낸다.
- 화면의 모든 날짜는 KST(`YYYY-MM-DD HH:mm`)로 표시하고, `datetime-local` 입력도 KST로 해석해 ISO로 보낸다.
