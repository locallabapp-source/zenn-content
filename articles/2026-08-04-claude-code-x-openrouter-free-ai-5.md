---
title: "Claude Code × OpenRouter :free モデルで AI 開発コストをほぼゼロにする5つの設定パターン"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- OpenRouter の `:free` サフィックスモデルを活用すれば LLM 推論コストをほぼゼロにできる
- Claude Code (claude.ai/code) は `ANTHROPIC_BASE_URL` のオーバーライドで外部エンドポイントを向けられる
- 2026 年時点で使える高品質 `:free` モデルは Qwen3、DeepSeek-R1、Gemma 3 など多数存在する
- 用途に応じた「コスト×品質」の最適化パターンを 5 つ紹介する

---

## 背景: LLM コストは「積み重ねると痛い」

AI エージェントを日常的に回していると、気づけば月の API 請求が数万円を超えていることがある。Claude Opus 4 はプログラミング補助として抜群に優秀だが、**単純なドキュメント要約・翻訳・ログ解析** のようなタスクに Opus クラスを投入するのはコスト設計として合理的ではない。

OpenRouter は複数プロバイダの LLM を単一エンドポイント (`https://openrouter.ai/api/v1`) で使えるプロキシサービスだ。ここで重要なのが **`:free` サフィックス** の存在—モデル ID 末尾に `:free` を付けると、レート制限付きながら **課金ゼロ** でそのモデルを呼べる。

---

## OpenRouter `:free` モデルの仕組み

### エンドポイントと認証

```
Base URL : https://openrouter.ai/api/v1
Auth     : Authorization: Bearer <OPENROUTER_API_KEY>
形式     : OpenAI 互換 (chat/completions)
```

OpenAI SDK・Anthropic SDK の `base_url` をこのエンドポイントに向け、`api_key` を OpenRouter キーに替えるだけで既存コードがそのまま動く。

### `:free` ティアの制約 (2026-05 時点の公式ドキュメント準拠)

| 項目 | 内容 |
|---|---|
| 課金 | $0 (無料) |
| レート制限 | 約 20 req/min・1,000 req/day (モデルによって異なる) |
| コンテキスト長 | 有料版と同一 (モデル依存) |
| ストリーミング | 対応 |
| キャッシュ | 利用可 |

> ⚠️ レート制限は OpenRouter 側で随時変更される。本番 SLA が必要なワークロードには `:free` モデルのみの依存は避けること。

---

## 代表的な `:free` モデルと特性

### 2026 年前半に実績のある主要モデル

| モデル ID (OpenRouter) | パラメータ | 強み |
|---|---|---|
| `qwen/qwen3-235b-a22b:free` | 235B (MoE) | コーディング・推論・多言語 |
| `deepseek/deepseek-r1:free` | 671B (MoE) | 数学・論理推論・Chain-of-Thought |
| `google/gemma-3-27b-it:free` | 27B | 軽量・速い・日本語そこそこ |
| `meta-llama/llama-4-maverick:free` | 400B (MoE) | 汎用・長文処理 |
| `mistralai/mistral-small-3.2-24b-instruct:free` | 24B | ヨーロッパ言語・コード |

> モデル一覧は `https://openrouter.ai/models?q=:free` で随時確認できる。パラメータ数や特性は OpenRouter の公式モデルカード参照。

---

## 5 つの設定パターン

### パターン 1: Python スクリプトで最小構成

最もシンプルな接続確認。`openai` ライブラリを流用する。

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="sk-or-v1-xxxx",  # OpenRouter API Key
)

response = client.chat.completions.create(
    model="qwen/qwen3-235b-a22b:free",
    messages=[{"role": "user", "content": "Rustのライフタイム注釈を3行で説明して"}],
    max_tokens=512,
)
print(response.choices[0].message.content)
```

ポイント: `model` パラメータを変えるだけで別モデルに切り替えられる。

---

### パターン 2: 環境変数でモデルを外部化

ハードコードをなくし、`.env` でモデルを切り替えられるようにする。

```python
import os
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()

client = OpenAI(
    base_url=os.environ["OPENROUTER_BASE_URL"],  # https://openrouter.ai/api/v1
    api_key=os.environ["OPENROUTER_API_KEY"],
)

MODEL = os.environ.get("FREE_MODEL", "qwen/qwen3-235b-a22b:free")

def ask(prompt: str) -> str:
    res = client.chat.completions.create(
        model=MODEL,
        messages=[{"role": "user", "content": prompt}],
        max_tokens=1024,
    )
    return res.choices[0].message.content
