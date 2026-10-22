---
title: "Claude Code + OpenRouter 無料モデル 5 選を徹底比較 — コスト 0 円で AI エージェントを回す 2026 年版"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Claude Code のサブエージェントに `:free` モデルを渡せば API コストをほぼゼロにできる
- 2026 年 5 月時点で OpenRouter に存在する主要 `:free` モデル 5 種を速度・品質・レート制限で比較
- 「重い思考 → Sonnet / 軽い補助タスク → :free モデル」の二段構えが現実的な最適解

---

## 背景

Claude Code をローカル開発環境に組み込んでいると、必然的に悩むのが **LLM コスト** だ。  
1 日に数十回 MR を作成するような自動化ワークフローを組むと、Claude 3.5 Sonnet の API 費用が想定外に膨らむことがある。

そこで注目したいのが **OpenRouter の `:free` サフィックスモデル**。  
モデル ID の末尾に `/free` がついたモデルは、レート制限はあるものの API コスト 0 円で利用できる。  
Claude Code から `ANTHROPIC_BASE_URL` や `OPENAI_BASE_URL` を OpenRouter のエンドポイントに向けることで、サブエージェントの一部タスクをこれらに委譲できる。

本記事では 2026 年 5 月時点の主要 5 モデルを実際に触れた感覚も交えながら比較する。

