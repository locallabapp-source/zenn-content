---
title: "Zenn 2026年版・OpenRouter 無料モデル8選を速度・品質・コンテキスト長で徹底比較"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- OpenRouter の `:free` モデルは 2026 年現在 **20 モデル超** が無料で利用可能
- 本記事では実用的な **8 モデル** を選定し、速度・品質・コンテキスト長・用途適性で比較
- 「コスト ¥0 で本番運用できるモデル」を探している個人開発者・スタートアップ向けのガイド

---

## なぜ今 OpenRouter の `:free` モデルを使うのか

LLM の API コストは、個人開発や小規模 SaaS にとって無視できない固定費になりがちです。OpenRouter は複数の LLM プロバイダを統一 API で呼び出せるプロキシサービスですが、その中に **`:free` サフィックスを持つ無料モデル** が常時複数ラインナップされています。

無料モデルの制約は主に次の 3 点です:

1. **レートリミット** — 無料枠は 1 分あたりのリクエスト数・トークン数に上限あり
2. **モデルの可用性** — プロバイダ側の提供ポリシー変更で突然終了することがある
3. **コンテキスト長の差** — 有料版より短いことが多い

それでも「プロトタイプ〜中規模 SaaS の補助処理」には十分すぎるパワーがあります。

---

## 比較対象 8 モデル

以下、2026 年 5 月時点での公式ドキュメント・モデルカードをもとにした比較です。

| モデル ID (OpenRouter) | ベースモデル | コンテキスト長 | 特徴 |
|---|---|---|---|
| `google/gemini-2.0-flash-exp:free` | Gemini 2.0 Flash | 1,048,576 tokens | 超長コンテキスト・マルチモーダル対応 |
| `meta-llama/llama-3.3-70b-instruct:free` | Llama 3.3 70B | 131,072 tokens | 高品質・多言語・OSS 最強クラス |
| `qwen/qwen3-8b:free` | Qwen3 8B | 40,960 tokens | 軽量・日本語強め・Thinking モード搭載 |
| `qwen/qwen3-14b:free` | Qwen3 14B | 40,960 tokens | 8B より品質向上・推論力高い |
| `mistralai/mistral-7b-instruct:free` | Mistral 7B | 32,768 tokens | 軽快・英語コード生成向き |
| `microsoft/phi-3-mini-128k-instruct:free` | Phi-3 Mini | 128,000 tokens | 超軽量・長コンテキスト・コード特化 |
| `deepseek/deepseek-r1:free` | DeepSeek R1 | 163,840 tokens | CoT 推論特化・数学・コーディング強 |
| `nousresearch/hermes-3-llama-3.1-405b:free` | Hermes 3 405B | 131,072 tokens | 大規模・Function Calling・ツール使用 |

---

## 各モデル詳細

### 1. `google/gemini-2.0-flash-exp:free` — 長文処理の王者

**コンテキスト 100 万トークン超** という圧倒的なウィンドウが最大の武器です。PDF 全文を貼り付けて要約させる、巨大なコードベースをまとめてレビューさせる、といったユースケースで無二の存在です。

- ✅ マルチモーダル (画像入力対応)
- ✅ 応答速度が速い (Flash 系)
- ⚠️ 実験版のため API 仕様変更リスクあり
- ⚠️ 無料枠のレートリミットは比較的厳しめ

**向いている用途:** ドキュメント要約・コードベース全体レビュー・画像 OCR+解析

---

### 2. `meta-llama/llama-3.3-70b-instruct:free` — バランス最強の OSS モデル

Meta の Llama 3.3 70B は、オープンソース LLM の中で 2025〜2026 年にかけて最もバランスの取れたモデルの一つです。英語はもちろん、日本語・中国語・スペイン語など多言語での品質が高く、汎用アシスタントとして使いやすい。

- ✅ 高品質な指示追従性
- ✅ 多言語対応 (日本語も実用レベル)
- ✅ 131K コンテキスト
- ⚠️ 70B なので応答レイテンシはやや高い

**向いている用途:** 汎用チャットボット・多言語コンテンツ生成・要約

---

### 3. `qwen/qwen3-8b:free` — 日本語タスクの掘り出し物

Alibaba Cloud の Qwen3 シリーズは日本語・中国語の品質が特に高く、8B という軽量サイズながら Thinking モード (CoT 推論の明示的実行) を搭載しています。`/no_think` プロンプトで通常モード、`/think` で推論ステップを出力する切り替えが可能です。

```python
# Thinking モードを有効にする例
messages = [
    {"role": "user", "content": "/think\n次の数列の規則を説明してください: 1, 1, 2, 3, 5, 8, 13"}
]
```

- ✅ 日本語品質が高い (8B にしては)
- ✅ Thinking モードで複雑推論が可能
- ✅ 応答が速い
- ⚠️ 40K コンテキストは短め
- ⚠️ 大規模推論タスクには 14B 以上を推奨

