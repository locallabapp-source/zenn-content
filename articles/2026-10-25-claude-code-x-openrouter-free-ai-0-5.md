---
title: "Claude Code × OpenRouter Free モデルで作る AI エージェント開発環境：月額ほぼ0円で始める5つのセットアップ手順"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Claude Code のサブエージェント呼び出しを OpenRouter の `:free` モデルにルーティングすれば、コスト激減
- 2026年時点で `:free` 枠で実用になるモデルが 10 本以上存在する
- `OPENROUTER_API_KEY` 1 本で Qwen3・Gemma3・Llama4 等を切り替え可能
- ローカル Ollama との使い分けパターンも合わせて紹介

---

## 背景：AI エージェント開発の「隠れコスト」問題

Claude Code や Cursor を使い始めると、最初に気づくのが **LLM 呼び出しコストの積み上がり速度** です。

メインの推論はともかく、「要約」「リスト抽出」「フォーマット変換」のような軽作業まで GPT-4o や Claude Sonnet を呼んでいると、エージェント 1 本あたり月 $30〜$60 に達することは珍しくありません。

OpenRouter の `:free` サフィックス付きモデルを使えば、こうした **補助的タスクのコストをほぼゼロに抑える**ことができます。本記事では、実際のセットアップ手順と使い分け戦略をまとめます。

---

## OpenRouter `:free` モデルとは

OpenRouter は複数の LLM API を統一エンドポイント (`https://openrouter.ai/api/v1`) で提供するプロキシサービスです。

モデル名に `/model-name:free` と付けると、**レート制限付きながら無料で推論できる**枠が利用できます（2026年5月現在）。

代表的な `:free` モデルの例：

| モデル名 (OpenRouter ID) | パラメータ規模 | 用途の目安 |
|---|---|---|
| `qwen/qwen3-8b:free` | 8B | 要約・分類・JSON 抽出 |
| `qwen/qwen3-30b-a3b:free` | 30B (MoE) | 軽量推論・コード補完 |
| `google/gemma-3-27b-it:free` | 27B | 翻訳・文体変換 |
| `meta-llama/llama-4-scout:free` | 17B active | 汎用タスク・RAG チャンク要約 |
| `mistralai/mistral-7b-instruct:free` | 7B | 高速分類・ルーティング判定 |
| `deepseek/deepseek-r1-0528:free` | 671B (MoE) | 複雑な推論・長文整理 |

