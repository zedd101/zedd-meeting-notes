# 리서치: 녹음 소스별 저장 경로·포맷·권한

> 조사일: 2026-09-29.
> 일부 사이트는 조사 환경의 네트워크 제한으로 직접 열지 못해 검색 결과 요약에 의존했습니다. 그런 항목은 **(미검증)** 으로 표시했습니다.
> 실제 경로는 설정 마법사가 사용자 PC에서 탐지해서 확인합니다.

## 요약: v1 지원 소스

| 소스 | 맥 기본 경로 | 윈도우 기본 경로 | 포맷 | v1 처리 방식 |
|---|---|---|---|---|
| **Alt** | 비공개 (탐지 필요) | 비공개 | 비공개 | **내보내기 폴더** 감시 (오디오 또는 전사 텍스트) |
| **Zoom 로컬 녹화** | `~/Documents/Zoom/<yyyy-mm-dd hh.mm.ss 회의명>/` | `C:\Users\<u>\Documents\Zoom\...` | `audio_only.m4a`, `*.mp4`, `chat.txt` | 프리셋 감시 (`**/audio_only*.m4a`) |
| **맥 음성메모** (아이폰 iCloud 동기화 포함) | `~/Library/Group Containers/group.com.apple.VoiceMemos.shared/Recordings/` | – | `.m4a` | 기본: "공유 → 감시 폴더에 저장". 고급: 전체 디스크 접근 후 직접 감시 (복사만) |
| **윈도우 녹음기** | – | `C:\Users\<u>\Documents\Sound recordings\` (OneDrive로 옮겨질 수 있음) | `.m4a` | 프리셋 감시 |
| **AirDrop** (아이폰 → 맥) | `~/Downloads` | – | `.m4a` 등 | 프리셋 감시 (오디오 확장자만) |
| **QuickTime / OBS / Audacity** | 사용자 지정 (OBS 기본 `~/Movies`) | 사용자 지정 (OBS 기본 `Videos`) | m4a / mkv·mp4 / wav·mp3 | 사용자 폴더 감시 |
| **직접 드래그앤드롭** | – | – | 모든 지원 포맷 | 수동 가져오기 |

**지원 입력 확장자**
- 오디오·영상: `.m4a` `.mp3` `.wav` `.aac` `.flac` `.ogg` `.opus` `.mp4` `.mov` `.webm` `.mkv`
- 전사 텍스트(STT 생략): `.txt` `.md` `.srt` `.vtt`

---

## 1. Alt (altalt.io)

- **정체**: KAIST 출신 3인 팀이 만든 **온디바이스 AI 노트앱**입니다. 2025년 11월 Show HN에 공개됐습니다.
  - 녹음, 100개 언어 실시간 전사, 화자 구분(화자별 요약), 핵심 요약, 실시간 번역, YouTube 링크 전사를 지원합니다.
  - 기존 녹음 파일(휴대폰·워치 음성메모)을 업로드해 처리할 수도 있습니다.
- **플랫폼**: macOS 13.3+ **Apple Silicon 전용**, Windows, iPhone/iPad, Android(`com.clap.alt`).
- **가격**: 로컬에서 처리하므로 전사·요약은 무료 무제한입니다. Pro는 월 $4이며 "가장 성능 좋은 모델"을 제공합니다(클라우드 또는 큰 모델로 추정).
- **시스템 소리**: "Zoom·Google Meet과 매끄럽게 동작"한다고 홍보하므로 마이크와 시스템 소리를 함께 녹음하는 것으로 보입니다. 구현 방식은 **(미검증)**.
- **저장 위치·포맷**: "데이터는 PC에만 저장"된다는 설명만 있고, **기본 경로와 오디오 포맷은 공개 문서에 없습니다.**
- **내보내기**: "언제든 원하는 형식으로 내보내기"라고 홍보합니다. 정확한 형식 목록은 **(미검증)**.
- **연동**: 공개 API·웹훅·노션 연동이 없습니다.
  - 유일한 연동 사례는 서드파티 `BiQnT/alt2obsidian`입니다. 로컬 파일이 아니라 Alt의 **공개 공유 URL**(`https://www.altalt.io/en/note/<uuid>`)을 긁어오는 방식입니다.

### Alt 연동 전략

