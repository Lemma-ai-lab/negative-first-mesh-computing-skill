# Negative-First Mesh 安装与应用指南

[한국어](INSTALL.ko.md) | [English](INSTALL.en.md) | [日本語](INSTALL.ja.md) | **[简体中文](INSTALL.zh-CN.md)**

验证日期：**2026-07-24**

本技能可以直接安装到支持 `SKILL.md` Agent Skills格式的Codex、Claude Code和Gemini CLI。对于不支持原生技能发现的LLM，应通过项目指令或系统/开发者提示词应用。

## 1. 选择语言

实际执行入口必须准确命名为 `SKILL.md`。

| 语言 | 源文件 |
|---|---|
| 韩文 | `SKILL.ko.md` |
| 英文 | `SKILL.en.md` |
| 日文 | `SKILL.ja.md` |
| 简体中文 | `SKILL.zh-CN.md` |

```bash
mkdir -p /path/to/target/negative-first-mesh
cp SKILL.zh-CN.md /path/to/target/negative-first-mesh/SKILL.md
```

## 2. Codex与Gemini CLI共用路径

```bash
mkdir -p .agents/skills/negative-first-mesh
cp SKILL.zh-CN.md .agents/skills/negative-first-mesh/SKILL.md
```

用户级：

```bash
mkdir -p "$HOME/.agents/skills/negative-first-mesh"
cp SKILL.zh-CN.md "$HOME/.agents/skills/negative-first-mesh/SKILL.md"
```

Claude Code使用 `.claude/skills/`。

## 3. OpenAI Codex

### 项目安装

```bash
mkdir -p .agents/skills/negative-first-mesh
cp SKILL.zh-CN.md .agents/skills/negative-first-mesh/SKILL.md
```

### 用户安装

```bash
mkdir -p "$HOME/.agents/skills/negative-first-mesh"
cp SKILL.zh-CN.md "$HOME/.agents/skills/negative-first-mesh/SKILL.md"
```

### 管理员安装

```bash
sudo mkdir -p /etc/codex/skills/negative-first-mesh
sudo cp SKILL.zh-CN.md /etc/codex/skills/negative-first-mesh/SKILL.md
```

验证：

```text
/skills
```

显式调用：

```text
$negative-first-mesh 先排除矛盾、超出范围和缺乏证据的候选项，再核实这个主张。
```

新技能未显示时重启Codex。

## 4. Anthropic Claude Code

### 项目安装

```bash
mkdir -p .claude/skills/negative-first-mesh
cp SKILL.zh-CN.md .claude/skills/negative-first-mesh/SKILL.md
```

### 个人安装

```bash
mkdir -p "$HOME/.claude/skills/negative-first-mesh"
cp SKILL.zh-CN.md "$HOME/.claude/skills/negative-first-mesh/SKILL.md"
```

```bash
claude
```

直接调用：

```text
/negative-first-mesh
```

现有技能目录的修改通常可实时检测；如果会话开始后才创建顶层目录，可能需要重启。

## 5. 上传到Claude网页或桌面应用

1. 打开 **Customize → Skills**。
2. 点击 `+`，选择 **Create skill**。
3. 选择 **Upload a skill**。
4. 上传 `negative-first-mesh-zh-CN.zip`。
5. 启用技能。

自定义技能需要启用代码执行功能。

## 6. Google Gemini CLI

### 工作区安装

```bash
mkdir -p .gemini/skills/negative-first-mesh
cp SKILL.zh-CN.md .gemini/skills/negative-first-mesh/SKILL.md
```

与Codex共用时使用 `.agents/skills/`。

### 用户安装

```bash
mkdir -p "$HOME/.gemini/skills/negative-first-mesh"
cp SKILL.zh-CN.md "$HOME/.gemini/skills/negative-first-mesh/SKILL.md"
```

### 链接本地目录

```bash
gemini skills link /absolute/path/to/negative-first-mesh --scope workspace
```

### 从Git安装

```bash
gemini skills install https://github.com/example/negative-first-mesh.git \
  --scope workspace
```

### 验证

```text
/skills list
/skills reload
```

```bash
gemini skills list --all
```

## 7. 使用 `GEMINI.md` 持续应用

```markdown
# Project reasoning policy

For factual, diagnostic, comparison, recommendation, and retrieval questions:
1. Expand plausible candidates.
2. Eliminate contradicted, out-of-scope, and unsupported candidates first.
3. Distinguish false, not found in scope, and unknown.
4. Generate an answer only from surviving evidence.
```

导入单独文件：

```markdown
@./negative-first-mesh-instructions.md
```

## 8. 不支持原生技能的LLM

将 `SKILL.zh-CN.md` 上传为项目文件，并加入：

```text
对于事实核验、诊断、比较、推荐和信息检索问题，
应用附件SKILL.md中的Negative-First Mesh流程。
优先排除矛盾、超出范围和缺乏证据的候选项，
并区分FALSE、NOT_FOUND_IN_SCOPE和UNKNOWN。
```

API集成时，只在相关请求中把技能正文加入系统或开发者提示词。

## 9. 语言版发布ZIP

- `packages/negative-first-mesh-ko.zip`
- `packages/negative-first-mesh-en.zip`
- `packages/negative-first-mesh-ja.zip`
- `packages/negative-first-mesh-zh-CN.zip`

每个ZIP：

```text
negative-first-mesh/
├── SKILL.md
├── README.md
├── EXAMPLES.md
├── TESTS.md
└── INSTALL.md
```

## 10. 安装验证

```text
核实社交媒体截图中的政策变化是否属实。
不要编造证据，并区分错误、在验证范围内未找到和无法判断。
```

## 11. 更新与删除

```bash
rm -rf .agents/skills/negative-first-mesh
rm -rf .claude/skills/negative-first-mesh
```

Gemini CLI：

```text
/skills reload
```

```bash
gemini skills uninstall negative-first-mesh --scope workspace
```

## 12. 注意事项

- 技能不能覆盖平台安全政策或更高优先级指令。
- Negative-First并不意味着总是否定回答。
- 未找到证据不能转换为错误。
- 最新问题仍需要最新官方资料或网页访问。
- 产品更新可能改变安装路径和命令。

## 公式文書 / 官方文档

- OpenAI Codex: https://developers.openai.com/codex/build-skills
- Claude Code: https://code.claude.com/docs/en/skills
- Claude custom skill upload: https://support.claude.com/en/articles/12512180-use-skills-in-claude
- Gemini CLI Agent Skills: https://geminicli.com/docs/cli/skills/
- Gemini CLI skill management: https://geminicli.com/docs/cli/using-agent-skills/
- Gemini CLI `GEMINI.md`: https://geminicli.com/docs/cli/gemini-md/
