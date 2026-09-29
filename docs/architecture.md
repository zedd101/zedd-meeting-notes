# 기술 아키텍처

> 관련: [PRD](PRD.md) · [UX 흐름](ux-flows.md) · [데스크탑 스택 리서치](research/desktop-stack.md)

## 1. 전체 구조

```
┌──────────────────────────── Electron 앱 ─────────────────────────────┐
│                                                                       │
│  renderer (React + shadcn/ui)          preload                        │
│  ┌──────────────────────────┐   ┌──────────────────┐                  │
│  │ 화면·폼·미리보기          │◀─▶│ contextBridge      │                  │
│  │ TanStack Router / Query  │   │ 타입 있는 IPC API │                  │
│  └──────────────────────────┘   └────────┬─────────┘                  │
│                                           │ ipcMain.handle / events     │
│  main (Node)                              ▼                             │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ sources ─▶ pipeline (단계 상태 머신 + 큐)                         │  │
│  │              ├─ audio (ffmpeg-static)                            │  │
│  │              ├─ stt ──────────▶ utilityProcess (whisper.cpp)     │  │
│  │              │                  └─ 클라우드: Soniox / Groq …     │  │
│  │              ├─ llm (Vercel AI SDK: anthropic/openai/google/     │  │
│  │              │        openrouter/openai-compatible)               │  │
│  │              ├─ render (MeetingMinutes → DocAST → notion | md)    │  │
│  │              ├─ notion (@notionhq/client, 레이트리밋 큐)          │  │
│  │              └─ output (마크다운 저장, 원본 보관)                  │  │
│  │ store (설정·회의 JSON)   secrets (safeStorage)   updater(알림)    │  │
│  └────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
        │                 │                   │                  │
   사용자 폴더        STT 클라우드(선택)     LLM 제공자         Notion API
   (.md, 원본)
```

- **renderer는 네트워크와 파일시스템에 직접 접근하지 않습니다.** 모든 작업은 IPC로 main에 요청합니다.
  - 설정: `contextIsolation: true`, `sandbox: true`, `nodeIntegration: false`
- **API 키는 main에서만 복호화**하고 renderer로 보내지 않습니다.

## 2. 디렉토리 구조 (M1에서 생성)

```
zedd-meeting-notes/
├─ docs/                         # 기획·리서치 (현재)
├─ resources/
│  ├─ bin/<platform>-<arch>/     # whisper.cpp CLI (스파이크 결과에 따라)
│  └─ samples/ko-sample.m4a      # 설정 마법사 테스트용 샘플
├─ src/
│  ├─ main/
│  │  ├─ index.ts                # 앱 수명주기, 창, 트레이
│  │  ├─ ipc/                    # 채널별 핸들러 (zod로 입력 검증)
│  │  ├─ sources/                # watcher.ts(chokidar), presets.ts, detect.ts(설정 마법사 탐지)
│  │  ├─ pipeline/               # queue.ts, stages.ts, state.ts (단계 상태 머신)
│  │  ├─ audio/                  # ffmpeg 변환, 길이 측정
│  │  ├─ stt/                    # provider.ts(인터페이스), local-whisper.ts, soniox.ts, groq.ts, worker.ts(utilityProcess)
│  │  ├─ llm/                    # providers.ts(프리셋·커스텀·기본값), summarize.ts, template-analyze.ts, prompts/
│  │  ├─ templates/              # 기본 템플릿, 템플릿 파서(Template Spec 마크다운 ↔ TemplateSpec)
│  │  ├─ render/                 # doc-ast.ts, to-notion.ts, to-markdown.ts
│  │  ├─ notion/                 # client.ts(레이트리밋), setup.ts(DB 생성), publish.ts(upsert), schema-check.ts
│  │  ├─ output/                 # markdown-writer.ts, archive.ts(Google Drive 탐지·복사)
│  │  ├─ store/                  # settings.ts, meetings.ts(userData JSON), search.ts(MiniSearch)
│  │  ├─ secrets/                # safeStorage 래퍼
│  │  └─ updater/                # GitHub Releases 새 버전 확인
│  ├─ preload/index.ts
│  ├─ shared/                    # 프로세스 공통: schema/*.ts(zod), ipc-contract.ts, i18n 키
│  └─ renderer/
│     ├─ routes/                 # onboarding, inbox, meeting.$id, history, templates, settings/*
│     ├─ components/             # shadcn/ui + 도메인 컴포넌트
│     └─ locales/ko.json
├─ tests/                        # vitest (렌더러 스냅샷, 파서, 파이프라인)
├─ electron-builder.yml
└─ .github/workflows/release.yml
```

## 3. 파이프라인 상태 머신

