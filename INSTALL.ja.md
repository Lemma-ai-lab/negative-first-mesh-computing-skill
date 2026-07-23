# Negative-First Mesh インストール・適用ガイド

[한국어](INSTALL.ko.md) | [English](INSTALL.en.md) | **[日本語](INSTALL.ja.md)** | [简体中文](INSTALL.zh-CN.md)

確認基準日: **2026-07-24**

このスキルは `SKILL.md` 形式をサポートするCodex、Claude Code、Gemini CLIへ直接インストールできる。ネイティブスキル非対応LLMでは、プロジェクト指示またはシステム・開発者プロンプトとして適用する。

## 1. 言語を選択する

実行ファイル名は正確に `SKILL.md` とする。

| 言語 | 元ファイル |
|---|---|
| 韓国語 | `SKILL.ko.md` |
| 英語 | `SKILL.en.md` |
| 日本語 | `SKILL.ja.md` |
| 中国語簡体字 | `SKILL.zh-CN.md` |

```bash
mkdir -p /path/to/target/negative-first-mesh
cp SKILL.ja.md /path/to/target/negative-first-mesh/SKILL.md
```

## 2. CodexとGemini CLIの共通パス

```bash
mkdir -p .agents/skills/negative-first-mesh
cp SKILL.ja.md .agents/skills/negative-first-mesh/SKILL.md
```

ユーザー全体:

```bash
mkdir -p "$HOME/.agents/skills/negative-first-mesh"
cp SKILL.ja.md "$HOME/.agents/skills/negative-first-mesh/SKILL.md"
```

Claude Codeは `.claude/skills/` を使用する。

## 3. OpenAI Codex

### プロジェクト

```bash
mkdir -p .agents/skills/negative-first-mesh
cp SKILL.ja.md .agents/skills/negative-first-mesh/SKILL.md
```

### ユーザー全体

```bash
mkdir -p "$HOME/.agents/skills/negative-first-mesh"
cp SKILL.ja.md "$HOME/.agents/skills/negative-first-mesh/SKILL.md"
```

### 管理者

```bash
sudo mkdir -p /etc/codex/skills/negative-first-mesh
sudo cp SKILL.ja.md /etc/codex/skills/negative-first-mesh/SKILL.md
```

確認と呼び出し:

```text
/skills
```

```text
$negative-first-mesh この主張を、矛盾・範囲外・根拠不足の候補から先に除外して検証して。
```

新しいスキルが表示されない場合はCodexを再起動する。

## 4. Anthropic Claude Code

### プロジェクト

```bash
mkdir -p .claude/skills/negative-first-mesh
cp SKILL.ja.md .claude/skills/negative-first-mesh/SKILL.md
```

### 個人

```bash
mkdir -p "$HOME/.claude/skills/negative-first-mesh"
cp SKILL.ja.md "$HOME/.claude/skills/negative-first-mesh/SKILL.md"
```

```bash
claude
```

直接呼び出し:

```text
/negative-first-mesh
```

既存スキルフォルダの変更は通常セッション中に検出される。セッション開始後に最上位フォルダを初作成した場合は再起動する。

## 5. Claude Web・デスクトップへのアップロード

1. **Customize → Skills**を開く。
2. `+` → **Create skill**を選ぶ。
3. **Upload a skill**を選ぶ。
4. `negative-first-mesh-ja.zip`をアップロードする。
5. スキルを有効化する。

カスタムスキルにはコード実行機能が必要である。

## 6. Google Gemini CLI

### ワークスペース

```bash
mkdir -p .gemini/skills/negative-first-mesh
cp SKILL.ja.md .gemini/skills/negative-first-mesh/SKILL.md
```

Codexと共有する場合は `.agents/skills/` を使用する。

### ユーザー全体

```bash
mkdir -p "$HOME/.gemini/skills/negative-first-mesh"
cp SKILL.ja.md "$HOME/.gemini/skills/negative-first-mesh/SKILL.md"
```

### ローカルフォルダをリンク

```bash
gemini skills link /absolute/path/to/negative-first-mesh --scope workspace
```

### Gitからインストール

```bash
gemini skills install https://github.com/example/negative-first-mesh.git \
  --scope workspace
```

### 確認

```text
/skills list
/skills reload
```

```bash
gemini skills list --all
```

## 7. `GEMINI.md`で常時適用する

```markdown
# Project reasoning policy

For factual, diagnostic, comparison, recommendation, and retrieval questions:
1. Expand plausible candidates.
2. Eliminate contradicted, out-of-scope, and unsupported candidates first.
3. Distinguish false, not found in scope, and unknown.
4. Generate an answer only from surviving evidence.
```

別ファイルを読み込む場合:

```markdown
@./negative-first-mesh-instructions.md
```

## 8. ネイティブスキル非対応LLM

`SKILL.ja.md`をプロジェクトへ添付し、次を追加する。

```text
事実確認、診断、比較、推薦、検索の質問では、
添付したSKILL.mdのNegative-First Mesh手順を適用する。
矛盾、範囲外、根拠不足の候補を先に除外し、
FALSE / NOT_FOUND_IN_SCOPE / UNKNOWNを区別する。
```

APIでは関連質問にだけスキル本文をシステムまたは開発者指示として挿入する。

## 9. 言語別配布ZIP

- `packages/negative-first-mesh-ko.zip`
- `packages/negative-first-mesh-en.zip`
- `packages/negative-first-mesh-ja.zip`
- `packages/negative-first-mesh-zh-CN.zip`

各ZIP:

```text
negative-first-mesh/
├── SKILL.md
├── README.md
├── EXAMPLES.md
├── TESTS.md
└── INSTALL.md
```

## 10. 動作確認

```text
SNS画像の政策変更が事実か確認して。
根拠を作らず、偽・確認範囲内で未発見・判断不能を区別して。
```

## 11. 更新と削除

```bash
rm -rf .agents/skills/negative-first-mesh
rm -rf .claude/skills/negative-first-mesh
```

Gemini CLI:

```text
/skills reload
```

```bash
gemini skills uninstall negative-first-mesh --scope workspace
```

## 12. 注意事項

- スキルは上位の安全ポリシーを上書きしない。
- Negative-Firstは常に否定回答をする方式ではない。
- 見つからないことを偽へ変換しない。
- 最新質問には最新の公式資料またはウェブアクセスが必要である。
- 製品更新によりパスやコマンドが変わる可能性がある。

## 公式文書 / 官方文档

- OpenAI Codex: https://developers.openai.com/codex/build-skills
- Claude Code: https://code.claude.com/docs/en/skills
- Claude custom skill upload: https://support.claude.com/en/articles/12512180-use-skills-in-claude
- Gemini CLI Agent Skills: https://geminicli.com/docs/cli/skills/
- Gemini CLI skill management: https://geminicli.com/docs/cli/using-agent-skills/
- Gemini CLI `GEMINI.md`: https://geminicli.com/docs/cli/gemini-md/
