---
title: "Claude Code × OpenRouter Free Models で月0円 LLM 開発環境を作る3ステップ"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Claude Code のサブエージェントや補助 LLM 呼び出しに **OpenRouter の無料モデル** を活用すると、開発中の LLM コストをほぼゼロに抑えられる
- 設定は環境変数 1 本 + モデル名指定のみ。既存コードの変更は最小限
- 2026年現在、品質面で実用に耐えるモデルが `:free` サフィックスで複数提供されている

---

## はじめに

LLM を組み込んだアプリ開発をしていると、開発・検証フェーズだけで API 料金が積み上がる問題に直面します。

本番は GPT-4o や Claude Sonnet を使うとしても、**プロトタイプ段階 / CI での回帰テスト / エージェントのサブタスク** に同じモデルを使い続けるのは費用対効果が悪い。

OpenRouter は OpenAI 互換エンドポイントで多数のモデルを束ねるプロキシサービスですが、**モデル名末尾に `:free` を付けると無料枠でアクセスできるモデル**が複数存在します。この記事では、その仕組みと実際の組み込み方を整理します。

> ⚠️ `:free` モデルはレートリミットが厳しく、本番トラフィックには向きません。開発・検証フェーズ限定の使い方として読んでください。

---

## 1. OpenRouter `:free` モデルとは何か

### 仕組み

OpenRouter は各モデルプロバイダと契約し、API を統一インターフェースで提供しています。一部のモデルは **プロバイダ側のプロモーション枠** や **研究・OSS 普及目的** で無料公開されており、OpenRouter はそれを `:free` サフィックスで識別しています。

```
モデル指定例:
  通常:  google/gemma-3-27b-it
  無料:  google/gemma-3-27b-it:free
```

課金モデルと無料モデルは**同じモデルウェイト**を参照しており、品質は同等です。違いはレートリミット（RPM / TPM）と優先度キューです。

### 2026年時点で使えるおもな `:free` モデル

| モデル名 | コンテキスト | 強み |
|---|---|---|
| `google/gemma-3-27b-it:free` | 128K | 多言語・コード生成 |
| `meta-llama/llama-3.3-70b-instruct:free` | 131K | 汎用推論・英語品質 |
| `mistralai/mistral-7b-instruct:free` | 32K | 軽量・高速 |
| `qwen/qwen3-8b:free` | 32K | 日本語・コード |
| `deepseek/deepseek-r1-0528:free` | 64K | 推論・数学 |
| `microsoft/phi-3-mini-128k-instruct:free` | 128K | エッジ向け小型 |