```

```ini
# .env
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1
OPENROUTER_API_KEY=sk-or-v1-xxxx
FREE_MODEL=deepseek/deepseek-r1:free
```

`.env` を変えるだけでモデルを差し替えられるため、A/B 比較が容易になる。

---

### パターン 3: フォールバック付きリトライ (レート制限対策)

`:free` モデルは混雑時に `429 Too Many Requests` が返る。指数バックオフ + モデルフォールバックで安定させる。

```python
import time
import logging
from openai import OpenAI, RateLimitError

logger = logging.getLogger(__name__)

FREE_MODELS = [
    "qwen/qwen3-235b-a22b:free",
    "deepseek/deepseek-r1:free",
    "google/gemma-3-27b-it:free",
]

def ask_with_fallback(client: OpenAI, prompt: str, max_retries: int = 3) -> str:
    for model in FREE_MODELS:
        for attempt in range(max_retries):
            try:
                res = client.chat.completions.create(
                    model=model,
                    messages=[{"role": "user", "content": prompt}],
                    max_tokens=1024,
                )
                return res.choices[0].message.content
            except RateLimitError:
                wait = 2 ** attempt
                logger.warning(f"{model} rate limited. retry in {wait}s")
                time.sleep(wait)
        logger.warning(f"{model} exhausted, trying next model")

    raise RuntimeError("All free models exhausted")
```

このパターンでは **モデルの優先順位** を `FREE_MODELS` リストで制御できる。用途に応じて並び替えるだけでよい。

---

### パターン 4: TypeScript / Node.js での利用

フロントエンド・フルスタック開発者向け。`openai` npm パッケージを使う。

```typescript
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "https://openrouter.ai/api/v1",
  apiKey: process.env.OPENROUTER_API_KEY!,
  defaultHeaders: {
    "HTTP-Referer": "https://locallab.jp",   // OpenRouter の利用元を明示 (任意)
    "X-Title": "Jimolab Tech Demo",          // OpenRouter ダッシュボードに表示される名前
  },
});

async function summarize(text: string): Promise<string> {
  const res = await client.chat.completions.create({
    model: "qwen/qwen3-235b-a22b:free",
    messages: [
      { role: "system", content: "日本語で要約してください。箇条書き3点で。" },
      { role: "user", content: text },
    ],
    max_tokens: 256,
  });
  return res.choices[0].message.content ?? "";
}
```

`HTTP-Referer` と `X-Title` ヘッダーは OpenRouter の **ダッシュボード上でリクエスト元を識別** するために使う任意フィールド。複数アプリを使い分けるときに便利。

---

### パターン 5: モデルのコスト×品質マトリクスで自動ルーティング

タスク種別に応じてモデルを自動選択するルータークラス。**コーディング → Qwen3 / 推論 → DeepSeek-R1 / 翻訳 → Gemma** のように仕分ける。

```python
from dataclasses import dataclass
from enum import Enum
from openai import OpenAI

class TaskType(Enum):
    CODING = "coding"
    REASONING = "reasoning"
    TRANSLATION = "translation"
    SUMMARIZATION = "summarization"
    GENERAL = "general"

@dataclass
class ModelRoute:
    model: str
    max_tokens: int
    system_prompt: str

ROUTING_TABLE: dict[TaskType, ModelRoute] = {
    TaskType.CODING: ModelRoute(
        model="qwen/qwen3-235b-a22b:free",
        max_tokens=2048,
        system_prompt="You are an expert software engineer. Write clean, idiomatic code.",
    ),
    TaskType.REASONING: ModelRoute(
        model="deepseek/deepseek-r1:free",
        max_tokens=4096,
        system_prompt="Think step by step. Show your reasoning before the answer.",
    ),
    TaskType.TRANSLATION: ModelRoute(
        model="google/gemma-3-27b-it:free",
        max_tokens=1024,
        system_prompt="Translate accurately, preserving tone and nuance.",
    ),
    TaskType.SUMMARIZATION: ModelRoute(
        model="meta-llama/llama-4-maverick:free",
        max_tokens=512,
        system_prompt="Summarize concisely in bullet points.",
    ),
    TaskType.GENERAL: ModelRoute(
        model="qwen/qwen3-235b-a22b:free",
        max_tokens=1024,
        system_prompt="Answer helpfully and concisely.",
    ),
}

