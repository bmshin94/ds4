# DwarfStar (ds4) 완전 정복 가이드 🌟

> 이 문서는 DwarfStar 프로젝트를 처음 접한 사람을 위해
> **"이게 뭔지 → 어떻게 쓰는지 → 어떻게 돈을 버는지"** 순서로 정리한 한국어 가이드입니다.

## 📌 프로젝트 주소

| 항목 | 링크 |
| --- | --- |
| 이 저장소 | https://github.com/bmshin94/ds4 |
| 모델 가중치 (Hugging Face) | https://huggingface.co/antirez/deepseek-v4-gguf |
| 원본 모델 카드 | https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash |
| 기반이 된 프로젝트 | https://github.com/ggml-org/llama.cpp |

---

## 1. 한 줄 요약

> **"내 컴퓨터 안에 ChatGPT를 통째로 설치하는 프로그램"**

DwarfStar는 **DeepSeek V4 / GLM 5.2 전용 로컬 추론 엔진**입니다.
클라우드 서버가 아니라 내 맥북, 내 워크스테이션 안에서 거대 언어 모델을 직접 실행합니다.

### 클라우드 AI와 뭐가 다른가

| | 클라우드 AI (ChatGPT 등) | DwarfStar |
| --- | --- | --- |
| 실행 위치 | 저 멀리 회사 서버 | **내 컴퓨터** |
| 인터넷 | 필수 | **불필요** |
| 비용 | 토큰당 과금 | **전기값만** |
| 데이터 | 외부로 전송됨 | **밖으로 안 나감** |

### 왜 llama.cpp 대신 이걸 쓰나

```
llama.cpp  = 만능 스위스 아미 나이프 (아무 모델이나 다 돌림)
DwarfStar  = DeepSeek V4 전용 사시미 칼 (딱 이것만, 대신 극한으로 빠름)
```

README에 **"범용 GGUF 로더가 아니다"** 라고 명시되어 있습니다.
모델 로딩 · 프롬프트 렌더링 · 툴 호출 · KV 캐시 · HTTP 서버 · 코딩 에이전트를
**하나의 모델에 맞춰 통째로 같이 설계**했기 때문에 빠르고 안정적입니다.

### 만든 사람

**antirez (Salvatore Sanfilippo)** — Redis 창시자.
README에 GPT-5.5 / 5.6 / Claude Fable의 도움을 받아 개발했다고 **직접 공개**되어 있습니다.

---

## 2. 폴더 구조 한눈에 보기

| 파일 / 폴더 | 크기 | 역할 (쉽게) |
| --- | ---: | --- |
| `ds4.c` | 2.9 MB | **엔진** — 모델 로딩, 토크나이저, 추론의 심장 |
| `ds4_cli.c` | 84 KB | **채팅창** — `./ds4` 터미널 대화 |
| `ds4_server.c` | 708 KB | **콘센트** — OpenAI/Anthropic 호환 HTTP API |
| `ds4_agent.c` | 445 KB | **비서** — 내장 코딩 에이전트 |
| `ds4_metal.m` + `metal/` | 1.9 MB | 맥 GPU(Metal) 커널 |
| `ds4_cuda.cu` + `cuda/` | 1.3 MB | NVIDIA GPU 커널 |
| `rocm/` | - | AMD GPU (Strix Halo) 커널 |
| `ds4_distributed.c` | 324 KB | 여러 컴퓨터를 파이프라인으로 연결 |
| `ds4_tp.c` | 87 KB | 텐서 병렬 (RDMA) |
| `ds4_kvstore.c` / `ds4_ssd.c` | - | 대화 기억(KV 캐시) 저장 / SSD 스트리밍 |
| `ds4_eval.c` | 207 KB | 성능 평가 |
| `gguf-tools/` | - | 모델 양자화·imatrix 제작 도구 |
| `dir-steering/` | - | **모델 성격 조종 다이얼** |
| `speed-bench/` | - | 기기별 속도 측정 CSV + 그래프 |
| `tests/` | - | 단위·통합 테스트 |
| `download_model.sh` | 13 KB | 모델 자동 다운로드 |
| `misc/` | - | 부가 설계 문서 |

### 참고 문서

