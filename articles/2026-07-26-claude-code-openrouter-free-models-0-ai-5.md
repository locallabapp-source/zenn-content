---
title: "Claude Code + OpenRouter Free Models で月0円のAIエージェント開発環境を構築する5つの設計パターン"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Claude Code のサブエージェント呼び出しに OpenRouter の `:free` モデルを組み合わせると、ルーティング次第でコストをほぼゼロに抑えられる
- 「タスク複雑度に応じたモデル振り分け」「バッファキャッシュ活用」「並列化」「フォールバックチェーン」「プロンプト圧縮」の 5 パターンを解説
- 無料モデル単体では品質が不安定なため、オーケストレーション層で補完するのがコツ

---

## はじめに

AI エージェントを自社サービスに組み込みたいと思ったとき、最初の壁は「**推論コスト**」です。GPT-4o や Claude Sonnet を素直に呼び出し続けると、開発・検証フェーズだけで月数万円が飛んでいきます。

OpenRouter は複数の LLM を統一エンドポイントで呼び出せる API ゲートウェイで、`/api/v1/chat/completions` に `model: "<model-id>:free"` を指定するだけで、**無料枠のモデルにアクセスできます**。  
一方、Claude Code は Anthropic が提供するコーディング特化の LLM で、サブエージェントの呼び出しやコンテキスト分割が得意です。

この 2 つを組み合わせると、「**重い判断は Claude Code、軽い処理は OpenRouter Free**」という役割分担が自然に生まれます。本記事ではその設計パターンを 5 つ紹介します。

---

## 前提知識

### OpenRouter `:free` モデルとは

OpenRouter では、一部のモデルにレート制限付きの無料枠が提供されています。2026 年時点での代表例:

| モデル ID (例) | 無料枠の特徴 |
|---|---|
| `meta-llama/llama-3.1-8b-instruct:free` | 軽量・高速・英語が強い |
| `qwen/qwen-2.5-72b-instruct:free` | 日本語対応・高精度 |
| `mistralai/mistral-7b-instruct:free` | 軽量・instruction following が安定 |
| `google/gemma-3-12b-it:free` | コード生成・補完に強い |

`:free` は RPM (Requests Per Minute) と TPD (Tokens Per Day) に上限があるため、**高頻度・大量トークンのタスクには向かない**点に注意が必要です。

### Claude Code のサブエージェント呼び出し

Claude Code は `tool_use` ブロックや外部 API 呼び出しを通じて、別の LLM エンドポイントにタスクを委譲できます。いわゆる「オーケストレーター/ワーカー」パターンです。

```python
# 疑似コード: オーケストレーターが OpenRouter に委譲する例
import httpx

def call_free_model(prompt: str, model: str = "qwen/qwen-2.5-72b-instruct:free") -> str:
    response = httpx.post(
        "https://openrouter.ai/api/v1/chat/completions",
        headers={"Authorization": f"Bearer {OPENROUTER_API_KEY}"},
        json={
            "model": model,
            "messages": [{"role": "user", "content": prompt}],
        },
    )
    return response.json()["choices"][0]["message"]["content"]
```

上記はあくまで**公式ドキュメントに準拠した最小例**です。実際にはリトライ・タイムアウト・ストリーミング処理を追加します。

---

## 5 つの設計パターン

### パターン 1: タスク複雑度ルーティング

**「この処理、本当に Sonnet が必要か?」** という問いが出発点です。

#### 複雑度スコアリング

```python
from dataclasses import dataclass
from enum import Enum

class Complexity(Enum):
    LOW = "low"       # 翻訳・要約・フォーマット変換
    MEDIUM = "medium" # 軽いコード生成・分類・抽出
    HIGH = "high"     # 複雑な推論・マルチステップ・設計判断

@dataclass
class TaskRouter:
    def route(self, task_description: str) -> str:
        complexity = self._estimate_complexity(task_description)
        match complexity:
            case Complexity.LOW:
                return "meta-llama/llama-3.1-8b-instruct:free"
            case Complexity.MEDIUM:
                return "qwen/qwen-2.5-72b-instruct:free"
            case Complexity.HIGH:
                return "anthropic/claude-sonnet-4-5"  # 有料モデルを温存

    def _estimate_complexity(self, desc: str) -> Complexity:
        # キーワードベースの簡易スコアリング (本番では LLM 判定も可)
        high_signals = ["設計", "アーキテクチャ", "トレードオフ", "複数ステップ"]
        low_signals = ["翻訳", "要約", "箇条書き", "フォーマット"]
        score = sum(s in desc for s in high_signals) - sum(s in desc for s in low_signals)
        if score >= 2:
            return Complexity.HIGH
        elif score >= 0:
            return Complexity.MEDIUM
        return Complexity.LOW
```

このルーターを Claude Code のオーケストレーター層に組み込むだけで、**単純タスクの 70〜80% を無料モデルに委譲**できるようになります。

#### コスト削減イメージ

```
Before: 全タスクを Sonnet → 月 $50〜$200
After:  LOW/MEDIUM を :free、HIGH のみ Sonnet → 月 $5〜$30
```

