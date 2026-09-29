# 리서치: 데스크탑 기술 스택·서명·배포

> 조사일: 2026-09-29. 일부 공식 문서(electron.build, v2.tauri.app, learn.microsoft.com)는 조사 환경에서 직접 열지 못했습니다. 해당 항목은 **(미검증)** 으로 표시했습니다.

## 1. 결론: Next.js 대신 Electron + electron-vite + React

| 기준 | **Electron + electron-vite + React** ✅ | Tauri v2 + React | Nextron (Electron + Next.js) |
|---|---|---|---|
| 번들 크기 | 약 85–200 MB | 약 3–10 MB (OS 웹뷰) | Electron과 비슷하고 Next.js만큼 더 무거움 |
| Node 라이브러리 (노션 SDK, LLM SDK, chokidar) | main 프로세스에서 그대로 사용 | 웹뷰 또는 Node 사이드카 필요. **노션 API는 브라우저 요청(CORS) 차단** → Rust HTTP 플러그인으로 우회해야 함 | 그대로 사용 |
| 시스템 소리 캡처 | Electron 39+ 기본 지원 (맥 ScreenCaptureKit, 윈도우 WASAPI) | Rust나 플러그인 필요 | Electron과 같음 |
| whisper·ffmpeg 실행 | `extraResources` 바이너리를 `child_process`로 실행 | `externalBin` 사이드카 (잘 동작) | Electron과 같음 |
| 자동 업데이트 | electron-updater + GitHub Releases | updater 플러그인 + `latest.json` | Electron과 같음 |
| React/Next.js 개발자 친숙도 | 높음 | 중간 (Rust) | 높지만 계층이 많음 |
| 성숙도 | 매우 높음 (Electron 44.x, electron-vite 5 안정) | 높음 | 보통 (소수 메인테이너, 메인테이너 모집 중) |

### Next.js를 쓰지 않는 이유

- 데스크탑 앱 안에서 Next.js는 **정적 내보내기(`output: 'export'`)** 로만 쓸 수 있습니다.
  - 그러면 **API Routes, Server Actions, SSR, 이미지 최적화를 모두 쓸 수 없습니다.**
  - 결국 남는 건 파일 기반 라우팅뿐이고, 빌드는 더 무겁고 느려집니다.
- 이 앱의 실제 "백엔드" 작업은 모두 **Electron main 프로세스(Node)** 에서 돌아야 합니다. 폴더 감시, STT, LLM 호출, 노션 호출, 파일 저장이 그렇습니다. renderer는 IPC로 요청만 보냅니다.
- React, Tailwind, shadcn/ui 경험은 그대로 이어집니다. 라우팅만 TanStack Router(hash 모드)로 바뀝니다.

### Tauri를 쓰지 않는 이유

- 핵심 파이프라인이 전부 Node 라이브러리입니다. 노션 API는 브라우저 요청을 막아서 Rust나 사이드카로 옮겨야 합니다.
- whisper 모델이 수백 MB~수 GB라서 Tauri의 번들 크기 이점이 의미가 작습니다.

## 2. GitHub Releases 배포 파이프라인

```yaml
# .github/workflows/release.yml (개요)
on: { push: { tags: ['v*'] } }
permissions: { contents: write }
jobs:
  build:
    strategy:
      matrix:
        include:
          - os: macos-latest      # arm64 러너
            args: --mac --arm64
          - os: macos-13          # x64 러너 (ffmpeg-static 등 네이티브 바이너리를 아키텍처별로 받기 위해 분리)
            args: --mac --x64
          - os: windows-latest
            args: --win --x64
    runs-on: ${{ matrix.os }}
    steps:
      - checkout → setup-node → npm ci
      - npm run build && npx electron-builder ${{ matrix.args }} --publish always
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
```

- 산출물:
  - 맥: `dmg`(설치용)와 `zip`(자동 업데이트용)
  - 윈도우: NSIS `exe`
  - 업데이트 메타데이터: `latest.yml`, `latest-mac.yml`
- 릴리스는 **초안(draft)** 으로 올라갑니다. 확인한 뒤 수동으로 공개합니다.
- x64 맥 러너 라벨은 GitHub 쪽 정책에 따라 바뀔 수 있습니다. M1에서 확인합니다.
- 유니버설 빌드도 가능하지만, 네이티브 바이너리 병합 문제가 있어 아키텍처별 빌드를 기본으로 합니다.

## 3. 코드 서명

### macOS

| 단계 | 사용자 경험 | 자동 업데이트 |
|---|---|---|
| **미서명 (arm64)** | "손상되었기 때문에 열 수 없습니다" → 사용 불가 | ❌ |
| **ad-hoc 서명** (`identity: '-'`) — **프로토타입** | 처음 열 때 차단됨 → **시스템 설정 → 개인정보 보호 및 보안 → "그래도 열기"** (앱당 1회) | ❌ (Squirrel.Mac은 서명 필수) → **인앱 "새 버전 알림"**(GitHub Releases API 확인 + 다운로드 링크) |
| **Developer ID 서명 + 공증** ($99/년) — 판매 단계 | 경고 없음 | ✅ electron-updater |

