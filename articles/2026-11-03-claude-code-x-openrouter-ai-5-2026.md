---
title: "Claude Code × OpenRouter で実現する \"ゼロコスト AI 開発\"：無料モデル 5 選と使い分け完全ガイド 2026"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- OpenRouter の `:free` エンドポイントを使えば **API コスト $0** で本格的な LLM 開発ができる
- Claude Code の `ANTHROPIC_API_KEY` を OpenRouter に差し替えるだけで即日移行可能
- 無料モデルには **レート制限・コンテキスト上限** があるため、タスク特性に合わせた使い分けが重要
- 2026 年時点で実用レベルの無料モデルは 5 つ選定できる

---

## なぜ無料モデルを真剣に使うか

LLM の開発コストは気付いたら膨らむ。プロトタイプを量産する段階・CI でのコード自動レビュー・夜間バッチでの要約処理——こうした **"作って壊す"フェーズ** に月数万円のトークン費用を払い続けるのは非合理だ。

OpenRouter は 2024 年末から各プロバイダーの無料枠を一本化した `:free` サフィックス型のルーティングを提供し始めた。2026 年現在、商用プロジェクトでも十分使える品質のモデルが複数ラインナップに並んでいる。

本記事では **実際の開発ワークフローへの組み込み方** を中心に解説する。

---

## OpenRouter `:free` モデルの仕組み

### エンドポイント

```
https://openrouter.ai/api/v1/chat/completions
```

OpenAI 互換 API なので、`openai` Python ライブラリや Vercel AI SDK がそのまま使える。

### モデル ID の命名規則

```
<provider>/<model-name>:free
```

例:
- `google/gemma-3-27b-it:free`
- `meta-llama/llama-3.3-70b-instruct:free`
- `mistralai/mistral-7b-instruct:free`
- `qwen/qwen3-14b:free`
- `deepseek/deepseek-r1-distill-qwen-32b:free`

`:free` を外すと有料版に切り替わる。**同一コードで切り替えられる**のが最大のメリットだ。

### 無料枠の制約 (2026 年 5 月時点)

| 制限項目 | 一般的な値 |
|---|---|
| RPM (Requests per Minute) | 10〜20 |
| TPM (Tokens per Minute) | 10,000〜40,000 |
| コンテキスト長 | モデル依存 (4K〜128K) |
| 1 日あたりリクエスト上限 | 無制限〜200 (モデルによる) |

RPM が低いため、**並列リクエストを大量に飛ばすユースケースには向かない**。順次処理やバッチ間隔を設けた設計が必要。

---

## 2026 年版 無料モデル 5 選

### 1. `meta-llama/llama-3.3-70b-instruct:free`

**得意領域: コード生成・汎用タスク・日本語**

Meta の Llama 3.3 70B は、2024 年末に公開されて以来 OSS 最高峰のひとつとして評価が定着している。70B パラメータながら量子化推論の恩恵で OpenRouter 上でも安定して動く。

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="<YOUR_OPENROUTER_KEY>",
)

response = client.chat.completions.create(
    model="meta-llama/llama-3.3-70b-instruct:free",
    messages=[
        {
            "role": "user",
            "content": "Python で CSV を読み込んで JSON に変換するスクリプトを書いて"
        }
    ],
)
print(response.choices[0].message.content)
```

**実測値の目安**:
- 日本語質問応答: GPT-4o mini と同等〜やや下
- コード生成 (HumanEval ベース): 70〜75% 通過率
- レイテンシ: 2〜6 秒 (トークン数依存)

---

### 2. `google/gemma-3-27b-it:free`

**得意領域: 長文要約・RAG・多言語**

Google の Gemma 3 シリーズは 128K コンテキストをサポートし、**長文処理が無料モデル中トップクラス**。PDF 全文を詰め込んでサマリーを取る用途では他を圧倒する。

Instruction Tuning 版 (`-it`) はチャット形式に最適化されており、システムプロンプトへの追随が安定している。

```python
response = client.chat.completions.create(
    model="google/gemma-3-27b-it:free",
    messages=[
        {
            "role": "system",
            "content": "あなたは技術文書のサマライザーです。要点を箇条書きで返してください。"
        },
        {
            "role": "user",
            "content": long_document_text  # 数万字でも OK
        }
    ],
    max_tokens=1024,
)
```

**注意点**: Gemma 3 は商用利用に **Gemma Terms of Use** の同意が必要。MMS ライセンスではないが、独自条項がある。OpenRouter 経由でも利用規約は Google のものが適用される点を忘れずに。

---

### 3. `qwen/qwen3-14b:free`

**得意領域: 思考推論・数学・コード**

Alibaba が 2025 年にリリースした Qwen3 シリーズ。14B モデルでありながら **thinking モード**（`/think` トークンによる Chain-of-Thought）を内蔵しており、推論問題で他の 14B クラスを大きく上回る。

無料モデルの中では **コストパフォーマンスが最高峰**と言っても過言ではない。

```python
response = client.chat.completions.create(
    model="qwen/qwen3-14b:free",
    messages=[
        {
            "role": "user",
            # /think プレフィックスで思考モードを明示的に ON
            "content": "/think\nこの SQL クエリのパフォーマンスを改善する方法を考えてください:\nSELECT * FROM orders WHERE status = 'pending' ORDER BY created_at"
        }
    ],
)
```

thinking モードでは `<think>...</think>` ブロックが出力に含まれるため、パースが必要:

```python
import re

