# call-agent

**한국어** · [English](./README.md)

> 한 줄 요약: 사용 중인 AI CLI(예: Claude Code)가 못 하는 일을 다른 AI CLI(Codex, Antigravity, Kiro, Claude Code, NotebookLM)에게 **자동으로 떠넘기게** 해주는 단일 "위임 스킬"(`call-agent`)입니다.

---

## 0. 이 문서를 읽기 전에 — 용어 미리보기

처음 보는 단어가 많을 수 있어 먼저 정리합니다.

| 용어 | 풀어쓰면 |
|---|---|
| **Agentic CLI** | 터미널에서 도는 AI 도구. 사람이 "이거 해줘" 하면 스스로 파일을 읽고/고치고/명령을 실행함. 대표 예: Claude Code, OpenAI Codex CLI, Google Antigravity(`agy`), AWS Kiro CLI(`kiro-cli`). |
| **Host (호스트)** | 지금 당신이 켜놓고 대화 중인 CLI. 보통 Claude Code 또는 Codex. |
| **Skill (스킬)** | Claude Code / Codex가 "특정 상황에서 어떻게 행동할지"를 적어둔 폴더. `SKILL.md` 한 장 + 부속 스크립트로 구성. 호스트가 자동으로 발견·로드. |
| **Router (라우터)** | `SKILL.md`가 직접 일을 처리하지 않고, 요청을 분류해 딱 하나의 상세 참조 파일만 로드하는 스킬. `call-agent`가 라우터입니다. |
| **Delegation (위임)** | 호스트가 직접 처리하지 않고 **다른 CLI를 셸에서 실행**시켜 결과를 받아오는 행동. 이 저장소가 제공하는 핵심 동작. |
| **MCP** | Model Context Protocol. 여러 AI 도구가 공통으로 외부 서버(파일·DB·웹 등)에 접근하기 위한 규약. `kiro` 타깃의 cross-registry 기능에서 등장. |
| **RAG** | Retrieval-Augmented Generation. 모델이 답하기 전에 외부 문서를 먼저 검색해 근거로 사용하는 방식. NotebookLM이 잘하는 일. |

---

## 1. 왜 만들었나요?

요즘 비슷한 시기에 여러 회사가 "내 AI CLI"를 내놓았습니다. 다 똑같아 보이지만 **각자 잘하는 게 다릅니다.**

- **Claude Code** — 코드 편집·기획·1M 토큰 컨텍스트로 큰 코드베이스 분석에 강함. 이미지 생성은 못 함.
- **OpenAI Codex CLI** — 이미지 생성(`gpt-image-2`)이 가능하고, `codex review`로 PR 단위 리뷰가 강함.
- **Google Antigravity (`agy`)** — Google Search를 모델 추론 단계에 끼워주는 grounding 기능, gnomAD/UniProt 같은 과학 DB 직결.
- **AWS Kiro CLI (`kiro-cli`)** — 헤드리스 자동화에 강하고, "자연어 → 셸 명령" 번역 기능이 있으며, AWS Bedrock 계열 모델로 2차 의견을 받음.
- **NotebookLM** — PDF·웹·유튜브를 통째로 넣고 정확한 RAG 답변, 오디오 개요 생성.

이 상황에서 흔히 일어나는 불편:

> "지금 Claude Code로 작업 중인데 다이어그램 이미지가 하나 필요해. 일일이 다른 터미널 열어서 Codex 켜고… 귀찮네."

**call-agent**은 그 귀찮음을 없앱니다. Claude Code 안에서 "이미지 하나 만들어줘" 라고 하면, `call-agent` 스킬이 발동해 요청을 Codex로 라우팅하고, 이미지를 받아와 경로를 보고합니다. 사용자는 CLI를 갈아탈 필요가 없습니다.

---

## 2. 스킬 하나, 의도 기반 라우팅

예전 버전은 스킬을 **5개**(`agy-call`, `codex-call`, …) 따로 두었습니다. 지금은 **하나**의 스킬 `call-agent`로 통합되어 **의도 기반 라우팅**을 합니다 — `supergoal` 같은 스킬이 쓰는 "스킬 한 장 + 라우터" 패턴과 동일합니다.

