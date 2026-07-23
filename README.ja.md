# Negative-First Mesh Computing

[한국어](README.ko.md) | [English](README.en.md) | **日本語** | [简体中文](README.zh-CN.md)

## 「否定先行メッシュ推論」の概念概要とシステム設計

`Negative-First Mesh Computing` は、質問を受け取った直後に最もそれらしい答えを生成するのではなく、考えられる候補をメッシュ状に同時活性化し、**明らかに該当しない候補から先にOFFにする**ことで答えを導く計算・推論アーキテクチャである。

中心原則は次のとおりである。

> 正解を候補ごとに順番に探さない。  
> 候補全体を同時に起動し、違うものを先に消し、残ったものだけで答える。

一般的な検索システムや生成AIは、候補を順次探索するか、現在の文脈で最も確率の高い出力を連続して生成する。これに対して Negative-First Mesh は、質問を複数の条件信号に分解し、それらを候補メッシュ全体へ伝播させた後、不一致・矛盾・範囲外・根拠不足の候補を先に除外する。

最終回答に使えるのは、生き残った候補だけである。候補が一つも残らない場合、内容を作り出してはならず、次のいずれかに分類する。

- 偽である
- 定義された範囲内では見つからない
- 根拠不足のため判断できない
- 質問自体の条件が矛盾している
- 次の候補グループを探索する必要がある

---

# 1. 問題定義

現在のAIと従来型コンピュータは、データ量の多い探索・推論において複数の限界を持つ。

## 1.1 順次探索のコスト

データが増えるほど、候補を一件ずつ比較したり、検索・スコアリング・並べ替え・再検証を繰り返したりする必要がある。

```text
候補1を確認 → 除外
候補2を確認 → 除外
候補3を確認 → 維持
...
候補Nを確認
```

GPUやベクトル検索は多くの比較を並列化するが、論理的には候補を読み込み、点数を計算し、順位付けし、結果を再評価する処理が残る。

## 1.2 生成先行AIのハルシネーション

生成AIは、直接的な根拠が十分でなくても、言語として自然な続きを作ることができる。

```text
質問
  ↓
可能性の高い文章を生成
  ↓
根拠の空白を確率的に補完
  ↓
もっともらしいが誤った回答
```

## 1.3 「見つからない」と「分からない」の混同

次の三つは同じ意味ではない。

- 実際に存在しない。
- 指定されたデータ範囲では見つからなかった。
- 現在利用できる資料からは判断できない。

多くのシステムはこの違いを保持できず、一度の検索失敗を全体的な不存在へ拡大してしまう。

## 1.4 データと演算の分離

多くのコンピュータでは、比較や推論の前にデータをメモリから演算装置へ移動する。AIシステムが大きくなるほど、演算量だけでなく、メモリ移動とデータ探索もボトルネックになる。

Negative-First Mesh の長期目標は、**データが保存されている場所、またはその直近で条件比較と候補抑制が同時に起きる構造**を実現することである。

---

# 2. 中核概念

## 2.1 候補メッシュ

情報を一つのファイルやアドレスだけで保持するのではなく、複数の意味と関係を持つノードおよびリンクとして表現する。

例えば `リンゴ` という情報は次のような関係を持つ。

```text
リンゴ
├─ 分類: 果物
├─ 色: 赤、緑
├─ 形: 丸い
├─ 機能: 食べられる
├─ 関係: 木になる
└─ 属性: 甘い、酸っぱい
```

質問が入力されると、文章全体をそのまま検索するのではなく、質問を構成する各条件が関連メッシュへ伝播する。

```text
質問: 「食べられる赤い果物」

食べられる ─┐
赤い ───────┼─ リンゴノードが活性化
果物 ───────┘
```

## 2.2 否定先行抑制

各候補は、最初は暫定的な活性状態から始まる。必須条件に明確に違反した候補は直ちにOFFにする。

```text
候補A: 果物ではない       → OFF
候補B: 食べられない       → OFF
候補C: 赤くない           → OFF
候補D: すべての条件を満たす → ON
```

必須条件の一つでも失敗した場合、その候補を有効な答えとして扱うための追加計算は行わない。

## 2.3 ゼロ検出

活性候補が一つも残らない状態を、独立した計算結果として扱う。