> ⚠️ `:free` モデルは **RPM (requests per minute) / TPD (tokens per day) の制限**があります。公式 [OpenRouter Docs](https://openrouter.ai/docs) で最新の制限値を確認してください。

---

## セットアップ手順 5 ステップ

### ① OpenRouter アカウント作成と API キー発行

1. [openrouter.ai](https://openrouter.ai) でアカウント作成
2. ダッシュボード → **Keys** → **Create Key**
3. 生成されたキーを環境変数にセット

```bash
export OPENROUTER_API_KEY="sk-or-v1-xxxxxxxx"
```

クレジットカードを登録しなければ `:free` モデルのみ利用可能になります。課金上限を $5 に設定したうえで有料モデルも混在させる運用が実用的です。

---

### ② OpenAI 互換エンドポイントとして呼び出す

OpenRouter は OpenAI SDK の `base_url` を差し替えるだけで動きます。

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="sk-or-v1-xxxxxxxx",  # 実際は os.environ から取得
)

response = client.chat.completions.create(
    model="qwen/qwen3-8b:free",
    messages=[
        {"role": "user", "content": "以下の JSON を要約してください:\n{...}"}
    ],
    max_tokens=512,
)
print(response.choices[0].message.content)
```

`extra_headers` に `HTTP-Referer` と `X-Title` を付けるとダッシュボードでの識別が楽になります。

```python
response = client.chat.completions.create(
    model="qwen/qwen3-8b:free",
    messages=[...],
    extra_headers={
        "HTTP-Referer": "https://yourproject.example.com",
        "X-Title": "My Agent Pipeline",
    },
)
```

---

### ③ LangChain / LangGraph からの接続

LangChain の `ChatOpenAI` を使う場合：

```python
from langchain_openai import ChatOpenAI

llm_free = ChatOpenAI(
    model="qwen/qwen3-8b:free",
    openai_api_key="sk-or-v1-xxxxxxxx",
    openai_api_base="https://openrouter.ai/api/v1",
    temperature=0.2,
    max_tokens=1024,
)
```

LangGraph でエージェントノードを定義するとき、**重いノード (計画・最終回答) は Sonnet**、**軽いノード (中間要約・ルーティング判定) は `:free` モデル**に振り分けるパターンが費用対効果に優れます。

```python
from langgraph.graph import StateGraph

def router_node(state):
    # ルーティング判定は無料モデルで十分
    result = llm_free.invoke(state["messages"])
    return {"next": result.content.strip()}

def final_answer_node(state):
    # 最終回答生成は有料モデル
    result = llm_premium.invoke(state["messages"])
    return {"answer": result.content}
```

---

### ④ Ollama との使い分けパターン

**オフライン・低レイテンシ・プライベートデータ** を扱う場面では Ollama のローカルモデルが有効です。

| 条件 | 推奨モデル |
|---|---|
| GPU なし・CPU only | Ollama `qwen3:0.6b` / `gemma3:1b` |
| GPU 8GB (RTX 3070 等) | Ollama `qwen3:8b` |
| インターネット可・無料でよい | OpenRouter `:free` |
| 高精度・最終出力 | OpenRouter 有料 or Anthropic direct |

Ollama も OpenAI 互換エンドポイントを持つため、`base_url` を `http://localhost:11434/v1` に変えるだけで同じコードが動きます。

```python
llm_local = ChatOpenAI(
    model="qwen3:8b",
    openai_api_key="ollama",  # ダミー文字列で OK
    openai_api_base="http://localhost:11434/v1",
)
```

**ルーティングロジックの実装例**（環境変数で切り替え）：

```python
import os

INFERENCE_BACKEND = os.getenv("INFERENCE_BACKEND", "openrouter_free")

def get_cheap_llm():
    if INFERENCE_BACKEND == "ollama":
        return ChatOpenAI(
            model="qwen3:8b",
            openai_api_key="ollama",
            openai_api_base="http://localhost:11434/v1",
        )
    # デフォルト: OpenRouter free
    return ChatOpenAI(
        model="qwen/qwen3-8b:free",
        openai_api_key=os.environ["OPENROUTER_API_KEY"],
        openai_api_base="https://openrouter.ai/api/v1",
    )
```

---

### ⑤ レート制限への対処：指数バックオフ実装

`:free` モデルは RPM 制限が低いため、並列エージェントを走らせると `429 Too Many Requests` が頻発します。

`tenacity` ライブラリで指数バックオフを実装するのが定石です。

```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type,
)
from openai import RateLimitError

@retry(
    retry=retry_if_exception_type(RateLimitError),
    wait=wait_exponential(multiplier=1, min=2, max=60),
    stop=stop_after_attempt(6),
)
def call_free_model(messages: list[dict]) -> str:
    response = client.chat.completions.create(
        model="qwen/qwen3-8b:free",
        messages=messages,
        max_tokens=512,
    )
    return response.choices[0].message.content
```

並列度を上げたい場合は、**タスクをキューに入れて `asyncio.Semaphore` で同時呼び出し数を制限**する方法が有効です。

```python
import asyncio
from openai import AsyncOpenAI

async_client = AsyncOpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key=os.environ["OPENROUTER_API_KEY"],
)

semaphore = asyncio.Semaphore(3)  # 同時 3 リクエストまで

async def safe_call(messages: list[dict]) -> str:
    async with semaphore:
        response = await async_client.chat.completions.create(
            model="qwen/qwen3-8b:free",
            messages=messages,
        )
        return response.choices[0].message.content
```

---

## モデル品質の実用的な評価軸

「無料だから品質が低い」とは限りません。用途別に評価軸を変えることが大切です。

### JSON 抽出・構造化タスク

`qwen3-8b:free` は **Function Calling / JSON Mode** に対応しており、プロンプトに `response_format={"type": "json_object"}` を指定すると安定した JSON を返します。

```python
response = client.chat.completions.create(
    model="qwen/qwen3-8b:free",
    messages=[
        {
            "role": "system",
            "content": "ユーザーの入力から以下のスキーマで JSON を返してください。",
        },
        {"role": "user", "content": "東京都渋谷区で働くエンジニア、28歳"},
    ],
    response_format={"type": "json_object"},
)
```

### 長文要約

`deepseek/deepseek-r1-0528:free` は 128K コンテキストを持ち、長い技術ドキュメントの要約に使えます。ただしレイテンシが高め (5〜15 秒) なため、リアルタイム性が不要なバッチ処理向きです。

### コード生成

軽微な補完・変数名提案程度なら `qwen3-30b-a3b:free` (MoE 30B) が実用域。複雑なアーキテクチャ設計には有料モデルを使う「二段構え」が費用対効果の観点から合理的です。

---

## よくある落とし穴と対処法

### 落とし穴 1：`model not found` エラー

`:free` モデルは**提供・廃止のサイクルが速い**です。コードに直書きせず、設定ファイルで管理しましょう。

```yaml
# config/models.yaml
cheap_model: "qwen/qwen3-8b:free"
premium_model: "anthropic/claude-sonnet-4"
fallback_model: "google/gemma-3-27b-it:free"
```

### 落とし穴 2：コンテキスト長オーバー

`:free` モデルは有料版より最大コンテキストが短い場合があります。大きなペイロードを渡す前に **トークン数を事前チェック**する処理を挟みましょう。

```python
import tiktoken

def truncate_to_limit(text: str, limit: int = 6000) -> str:
    enc = tiktoken.get_encoding("cl100k_base")
    tokens = enc.encode(text)
    if len(tokens) > limit:
        return enc.decode(tokens[:limit])
    return text
```

### 落とし穴 3：思考トークン (`<think>...</think>`) の混入

Qwen3 や DeepSeek R1 系は**思考プロセスをレスポンスに含める**ことがあります。パースする前に除去しましょう。

```python
import re

def strip_thinking(text: str) -> str:
    return re.sub(r"<think>.*?</think>", "", text, flags=re.DOTALL).strip()
```

---

## コスト試算：1 日 500 回呼び出しの比較

| 構成 | 1 日あたりコスト (概算) |
|---|---|
| 全呼び出しを Claude Sonnet | $3.50〜$7.00 |
| 補助タスクを `:free` に切り替え (7 割) | $1.05〜$2.10 |
| 補助タスクを Ollama ローカル (7 割) | $0.35〜$0.70 (電気代のみ) |
| OpenRouter `:free` (全量) | $0 ※レート制限内 |

実際のプロジェクトでは「補助タスク 70% を `:free` にルーティング + 最終回答 30% を有料モデル」というハイブリッド構成が現実的かつ費用対効果が高いです。

---

## まとめ

- OpenRouter `:free` モデルは**補助タスクのコスト圧縮**に非常に有効
- OpenAI SDK の `base_url` を変えるだけで接続でき、**LangChain / LangGraph とも相性が良い**
- レート制限対策に `tenacity` の指数バックオフと `asyncio.Semaphore` は必須
- Ollama ローカルと OpenRouter を `INFERENCE_BACKEND` 環境変数で切り替える設計が保守しやすい
- `:free` モデルのリスト・制限値は変動するため、**設定ファイル外出し + 定期見直し**を推奨

AI エージェント開発の初期フェーズで「とにかく動く試作を作りたい」場面では、OpenRouter `:free` モデルはほぼベストの選択肢です。ぜひ試してみてください。

---

## 参考リンク

- [OpenRouter 公式ドキュメント](https://openrouter.ai/docs)
- [OpenRouter モデル一覧](https://openrouter.ai/models)
- [LangChain ChatOpenAI ドキュメント](https://python.langchain.com/docs/integrations/chat/openai/)
- [tenacity — Python リトライライブラリ](https://github.com/jd/tenacity)
- [Ollama 公式サイト](https://ollama.com)
- [Qwen3 技術レポート (Hugging Face)](https://huggingface.co/Qwen/Qwen3-8B)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