`call-agent/SKILL.md`는 얇은 라우터입니다:

1. 작은 표를 기준으로 요청을 분류합니다(어느 타깃 도구인지, 어떤 기능인지).
2. 정확히 하나의 상세 파일 `reference/<target>/call.md`만 로드해 따릅니다.

스킬 5개 대신 1개로 둔 이유:

- 라우팅 시점에 모델이 봐야 할 **description이 5개가 아니라 1개**.
- 새 타깃을 추가할 **한 곳**: `reference/<name>/call.md`를 넣고 라우터 표에 한 줄 추가.
- **호스트 중립.** 같은 스킬이 Claude Code와 Codex 양쪽에 설치됩니다. 라우터의 0번 규칙: *자기 자신은 절대 부르지 않는다* — Claude Code 안에서는 `claude`로, Codex 안에서는 `codex`로 라우팅하지 않습니다.

### `call-agent`는 언제 자동 발동하나요?

두 가지 조건 중 하나일 때만 발동합니다. 의도치 않게 외부 CLI가 호출되면 비용·시간 낭비라서 일부러 좁게 잡았습니다.

1. **사용자가 도구 이름을 명시한 경우** — "agy로 검색해줘", "ask codex", "NotebookLM에 넣어줘" 등.
2. **호스트 CLI가 native로 못 하는 기능을 요청한 경우** — 예: Claude Code 사용 중 "이 도식 이미지로 그려줘" → Claude Code는 이미지 생성을 못 하므로 `codex`로 라우팅.

루틴한 작업(`이 코드 리뷰해줘`, `이 기능 계획 짜줘`)에는 발동하지 않습니다. 호스트 본인이 충분히 잘하기 때문입니다.

---

## 3. `call-agent`가 라우팅할 수 있는 타깃

| 타깃 | 어떤 능력을 빌려오나? | 자주 쓰는 사례 | 명시 트리거 단어 |
|---|---|---|---|
| `codex` | OpenAI Codex | 고품질 이미지 생성(`gpt-image-2`), `codex review` 명시적 코드 리뷰 | "codex" |
| `agy` | Google Antigravity | 최신 웹 grounding 검색, 이미지 생성, gnomAD/UniProt/PubMed 과학 DB, 2차 의견 | "agy", "antigravity", "gemini cli" |
| `kiro` | AWS Kiro CLI(`kiro-cli`) | 자연어 → 셸 번역, MCP 서버 교차 등록, AWS Bedrock 모델 2차 의견 | "kiro", "kiro-cli" |
| `claude` | Claude Code | 1M 컨텍스트 plan-mode 기획, 대형 코드베이스 심층 리뷰(비-Claude 호스트용) | "claude", "claude code" |
| `notebooklm` | Google NotebookLM | PDF/URL/YouTube corpus RAG, 오디오 개요 | "notebooklm", "nblm" |
| `gpt-pro` | ChatGPT Pro(웹) | ChatGPT Pro 딥리즈닝 2차 의견 — 시크릿 제거된 컨텍스트 번들을 만들어 chatgpt.com에 수동 붙여넣기(정액 구독, API 비용 없음). 실험적 라이브 브라우저 드라이브 옵션 | "gpt-pro", "chatgpt pro", "gpt5 pro" |

각 타깃의 정확한 호출법·플래그·래퍼 스크립트는 `skills/call-agent/reference/<target>/`에 있습니다. 전체 결정 표는 라우터 `skills/call-agent/SKILL.md`에서 볼 수 있습니다.

### 모델 선택

래퍼는 상대 CLI가 자체 기본 모델을 갖고 있으면(`codex`, `agy`) 모델을 하드코딩하지 않고, `kiro-chat.sh`는 선택적 `--model` 플래그를 받습니다. `claude` 래퍼는 기본값 `opus`를 쓰고 `CLAUDE_MODEL`(별칭 `sonnet`·`haiku`·`fable` 또는 전체 모델명)로 바꿀 수 있습니다. 리뷰는 `CLAUDE_REVIEW_MODEL` → `CLAUDE_MODEL` → `opus` 순으로 적용합니다. 셸 사전 점검은 저렴한 모델을 고정해 쓰며, `haiku` 별칭이 사라지면 `CLAUDE_PROBE_MODEL`로 바꿔 주세요.

