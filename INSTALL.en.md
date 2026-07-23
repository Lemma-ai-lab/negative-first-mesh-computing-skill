# Negative-First Mesh Installation and Integration Guide

[한국어](INSTALL.ko.md) | **[English](INSTALL.en.md)** | [日本語](INSTALL.ja.md) | [简体中文](INSTALL.zh-CN.md)

Verified on: **2026-07-24**

This skill can be installed directly in Codex, Claude Code, and Gemini CLI, which support Agent Skills based on `SKILL.md`. For LLMs without native skill discovery, use project instructions or a system/developer prompt.

## 1. Select a Language

The executable entry file must be named exactly `SKILL.md`.

| Language | Source file |
|---|---|
| Korean | `SKILL.ko.md` |
| English | `SKILL.en.md` |
| Japanese | `SKILL.ja.md` |
| Simplified Chinese | `SKILL.zh-CN.md` |

```bash
mkdir -p /path/to/target/negative-first-mesh
cp SKILL.en.md /path/to/target/negative-first-mesh/SKILL.md
```

Keep one executable `SKILL.md` in each installed skill folder.

## 2. Shared Codex and Gemini Path

Codex and Gemini CLI can both discover `.agents/skills/`.

```bash
mkdir -p .agents/skills/negative-first-mesh
cp /path/to/negative-first-mesh-skill/SKILL.en.md \
  .agents/skills/negative-first-mesh/SKILL.md
```

User scope:

```bash
mkdir -p "$HOME/.agents/skills/negative-first-mesh"
cp /path/to/negative-first-mesh-skill/SKILL.en.md \
  "$HOME/.agents/skills/negative-first-mesh/SKILL.md"
```

Claude Code uses `.claude/skills/`.

## 3. OpenAI Codex

### Repository Installation

```bash
mkdir -p .agents/skills/negative-first-mesh
cp SKILL.en.md .agents/skills/negative-first-mesh/SKILL.md
```

### User Installation

```bash
mkdir -p "$HOME/.agents/skills/negative-first-mesh"
cp SKILL.en.md "$HOME/.agents/skills/negative-first-mesh/SKILL.md"
```

### Administrator Installation

```bash
sudo mkdir -p /etc/codex/skills/negative-first-mesh
sudo cp SKILL.en.md /etc/codex/skills/negative-first-mesh/SKILL.md
```

### Verify and Invoke

```text
/skills
```

```text
$negative-first-mesh Verify this claim by eliminating contradictions,
out-of-scope evidence, and unsupported candidates first.
```

Codex may also invoke the skill automatically when the request matches its YAML `description`. Restart Codex if a newly installed skill does not appear.

## 4. Anthropic Claude Code

### Project Installation

```bash
mkdir -p .claude/skills/negative-first-mesh
cp SKILL.en.md .claude/skills/negative-first-mesh/SKILL.md
```

### Personal Installation

```bash
mkdir -p "$HOME/.claude/skills/negative-first-mesh"
cp SKILL.en.md "$HOME/.claude/skills/negative-first-mesh/SKILL.md"
```

Start Claude Code:

```bash
claude
```

Direct invocation:

```text
/negative-first-mesh
```

Existing skill-folder changes are normally detected live. Restart Claude Code if the top-level skill directory was created after the session began.

## 5. Claude Web and Desktop Upload

1. Open **Customize → Skills**.
2. Select `+`, then **Create skill**.
3. Select **Upload a skill**.
4. Upload `negative-first-mesh-en.zip`.
5. Enable the skill.

Custom skills require code execution. Organization sharing depends on the account and organization policy.

## 6. Google Gemini CLI

Gemini CLI discovers `.gemini/skills/` and the interoperable `.agents/skills/` alias.

### Workspace Copy

```bash
mkdir -p .gemini/skills/negative-first-mesh
cp SKILL.en.md .gemini/skills/negative-first-mesh/SKILL.md
```