raw = response.choices[0].message.content
thinking = re.findall(r'<think>(.*?)</think>', raw, re.DOTALL)
answer = re.sub(r'<think>.*?</think>', '', raw, flags=re.DOTALL).strip()
```

---

### 4. `deepseek/deepseek-r1-distill-qwen-32b:free`

**得意領域: 数学・アルゴリズム問題・コード補完**

DeepSeek R1 の蒸留版。フルの R1 (671B MoE) を 32B に蒸留しているにもかかわらず、**MATH/AIME ベンチマークでは GPT-4o に匹敵**するスコアを出す。

アルゴリズム設計・競技プログラミング的なコード生成において無料モデル中最強クラス。

```python
response = client.chat.completions.create(
    model="deepseek/deepseek-r1-distill-qwen-32b:free",
    messages=[
        {
            "role": "user",
            "content": "グラフの最短経路をダイクストラ法で実装してください (Python, 型ヒント付き)"
        }
    ],
    # R1 系は長めに思考するので max_tokens は余裕を持たせる
    max_tokens=4096,
)
```

**注意**: DeepSeek モデルは利用規約上 **中国法の管轄**が含まれる。個人情報・機密情報を入力しないことを強く推奨する。ダミーデータ・汎用コード生成用途に限定するのが安全。

---

### 5. `mistralai/mistral-7b-instruct:free`

**得意領域: 軽量・高速・安定性**

Mistral 7B は 2023 年の登場から変わらず「小さく速く使える」という評価が定着している。**RPM 制限を意識したリトライ処理の試験用途**や、高頻度・軽量な分類タスク (ラベリング / 感情分析 / 短文要約) に向いている。

レイテンシが 5 モデル中最短で、CI パイプライン組み込みに最適:

```python
import time
from openai import OpenAI, RateLimitError

def classify_with_retry(text: str, max_retries: int = 3) -> str:
    client = OpenAI(
        base_url="https://openrouter.ai/api/v1",
        api_key="<YOUR_OPENROUTER_KEY>",
    )
    for attempt in range(max_retries):
        try:
            res = client.chat.completions.create(
                model="mistralai/mistral-7b-instruct:free",
                messages=[
                    {
                        "role": "user",
                        "content": f"次のテキストを positive/negative/neutral で分類: {text}"
                    }
                ],
                max_tokens=16,
            )
            return res.choices[0].message.content.strip().lower()
        except RateLimitError:
            wait = 2 ** attempt
            print(f"Rate limited. Waiting {wait}s...")
            time.sleep(wait)
    raise RuntimeError("Max retries exceeded")
```

---

## モデル選定チートシート

| ユースケース | 推奨モデル | 理由 |
|---|---|---|
| コード生成 (汎用) | Llama 3.3 70B | 70B の安定した推論力 |
| 長文要約・RAG | Gemma 3 27B | 128K コンテキスト |
| 数学・アルゴリズム | DeepSeek R1 Distill 32B | R1 蒸留の推論能力 |
| 思考推論・CoT | Qwen3 14B | thinking モード内蔵 |
| 高速・軽量分類 | Mistral 7B | 速度とシンプルさ |

---

## Claude Code と組み合わせる実践パターン

Claude Code 環境では `ANTHROPIC_API_KEY` の代わりに OpenRouter を指定できる。`claude` CLI または VS Code 拡張の設定で Base URL を差し替える方法だ。

ただし **Claude Code のコア (Sonnet / Opus) は Anthropic 直接接続**が前提のため、OpenRouter 経由での Claude モデル利用はレート差・挙動差が出ることがある。

**実用的な棲み分け**:

```
Claude Code (Sonnet)  →  対話・設計・複雑なリファクタ
OpenRouter :free      →  バッチ処理・自動化スクリプト内の補助 LLM
```

たとえば「大量のコミットメッセージを要約して変更履歴 md を生成する」バッチなら OpenRouter :free で十分。コスト $0 で 1 日 100 件処理できる。

```python
# git log を読んで changelog を自動生成する例 (概念コード)
import subprocess
from openai import OpenAI

def generate_changelog(since_tag: str) -> str:
    # git log から commit メッセージ取得
    result = subprocess.run(
        ["git", "log", f"{since_tag}..HEAD", "--oneline"],
        capture_output=True, text=True
    )
    commits = result.stdout.strip()

    client = OpenAI(
        base_url="https://openrouter.ai/api/v1",
        api_key="<YOUR_OPENROUTER_KEY>",
    )
    response = client.chat.completions.create(
        model="meta-llama/llama-3.3-70b-instruct:free",
        messages=[
            {
                "role": "system",
                "content": "Git コミット一覧から Keep a Changelog 形式の CHANGELOG.md を生成してください。"
            },
            {"role": "user", "content": commits}
        ],
    )
    return response.choices[0].message.content
