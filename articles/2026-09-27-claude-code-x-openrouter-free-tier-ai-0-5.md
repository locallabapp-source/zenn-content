---
title: "Claude Code × OpenRouter Free Tier で AI 開発コストを月0円に近づけた5つの設計パターン"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- OpenRouter の `:free` モデル群は 2026 年現在、実用レベルのコード生成・要約・分類タスクをほぼ無償でこなせる
- コスト最適化の鍵は「モデルをタスク特性で分流する」こと
- Claude Code をオーケストレーターとして使いつつ、自動化ループの実行馬力は Free モデルに委ねる設計が費用対効果が高い
- 本記事で紹介する5パターンはすべて公開 API の仕様のみに基づく一般的アーキテクチャ

---

## 背景

AI 開発の現場で最初につまずくのは「モデル API 料金が予想より高い」問題だ。GPT-4o や Claude Opus を試しに叩いていると、検証フェーズだけで数万円が飛ぶことがある。

OpenRouter は複数モデルを単一エンドポイントで切り替えられるプロキシサービスで、無料枠 (`:free` suffix を持つモデル) を提供している。2026 年 5 月時点では以下のようなモデルが `:free` で利用できる状態が続いている。

| モデル名 | 得意領域 | コンテキスト長 |
|---|---|---|
| `meta-llama/llama-3.3-70b-instruct:free` | 汎用・日本語品質高め | 128k |
| `mistralai/mistral-7b-instruct:free` | 軽量・高速・英語 | 32k |
| `google/gemma-3-27b-it:free` | コード生成・推論 | 96k |
| `qwen/qwen3-8b:free` | 多言語・コスト効率最高 | 128k |
| `deepseek/deepseek-r1-0528:free` | 長文推論・ベンチマーク強い | 128k |

> ⚠️ Free モデルはレートリミットがあり、高トラフィックなプロダクション用途には向かない。開発・自動化・社内ツール用途に絞るのが現実的。

以下では「Claude Code をプランナー (思考担当)・OpenRouter Free モデルをワーカー (実行担当)」に分けた設計パターンを5つ紹介する。

---

## パターン 1: タスク難易度ルーティング (Tiered Dispatch)

### 考え方

すべてのプロンプトを同じモデルに送るのは料金の無駄遣いだ。タスクを3段階に分類し、モデルを振り分ける。

```
Tier 1 (Free)    : 分類・要約・定型変換・Lint 指摘
Tier 2 (Haiku 等): 軽量コード生成・短い Q&A
Tier 3 (Sonnet)  : 複雑な設計判断・コードレビュー・長文推論
```

### 実装イメージ (Python 疑似コード)

```python
from openai import OpenAI  # OpenRouter は OpenAI SDK 互換

TIERS = {
    "free":   "meta-llama/llama-3.3-70b-instruct:free",
    "mid":    "anthropic/claude-haiku-4",
    "strong": "anthropic/claude-sonnet-4-5",
}

def classify_task(prompt: str) -> str:
    """タスクの難易度を Tier に変換する (簡易ヒューリスティック)"""
    if len(prompt) < 500 and any(k in prompt for k in ["要約", "分類", "翻訳"]):
        return "free"
    if "設計" in prompt or "アーキテクチャ" in prompt:
        return "strong"
    return "mid"

def call_llm(prompt: str, client: OpenAI) -> str:
    tier = classify_task(prompt)
    model = TIERS[tier]
    resp = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
    )
    return resp.choices[0].message.content
```

### 効果

単純な要約・変換タスクが全体の 60〜70% を占める場合、理論上は Tier 1 への移行だけで API コストを 6 割削減できる。実際には品質確認のための A/B 比較が必要なので、最初は `log_tier=True` でどちらのモデルを使ったかを全件記録しておくと良い。

---

## パターン 2: Draft → Review 二段パイプライン

### 考え方

「下書きは安いモデル・レビューは強いモデル」という分業パターンは LLM API の最もシンプルなコスト最適化手法だ。

```
[Free モデル] → 初稿生成 (Draft)
      ↓
[Sonnet 等]   → diff レビュー (Review) ← ここだけ課金
      ↓
[Free モデル] → 指摘を反映して最終稿 (Patch)
```

肝は「レビューに渡すのは初稿全文ではなく、変更差分のみ」にすることだ。プロンプトトークン数を圧縮できる。

