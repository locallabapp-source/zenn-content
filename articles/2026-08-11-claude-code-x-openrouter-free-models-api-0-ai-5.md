---
title: "Claude Code × OpenRouter Free Models：API コスト ¥0 で AI エージェント開発を回す5つの設計パターン"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- OpenRouter の `:free` モデルは 2026 年時点で実用的なクオリティのモデルが複数存在する
- Claude Code から OpenRouter 経由でモデルを叩く際、コスト・レート制限・フォールバックを設計しないと詰まる
- 本記事では「無料モデルを最大限活用しながらエージェント品質を落とさない」5 つの設計パターンを解説する

---

## 背景：AI エージェント開発のコスト問題

AI エージェントを自社プロダクトに組み込む際、真っ先に壁になるのが **LLM の API コスト**だ。

Claude Sonnet や GPT-4o をそのまま使うと、1 エージェント・1 日の試験運用でも数千〜数万トークンが飛ぶ。開発フェーズで累積すると月の AWS 費用より LLM 代の方が高い、という逆転現象は決して珍しくない。

OpenRouter は複数 LLM プロバイダへの統一エンドポイントを提供するサービスで、**モデル名の末尾に `:free` を付けるだけで無料枠のモデルに切り替えられる**という特徴がある。

```
# 有料モデル
meta-llama/llama-3.3-70b-instruct

# 同等の無料枠モデル
meta-llama/llama-3.3-70b-instruct:free
```

2026 年時点では Llama 3.3 70B, Qwen3 32B, Mistral 7B Instruct, Gemma 3 27B など、ベンチマーク上はかなり実用的なモデルが `:free` で利用できる。

ただし「無料だから全部これでいい」と考えると、**レート制限・コンテキスト長の制約・応答品質のバラつき**で詰まる。本記事ではそのギャップを埋める 5 つの設計パターンを解説する。

---

## 前提：OpenRouter の仕様おさらい

OpenRouter は OpenAI 互換 API として動作する。エンドポイントは以下の通り。

```
https://openrouter.ai/api/v1/chat/completions
```

リクエスト構造は OpenAI の `/v1/chat/completions` と同一なので、既存の `openai` SDK がそのまま使える。

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="<your-openrouter-api-key>",
)

response = client.chat.completions.create(
    model="meta-llama/llama-3.3-70b-instruct:free",
    messages=[{"role": "user", "content": "Hello!"}],
)
print(response.choices[0].message.content)
```

`:free` モデルの主な制約:

| 制約項目 | 目安 |
|---|---|
| RPM (Requests Per Minute) | 10〜20 RPM (モデルによる) |
| TPM (Tokens Per Minute) | 20,000〜100,000 TPM |
| コンテキスト長 | 有料版より短い場合あり (8K〜32K が多い) |
| 優先度 | 有料リクエストより低い (高負荷時に遅延) |

これらを踏まえた上で、次のパターンを設計する。

---

## パターン 1：タスク難易度でモデルを tier 分け

「全タスクを同一モデルに投げる」のは非効率だ。**単純タスクは無料 7B、複雑タスクは有料 Sonnet** という tier 分けをルール化することで、コストを最小化しながら品質を保てる。

```python
TASK_MODEL_MAP = {
    "summarize":    "google/gemma-3-27b-it:free",   # 要約：無料 27B で十分
    "classify":     "mistralai/mistral-7b-instruct:free",  # 分類：7B で高速
    "code_review":  "qwen/qwen3-32b:free",          # コードレビュー：32B 推奨
    "architecture": "anthropic/claude-sonnet-4-5",  # 設計判断：有料 Sonnet
}

def call_llm(task_type: str, prompt: str) -> str:
    model = TASK_MODEL_MAP.get(task_type, "qwen/qwen3-32b:free")
    return client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
    ).choices[0].message.content
```

**判断基準の目安:**

- 入力→出力が定型に近い (分類・要約・フォーマット変換) → 7B〜27B free
- コード生成・デバッグ・軽度の推論 → 32B〜70B free
- ステート管理・複数ステップ計画・高度なコード設計 → 有料モデル

この分類をプロジェクト全体で統一するだけで、有料 LLM への呼び出し比率を **20〜30% 以下** に抑えられることが多い。

---

## パターン 2：自動フォールバック付きリトライ

`:free` モデルは高負荷時に `529 Too Many Requests` や `503 Service Unavailable` を返す。これをハンドリングしないとエージェントが止まる。

```python
import time
import httpx
from openai import OpenAI, RateLimitError, APIStatusError

FREE_MODELS_PRIORITY = [
    "meta-llama/llama-3.3-70b-instruct:free",
    "qwen/qwen3-32b:free",
    "google/gemma-3-27b-it:free",
]
FALLBACK_PAID_MODEL = "anthropic/claude-sonnet-4-5"

