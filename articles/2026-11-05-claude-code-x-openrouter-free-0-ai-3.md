---
title: "Claude Code × OpenRouter Free モデル：コスト0円でAIコーディング支援を実現する3つの設定"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Claude Code は `ANTHROPIC_API_KEY` 以外に `CLAUDE_CODE_USE_BEDROCK` / `CLAUDE_CODE_USE_VERTEX` を経由したり、`--model` フラグで任意モデルを指定できる
- OpenRouter の `:free` サフィックスモデルを組み合わせると**APIコスト $0** でサブタスク処理を回せる
- ただし「全部 free にすれば最強」は罠。タスク種別ごとに使い分けることが品質とコストのバランスを最適化する鍵

---

## なぜ「コスト0円AI」が現実的になったのか

2024年末〜2025年にかけて、OpenRouterが公開した `:free` モデル群が急速に充実した。

| モデル名（OpenRouter ID） | コンテキスト | 特徴 |
|---|---|---|
| `google/gemini-2.0-flash-exp:free` | 1M tokens | 超高速・長文対応 |
| `meta-llama/llama-3.3-70b-instruct:free` | 128K | 多言語・コード生成 |
| `qwen/qwen3-235b-a22b:free` | 40K | 推論精度が高い |
| `mistralai/mistral-7b-instruct:free` | 32K | 軽量・構造化抽出 |
| `microsoft/phi-4:free` | 16K | 小型モデル最高クラスの推論 |

これらは **レート制限あり・無保証** というトレードオフを受け入れれば、1トークンも課金されない。

開発チームのAIパイプラインにおいて、「要点整理」「フォーマット変換」「コメント生成」のような**思考深度が浅いタスク**を free モデルへ移譲することで、**課金モデルを本当に必要な推論にだけ使う**設計が可能になった。

---

## 設定 1：OpenRouter を Claude Code の代替バックエンドとして使う

Claude Code は `~/.claude/settings.json`（または `CLAUDE_CODE_SETTINGS` 環境変数）でモデルを制御できる。さらに、**OpenRouter は Anthropic API 互換エンドポイントを提供している**ため、以下の環境変数だけで接続先を切り替えられる。

```bash
# .env.local または direnv の .envrc に記載（git commit 禁止）
export ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1"
export ANTHROPIC_API_KEY="sk-or-v1-xxxxxxxxxxxx"  # OpenRouter API Key
```

> ⚠️ **注意**: この設定は公式サポート外の使い方です。OpenRouter のエンドポイント仕様変更により動作が変わる可能性があります。常に公式ドキュメントで最新情報を確認してください。

起動後、Claude Code のモデル選択で `openrouter/` プレフィックスのモデルが見えるはずだ。

---

## 設定 2：タスク種別ごとのモデル使い分けマトリクス

「全タスクを free モデルで回す」と品質が大きく落ちる。実用的な使い分けは以下の通り。

```
┌─────────────────────────────────────────────────────────┐
│          タスク分類 × モデル選択マトリクス              │
├──────────────────────────┬─────────────────────────────┤
│ タスク種別               │ 推奨モデル                  │
├──────────────────────────┼─────────────────────────────┤
│ 設計・アーキテクチャ議論 │ claude-sonnet (有料)        │
│ 複雑なバグ調査           │ claude-sonnet (有料)        │
│ PR レビュー (複雑な差分) │ claude-sonnet (有料)        │
├──────────────────────────┼─────────────────────────────┤
│ コメント・ドキュメント生成│ llama-3.3-70b:free         │
│ 変数名・関数名のサジェスト│ phi-4:free                 │
│ コード要約・changelog 生成│ gemini-2.0-flash-exp:free  │
│ フォーマット・lint 修正  │ mistral-7b:free             │
│ 単体テスト スケルトン生成 │ qwen3-235b-a22b:free       │
└──────────────────────────┴─────────────────────────────┘
```

この分類の根拠は **「アウトプットの品質劣化が人間にすぐ検知できるか」** という観点だ。

変数名の提案が微妙でもエンジニアはすぐ気づいて修正できる。しかし設計判断のミスは後段のコードに伝播するため、ここだけは有料モデルの推論精度を使う価値がある。

---

## 設定 3：Claude Code の SubAgent パターンで free モデルへ処理委譲する

Claude Code の `Task` ツールを使うと、メインの推論コンテキストから切り出してサブタスクを別プロセスで処理できる。このサブタスクのモデルを free モデルに向けることで、**全体コストを下げながらメイン品質を維持**できる。

```typescript
// MCP / Tool use のイメージ（概念コード）
// 実際の Claude Code API は公式ドキュメントを参照
const result = await runSubAgent({
  model: "meta-llama/llama-3.3-70b-instruct:free",
  task: "以下の diff に日本語コメントを付与してください",
  input: diffContent,
});
```