```python
import difflib

def draft_then_review(task: str, client_free: OpenAI, client_paid: OpenAI) -> str:
    # Step 1: Free モデルで初稿
    draft = client_free.chat.completions.create(
        model="google/gemma-3-27b-it:free",
        messages=[{"role": "user", "content": task}],
    ).choices[0].message.content

    # Step 2: 差分ベースのレビュー指示
    review_prompt = f"""
以下のコードの問題点を JSON 形式で指摘してください。
修正コードは不要・指摘箇所と理由だけを返してください。

```python
{draft}
```
"""
    feedback = client_paid.chat.completions.create(
        model="anthropic/claude-sonnet-4-5",
        messages=[{"role": "user", "content": review_prompt}],
    ).choices[0].message.content

    # Step 3: Free モデルで修正適用
    patch_prompt = f"以下のコードを次のフィードバックに従って修正してください。\n\nコード:\n{draft}\n\nフィードバック:\n{feedback}"
    final = client_free.chat.completions.create(
        model="google/gemma-3-27b-it:free",
        messages=[{"role": "user", "content": patch_prompt}],
    ).choices[0].message.content

    return final
```

### 注意点

レビューモデルへ渡すトークンを削るために「修正コードを返すな・指摘だけ返せ」と明示的に指示する部分がポイント。出力トークンも課金対象のモデルでは特に有効だ。

---

## パターン 3: ベクトル検索 × RAG で Few-shot コストを削る

### 考え方

Few-shot プロンプトに大量の例を詰め込むとトークン消費が増大する。代わりに「クエリに意味的に近い例だけを動的に取得する」RAG パターンを使えば、入力トークンを 70〜80% 削減できることがある。

```
[クエリ] → embedding → 類似例を上位 K 件取得
                                 ↓
                    [Free モデル] ← Few-shot (K 件のみ)
```

#### シンプルな RAG 実装 (FAISS + OpenRouter)

```python
import numpy as np
import faiss
from openai import OpenAI

# OpenAI embedding (text-embedding-3-small) で例をインデックス化
embed_client = OpenAI()  # OpenAI 本家 (埋め込みは安価)
router_client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="<YOUR_OPENROUTER_KEY>",
)

def embed(text: str) -> np.ndarray:
    return np.array(
        embed_client.embeddings.create(
            model="text-embedding-3-small", input=text
        ).data[0].embedding,
        dtype="float32",
    )

def build_index(examples: list[dict]) -> tuple[faiss.Index, list[dict]]:
    vecs = np.stack([embed(ex["question"]) for ex in examples])
    index = faiss.IndexFlatIP(vecs.shape[1])
    faiss.normalize_L2(vecs)
    index.add(vecs)
    return index, examples

def rag_complete(query: str, index: faiss.Index, examples: list[dict], k: int = 3) -> str:
    q_vec = embed(query).reshape(1, -1)
    faiss.normalize_L2(q_vec)
    _, ids = index.search(q_vec, k)
    shots = "\n\n".join(
        f"Q: {examples[i]['question']}\nA: {examples[i]['answer']}" for i in ids[0]
    )
    prompt = f"{shots}\n\nQ: {query}\nA:"
    return router_client.chat.completions.create(
        model="qwen/qwen3-8b:free",
        messages=[{"role": "user", "content": prompt}],
    ).choices[0].message.content
```

`text-embedding-3-small` は 100 万トークンあたり $0.02 と非常に安価で、RAG で節約できるトークンコストのほうが大きくなるケースが多い。

---

## パターン 4: ストリーミング × 早期終了 (Speculative Stop)

### 考え方

LLM の出力は途中で「もう欲しい情報が取れた」と判断できることがある。ストリーミング API を使いつつ、必要なデータが出た時点で接続を切ると**出力トークン課金を節約**できる。

Free モデルには課金がないが、このパターンはレートリミット消費の節約・レスポンス速度向上として有効だ。

```python
import json

def extract_json_streaming(prompt: str, client: OpenAI) -> dict:
    """JSON が完成した時点でストリームを打ち切る"""
    buffer = ""
    with client.chat.completions.create(
        model="meta-llama/llama-3.3-70b-instruct:free",
        messages=[{"role": "user", "content": prompt}],
        stream=True,
    ) as stream:
        for chunk in stream:
            delta = chunk.choices[0].delta.content or ""
            buffer += delta
            # 閉じ括弧が現れたら JSON として parse を試みる
            if "}" in buffer:
                candidate = buffer[buffer.index("{"):buffer.rindex("}") + 1]
                try:
                    result = json.loads(candidate)
                    stream.close()  # 早期終了
                    return result
                except json.JSONDecodeError:
                    pass  # まだ不完全なので続行
    return json.loads(buffer)
```

### 応用

- Markdown のセクション単位で処理を分割し、必要なセクションだけパースして残りを捨てる
- 「最初の 200 トークンだけ読んで分類 → 分類結果が高信頼なら処理打ち切り」

