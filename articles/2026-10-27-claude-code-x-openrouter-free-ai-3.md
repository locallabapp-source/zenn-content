---
title: "Claude Code × OpenRouter Free モデル：AI エージェントのコスト最適化を実現する3つの設計パターン"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- OpenRouter の `:free` モデルを活用すると、AIエージェントの自動化ループをほぼゼロコストで回せる
- 「思考整理には上位モデル・量産タスクには無料モデル」という **ティア分離** が費用対効果を最大化する
- Rate Limit・品質劣化・フォールバック設計の3点を押さえれば本番運用に耐えられる

---

## 背景：AI エージェントは「コスト爆発」しやすい

Claude Code や LangChain / LangGraph を使って AIエージェントを組むと、すぐに月数万円のAPI費用が発生する。原因はシンプルで、**エージェントはループする** からだ。

```
[Plannerが計画立案] → [Executorが実行] × N回 → [Reviewerが品質確認]
```

このループが1日に数十〜数百回回ると、GPT-4oやClaude Opus はあっという間に費用を消費する。

解決策は「**全タスクに同じモデルを使わない**」こと。2026年現在、OpenRouterには複数の高品質な `:free` モデルが存在し、使い方を工夫すれば **品質を落とさずにコストを90%以上削減** できる。

---

## OpenRouter `:free` モデルの現状（2026年版）

OpenRouterでは、モデル名の末尾に `:free` を付けることで無料枠のエンドポイントを利用できる。2026年時点で実用的な主要モデルを以下に示す。

| モデル | 特徴 | 向いているタスク |
|---|---|---|
| `qwen/qwen3-235b-a22b:free` | MoE・高い指示追従性 | 文書生成・要約・翻訳 |
| `meta-llama/llama-4-maverick:free` | バランス型・英語強 | コードレビュー・QA |
| `google/gemini-2.0-flash-exp:free` | 高速・長コンテキスト | ログ分析・一括処理 |
| `deepseek/deepseek-r1:free` | 推論特化 | 数値計算・論理分析 |
| `mistralai/mistral-small-3.2-24b:free` | 軽量・低レイテンシ | 分類・ラベリング |

> **注意**: `:free` モデルは Rate Limit が厳しく（多くは 20 req/min 前後）、商用利用ポリシーは各モデルのライセンスに依存する。本番導入前に必ず OpenRouter 公式ドキュメントと各モデルのライセンス（Llama 3 Community License / Apache-2.0 / Qwen License 等）を確認すること。

---

## 設計パターン 1: ティア分離アーキテクチャ

**考え方**: タスクを「コスト・品質の要求度」でティア分けし、それぞれに最適なモデルを割り当てる。

```
Tier S (高品質必須)     → Claude Sonnet / GPT-4o
Tier A (品質・速度重視) → Qwen3-235b-a22b:free / Llama-4-Maverick:free
Tier B (量産・定型)     → Gemini-2.0-flash-exp:free / Mistral-Small:free
```

**実装イメージ（Python疑似コード）:**

```python
MODEL_TIERS = {
    "S": "anthropic/claude-sonnet-4",
    "A": "qwen/qwen3-235b-a22b:free",
    "B": "google/gemini-2.0-flash-exp:free",
}

def select_model(task_type: str) -> str:
    tier_map = {
        "final_review":      "S",  # 最終成果物の品質チェック
        "code_generation":   "A",  # コード生成
        "log_summarization": "B",  # ログ要約
        "classification":    "B",  # 分類ラベリング
        "translation":       "A",  # 翻訳
    }
    tier = tier_map.get(task_type, "A")
    return MODEL_TIERS[tier]
```

この設計の利点は、**タスクの種類によるモデル選択ロジックを1箇所に集約** できることだ。モデルの入れ替えや追加もここだけ変えれば済む。

---

## 設計パターン 2: フォールバックチェーン

`:free` モデルはRate Limitに引っかかると `429 Too Many Requests` を返す。本番エージェントでこれが起きると処理が止まる。

**解決策**: フォールバックチェーンを実装し、Free → Free' → 有料モデルの順で自動リトライする。

```python
import time
import openai  # openai SDK は OpenRouter にも互換

FALLBACK_CHAIN = [
    "qwen/qwen3-235b-a22b:free",
    "meta-llama/llama-4-maverick:free",
    "google/gemini-2.0-flash-exp:free",
    "anthropic/claude-haiku-4",  # 有料だが安価・最終手段
]

def chat_with_fallback(messages: list, task_type: str) -> str:
    preferred = select_model(task_type)
    chain = [preferred] + [m for m in FALLBACK_CHAIN if m != preferred]

    for model in chain:
        try:
            response = client.chat.completions.create(
                model=model,
                messages=messages,
                timeout=30,
            )
            return response.choices[0].message.content
        except openai.RateLimitError:
            print(f"[WARN] Rate limited on {model}, trying next...")
            time.sleep(2)
        except openai.APIError as e:
            print(f"[ERROR] {model} failed: {e}")
            continue

    raise RuntimeError("All models in fallback chain exhausted")
```