- **Sequoia(15)부터 Control-클릭 → 열기 우회가 없어졌습니다.**
  - README에 "그래도 열기" 절차를 스크린샷과 함께 넣습니다.
  - 대안으로 `xattr -dr com.apple.quarantine /Applications/<앱>.app` 명령도 안내합니다. Tahoe(26)에서는 항상 되지는 않을 수 있습니다 **(미검증)**.
- ⚠️ **ad-hoc 서명의 부작용**: 서명이 빌드마다 바뀌기 때문에 macOS 권한(전체 디스크 접근·마이크 등)과 키체인 접근 허용이 **업데이트할 때마다 초기화될 수 있습니다.**
  - 그래서 v1은 전체 디스크 접근이 필요 없는 흐름을 기본으로 합니다. 음성메모도 "공유 → 감시 폴더" 방식이 기본입니다.
- **판매 전환 시 CI 설정**:
  - 공증용 App Store Connect API 키: `APPLE_API_KEY`, `APPLE_API_KEY_ID`, `APPLE_API_ISSUER`
  - Developer ID 인증서: `CSC_LINK`, `CSC_KEY_PASSWORD`
  - 번들한 whisper·ffmpeg 바이너리도 하드닝된 런타임으로 서명해야 합니다.

### Windows

- 미서명이면 "Windows의 PC 보호" 창이 뜹니다. **추가 정보 → 실행**을 누르면 설치됩니다.
- electron-updater의 NSIS 자동 업데이트는 미서명이어도 동작합니다.
- 2024년부터는 EV 인증서도 SmartScreen 평판을 즉시 받지 못하고, 설치가 쌓이며 평판이 올라갑니다.
- 서명 옵션:

| 옵션 | 비용 | 한국 개인 가능? |
|---|---|---|
| Azure Artifact Signing (구 Trusted Signing) | 월 $9.99~ | ❌ 개인은 미국·캐나다만. 한국은 **법인만** |
| SignPath Foundation | 무료 | 공개 저장소 + OSI 오픈소스 라이선스일 때 |
| Certum Open Source 인증서 | 연 약 €30 + 카드·리더 약 €85 | ✅ |

## 4. API 키 보안 저장

- `keytar`는 2022년 12월부터 보관(archived) 상태로 유지보수가 끝났습니다. VS Code 등도 **Electron `safeStorage`** 로 옮겼습니다.
  - `safeStorage`는 macOS에서 키체인, Windows에서 DPAPI를 씁니다.
- 저장 패턴:
  1. `safeStorage.encryptString`으로 암호화합니다.
  2. 결과를 base64로 앱 설정에 저장합니다.
  3. **main 프로세스에서만 복호화**하고, 키는 renderer로 절대 넘기지 않습니다.

## 5. 기본 저장 경로

- `app.getPath('documents')`
  - 맥: `~/Documents`
  - 윈도우: `C:\Users\<u>\Documents` (OneDrive 문서 폴더 백업을 켜면 OneDrive 아래로 바뀔 수 있음)
- **기본값**: `<문서>/MeetingNotes/<프로젝트>/YYYY/MM/YYYY-MM-DD_회의명.md`
- 앱 내부 데이터는 `app.getPath('userData')`에 둡니다(회의별 JSON·전사본·로그).
  - 맥: `~/Library/Application Support/<앱>`
  - 윈도우: `%APPDATA%\<앱>`

## 6. UI 스택

| 영역 | 선택 |
|---|---|
| 컴포넌트 | shadcn/ui + Tailwind v4 (Vite 그대로 사용 가능) |
| 라우팅 | TanStack Router (hash 모드) |
| 상태 | Zustand + TanStack Query (IPC 호출 캐시) |
| 마크다운 미리보기 | `react-markdown` + `remark-gfm` |
| 자유 섹션 편집 | CodeMirror 6 (마크다운 원문 + 미리보기) — WYSIWYG 에디터는 마크다운을 다시 직렬화하며 형식을 바꿀 수 있어 피함 |
| 다국어 | `i18next` + `react-i18next` (한국어 기본, 키만 영어 대비) |
| 검색 | MiniSearch (로컬 회의록 전문 검색) |

## 출처

- https://github.com/saltyshiomix/nextron · https://nextjs.org/docs/app/guides/static-exports · https://nextjs.org/docs/messages/api-routes-static-export
- https://github.com/alex8088/electron-vite/releases · https://www.pkgpulse.com/guides/electron-vs-tauri-2026
- https://github.com/electron/electron/issues/52738 · https://www.electronjs.org/docs/latest/api/desktop-capturer/
- https://www.electron.build/docs/publish/ · https://www.electron.build/docs/features/auto-update/ · https://www.electronjs.org/docs/latest/api/auto-updater
- https://mjtsai.com/blog/2024/07/05/sequoia-removes-gatekeeper-contextual-menu-override/ · https://support.apple.com/en-us/102445 · https://developer.apple.com/programs/enroll/
- https://learn.microsoft.com/en-us/azure/artifact-signing/faq · https://signpath.org/ · https://shop.certum.eu/code-signing.html
- https://www.todesktop.com/blog/posts/windows-apps-psa-ev-certs-do-not-grant-immediate-reputation-anymore
- https://freek.dev/2103-replacing-keytar-with-electrons-safestorage-in-ray
- https://support.google.com/drive/answer/12178485