**向いている用途:** 日本語 FAQ ボット・短文要約・Thinking モードを使った簡易推論

---

### 4. `qwen/qwen3-14b:free` — 8B の上位互換

Qwen3 14B は 8B の品質向上版です。コンテキスト長は同じ 40K ですが、複雑な指示追従・コード生成・多段推論の品質が一段上がります。レイテンシは 8B より若干高いですが、日本語 SaaS の本番補助処理として使いやすいサイズ感です。

- ✅ Thinking モード対応 (8B と同様)
- ✅ コード生成品質が向上
- ✅ 日本語・中国語に引き続き強い
- ⚠️ 40K コンテキストは変わらず

**向いている用途:** 日本語コンテンツ自動生成・コードレビュー補助・RAG の回答生成

---

### 5. `mistralai/mistral-7b-instruct:free` — 軽量英語コード生成

Mistral AI の 7B Instruct は英語コード生成・補完において 7B クラス最高水準です。日本語品質は限定的ですが、英語ドキュメント生成・コメント補完・単純な Function Calling に向いています。

- ✅ 応答が速い (7B の恩恵)
- ✅ 英語コード生成が得意
- ⚠️ 日本語品質は Qwen3 に劣る
- ⚠️ 32K コンテキストは標準的

**向いている用途:** 英語コードコメント生成・GitHub Actions の YAML 生成補助・英文テスト生成

---

### 6. `microsoft/phi-3-mini-128k-instruct:free` — 超軽量×長コンテキストの異端児

Microsoft Phi-3 Mini は **3.8B パラメータで 128K コンテキスト** という設計が特徴的です。パラメータ数の割にコード理解・数学的推論が得意で、「小さいのに賢い」モデルの代表格。エッジデプロイやリソース制約の強い環境での利用実績もあります。

```python
# phi-3 mini に長いログを解析させる例
system_prompt = "あなたはシステムログアナライザです。異常なパターンを検出してください。"
# 128K あるのでアプリログ数千行を直接渡せる
```

- ✅ 超軽量 (3.8B)
- ✅ 128K コンテキストで長文解析可能
- ✅ コード・数学に強い
- ⚠️ 汎用品質は大型モデルに劣る

**向いている用途:** ログ解析・コードスニペットレビュー・軽量数学計算補助

---

### 7. `deepseek/deepseek-r1:free` — 推論特化の最強クラス

DeepSeek R1 は Chain-of-Thought (CoT) 推論を前面に出した設計で、数学・アルゴリズム問題・コーディングコンテスト水準のタスクで GPT-4 クラスを超えるベンチマーク結果を出しています。レスポンスに `<think>...</think>` タグで推論過程が含まれるのが特徴です。

- ✅ 数学・推論タスクで最強クラス
- ✅ 163K コンテキスト
- ✅ コーディング能力が高い
- ⚠️ `<think>` ブロックが長くなるため応答レイテンシが高い
- ⚠️ 「思考を飛ばしたい」場合は別途プロンプト調整が必要

**向いている用途:** 複雑アルゴリズム実装・数学的検証・多段 if-else ロジックの設計補助

---

### 8. `nousresearch/hermes-3-llama-3.1-405b:free` — 大規模 Function Calling

NousResearch の Hermes 3 は Llama 3.1 405B をベースに **Function Calling・ツール使用・構造化出力** の精度を高めたファインチューン版です。405B という圧倒的サイズにより、複雑なツール呼び出しシーケンスの追従精度が高い。

```json
// Function Calling の例
{
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "search_database",
        "parameters": {
          "type": "object",
          "properties": {
            "query": {"type": "string"},
            "limit": {"type": "integer"}
          }
        }
      }
    }
  ]
}
```

- ✅ 405B クラスの品質
- ✅ Function Calling 精度が高い
- ✅ 構造化 JSON 出力が安定
- ⚠️ 応答レイテンシが最も高い
- ⚠️ 可用性が不安定になることがある (大型モデルゆえ)

**向いている用途:** エージェント・ツール呼び出しワークフロー・複雑な JSON スキーマ準拠出力

---

## モデル選定フローチャート

```
タスクの性質は?
│
├── 長文処理 (50K tokens 超) → Gemini 2.0 Flash Exp
│
├── 推論・数学・コーディング (難問) → DeepSeek R1
│
├── Function Calling / ツール使用 → Hermes 3 405B
│
├── 日本語 SaaS の本番補助 → Qwen3 14B
│
├── 軽量・高速・英語コード → Mistral 7B or Phi-3 Mini
│
└── 汎用 (質優先) → Llama 3.3 70B
```

---

## OpenRouter への接続サンプルコード