具体的には：

1. **メインエージェント**（claude-sonnet）がタスクを分解・統合・最終判断
2. **サブエージェント**（free モデル）がドキュメント生成・フォーマット変換を担当
3. サブエージェントの出力をメインがレビューして採否を決定

この「オーケストレーター / ワーカー」パターンは [Anthropic のマルチエージェント設計ガイド](https://docs.anthropic.com/en/docs/agents) で推奨されているアーキテクチャだ。

---

## ハマりポイント 3 選

### 1. `:free` モデルのレート制限は予告なし

OpenRouter の free tier は**1分あたりのリクエスト数とトークン数に上限**があり、スパイク時に `429` が返る。リトライロジックとフォールバック先の有料モデルを用意しておくこと。

```python
import time
import openai

def call_with_retry(client, model, messages, max_retries=3):
    for i in range(max_retries):
        try:
            return client.chat.completions.create(
                model=model,
                messages=messages
            )
        except openai.RateLimitError:
            wait = 2 ** i  # exponential backoff
            time.sleep(wait)
    # フォールバック: 有料モデルへ切り替え
    return client.chat.completions.create(
        model="anthropic/claude-3-5-sonnet",
        messages=messages
    )
```

### 2. `:free` モデルはコンテキストウィンドウが小さいものがある

`mistral-7b` は 32K なので、大きなファイルをそのまま渡すとコンテキスト超過でエラーになる。事前にチャンキングする必要がある。

```python
def chunk_text(text: str, max_chars: int = 20000) -> list[str]:
    """シンプルな文字数ベースのチャンキング"""
    return [text[i:i+max_chars] for i in range(0, len(text), max_chars)]
```

### 3. 日本語処理の品質差に注意

free モデルの日本語品質は**モデルによって大きく差がある**。2026年時点では：

- 🟢 **良好**: qwen3-235b-a22b、gemini-2.0-flash-exp、llama-3.3-70b
- 🟡 **普通**: phi-4（英語優先だが日本語も一応動く）
- 🔴 **注意**: mistral-7b（日本語のニュアンスが崩れやすい）

コードコメントを日本語で生成させるなら qwen3 か gemini-flash を選ぶのが安定する。

---

## コスト計算の実例

Sonnet のみ使った場合と使い分け後の比較（1日100回AIアシスト利用のケース）：

```
【Before: すべて claude-sonnet】
コメント生成 × 40回 = $0.0040 (100 tokens × $0.000003 × 40 ≒ 見積もり)
設計相談    × 30回 = $0.0150
コード生成  × 30回 = $0.0120
合計 ≒ $0.0310 / day × 20 営業日 = $0.62 / month

【After: 使い分け後】
コメント生成 × 40回 = $0.00  (free モデル)
設計相談    × 30回 = $0.0150 (sonnet)
コード生成  × 30回 = $0.00  (free モデル for scaffold)
合計 ≒ $0.0150 / day × 20 営業日 = $0.30 / month
```

> 数値はあくまで**概算イメージ**です。実際は OpenRouter の最新料金ページと Anthropic の Pricing ページで確認してください。

金額としては小さく見えるが、**チームで複数エージェントを並列実行**するとこの差が月数千円〜数万円規模になる。

---

## まとめ

| ポイント | 内容 |
|---|---|
| **設定1** | `ANTHROPIC_BASE_URL` を OpenRouter に向けるだけで接続先を切り替えられる |
| **設定2** | タスク種別で「有料 vs free」を仕分けるマトリクスを持つ |
| **設定3** | SubAgent パターンで free モデルへ定型処理を委譲する |
| **注意点** | レート制限・コンテキスト長・日本語品質の差を必ず考慮する |

AIコーディング支援のコストは、**どのタスクにどのモデルを使うか**の設計次第で大きく変わる。全部を最高品質モデルに任せるのではなく、「この判断は人間 or 高品質モデルにしか任せられない」というラインを明確に引くことが、持続可能なAI活用の第一歩だ。

---

## 参考リンク

- [OpenRouter Documentation](https://openrouter.ai/docs)
- [OpenRouter Free Models 一覧](https://openrouter.ai/models?order=newest&supported_parameters=free)
- [Anthropic Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)
- [Anthropic Multi-agent Design Guide](https://docs.anthropic.com/en/docs/agents)
- [Qwen3 HuggingFace Model Card](https://huggingface.co/Qwen/Qwen3-235B-A22B) (Apache 2.0)
- [Meta Llama 3.3 HuggingFace Model Card](https://huggingface.co/meta-llama/Llama-3.3-70B-Instruct) (Llama 3.3 Community License)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
