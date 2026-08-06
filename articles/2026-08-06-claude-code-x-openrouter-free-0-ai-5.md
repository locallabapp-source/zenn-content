---
title: "Claude Code × OpenRouter `:free` モデルで月額 $0 のAI開発環境を構築する5つの設定"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- OpenRouter の `:free` サフィックスモデルは **商用利用可・月額 $0** で使える強力な選択肢
- Claude Code の `ANTHROPIC_BASE_URL` 切り替えで OpenRouter 経由ルーティングが実現できる
- Qwen3・Gemini 2.0 Flash・Llama 3.3 など **2026 年現在の高品質モデルが無料枠に並ぶ**
- タスク種別でモデルを使い分けるルーティング設計が運用コスト最適化の鍵

---

## 背景: LLM 開発コストの現実

Claude Sonnet を本番で使い続けると、**中規模チームでも月 $200〜$500 超え**になることがある。Claude Code によるエージェント的な反復実行では、1 タスクで数十回のAPIコールが発生するため、コスト増加が加速しやすい。

OpenRouter は数十のモデルプロバイダを統一エンドポイント(`https://openrouter.ai/api/v1`)で束ねるプロキシサービスであり、モデル名に `:free` を付けることでレート制限付きの **無料枠** を使える。これを Claude Code のカスタムベース URL と組み合わせる設定が、**2026 年現在の現実解**になりつつある。

> **注**: `:free` モデルはレート制限と可用性の変動があるため、本番クリティカルパスには向かない。開発・実験・バッチ非同期タスク向けの戦略として活用する。

---

## 1. OpenRouter `:free` モデルの現状 (2026年春)

2026年5月時点で `:free` 枠に並ぶ主要モデル(OpenRouter 公式ページ掲載ベース):

| モデル識別子 | コンテキスト | 特徴 |
|---|---|---|
| `qwen/qwen3-235b-a22b:free` | 128K | MoE アーキテクチャ、思考モード切替対応 |
| `qwen/qwen3-30b-a3b:free` | 128K | 軽量 MoE、速度優先 |
| `google/gemini-2.0-flash-exp:free` | 1M | 超長コンテキスト、マルチモーダル |
| `meta-llama/llama-3.3-70b-instruct:free` | 128K | 汎用指示追従、英語強 |
| `deepseek/deepseek-chat-v3-0324:free` | 64K | コード生成が高品質 |
| `mistralai/mistral-small-3.1-24b-instruct:free` | 128K | 多言語・軽量 |

