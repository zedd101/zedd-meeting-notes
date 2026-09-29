# 리서치: 노션 API와 회의록 베스트 프랙티스

> 조사일: 2026-09-29.
> developers.notion.com은 조사 환경의 네트워크 제한으로 직접 열지 못했습니다.
> 내용은 두 곳에서 확인했습니다: 공식 JS SDK(`makenotion/notion-sdk-js`, 노션 API 스키마로 생성된 코드) 소스, 그리고 문서 검색 결과. 미검증 항목은 표시했습니다.

## 1. API 버전

- 최신: **`2026-03-11`**. SDK 기본값은 여전히 `2025-09-03`입니다.
- `2026-03-11`의 주요 변경:
  - 블록 추가 시 `after` 대신 `position` 사용 (`after_block` / `start` / `end`)
  - `archived` 대신 `in_trash` 사용
  - `transcription` 블록 대신 `meeting_notes` 블록 사용
- **데이터 소스 모델** (`2025-09-03`~): 데이터베이스 하나가 여러 데이터 소스를 가질 수 있습니다. 그래서 조회와 스키마 변경은 `data_source_id`로 합니다.
- **앱 방침**: `Notion-Version`을 고정하고, 버전 업그레이드는 명시적으로 합니다.

## 2. 인증

| | 내부 통합 (Internal) | 공개 통합 (Public OAuth) |
|---|---|---|
| 방식 | 워크스페이스 하나, 고정 토큰(`ntn_…`). 사용자가 특정 페이지를 통합에 **공유** | OAuth 2.0 코드 플로우 |
| 리디렉션 URI | 해당 없음 | 정확히 일치해야 함. localhost는 개발용으로만 흔히 사용. 커스텀 스킴(`myapp://`) 지원 근거 없음 |
| 클라이언트 시크릿 | 해당 없음 | 토큰 교환에 필수이고 **PKCE 미지원** → 데스크탑 앱에 안전하게 둘 수 없어 서버가 필요 |

- **결정: 프로토타입은 내부 통합 토큰을 씁니다.**
  1. 사용자가 노션에서 통합을 만들고 토큰을 복사해 앱에 붙여넣습니다.
  2. 앱은 토큰을 OS 키체인에 저장합니다.
  3. 사용자가 "회의록" 부모 페이지를 통합에 공유합니다.
- 다중 사용자 배포 단계에서는 공개 OAuth(시크릿 보관용 작은 서버 필요)나 노션 호스팅 MCP(OAuth 2.1 + PKCE)를 검토합니다.

## 3. 생성 가능한 블록

- **제목**: `heading_1`~`heading_3`(+`heading_4`), `is_toggleable`로 토글 헤딩
- **강조**: `callout`(아이콘·색상), `quote`, `divider`
- **목록과 접기**: `toggle`, `bulleted_list_item`, `numbered_list_item`, `to_do`
- **표**: `table` / `table_row` (헤더 행·헤더 열 옵션)
- **레이아웃**: `column_list` / `column`, `table_of_contents`, `breadcrumb`, `tab`(신규)
- **코드와 수식**: `code`(언어 목록에 `mermaid` 포함), `equation`
- **링크와 미디어**: `bookmark`, `embed`, `link_to_page`, `synced_block`, `image`, `file`, `audio`, `video`, `pdf`
- **rich text**: 굵게·기울임·취소선·밑줄·코드, 글자색과 `*_background` 배경색, 멘션(사용자·날짜·페이지·데이터베이스)

**mermaid**: 노션 앱에서는 mermaid 코드 블록이 다이어그램으로 보입니다.
- API로 코드 블록은 만들 수 있지만, 보기 모드(코드/미리보기)는 설정할 방법을 찾지 못했습니다.
- 사용자가 미리보기로 바꾸기 전까지는 코드로 보일 수 있습니다 → 구조도는 **보조 수단**으로만 씁니다.

## 4. 제한

| 항목 | 값 |
|---|---|
| 요청 빈도 | Free·Plus: 통합당 **분당 180회**(약 초당 3회), Business·Enterprise: 분당 600회. 워크스페이스 단위 한도도 있음. `Retry-After` 최대 60초 |
| 배열 크기 | 모든 배열(children, rich_text) 최대 **100개** |
| 문자열 | rich text 한 조각·URL **2,000자** |
| 중첩 | 한 요청에서 **2단계**까지. 더 깊은 자식은 후속 append |
| 페이로드 | 알려진 값은 1,000블록 / 500KB (현재 수치 **미검증**) |
| 파일 업로드 | 무료 워크스페이스는 **파일당 5 MiB**, 유료는 5 GiB (20 MiB 초과 시 멀티파트) |

## 5. 데이터베이스와 뷰

- **회의록 DB**:
  - 속성: `title`, `date`, `people`(노션 멤버), `multi_select`(외부 참석자·태그), `select`(프로젝트), `status`, `relation`
  - 모두 API로 만들 수 있습니다.
- **액션아이템 DB**: 담당자(`people`), 기한(`date`), 상태(`status`), 회의록 DB와의 `relation`(양방향 `dual_property`), `rollup`.
- **Views API**:
  - `views.create/retrieve/update/delete/list`로 표·보드·목록·캘린더·타임라인·갤러리·폼·차트·지도·대시보드 뷰를 만들 수 있습니다.
  - 필터·정렬·그룹도 설정할 수 있습니다. `2025-09-03` 이상 버전이 필요합니다.
  - 출시 시점은 2026-04경으로 보이며 **미검증**입니다.
  - → P1으로 두고, 실패하면 수동 뷰 추가 가이드를 보여줍니다.