> ⚠️ `:free` モデルのレート制限・提供状況は変動します。最新情報は [openrouter.ai/models](https://openrouter.ai/models) を確認してください。

---

## OpenRouter `:free` モデルとは

OpenRouter は複数の LLM プロバイダへのリクエストを一本化するプロキシサービスだ。  
多くのモデルは per-token 課金だが、一部のモデルは提供元 (Meta / Google / Mistral / DeepSeek 等) のリソース提供やプロモーションの形で無料枠として公開されている。

利用条件:

| 項目 | 内容 |
|---|---|
| レート制限 | 概ね 20 req/min・200 req/day (モデルにより異なる) |
| コンテキスト長 | モデル依存 (4K〜128K) |
| 認証 | OpenRouter API キー 1 つで全モデルアクセス |
| SLA | なし (ベストエフォート) |

本番 SaaS の critical path には向かないが、**CIのドキュメント自動生成・コードレビュー補助・翻訳・要約** といった非同期バッチタスクには十分機能する。

---

## 比較対象 5 モデル

2026 年 5 月時点で `:free` として利用可能な代表的なモデルを選定した。

| # | モデル ID (OpenRouter) | ベース | コンテキスト |
|---|---|---|---|
| 1 | `meta-llama/llama-3.3-70b-instruct:free` | Llama 3.3 70B | 128K |
| 2 | `google/gemini-2.0-flash-exp:free` | Gemini 2.0 Flash | 1M |
| 3 | `deepseek/deepseek-r1:free` | DeepSeek R1 | 64K |
| 4 | `mistralai/mistral-7b-instruct:free` | Mistral 7B | 32K |
| 5 | `qwen/qwen3-8b:free` | Qwen3 8B | 32K |

---

## 各モデルの特徴と使いどころ

### 1. Llama 3.3 70B Instruct `:free`

Meta の Llama 3.3 70B は、オープンウェイトモデルの中では 2025 年末時点で最高クラスの汎用性を持つ。  
instruction-following の精度が高く、**コードレビューコメントの生成・README の自動補完・Git コミットメッセージ生成**に向いている。

```python
# OpenAI 互換 API 経由で呼び出す例
import openai

client = openai.OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="sk-or-...",
)

response = client.chat.completions.create(
    model="meta-llama/llama-3.3-70b-instruct:free",
    messages=[
        {"role": "user", "content": "次の Python 関数に docstring を追加してください:\ndef add(a, b): return a + b"}
    ],
)
print(response.choices[0].message.content)
```

**強み**: 70B の推論品質・長文コンテキスト 128K・instruction-following 安定  
**弱み**: レスポンスが若干遅い (70B ゆえ)・レート制限に引っかかりやすい

---

### 2. Gemini 2.0 Flash Experimental `:free`

Google の Gemini 2.0 Flash は **コンテキスト長 1M トークン**という圧倒的なスペックが武器。  
長大なコードベース全体を渡してのバグ解析や、複数ファイルにまたがる差分レビューに向いている。

Experimental モデルのため品質にムラがあるが、コンテキスト長を生かしたユースケースでは他モデルの追随を許さない。

**使いどころ**:
- リポジトリ全体の `README.md` 一括生成
- 長文ログ解析
- 大量の YAML/JSON の lint + 修正提案

**強み**: 1M コンテキスト・マルチモーダル対応  
**弱み**: Experimental ゆえの不安定さ・レート制限が厳しいことがある

---

### 3. DeepSeek R1 `:free`

DeepSeek R1 は思考連鎖 (Chain-of-Thought) に特化したモデルで、**数学・アルゴリズム問題・ロジック検証**に驚くほど強い。  
無料モデルの中では「最も思考する」モデルと言っても過言ではない。

OpenRouter を通じた R1 `:free` はレスポンスに `<think>...</think>` タグが含まれることが多い。  
このタグを除去してから使うと出力がすっきりする。

```python
import re

def strip_thinking(text: str) -> str:
    """DeepSeek R1 の <think> タグを除去する"""
    return re.sub(r"<think>.*?</think>", "", text, flags=re.DOTALL).strip()
```

**強み**: アルゴリズム推論・バグ原因特定・テスト設計  
**弱み**: 思考時間が長い・日本語の自然さはやや劣る

---

### 4. Mistral 7B Instruct `:free`

Mistral 7B は軽量・高速・低レイテンシという特性を持つ。  
`:free` モデルの中では **最もレスポンスが速い**部類で、インタラクティブな補助 (型定義の補完・変数名提案・コメント翻訳) に向く。

2024 年リリースの古いモデルだが、7B という小ささゆえにレート制限に引っかかりにくい。  
「とにかく返答を速く返してほしい処理」に使い続けられる安定した選択肢。

**強み**: 低レイテンシ・安定性  
**弱み**: 長文・複雑な推論は苦手・コンテキスト 32K

---

### 5. Qwen3 8B `:free`

Alibaba が 2025 年にリリースした Qwen3 シリーズの 8B モデル。  
**多言語対応 (日本語・中国語・英語)** が優秀で、国際向けコンテンツ生成や多言語ドキュメント翻訳に向いている。

Thinking Mode (QwQ 系の思考連鎖) と Non-Thinking Mode を動的に切り替えられる設計になっており、  
`/no_think` を最初に指定するとシンプルな応答モードになる。

```
# Non-Thinking モードで呼び出す
system: "/no_think あなたは日英翻訳アシスタントです。"
user: "以下の文章を英語に翻訳してください: ..."
```

**強み**: 多言語・Thinking/Non-Thinking 切り替え  
**弱み**: 8B ゆえの限界・複雑な推論はDeepSeek R1 に劣る

---

## 総合比較表

| モデル | 速度 | 日本語品質 | コード品質 | 推論力 | コンテキスト |
|---|---|---|---|---|---|
| Llama 3.3 70B | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 128K |
| Gemini 2.0 Flash Exp | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | **1M** |
| DeepSeek R1 | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 64K |
| Mistral 7B | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ | 32K |
| Qwen3 8B | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | 32K |

---

## Claude Code との組み合わせ方

Claude Code は `CLAUDE.md` の設定に加え、bash tool 経由で任意の API を呼び出せる。  
`:free` モデルを補助タスクに使う設計パターンを紹介する。

### パターン A: 役割分担型

```
重いタスク (設計判断・複雑なリファクタ)
  → Claude 3.5 Sonnet (高品質)

軽いタスク (コメント生成・型補完・翻訳・要約)
  → OpenRouter :free モデル (コスト 0)
```

これだけで 1 日の API コストを 30〜50% 削減できるケースがある。

### パターン B: 並列検証型

同一のコードレビュープロンプトを 3 モデルに投げ、多数決で採用案を決める。  
`:free` モデルを並列に叩いてもコストはゼロなので、積極的に使える。

```python
import asyncio
import openai

FREE_MODELS = [
    "meta-llama/llama-3.3-70b-instruct:free",
    "deepseek/deepseek-r1:free",
    "qwen/qwen3-8b:free",
]

async def review_with_model(client, model: str, code: str) -> str:
    resp = await client.chat.completions.acreate(
        model=model,
        messages=[{"role": "user", "content": f"次のコードをレビューしてください:\n```\n{code}\n```"}],
    )
    return resp.choices[0].message.content

async def ensemble_review(code: str) -> list[str]:
    client = openai.AsyncOpenAI(
        base_url="https://openrouter.ai/api/v1",
        api_key="sk-or-...",
    )
    results = await asyncio.gather(
        *[review_with_model(client, m, code) for m in FREE_MODELS]
    )
    return list(results)
```

### パターン C: フォールバック型

Sonnet が高負荷 or 予算上限に達したときの自動フォールバック先として `:free` モデルを設定する。

```python
def get_client_with_fallback(primary="anthropic/claude-3.5-sonnet", fallback="meta-llama/llama-3.3-70b-instruct:free"):
    """予算上限エラー時に自動で :free モデルへ切り替える"""
    # ... 実装は OpenRouter の x-budget-token ヘッダや例外ハンドリングで実現
    pass
```

---

## 注意点・ハマりどころ

### 1. レート制限は daily リセット

`:free` モデルは一般的に 200 req/day 前後の制限がある。  
並列で叩きすぎると昼過ぎに枯渇するので、exponential backoff + キュー管理は必須。

```python
import time
import random

def call_with_backoff(fn, max_retries=5):
    for attempt in range(max_retries):
        try:
            return fn()
        except Exception as e:
            if "rate limit" in str(e).lower():
                wait = (2 ** attempt) + random.uniform(0, 1)
                time.sleep(wait)
            else:
                raise
    raise RuntimeError("Max retries exceeded")
```

### 2. `:free` モデルは予告なく廃止される

OpenRouter 上の `:free` モデルはプロバイダ都合で突然終了することがある。  
利用するモデル ID をハードコードせず、設定ファイルや環境変数で切り替えられるようにしておくこと。

### 3. `X-Title` ヘッダで識別

OpenRouter のダッシュボードでどのアプリからのリクエストか識別するために `X-Title` ヘッダを付けると管理しやすい。

```python
client = openai.OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="sk-or-...",
    default_headers={
        "X-Title": "MyApp ReviewBot",
    },
)
```

---

## まとめ

| ユースケース | おすすめモデル |
|---|---|
| コードレビュー・ドキュメント生成 | Llama 3.3 70B |
| 巨大ファイル・ログ解析 | Gemini 2.0 Flash Exp |
| バグ原因の論理的特定 | DeepSeek R1 |
| 高速な補完・変数名提案 | Mistral 7B |
| 多言語コンテンツ・日本語翻訳 | Qwen3 8B |

Claude Code + OpenRouter `:free` モデルの組み合わせは「コスト 0 で AI エージェントを回す」現実的な選択肢だ。  
ただし SLA なし・レート制限あり・廃止リスクありという性質を理解した上で、**非クリティカルなバッチタスクに限定して使う**のが安全な運用方針だろう。

無料モデルの品質は年々上がっており、来年には今回紹介したモデルがすべて入れ替わっている可能性もある。  
OpenRouter の [models ページ](https://openrouter.ai/models?q=free) を定期的にウォッチして、ベストな `:free` モデルを使い続けてほしい。

---

## 参考リンク

- [OpenRouter 公式ドキュメント](https://openrouter.ai/docs)
- [OpenRouter Models 一覧](https://openrouter.ai/models)
- [Llama 3.3 リリースノート (Meta)](https://ai.meta.com/blog/llama-3/)
- [DeepSeek R1 技術レポート (DeepSeek)](https://github.com/deepseek-ai/DeepSeek-R1)
- [Qwen3 技術ブログ (Alibaba Cloud)](https://qwenlm.github.io/blog/qwen3/)
- [Gemini 2.0 Flash 概要 (Google)](https://deepmind.google/technologies/gemini/flash/)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