1. **v1**: 사용자가 Alt에서 **내보낸 파일**(오디오 또는 전사 텍스트)을 지정 폴더에 저장하면 앱이 감지합니다.
   - 전사 텍스트면 STT를 건너뛰고 바로 요약합니다. Alt의 화자 구분 결과도 그대로 씁니다.
2. **설정 마법사의 Alt 탐지**: Alt가 설치된 PC에서 아래 순서로 저장 위치를 찾고, 결과를 사용자에게 보여줍니다.
   ```bash
   # 1) 앱 번들 ID 확인
   osascript -e 'id of app "Alt"'
   # 2) 후보 경로 스캔
   ls -d ~/Library/Application\ Support/*[Aa]lt* ~/Library/Containers/*alt*/Data ~/Documents/Alt 2>/dev/null
   # 3) 방금 녹음한 파일 찾기 (녹음 직후 실행)
   mdfind -onlyin ~ 'kMDItemContentModificationDate >= $time.today' | grep -Ei '\.(m4a|wav|caf|opus|webm)$'
   ```
   - Alt 데이터가 샌드박스 컨테이너에 있으면 macOS 14+에서 "다른 앱의 데이터에 접근하려고 합니다" 권한 창이 뜹니다.
   - 그래서 **내부 저장소 직접 감시는 P2**로 두고, 내보내기 방식을 기본으로 합니다.
3. **향후**: Alt가 API나 내보내기 자동화를 제공하면 전용 커넥터를 추가합니다.

