---
title: "Claude Code × OpenRouter で始める AI コーディング環境：無料枠だけでここまでできた 7 つの構成パターン"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Claude Code は `ANTHROPIC_API_KEY` を設定すれば動くが、OpenRouter の無料モデルを組み合わせると**コストゼロで大半のタスクをこなせる**
- `claude` CLI の `--model` フラグと `ANTHROPIC_BASE_URL` 環境変数を活用してルーティングを切り替える
- 7 つのユースケース別構成パターンを紹介・どこで有料枠が必要になるかの境界線も整理

---

## 背景：AI コーディングツールのコスト問題

Claude Code（Anthropic 公式 CLI）は高品質なコード生成・編集・テスト生成をターミナルから直接行えるツールとして注目を集めています。しかし、デフォルトでは Anthropic API に直接接続するため、ヘビーユーズすると月額コストが気になります。

一方 **OpenRouter** は、200 以上の LLM を単一の OpenAI 互換エンドポイント（`https://openrouter.ai/api/v1`）で提供しており、その中には**完全無料のモデル**（`:free` サフィックス付き）が数十件含まれています。

この 2 つを組み合わせると：

```
Claude Code の UX + OpenRouter の無料モデルの経済性
```

という構成が実現できます。本記事では、その具体的な 7 パターンを紹介します。

---

## 前提知識

### Claude Code の仕組み

Claude Code は内部で Anthropic Messages API を呼び出す CLI です。以下の環境変数でエンドポイントと認証を上書きできます。

```bash
export ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1
export ANTHROPIC_API_KEY=sk-or-v1-xxxxxxxxxxxx  # OpenRouter の API キー
```

`ANTHROPIC_BASE_URL` を設定すると、Claude Code が送るリクエストは設定先エンドポイントへ向かいます。OpenRouter は Messages API の形式を受け付けるため、互換性があります。

### OpenRouter の無料枠（`:free` モデル）

2025〜2026 年時点で利用可能な主な `:free` モデル（公式サイト参照）：

| モデル名 | コンテキスト | 特徴 |
|---|---|---|
| `google/gemini-2.0-flash-exp:free` | 1M token | マルチモーダル・高速 |
| `meta-llama/llama-3.3-70b-instruct:free` | 128k token | 汎用コード生成 |
| `qwen/qwen3-235b-a22b:free` | 32k token | 推論強化・コード品質高 |
| `mistralai/mistral-7b-instruct:free` | 32k token | 軽量・高速 |
| `deepseek/deepseek-r1:free` | 64k token | 長いチェーン推論 |

> ⚠️ `:free` モデルはレートリミットが厳しめ（例: 20 req/min）で、ダウンタイムが発生することがあります。本番クリティカルなフローには向きません。

---

## 7 つの構成パターン

### パターン 1：Shell プロファイルで切り替え可能にする

最もシンプルな構成です。`~/.bashrc` または `~/.zshrc` に以下を追記します。

```bash
# OpenRouter (無料モデル)
alias claude-free='ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1 \
  ANTHROPIC_API_KEY=<your-openrouter-key> \
  CLAUDE_MODEL=qwen/qwen3-235b-a22b:free \
  claude'

# Anthropic 本家 (有料)
alias claude-pro='ANTHROPIC_API_KEY=<your-anthropic-key> \
  ANTHROPIC_BASE_URL=https://api.anthropic.com \
  claude'
```

使用例：

```bash
# 下書き・調査フェーズはコストゼロ
claude-free "このコードのテストを書いて"

# 最終レビュー・複雑なリファクタは本家 Sonnet
claude-pro "この PR の設計上の問題点を指摘して"
```

**ユースケース**：日常的なコード補完・ドキュメント生成・コメント追加など低リスクタスク

---

### パターン 2：`.claude/settings.json` でプロジェクト別モデルを固定

Claude Code はプロジェクトルートに `.claude/settings.json` を置くことでプロジェクト固有の設定を持てます。

```json
{
  "model": "google/gemini-2.0-flash-exp:free",
  "env": {
    "ANTHROPIC_BASE_URL": "https://openrouter.ai/api/v1"
  }
}
```

これにより、`cd project-a && claude` するだけで自動的に無料モデルが適用されます。有料モデルが必要なプロジェクトは別途上書き設定を置くだけです。

**ユースケース**：チーム開発でメンバーごとのキー管理を統一したいとき

---

### パターン 3：フォールバック付きシェル関数

無料枠はレートリミットがあるため、失敗時に自動で有料枠へ fallback する関数を組めます。

```bash
claude-smart() {
  # まず無料モデルで試みる
  ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1 \
  ANTHROPIC_API_KEY=$OPENROUTER_KEY \
  CLAUDE_MODEL=meta-llama/llama-3.3-70b-instruct:free \
  claude "$@"
  
  local exit_code=$?
  
  # レートリミット (429) や 5xx の場合は本家へ切り替え
  if [ $exit_code -ne 0 ]; then
    echo "⚠️  Falling back to Anthropic direct API..."
    ANTHROPIC_API_KEY=$ANTHROPIC_KEY claude "$@"
  fi
}
```