```text
活性候補数 > 0
→ 生存候補を検証

活性候補数 = 0
→ 回答生成を禁止
→ 不在・偽・不確実・範囲不足を分類
```

このアーキテクチャでは `0` は単なる失敗ではない。次の探索グループへ移動するか、正確な不確実性状態を返すための制御信号である。

## 2.4 グループ遷移

完全一致グループの候補がゼロの場合、探索範囲を制御された順序で拡張する。

```text
完全一致表現
  ↓ 0
表記・綴りの変形
  ↓ 0
同義語・略語
  ↓ 0
上位概念・下位概念
  ↓ 0
関係・原因・結果
  ↓ 0
判断不能、または範囲内で未発見
```

探索を拡張するほど元の質問との意味距離が大きくなるため、各候補に距離ペナルティを付与する。

## 2.5 根拠制約付き回答

最終回答は、十分な根拠を持つ候補からのみ構成する。

```text
ACTIVE_SUPPORTED
→ 回答に直接使用可能

ACTIVE_TENTATIVE
→ 可能性または推定としてのみ表示

OFF_UNSUPPORTED
→ 回答から除外

UNKNOWN_UNRESOLVED
→ 判断不能として表示
```

---

# 3. 既存技術との関係

Negative-First Mesh は一つの独立技術だけではなく、複数の既存概念を一つのアーキテクチャへ統合する。

## 3.1 内容アドレスメモリ

アドレスを指定してデータを読むのではなく、検索したい内容を入力し、多数の保存項目と同時比較する。

Negative-First Mesh はこの比較を、完全一致キーだけでなく、意味・関係・条件・時間・情報源の信頼性まで拡張する。

## 3.2 連想記憶

部分的な手掛かりから完全なパターンを復元する。

```text
部分的な質問
→ 関連ノードが同時活性化
→ 候補が競合し相互抑制
→ 安定したパターンへ収束
```

## 3.3 ニューロモーフィック計算

ニューロンとシナプスを参考にした分散処理要素がイベントへ反応し、必要な領域だけを活性化する。

## 3.4 インメモリコンピューティング

メモリ内部または近傍で比較・積算を行い、データ移動コストを減らす。

## 3.5 波動・干渉計算

電気信号、光信号、位相、周波数、到達時間差を利用し、複数候補の強め合いと打ち消し合いを並列処理する。

## 3.6 グラフ推論

エンティティ、属性、原因、結果、情報源、時間関係をノードとエッジで表現する。

Negative-First Mesh の特徴は、これらを次の順序で統合することにある。

```text
候補領域全体を活性化
→ 否定条件を先に伝播
→ 無効候補を抑制
→ ゼロを検出
→ 次のグループへ遷移
→ 根拠のある生存候補だけを生成AIへ渡す
```

---

# 4. 目標

## 4.1 第1段階目標

ソフトウェアで次の機能を実装する。

- 質問条件の自動分解
- 候補グループ生成
- 複数データソースの同時検索
- 不一致候補の早期除外
- 不在・偽・判断不能の分離
- 根拠ベースの回答生成
- ハルシネーション抑制
- 根拠および候補除外ログの追跡

## 4.2 第2段階目標

ベクトルDB、グラフDB、検索エンジン、ルールエンジンを統合し、大規模並列連想検索を実装する。

## 4.3 長期目標

メモリスタ、FPGA、光回路、ニューロモーフィックチップ、多層波動ネットワークなどの物理基盤へ候補活性化と抑制処理を移す。

---

# 5. 全体アーキテクチャ

```text
┌──────────────────────────────┐
│          ユーザー質問         │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 1. Question Normalizer       │
│ - 対象・主張・条件の分解       │
│ - 範囲・時点・危険度・出力型    │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 2. Mesh Planner              │
│ - 候補グループ生成            │
│ - 検索軸・関係軸生成           │
│ - 並列探索計画                │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 3. Candidate Activator       │
│ - 完全一致候補                │
│ - 類似候補                   │
│ - 関係候補                   │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 4. Negative Gate Engine      │
│ - 定義不一致                  │
│ - 必須条件違反                │
│ - 時点不一致                  │
│ - 矛盾                       │
│ - 根拠不足                   │
│ - 安全基準未達                │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 5. Zero Detector             │
│ - 活性候補数を計算            │
│ - ゼロ状態を分類              │
│ - 次グループを選択            │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 6. Evidence Resolver         │
│ - 情報源信頼度                │
│ - 最新性                     │
│ - 直接性                     │
│ - 候補間矛盾                  │
│ - 意味距離                   │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 7. Answer Composer           │
│ - 生存候補のみ使用            │
│ - 事実・推定・不明を分離       │
│ - 範囲と限界を表示            │
└──────────────────────────────┘
```