## 6. 마크다운 ↔ 노션

- **노션 자체 마크다운 입력**(2026-02경 출시):
  - `POST /v1/pages`에 `markdown` 필드로 페이지를 만들 수 있습니다.
  - 페이지 마크다운 편집(`insert_content`, `replace_content` 등)과 읽기(`GET /v1/pages/{id}/markdown`)도 됩니다.
  - 콜아웃·컬럼·토글 헤딩은 노션 확장 문법(XML 비슷한 태그)으로 씁니다.
  - 읽기 엔드포인트가 공개 통합에서만 된다는 제3자 주장이 있으며 **미검증**입니다.
- **라이브러리**:
  - `@tryfabric/martian`(마크다운 → 노션 블록): 마지막 npm 배포가 2022년입니다.
  - `notion-to-md`(노션 → 마크다운): 3.x 안정판입니다.
- **결정**:
  - 앱은 **블록 JSON을 직접 생성**합니다. 색상·멘션·표 셀을 정밀하게 제어하고, 재발행 시 블록 단위로 교체하기 위해서입니다.
  - 마크다운 입력은 참고용으로만 씁니다. 사용자가 `freeform` 섹션에 쓴 마크다운은 DocAST로 파싱해 블록으로 바꿉니다.

## 7. 경쟁 기능: 노션 AI 회의 노트

- 2025-05 출시, `/meet` 명령으로 씁니다.
  - 오디오만 녹음해서 전사본·요약·액션아이템을 만듭니다.
  - 한국어 포함 약 16개 언어를 지원합니다.
- 제대로 쓰려면 **Business(인당 월 $20) 이상** 플랜이 필요합니다.
- API에도 `POST /v1/blocks/meeting_notes`가 생겨, 오디오 파일로 회의 노트를 만들 수 있습니다.
- **이 앱의 차별점**:
  - 무료 플랜에서도 동작합니다.
  - **회사 양식** 반영
  - 로컬 STT와 로컬 마크다운 보관
  - 액션아이템 DB와 **회의 간 이월 추적**
  - 다양한 녹음 소스

## 8. 회의록 베스트 프랙티스 (기본 템플릿 근거)

1. **컨설팅 펌**
   - **피라미드 원칙 / 결론 먼저**(Minto, McKinsey): 결론을 먼저 쓰고 근거를 아래에 둡니다.
   - **Bain RAPID**: 추천·의견·합의·결정·실행 역할 가운데 **누가 결정했는지**를 기록합니다.
   - **RAID 로그**: 리스크·가정·이슈·의존성
   - 마무리는 항상 "다음 단계: 담당자, 기한"입니다.
2. **국내 대기업**
   - 삼성(2016 "스타트업 삼성" 조직문화 혁신): 최소 인원, 1시간 이내, 모두 발언, **반드시 결론 도출**
   - LG전자 **"1-1-1"**: 자료는 1일 전 공유, 회의는 1시간 이내, 결과는 1일 내 공유
   - 일반 회의록 항목: 회의명, 일시, 장소, 참석자, 작성자, 안건, 논의내용, 결정사항(기존 결정 변경은 따로 표시), Action Item(담당자·기한·내용), 특이사항, 차기 회의
   - 차기 회의에서 **이전 Action Item을 점검**합니다.
3. **Amazon / Google / Atlassian / Notion**
   - Amazon: 서술형 메모를 먼저 읽고 토론 → 회의록에 사전 자료 링크
   - Google Docs "@회의록": 캘린더에서 제목·날짜·참석자를 가져오고 체크박스 액션
   - Confluence: 목표·참석자·논의 주제·`/decision`·`/action item`
   - Notion AI: 요약·액션아이템·녹취록 탭

→ 이 원칙들을 합친 결과가 [기본 템플릿](../templates/default-minutes.md)이고, 노션 표현은 [노션 레이아웃 가이드](../templates/notion-layout.md)에 있습니다.

## 출처

- SDK: https://github.com/makenotion/notion-sdk-js · 변경 이력: https://developers.notion.com/page/changelog
- 인증: https://developers.notion.com/guides/get-started/authorization · https://developers.notion.com/guides/get-started/internal-connections · https://developers.notion.com/guides/mcp/build-mcp-client
- 제한: https://developers.notion.com/reference/request-limits · 파일: https://developers.notion.com/guides/data-apis/working-with-files-and-media
- 뷰: https://developers.notion.com/guides/data-apis/working-with-views
- 마크다운: https://developers.notion.com/guides/data-apis/working-with-markdown-content · https://developers.notion.com/guides/data-apis/enhanced-markdown
- mermaid: https://lukemerrett.com/using-mermaid-flowchart-syntax-in-notion/
- 노션 AI 회의 노트: https://www.notion.com/help/ai-meeting-notes
- 라이브러리: https://github.com/tryfabric/martian · https://www.npmjs.com/package/notion-to-md
- 회의록: https://slideworks.io/resources/the-pyramid-principle-mckinsey-toolbox-with-examples · https://www.bain.com/insights/rapid-decision-making/ · https://asana.com/resources/raid-log · https://news.samsung.com/kr/4576 · https://brunch.co.kr/@goodgdg/8 · https://www.atlassian.com/software/confluence/templates/meeting-notes · https://blog.google/products-and-platforms/products/workspace/google-docs-meeting-notes/
