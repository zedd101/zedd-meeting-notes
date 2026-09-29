# 리서치: 음성인식(STT) 선택지

> 조사일: 2026-09-29. 가격은 대략적인 공개 가격이며 수시로 바뀝니다.
> 한국어 정확도 수치는 서로 다른 벤치마크에서 가져온 것이라 **직접 비교에 한계**가 있습니다.
> M3에서 실제 회의 골든셋으로 다시 측정합니다.

## 결론

| 모드 | 제공자 | 사용하는 경우 |
|---|---|---|
| **기본: 로컬** | whisper.cpp `large-v3-turbo` (양자화 모델 기본) | 회의 내용을 외부로 보내면 안 될 때, 비용 0 |
| **권장 클라우드** | **Soniox** | 한국어 정확도와 화자 구분이 필요할 때 (~$0.10/시간) |
| 무료 클라우드 | Groq `whisper-large-v3-turbo` | 하루 8시간 무료. 화자 구분 없음 |
| 이후 추가 | ElevenLabs Scribe v2, AssemblyAI, OpenAI, Deepgram | 공통 `SttProvider` 인터페이스로 확장 |
| 생략 | – | Alt 등에서 **전사 텍스트를 받은 경우** |

> ⚠️ **로컬 Whisper의 한계**
> - 한국어 CER(글자 오류율)이 약 10–11%입니다.
> - **화자 구분이 사실상 없습니다**(whisper.cpp의 tinydiarize는 영어용이고 약합니다).
> - 그래서 앱은 로컬 모드일 때 설정 화면에 "정확도·화자 구분이 필요하면 Soniox 권장" 안내를 보여줍니다.
> - 화자 구분이 없으면 LLM은 발언 속 **명시적 호명**("철수 씨가 말한 것처럼")으로만 발언자를 추정합니다.

## 1. 로컬 STT

### whisper.cpp (MIT)

- **가속**: Metal과 Core ML(맥), Vulkan과 CUDA(윈도우). 음성 구간 검출(Silero VAD) 내장.
- **입력**: 16kHz 모노 16비트 WAV → `ffmpeg -i in.m4a -ar 16000 -ac 1 -c:a pcm_s16le out.wav` (앱에 `ffmpeg-static` 번들)

| 모델 | 디스크 | 메모리 | 속도 (Apple Silicon) | 비고 |
|---|---|---|---|---|
| tiny | 75 MiB | – | 매우 빠름 | 한국어 품질 낮음 |
| large-v3 | 2.9 GiB | ~3.9 GB | M3/M4에서 실시간의 2–3배 | 최고 품질, 느림 |
| **large-v3-turbo** | ~1.6 GB (q5 양자화 ~550 MB) | 더 적음 | large-v3보다 약 2배 빠름 | **기본 추천** |

- **목표 성능**: 60분 회의를 M 시리즈 맥에서 10분 안에 전사합니다. 실측은 M1 패키징 스파이크에서 합니다.
- **모델 배포**: 앱에 넣지 않고 **첫 실행 때 다운로드**합니다(이어받기 지원, 크기 안내).

### Node/Electron 연결 방식 (M1 스파이크에서 결정)

| 방식 | 장점 | 단점 |
|---|---|---|
| **whisper.cpp CLI 바이너리 번들** + `child_process` | 패키징 단순, 크래시 격리, Metal 직접 사용 | 진행률은 stdout을 파싱해야 함 |
| `@fugood/whisper.node` | 유지보수 활발, 사전 빌드 바이너리, Metal·Vulkan·CUDA | 네이티브 모듈 → `asarUnpack` 필요, 아키텍처별 빌드 |
| `whisper-node-addon` | Electron 무설정을 표방 | 성숙도 확인 필요 |
| `nodejs-whisper` / `smart-whisper` / `whisper-node` | – | 오래됐거나 업데이트가 뜸함 |

- 어느 방식이든 STT는 Electron `utilityProcess`에서 실행해 UI 멈춤을 막습니다.

### 그 밖의 로컬 후보 (P2 검토)

- **Qwen3-ASR, SenseVoice**: 한국어·중국어·일본어 성능이 Whisper보다 좋다는 보고가 있습니다(Qwen3-ASR 한국어 CER 3.54% 보고, **미검증**).
- **로컬 화자 구분**:
  - pyannote community-1: CC-BY-4.0, **Python 전용**이라 Electron 번들이 어렵습니다.
  - sherpa-onnx: ONNX 기반이라 Node에서 쓸 수 있는 후보입니다.