### User Copy

```bash
mkdir -p "$HOME/.gemini/skills/negative-first-mesh"
cp SKILL.en.md "$HOME/.gemini/skills/negative-first-mesh/SKILL.md"
```

### Link a Local Folder

```bash
gemini skills link /absolute/path/to/negative-first-mesh --scope workspace
```

### Install from Git

```bash
gemini skills install https://github.com/example/negative-first-mesh.git \
  --scope workspace
```

### Verify

```text
/skills list
/skills reload
```

```bash
gemini skills list --all
```

Gemini CLI may request consent when a skill accesses its bundled directory.

## 7. Persistent Gemini Instructions with `GEMINI.md`

```markdown
# Project reasoning policy

For factual, diagnostic, comparison, recommendation, and retrieval questions:
1. Expand plausible candidates.
2. Eliminate contradicted, out-of-scope, and unsupported candidates first.
3. Distinguish false, not found in scope, and unknown.
4. Generate an answer only from surviving evidence.
```

Import another file when needed:

```markdown
@./negative-first-mesh-instructions.md
```

`GEMINI.md` is persistent context; an Agent Skill is activated selectively for relevant work.

## 8. LLMs Without Native Skill Discovery

Attach `SKILL.en.md` to the project and add:

```text
For fact checking, diagnosis, comparison, recommendation, and retrieval,
apply the Negative-First Mesh workflow from the attached SKILL.md.
Eliminate contradicted, out-of-scope, and unsupported candidates first.
Distinguish FALSE, NOT_FOUND_IN_SCOPE, and UNKNOWN.
```

For APIs, inject the skill body conditionally as a system/developer instruction only for relevant requests. This avoids adding the full token cost to every request.

Manual compatibility prompt:

```text
Do not generate the answer first.
Expand plausible candidates, then eliminate:
1. definition mismatches,
2. mandatory-condition failures,
3. time or scope mismatches,
4. unsupported claims,
5. strong contradictions.

Answer only from surviving evidence.
If none survives, distinguish FALSE, NOT_FOUND_IN_SCOPE, and UNKNOWN.
```

## 9. Language-Specific Packages

- `packages/negative-first-mesh-ko.zip`
- `packages/negative-first-mesh-en.zip`
- `packages/negative-first-mesh-ja.zip`
- `packages/negative-first-mesh-zh-CN.zip`

Each archive contains:

```text
negative-first-mesh/
├── SKILL.md
├── README.md
├── EXAMPLES.md
├── TESTS.md
└── INSTALL.md
```

## 10. Installation Test

```text
Verify whether a social-media screenshot about a policy change is true.
Do not invent evidence. Distinguish false, not found within the checked scope, and unknown.
```

## 11. Update and Removal

Codex:

```bash
rm -rf .agents/skills/negative-first-mesh
```

Claude Code:

```bash
rm -rf .claude/skills/negative-first-mesh
```

Gemini CLI:

```text
/skills reload
```

```bash
gemini skills uninstall negative-first-mesh --scope workspace
```

## 12. Notes

- A skill does not override platform safety policies or higher-priority instructions.
- “Negative-First” does not mean automatically answering negatively.
- Failure to find evidence must not be converted into falsity.
- Current questions require current official sources or web access.
- Without file, web, or database tools, the skill can judge only the supplied context.
- Product paths and commands may change; check official documentation.

## Official References

- OpenAI Codex skills: https://developers.openai.com/codex/build-skills
- Claude Code skills: https://code.claude.com/docs/en/skills
- Claude custom skills upload: https://support.claude.com/en/articles/12512180-use-skills-in-claude
- Gemini CLI Agent Skills: https://geminicli.com/docs/cli/skills/
- Gemini CLI skill management: https://geminicli.com/docs/cli/using-agent-skills/
- Gemini CLI `GEMINI.md`: https://geminicli.com/docs/cli/gemini-md/