```
detected ─▶ queued ─▶ converting ─▶ transcribing ─▶ summarizing ─▶ review ─▶ publishing ─▶ done
                          │              │                │                     │
                          └──────────────┴────── failed(stage, errorCode) ◀─────┘
                                                     │
                                                     └─ retry → 실패한 단계부터 재시작
```

- 각 단계가 끝날 때마다 산출물을 `userData/meetings/<id>/`에 저장합니다.
  - `audio.wav`(임시), `transcript.json`, `minutes.json`, `meeting.json`
  - 그래서 재시도할 때 앞 단계를 다시 하지 않습니다.
- 입력이 전사 텍스트면 `converting`과 `transcribing`을 건너뜁니다.
  - `.srt`/`.vtt`는 타임스탬프 구간으로, `.txt`는 문단 단위 구간으로 파싱합니다.
- 큐 동시성: 전사 1, LLM 2, 노션 1.
- 앱을 시작하면 미완료 회의를 복구합니다(`transcribing` 중이던 것은 `queued`로 되돌림).

## 4. 데이터 모델

`src/shared/schema`의 zod 정의입니다(개요).

```ts
// 회의
Meeting = {
  id: string;                         // ULID
  source: { kind: 'zoom'|'voice-memos'|'alt'|'win-recorder'|'folder'|'manual'; path: string; hash: string };
  meta: { title: string; startedAt: string; endedAt?: string; location?: string;
          projectId?: string; attendees: PersonRef[]; tags: string[] };
  stage: Stage; error?: { stage: Stage; code: ErrorCode; message: string };
  stt?: { provider: string; model?: string; durationSec: number; diarization: boolean };
  llm?: { provider: string; model: string; inputTokens: number; outputTokens: number };
  costUsd: { stt: number; llm: number };
  templateRef: `${string}@${number}`;
  outputs: { markdownPath?: string; notionPageId?: string; notionUrl?: string;
             notionLastEditedAtPublish?: string; archivedAudioPath?: string };
  createdAt: string; updatedAt: string;
};

// 전사본
Transcript = { language: string; segments: { id: string; start: number; end: number; speaker?: string; text: string }[];
               speakerMap: Record<string, PersonRef> };

// 회의록 (LLM 출력 = 단일 원본)
MeetingMinutes = {
  sections: Section[];                // 템플릿 순서 그대로
};
Section =
  | { type: 'meta'; title: string; objective?: string }
  | { type: 'summary'; title: string; conclusion: string; points: string[] }
  | { type: 'decisions'; title: string; items: { id: string; text: string; decider?: string; rationale?: string;
                                                changedFrom?: string; evidence: string[] }[] }
  | { type: 'actions'; title: string; items: { actionItemId: string; task: string; owner?: PersonRef | null;
                                              due?: string | null; status: 'todo'|'doing'|'done'|'hold';
                                              decisionId?: string; evidence: string[] }[] }
  | { type: 'risks'; title: string; bottleneck?: { text: string; owner?: PersonRef };
      items: { kind: 'issue'|'risk'|'dependency'|'assumption'; text: string; impact: 'high'|'mid'|'low'; mitigation?: string; owner?: PersonRef }[] }
  | { type: 'discussion'; title: string; agendas: { title: string; context?: string; points: { speaker?: string; text: string }[]; outcome?: string }[] }
  | { type: 'open_questions'; title: string; items: { text: string; owner?: PersonRef; due?: string }[] }
  | { type: 'carryover'; title: string; items: { actionItemId: string; task: string; owner?: PersonRef;
                                                prevStatus: string; verdict: 'done'|'doing'|'not_mentioned'; evidence: string[] }[] }
  | { type: 'next_meeting'; title: string; when?: string; agenda: string[]; preRead?: string }
  | { type: 'appendix'; title: string; links: { label: string; href: string }[] }
  | { type: 'freeform'; title: string; markdown: string };
// owner·due가 null이면 "미정"으로 렌더링

// 템플릿
TemplateSpec = { id: string; version: number; name: string;
                 sections: { title: string; type: SectionType; instruction: string; required: boolean }[] };

// 설정
ProviderConfig = { id: string; kind: 'anthropic'|'openai'|'google'|'openrouter'|'ollama'|'lmstudio'|'custom';
                   label: string; baseUrl?: string; model: string; enabled: boolean; isDefault: boolean; hasKey: boolean };
Project = { id: string; name: string; notionSelect: string; outputDir?: string; templateRef?: string;
            defaultAttendees: PersonRef[]; glossary: string[]; routing: { sourceKind?: string; titleIncludes?: string[] }[] };
Person  = { id: string; name: string; aliases: string[]; title?: string; notionUserId?: string };
```