- `README.md` — 메인 문서 (76 KB)
- `AGENT.md` — 코드 설계 철학 및 규칙
- `CONTRIBUTING.md` — 기여 시 회귀 테스트 가이드
- `QA_BEFORE_RELEASES.md` — 릴리스 전 QA 매트릭스
- `MODEL_CARD.md` — DeepSeek V4 모델 카드 요약
- `STRIXHALO.md` — AMD Strix Halo 전용 안내

---

## 3. 핵심 기술 (쉬운 비유)

### 1) 비대칭 2비트 양자화
모델 용량의 대부분인 **라우팅 전문가(MoE experts)만** 2비트로 압축하고,
품질에 민감한 나머지(공유 전문가, 프로젝션, 라우팅)는 **원본 그대로** 둡니다.

> 무손실 원본 사진 → 화질 거의 그대로인 JPG 만든 느낌 📸

덕분에 284B 모델이 **81GB**로 줄어 맥북에 들어갑니다.

### 2) SSD 스트리밍
램이 부족해도 실행됩니다. 자주 안 쓰는 전문가는 SSD에 두고 필요할 때 읽어옵니다.
단순히 읽는 게 아니라 **"현재 레이어를 계산하는 동안 다음 레이어를 미리 로딩"** 해서
대기 시간을 숨깁니다.

> 책상엔 자주 보는 책만, 나머지는 책장에 📚

### 3) 여러 대 묶기
- 맥북 2대 → 썬더볼트 RDMA 텐서 병렬
- 여러 서버 → 파이프라인 병렬로 램 합치기
- 구형 NVIDIA L40S 8장 → 사내 LLM 서버로 부활 (생성 120 t/s, 프리필 2000 t/s)

### 4) 네이티브 에이전트
일반적인 에이전트는 `에이전트 ↔ API 서버`를 왔다갔다 하지만,
DwarfStar는 **추론 엔진 안에 에이전트가 들어있습니다.**
소켓 경계가 없어서 대화 세션 = KV 캐시 그 자체 → 세션 시작·툴 호출·출력이 **즉각적**.

