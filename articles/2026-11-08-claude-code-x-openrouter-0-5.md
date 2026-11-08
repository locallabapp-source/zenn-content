---
title: "Claude Code × OpenRouter 無料モデル：コスト0円で自律エージェントを動かす5つの設計パターン"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- OpenRouter の `:free` サフィックスモデルを使えば LLM 呼び出しコストをゼロにできる
- Claude Code はカスタムモデルを環境変数一本で差し替えられる
- 無料モデルでもツールコール・マルチターン推論が動く組み合わせは存在する
- レートリミット・コンテキスト長・ツールコール対応の 3 軸で選定すると安定運用できる

---

## 背景：「エージェント = 高コスト」という思い込みを壊す

LLM を使った自律エージェントを日常業務に組み込もうとしたとき、真っ先に壁になるのがコストだ。GPT-4o や Claude Opus を 1 日数十回呼び出せば、開発フェーズだけで月数万円に達することも珍しくない。

しかし 2025〜2026 年にかけて、**OpenRouter の `:free` モデル群が実用レベルに達してきた**。Qwen3・Llama 3.3・Mistral 系の最新モデルが無料で叩けるようになり、ツールコール（function calling）対応モデルも増えた。

本記事では、**Claude Code をオーケストレーターとして活用しつつ、実作業は OpenRouter 無料モデルに委譲する**という構成を 5 つのパターンに整理して解説する。

---

## 前提知識：OpenRouter の `:free` モデルとは

OpenRouter は複数の LLM プロバイダーを統一 API（OpenAI 互換）で提供するプロキシサービスだ。

`:free` サフィックスを持つモデルは、以下の制約下で**0 ドルで利用できる**：

| 制約項目 | 一般的な値 |
|---|---|
| レートリミット | 20 req/min（モデルによって異なる） |
| コンテキスト長 | 8K〜128K（モデルによって異なる） |
| ツールコール対応 | モデル次第（要確認） |
| 優先度 | 有料モデルより低い（混雑時遅延あり） |

2026 年 5 月時点で注目の `:free` モデル（実装例として列挙）：

```
qwen/qwen3-235b-a22b:free          # 推論力が高く MoE 構成で軽量
meta-llama/llama-3.3-70b-instruct:free
mistralai/mistral-7b-instruct:free
google/gemma-3-27b-it:free
```

> **注意**: モデルの提供状況は頻繁に変わる。[openrouter.ai/models](https://openrouter.ai/models) で `free` フィルタをかけて最新状況を確認すること。

---

## Claude Code とのインテグレーション基礎

Claude Code は `ANTHROPIC_API_KEY` と `claude` CLI を中心に動くが、**OpenAI 互換 API を向く設定**も公式にサポートしている。

```bash
# .env or export
OPENROUTER_API_KEY=sk-or-...
OPENAI_BASE_URL=https://openrouter.ai/api/v1
OPENAI_API_KEY=$OPENROUTER_API_KEY
```

Claude Code の `claude` コマンド自体は Anthropic に向くが、**サブエージェントや自作スクリプトを OpenAI SDK 経由で OpenRouter に向ける**のが基本戦略だ。

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="sk-or-...",  # 実際は環境変数から読む
)

response = client.chat.completions.create(
    model="qwen/qwen3-235b-a22b:free",
    messages=[{"role": "user", "content": "Hello"}],
)
```

---

## 5 つの設計パターン

### パターン 1: ルーティング型 — タスク複雑度でモデルを切り替える

**概要**: 簡単なタスクは `:free` モデルへ、複雑な判断のみ有料モデルへルーティングする。

```python
def route_model(task: str, complexity: str) -> str:
    """タスク複雑度でモデルを決定する"""
    routing_table = {
        "low":    "qwen/qwen3-235b-a22b:free",
        "medium": "meta-llama/llama-3.3-70b-instruct:free",
        "high":   "anthropic/claude-3.5-sonnet",  # 有料
    }
    return routing_table.get(complexity, routing_table["medium"])
```

**適用場面**: ファイルのサマリー生成（low）/ コードレビュー（medium）/ アーキテクチャ設計判断（high）

**ポイント**: 複雑度の判定ロジック自体も LLM で行うと「判定に金がかかる」という本末転倒が起きる。**ヒューリスティックベースの事前分類**（トークン数・タスクタイプのラベル）で判定するのが実践的だ。

---

### パターン 2: ファンアウト型 — 並列実行で品質を担保する

**概要**: 同一タスクを複数の `:free` モデルに並列投入し、結果をアグリゲートする。

```python
import asyncio

