# Negative-First Mesh 설치 및 적용 가이드

[**한국어**](INSTALL.ko.md) | [English](INSTALL.en.md) | [日本語](INSTALL.ja.md) | [简体中文](INSTALL.zh-CN.md)

검증 기준일: **2026-07-24**

이 스킬은 Agent Skills 형식의 `SKILL.md`를 지원하는 Codex, Claude Code, Gemini CLI에 직접 설치할 수 있다. 네이티브 스킬을 지원하지 않는 일반 채팅형 LLM에서는 프로젝트 지침이나 시스템·개발자 프롬프트로 적용한다.

## 1. 설치 언어 선택

에이전트가 실행하는 진입 파일 이름은 정확히 `SKILL.md`여야 한다. 언어별 파일 중 하나를 선택하여 설치 폴더에서 `SKILL.md`로 복사한다.

| 언어 | 원본 파일 |
|---|---|
| 한국어 | `SKILL.ko.md` |
| 영어 | `SKILL.en.md` |
| 일본어 | `SKILL.ja.md` |
| 중국어 간체 | `SKILL.zh-CN.md` |

```bash
mkdir -p /path/to/target/negative-first-mesh
cp SKILL.ko.md /path/to/target/negative-first-mesh/SKILL.md
```

설치용 스킬 폴더에는 실행 진입점인 `SKILL.md`를 하나만 두는 것이 안전하다.

## 2. Codex와 Gemini CLI 공통 경로

Codex와 Gemini CLI는 `.agents/skills/` 경로를 공통으로 사용할 수 있다.

### 프로젝트 범위

```bash
mkdir -p .agents/skills/negative-first-mesh
cp /path/to/negative-first-mesh-skill/SKILL.ko.md \
  .agents/skills/negative-first-mesh/SKILL.md
```

### 사용자 전체 범위

```bash
mkdir -p "$HOME/.agents/skills/negative-first-mesh"
cp /path/to/negative-first-mesh-skill/SKILL.ko.md \
  "$HOME/.agents/skills/negative-first-mesh/SKILL.md"
```

Claude Code는 별도의 `.claude/skills/` 경로를 사용한다.

## 3. OpenAI Codex

### 프로젝트 설치

프로젝트 루트에서:

```bash
mkdir -p .agents/skills/negative-first-mesh
cp /path/to/negative-first-mesh-skill/SKILL.ko.md \
  .agents/skills/negative-first-mesh/SKILL.md
```

저장소에 커밋하면 팀이 같은 스킬을 사용할 수 있다.

### 사용자 전체 설치

```bash
mkdir -p "$HOME/.agents/skills/negative-first-mesh"
cp /path/to/negative-first-mesh-skill/SKILL.ko.md \
  "$HOME/.agents/skills/negative-first-mesh/SKILL.md"
```

### 관리자 범위 설치

```bash
sudo mkdir -p /etc/codex/skills/negative-first-mesh
sudo cp SKILL.ko.md /etc/codex/skills/negative-first-mesh/SKILL.md
```

### 확인 및 호출

Codex에서:

```text
/skills
```

명시적 호출:

```text
$negative-first-mesh 이 주장을 반례·범위·근거 기준으로 먼저 제거하며 검증해줘.
```

질문이 YAML의 `description`과 일치하면 자동으로 선택될 수도 있다. 새 스킬이 보이지 않으면 Codex를 다시 시작한다.

## 4. Anthropic Claude Code

### 프로젝트 설치

```bash
mkdir -p .claude/skills/negative-first-mesh
cp /path/to/negative-first-mesh-skill/SKILL.ko.md \
  .claude/skills/negative-first-mesh/SKILL.md
```

### 사용자 전체 설치

```bash
mkdir -p "$HOME/.claude/skills/negative-first-mesh"
cp /path/to/negative-first-mesh-skill/SKILL.ko.md \
  "$HOME/.claude/skills/negative-first-mesh/SKILL.md"
```

### 확인 및 호출

```bash
claude
```

직접 호출:

```text
/negative-first-mesh
```

또는 자연어로 요청한다.

```text
이 질문을 Negative-First 방식으로 검증하고, 근거 없는 후보를 먼저 제외해줘.
```

기존 스킬 폴더의 수정은 일반적으로 세션 중 감지된다. 세션 시작 후 최상위 `.claude/skills/` 폴더를 처음 생성했다면 재시작이 필요할 수 있다.

## 5. Claude 웹·데스크톱 앱 업로드

Claude의 일반 채팅이나 Cowork에서는 언어별 ZIP을 사용자 정의 스킬로 업로드할 수 있다.

1. **Customize → Skills**로 이동한다.
2. `+`를 누르고 **Create skill**을 선택한다.
3. **Upload a skill**을 선택한다.
4. `negative-first-mesh-ko.zip`을 업로드한다.
5. 스킬 목록에서 활성화한다.

사용자 정의 스킬은 코드 실행 기능이 필요하며, 조직 공유 범위는 계정과 조직 정책에 따라 달라질 수 있다.

## 6. Google Gemini CLI

Gemini CLI는 `.gemini/skills/`와 상호운용 경로인 `.agents/skills/`를 인식한다.

### 프로젝트에 직접 복사

```bash
mkdir -p .gemini/skills/negative-first-mesh
cp /path/to/negative-first-mesh-skill/SKILL.ko.md \
  .gemini/skills/negative-first-mesh/SKILL.md
```

Codex와 공유하려면 `.agents/skills/`를 사용한다.

### 사용자 전체 설치