---

## パターン 5: フォールバック連鎖 (Cascade Fallback)

### 考え方

Free モデルはレートリミットに引っかかったり、稀に品質が不安定だったりする。`try/except` でキャッチし、有料モデルへ自動フォールバックする連鎖設計にしておくと可用性が上がる。

```python
from typing import Callable
import time

MODEL_CASCADE = [
    ("qwen/qwen3-8b:free",                         True),   # (model, is_free)
    ("meta-llama/llama-3.3-70b-instruct:free",     True),
    ("anthropic/claude-haiku-4",                   False),  # 有料・最終手段
]

def cascade_complete(prompt: str, client: OpenAI) -> tuple[str, str]:
    """(応答テキスト, 使用モデル名) を返す"""
    for model, is_free in MODEL_CASCADE:
        try:
            resp = client.chat.completions.create(
                model=model,
                messages=[{"role": "user", "content": prompt}],
                timeout=30,
            )
            return resp.choices[0].message.content, model
        except Exception as e:
            print(f"[WARN] {model} failed: {e}. Falling back...")
            if is_free:
                time.sleep(2)  # Free モデル連続失敗時は少し待つ
            continue
    raise RuntimeError("All models in cascade failed.")
```

#### コスト監視と組み合わせる

```python
from collections import Counter

usage_counter: Counter = Counter()

def cascade_with_tracking(prompt: str, client: OpenAI) -> str:
    text, model = cascade_complete(prompt, client)
    usage_counter[model] += 1
    if "free" not in model:
        print(f"[COST] Paid model invoked: {model} (total: {usage_counter[model]})")
    return text
```

有料モデルへの落下率を週次で確認し、Free モデルのレートリミット上限を超えていないか定期監視するとよい。

---

## 5つのパターンの使い分けまとめ

| パターン | 主な効果 | 最適な場面 |
|---|---|---|
| Tiered Dispatch | コスト 50〜70% 削減 | タスク種別が事前に分類できるとき |
| Draft → Review | 品質維持しつつコスト 40% 削減 | コード生成・長文ドキュメント |
| RAG Few-shot | 入力トークン 70% 削減 | 事例が大量にある Q&A・変換 |
| Speculative Stop | レートリミット消費削減・高速化 | JSON 抽出・構造化出力 |
| Cascade Fallback | 可用性向上 | 24h 自動化パイプライン |

実際の開発では複数を組み合わせることが多い。典型的な構成例:

```
RAG Few-shot → Tiered Dispatch → Cascade Fallback
                    ↓ (Tier 2 or 3 選択時)
             Draft → Review パイプライン
```

---

## ベンチマーク: Free vs Paid の品質差はどこで出るか

個人的な使用経験と公開されているベンチマーク (LMSYS Chatbot Arena / HuggingFace Open LLM Leaderboard) を参考にすると、以下のようなタスクで顕著な差が出やすい。

| タスク | Free モデルで十分か |
|---|---|
| テキスト分類 (3〜5 クラス) | ✅ 十分 |
| 日本語要約 (500 字以下) | ✅ 十分 |
| 英語 → 日本語翻訳 | ✅ ほぼ十分 |
| 単純な CRUD コード生成 | ✅ 十分 |
| 複雑なアルゴリズム設計 | ⚠️ 要検証 |
| 長文一貫性 (8k token 超) | ⚠️ モデル依存 |
| 微妙なニュアンス判断 | ❌ 有料推奨 |
| セキュリティレビュー | ❌ 有料推奨 |

---

## まとめ

OpenRouter の Free モデルは 2026 年時点で「実業務の自動化ループ」に十分使えるレベルに成熟した。重要なのはモデルを一つに固定せず、タスクの難易度・種別・重要度に応じてルーティングすること。

紹介した5パターンは互いに直交しており、組み合わせることで **「品質は維持・コストは最小化」** という理想に近づける。まず Tiered Dispatch だけでも導入してみると、意外なほど多くのタスクが Free Tier で完結することに気づくはずだ。

---

## 参考リンク

- [OpenRouter モデル一覧 (公式)](https://openrouter.ai/models)
- [OpenAI Python SDK ドキュメント](https://platform.openai.com/docs/api-reference)
- [FAISS ドキュメント (Meta Research)](https://faiss.ai/)
- [LMSYS Chatbot Arena Leaderboard](https://chat.lmsys.org/?leaderboard)
- [HuggingFace Open LLM Leaderboard](https://huggingface.co/spaces/HuggingFaceH4/open_llm_leaderboard)
- [text-embedding-3-small 料金 (OpenAI Pricing)](https://openai.com/pricing)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします！