---

### パターン 2: プロンプトキャッシュ + `:free` モデルの併用

Anthropic の Prompt Caching は、**同一プレフィックスが 2 回目以降にキャッシュヒットするとトークン単価が 90% 減**になる仕組みです。  
一方、OpenRouter Free はキャッシュの概念がない代わりにゼロコストです。

この 2 つの特性を活かした設計:

```
[長い共通コンテキスト (システムプロンプト + RAG 結果)]
        │
        ├─ 判断が必要な部分 → Claude (キャッシュヒットで安価)
        └─ 繰り返しの定型処理 → OpenRouter :free (完全無料)
```

具体的な使い分け:

| 処理 | 推奨モデル | 理由 |
|---|---|---|
| 長い PDF の要約 (繰り返し) | `:free` + 圧縮プロンプト | キャッシュより単純な圧縮が効果的 |
| コードレビュー (システムプロンプト固定) | Claude + Prompt Cache | キャッシュヒット率が高い |
| ユーザー入力の intent 分類 | `:free` (軽量モデル) | 短いプロンプトで十分 |
| 多段階エージェントの最終判断 | Claude | 精度最優先 |

---

### パターン 3: 並列ファンアウト + マージ

単一の難しいタスクを「複数の簡単なサブタスク」に分解して、それぞれを無料モデルで並列処理し、結果をオーケストレーターがマージする手法です。

```python
import asyncio
from typing import Callable

async def fan_out_and_merge(
    task: str,
    subtask_prompts: list[str],
    worker_model: str = "qwen/qwen-2.5-72b-instruct:free",
    merge_fn: Callable[[list[str]], str] = lambda results: "\n\n".join(results),
) -> str:
    """
    subtask_prompts: タスクを分割したプロンプトのリスト
    worker_model: 各サブタスクを処理する無料モデル
    merge_fn: 結果をまとめる関数 (Claude に渡しても良い)
    """
    async def call_worker(prompt: str) -> str:
        # 実際は httpx.AsyncClient を使用
        return await async_call_openrouter(prompt, model=worker_model)

    results = await asyncio.gather(*[call_worker(p) for p in subtask_prompts])
    return merge_fn(list(results))
```

#### 適用例: 長文ドキュメントの構造化抽出

```
[入力: 50 ページの仕様書]
        │
  ページ単位で 50 分割
        │
  各分割を :free で並列処理 (50 並列)
        │
  抽出結果を Claude Sonnet でマージ・整形
```

並列度の上限は OpenRouter の RPM 制限に合わせて調整します。無料プランでは概ね 20〜60 RPM 程度が目安です (公式ドキュメントで要確認)。

---

### パターン 4: フォールバックチェーン

`:free` モデルはレート制限に引っかかったり、応答品質が期待を下回ることがあります。本番環境では**フォールバックチェーン**を設計しておくことが重要です。

```python
from typing import Optional
import httpx

FREE_MODEL_CHAIN = [
    "qwen/qwen-2.5-72b-instruct:free",
    "meta-llama/llama-3.1-8b-instruct:free",
    "mistralai/mistral-7b-instruct:free",
]
PAID_FALLBACK = "anthropic/claude-haiku-4-5"  # 最終フォールバックは安価な有料モデル

async def call_with_fallback(
    prompt: str,
    quality_check: Optional[Callable[[str], bool]] = None,
) -> tuple[str, str]:
    """
    Returns: (応答テキスト, 使用モデル名)
    quality_check: 応答品質を検証する関数 (None なら無条件に通過)
    """
    for model in FREE_MODEL_CHAIN:
        try:
            result = await async_call_openrouter(prompt, model=model, timeout=10.0)
            if quality_check is None or quality_check(result):
                return result, model
        except (httpx.TimeoutException, RateLimitError):
            continue  # 次のモデルへ

    # 全無料モデルが失敗 → 有料モデルで確実に処理
    result = await async_call_openrouter(prompt, model=PAID_FALLBACK)
    return result, PAID_FALLBACK
```

#### フォールバックの品質チェック例

```python
def basic_quality_check(response: str) -> bool:
    """極端に短い・明らかにエラーな応答を弾く"""
    if len(response.strip()) < 50:
        return False
    error_phrases = ["I cannot", "I'm unable", "エラーが発生", "申し訳ありません"]
    return not any(phrase in response for phrase in error_phrases)
```

---

### パターン 5: プロンプト圧縮パイプライン

無料モデルはコンテキストウィンドウの上限が小さかったり、長いプロンプトでの精度が落ちる傾向があります。**事前にプロンプトを圧縮**することで、品質を維持しながらトークン数を削減できます。

#### 圧縮の 3 段階

```
[元のプロンプト: 4,000 tokens]
        │
  Step 1: 構造的削減
  ・不要な丁寧語を除去
  ・重複説明を 1 箇所に集約
  ・箇条書き → コード的表現に変換
        │
  Step 2: RAG 精度向上
  ・埋め込みスコアが低いチャンクを除外
  ・関連度 0.75 未満のコンテキストを切り捨て
        │
  Step 3: 要約モデルで事前圧縮
  ・長い背景説明を別の :free モデルで先に要約
  ・要約結果をメインプロンプトに差し込む
        │
  [圧縮後: 1,200 tokens] → :free モデルへ
```