---

# 6. 主要コンポーネント

## 6.1 Question Normalizer

生の質問を構造化フレームへ変換する。

```json
{
  "target": "探索または判定する対象",
  "claim": "検証する主張",
  "required_conditions": [],
  "excluded_conditions": [],
  "scope": {
    "sources": [],
    "time_range": null,
    "region": null,
    "closed_world": false
  },
  "freshness_required": false,
  "risk_level": "LOW | MEDIUM | HIGH",
  "answer_type": "FACT | SEARCH | DIAGNOSIS | DESIGN | RECOMMENDATION"
}
```

### 役割

- 曖昧な対象を分離
- 否定条件と必須条件を抽出
- 探索範囲が閉世界か判定
- 最新確認が必要か判定
- 誤回答の危険度を設定

---

## 6.2 Mesh Planner

候補を探索する軸と順序を定義する。

### 標準候補グループ

1. 完全一致
2. 表記・綴りの変形
3. 同義語・略語
4. 意味類似
5. 上位概念
6. 下位概念
7. 隣接時間範囲
8. 隣接地域
9. 原因・結果
10. 反対概念
11. 情報源間クロスチェック
12. ユーザー前提の代替解釈

### グループ定義例

```json
{
  "group_id": "G03",
  "group_type": "SYNONYM",
  "distance": 0.15,
  "activation_policy": "ON_PREVIOUS_ZERO",
  "queries": [
    "オフライン利用",
    "インターネット接続なしで実行",
    "初回認証後のローカル利用"
  ]
}
```

---

## 6.3 Candidate Activator

検索結果、ルール結果、グラフ経路、モデル推定を候補ノードへ変換する。

```json
{
  "candidate_id": "C1024",
  "label": "候補説明",
  "group_id": "G03",
  "state": "ACTIVE_TENTATIVE",
  "matched_conditions": [],
  "failed_conditions": [],
  "evidence_ids": [],
  "distance": 0.15
}
```

すべての候補を物理的に同時処理できない環境でも、同一グループの候補は同じゲート順序でバッチ処理する。

---

## 6.4 Negative Gate Engine

Negative-First Mesh の中核コンポーネントである。

### 定義ゲート

候補と質問対象の定義が一致するかを確認する。

```text
質問: 定期購読のない買い切りアプリ
候補: 毎月の支払いが必要なサブスクリプションサービス

定義不一致 → OFF_FALSE
```

### 条件ゲート

必須条件を一つでも満たさない候補をOFFにする。

```text
必須:
- 1,000ドル以下
- メモリ16GB以上
- 定期購読なし

候補が月額契約専用 → OFF_FALSE
```

### 時間ゲート

資料の時点が質問の時点に適合するか確認する。

```text
2024年の発売価格
現在の販売価格を質問
→ OFF_OUT_OF_SCOPE
```

### 存在ゲート

検証範囲が閉じている場合、存在の有無を判定する。

```text
アップロードされた説明書3件を全件検索
オフライン利用の記述なし
→ NOT_FOUND_IN_SCOPE
```

開世界で検索に失敗しても、不存在の証明として扱わない。

```text
ウェブ検索で発見できない
→ UNKNOWN_UNRESOLVED
```

### 矛盾ゲート

より強い根拠または内部矛盾が存在する候補を除外する。

### 根拠ゲート

回答に含めるだけの直接根拠がない候補を `OFF_UNSUPPORTED` へ移す。

### 因果ゲート

相関関係を検証済みの原因として断定しない。

### 安全ゲート

医療・法律・金融・セキュリティなど高リスク領域では根拠しきい値を高くする。

---

# 7. 候補状態モデル