> モデル一覧は [openrouter.ai/models](https://openrouter.ai/models) で随時確認してください（無料枠の追加・削除があります）。

---

## 2. 環境構築 (3ステップ)

### Step 1: OpenRouter API キーを取得する

1. [openrouter.ai](https://openrouter.ai) にサインアップ
2. Settings → Keys → **Create Key**
3. 生成されたキーを安全な場所に保存

無料プランでも API キーは発行されます。クレジットカード登録なしで `:free` モデルのみ使用可能。

### Step 2: 環境変数を設定する

OpenRouter は **OpenAI 互換エンドポイント** を提供しているため、`OPENAI_BASE_URL` をオーバーライドするだけで既存の OpenAI SDK がそのまま使えます。

```bash
# .env.local (または shell の export)
OPENAI_BASE_URL=https://openrouter.ai/api/v1
OPENAI_API_KEY=sk-or-v1-xxxxxxxxxxxx   # OpenRouter のキー
```

`openai` SDK を使っている既存コードは**変更不要**。

### Step 3: モデル名を `:free` に切り替える

```python
# Python (openai >= 1.0)
from openai import OpenAI

client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="sk-or-v1-xxxx",
)

response = client.chat.completions.create(
    model="google/gemma-3-27b-it:free",   # ← ここだけ変更
    messages=[
        {"role": "user", "content": "Rust の所有権を一文で説明して"}
    ],
)
print(response.choices[0].message.content)
```

```typescript
// TypeScript (openai npm パッケージ)
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "https://openrouter.ai/api/v1",
  apiKey: process.env.OPENAI_API_KEY,
});

const res = await client.chat.completions.create({
  model: "meta-llama/llama-3.3-70b-instruct:free",
  messages: [{ role: "user", content: "Hello!" }],
});
```

---

## 3. Claude Code との組み合わせ方

Claude Code (Anthropic の CLI エージェント) は、内部でサブタスクやツール呼び出しを行う際に**追加の LLM 呼び出し**が発生します。開発中にこのコストが気になる場合、以下のパターンが有効です。

### パターン A: Claude Code の外部ツールとして OpenRouter を呼ぶ

Claude Code の `tools` 定義に、OpenRouter へのプロキシ関数を登録します。

```python
# tool_openrouter_free.py
import os
import httpx

def call_free_llm(prompt: str, model: str = "qwen/qwen3-8b:free") -> str:
    """
    Claude Code から呼び出せる軽量 LLM ツール。
    サブタスクのドラフト生成や要約に使う。
    """
    resp = httpx.post(
        "https://openrouter.ai/api/v1/chat/completions",
        headers={
            "Authorization": f"Bearer {os.environ['OPENROUTER_API_KEY']}",
            "Content-Type": "application/json",
        },
        json={
            "model": model,
            "messages": [{"role": "user", "content": prompt}],
        },
        timeout=30,
    )
    resp.raise_for_status()
    return resp.json()["choices"][0]["message"]["content"]
```

Claude Code は「重い推論は自分（Sonnet）で行い、ドラフト生成や前処理は `call_free_llm` に任せる」という役割分担が自然にできます。

### パターン B: 環境変数でモデルを切り替える

CI / ローカル開発で使うモデルを環境変数で分岐させます。

```python
import os

def get_llm_config() -> dict:
    env = os.getenv("APP_ENV", "development")
    if env == "production":
        return {
            "base_url": "https://api.anthropic.com/",  # or openai
            "model": "claude-sonnet-4-5",
        }
    else:
        # 開発・CI では無料モデルを使う
        return {
            "base_url": "https://openrouter.ai/api/v1",
            "model": "google/gemma-3-27b-it:free",
        }
```

これだけで、**開発中の LLM コストをほぼゼロ**にしつつ、本番では高品質モデルを維持できます。

---

## 実運用でのハマりポイントと対処法

### レートリミット (429 Too Many Requests)

`:free` モデルは RPM（1分あたりリクエスト数）が厳しく制限されています。連続呼び出しが多い場合はエクスポネンシャルバックオフを必ず実装してください。

```python
import time
import random

def call_with_backoff(fn, max_retries=5):
    for attempt in range(max_retries):
        try:
            return fn()
        except Exception as e:
            if "429" in str(e) and attempt < max_retries - 1:
                wait = (2 ** attempt) + random.random()
                print(f"Rate limited. Waiting {wait:.1f}s...")
                time.sleep(wait)
            else:
                raise
```

### モデルの応答が遅い / タイムアウト

`:free` モデルは**優先度キューが低い**ため、混雑時にレスポンスが遅延します。タイムアウトを長め（30〜60秒）に設定し、同期的な UX には使わないのが無難です。

### JSON モード / Function Calling の互換性

モデルによって Function Calling や `response_format: { type: "json_object" }` のサポート状況が異なります。OpenRouter のモデル詳細ページで `supported_generation_methods` を確認してから使ってください。

```python
# JSON モードが使えるか確認してから呼ぶ
response = client.chat.completions.create(
    model="meta-llama/llama-3.3-70b-instruct:free",
    response_format={"type": "json_object"},  # 未対応モデルは省く
    messages=[...],
)
```

---

## モデル選定の目安

| ユースケース | 推奨 `:free` モデル | 理由 |
|---|---|---|
| 日本語テキスト生成・要約 | `qwen/qwen3-8b:free` | 日本語トークン効率が良い |
| 英語コード補完・レビュー | `meta-llama/llama-3.3-70b-instruct:free` | コード品質が高い |
| 数学・ステップ推論 | `deepseek/deepseek-r1-0528:free` | Chain-of-Thought 組み込み済み |
| 高速・軽量サブタスク | `mistralai/mistral-7b-instruct:free` | レイテンシ優先 |
| 長文コンテキスト (128K+) | `google/gemma-3-27b-it:free` | コンテキスト窓が広い |

---

## まとめ

| ポイント | 内容 |
|---|---|
| エンドポイント | `https://openrouter.ai/api/v1`（OpenAI 互換） |
| 切り替えコスト | 環境変数 2 本 + モデル名変更のみ |
| コスト削減 | 開発フェーズの LLM 費用をほぼゼロに |
| 注意点 | レートリミット・タイムアウト・Function Calling 互換性の確認 |

本番品質のモデルは高価ですが、**開発・検証フェーズでは無料モデルで十分なケースが多い**。環境変数 1 本で切り替えられる設計にしておくだけで、チーム全体の LLM 費用を大幅に削減できます。

---

## 参考リンク

- [OpenRouter 公式ドキュメント](https://openrouter.ai/docs)
- [OpenRouter モデル一覧 (`:free` フィルタ付き)](https://openrouter.ai/models?q=free)
- [openai-python SDK](https://github.com/openai/openai-python)
- [Claude Code 公式ドキュメント](https://docs.anthropic.com/claude-code)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