async def fan_out_generate(prompt: str) -> list[str]:
    models = [
        "qwen/qwen3-235b-a22b:free",
        "meta-llama/llama-3.3-70b-instruct:free",
        "google/gemma-3-27b-it:free",
    ]
    tasks = [call_model(model, prompt) for model in models]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    # 例外は除外して返す
    return [r for r in results if not isinstance(r, Exception)]
```

**適用場面**: テストコード生成・翻訳・ドキュメント草案。複数の出力からベストを人間が選ぶ、あるいは多数決ロジックで最終成果物を決定する。

**ポイント**: `:free` モデルはレートリミットが厳しいため、`asyncio.Semaphore` でリクエスト数を絞ることが必須。

```python
sem = asyncio.Semaphore(5)  # 同時 5 リクエストまで

async def call_model_limited(model: str, prompt: str) -> str:
    async with sem:
        return await call_model(model, prompt)
```

---

### パターン 3: フォールバックチェーン型 — 無料を最優先・失敗時に有料へ

**概要**: `:free` モデルが失敗（タイムアウト・レートリミット）したときのみ、有料モデルにフォールバックする。

```python
from tenacity import retry, stop_after_attempt, wait_exponential

FALLBACK_CHAIN = [
    "qwen/qwen3-235b-a22b:free",
    "meta-llama/llama-3.3-70b-instruct:free",
    "anthropic/claude-3.5-haiku",  # 最終フォールバック（有料・低コスト）
]

async def call_with_fallback(prompt: str) -> str:
    last_error = None
    for model in FALLBACK_CHAIN:
        try:
            return await call_model(model, prompt)
        except Exception as e:
            last_error = e
            continue
    raise last_error
```

**ポイント**: 有料モデルへの落下を**ログに記録**することで、どのタスクがコストを発生させているか可視化できる。コスト最適化の改善サイクルに直結する。

---

### パターン 4: ツールコール委譲型 — 無料モデルに関数実行を任せる

**概要**: Claude Code がオーケストレーションを担い、**個々のツール実行判断を `:free` モデルに委ねる**。

ツールコール対応モデルの確認が前提になる。OpenRouter ではモデルの `supported_parameters` に `tools` が含まれているか確認する。

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "ファイルを読み込んで内容を返す",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string"}
                },
                "required": ["path"],
            },
        },
    }
]

response = client.chat.completions.create(
    model="qwen/qwen3-235b-a22b:free",
    messages=messages,
    tools=tools,
    tool_choice="auto",
)
```

**適用場面**: ファイルツリーの探索・grep 検索の組み合わせ・外部 API からのデータ取得。作業の「どのツールをどの順で叩くか」という判断を無料モデルに任せられると、コストが大幅に削減される。

**注意点**: `:free` モデルのツールコール精度は有料モデルに比べて不安定なケースがある。**JSON スキーマを簡潔に保つ**（プロパティを 5 つ以内に絞る）ことで精度が上がる傾向がある。

---

### パターン 5: 段階的精緻化型 — ドラフトを無料で、推敲を最小コストで

**概要**: 草案生成（トークンが多い）を `:free` モデルで行い、**最終的な品質チェックのみ軽量有料モデル**で行う。

```
[タスク] ─→ [Free: 草案生成 2000 tokens out]
                │
                ↓
           [Haiku: レビュー 200 tokens out]
                │
                ↓
           [最終成果物]
```

```python
async def draft_then_refine(task_description: str) -> str:
    # ステップ 1: 無料モデルで草案
    draft = await call_model(
        "qwen/qwen3-235b-a22b:free",
        f"以下のタスクの草案を作成してください:\n{task_description}"
    )

    # ステップ 2: 軽量有料モデルで品質チェックのみ
    refined = await call_model(
        "anthropic/claude-3.5-haiku",
        f"以下の草案の品質問題（矛盾・誤り・不明瞭な点）を 3 点以内で指摘してください:\n{draft}"
    )

    return {"draft": draft, "review": refined}
```

**ポイント**: この構成だと生成トークンの 90% 以上を無料モデルが担い、有料モデルへの支払いは「指摘文の生成」という少量トークンだけで済む。

---

## モデル選定の 3 軸評価フレームワーク

`:free` モデルを選ぶとき、以下の 3 軸でスコアリングすると失敗が少ない。