| 状態 | 説明 |
|---|---|
| `ACTIVE_SUPPORTED` | 強い根拠により候補を維持 |
| `ACTIVE_TENTATIVE` | 可能性はあるが追加検証が必要 |
| `OFF_FALSE` | 質問条件と明確に不一致 |
| `OFF_CONTRADICTED` | より強い根拠または矛盾により除外 |
| `OFF_OUT_OF_SCOPE` | 現在の時間・地域・資料範囲外 |
| `OFF_UNSUPPORTED` | 主張するための根拠が不足 |
| `UNKNOWN_UNRESOLVED` | 真偽を判定できる資料が不足 |
| `NOT_FOUND_IN_SCOPE` | 定義された閉じた範囲で未発見 |

重要な規則:

```text
根拠がない ≠ 偽
見つからない ≠ 存在しない
可能性が低い ≠ 不可能
```

---

# 8. ゼロ検出とグループ遷移

## 8.1 ゼロ検出条件

```text
ACTIVE_SUPPORTED 件数 = 0
AND
ACTIVE_TENTATIVE 件数 = 0
```

## 8.2 ゼロ状態の分類

```text
質問条件同士が矛盾
→ CONTRADICTED_QUERY

閉じたデータ集合を全件確認
→ NOT_FOUND_IN_SCOPE

強い反証により主張が不成立
→ FALSE

利用可能な情報源が不足
→ UNKNOWN_UNRESOLVED

次の関連候補グループが存在
→ EXPAND_NEXT_GROUP
```

## 8.3 グループ遷移方針

```text
G1 完全一致
  ↓ 0
G2 表記変形
  ↓ 0
G3 同義語
  ↓ 0
G4 意味類似
  ↓ 0
G5 上位・下位概念
  ↓ 0
G6 関係・原因・結果
  ↓ 0
最終ゼロ状態判定
```

各グループは元の質問からの意味距離を持つ。

```text
distance = 0.00  完全一致
distance = 0.10  表記変形
distance = 0.20  同義語
distance = 0.35  意味類似
distance = 0.50  上位・下位概念
distance = 0.70  隣接関係
```

距離が大きいほど、最終回答では確定度の低い表現を用いる。

---

# 9. 信頼度計算

候補の信頼度は次の要素で評価できる。

- `D`: 直接根拠の強さ
- `S`: 情報源の信頼性
- `C`: 条件充足度
- `R`: 最新性
- `X`: 矛盾ペナルティ
- `G`: 意味距離ペナルティ

```text
confidence =
D × S × C × R × (1 - X) × (1 - G)
```

推奨判定:

| スコア | 判定 |
|---:|---|
| 0.85以上 | 確認済み |
| 0.65～0.84 | 可能性が高い |
| 0.40～0.64 | 不確実 |
| 0.40未満 | 回答候補から除外、または推測としてのみ表示 |

この値は数学的に保証された確率ではなく、内部比較と優先順位付けのための指標である。

---

# 10. 回答生成ポリシー

## 10.1 標準回答構造

```text
[直接回答]
最も強い根拠を持つ結論

[判定]
確認済み / 可能性が高い / 不確実 / 範囲内で未発見 / 判断不能 / 偽

[検証範囲]
確認した資料・時点・地域

[先に除外された主要候補]
重要な除外候補と理由

[残る不確実性]
まだ確認できていない部分
```

一般ユーザー向け回答では、内部状態名をそのまま見せず、自然な表現へ変換する。

## 10.2 確定回答を禁止する条件

次の場合は確定回答を生成しない。

- すべての候補が `OFF_UNSUPPORTED`
- 生存候補同士が強く矛盾する
- 最新確認が必要だが資料を取得できない
- 閉じた範囲か開いた範囲か不明
- 中核条件の曖昧さが結果を大きく変える
- 高リスク問題で根拠しきい値を満たさない

---

# 11. ソフトウェア実装設計

## 11.1 推奨構成

```text
Frontend
- 質問入力
- 検証範囲選択
- 候補メッシュ可視化
- 除外理由表示
- 最終回答・根拠表示

API
- 質問正規化
- 候補グループ生成
- 検索・ツールオーケストレーション
- Negative Gate実行
- ゼロ検出
- 回答生成

Storage
- PostgreSQL: 質問、候補、状態、根拠、実行ログ
- Redis: 活性候補、グループキュー、一時スコア
- Vector DB: 意味類似検索
- Graph DBまたはPostgreSQLグラフモデル: 関係探索
- Object Storage: 原文、添付、根拠スナップショット

Workers
- exact-search-worker
- semantic-search-worker
- graph-search-worker
- contradiction-worker
- evidence-verification-worker
- freshness-check-worker
```