**ユースケース**：CI パイプライン内のコード品質チェックなど、失敗してはならないが頻度が低いタスク

---

### パターン 4：Makefile でタスク別モデルを分離

プロジェクトの `Makefile` に AI タスクを定義し、タスクの性質に応じてモデルを分けます。

```makefile
# 低コスト：テスト生成・コメント付与
.PHONY: ai-test ai-docs ai-review

ai-test:
	ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1 \
	ANTHROPIC_API_KEY=$(OPENROUTER_KEY) \
	claude --model qwen/qwen3-235b-a22b:free \
	  "src/ 配下の未テストの関数にユニットテストを生成して"

ai-docs:
	ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1 \
	ANTHROPIC_API_KEY=$(OPENROUTER_KEY) \
	claude --model google/gemini-2.0-flash-exp:free \
	  "JSDoc コメントを全関数に付与して"

# 高精度：アーキテクチャレビューは本家
ai-review:
	ANTHROPIC_API_KEY=$(ANTHROPIC_KEY) \
	claude --model claude-sonnet-4-5 \
	  "この PR の設計を Reviewして LGTM か修正点を返して"
```

**ユースケース**：チームで AI 活用を標準化したい・コストの見積もりをタスク単位で管理したい場合

---

### パターン 5：Python スクリプトで動的ルーティング

より高度なユースケースでは、トークン数やタスク種別に応じて動的にモデルを選択するラッパーを Python で書けます。

```python
#!/usr/bin/env python3
"""
OpenRouter × Anthropic Claude Code のスマートルーター。
タスク種別とプロンプト長でモデルを自動選択。
依存: anthropic>=0.25.0
"""

import anthropic
import os
import sys

FREE_MODELS = [
    "qwen/qwen3-235b-a22b:free",
    "meta-llama/llama-3.3-70b-instruct:free",
    "google/gemini-2.0-flash-exp:free",
]

PAID_MODEL = "claude-sonnet-4-5"

def select_model(prompt: str) -> tuple[str, str | None]:
    """
    Returns: (model_name, base_url)
    base_url が None のときは Anthropic 本家
    """
    token_estimate = len(prompt.split()) * 1.3
    
    # 長いコンテキスト (>40k tokens 相当) は Gemini Flash Free
    if token_estimate > 40_000:
        return "google/gemini-2.0-flash-exp:free", "https://openrouter.ai/api/v1"
    
    # コードレビュー・設計判断はキーワードで判定して本家へ
    critical_keywords = ["アーキテクチャ", "セキュリティ", "パフォーマンス最適化", "本番"]
    if any(kw in prompt for kw in critical_keywords):
        return PAID_MODEL, None
    
    # それ以外は無料モデル
    return FREE_MODELS[0], "https://openrouter.ai/api/v1"


def run(prompt: str) -> str:
    model, base_url = select_model(prompt)
    
    kwargs = {
        "api_key": os.environ["OPENROUTER_KEY"] if base_url else os.environ["ANTHROPIC_KEY"],
    }
    if base_url:
        kwargs["base_url"] = base_url
    
    client = anthropic.Anthropic(**kwargs)
    
    print(f"→ Using model: {model}", file=sys.stderr)
    
    message = client.messages.create(
        model=model,
        max_tokens=4096,
        messages=[{"role": "user", "content": prompt}],
    )
    return message.content[0].text


if __name__ == "__main__":
    prompt = " ".join(sys.argv[1:]) or sys.stdin.read()
    print(run(prompt))
```

**ユースケース**：バッチ処理・自動コードレビューボット・Slack 連携 bot など

---

### パターン 6：GitHub Actions で無料枠フル活用

PR が作成されたとき自動でコードレビューコメントを付ける Actions。無料モデルを使えばコストはゼロです。

```yaml
# .github/workflows/ai-review.yml
name: AI Code Review (Free Tier)

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Install Claude Code CLI
        run: npm install -g @anthropic-ai/claude-code

      - name: Get diff
        id: diff
        run: |
          git diff origin/${{ github.base_ref }}...HEAD -- '*.ts' '*.tsx' '*.py' '*.rs' \
            | head -c 8000 > /tmp/diff.txt
          echo "diff=$(cat /tmp/diff.txt)" >> $GITHUB_OUTPUT

      - name: Run AI Review
        env:
          ANTHROPIC_BASE_URL: https://openrouter.ai/api/v1
          ANTHROPIC_API_KEY: ${{ secrets.OPENROUTER_KEY }}
        run: |
          cat /tmp/diff.txt | claude \
            --model meta-llama/llama-3.3-70b-instruct:free \
            --print \
            "以下の diff をレビューして、バグ・型安全性の問題・可読性の改善点を箇条書きで指摘してください。問題なければ '✅ LGTM' とだけ返してください。" \
            > /tmp/review.txt
          cat /tmp/review.txt

      - name: Post Comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const review = fs.readFileSync('/tmp/review.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## 🤖 AI Review (llama-3.3-70b)\n\n${review}`,
            });