def robust_call(messages: list, use_free: bool = True) -> str:
    models = FREE_MODELS_PRIORITY if use_free else [FALLBACK_PAID_MODEL]

    for model in models:
        for attempt in range(3):
            try:
                resp = client.chat.completions.create(
                    model=model,
                    messages=messages,
                    timeout=30,
                )
                return resp.choices[0].message.content
            except RateLimitError:
                wait = 2 ** attempt  # exponential backoff: 1s, 2s, 4s
                time.sleep(wait)
            except APIStatusError as e:
                if e.status_code in (503, 529):
                    time.sleep(2)
                    continue
                raise

    # 全 free モデルが失敗したら有料へ
    return robust_call(messages, use_free=False)
```

**ポイント:**

1. 同系統のモデルを複数保持し、失敗したら次のモデルへ横移動
2. Exponential backoff で RPM 制限を回避
3. 全 free が失敗したときだけ有料へフォールバック (= コスト発生は最終手段)

---

## パターン 3：コンテキスト長を意識したチャンキング

`:free` モデルは多くの場合コンテキスト長が 8K〜32K に制限される。長いドキュメントを丸ごと投げると `context_length_exceeded` エラーになる。

**解決策: Map-Reduce 型チャンキング**

```python
def chunk_text(text: str, max_tokens: int = 6000) -> list[str]:
    """
    簡易トークン数推定 (日本語: 1 文字 ≈ 1.5 token / 英語: 1 word ≈ 1.3 token)
    本番では tiktoken 等を使う
    """
    chars_per_chunk = int(max_tokens / 1.5)
    return [text[i:i+chars_per_chunk] for i in range(0, len(text), chars_per_chunk)]


def map_reduce_summarize(long_text: str) -> str:
    chunks = chunk_text(long_text)

    # Map: 各チャンクを個別に要約
    summaries = []
    for chunk in chunks:
        s = robust_call([
            {"role": "system", "content": "与えられたテキストを 200 字以内で要約してください。"},
            {"role": "user", "content": chunk},
        ])
        summaries.append(s)

    # Reduce: 要約を結合して最終要約
    combined = "\n\n".join(summaries)
    final = robust_call([
        {"role": "system", "content": "以下の部分要約を統合し、400 字以内の最終要約を作成してください。"},
        {"role": "user", "content": combined},
    ])
    return final
```

Map ステップは並列化も容易で、`asyncio.gather` + `aiohttp` を使えば大量チャンクも高速処理できる。

```python
import asyncio

async def async_map(chunks: list[str]) -> list[str]:
    tasks = [async_robust_call(chunk) for chunk in chunks]
    return await asyncio.gather(*tasks)
```

---

## パターン 4：Structured Output で JSON 強制

エージェントのパイプラインでは、LLM の出力を後続ステップで parse する場面が多い。`:free` モデルは有料モデルより指示追従性が低いため、自由形式のプロンプトでは JSON が崩れやすい。

**対策: `response_format` + スキーマ定義 + post-process バリデーション**

OpenRouter は OpenAI の `response_format: { type: "json_object" }` 指定をサポートしているモデルが多い。

```python
import json
from pydantic import BaseModel, ValidationError

class TaskResult(BaseModel):
    status: str       # "done" | "error" | "needs_review"
    confidence: float  # 0.0 〜 1.0
    summary: str

def get_structured_result(task_description: str) -> TaskResult:
    resp = client.chat.completions.create(
        model="meta-llama/llama-3.3-70b-instruct:free",
        messages=[
            {
                "role": "system",
                "content": (
                    "あなたはタスク評価エージェントです。"
                    "必ず以下の JSON スキーマで返答してください:\n"
                    '{"status": "done"|"error"|"needs_review", '
                    '"confidence": 0.0〜1.0, '
                    '"summary": "説明文"}'
                ),
            },
            {"role": "user", "content": task_description},
        ],
        response_format={"type": "json_object"},  # サポートモデルのみ
    )

    raw = resp.choices[0].message.content
    try:
        data = json.loads(raw)
        return TaskResult(**data)
    except (json.JSONDecodeError, ValidationError):
        # パース失敗時はもう一度プロンプトを補強して retry
        return retry_with_stricter_prompt(task_description)
```

`response_format` が効かないモデルへのフォールバックとして、**出力の最初の `{` から最後の `}` を regex で抽出**する手法も有効。

```python
import re

def extract_json(text: str) -> dict:
    match = re.search(r'\{.*\}', text, re.DOTALL)
    if match:
        return json.loads(match.group())
    raise ValueError(f"JSON not found in: {text[:200]}")
```

---

## パターン 5：メタ評価ループで品質を保証

`:free` モデルの出力は有料モデルより品質のばらつきが大きい。重要なタスクでは「**軽量な評価モデルでセルフチェック**」を挟むことで品質を担保できる。

```python
EVALUATOR_MODEL = "qwen/qwen3-32b:free"  # 評価用も無料モデルで OK