### 5) 방향성 스티어링 (`dir-steering/`)
["Refusal in Language Models Is Mediated by a Single Direction"](https://arxiv.org/abs/2406.11717)
논문 기반으로 **모델 내부 활성값을 실시간 조작**합니다. 파인튜닝이 필요 없습니다.

```text
y = y - scale * direction[layer] * dot(direction[layer], y)
```

- "말 좀 짧게 해"
- "렌터카 챗봇이니까 코딩 질문엔 대답하지 마"

이런 걸 벡터 하나로 조종합니다. **파인튜닝(수백만원 + 몇 주) 없이 즉시 적용.**

---

## 4. 속도 (README 기준)

| 기기 | 백엔드 | 컨텍스트 | 프리필 | 생성 |
| --- | --- | ---: | ---: | ---: |
| MacBook Pro M5 Max 128GB | Metal | 2048 | 790.18 t/s | **39.35 t/s** |
| MacBook Pro M5 Max 128GB | Metal | 16384 | 572.53 t/s | 36.14 t/s |
| MacBook Pro M5 Max 128GB | Metal | 32768 | 557.04 t/s | 34.36 t/s |
| MacBook Pro M5 Max 128GB | Metal | 65536 | 398.50 t/s | 27.64 t/s |
| DGX Spark GB10 128GB | CUDA | 2048 | 825.76 t/s | 18.05 t/s |
| DGX Spark GB10 128GB | CUDA | 65536 | 822.98 t/s | 13.84 t/s |

> 39 t/s면 사람이 읽는 속도보다 훨씬 빠릅니다. 로컬치고 매우 우수한 수치입니다.

---

## 5. 설치 및 사용법

### STEP 0. 내 컴퓨터가 되는지 확인

| 내 장비 | 가능 | 추천 모델 |
| --- | :---: | --- |
| 맥 128GB | 최고 | `ds4f-q2` / `ds4f-q2-q4` |
| 맥 96GB | 가능 | `ds4f-q2` |
| 맥 64GB | SSD 스트리밍으로 | `ds4f-q2` |
| 맥 32GB 이하 | **불가** | - |
| 맥 256GB+ | 가능 | `ds4f-q4` |
| 맥 512GB | 가능 | `pro-q2-imatrix` |
| Linux + NVIDIA | 가능 | `ds4f-q2` |
| DGX Spark (GB10) | 가능 | `ds4f-q2` |
| AMD Strix Halo | 가능 | GLM 모델 |
| **Windows** | **불가** | - |

```sh
sysctl hw.memsize   # 맥: 메모리 확인
df -h .             # 디스크 여유 공간 확인 (모델이 80~430GB!)
```

### STEP 1. 준비물

```sh
# macOS
xcode-select --install

# Linux + NVIDIA
nvcc --version   # CUDA 툴킷 확인
```

### STEP 2. 빌드

```sh
git clone https://github.com/bmshin94/ds4
cd ds4

# 자기 환경에 맞는 것 하나만
make                  # macOS (Metal)
make cuda-spark       # Linux, DGX Spark / GB10
make cuda-generic     # Linux, 일반 NVIDIA GPU
make strix-halo       # Linux, AMD Strix Halo
make cpu              # CPU 전용 (디버그/참조용, 매우 느림)
```

빌드하면 실행 파일 5개가 생성됩니다.

| 실행 파일 | 용도 |
| --- | --- |
| `./ds4` | 터미널 채팅 |
| `./ds4-server` | **API 서버 (수익화 핵심)** |
| `./ds4-agent` | 코딩 에이전트 |
| `./ds4-bench` | 속도 측정 |
| `./ds4-eval` | 성능 평가 |

### STEP 3. 모델 다운로드

```sh
./download_model.sh ds4f-q2      # 81GB, 대부분 이걸로 시작
```

| 명령 | 용량 | 대상 |
| --- | ---: | --- |
| `ds4f-q2` | 81 GB | 96/128GB 램 (권장) |
| `ds4f-q2-q4` | 98 GB | 128GB 맥북, 품질 향상 |
| `ds4f-q4` | 153 GB | 256GB 이상 |
| `ds4f-mxfp4` | 156 GB | 네이티브 MXFP4 유지 |
| `pro-q2-imatrix` | 430 GB | 512GB 램 |

- 다운로드가 끊겨도 `curl -C -`로 **이어받기** 됩니다.
- 파일은 `./gguf/`에 저장되고 `./ds4flash.gguf`가 자동으로 연결됩니다.
- `HF_TOKEN` 또는 `--token TOKEN`으로 인증 가능 (공개 파일은 불필요).

### STEP 4. 실행

#### (A) 터미널 채팅

```sh
./ds4                                 # 대화 모드
./ds4 -p "레디스 스트림 설명해줘"       # 한 번만 질문
./ds4 --nothink -p "빠르게 답해줘"      # 사고 과정 끄고 빠르게
```

대화 중 명령어:

```text
/help          도움말
/think         깊게 생각 모드
/nothink       바로 답변 모드
/ctx 32000     컨텍스트 크기 조절
/read FILE     파일 읽어오기
/new           새 대화
/quit          종료
```

`Ctrl+C` = 생성 중단 후 프롬프트로 복귀.

#### (B) 램이 부족할 때 (SSD 스트리밍)

```sh
./ds4 -m ./ds4flash.gguf --ssd-streaming --ssd-streaming-cache-experts 32GB
```

#### (C) API 서버 — 가장 중요

```sh
./ds4-server --ctx 100000 \
  --kv-disk-dir /tmp/ds4-kv \
  --kv-disk-space-mb 8192
```

기본 주소는 `http://127.0.0.1:8000` 입니다.

**지원 엔드포인트 (`ds4_server.c` 확인 완료)**

```text
POST /v1/chat/completions   OpenAI 형식
POST /v1/messages           Anthropic(Claude) 형식
POST /v1/responses          OpenAI Responses 형식
POST /v1/completions        구형 형식
GET  /v1/models             모델 목록
```

주요 옵션:

| 옵션 | 기본값 | 설명 |
| --- | --- | --- |
| `--host HOST` | `127.0.0.1` | 바인드 주소 |
| `--port N` | `8000` | 포트 |
| `--cors` | 꺼짐 | 브라우저 JS 클라이언트용 헤더 |
| `--batched-session N` | 1 | **동시 접속 세션 수** |
| `--kv-disk-dir DIR` | 꺼짐 | 디스크 KV 캐시 |
| `--trace FILE` | 꺼짐 | 프롬프트/출력/툴콜 로깅 |

여러 명 동시 접속:

```sh
./ds4-server --ctx 32768 --batched-session 8 --cors
```

> 컨텍스트 1M 토큰은 약 26GB의 메모리를 사용합니다.
> 128GB 램에서 2비트 양자화(81GB)를 쓴다면 **10만~30만 토큰**이 현실적입니다.

#### (D) 코딩 에이전트

```sh
./ds4-agent
./ds4-agent --chdir /path/to/ds4    # 다른 폴더에서 실행할 때
```

- 세션 저장 위치: `~/.ds4/kvcache`
- `/save` 저장 · `/list` 목록 · `/switch <sha>` 전환 · `/del <sha>` 삭제
- `/strip <sha>` — 대화 텍스트는 남기고 무거운 KV만 제거

#### (E) 벤치마크

```sh
./ds4-bench -m ds4flash.gguf \
  --prompt-file speed-bench/promessi_sposi.txt \
  --ctx-start 2048 --ctx-max 65536 --step-incr 2048 --gen-tokens 128
```

---

## 6. 보안 경고 (중요) 🚨

코드를 직접 확인한 결과:

```sh
grep -in "auth|bearer|api-key" ds4_server.c   # → 결과 없음
```

> **`ds4-server`에는 인증 기능이 전혀 없습니다.**

기본 바인드가 `127.0.0.1`이라 로컬에서는 안전하지만,
`--host 0.0.0.0`으로 인터넷에 직접 노출하면 **누구나 무제한으로 사용**할 수 있습니다.

**반드시 앞단에 인증·과금 게이트웨이를 두세요.** (아래 8장 참고)

---

## 7. 수익화 아이디어

### 라이선스 확인 — 좋은 소식

```text
LICENSE            → MIT (상업적 이용 허용)
MODEL_CARD.md:231  → "저장소와 모델 가중치 모두 MIT 라이선스"
```

**엔진도 MIT, 모델 가중치도 MIT** → 상업적으로 사용해도 법적 문제가 없습니다.
(다수의 로컬 LLM이 상업 이용을 제한하는 것과 대조적인 장점)

### 아이디어 (현실성 순)

#### 1위. 사내 전용 AI 구축 서비스 ★★★★★

- **타겟**: 병원, 법무법인, 회계법인, 금융, 공공기관, 제조 대기업
- **셀링포인트**: *"데이터가 회사 밖으로 1바이트도 나가지 않습니다"*

수요가 확실한 이유:

| 업종 | 클라우드 AI를 못 쓰는 이유 |
| --- | --- |
| 의료 | 환자 정보 — 법적 제약 |
| 법무 | 사건 기록 유출 리스크 |
| 제조 | 설계 도면 기술 유출 |
| 공공 | 폐쇄망이라 인터넷 자체가 없음 |

| 수익 항목 | 가격대 |
| --- | --- |
| 구축비 (1회) | 500만 ~ 3,000만원 |
| 월 유지보수 | 50만 ~ 300만원 |
| 커스터마이징 | 별도 |

README에 **구형 NVIDIA L40S 8장으로 사내 LLM 서버를 구성한 사례**가 있습니다.
vLLM이 지원을 끊은 구형 GPU 서버를 되살려주는 것만으로도 사업이 됩니다.

#### 2위. 저렴한 API 리셀링 ★★★★

```text
[고객] → [PHP 게이트웨이: 인증·과금·제한] → [ds4-server: 실제 AI]
```

- 셀링포인트: "OpenAI 호환 API인데 훨씬 저렴" + "국내 서버라 빠름"
- **솔직한 리스크**: 클라우드 API 가격이 계속 내려가고 있어 순수 가격 경쟁은 어렵습니다.
  → **"데이터 국내 보관 + 무제한 정액제"** 로 차별화해야 승산이 있습니다.

#### 3위. 특화 SaaS ★★★★

일반 챗봇이 아니라 한 분야만 깊게:

- 계약서 검토 SaaS
- 진료기록 요약 SaaS
- 상담 녹취록 분석 SaaS
- 사내 문서 검색 (RAG)

여기서 **`dir-steering/`이 킬러 기능**입니다.
파인튜닝 없이 모델 성격을 즉시 조정할 수 있어 경쟁사가 따라오기 어려운 차별점이 됩니다.

#### 4위. 온프레미스 AI 어플라이언스 ★★★

"전원만 꽂으면 되는 AI 박스"를 하드웨어와 함께 완제품으로 판매.
IT 담당자가 없는 중소기업·병원 대상.

#### 5위. 교육 / 컨설팅 ★★★

- "로컬 LLM 구축 실무" 강의
- 기업 도입 컨설팅
- C 코드가 깔끔해서 교재로도 우수

### 시작 전 리스크

| 리스크 | 내용 |
| --- | --- |
| 초기 비용 | 맥 128GB 약 500만원+, GPU 서버 수천만원 |
| 베타 품질 | README에 "불안정 가능" 명시 |
| 모델 교체 | "더 좋은 모델이 나오면 기존 모델 제거 가능" 명시 |
| Windows 미지원 | 고객사가 Windows 서버면 사용 불가 |
| 가격 경쟁 | 클라우드 API가 계속 저렴해지는 중 |

> **추천: 1번(사내 AI 구축)부터.** 초기 투자가 적고, 프라이버시라는 명분이 확실하며, B2B라 단가가 높습니다.

---

## 8. PHP로 만들 수 있나?

### 결론: 절반은 불가, 절반은 완전 가능

| 부분 | PHP로? | 이유 |
| --- | :---: | --- |
| **AI 엔진** (`ds4.c`) | **불가** | GPU 커널 직접 제어, 수동 메모리 관리 필요 |
| **제품/서비스 레이어** | **완전 가능** | 여기가 PHP의 무대 |

### 엔진을 PHP로 못 만드는 이유

```text
ds4.c        2.9 MB   ← C
ds4_metal.m  1.9 MB   ← macOS GPU 전용
ds4_cuda.cu  1.3 MB   ← NVIDIA GPU 전용
```

1. GPU에 직접 명령을 내려야 하는데 PHP에는 그런 기능이 없음
2. 81GB 메모리를 바이트 단위로 관리해야 하는데 PHP는 자동 관리라 불가능
3. 속도가 수백~수천 배 차이 — PHP로 짜면 한 글자에 수 분

> 비유: **"PHP로 자동차 엔진을 깎겠다"**.
> PHP는 엔진을 깎는 도구가 아니라, **잘 만든 엔진으로 서비스를 만드는 도구**입니다.

### PHP가 꼭 필요한 자리

`ds4-server`에 인증이 전혀 없다는 점(6장)을 기억하세요.
**그 앞을 PHP가 지키는 것**이 정확한 역할 분담입니다.

```text
                고객 (인터넷)
                     │ HTTPS
        ┌────────────────────────────┐
        │  PHP (Laravel 등)          │  ← 직접 만들 부분
        │  · 로그인 / API 키 발급     │
        │  · 사용량 측정 & 과금       │
        │  · 요청 제한 (rate limit)   │
        │  · 채팅 웹 UI              │
        │  · 대화 기록 DB 저장        │
        └────────────────────────────┘
                     │ 내부망 (127.0.0.1)
        ┌────────────────────────────┐
        │  ds4-server (C)            │  ← 그냥 갖다 쓰기
        │  실제 AI 계산만 담당         │
        └────────────────────────────┘
```

**PHP가 담당하는 일 = 돈을 버는 일 전부**입니다.
AI 계산은 이미 만들어진 것을 쓰고, 비즈니스 로직만 작성하면 됩니다.

### 실제 PHP 예제

#### 1) 기본 호출

```php
<?php
function askAI(string $message): string {
    $ch = curl_init('http://127.0.0.1:8000/v1/chat/completions');

    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER     => ['Content-Type: application/json'],
        CURLOPT_TIMEOUT        => 300,   // AI는 느리므로 넉넉하게
        CURLOPT_POSTFIELDS     => json_encode([
            'model'    => 'deepseek-v4-flash',
            'messages' => [
                ['role' => 'system', 'content' => '너는 친절한 상담원이야.'],
                ['role' => 'user',   'content' => $message],
            ],
            'max_tokens' => 2000,
        ], JSON_UNESCAPED_UNICODE),
    ]);

    $res = json_decode(curl_exec($ch), true);
    curl_close($ch);

    return $res['choices'][0]['message']['content'] ?? '(응답 없음)';
}

echo askAI('안녕! 자기소개 해줘');
```

#### 2) 실시간 스트리밍 (타이핑 효과)

```php
<?php
header('Content-Type: text/event-stream');
header('Cache-Control: no-cache');

$ch = curl_init('http://127.0.0.1:8000/v1/chat/completions');
curl_setopt_array($ch, [
    CURLOPT_POST       => true,
    CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
    CURLOPT_POSTFIELDS => json_encode([
        'model'    => 'deepseek-v4-flash',
        'stream'   => true,              // 스트리밍 활성화
        'messages' => [['role' => 'user', 'content' => $_GET['q']]],
    ], JSON_UNESCAPED_UNICODE),

    // 조각이 도착할 때마다 즉시 브라우저로 전달
    CURLOPT_WRITEFUNCTION => function ($ch, $chunk) {
        echo $chunk;
        ob_flush();
        flush();
        return strlen($chunk);
    },
]);
curl_exec($ch);
curl_close($ch);
```

#### 3) 과금 게이트웨이 (수익화 핵심)

```php
<?php
// 1) API 키 검증
$key  = str_replace('Bearer ', '', $_SERVER['HTTP_AUTHORIZATION'] ?? '');
$user = $db->fetch("SELECT * FROM users WHERE api_key = ?", [$key]);

if (!$user) {
    http_response_code(401);
    exit(json_encode(['error' => 'API 키가 올바르지 않습니다']));
}

// 2) 잔액 확인
if ($user['credits'] <= 0) {
    http_response_code(402);
    exit(json_encode(['error' => '크레딧이 부족합니다']));
}

// 3) 실제 AI 호출
$result = callDs4Server($_POST);

// 4) 사용한 토큰만큼 차감 → 이것이 매출
$used = $result['usage']['total_tokens'];
$db->exec("UPDATE users SET credits = credits - ? WHERE id = ?", [$used, $user['id']]);
$db->exec("INSERT INTO usage_log (user_id, tokens, created_at) VALUES (?,?,NOW())",
          [$user['id'], $used]);

echo json_encode($result);
```

> **보안 필수**: `ds4-server`는 절대 `--host 0.0.0.0`으로 인터넷에 직접 노출하지 마세요.
> `127.0.0.1`에 두고 PHP를 통해서만 접근하게 해야 합니다.

### 추천 스택

```text
[Nginx]        HTTPS, 리버스 프록시
    │
[Laravel]      회원, 결제(토스/아임포트), API 키, 관리자
    │
[Redis]        캐시 + 대기열
    │
[ds4-server]   AI 엔진 (127.0.0.1:8000)
    │
[MySQL]        사용자, 대화 기록, 과금 내역
```

### MVP 4주 플랜

| 주차 | 할 일 |
| --- | --- |
| 1주 | ds4 설치 + 서버 기동 + PHP 호출 성공 |
| 2주 | 로그인 + API 키 발급 + 채팅 웹 UI |
| 3주 | 사용량 측정 + 요금제 + 결제 연동 |
| 4주 | 관리자 페이지 + 배포 |

---

## 9. 최종 요약

| 질문 | 답 |
| --- | --- |
| **이게 뭔가** | 내 컴퓨터에서 거대 AI를 직접 돌리는 전용 추론 엔진 |
| **언제 쓰나** | 프라이버시가 중요할 때, API 비용을 없애고 싶을 때, 오프라인 |
| **설치** | `git clone` → `make` → `./download_model.sh ds4f-q2` → `./ds4` |
| **필요 사양** | 맥 64GB 이상 (권장 128GB), 또는 CUDA/ROCm Linux. Windows 불가 |
| **수익화** | MIT 라이선스로 자유. **사내 AI 구축(B2B)** 부터 추천 |
| **PHP로 가능?** | 엔진은 불가, **서비스 레이어는 가능 — 그리고 거기가 돈이 되는 곳** |

### 프로젝트 철학 (README 인용)

> *"AI 코딩 에이전트가 있다면, 그것을 인터페이스 삼아 이 프로젝트를 탐색하고 수정해서
> 나만의 셋업을 만들어라. 문서화되지 않은 것도 의외로 쉽게 달성할 수 있다."*

이 프로젝트는 **완성품이 아니라 잘 만들어진 템플릿이자 레일**로 쓰라는 의도로 배포되었습니다.
