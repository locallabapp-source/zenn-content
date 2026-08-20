---
title: "Claude Code × OpenRouter Free Models で月0円AI開発環境を構築する5つの設定"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Claude Code の `ANTHROPIC_API_KEY` を差し替えるだけで OpenRouter 経由の free モデルに切り替えられる
- `claude --model` フラグ + `.claude/settings.json` でプロジェクト単位のモデル固定が可能
- 無料枠内でコードレビュー・ドキュメント生成・テスト生成を回す実践パターンを紹介
- コスト $0 を維持しつつ GPT-4o 相当のスループットを得る設定手順を完全解説

---

## 背景

Claude Code は Anthropic の公式 CLI エージェントとして 2025 年に GA し、コーディング支援の現場で急速に普及しています。しかし **従量課金モデルを全タスクに使い続けるとコストが青天井**になる問題があります。

OpenRouter は 100 以上のモデルを単一 API キーで扱えるルーティングサービスで、`google/gemma-3-27b-it:free` や `meta-llama/llama-4-maverick:free` など **rate limit 付きながら無料で使えるモデル** を多数提供しています。

この 2 つを組み合わせることで、**コスト $0 の AI 開発環境** を作れます。

---

## 前提知識

| 項目 | 内容 |
|---|---|
| Claude Code | Anthropic 製 CLI。`npm i -g @anthropic-ai/claude-code` でインストール |
| OpenRouter | https://openrouter.ai。無料アカウントで API キー取得可 |
| OpenRouter Free Tier | RPM / RPD 制限あり・商用利用は各モデルライセンス依存 |
| 対象読者 | Node.js / npm の基本操作ができる中級者以上 |

---

## 設定 1: 環境変数でエンドポイントを差し替える

Claude Code は内部で `ANTHROPIC_API_KEY` と `ANTHROPIC_BASE_URL` を参照します。OpenRouter は Anthropic 互換の `/v1/messages` エンドポイントを提供しているため、**2 つの環境変数を差し替えるだけ**で接続先を変更できます。

```bash
# ~/.bashrc または ~/.zshrc に追記
export ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1"
export ANTHROPIC_API_KEY="sk-or-v1-xxxxxxxxxxxxxxxxxx"  # OpenRouter key
```

> ⚠️ OpenRouter の API キーは `sk-or-v1-` プレフィックスで始まります。Anthropic 純正キーと混同しないよう管理してください。

シェル再起動後に動作確認:

```bash
claude --version          # バージョン確認
claude "Hello, which model are you?" --model google/gemma-3-27b-it:free
```

---

## 設定 2: プロジェクト単位で使用モデルを固定する

毎回 `--model` フラグを指定するのは面倒です。Claude Code はプロジェクトルートの `.claude/settings.json` を読み込むため、そこでデフォルトモデルを固定できます。

```json
// .claude/settings.json
{
  "model": "google/gemma-3-27b-it:free",
  "permissions": {
    "allow": [
      "Bash(git:*)",
      "Bash(npm:*)",
      "Read(**)",
      "Write(**)"
    ]
  }
}
```

このファイルをリポジトリに含めることで、**チーム全員が同じ設定を共有**できます。ただし API キー自体は `.env` や各自の環境変数で管理し、このファイルには含めないでください。

---

## 設定 3: タスク種別ごとにモデルを使い分ける

2026 年時点で OpenRouter が提供している主要な `:free` モデルの特性は以下の通りです (公式 Models ページ準拠):

| モデル | 強み | 弱み | 推奨用途 |
|---|---|---|---|
| `google/gemma-3-27b-it:free` | 多言語・長コンテキスト | コード生成はやや弱め | ドキュメント生成・コメント追加 |
| `meta-llama/llama-4-maverick:free` | コーディング精度高 | コンテキスト長やや短め | コードレビュー・リファクタリング |
| `microsoft/phi-4:free` | 推論・数学が得意 | 長文生成は遅め | アルゴリズム説明・テスト生成 |
| `deepseek/deepseek-r1:free` | 思考チェーン付き推論 | レスポンスが遅い場合あり | バグ原因分析・設計レビュー |
| `qwen/qwen3-235b-a22b:free` | 大規模パラメータ | RPM 制限が厳しめ | 大規模リファクタリング |

Shell alias を使ってタスク別にサッと呼び分ける設定:

```bash
# ~/.bashrc
alias cr='claude --model meta-llama/llama-4-maverick:free'   # code review
alias doc='claude --model google/gemma-3-27b-it:free'        # documentation
alias tg='claude --model microsoft/phi-4:free'               # test generation
alias dbg='claude --model deepseek/deepseek-r1:free'         # debug / analysis
```

使用例:

```bash
# コードレビュー
cr "この PR の差分をレビューして改善点を列挙してください" < git diff HEAD~1

# テスト生成
tg "以下の関数のユニットテストを Vitest で書いてください" < src/utils/parser.ts

# ドキュメント生成
doc "JSDoc コメントを付けて型情報も補完してください" < src/api/client.ts
```

---

## 設定 4: rate limit に引っかからないためのキュー戦略

OpenRouter の free モデルは **RPM (Requests Per Minute) が 10〜20 程度** に制限されているケースが多く、大量のファイルを一気に処理しようとするとエラーになります。

`xargs` + `sleep` の組み合わせで簡易キューを実装できます:

```bash
#!/bin/bash
# review-all.sh: src/ 配下の .ts ファイルを順番にレビュー

find src -name "*.ts" | while read file; do
  echo "=== Reviewing: $file ==="
  claude --model meta-llama/llama-4-maverick:free \
    "コードレビューして問題点を日本語で出力してください:" \
    < "$file"
  sleep 4  # 60 / RPM_LIMIT = 60 / 15 = 4 秒インターバル
done
```

より丁寧に制御したい場合は `ts-node` スクリプトで exponential backoff を実装するか、後述の `openrouter-limit-check` アプローチを使います。

---

## 設定 5: フォールバック設定でコスト急増を防ぐ

free モデルがすべてダウンしたとき、何も設定していないと Claude Code がデフォルトモデル (課金発生) にフォールバックするリスクがあります。

`.claude/settings.json` にモデルを明示固定している場合、存在しないモデルを指定するとエラーで止まるためフォールバックしません。しかし **グローバル設定だけに頼っている場合** はシェルスクリプトでガードを追加することを推奨します:

```bash
#!/bin/bash
# claude-free.sh: free モデル専用ラッパー

MODEL="${CLAUDE_FREE_MODEL:-google/gemma-3-27b-it:free}"

# :free サフィックスが付いていなければ実行しない
if [[ "$MODEL" != *":free" ]]; then
  echo "ERROR: Non-free model specified: $MODEL" >&2
  echo "Set CLAUDE_FREE_MODEL to a ':free' model." >&2
  exit 1
fi

exec claude --model "$MODEL" "$@"
```

```bash
chmod +x ./claude-free.sh
alias cl='./claude-free.sh'
```

これで誤って有料モデルを呼び出すヒューマンエラーを防げます。

---

## 実践例: PR 自動コメント生成パイプライン

以下は GitHub Actions ではなく **ローカルで動かす** PR コメント草稿生成スクリプトの例です。OSS・公開ドキュメントの範囲で実装できます:

```bash
#!/bin/bash
# pr-review-draft.sh
# Usage: ./pr-review-draft.sh <base-branch>

BASE="${1:-main}"
DIFF=$(git diff "$BASE"...HEAD)

if [ -z "$DIFF" ]; then
  echo "差分がありません"
  exit 0
fi

claude --model meta-llama/llama-4-maverick:free \
  "以下の git diff を技術的にレビューし、Markdown 形式でレビューコメント草稿を作成してください。
  
観点:
1. バグ・ロジックエラー
2. セキュリティリスク (SQL インジェクション・XSS・認証バイパス等)
3. パフォーマンス改善余地
4. 可読性・命名規則

diff:
$DIFF" \
  > pr-review-draft.md

echo "✅ pr-review-draft.md に保存しました"
```

実行すると `pr-review-draft.md` にレビュー草稿が出力され、そのまま GitHub PR コメントにコピペできます。

---

## よくあるトラブルと解決策

### `invalid_api_key` エラーが出る

OpenRouter キーを設定したのに Anthropic エンドポイントに接続しようとしている可能性があります。`ANTHROPIC_BASE_URL` が正しく設定されているか確認してください:

```bash
echo $ANTHROPIC_BASE_URL
# → https://openrouter.ai/api/v1 が期待値
```

### `model not found` エラーが出る

OpenRouter のモデル名はバージョンや提供停止で変わることがあります。最新のモデル一覧は https://openrouter.ai/models で確認し、`:free` フィルターを使って現在利用可能なモデルを確認してください。

### レスポンスが途中で切れる

free モデルは `max_tokens` のデフォルトが低い場合があります。長い出力が必要なときは:

```bash
claude --model google/gemma-3-27b-it:free \
  --max-tokens 4096 \
  "長いドキュメントを生成してください..."
```

---

## コスト比較まとめ

| シナリオ | Claude 3.5 Sonnet (有料) | OpenRouter :free モデル |
|---|---|---|
| 1 日 50 回レビュー | 約 $2〜$5 (コードの長さによる) | $0 (rate limit 内) |
| 週 5 日運用・月換算 | 約 $40〜$100 | $0 |
| 大規模リファクタ (10万トークン) | 約 $3 | $0 (分割して送信) |

もちろん **品質・速度・安定性は有料モデルが上** ですが、ドラフト生成・コメント付与・テンプレート出力のような反復作業は free モデルで十分なケースが多く、**有料モデルは最終判断と高品質アウトプットが必要な場面に集中投資** する使い分けが合理的です。

---

## まとめ

1. `ANTHROPIC_BASE_URL` + `ANTHROPIC_API_KEY` の 2 変数差し替えで Claude Code から OpenRouter の free モデルに接続できる
2. `.claude/settings.json` でプロジェクト単位のモデル固定が可能
3. Shell alias でタスク別モデル使い分けを自動化するとスムーズ
4. `sleep` によるレート制御スクリプトで rate limit エラーを回避
5. `:free` サフィックスチェックのラッパースクリプトで意図しない課金を防ぐ

free モデルの精度は日々向上しており、**2026 年現在では中〜大規模プロジェクトの補助ツールとして実用に耐えるレベル** に達しています。まずは小さなプロジェクトで設定 1 だけ試してみてください。

---

## 参考リンク

- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code) — Anthropic
- [OpenRouter Models](https://openrouter.ai/models) — openrouter.ai
- [OpenRouter API Reference](https://openrouter.ai/docs/api-reference) — openrouter.ai
- [Claude Code Settings Reference](https://docs.anthropic.com/en/docs/claude-code/settings) — Anthropic
- [OpenRouter Free Tier Limits](https://openrouter.ai/docs/limits) — openrouter.ai

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