```

---

## レート制限に強い実装パターン

無料モデルは RPM が低い。**指数バックオフ + ジッター**でレート制限エラーを吸収する:

```python
import random
import time
from openai import RateLimitError

def call_with_backoff(client, model: str, messages: list, **kwargs):
    """指数バックオフ付きリクエスト"""
    base_delay = 1.0
    max_delay = 60.0
    max_attempts = 6

    for attempt in range(max_attempts):
        try:
            return client.chat.completions.create(
                model=model,
                messages=messages,
                **kwargs
            )
        except RateLimitError as e:
            if attempt == max_attempts - 1:
                raise
            # ジッター付き指数バックオフ
            delay = min(base_delay * (2 ** attempt) + random.uniform(0, 1), max_delay)
            print(f"Rate limited (attempt {attempt + 1}). Retry in {delay:.1f}s")
            time.sleep(delay)
```

バッチ処理では `asyncio.Semaphore` で同時実行数を制限するのも有効:

```python
import asyncio
from openai import AsyncOpenAI

async def process_batch(items: list[str], concurrency: int = 3):
    client = AsyncOpenAI(
        base_url="https://openrouter.ai/api/v1",
        api_key="<YOUR_OPENROUTER_KEY>",
    )
    semaphore = asyncio.Semaphore(concurrency)

    async def process_one(item: str) -> str:
        async with semaphore:
            res = await client.chat.completions.create(
                model="qwen/qwen3-14b:free",
                messages=[{"role": "user", "content": item}],
            )
            await asyncio.sleep(0.5)  # RPM バッファ
            return res.choices[0].message.content

    return await asyncio.gather(*[process_one(item) for item in items])
```

---

## モデル切り替えをコード変更ゼロにする設計

`.env` にモデル名を外出しし、フォールバックチェーンを構成する:

```python
# config.py
import os

MODEL_CHAIN = os.getenv(
    "LLM_MODEL_CHAIN",
    "meta-llama/llama-3.3-70b-instruct:free,"
    "qwen/qwen3-14b:free,"
    "mistralai/mistral-7b-instruct:free"
).split(",")
```

```python
# llm_client.py
from openai import OpenAI, RateLimitError
from config import MODEL_CHAIN

def chat(messages: list, **kwargs) -> str:
    client = OpenAI(
        base_url="https://openrouter.ai/api/v1",
        api_key=os.environ["OPENROUTER_API_KEY"],
    )
    last_error = None
    for model in MODEL_CHAIN:
        try:
            res = client.chat.completions.create(
                model=model.strip(),
                messages=messages,
                **kwargs
            )
            return res.choices[0].message.content
        except RateLimitError as e:
            last_error = e
            continue  # 次のモデルにフォールバック
    raise last_error
```

このパターンにより **プライマリモデルがレート制限に達したら自動でセカンダリへ** 切り替わる。コスト $0 を維持しつつ可用性を上げられる。

---

## ライセンス早見表

| モデル | ライセンス | 商用利用 | 注意点 |
|---|---|---|---|
| Llama 3.3 70B | Meta Llama 3 Community License | ✅ (月間 7 億 MAU 未満) | 月 7 億超は Meta に要問い合わせ |
| Gemma 3 27B | Gemma Terms of Use | ✅ | Google の独自規約・コンテンツポリシー要確認 |
| Qwen3 14B | Apache 2.0 | ✅ | 制限なし |
| DeepSeek R1 Distill 32B | MIT | ✅ | 入力データの取り扱いに注意 (中国法管轄) |
| Mistral 7B | Apache 2.0 | ✅ | 制限なし |

---

## まとめ

OpenRouter の `:free` モデルは「無料だから妥協する」ものではなく、**タスクに合わせて使えば有料モデルと遜色ない結果**を出せる。

- **コード生成**: Llama 3.3 70B
- **長文処理**: Gemma 3 27B
- **推論・数学**: DeepSeek R1 Distill
- **CoT 思考**: Qwen3 14B
- **高速・軽量**: Mistral 7B

この 5 本柱をフォールバックチェーンに組み込み、Claude Code (有料) との役割分担を明確にすれば、**開発コストを大幅に圧縮しつつ生産性を維持**できる。

日々の自動化バッチ・CI 補助・プロトタイピングは `:free` で回し、設計判断・複雑なリファクタリング・顧客向け成果物にのみ有料モデルを充てる——この棲み分けが 2026 年の AI 開発コスト最適化の基本戦略だ。

---

## 参考リンク

- [OpenRouter Documentation](https://openrouter.ai/docs)
- [OpenRouter Model List](https://openrouter.ai/models)
- [Meta Llama 3 Community License](https://llama.meta.com/llama3/license/)
- [Gemma Terms of Use](https://ai.google.dev/gemma/terms)
- [Qwen3 Technical Report (arXiv)](https://arxiv.org/abs/2505.09388)
- [DeepSeek R1 Paper (arXiv)](https://arxiv.org/abs/2501.12948)
- [Mistral 7B Paper (arXiv)](https://arxiv.org/abs/2310.06825)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