> 掲載モデルは随時変動する。[openrouter.ai/models](https://openrouter.ai/models) でフィルタ `free` を使い最新状況を確認すること。

---

## 2. Claude Code に OpenRouter を繋ぐ設定

Claude Code は `ANTHROPIC_BASE_URL` 環境変数と `ANTHROPIC_API_KEY` の差し替えで、OpenRouter 互換エンドポイントに切り替えられる。

### 基本設定

```bash
export ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1"
export ANTHROPIC_API_KEY="sk-or-v1-xxxxxxxxxxxx"  # OpenRouter の API キー
```

Claude Code 起動時に `--model` フラグでモデルを指定する:

```bash
claude --model "qwen/qwen3-235b-a22b:free" "このコードをリファクタしてください"
```

### `~/.claude/settings.json` による永続設定

Claude Code はグローバル設定ファイルでデフォルトモデルを上書きできる:

```json
{
  "model": "qwen/qwen3-235b-a22b:free",
  "fallbackModel": "google/gemini-2.0-flash-exp:free"
}
```

> **注意点**: `settings.json` の仕様は Claude Code のバージョンで変化する可能性があるため、`claude --help` で最新オプションを確認すること。

---

## 3. タスク種別によるモデルルーティング設計

`:free` モデルはレート制限があるため、**全タスクを同一モデルに投げない**設計が重要。

```
┌─────────────────────────────────┐
│          タスク種別             │
└───────┬────────────┬────────────┘
        │            │
   軽量タスク    重量タスク
  (コメント生成  (アーキテクチャ
   typo 修正)    設計・複雑デバッグ)
        │            │
        ▼            ▼
 :free モデル   有料モデル(Sonnet等)
 qwen3-30b      claude-sonnet-4-5
```

シェルスクリプトで切り替えを自動化する例:

```bash
#!/bin/bash
# model-router.sh

TASK_TYPE="${1:-light}"

if [ "$TASK_TYPE" = "heavy" ]; then
  export ANTHROPIC_BASE_URL="https://api.anthropic.com"
  export ANTHROPIC_API_KEY="${ANTHROPIC_API_KEY_PROD}"
  MODEL="claude-sonnet-4-5"
else
  export ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1"
  export ANTHROPIC_API_KEY="${OPENROUTER_API_KEY}"
  MODEL="qwen/qwen3-235b-a22b:free"
fi

echo "Using model: $MODEL (base: $ANTHROPIC_BASE_URL)"
claude --model "$MODEL" "${@:2}"
```

使い方:

```bash
# 軽量: コメント追加・フォーマット修正など
./model-router.sh light "src/utils.ts のコメントを補完して"

# 重量: 設計相談・複雑なリファクタ
./model-router.sh heavy "このサービス層のDI設計を見直して"
```

---

## 4. OpenRouter の `X-Title` ヘッダーで用途を明示する

OpenRouter はリクエストヘッダーで **アプリ名・URL** を受け付け、ダッシュボードでの集計に使える。Claude Code 経由では直接設定できないが、`curl` や SDK 直接呼び出しでは有効な習慣:

```bash
curl https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer ${OPENROUTER_API_KEY}" \
  -H "X-Title: jimolab-dev-agent" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen/qwen3-235b-a22b:free",
    "messages": [{"role":"user","content":"Rustのエラーハンドリングを最適化して"}]
  }'
```

このヘッダーを入れておくと、**用途別のレート制限追跡**が後から可能になる。

---

## 5. `:free` モデルの品質ギャップを埋める3つのプロンプト戦略

無料モデルは有料モデルと比べて指示追従の精度が落ちる場合がある。以下の戦略で品質ギャップを縮小できる。

### 5-1. 思考ステップを明示する

```
以下の手順で回答してください:
1. 問題の原因を特定する
2. 修正方針を 1 行で述べる
3. 修正後のコードを出力する
4. 変更点の差分サマリを書く
```

Qwen3 系は `<think>` タグ付き思考モードを持つが、OpenRouter 経由では `enable_thinking` パラメータが必要な場合がある:

```json
{
  "model": "qwen/qwen3-235b-a22b:free",
  "messages": [...],
  "extra_body": {
    "thinking": {
      "type": "enabled",
      "budget_tokens": 1024
    }
  }
}
```

### 5-2. Few-shot 例を 1〜2 件付ける

特にコード変換タスクでは入力/出力の例を 1 件添えるだけで精度が大きく向上する:

```
# 例
入力: const x = data.map(d => d.value)
出力: const values = data.map((datum) => datum.value)
# 理由: 変数名を意味のある名前に変更

# タスク
入力: const r = items.filter(i => i.active)
出力:
```

### 5-3. 出力形式を JSON で縛る

```json
{
  "model": "deepseek/deepseek-chat-v3-0324:free",
  "response_format": { "type": "json_object" },
  "messages": [{
    "role": "user",
    "content": "このコードのバグを { \"bug\": \"...\", \"fix\": \"...\", \"risk\": \"low|medium|high\" } 形式で出力して"
  }]
}
```

---

## `:free` モデル選定チートシート

```
目的                      推奨 :free モデル
─────────────────────────────────────────
コード生成・補完           deepseek/deepseek-chat-v3-0324:free
長文読み込み・要約         google/gemini-2.0-flash-exp:free (1M ctx)
多言語・日本語品質         qwen/qwen3-235b-a22b:free
汎用指示追従(英語)         meta-llama/llama-3.3-70b-instruct:free
速度優先(軽量タスク)       qwen/qwen3-30b-a3b:free
```

---

## まとめ

| ポイント | 内容 |
|---|---|
| **エンドポイント切替** | `ANTHROPIC_BASE_URL` を OpenRouter に向けるだけで Claude Code が動く |
| **コスト削減** | 開発・実験フェーズのタスクを `:free` に逃がすことで有料消費を圧縮 |
| **品質担保** | 重要タスクは有料モデルへ自動ルーティング |
| **プロンプト設計** | 思考ステップ明示 + Few-shot + JSON 縛りで `:free` の精度を最大化 |

OpenRouter `:free` を賢く使えば、**プロトタイプから本番移行直前まで LLM コストをほぼゼロに抑える**開発ループが実現できる。有料モデルは本当に必要な判断・生成にだけ集中投下する、という設計思想が 2026 年の LLM 開発の現実解になりつつある。

---

## 参考リンク

- [OpenRouter 公式サイト](https://openrouter.ai)
- [OpenRouter モデル一覧 (free フィルタ)](https://openrouter.ai/models?q=free)
- [Claude Code 公式ドキュメント](https://docs.anthropic.com/ja/docs/claude-code)
- [Qwen3 技術レポート (Hugging Face)](https://huggingface.co/Qwen/Qwen3-235B-A22B)
- [DeepSeek-V3 技術レポート](https://github.com/deepseek-ai/DeepSeek-V3)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