このパターンにより、**`:free` モデルが一時的に使えない場面でも自動で次の選択肢に切り替わる**。有料モデルへの到達は稀になるため、コストは抑えられたまま稼働率を高められる。

---

## 設計パターン 3: プロンプト圧縮によるコスト抑制

`:free` モデルでもトークン数が多いほどレスポンスが遅くなる。また、有料モデルへフォールバックした際のコストも跳ね上がる。

**対策**: エージェントが送るプロンプトを「システムプロンプト + 最小限の文脈」に圧縮する。

**Before（冗長な文脈）:**

```
あなたは優秀なソフトウェアエンジニアです。
以下の全ログ（10,000行）を確認して問題を見つけてください。
[ログ全文...]
```

**After（圧縮された文脈）:**

```
Error log summary (100 lines, filtered by ERROR/WARN):
[フィルタ済みログ...]
Task: Identify root cause. Output JSON: {"cause": str, "severity": "high|mid|low"}
```

圧縮のポイントは3つ:

1. **前処理でノイズを除去** — `grep -E "ERROR|WARN"` 等でフィルタしてから渡す
2. **出力形式を明示** — JSON / Markdown / 1行 など形式を指定するとトークン節約にもなる
3. **ロールプロンプトを短く** — 長いシステムプロンプトは最初の数ターンだけ渡し、以降は省略する

---

## Claude Code との組み合わせ

Claude Code はデフォルトで Anthropic API を使うが、**`ANTHROPIC_BASE_URL` や model override を使うことで OpenRouter 経由でも動作させられる**（非公式な使い方になるため公式サポート外である点に注意）。

より現実的な組み合わせは以下の構成だ:

```
Claude Code（Sonnet / 有料）
    │
    ├── 思考整理・意思決定・ユーザー向け最終出力
    │
    └── サブタスクの委譲
            │
            ├── OpenRouter :free（Qwen3）← 量産・繰り返し処理
            └── OpenRouter :free（Gemini）← 並列 batch 処理
```

Claude Code が「考える」部分を担当し、`:free` モデルが「量産する」部分を担当する **分業体制** がコスト最適化の本質だ。

---

## 実運用で気をつける3つのポイント

### 1. レート制限の監視

`:free` モデルは利用が集中すると制限が厳しくなる。`X-RateLimit-Remaining` ヘッダを監視して、残量が少なくなったら自動でモデルを切り替えるロジックを入れておくと安定する。

```python
def check_rate_limit_and_switch(response_headers: dict) -> str | None:
    remaining = int(response_headers.get("X-RateLimit-Remaining", 999))
    if remaining < 5:
        return "google/gemini-2.0-flash-exp:free"  # 別の :free へ切替
    return None
```

### 2. 品質モニタリング

`:free` モデルは有料モデルに比べて指示追従性が低い場面がある。特に**JSON出力の形式崩れ**が起きやすい。`pydantic` や `jsonschema` で出力バリデーションをかけ、パース失敗時は上位モデルでリトライする仕組みを入れておくと安定する。

```python
from pydantic import BaseModel, ValidationError

class TaskOutput(BaseModel):
    cause: str
    severity: str

def parse_output(raw: str) -> TaskOutput:
    try:
        import json
        return TaskOutput(**json.loads(raw))
    except (json.JSONDecodeError, ValidationError):
        # フォールバック：上位モデルで再生成
        raise OutputParseError("Invalid format, escalating to paid model")
```

### 3. ログとコスト可視化

OpenRouter はダッシュボードでトークン消費量を確認できる。`:free` モデルと有料モデルの **呼び出し比率** を定期的にチェックし、フォールバックが多すぎる場合はフォールバックチェーンの先頭モデルを変更する。

---

## まとめ

| 設計パターン | 効果 | 難易度 |
|---|---|---|
| ティア分離アーキテクチャ | コスト最大 90% 削減 | ★★☆ |
| フォールバックチェーン | 稼働率 99%+ 維持 | ★★★ |
| プロンプト圧縮 | レスポンス速度 2〜3倍 | ★☆☆ |

3つを組み合わせることで、**品質を保ちながらAIエージェントの運用コストを大幅に下げる** ことが現実的になる。

AIエージェントの本番運用で「コストが怖くて動かせない」という状況を打破するための第一歩として、まずはティア分離から試してみてほしい。

---

## 参考リンク

- [OpenRouter 公式ドキュメント](https://openrouter.ai/docs)
- [OpenRouter モデル一覧・レート制限](https://openrouter.ai/models)
- [Qwen3 技術レポート（Qwen 公式）](https://qwenlm.github.io/blog/qwen3/)
- [Llama 4 モデルカード（Meta AI）](https://ai.meta.com/blog/llama-4-multimodal-intelligence/)
- [Pydantic ドキュメント](https://docs.pydantic.dev/)
- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