```python
def compress_prompt(
    original: str,
    target_tokens: int = 1200,
    summarizer_model: str = "meta-llama/llama-3.1-8b-instruct:free",
) -> str:
    """長いプロンプトを圧縮して :free モデルに最適化する"""
    estimated_tokens = len(original) // 4  # 大雑把な日本語トークン見積もり
    
    if estimated_tokens <= target_tokens:
        return original  # 圧縮不要
    
    compression_prompt = f"""以下のテキストを {target_tokens * 4} 文字以内に要約してください。
重要な指示・制約・数値は必ず保持すること。

---
{original}
---

要約:"""
    
    return call_openrouter_sync(compression_prompt, model=summarizer_model)
```

**注意**: 圧縮モデル自体もコストがかかるため、「圧縮コスト < 節約トークンコスト」になるレベルのプロンプトにだけ適用します。目安は 3,000 tokens 以上。

---

## 5 パターンの組み合わせ方

実際のエージェントシステムでは、これら 5 つを組み合わせることで相乗効果が生まれます。

```
[ユーザーリクエスト受信]
        │
  パターン 1: 複雑度判定
        │
  LOW/MEDIUM ─→ パターン 5: プロンプト圧縮
                       │
               パターン 4: フォールバックチェーンで :free 呼び出し
                       │
  HIGH ─────→ パターン 3: サブタスク分解
                       │
               パターン 4: 各サブタスクに :free 投入
                       │
               パターン 2: キャッシュを活かして Claude でマージ
        │
  [最終レスポンス]
```

このアーキテクチャを採用した場合の**コスト構造の変化**:

| フェーズ | モデル | コスト |
|---|---|---|
| 複雑度判定 (パターン 1) | `:free` (軽量) | $0 |
| プロンプト圧縮 (パターン 5) | `:free` (軽量) | $0 |
| LOW/MEDIUM タスク本体 | `:free` (高品質) | $0 |
| HIGH タスクのサブ処理 | `:free` (並列) | $0 |
| HIGH タスクのマージ・最終判断 | Claude (キャッシュ) | ↓ 90% 減 |

---

## 実装時の注意点

### 1. レート制限の監視

OpenRouter の無料枠は予告なく変更されることがあります。本番運用では `X-RateLimit-Remaining` ヘッダを監視し、枯渇したら即時有料モデルへ切り替えるロジックが必須です。

```python
def check_rate_limit(response_headers: dict) -> bool:
    """残りレート制限が少ない場合は True を返す (フォールバック推奨)"""
    remaining = int(response_headers.get("X-RateLimit-Remaining", 100))
    return remaining < 10
```

### 2. モデル品質のドリフト

`:free` モデルはバージョンアップや差し替えが頻繁です。定期的に**品質評価テストを自動化**して、モデル変更を検知できるようにしておきましょう。

```python
QUALITY_BENCHMARK = [
    ("日本語で「りんご」を英語に翻訳してください", lambda r: "apple" in r.lower()),
    ("1 + 1 = ?", lambda r: "2" in r),
    ("FizzBuzz を Python で書いてください", lambda r: "fizz" in r.lower()),
]

def run_quality_check(model: str) -> float:
    """0.0〜1.0 のスコアを返す"""
    passed = sum(
        check_fn(call_openrouter_sync(prompt, model=model))
        for prompt, check_fn in QUALITY_BENCHMARK
    )
    return passed / len(QUALITY_BENCHMARK)
```

### 3. 日本語対応の注意

軽量モデル (7B 以下) は日本語精度が英語より大幅に落ちることがあります。日本語タスクには `qwen/qwen-2.5-72b-instruct:free` か `google/gemma-3-12b-it:free` など**大きめのモデル**を選ぶと安定します。

---

## まとめ

| パターン | 主な効果 | 難易度 |
|---|---|---|
| 1. 複雑度ルーティング | コスト 60〜80% 削減の基本形 | ★★☆ |
| 2. キャッシュ + Free 併用 | キャッシュヒット率向上 | ★★☆ |
| 3. 並列ファンアウト | スループット向上 + コスト分散 | ★★★ |
| 4. フォールバックチェーン | 可用性・信頼性向上 | ★★☆ |
| 5. プロンプト圧縮 | トークン削減 + Free モデル精度向上 | ★★★ |

「全部 Sonnet で解決」から「重要な部分だけ Sonnet」へ設計を転換するだけで、**開発・検証フェーズのコストを大幅に圧縮**できます。  
OpenRouter の無料枠は変動するため、**フォールバック設計を最初から組み込む**のが長期運用のポイントです。

まずはパターン 1 (複雑度ルーティング) だけ導入して、コスト変化を計測してみてください。

---

## 参考リンク

- [OpenRouter 公式ドキュメント](https://openrouter.ai/docs)
- [Anthropic Prompt Caching ドキュメント](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)
- [OpenRouter モデル一覧](https://openrouter.ai/models)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