## 2. 클라우드 STT

| 서비스 | 시간당 가격 | 한국어 | 화자 구분 | 비고 |
|---|---|---|---|---|
| **Soniox** | ~$0.10 (비동기) | AssemblyAI의 2026-09 벤치마크에서 한국어 최상위 (FLEURS CER 4.4%) | ✅ 포함 | 파일당 300분 제한. **v1 클라우드 기본** |
| AssemblyAI Universal-2 | $0.15 (+$0.02 화자 구분) | 99개 언어 중 하나 | ✅ | Universal-3.5 Pro의 한국어 지원은 **미검증** |
| ElevenLabs Scribe v2 | $0.22 | "좋음" 등급 | ✅ | 전체 오류율 최저 수준 (2026-06) |
| Deepgram Nova-3 | ~$0.26 | 2025년에 한국어 추가 | ✅ (배치) | 구 모델 nova-2는 한국어 성능 낮음 |
| OpenAI `gpt-transcribe` | $0.27 | ✅ | ❌ **화자 구분 없음** | 2026-07 출시. 25MB 업로드 제한 → 분할 필요 |
| OpenAI `whisper-1` / `gpt-4o-transcribe` | $0.36 | ✅ | diarize 모델은 폐기 예정 (**미검증**) | |
| Groq whisper-large-v3-turbo | $0.04 (**무료 티어: 하루 8시간**) | Whisper 수준 | ❌ | 저렴한 무료 옵션 |
| 리턴제로(RTZR) / 네이버 CLOVA Speech | 비공개 | RTZR 벤치마크에서 최상위 | ✅ | 국내 업체. 해당 벤치마크는 RTZR 자체 발표 |

### 비용 예시 (1시간 회의)

- 로컬: $0
- Soniox: 약 $0.10
- Groq: 무료 티어 안이면 $0
- LLM 요약 비용은 별도입니다. 1시간 회의 전사본은 한국어 약 2~4만 토큰이며, 모델에 따라 수 센트에서 수십 센트가 듭니다.
- 앱은 처리 전에 **예상 비용**을 보여줍니다.

## 3. 공통 인터페이스 설계

```ts
interface SttProvider {
  id: 'local-whisper' | 'soniox' | 'groq' | 'openai' | 'elevenlabs' | 'assemblyai' | string;
  capabilities: { diarization: boolean; languageHint: boolean; vocabularyHint: boolean; maxFileMB?: number };
  estimateCost(durationSec: number): number;      // USD, 로컬은 0
  transcribe(input: AudioInput, opts: {
    language: 'ko' | 'auto';
    vocabulary: string[];                          // 용어집(사람 이름·제품명)
    onProgress: (ratio: number) => void;
    signal: AbortSignal;
  }): Promise<Transcript>;                         // segments[{id, start, end, speaker?, text}]
}
```

- **용어집 전달**: whisper는 `initial_prompt`로, 클라우드는 각 서비스의 context·keyterm 기능으로 넘깁니다.
- **로컬 전용 모드**가 켜져 있으면 클라우드 제공자는 목록에서 비활성화되고, 호출도 코드 수준에서 차단됩니다.

출처: [whisper.cpp](https://github.com/ggml-org/whisper.cpp) · [whisper.node](https://github.com/mybigday/whisper.node) · [whisper-node-addon](https://github.com/Kutalia/whisper-node-addon) · [Apple Silicon 벤치마크](https://justvoice.ai/blog/whisper-benchmark-apple-silicon-m3-m4) · [한국어 음성인식 벤치마크(RTZR)](https://github.com/rtzr/Awesome-Korean-Speech-Recognition) · [Qwen3-ASR vs Whisper](https://whispernotes.app/blog/qwen3-asr-vs-whisper) · [pyannote community-1](https://huggingface.co/pyannote/speaker-diarization-community-1) · [Soniox](https://soniox.com/korea) · [AssemblyAI 가격](https://www.assemblyai.com/pricing) · [ElevenLabs 가격](https://elevenlabs.io/pricing/api) · [Deepgram 한국어](https://deepgram.com/learn/deepgram-expands-nova-3-with-11-new-languages-across-europe-and-asia) · [OpenAI Audio FAQ](https://help.openai.com/en/articles/7031512-audio-api-faq) · [Groq](https://console.groq.com/docs/model/whisper-large-v3) · [RTZR](https://developers.rtzr.ai/)