```

**ユースケース**：OSS プロジェクト・スタートアップの小規模チームでの CI/CD 自動化

---

### パターン 7：DeepSeek-R1 Free で複雑な推論タスクを委譲

`deepseek/deepseek-r1:free` は **Chain-of-Thought 推論**が強く、アルゴリズム選択・データ構造の設計・バグの根本原因分析など「じっくり考えさせたい」タスクに向いています。

```bash
# 複雑なバグ調査
git log --oneline -20 | claude \
  --model deepseek/deepseek-r1:free \
  "このコミット履歴と以下のエラーログから、バグが混入したコミットを特定し、修正方針を提案してください。"
```

R1 モデルは `<think>` タグ内で内部推論を行い、最終的な回答のみ出力します（OpenRouter 経由では thinking tokens はカウントされません）。

**ユースケース**：パフォーマンスボトルネックの原因分析・難読レガシーコードの解読

---

## コスト感のまとめ

| タスク | 推奨モデル | コスト |
|---|---|---|
| コメント付与・リファクタ提案 | Qwen3-235B:free | ¥0 |
| テスト自動生成 | Llama-3.3-70B:free | ¥0 |
| 長文ファイル一括解析 | Gemini-2.0-Flash:free | ¥0 |
| 複雑なアルゴリズム設計 | DeepSeek-R1:free | ¥0 |
| セキュリティレビュー・本番アーキテクチャ決定 | claude-sonnet-4-5 (有料) | ~$0.003/1k tokens |
| 超複雑な長文推論 | claude-sonnet-4-5 extended thinking | 要精査 |

**実感として**、日常的なコーディングタスクの 70〜80% は `:free` モデルで十分です。残り 20〜30% の「判断が重要なタスク」に有料枠を集中させると、月のコストを大幅に抑えられます。

---

## 注意点と落とし穴

### 1. モデルの互換性問題

Claude Code は内部で Anthropic 独自の tool_use 形式（`tool_use` type）を使うことがあります。OpenRouter 経由のモデルによっては tool_use の互換性が不完全なケースがあります。

対処法：`claude --no-tools` フラグで tool 系機能をオフにするか、ファイル操作系の機能（`Edit`, `Create` など）を多用するタスクには本家 Anthropic を使う。

### 2. プロンプトキャッシュが効かない

Anthropic 本家は Prompt Caching（同一プレフィックスを 5 分間キャッシュ・90% 削減）をサポートしますが、OpenRouter 経由では原則このキャッシュは機能しません。長いシステムプロンプトを繰り返す場合はコスト計算が変わります。

### 3. データプライバシー

OpenRouter は各モデルプロバイダーへリクエストをルーティングします。機密コード（顧客データを含む実装など）を送る場合は利用規約・データ保持ポリシーを必ず確認してください。OSS 開発・学習用途では問題ありません。

### 4. レートリミットの現実

`:free` モデルは混雑時に `429 Too Many Requests` を返します。CI で安定稼働させたい場合は、1 秒あたり 1 リクエスト程度に throttle するか、複数の無料モデルをラウンドロビンする設計が有効です。

---

## まとめ

| # | パターン | 難易度 | 効果 |
|---|---|---|---|
| 1 | Shell エイリアス | ★☆☆ | 即効・個人開発向け |
| 2 | `.claude/settings.json` | ★☆☆ | チーム設定の統一 |
| 3 | フォールバック関数 | ★★☆ | 安定性向上 |
| 4 | Makefile タスク分離 | ★★☆ | コスト可視化 |
| 5 | Python 動的ルーター | ★★★ | 高度な自動化 |
| 6 | GitHub Actions CI | ★★☆ | 0 円自動レビュー |
| 7 | DeepSeek-R1 推論委譲 | ★★☆ | 複雑タスク強化 |

Claude Code の完成度の高い UX と、OpenRouter の豊富な無料モデルは相性が良く、工夫次第でコストを大幅に削減しながら AI コーディングの恩恵を受けられます。まずはパターン 1 のエイリアスから試してみてください。

---

## 参考リンク

- [Claude Code 公式ドキュメント](https://docs.anthropic.com/ja/docs/claude-code/overview) — Anthropic
- [OpenRouter モデル一覧](https://openrouter.ai/models) — OpenRouter
- [Anthropic Python SDK](https://github.com/anthropic-sdk/anthropic-python) — MIT License
- [DeepSeek-R1 技術レポート](https://github.com/deepseek-ai/DeepSeek-R1) — MIT License
- [Llama 3.3 Model Card](https://huggingface.co/meta-llama/Llama-3.3-70B-Instruct) — Meta Llama License 3.3
- [Qwen3 技術レポート](https://qwenlm.github.io/blog/qwen3/) — Apache 2.0

> 本記事で紹介したコードスニペットはすべて一般公開された OSS・公式 API の使い方の例示であり、特定のサービス固有の構成・機密情報は含みません。

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします！