출처: [altalt.io/en](https://www.altalt.io/en) · [features](https://www.altalt.io/en/features) · [pricing](https://www.altalt.io/en/pricing) · [HN](https://news.ycombinator.com/item?id=46069536) · [App Store](https://apps.apple.com/us/app/alt-ai-meeting-notes/id6759488047) · [alt2obsidian](https://github.com/BiQnT/alt2obsidian)

## 2. 맥 음성메모 (+ 아이폰)

- **경로 (macOS Sonoma 14+)**: `~/Library/Group Containers/group.com.apple.VoiceMemos.shared/Recordings/`
  - 이전 macOS에서는 `~/Library/Application Support/com.apple.voicememos/Recordings/`였습니다.
- **포맷**: `.m4a` (AAC, 무손실 설정 시 ALAC).
- **메타데이터**:
  - 파일명은 내부 ID라서 사람이 알아볼 수 없습니다.
  - 제목과 날짜는 SQLite `CloudRecordings.db`의 `ZCLOUDRECORDING` 테이블에 있습니다(`ZPATH`, `ZCUSTOMLABEL`, `ZENCRYPTEDTITLE`, `ZDATE`).
- **전사본 내장**: macOS 15+에서는 애플이 만든 전사본이 m4a 안의 `tsrp` 원자(atom)에 들어갑니다. P2에서 이를 읽어 STT를 생략하는 기능을 검토합니다.
- **권한**: 보호 폴더라서 앱에 **전체 디스크 접근** 권한이 필요하거나, 접근 시 권한 창이 뜹니다.
  - 미서명(ad-hoc 서명) 앱은 빌드마다 서명이 바뀌어 **업데이트할 때마다 이 권한이 초기화될 수 있습니다.**
- ⚠️ **절대 이동·이름 변경 금지**: 음성메모 DB가 깨집니다. 반드시 **복사**만 합니다.
- **아이폰 녹음 가져오기**
  - 아이폰 설정 → iCloud → 음성 메모를 켜면 위 맥 폴더로 자동 동기화됩니다.
  - AirDrop으로 보내면 `~/Downloads`에 `.m4a`로 들어옵니다.
  - 공유 시트 → 파일 앱 → iCloud Drive에 저장하면 `~/Library/Mobile Documents/com~apple~CloudDocs/...`에 들어옵니다.
  - AirDrop이나 공유로 내보낸 파일에는 제목·날짜·전사본이 없습니다.

**v1 방식**
- 기본: 음성메모에서 **공유 → "파일에 저장" → 감시 폴더**. 권한 문제가 없습니다.
- 고급 옵션: 전체 디스크 접근을 허용하면 음성메모 폴더를 직접 감시합니다. 복사 전용이며 제목은 DB에서 읽습니다.

출처: [voicememoexporter](https://www.voicememoexporter.com/where-are-voice-memos-stored-on-mac) · [macOSVoiceMemosExporter](https://github.com/robbyHuelsi/macOSVoiceMemosExporter) · [tsrp atom](https://gist.github.com/pedramamini/f4efacfe7080e07e18f54e13d8243dc1)

## 3. Zoom (무료 Basic 플랜)

- **로컬 녹화는 무료 플랜에서도 됩니다.** 클라우드 녹화는 유료입니다.
- **경로**: `~/Documents/Zoom/<yyyy-mm-dd hh.mm.ss 회의 주제>/`. 윈도우도 `Documents\Zoom` 아래 같은 구조입니다.
- **파일**:
  - 녹음·녹화: `audio_only.m4a`, `video*.mp4`(또는 `zoom_0.mp4`)
  - 기타: `chat.txt`, 선택 시 `.vtt`
- **타이밍**: 폴더는 회의 중에 먼저 생기고, 파일은 **회의가 끝난 뒤 변환이 완료돼야** 생깁니다. 변환에 실패하면 `double_click_to_convert_*.zoom` 파일이 남습니다.
- **메타데이터 추론**: 폴더명에서 날짜·시각·회의 주제를 파싱합니다. 회의 주제의 키워드로 프로젝트를 자동 지정하는 라우팅 규칙을 둡니다.
- **감시 규칙**:
  - `**/audio_only*.m4a`를 우선 처리합니다. 없으면 mp4에서 오디오를 추출합니다.
  - 파일 크기가 N초 동안 변하지 않을 때까지 기다린 뒤 처리합니다.
  - `.zoom` 파일이 보이면 "Zoom에서 변환을 완료해 주세요"라고 안내합니다.

출처: [Zoom 녹화 포맷](https://support.zoom.com/hc/en/article?id=zm_kb&sysparm_article=KB0064394) · [Zoom 폴더 위치](https://www.easytweaks.com/zoom-recordings-folder-location/)

## 4. 그 밖의 소스

| 소스 | 무료 녹음 | 저장 위치 | 비고 |
|---|---|---|---|
| Google Meet | ❌ (Business Standard 이상) | Google Drive (클라우드) | v1 대상 아님 |
| Microsoft Teams 무료 | ❌ (M365 구독 필요) | 클라우드, 30일 보관 | v1 대상 아님 |
| QuickTime Player (파일 → 새로운 오디오 녹음) | ✅ | 저장할 때 선택 | 마이크만 녹음 |
| OBS | ✅ | `~/Movies` / `Videos` | 맥 시스템 소리: macOS 13+ ScreenCaptureKit 또는 BlackHole |
| Audacity | ✅ | 내보낼 때 선택 | 맥 시스템 소리는 BlackHole 필요, 윈도우는 WASAPI 루프백 |
| 윈도우 녹음기 | ✅ | `Documents\Sound recordings\` | OneDrive 리다이렉트 확인 필요 |
| Granola / Otter / tl;dv | 제한적 무료 | 클라우드 | **경쟁 제품**. 감시할 로컬 파일이 없음 |

출처: [Google Meet 녹화 조건](https://fireflies.ai/blog/record-google-meet) · [Teams free](https://support.microsoft.com/en-us/teams/free/meetings/manage-meeting-and-call-recordings-in-microsoft-teams-free) · [QuickTime](https://support.apple.com/guide/quicktime-player/record-audio-qtpf25d6f827/mac) · [OBS 저장 위치](https://cubix.design/resources/where-do-obs-recordings-go-on-windows-mac) · [윈도우 녹음기](https://learn.microsoft.com/en-us/answers/questions/2634459/where-does-the-sound-recorder-app-save-the-recordi)

## 5. 무료 녹음 서비스 추천 (사용자 안내용)

| 상황 | 추천 | 이유 |
|---|---|---|
| 맥에서 온라인 회의 + 대면 회의 (시스템 소리 포함) | **Alt** | 무료 무제한, 시스템 소리, 자체 화자 구분. 전사본을 내보내면 STT 비용 0 |
| Zoom 회의 | **Zoom 로컬 녹화** | 무료, 자동 폴더 구조, 오디오만 따로 저장 |
| 대면 회의, 폰만 있을 때 | **아이폰 음성메모** + iCloud 동기화 | 추가 설치 없음. 맥으로 자동 동기화 |
| 윈도우 PC | **윈도우 녹음기**(마이크) / **Alt 윈도우**(시스템 소리) | 기본 앱 또는 무료 |
| 앱 자체 녹음 | **v1.5 예정** | 아래 6절 |

## 6. 앱 자체 녹음 (v1.5) 실현 가능성

> 요청 문구의 "TTS"는 문맥상 **STT(음성 → 텍스트)** 로 해석했습니다. STT는 [stt-options.md](stt-options.md)를 참고하세요.
> 여기서는 **녹음(오디오 캡처)** 만 다룹니다.

| 대상 | 방법 | 필요한 권한 | 비고 |
|---|---|---|---|
| 마이크 (맥·윈도우) | `navigator.mediaDevices.getUserMedia` | 마이크 (`NSMicrophoneUsageDescription`) | 간단 |
| 시스템 소리 — macOS 14.2+ | **Core Audio Process Tap** (`AudioHardwareCreateProcessTap`). Swift 헬퍼(예: AudioTee)를 자식 프로세스로 실행해 PCM 스트림을 받음 | **"시스템 오디오 녹음만"** 권한(`NSAudioCaptureUsageDescription`). 화면 녹화 권한 불필요 | 특정 앱(Zoom 등)만 캡처할 수도 있음 |
| 시스템 소리 — macOS 13.x | Electron 39+ `setDisplayMediaRequestHandler` + `audio: 'loopback'` (ScreenCaptureKit) | 화면 녹화 | 알려진 버그: `useSystemPicker: false`면 빈 트랙이 나옴 ([electron#52738](https://github.com/electron/electron/issues/52738)) |
| 시스템 소리 — 윈도우 | `desktopCapturer` / `getDisplayMedia` + `audio: 'loopback'` (WASAPI) | 추가 권한 없음 | 안정적 |
| 최후 대안 (맥) | BlackHole 가상 오디오 장치 + 다중 출력 장치 | – | 사용자가 직접 설정해야 해서 UX가 나쁨 |

**설계 원칙**
- **마이크와 시스템 소리를 별도 트랙으로 저장**합니다. 마이크 트랙은 "나", 시스템 트랙은 "상대"로 구분할 수 있어 로컬 STT에 화자 구분이 없어도 도움이 됩니다. ffmpeg로 합친 버전도 함께 저장합니다.
- 서명과 공증, 하드닝된 런타임을 갖춰야 권한 창이 안정적으로 동작합니다. → v1.5는 **서명 전환 이후**가 이상적입니다.
- 참고 구현: [insidegui/AudioCap](https://github.com/insidegui/AudioCap), [makeusabrew/audiotee](https://github.com/makeusabrew/audiotee), [electron-audio-loopback](https://github.com/alectrocute/electron-audio-loopback), [Recall.ai: Core Audio taps](https://www.recall.ai/blog/core-audio-taps), [asleep-ai/listener-ai](https://github.com/asleep-ai/listener-ai)(Electron 회의 녹음 → 전사 → 노션, 구조 참고)

## 7. 원본 녹음 보관 (구글 드라이브)

- **권장: Google Drive 데스크톱 앱의 동기화 폴더에 복사**합니다. API 연동, OAuth, 구글 클라우드 프로젝트가 모두 필요 없고 무료 15GB를 씁니다.
  - 맥: `~/Library/CloudStorage/GoogleDrive-<email>/My Drive` (한국어 환경에선 `내 드라이브`일 수 있음 **(미검증)**)
  - 윈도우: 가상 드라이브 문자. 보통 `G:\My Drive`지만 바뀔 수 있습니다 **(미검증)**.
- 탐지 방법:
  - 맥: `~/Library/CloudStorage/GoogleDrive-*`를 스캔합니다.
  - 윈도우: 드라이브 문자를 돌며 `My Drive` 또는 `내 드라이브` 폴더를 찾습니다.
  - 찾지 못하면 폴더 선택 창을 띄웁니다.
- 설치되어 있지 않으면 [Google Drive for desktop](https://support.google.com/drive/answer/10838124) 설치 가이드를 보여줍니다.
- Google Drive API 직접 연동은 OAuth 동의 화면과 민감 범위 검수 부담이 커서 **v2 이후**로 미룹니다.