- **LLM 출력 스키마는 템플릿에서 동적으로 만듭니다.** 템플릿 섹션 목록에 맞춰 `Section` 유니언의 해당 스키마를 순서대로 조합합니다.
  - OpenAI strict 모드처럼 선택 필드를 허용하지 않는 제공자는 `nullable`로 변환합니다.
  - 구조화 출력이 약한 제공자·모델(작은 로컬 모델 등)은 JSON 모드와 zod 검증, 1회 복구 재시도로 처리합니다.
- **`actionItemId`는 발행 전까지 앱이 부여합니다.** 이월 항목은 노션 액션 DB의 `앱 ID`를 그대로 씁니다.

## 5. LLM 계층

- **라이브러리**: Vercel AI SDK(`ai`) + `@ai-sdk/anthropic`, `@ai-sdk/openai`, `@ai-sdk/google`, `@openrouter/ai-sdk-provider`, `@ai-sdk/openai-compatible`
  - Ollama, LM Studio, 커스텀 제공자는 모두 OpenAI 호환 방식으로 연결합니다.
- **기본 제공자 결정**:
  1. 키가 있고 켜져 있는 제공자를 모읍니다. 로컬 제공자는 키 대신 주소를 확인합니다.
  2. 1개면 그 제공자를 씁니다.
  3. 여러 개면 `isDefault`가 켜진 제공자를 씁니다.
  4. 하나도 없으면 **키 없는 모드**(F5)로 안내합니다.