def eval_and_retry(
    task: str,
    initial_output: str,
    max_retries: int = 2
) -> str:
    current = initial_output

    for i in range(max_retries):
        score_resp = client.chat.completions.create(
            model=EVALUATOR_MODEL,
            messages=[
                {
                    "role": "system",
                    "content": (
                        "あなたは出力品質の評価者です。"
                        "以下のタスクと出力を評価し、"
                        "スコア (0-10) と改善点を JSON で返してください: "
                        '{"score": int, "issues": ["...", ...]}'
                    ),
                },
                {
                    "role": "user",
                    "content": f"タスク:\n{task}\n\n出力:\n{current}",
                },
            ],
            response_format={"type": "json_object"},
        )

        eval_data = json.loads(score_resp.choices[0].message.content)
        score = eval_data.get("score", 10)

        if score >= 7:
            break  # 品質 OK

        # 品質不足 → issues を踏まえて再生成
        issues_text = "\n".join(eval_data.get("issues", []))
        current = robust_call([
            {
                "role": "system",
                "content": f"以下の問題を修正して出力を改善してください:\n{issues_text}",
            },
            {"role": "user", "content": task},
        ])

    return current
```

このパターンは「**Generator-Critic ループ**」と呼ばれるアーキテクチャの簡易版。本格的な実装では [LangGraph](https://github.com/langchain-ai/langgraph) (Apache 2.0) や [AutoGen](https://github.com/microsoft/autogen) (MIT) を使うのが一般的だが、上記のような自前実装でも十分機能する。

---

## 5 パターンの組み合わせ例

実際のエージェントパイプラインでは、これら 5 パターンを組み合わせる。

```
[入力]
   │
   ▼
[パターン1] タスク難易度判定 → モデル選択
   │
   ▼
[パターン3] コンテキスト長チェック → チャンキング (必要な場合)
   │
   ▼
[パターン2] フォールバック付きリトライ で LLM 呼び出し
   │
   ▼
[パターン4] Structured Output 強制 + バリデーション
   │
   ▼
[パターン5] メタ評価ループ (スコア < 7 なら再生成)
   │
   ▼
[出力]
```

このパイプラインにより、**有料 LLM への呼び出しを全体の 10〜20% に抑えながら、品質は Sonnet 単体使用に近い水準を維持**できる (経験的な値。実測値はタスクドメインによって変わる)。

---

## ベンチマーク参考：`:free` モデルの実力

以下は公開ベンチマーク (2025〜2026 年時点の各モデル公式発表値および Chatbot Arena スコア) を基にした比較。数値は公式発表時のものであり、時期によって変動する。

| モデル | MMLU | HumanEval | コンテキスト長 |
|---|---|---|---|
| Llama 3.3 70B Instruct | ~86% | ~72% | 128K (free 枠では 32K が多い) |
| Qwen3 32B | ~85% | ~75% | 128K (free 枠では 32K) |
| Gemma 3 27B | ~80% | ~65% | 128K (free 枠では 8K が多い) |
| Mistral 7B Instruct | ~65% | ~38% | 32K |
| Claude Sonnet (参考) | ~89% | ~85% | 200K |

Mistral 7B はスコアは低いが**レート制限が緩く**、分類や定型変換タスクなら十分実用的。コード生成・論理推論は Qwen3 32B か Llama 3.3 70B を推奨。

---

## まとめ

| パターン | 効果 | 難易度 |
|---|---|---|
| 1. タスク tier 分け | 有料 LLM コストを 70〜90% 削減 | ★☆☆ |
| 2. 自動フォールバック | 可用性を担保・エージェント停止防止 | ★★☆ |
| 3. チャンキング | 長文ドキュメントへの対応 | ★★☆ |
| 4. Structured Output 強制 | パース失敗率の大幅低減 | ★☆☆ |
| 5. メタ評価ループ | 品質保証・自律的な自己改善 | ★★★ |

「全部一気に実装する」必要はない。まずパターン 1 と 2 だけ入れて計測し、ボトルネックが出たら 3〜5 を追加するのがお勧め。

OpenRouter の `:free` モデルは今後もラインナップが拡充される見込みで、2026 年現在でも十分な品質のモデルが揃っている。コスト意識を持ちながら AI エージェントを設計することで、プロダクトの持続可能性は大きく変わる。

---

## 参考リンク

- [OpenRouter 公式ドキュメント](https://openrouter.ai/docs) — モデル一覧・API 仕様
- [OpenAI Python SDK](https://github.com/openai/openai-python) (Apache 2.0) — OpenRouter との互換利用
- [LangGraph](https://github.com/langchain-ai/langgraph) (Apache 2.0) — Generator-Critic ループの本格実装
- [AutoGen](https://github.com/microsoft/autogen) (MIT) — マルチエージェントフレームワーク
- [Pydantic v2](https://github.com/pydantic/pydantic) (MIT) — 構造化出力バリデーション
- [Chatbot Arena Leaderboard](https://lmarena.ai/) — モデル品質の参考ベンチマーク
- [tiktoken](https://github.com/openai/tiktoken) (MIT) — トークン数の正確な計測

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