```bash
mkdir -p "$HOME/.gemini/skills/negative-first-mesh"
cp /path/to/negative-first-mesh-skill/SKILL.ko.md \
  "$HOME/.gemini/skills/negative-first-mesh/SKILL.md"
```

### 로컬 폴더 연결

```bash
gemini skills link /absolute/path/to/negative-first-mesh --scope workspace
```

사용자 범위는 `--scope user`를 사용한다.

### Git 저장소에서 설치

```bash
gemini skills install https://github.com/example/negative-first-mesh.git \
  --scope workspace
```

### 확인 및 새로고침

Gemini CLI 내부:

```text
/skills list
/skills reload
```

터미널:

```bash
gemini skills list --all
```

스킬이 포함 파일과 폴더에 접근할 때 동의를 요청할 수 있다.

## 7. Gemini CLI의 `GEMINI.md` 방식

모든 질문에 같은 정책을 지속 적용하려면 프로젝트의 `GEMINI.md`에 요약 지침을 넣는다.

```markdown
# Project reasoning policy

For factual, diagnostic, comparison, recommendation, and retrieval questions:
1. Expand plausible candidates.
2. Eliminate contradicted, out-of-scope, and unsupported candidates first.
3. Distinguish false, not found in scope, and unknown.
4. Generate an answer only from surviving evidence.
```

별도 파일을 가져올 수도 있다.

```markdown
@./negative-first-mesh-instructions.md
```

`GEMINI.md`는 지속 지침이고, Agent Skill은 관련 작업에서 필요할 때 선택적으로 활성화되는 기능이다.

## 8. 네이티브 스킬을 지원하지 않는 LLM

웹 채팅, 로컬 모델 UI, API 래퍼처럼 `SKILL.md` 자동 검색을 지원하지 않는 환경에서는 다음 방식으로 적용한다.

### 프로젝트 지침

`SKILL.ko.md`를 프로젝트 파일로 첨부하고 다음 활성화 문구를 추가한다.

```text
사실 확인, 진단, 비교, 추천, 정보 검색 질문에서는
첨부한 SKILL.md의 Negative-First Mesh 절차를 적용한다.
모순, 범위 밖, 근거 부족 후보를 먼저 제거하고,
FALSE / NOT_FOUND_IN_SCOPE / UNKNOWN을 구분한다.
```

### 시스템 또는 개발자 프롬프트

API 사용 시 질문을 먼저 분류하고, 사실 확인·진단·비교·추천·검색 질문일 때만 스킬 본문을 시스템 또는 개발자 지침에 삽입한다. 모든 요청에 긴 본문을 항상 넣으면 입력 토큰과 비용이 증가한다.

### 수동 호환 프롬프트

```text
답을 먼저 만들지 마라.
가능한 후보와 해석을 펼친 뒤 다음을 먼저 제거하라:
1. 정의 불일치
2. 필수조건 위반
3. 시점·범위 불일치
4. 근거 부족
5. 강한 반증과 모순

남은 근거만으로 답하고, 후보가 없으면
FALSE / NOT_FOUND_IN_SCOPE / UNKNOWN을 구분하라.
```

## 9. 언어별 배포 ZIP

패키지에는 바로 업로드하거나 압축 해제해 복사할 수 있는 파일이 포함된다.

- `packages/negative-first-mesh-ko.zip`
- `packages/negative-first-mesh-en.zip`
- `packages/negative-first-mesh-ja.zip`
- `packages/negative-first-mesh-zh-CN.zip`

각 ZIP의 구조:

```text
negative-first-mesh/
├── SKILL.md
├── README.md
├── EXAMPLES.md
├── TESTS.md
└── INSTALL.md
```

## 10. 설치 검증 질문

```text
SNS 이미지에 적힌 정책 변경이 사실인지 확인해줘.
공식 근거가 없으면 사실처럼 만들지 말고,
거짓 / 확인 범위 내 미발견 / 판단 불가를 구분해줘.
```

정상 작동 시 주장·시점·출처를 분리하고, 이미지 문구를 곧바로 사실로 확정하지 않으며, 근거가 없으면 내용을 만들지 않는다.

## 11. 업데이트와 제거

### Codex

```bash
rm -rf .agents/skills/negative-first-mesh
```

파일 교체 후 필요하면 Codex를 다시 시작한다.

### Claude Code

```bash
rm -rf .claude/skills/negative-first-mesh
```

### Gemini CLI

```text
/skills reload
```

또는:

```bash
gemini skills uninstall negative-first-mesh --scope workspace
```

## 12. 주의사항

- 스킬은 플랫폼 안전정책이나 더 높은 우선순위 지침을 대체하지 않는다.
- “아니오 우선”은 무조건 부정적으로 답하는 방식이 아니다.
- 검색하지 못한 것을 거짓으로 처리해서는 안 된다.
- 최신 정보 질문에는 최신 공식 자료나 웹 접근이 필요하다.
- 파일·웹·데이터베이스 도구가 없으면 주어진 문맥 범위에서만 판정한다.
- 설치 경로와 명령은 제품 업데이트에 따라 바뀔 수 있으므로 공식 문서를 함께 확인한다.

## Official References

- OpenAI Codex skills: https://developers.openai.com/codex/build-skills
- Claude Code skills: https://code.claude.com/docs/en/skills
- Claude custom skills upload: https://support.claude.com/en/articles/12512180-use-skills-in-claude
- Gemini CLI Agent Skills: https://geminicli.com/docs/cli/skills/
- Gemini CLI skill management: https://geminicli.com/docs/cli/using-agent-skills/
- Gemini CLI `GEMINI.md`: https://geminicli.com/docs/cli/gemini-md/