| 軸 | 確認方法 | 重要度 |
|---|---|---|
| **ツールコール対応** | openrouter.ai のモデル詳細で `tools` 対応を確認 | ⭐⭐⭐ |
| **コンテキスト長** | エージェントが扱うファイルサイズの最大値と比較 | ⭐⭐⭐ |
| **レートリミット** | `/api/v1/models` エンドポイントの `rate_limit` フィールド | ⭐⭐ |
| **推論能力** | MMLU / HumanEval 等の公開ベンチマークで比較 | ⭐⭐ |
| **レスポンス速度** | TTFT（Time to First Token）の実測 | ⭐ |

### 実践的な選定フロー

```
ツールコール必要？
  ├─ Yes → tools 対応モデルに絞る
  │         └─ Qwen3 / Llama 3.3 系が安定
  └─ No  → テキスト生成品質で選ぶ
            └─ コンテキスト長 > 32K が必要なら
               Gemini 無料 API (1M context) も検討
```

---

## よくある落とし穴と対処法

### 1. レートリミット超過で本番が止まる

`:free` モデルはリクエスト数制限が厳格だ。**指数バックオフ + ジッター**を実装しておくことが必須。

```python
import random, asyncio

async def call_with_backoff(model: str, prompt: str, max_retries: int = 5) -> str:
    for attempt in range(max_retries):
        try:
            return await call_model(model, prompt)
        except RateLimitError:
            wait = (2 ** attempt) + random.uniform(0, 1)
            await asyncio.sleep(wait)
    raise Exception("max retries exceeded")
```

### 2. コンテキスト長超過でサイレントにトランケートされる

モデルごとにコンテキスト上限が異なる。**入力トークン数を事前にカウントして制限を超えたら分割**する。

```python
import tiktoken

def count_tokens(text: str, model: str = "gpt-4") -> int:
    # OpenRouter モデルの多くは tiktoken と互換
    enc = tiktoken.encoding_for_model(model)
    return len(enc.encode(text))

def chunk_if_needed(text: str, max_tokens: int = 6000) -> list[str]:
    if count_tokens(text) <= max_tokens:
        return [text]
    # 段落単位で分割（実装省略）
    return split_by_paragraph(text, max_tokens)
```

### 3. ツールコールの JSON が壊れる

小規模モデルはツールコール JSON を途中で壊すことがある。**JSON 修復ライブラリ**（`json-repair` 等）をパーサーの前段に置くと安定性が上がる。

```python
import json_repair  # pip install json-repair

def safe_parse_tool_call(raw: str) -> dict:
    try:
        return json.loads(raw)
    except json.JSONDecodeError:
        return json_repair.repair_json(raw, return_objects=True)
```

---

## コスト試算：1 日 100 タスクをこなした場合

| 構成 | モデル | 推定月額 |
|---|---|---|
| 全タスク GPT-4o | 有料のみ | ~$50-200 |
| 全タスク Claude Sonnet | 有料のみ | ~$30-150 |
| パターン 1（80% free） | `:free` + Haiku | ~$3-15 |
| パターン 3（95% free） | `:free` フォールバック | ~$1-5 |

> 数値は 1 タスクあたり平均 1,000 input tokens / 500 output tokens と仮定した概算。実際はタスク内容で大きく変動する。

`:free` モデル主体の構成なら、**月数百円〜数千円レベル**でエージェント運用が現実的になる。

---

## まとめ

OpenRouter `:free` モデルの実用性は 2026 年現在、エージェント用途に十分耐えうるレベルに達している。ポイントを再整理する。

1. **ルーティング**で複雑度に応じたモデル選択を自動化する
2. **ファンアウト**で品質を並列実行で担保する
3. **フォールバックチェーン**でコストを最小化しつつ可用性を確保する
4. **ツールコール委譲**で判断処理を無料モデルに任せる
5. **段階的精緻化**でトークン消費の 90% を無料枠に押し込む

「無料 = 低品質」という先入観を捨て、タスクと精度要件に応じてモデルを使い分けることが、持続可能なエージェント設計の鍵だ。

---

## 参考リンク

- [OpenRouter 公式ドキュメント](https://openrouter.ai/docs)
- [OpenRouter モデル一覧（free フィルタ付き）](https://openrouter.ai/models?q=free)
- [Qwen3 技術レポート (Qwen Team)](https://qwenlm.github.io/blog/qwen3/)
- [Llama 3.3 Model Card (Meta)](https://huggingface.co/meta-llama/Llama-3.3-70B-Instruct)
- [tenacity: Python リトライライブラリ](https://github.com/jd/tenacity)
- [json-repair: 破損 JSON 修復ライブラリ](https://github.com/mangiucugna/json_repair)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
