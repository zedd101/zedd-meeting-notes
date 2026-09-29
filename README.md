# Zedd Meeting Notes (가칭)

회의 녹음 파일을 넣으면 **회사 양식에 맞춘 회의록**을 만듭니다. 결과는 **노션에 읽기 좋게 발행**되고, **같은 내용이 로컬 마크다운으로 저장**됩니다. macOS와 Windows용 데스크탑 앱입니다.

> 📌 **현재 단계: M0 기획** — 코드는 아직 없고, 기획 문서만 있습니다.

## 어떻게 동작하나요

```
녹음 (Alt · Zoom · 음성메모 · 윈도우 녹음기 · 아무 폴더)
  → 음성인식 (기본: 이 PC에서 whisper / 선택: 클라우드)
  → 요약 (내 LLM API 키: Claude · GPT · Gemini · OpenRouter · 로컬 LLM)
  → 검토
  → 노션 회의록 DB + 액션아이템 DB  &  ~/Documents/MeetingNotes/*.md
```

- 별도 백엔드 서버가 없습니다. 모든 처리는 내 PC에서 하고, 키는 OS 키체인에 저장합니다.
- 회사 양식은 파일 업로드, 텍스트 붙여넣기, 직접 작성 중에 골라서 등록합니다. 양식이 없으면 [기본 템플릿](docs/templates/default-minutes.md)을 씁니다.
- 결정사항·할 일·담당자·병목이 첫 화면에 보이는 [노션 레이아웃](docs/templates/notion-layout.md)으로 발행합니다.

## 문서

| 문서 | 내용 |
|---|---|
| [PRD](docs/PRD.md) | 제품 요구사항: 목표, 기능, 지표, 로드맵, 리스크 |
| [아키텍처](docs/architecture.md) | 기술 구조, 데이터 모델, 파이프라인, 오류 처리 |
| [UX 흐름](docs/ux-flows.md) | 화면 구성, 설정 마법사, 사용자 시나리오 |
| [기본 회의록 템플릿](docs/templates/default-minutes.md) | 컨설팅 펌과 국내 대기업 관행을 합친 디폴트 양식 |
| [노션 레이아웃](docs/templates/notion-layout.md) | 섹션 → 노션 블록·마크다운 매핑 |
| [외부 LLM 프롬프트](docs/templates/external-llm-prompt.md) | 회사 양식을 앱 템플릿으로 바꾸는 프롬프트 |
| 리서치 | [녹음 소스](docs/research/recording-sources.md) · [음성인식](docs/research/stt-options.md) · [노션 API](docs/research/notion-api.md) · [데스크탑 스택](docs/research/desktop-stack.md) |

## 기술 스택 (예정)

Electron · electron-vite · React · TypeScript · Tailwind v4 · shadcn/ui · whisper.cpp · Vercel AI SDK · Notion API · electron-builder → GitHub Releases

## 설치 (예정)

- 프로토타입 단계에서는 코드 서명을 하지 않습니다.
- **macOS**: 처음 열 때 차단되면 **시스템 설정 → 개인정보 보호 및 보안 → "그래도 열기"** 를 누릅니다. 앱당 한 번만 하면 됩니다.
- **Windows**: "Windows의 PC 보호" 창이 뜨면 **추가 정보 → 실행**을 누릅니다.