---

# 12. データモデル

## 12.1 questions

```sql
CREATE TABLE questions (
    id BIGSERIAL PRIMARY KEY,
    raw_question TEXT NOT NULL,
    normalized_target TEXT,
    normalized_claim TEXT,
    answer_type VARCHAR(32),
    risk_level VARCHAR(16),
    closed_world BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 12.2 candidate_groups

```sql
CREATE TABLE candidate_groups (
    id BIGSERIAL PRIMARY KEY,
    question_id BIGINT NOT NULL REFERENCES questions(id),
    group_type VARCHAR(32) NOT NULL,
    semantic_distance NUMERIC(6,5) NOT NULL DEFAULT 0,
    sequence_no INTEGER NOT NULL,
    activated_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ
);
```

## 12.3 candidates

```sql
CREATE TABLE candidates (
    id BIGSERIAL PRIMARY KEY,
    group_id BIGINT NOT NULL REFERENCES candidate_groups(id),
    label TEXT NOT NULL,
    state VARCHAR(32) NOT NULL,
    confidence NUMERIC(6,5),
    matched_conditions JSONB NOT NULL DEFAULT '[]',
    failed_conditions JSONB NOT NULL DEFAULT '[]',
    disabled_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 12.4 evidence

```sql
CREATE TABLE evidence (
    id BIGSERIAL PRIMARY KEY,
    candidate_id BIGINT NOT NULL REFERENCES candidates(id),
    source_type VARCHAR(32) NOT NULL,
    source_uri TEXT,
    source_title TEXT,
    published_at TIMESTAMPTZ,
    retrieved_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    directness NUMERIC(6,5),
    reliability NUMERIC(6,5),
    freshness NUMERIC(6,5),
    excerpt TEXT
);
```

## 12.5 gate_results

```sql
CREATE TABLE gate_results (
    id BIGSERIAL PRIMARY KEY,
    candidate_id BIGINT NOT NULL REFERENCES candidates(id),
    gate_type VARCHAR(32) NOT NULL,
    passed BOOLEAN NOT NULL,
    resulting_state VARCHAR(32),
    reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

# 13. 中核API

## 質問実行

```http
POST /api/mesh/questions
Content-Type: application/json
```

```json
{
  "question": "アップロードされた説明書にオフライン利用機能が明記されていますか？",
  "scope": {
    "document_ids": [101, 102, 103],
    "closed_world": true
  },
  "mode": "negative_first"
}
```

## 実行結果

```json
{
  "status": "CONFIRMED",
  "answer": "初回認証後にオフライン利用できるという記述が確認されました。",
  "scope": {
    "document_ids": [101, 102, 103],
    "closed_world": true
  },
  "active_candidates": [
    {
      "candidate": "初回有効化後はインターネット接続なしで実行できる。",
      "confidence": 0.93
    }
  ],
  "disabled_candidates": [
    {
      "candidate": "クラウド同期機能",
      "state": "OFF_FALSE",
      "reason": "オフライン実行が可能であることを証明しない。"
    }
  ],
  "uncertainties": []
}
```

---

# 14. 中核アルゴリズム

```text
function answer_with_negative_first_mesh(question, sources):

    frame = normalize_question(question)
    groups = build_candidate_groups(frame)

    for group in groups:

        candidates = activate_candidates(group, sources)

        parallel_for candidate in candidates:

            if not definition_matches(candidate, frame):
                disable(candidate, OFF_FALSE)
                continue

            if violates_required_condition(candidate, frame):
                disable(candidate, OFF_FALSE)
                continue

            if outside_time_or_scope(candidate, frame):
                disable(candidate, OFF_OUT_OF_SCOPE)
                continue

            if contradicted(candidate):
                disable(candidate, OFF_CONTRADICTED)
                continue

            if lacks_required_evidence(candidate):
                disable(candidate, OFF_UNSUPPORTED)
                continue

            candidate.state = score_candidate(candidate)

        survivors = get_survivors(candidates)

        if survivors is not empty:
            verified = cross_check(survivors)

            if verified is not empty:
                return compose_answer_from_verified_only(verified)

        zero_state = classify_zero_state(candidates, frame)

        if zero_state == EXPAND_NEXT_GROUP:
            continue

        return report_zero_state(zero_state)

    return {
        "status": "UNKNOWN",
        "answer": "現在確認可能な範囲では、判断に必要な根拠が不足しています。"
    }
```

---

# 15. AIモデルとの統合

Negative-First Mesh は生成AIを排除するための構造ではない。生成AIの役割を縮小し、根拠のある結果を説明する段階に限定する。

## 従来構造

```text
質問
→ 大規模言語モデル
→ 検索または確率的推論
→ 回答生成
```

## 提案構造

```text
質問
→ 条件分解
→ 候補メッシュ全体を活性化
→ 該当しない候補を先に除外
→ 根拠検証
→ 生存候補だけを小規模または大規模言語モデルへ渡す
→ 説明生成
```

### 期待効果

- ハルシネーション減少
- 不要なトークン使用削減
- 検索範囲縮小
- 回答根拠の追跡
- 「分からない」の判定向上
- 閉じた企業データでの正確な不存在確認
- 複数AIモデル間のクロスチェック

---

# 16. 物理メッシュへの拡張

ソフトウェア実装が安定した後、一部の演算を物理メッシュへ移行できる。

## 16.1 使用可能な基盤

- メモリスタ・クロスバー
- FPGA並列比較ネットワーク
- 光干渉器
- 超伝導回路
- スピン波素子
- ニューロモーフィックチップ
- 多層PCB伝送ネットワーク
- 周波数・位相多重回路

## 16.2 物理ノードの役割

```text
入力信号
→ 複数経路へ同時伝播
→ 条件違反経路を打ち消す
→ 条件一致経路を強める
→ しきい値以上のノードだけ維持
→ 出力で活性パターンを測定
```

## 16.3 仮想多層化

物理基板を無制限に積層する代わりに、一つの物理経路へ複数の論理層を重ねる。

- 異なる周波数
- 異なる位相
- 異なるタイムスロット
- 異なる光波長
- 異なる偏光
- 異なる符号パターン
- 異なる方向

これにより、限られた物理資源からより大きな論理状態空間を構成できる。

---

# 17. 性能目標

## ソフトウェア第1段階

- 100万候補に対する完全条件の並列除外
- 候補グループ単位のバッチ検索
- 1秒以内の初回ゼロ状態判定
- 根拠追跡率100%
- 不在・不明の誤判定テストを通過

## ソフトウェア第2段階

- ベクトル・グラフ・ルールの統合処理
- 複数データソースの同時検索
- 数千万候補の分散処理
- 活性候補のリアルタイム可視化
- LLM呼び出し前に候補の90%以上を除外

## ハードウェア実験段階

- 8×8または16×16メッシュ
- 同位相の強め合いを検証
- 逆位相の打ち消しを検証
- 複数入力のしきい値判定を検証
- 候補ゼロ状態を電気的に検出
- FPGA制御とグループ遷移ロジックを接続

---

# 18. テスト基準

## 事実性

- 根拠不足を偽と判断していないか
- 古い情報を現在の事実として使っていないか
- 相関関係を因果関係として断定していないか

## 範囲

- 閉じた範囲でのみ不存在を確定しているか
- 確認していない外部範囲を明示しているか
- 意味的に遠い候補を完全一致の答えに偽装していないか

## ゼロ処理

- 候補がないとき内容を生成していないか
- 合理的な次グループへ拡張しているか
- `偽`、`範囲内で未発見`、`判断不能`を区別しているか

## 回答品質

- 結論が最初に示されるか
- 主要な除外理由が簡潔に説明されるか
- 不要な内部推論を露出していないか
- 根拠と不確実性が同時に示されるか

---

# 19. 限界

## 19.1 真の無限状態は不可能

有限の装置に無限個の独立状態を保存することはできない。ただし、周波数・位相・時間・空間・接続の組み合わせから非常に大きな状態空間を構築できる。

## 19.2 物理資源のコスト

完全並列処理は計算時間を短縮する代わりに、より多くのメモリ、接続、電力、チップ面積を必要とする。

## 19.3 完全結合グラフの拡張問題

ノード数が増えるほど接続数は急増する。実用システムでは次のような階層構造が必要である。

- 局所メッシュ
- 分野別メッシュ
- 上位要約メッシュ
- 疎結合
- 必要時のみ活性化する経路
- 波長・周波数多重による仮想接続

## 19.4 意味比較の不確実性

完全一致キーとは異なり、意味比較は確率的である。意味的に近い候補は、検証されるまで暫定候補として扱う必要がある。

## 19.5 不存在証明の難しさ

開世界では、見つからなかったことは存在しないことの証明ではない。Negative-First Mesh はこの限界を隠さず、結果状態として明示する。

---

# 20. 開発フェーズ

## Phase 1. スキルおよびルールエンジン

- 質問正規化
- 候補状態モデル
- Negative Gate
- ゼロ検出
- グループ遷移
- 回答テンプレート
- テストシナリオ

## Phase 2. 検索オーケストレータ

- 完全一致検索
- 意味検索
- グラフ検索
- ファイル検索
- ウェブ検索
- データベース検索
- クロスチェック

## Phase 3. 運用プラットフォーム

- 質問実行履歴
- 候補メッシュ可視化
- ゲート通過・除外理由
- 根拠管理
- 管理者設定可能なしきい値
- 分野別ポリシープロファイル
- 監査ログ

## Phase 4. AI統合

- 複数LLMによる候補生成
- 候補間矛盾チェック
- 根拠のない候補を自動OFF
- 生存コンテキストだけをLLMへ渡す
- 回答信頼度と不存在判定

## Phase 5. ハードウェア実験

- FPGAによる候補並列比較
- アナログ信号の強め合い・打ち消し
- メモリスタまたは光メッシュ実験
- ソフトウェアMesh Engineとハードウェアアクセラレータの統合

---

# 21. プロジェクト構成

```text
negative-first-mesh-skill/
├── README.md
├── README.ko.md
├── README.en.md
├── README.ja.md
├── README.zh-CN.md
├── SKILL.md
├── SKILL.ko.md
├── SKILL.en.md
├── SKILL.ja.md
├── SKILL.zh-CN.md
├── INSTALL.md
├── INSTALL.ko.md
├── INSTALL.en.md
├── INSTALL.ja.md
├── INSTALL.zh-CN.md
├── EXAMPLES.md
├── EXAMPLES.ko.md
├── EXAMPLES.en.md
├── EXAMPLES.ja.md
├── EXAMPLES.zh-CN.md
├── TESTS.md
├── TESTS.ko.md
├── TESTS.en.md
├── TESTS.ja.md
├── TESTS.zh-CN.md
└── packages/
    ├── negative-first-mesh-ko.zip
    ├── negative-first-mesh-en.zip
    ├── negative-first-mesh-ja.zip
    └── negative-first-mesh-zh-CN.zip
```

---

# 22. 一文定義

> Negative-First Mesh Computing は、質問を候補メッシュ全体へ伝播させ、不一致・矛盾・範囲外・根拠不足の候補を先に抑制し、生き残った根拠だけで答えるか、活性候補数がゼロになった理由を正確に分類する計算・推論アーキテクチャである。

---

# 23. 中核原則

```text
答えを先に作らない。
候補領域を先に広げる。
違うものを先に消す。
ゼロを失敗とみなさない。
ゼロになったら次のグループへ進む。
合理的な全グループで何も残らなければ作り出さない。
生き残った根拠が支える内容だけを述べる。
偽、不在、不明を区別する。
```
---

# 24. 現行LLMへの適用

このスキルはCodex、Claude Code、Gemini CLIへネイティブインストールできる。

| 対象 | プロジェクト位置 | ユーザー全体位置 |
|---|---|---|
| Codex | `.agents/skills/negative-first-mesh/SKILL.md` | `~/.agents/skills/negative-first-mesh/SKILL.md` |
| Claude Code | `.claude/skills/negative-first-mesh/SKILL.md` | `~/.claude/skills/negative-first-mesh/SKILL.md` |
| Gemini CLI | `.gemini/skills/negative-first-mesh/SKILL.md` または `.agents/skills/...` | `~/.gemini/skills/negative-first-mesh/SKILL.md` または `~/.agents/skills/...` |

言語別スキルの一つを選び、インストール先で正確に `SKILL.md` としてコピーする。ネイティブスキル非対応LLMではプロジェクト指示またはシステム・開発者プロンプトとして適用する。

詳細は [INSTALL.ja.md](INSTALL.ja.md) を参照する。