- **로컬 전용 모드**가 켜지면 로컬 제공자(ollama/lmstudio/커스텀 중 localhost)만 허용합니다. 클라우드 호출은 네트워크 계층에서 거부합니다.
- **프롬프트 구성** (`llm/prompts/summarize.ts`)
  - system: 역할, 작성 규칙([기본 템플릿 작성 규칙](templates/default-minutes.md#작성-규칙-llm-지시문에-포함)), 미정 규칙, 날짜 변환 규칙
  - user:
    - 회의 메타(회의 일자 포함)
    - 템플릿 섹션과 지시문
    - 용어집과 사람 디렉토리(별칭)
    - 이전 미완료 액션 목록
    - 전사본(`[S012 12:34 화자2] 텍스트` 형식)
  - 출력: 동적 zod 스키마
- **비용 추정**: 전사본 토큰 수(대략 문자 수 기반) × 제공자별 단가표(설정에서 수정 가능)
- **템플릿 분석** (`template-analyze.ts`): 업로드·붙여넣은 양식을 입력하면 `TemplateSpec`을 출력합니다. [외부 LLM 프롬프트](templates/external-llm-prompt.md)와 같은 규칙을 씁니다.

## 6. 렌더링: DocAST

```ts
type DocNode =
  | { t: 'heading'; level: 1|2|3; text: Inline[]; toggle?: boolean; children?: DocNode[] }
  | { t: 'paragraph'; text: Inline[] }
  | { t: 'callout'; tone: 'info'|'success'|'danger'|'warning'|'note'; icon: string; children: DocNode[] }
  | { t: 'list'; ordered: boolean; items: DocNode[][] }
  | { t: 'todo'; checked: boolean; text: Inline[] }
  | { t: 'table'; header: Inline[][]; rows: Inline[][][] }
  | { t: 'code'; lang: string; code: string }
  | { t: 'divider' } | { t: 'toc' }
  | { t: 'link_to_db'; db: 'actions' }
  | { t: 'child_page'; title: string; children: DocNode[] };
type Inline = { text: string; bold?: boolean; italic?: boolean; strike?: boolean; code?: boolean;
                color?: 'orange'|'red'|'gray'; href?: string;
                mention?: { user: string } | { date: string } };
```

- `toDocAST(minutes, template, meeting)`: 섹션 유형별 빌더가 [노션 레이아웃 매핑표](templates/notion-layout.md#3-섹션--블록-매핑표)대로 노드를 만듭니다.
- `toNotion(doc)`: 노션 블록 JSON으로 바꿉니다. 100블록과 2단계 중첩으로 나누고, 2,000자 분할도 여기서 합니다.
- `toMarkdown(doc, frontmatter)`: GFM, `> [!NOTE|TIP|CAUTION|WARNING]`, `- [ ]`, YAML frontmatter를 씁니다.
  - 노션 전용 노드는 다음처럼 대체합니다.
    - `toc`: 생략
    - `link_to_db`: 노션 URL 링크
    - `child_page`: 별도 파일(`_transcript.md`) 링크
- **테스트**: 샘플 `MeetingMinutes` 3종(짧은·긴·미정 많은 회의)으로 두 출력의 **골든 스냅샷**을 만들어 비교합니다.

## 7. 오류 처리

| 코드 | 상황 | 사용자 메시지 | 복구 액션 |
|---|---|---|---|
| `SRC_PERMISSION` | 폴더 권한 거부 (macOS TCC) | "이 폴더에 접근할 권한이 없어요" | [시스템 설정 열기] / 공유 방식 안내 |
| `SRC_INCOMPLETE` | 파일이 아직 쓰이는 중, iCloud 플레이스홀더, Zoom `.zoom` | "녹음 파일이 아직 준비되지 않았어요" | 자동 재확인 / Zoom 변환 안내 |
| `AUDIO_CODEC` | ffmpeg가 해석할 수 없는 파일 | "지원하지 않는 파일 형식이에요" | 지원 형식 안내 |
| `STT_MODEL_MISSING` | 로컬 모델 없음 | "음성인식 모델을 내려받아야 해요 (약 550MB)" | [다운로드] |
| `STT_TIMEOUT` / `STT_API` | 클라우드 오류, 한도 초과 | 제공자 오류 요약 | [재시도] / [로컬로 전환] |
| `LLM_AUTH` | 키 오류 | "API 키를 확인해 주세요" | [설정 열기] |
| `LLM_SCHEMA` | 복구 재시도 후에도 검증 실패 | "일부 섹션을 자동으로 정리하지 못했어요" | 자유 섹션으로 표시하고 검토 진행 / [다른 모델로 재생성] |
| `LLM_CONTEXT` | 컨텍스트 초과 | – (자동으로 나눠 요약) | – |
| `NOTION_AUTH` | 401 | "노션 연결이 끊어졌어요" | [다시 연결] |
| `NOTION_NOT_SHARED` | 404, 권한 없음 | "회의록 페이지가 통합에 공유되지 않았어요" | 공유 방법 안내 |
| `NOTION_SCHEMA_DRIFT` | 속성 이름 변경·삭제 | "노션 DB 구조가 바뀌었어요" | [자동 복구] / [매핑 다시 하기] |
| `NOTION_CONFLICT` | 마지막 발행 뒤 노션에서 수정됨 | "노션에서 수정된 내용이 있어요" | [덮어쓰기] / [노션 열기] / [취소] |
| `NOTION_RATE` | 429 | – (자동 대기 후 재시도) | – |
| `OUT_DISK` | 디스크 부족 | "저장 공간이 부족해요" | 임시 파일 정리 |
| `OUT_PATH` | 저장 폴더 없음, 권한 없음 | "저장 폴더를 찾을 수 없어요" | [폴더 선택] |

## 8. 보안 설계

- Electron 보안 체크리스트를 따릅니다: contextIsolation, sandbox, CSP, `webSecurity` 유지, 원격 콘텐츠 로드 금지, `shell.openExternal` 허용 목록.
- IPC 입력은 모두 zod로 검증하고, 파일 경로는 허용된 루트 안인지 확인합니다.
- 키는 `safeStorage.encryptString`으로 암호화해 base64로 저장합니다. 사용할 때만 main에서 복호화합니다.
- 로그(`userData/logs`)에는 회의 ID, 단계, 오류 코드, 소요 시간만 남깁니다.

## 9. 빌드 · 릴리스

- **electron-builder**
  - macOS: `dmg` + `zip`, `identity: '-'`(ad-hoc)
  - Windows: `nsis`
  - `asarUnpack`: `**/*.node`, `ffmpeg`, `whisper` 바이너리
- **Info.plist**
  - `NSMicrophoneUsageDescription`(v1.5)
  - `NSAudioCaptureUsageDescription`(v1.5)
  - `NSDocumentsFolderUsageDescription`
  - `NSDownloadsFolderUsageDescription`
- **CI**: [데스크탑 스택 리서치 2절](research/desktop-stack.md#2-github-releases-배포-파이프라인)의 매트릭스를 씁니다. 태그 `v*` → Releases 초안.
- **새 버전 알림**: 시작할 때와 24시간마다 GitHub Releases `latest`를 확인하고, 새 버전이 있으면 배너와 다운로드 링크를 띄웁니다. 윈도우는 electron-updater를 씁니다.

## 10. 테스트 전략

| 수준 | 대상 | 도구 |
|---|---|---|
| 단위 | 템플릿 파서, srt/vtt 파서, 날짜 변환, 비용 계산, 기본 제공자 결정 규칙 | vitest |
| 스냅샷 | `MeetingMinutes → DocAST → notion JSON / markdown` | vitest 골든 파일 |
| 통합 | 파이프라인(가짜 STT·LLM·노션 어댑터) 단계 전이, 재시도, 앱 재시작 복구 | vitest |
| 계약 | 노션 발행을 테스트 워크스페이스에 실제로 수행 (수동 실행) | 스크립트 |
| 품질 | 골든셋 10건의 액션 재현율·정밀도, CER | 평가 스크립트 (M2~M3) |
| E2E | 설정 마법사 → 샘플 처리 → 발행 | Playwright(Electron) |
