---
title: "Claude Code × OpenRouter Free モデル活用術：コスト0円でAIコーディング支援を最大化する5つの設定"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- OpenRouter の `:free` モデルは **コスト0円** でコーディング支援に使える
- Claude Code の `--model` フラグ / 環境変数で自在に切り替え可能
- モデルの特性・コンテキスト長・レート制限を把握すれば実用水準に達する
- 「重い思考タスク」と「軽い定型タスク」でモデルを分けるのが鉄則

---

## はじめに

AI コーディング支援ツールの月額コストが気になり始めていませんか？

Claude Code 単体でも十分強力ですが、**OpenRouter の無料モデル (`:free` サフィックス付き)** を組み合わせることで、定型的なタスク・下書き生成・コードフォーマット確認といった「コストを払いたくない処理」を0円で回せます。

本記事では、OpenRouter の Free モデルを Claude Code ワークフローに組み込む**実践的な5つのパターン**を解説します。

> **動作確認環境**: Claude Code 1.x 系 / OpenRouter API (2026年5月時点)

---

## 1. OpenRouter の `:free` モデルとは

OpenRouter は複数のモデルプロバイダーを単一の OpenAI 互換 API で束ねるルーターサービスです ([openrouter.ai](https://openrouter.ai))。

`:free` サフィックスが付いたモデルは **レート制限付きで無料** で使えます。2026年5月時点での主な無料モデルは次の通りです。

| モデル名 | コンテキスト長 | 強み |
|---|---|---|
| `google/gemini-2.0-flash-exp:free` | 1M tokens | 長文コード・大規模リポジトリ向け |
| `meta-llama/llama-3.3-70b-instruct:free` | 131k tokens | 汎用コーディング・バランス型 |
| `deepseek/deepseek-r1:free` | 164k tokens | 推論・アルゴリズム設計 |
| `qwen/qwen3-235b-a22b:free` | 131k tokens | 多言語対応・日本語精度高め |
| `mistralai/mistral-7b-instruct:free` | 32k tokens | 軽量・低レイテンシ |

> ⚠️ 無料モデルはプロバイダーの都合でラインナップが変わります。最新状況は [openrouter.ai/models?q=:free](https://openrouter.ai/models?q=%3Afree) で確認してください。

### レート制限の現実

`:free` モデルには **1分あたり20リクエスト・1日200リクエスト** 程度の制限が設けられているケースが多いです (モデルにより異なります)。

「全タスクを無料モデルで賄う」は現実的ではありませんが、「単純タスク専用レーン」として機能させれば十分です。

---

## 2. Claude Code での OpenRouter 接続設定

Claude Code は内部的に Anthropic API を呼び出しますが、**OpenAI 互換エンドポイントを持つプロバイダー** への切り替えは環境変数で対応できます。

### 基本セットアップ

```bash
# .env または shell の設定ファイルに追加
export OPENROUTER_API_KEY="sk-or-v1-xxxxxxxxxx"
```

Claude Code 自体は Anthropic モデルを前提としているため、**OpenRouter モデルを呼び出す場合は別途スクリプトを用意する**アプローチが現実的です。

```bash
# openrouter-chat.sh (軽量ラッパー例)
#!/usr/bin/env bash
# 引数: $1 = モデル名, $2 = プロンプトファイル

MODEL="${1:-meta-llama/llama-3.3-70b-instruct:free}"
PROMPT_FILE="${2:-/dev/stdin}"

curl -s https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "model": "$MODEL",
  "messages": [
    {
      "role": "user",
      "content": "$(cat "$PROMPT_FILE" | jq -Rs .)"
    }
  ],
  "max_tokens": 4096
}
EOF
```

> **ライセンス**: 上記スクリプト例は MIT ライセンスで自由に使えます。`jq` (Apache-2.0) が必要です。

---

## 3. 5つの実践的な活用パターン

### パターン 1: コミットメッセージの自動生成

`git diff --staged` の出力を Free モデルに渡して Conventional Commits 形式のメッセージを生成します。

```bash
# commit-msg-gen.sh
#!/usr/bin/env bash

DIFF=$(git diff --staged)

if [ -z "$DIFF" ]; then
  echo "ステージングエリアが空です"
  exit 1
fi

PROMPT="以下の git diff を見て、Conventional Commits 形式 (feat/fix/refactor/docs/chore) の
コミットメッセージを1行で生成してください。日本語で。

diff:
$DIFF"

echo "$PROMPT" | \
  curl -s https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg model "meta-llama/llama-3.3-70b-instruct:free" \
              --arg prompt "$PROMPT" \
        '{model: $model, messages: [{role: "user", content: $prompt}], max_tokens: 200}')" \
  | jq -r '.choices[0].message.content'
```

**ポイント**: コミットメッセージ生成は入出力が短く、レート制限の消費が最小です。軽量モデル (`mistral-7b`) でも品質が安定しやすいタスクです。

---

### パターン 2: README の日本語→英語翻訳

OSS に英語 README を追加したいが翻訳コストをかけたくない場面で活用できます。

```bash
# translate-readme.sh
#!/usr/bin/env bash

SRC_FILE="${1:-README.md}"
DEST_FILE="${2:-README.en.md}"

CONTENT=$(cat "$SRC_FILE")

PROMPT="以下の日本語 Markdown を自然な英語に翻訳してください。
コードブロック・見出し・リンク・バッジは変更しないでください。

$CONTENT"

curl -s https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg model "qwen/qwen3-235b-a22b:free" \
              --arg p "$PROMPT" \
        '{model: $model, messages: [{role: "user", content: $p}], max_tokens: 8192}')" \
  | jq -r '.choices[0].message.content' > "$DEST_FILE"

echo "✅ 翻訳完了: $DEST_FILE"
```

**なぜ Qwen3?**: 日英翻訳精度は Qwen3-235B が無料モデルの中でトップクラスです。日本語固有の敬語・技術用語の扱いが安定しています。

---

### パターン 3: TypeScript 型定義の補完・レビュー

`any` が残っている箇所を検出し、推奨型を提案させます。

```bash
# review-types.sh
#!/usr/bin/env bash

FILE="${1}"
CONTENT=$(cat "$FILE")

PROMPT="以下の TypeScript コードを見て:
1. \`any\` 型が使われている箇所を列挙してください
2. それぞれに対して具体的な代替型を提案してください
3. 理由も1行で添えてください

コード:
\`\`\`typescript
$CONTENT
\`\`\`"

curl -s https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg model "deepseek/deepseek-r1:free" \
              --arg p "$PROMPT" \
        '{model: $model, messages: [{role: "user", content: $p}], max_tokens: 2048}')" \
  | jq -r '.choices[0].message.content'
```

**なぜ DeepSeek-R1?**: 型推論・論理的なコード解析は R1 の推論能力が光るタスクです。レート制限が厳しい場面では Llama-3.3-70B でも代用できます。

---

### パターン 4: テストケースのスケルトン生成

実装コードから `describe / it` ブロックの骨格を生成し、テスト漏れを減らします。

```bash
# gen-test-skeleton.sh
#!/usr/bin/env bash

SRC="${1}"
FRAMEWORK="${2:-vitest}"  # vitest | jest | pytest

CONTENT=$(cat "$SRC")
LANG_EXT="${SRC##*.}"

PROMPT="${FRAMEWORK} を使ったテストファイルのスケルトンを生成してください。
- 各関数・メソッドに対応する describe/it ブロックを作る
- アサーションのプレースホルダー (expect(result).toBe(TODO)) を入れる
- エッジケース (null, 空文字, 上限値) の it ブロックも追加する
- 実装はせずスケルトンのみ出力する

実装コード (${LANG_EXT}):
$CONTENT"

curl -s https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg model "google/gemini-2.0-flash-exp:free" \
              --arg p "$PROMPT" \
        '{model: $model, messages: [{role: "user", content: $p}], max_tokens: 4096}')" \
  | jq -r '.choices[0].message.content'
```

**なぜ Gemini 2.0 Flash?**: 1M コンテキストのおかげで大きなソースファイルでも一括処理できます。コード生成のバランスが取れており、スケルトン生成のような「構造的な出力」を得意とします。

---

### パターン 5: PR 説明文の自動生成

GitHub Actions または手元スクリプトで PR テンプレートを自動補完します。

```bash
# gen-pr-body.sh
#!/usr/bin/env bash

BASE_BRANCH="${1:-main}"
DIFF=$(git diff "$BASE_BRANCH"...HEAD --stat)
COMMITS=$(git log "$BASE_BRANCH"...HEAD --oneline)

PROMPT="以下の git 情報から GitHub Pull Request の説明文を Markdown で生成してください。

# 構成
- ## 変更概要 (2-3行)
- ## 変更内容 (箇条書き)
- ## テスト方法
- ## スクリーンショット (該当なし でも OK)
- ## チェックリスト ([ ] テスト追加, [ ] ドキュメント更新)

## git diff --stat
$DIFF

## コミット一覧
$COMMITS"

curl -s https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg model "meta-llama/llama-3.3-70b-instruct:free" \
              --arg p "$PROMPT" \
        '{model: $model, messages: [{role: "user", content: $p}], max_tokens: 1024}')" \
  | jq -r '.choices[0].message.content'
```

---

## 4. モデル選定チートシート

上記パターンを一般化すると、タスク特性によるモデル選択指針は次のようになります。

```
タスク種別                       推奨 Free モデル
─────────────────────────────────────────────────
短文生成 (コミット・ラベル)      mistral-7b          ← 低レイテンシ
翻訳・リライト                   qwen3-235b-a22b     ← 日英精度
推論・型解析・アルゴリズム       deepseek-r1         ← 思考力
大規模コード一括処理             gemini-2.0-flash-exp ← 長コンテキスト
汎用コーディング                 llama-3.3-70b        ← バランス
```

> **コスト戦略の鉄則**: 「推論が必要なタスク」は Claude Sonnet など有料モデルへ。「定型・構造的・短文」なタスクは Free モデルへ。この分類を守るだけでコストを大幅に削減できます。

---

## 5. エラーハンドリングと Fallback 設計

Free モデルはレート制限に達した場合 `429 Too Many Requests` を返します。本番ワークフローに組み込む場合はフォールバックを実装しましょう。

```bash
# openrouter-with-fallback.sh
#!/usr/bin/env bash

MODELS=(
  "meta-llama/llama-3.3-70b-instruct:free"
  "qwen/qwen3-235b-a22b:free"
  "mistralai/mistral-7b-instruct:free"
)
PROMPT="$1"

for MODEL in "${MODELS[@]}"; do
  RESPONSE=$(curl -s -w "\n%{http_code}" \
    https://openrouter.ai/api/v1/chat/completions \
    -H "Authorization: Bearer $OPENROUTER_API_KEY" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg m "$MODEL" --arg p "$PROMPT" \
          '{model: $m, messages: [{role: "user", content: $p}], max_tokens: 2048}')")

  HTTP_CODE=$(echo "$RESPONSE" | tail -1)
  BODY=$(echo "$RESPONSE" | head -n -1)

  if [ "$HTTP_CODE" -eq 200 ]; then
    echo "$BODY" | jq -r '.choices[0].message.content'
    exit 0
  fi

  echo "⚠️  $MODEL: HTTP $HTTP_CODE → 次のモデルへフォールバック" >&2
  sleep 2
done

echo "❌ すべての Free モデルが利用不可。有料モデルへの切り替えを検討してください。" >&2
exit 1
```

**設計ポイント**:
- モデルリストを配列で管理し、上から順に試行する
- 429 以外 (400, 401, 5xx) は即時エラーにして無駄なリトライを避ける
- `sleep 2` でレート制限のクールダウンを待つ

---

## まとめ

| パターン | ベストモデル | コスト |
|---|---|---|
| コミットメッセージ生成 | mistral-7b | ¥0 |
| README 翻訳 | qwen3-235b-a22b | ¥0 |
| 型定義レビュー | deepseek-r1 | ¥0 |
| テストスケルトン | gemini-2.0-flash-exp | ¥0 |
| PR 説明文 | llama-3.3-70b | ¥0 |

OpenRouter の Free モデルは「無料だから品質が低い」と思われがちですが、**タスクを正しく分類して適切なモデルへ振り分ける**ことで、有料モデルと遜色ない結果を得られるケースは多くあります。

まずはコミットメッセージ生成から試してみてください。1週間で数十〜数百回の API コールが節約できます。

---

## 参考リンク

- [OpenRouter 公式ドキュメント](https://openrouter.ai/docs)
- [OpenRouter モデル一覧 (Free フィルタ)](https://openrouter.ai/models?q=%3Afree)
- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)
- [Conventional Commits 仕様](https://www.conventionalcommits.org/)
- [jq 公式ドキュメント](https://jqlang.github.io/jq/) (Apache-2.0)
- [curl 公式サイト](https://curl.se/) (MIT / curl ライセンス)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