---

## 4. 설치

### 4-0. 빠른 설치 (글로벌, 한 줄)

스킬을 코딩 에이전트의 **글로벌** 스킬 폴더에 한 번에 클론·링크:

```bash
git clone https://github.com/cskwork/call-agent && ./call-agent/install.sh
```

`call-agent`를 `~/.claude/skills`와 `~/.codex/skills` **양쪽**(있는 곳)에 symlink 하므로, 어느 프로젝트에서 Claude Code/Codex를 켜든 자동 로드됩니다. 새 세션만 열면 끝.

**에이전트한테 시키고 싶다면?** Claude Code나 Codex에 이 프롬프트를 붙여넣으세요:

> https://github.com/cskwork/call-agent 를 클론하고 `install.sh`를 실행해서 `call-agent` 스킬을 내 글로벌 `~/.claude/skills`와 `~/.codex/skills`에 설치해줘. 그리고 스킬이 인식되는지 확인해줘.

타깃별 제어(dry-run, 제거, 사전 준비)는 아래 4-1 → 4-3 참고.

### 4-1. 사전 준비 (타깃마다 다름)

본인이 **실제로 쓸 타깃만** 준비하면 됩니다. 전부 설치할 필요 없음.

| 타깃 | 필요한 바이너리 | 인증 방법 |
|---|---|---|
| `agy` | `agy` (Antigravity CLI) | `agy install` → Google 계정 로그인 |
| `kiro` | `kiro-cli` (AWS Kiro CLI) | `kiro-cli login` |
| `codex` | `codex` | `codex login` (ChatGPT) 또는 `OPENAI_API_KEY` 환경변수 |
| `notebooklm` | Python 3.10+, `notebooklm` CLI | `notebooklm login` (브라우저 1회) |
| `claude` | `claude` (Claude Code) | `claude auth login` 또는 `ANTHROPIC_API_KEY` 환경변수 |

### 4-2. 저장소 클론

```bash
# GitHub에서
git clone https://github.com/cskwork/call-agent
# 또는 사내 Gitea에서
git clone https://gitea.agentic-worker.store/Donga-AX/cc-agent-call.git

cd call-agent
```

### 4-3. 스킬 심볼릭 링크 생성

`install.sh`는 이 저장소의 `skills/call-agent` 폴더를 호스트 CLI가 보는 위치로 **symlink** 해줍니다. `~/.claude/skills`와 `~/.codex/skills` **양쪽 모두**에 링크되므로, 어떤 CLI를 켜든 다른 타깃에 닿을 수 있습니다. 복사가 아니라 링크라 저장소를 `git pull` 하면 호스트도 즉시 새 버전을 봅니다.

```bash
./install.sh                # call-agent를 양쪽 호스트 스킬 폴더에 링크
./install.sh --dry-run      # 실제로는 안 하고 어디에 무엇이 연결될지만 미리보기
./install.sh --uninstall    # call-agent 링크 제거(+ 남아있는 구 *-call 링크도 정리)
```

설치 후 호스트 CLI(Claude Code 등)를 새 세션으로 열면 스킬이 자동 로드됩니다.

> 예전 5개 스킬 구조에서 넘어오나요? `./install.sh --uninstall`이 낡은 `agy-call` / `codex-call` / … 심볼릭 링크까지 제거해 줍니다. 그 후 `./install.sh`로 단일 `call-agent`를 링크하세요.

---

## 5. 처음 써보기 — 5분 워크스루

상황: **Claude Code로 작업 중인데 README용 일러스트가 필요하다.**

```text
> README 상단에 들어갈 추상적 일러스트 하나 만들어줘. 1:1 비율.
```

Claude Code는 이미지 생성을 native로 못 합니다. `call-agent` 스킬이 트리거되어 다음을 자동 수행합니다:

1. "이미지 생성"으로 분류 → `codex` 타깃으로 라우팅하고 `reference/codex/call.md`를 로드
2. 프롬프트를 Codex 형식으로 다듬어 `codex` 실행, `gpt-image-2`로 이미지 생성
3. 결과 파일 경로를 받아 Claude Code 대화창에 보고

당신은 CLI를 갈아타지 않았습니다.

다른 예:

- "agy로 'gpt-image-2 가격' 검색해줘" → `agy`로 라우팅, Google Search grounding으로 응답
- "이 PDF 5개에 대해 NotebookLM에 묶고 'Section 3 결론' 물어봐" → `notebooklm`으로 라우팅, corpus 생성 후 질의

---

## 6. 트리거 정책이 보수적인 이유

라우터는 명시적 도구 이름이나 실제 호스트 기능 공백일 때만 위임을 발동합니다. 이유:

- 외부 CLI 호출은 보통 별도 토큰/크레딧을 소비합니다.
- 호스트가 스스로 잘하는 일을 굳이 외주 보내면 응답 속도가 느려집니다.
- 자동 호출이 폭주하면 사용자가 "이 도구가 지금 뭘 하고 있는지" 추적하기 어려워집니다.

그래서 키워드를 좁게 잡았고, 호스트 CLI 자기 자신에게는 절대 위임하지 않습니다.

---

## 7. 테스트

설치 후 동작 확인. 각 타깃의 스모크 테스트를 차례로 실행합니다:

```bash
./tests/run-all.sh                 # L0 + L1: 바이너리 존재, --help 동작, 스크립트 파싱
RUN_L2=1 ./tests/run-all.sh        # L2 추가: 실제 round-trip 프롬프트 (크레딧 소모)
RUN_L3=1 ./tests/run-all.sh        # L3 추가: 핵심 기능(이미지 생성 등) 실행 (크레딧 소모)
RUN_L4=1 ./tests/run-all.sh        # L4 추가: 장시간 비동기 작업(codex) (크레딧 소모)
```

CLI가 설치되지 않은 타깃은 FAIL이 아니라 **SKIP**으로 보고됩니다. 따라서 일부만 설치해도(예: `codex` + `claude`만) `RESULT: OK`로 끝납니다. 실제 오류일 때만 실패합니다.

각 타깃별 개별 스모크 테스트:

```bash
./skills/call-agent/reference/agy/tests/smoke.sh
./skills/call-agent/reference/codex/tests/smoke.sh
# ... 등
```

L2/L3은 실제로 외부 모델을 호출하므로 비용이 발생할 수 있습니다. CI에 거는 경우 L0/L1만 켜두는 것을 권장합니다.

---

## 8. 자주 묻는 질문

**Q. 호스트 CLI를 안 쓰는데 그냥 외부 CLI만 쓰면 안 되나요?**
됩니다. 이 저장소는 **호스트 안에서 흐름을 끊지 않고 외부 CLI를 부르는 경우**에만 의미가 있습니다. 단독 사용이라면 그냥 해당 CLI를 직접 쓰는 게 빠릅니다.

**Q. 스킬이 자동 호출되는 게 싫어요.**
호스트 CLI 측에서 `call-agent`를 비활성화하거나, `install.sh --uninstall`로 symlink만 제거하면 됩니다. 저장소 자체는 그대로 둬도 무방합니다.

**Q. 새 CLI(예: 차세대 도구)도 추가할 수 있나요?**
가능하고, 이제는 설치 스크립트를 건드릴 필요 없이 두 단계면 됩니다:
1. `skills/call-agent/reference/<new-name>/call.md` 작성(호출법, preflight, 스크립트).
2. `skills/call-agent/SKILL.md`의 **Route** 표에 한 줄 추가.

**Q. 보안은?**
이 저장소는 토큰을 보관하지 않습니다. 각 CLI의 인증은 해당 도구의 표준 방식(설정 파일 / 환경변수)을 그대로 사용합니다.

---

## 9. License

MIT