OpenRouter は OpenAI 互換 API を提供しているため、`openai` SDK をほぼそのまま使えます。

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="<YOUR_OPENROUTER_API_KEY>",
)

def chat(model: str, prompt: str) -> str:
    response = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
        extra_headers={
            "HTTP-Referer": "https://your-app.example.com",
            "X-Title": "Your App Name",
        },
    )
    return response.choices[0].message.content

# 無料モデルで試す
result = chat(
    model="meta-llama/llama-3.3-70b-instruct:free",
    prompt="Rust の所有権システムを 3 行で説明してください。",
)
print(result)
```

### TypeScript / Node.js 版

```typescript
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "https://openrouter.ai/api/v1",
  apiKey: process.env.OPENROUTER_API_KEY,
  defaultHeaders: {
    "HTTP-Referer": "https://your-app.example.com",
    "X-Title": "Your App Name",
  },
});

async function chat(model: string, prompt: string): Promise<string> {
  const res = await client.chat.completions.create({
    model,
    messages: [{ role: "user", content: prompt }],
  });
  return res.choices[0].message.content ?? "";
}

// 使用例
const answer = await chat(
  "qwen/qwen3-14b:free",
  "Next.js の App Router と Pages Router の違いを教えてください。"
);
console.log(answer);
```

---

## モデル切り替えでフォールバックを実装する

無料モデルはレートリミットに達すると `429` を返します。以下のような優先度付きフォールバックを組むと安定した自動化が実現できます。

```python
import time
from openai import OpenAI, RateLimitError

FREE_MODELS = [
    "qwen/qwen3-14b:free",
    "meta-llama/llama-3.3-70b-instruct:free",
    "mistralai/mistral-7b-instruct:free",
]

client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="<YOUR_OPENROUTER_API_KEY>",
)

def chat_with_fallback(prompt: str) -> str:
    for model in FREE_MODELS:
        try:
            res = client.chat.completions.create(
                model=model,
                messages=[{"role": "user", "content": prompt}],
            )
            return res.choices[0].message.content
        except RateLimitError:
            print(f"[WARN] {model} rate limited, trying next...")
            time.sleep(1)
    raise RuntimeError("All free models are rate limited.")
```

---

## 運用上の注意点 3 選

### 1. モデル廃止通知を購読する

無料モデルは突然廃止されることがあります。OpenRouter の Discord または `/api/v1/models` エンドポイントを定期的に叩いてモデル一覧を監視するスクリプトを組んでおくと安心です。

```bash
# モデル一覧を取得して :free のみ抽出
curl -s https://openrouter.ai/api/v1/models \
  | jq '[.data[] | select(.id | endswith(":free")) | {id, context_length}]'
```

### 2. `HTTP-Referer` ヘッダーは必ず設定する

OpenRouter の利用規約では、`HTTP-Referer` にアプリの URL または説明を設定することが推奨されています。これがないとレートリミットが厳しくなるケースがあります。

### 3. `:free` モデルはプロンプトキャッシュが効かない

有料モデルでは `prompt_caching` によりコスト削減できますが、無料モデルはその仕組みの対象外です。長いシステムプロンプトを何度も送る場合でも追加コストはかかりませんが、レイテンシは毎回フルで発生します。

---

## まとめ

| 用途 | 推奨モデル |
|---|---|
| 超長文処理・マルチモーダル | Gemini 2.0 Flash Exp |
| 汎用・多言語・バランス | Llama 3.3 70B |
| 日本語 SaaS 補助処理 | Qwen3 14B |
| 数学・推論・難問コーディング | DeepSeek R1 |
| Function Calling / エージェント | Hermes 3 405B |
| 軽量・高速・英語コード | Mistral 7B |
| 超軽量×長コンテキスト | Phi-3 Mini 128K |
| 日本語軽量・Thinking モード | Qwen3 8B |

OpenRouter の `:free` モデルは「品質ゼロコスト」の夢ではなく、**レートリミット管理とフォールバック設計さえすれば本番 SaaS の補助処理に十分使える**ことが伝わったでしょうか。2026 年の今、まずは `qwen/qwen3-14b:free` と `deepseek/deepseek-r1:free` を並べて自分のタスクで試してみることをお勧めします。

---

## 参考リンク

- [OpenRouter 公式ドキュメント](https://openrouter.ai/docs)
- [OpenRouter モデル一覧](https://openrouter.ai/models)
- [Qwen3 Technical Report (Hugging Face)](https://huggingface.co/Qwen/Qwen3-14B)
- [DeepSeek R1 論文 (arXiv)](https://arxiv.org/abs/2501.12948)
- [Meta Llama 3.3 モデルカード](https://huggingface.co/meta-llama/Llama-3.3-70B-Instruct)
- [Microsoft Phi-3 モデルカード](https://huggingface.co/microsoft/Phi-3-mini-128k-instruct)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