class FreeModelRouter:
    def __init__(self, client: OpenAI):
        self.client = client

    def ask(self, prompt: str, task: TaskType = TaskType.GENERAL) -> str:
        route = ROUTING_TABLE[task]
        res = self.client.chat.completions.create(
            model=route.model,
            messages=[
                {"role": "system", "content": route.system_prompt},
                {"role": "user", "content": prompt},
            ],
            max_tokens=route.max_tokens,
        )
        return res.choices[0].message.content

# 使い方
# router = FreeModelRouter(client)
# code = router.ask("Rustでフィボナッチ数列を実装して", TaskType.CODING)
# summary = router.ask(long_doc, TaskType.SUMMARIZATION)
```

---

## コスト比較: `:free` vs 有料モデル

以下は 2026 年 5 月時点での OpenRouter 公開料金表に基づく概算。

| モデル | Input ($/1M tok) | Output ($/1M tok) | 月 100 万出力トークンのコスト |
|---|---|---|---|
| Claude Opus 4 (有料) | ~$15 | ~$75 | ~$75 |
| Claude Sonnet 4 (有料) | ~$3 | ~$15 | ~$15 |
| GPT-4o (有料) | ~$2.5 | ~$10 | ~$10 |
| Qwen3-235B-A22B **:free** | $0 | $0 | **$0** |
| DeepSeek-R1 **:free** | $0 | $0 | **$0** |

> ⚠️ 有料モデルの価格は OpenRouter / 各プロバイダの公式ページで最新値を確認すること。`:free` モデルの提供状況・レート制限は変動する。

ただし「全部 `:free` でいい」わけではない。下表のような使い分けが現実的。

| ユースケース | 推奨 |
|---|---|
| コードレビューの最終判断・アーキテクチャ設計 | 有料 Sonnet / Opus |
| ドキュメント要約・翻訳・ラベリング | `:free` で十分 |
| プロトタイプのロジック検証 | `:free` で試してから有料へ |
| CI/CD の自動コメント・チェンジログ生成 | `:free` でコスト削減 |
| 本番ユーザー向けリアルタイム応答 | レート制限リスクあり → 有料 推奨 |

---

## よくある落とし穴と対処

### 1. `model not found` エラー

`:free` モデルは OpenRouter 側の在庫・提供状況で突然消えることがある。

**対処**: パターン 3 のフォールバックリストを常に 3 モデル以上維持する。

### 2. レスポンスが遅い

無料ティアは有料ティアより優先度が低いため、混雑時は数十秒待つことがある。

**対処**: `stream=True` でストリーミング受信し UX を改善する。バッチ処理なら非同期 + キューで並列化。

### 3. コンテキスト長が思ったより短い

モデルごとにコンテキスト長が異なる。長文を渡す前に必ず確認する。

```python
# OpenRouter の /models エンドポイントでスペック取得
import httpx, json

models = httpx.get(
    "https://openrouter.ai/api/v1/models",
    headers={"Authorization": f"Bearer {api_key}"}
).json()

for m in models["data"]:
    if ":free" in m["id"]:
        print(m["id"], "ctx:", m.get("context_length"))
```

### 4. 日本語品質のムラ

Qwen3 系・Llama 4 系は日本語がかなり安定している。Gemma 3 は英語・ヨーロッパ系が強く、日本語精度はやや落ちる傾向がある。**タスクと言語に合わせてモデルを選ぶ**こと。

---

## まとめ

- OpenRouter `:free` モデルは「使い捨て検証」「バッチ処理」「コスト削減が最優先のユースケース」で非常に有効
- Python / TypeScript どちらも OpenAI 互換 SDK がそのまま使えるため移行コストはほぼゼロ
- フォールバック + ルーティングを組み合わせれば、品質と可用性を保ちつつコストをほぼゼロに抑えられる
- ただし本番 SLA が必要な場面や、Opus / Sonnet クラスの品質が必要な場面では有料モデルを併用すること

LLM を「高価なブラックボックス」から「適材適所で使い分けるユーティリティ」に変える第一歩として、まず `:free` モデルで試してみることをお勧めする。

---

## 参考リンク

- [OpenRouter 公式ドキュメント](https://openrouter.ai/docs)
- [OpenRouter モデル一覧 (:free フィルタ)](https://openrouter.ai/models?q=:free)
- [openai-python GitHub](https://github.com/openai/openai-python)
- [openai-node GitHub](https://github.com/openai/openai-node)
- [Qwen3 技術レポート (Qwen 公式)](https://qwenlm.github.io/blog/qwen3/)
- [DeepSeek-R1 論文 (arXiv:2501.12948)](https://arxiv.org/abs/2501.12948)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
