---
title: "Claude Code × GitHub Actions で実現する AI レビュー自動化——5 ステップで CI に組み込む"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Claude Code の `claude` CLI はヘッドレス実行が可能で、CI 環境でも動作する
- GitHub Actions の `pull_request` トリガーと組み合わせると、PR 作成ごとに自動コードレビューが走る
- プロンプト設計・コスト制御・セキュリティの 3 点を押さえれば本番運用に耐えられる
- 今回紹介する構成はすべて公式ドキュメントと OSS の範囲内で完結する

---

## 背景：なぜ CI に AI レビューを組み込むのか

コードレビューのボトルネックはどのチームにも共通する課題だ。レビュワーが不在の夜間・週末にも PR は積まれ、レビュー待ちのまま翌朝を迎えることは珍しくない。

一方、LLM によるコードレビューは「精度が低い」「ハルシネーションが怖い」と敬遠されてきた。しかし 2025 年以降、Claude 3.x / Claude 4 系の文脈理解が飛躍的に向上し、**差分の意図を把握した上でのコメント** が実用レベルに達しつつある。

ポイントは「LLM に最終判断させない」こと。AI レビューは「初期スクリーニング」として使い、マージ判断は人間が行う——この役割分担が現実的だ。

---

## 使用する技術スタック

| コンポーネント | 役割 |
|---|---|
| **Anthropic Claude API** (claude-3-5-haiku / claude-3-5-sonnet) | コード差分の解析・コメント生成 |
| **GitHub Actions** | PR トリガーで CI ワークフローを起動 |
| **GitHub CLI (`gh`)** | PR へのコメント投稿 |
| **jq / bash** | 差分抽出・JSON 整形 |

Claude Code の `claude` CLI を直接使う方法と、`anthropic` SDK を Python スクリプトで呼ぶ方法の 2 通りがある。今回は後者のほうが CI で制御しやすいため Python スクリプトアプローチを採用する。

---

## ステップ 1：リポジトリシークレットを設定する

GitHub リポジトリの **Settings → Secrets and variables → Actions** に移動し、以下のシークレットを追加する。

| シークレット名 | 値 |
|---|---|
| `ANTHROPIC_API_KEY` | Anthropic Console で発行した API キー |

シークレット名は慣習的にすべて大文字スネークケースを使う。ワークフロー YAML 内では `${{ secrets.ANTHROPIC_API_KEY }}` の形で参照する。

---

## ステップ 2：差分抽出スクリプトを書く

`scripts/ai_review.py` として以下を配置する。依存パッケージは `anthropic` のみ。

```python
#!/usr/bin/env python3
"""
AI Code Review script for GitHub Actions.
Uses Anthropic API to review PR diffs and outputs markdown comments.
"""

import os
import sys
import anthropic

def build_review_prompt(diff: str, pr_title: str) -> str:
    return f"""You are a senior software engineer conducting a code review.
Review the following git diff from a pull request titled "{pr_title}".

Focus on:
1. Logic errors or potential bugs
2. Security vulnerabilities (injection, auth bypass, secrets in code)
3. Performance issues (N+1 queries, unnecessary loops)
4. Readability and naming conventions
5. Missing error handling

Respond in Japanese. Be concise. Use GitHub Markdown.
Start each issue with a severity label: 🔴 Critical / 🟡 Warning / 🔵 Info.
If no issues are found, write "✅ 特に指摘事項はありません。" only.

--- diff start ---
{diff[:12000]}
--- diff end ---
"""

def main() -> None:
    diff = sys.stdin.read()
    pr_title = os.environ.get("PR_TITLE", "Untitled PR")
    model = os.environ.get("REVIEW_MODEL", "claude-3-5-haiku-20241022")

    if not diff.strip():
        print("diff が空のため、レビューをスキップしました。")
        return

    client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

    message = client.messages.create(
        model=model,
        max_tokens=1024,
        messages=[
            {"role": "user", "content": build_review_prompt(diff, pr_title)}
        ],
    )

    print(message.content[0].text)

if __name__ == "__main__":
    main()
```

**ポイント解説:**

- `diff[:12000]` でコンテキストウィンドウを意識したトリミングを行う。haiku の入力コンテキストは 200k トークンだが、無制限に渡すとコストが跳ね上がる。大規模 diff では 12,000 文字程度を上限にするのが実用的だ
- `model` を環境変数で切り替え可能にしておくと、ブランチごとにコスト調整しやすい
- `max_tokens=1024` はレビューコメントの長さとして十分。ここを無闇に増やさない

---

## ステップ 3：GitHub Actions ワークフローを作成する

`.github/workflows/ai-review.yml` を以下のように記述する。

```yaml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]
    # ドキュメントのみの変更はスキップ
    paths-ignore:
      - '**.md'
      - 'docs/**'

permissions:
  pull-requests: write
  contents: read

jobs:
  ai-review:
    runs-on: ubuntu-latest
    # Draft PR は対象外
    if: github.event.pull_request.draft == false

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 差分取得のため全履歴が必要

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: pip install anthropic==0.40.0

      - name: Extract diff
        id: diff
        run: |
          git diff origin/${{ github.base_ref }}...HEAD \
            -- '*.py' '*.ts' '*.tsx' '*.js' '*.go' '*.rs' '*.rb' \
            > /tmp/pr.diff
          echo "lines=$(wc -l < /tmp/pr.diff)" >> $GITHUB_OUTPUT

      - name: Run AI review
        if: steps.diff.outputs.lines != '0'
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          PR_TITLE: ${{ github.event.pull_request.title }}
          REVIEW_MODEL: claude-3-5-haiku-20241022
        run: |
          cat /tmp/pr.diff | python scripts/ai_review.py > /tmp/review.md

      - name: Post review comment
        if: steps.diff.outputs.lines != '0'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          BODY=$(cat /tmp/review.md)
          gh pr comment ${{ github.event.pull_request.number }} \
            --body "## 🤖 AI コードレビュー (claude-3-5-haiku)

          ${BODY}

          ---
          > ⚠️ このコメントは AI が生成しました。最終判断は人間のレビュワーが行ってください。"
```

**ポイント解説:**

- `permissions: pull-requests: write` を明示する。デフォルトパーミッションが `read` のリポジトリでは、これがないとコメント投稿が 403 で失敗する
- `fetch-depth: 0` は重要。デフォルトの shallow clone では `git diff origin/main...HEAD` が正しく動作しない
- `paths-ignore` で Markdown / ドキュメントのみの PR を除外することで、無駄な API コールを防ぐ
- `if: github.event.pull_request.draft == false` で Draft PR をスキップ。作業中の差分を毎回レビューさせる必要はない

---

## ステップ 4：コスト制御とモデル選択

AI レビューを CI に組み込む際、最も懸念されるのがコストの膨張だ。以下の指針で制御する。

### モデル選択の考え方

| モデル | 用途 | 入力コスト (1M トークン) |
|---|---|---|
| claude-3-5-haiku-20241022 | 日常 PR の初期スクリーニング | $0.80 |
| claude-3-5-sonnet-20241022 | セキュリティ・重要機能の変更 | $3.00 |
| claude-opus-4 | 使用しない (コスト過大) | $15.00 |

通常の PR であれば **haiku で十分**。セキュリティ関連ファイル（`*auth*`, `*permission*`, `*crypto*`）の変更が含まれるときだけ sonnet に昇格させるロジックを追加するとコスト効率が高い。

```bash
# ファイル名でモデルを切り替える例
if git diff origin/$BASE...HEAD --name-only | grep -qE '(auth|permission|crypto|secret)'; then
  export REVIEW_MODEL="claude-3-5-sonnet-20241022"
else
  export REVIEW_MODEL="claude-3-5-haiku-20241022"
fi
```

### diff サイズによるスキップ

1,000 行を超える差分（大規模リファクタリング等）は AI レビューの精度が下がる傾向がある。スキップ判定を入れるのも一手だ。

```bash
DIFF_LINES=$(wc -l < /tmp/pr.diff)
if [ "$DIFF_LINES" -gt 1000 ]; then
  echo "diff が大きすぎるため AI レビューをスキップします (${DIFF_LINES} 行)"
  exit 0
fi
```

---

## ステップ 5：プロンプト設計のベストプラクティス

AI レビューの品質はプロンプトで 8 割が決まる。実運用から得た知見をまとめる。

### ✅ 効果的なプロンプトの構造

```
1. ロール定義 (senior engineer)
2. 具体的なレビュー観点のリスト
3. 出力フォーマットの指定 (severity label)
4. 長さの制約 ("be concise")
5. diff の境界を明確にするデリミタ
```

### ✅ ノイズを減らすための指示

```
- テストコードへの軽微な指摘は省略してください
- 既存のコーディングスタイルには言及しないでください
- 差分にないコードを推測して指摘しないでください
```

### ❌ やりがちな失敗

- **diff 全体を無条件に渡す**: コンテキストウィンドウを圧迫し、後半の diff への注意が散漫になる
- **「すべての問題を指摘して」と書く**: ノイズが増え、重要な指摘が埋もれる
- **出力フォーマットを指定しない**: コメントが長文になりすぎ、PR 上で読みにくくなる

---

## 発展：既存 OSS との比較

同様の目的で使われる OSS と本構成を比較する。

| ツール | 特徴 | 向いているケース |
|---|---|---|
| **[CodeRabbit](https://coderabbit.ai)** | SaaS・設定不要・詳細なレビュー | チームに設定担当者がいない場合 |
| **[pr-agent (CodiumAI)](https://github.com/Codium-ai/pr-agent)** | OSS・多モデル対応・高機能 | 細かいカスタマイズが必要な場合 |
| **本構成** | シンプル・フルコントロール・コスト透明 | 挙動を自分で制御したい・既存 CI に溶け込ませたい |

pr-agent は MIT ライセンスで公開されており、本構成と組み合わせることも可能だ。ただし依存が増えるため、「まず動かしてみたい」段階では今回のようなミニマル構成から始めることを推奨する。

---

## セキュリティ上の注意点

### diff に秘密情報が含まれるリスク

`.env` ファイルや API キーがコミットされた場合、その内容が Claude API に送信される。これを防ぐには以下の対策が有効だ。

```bash
# .env 系ファイルを diff から除外
git diff origin/$BASE...HEAD \
  -- ':!*.env' ':!**/.env.*' ':!**/secrets/**' \
  > /tmp/pr.diff
```

### フォークからの PR

パブリックリポジトリでフォークからの PR を受け付ける場合、`ANTHROPIC_API_KEY` は `pull_request` イベントでは自動的に利用不可になる（GitHub のセキュリティ仕様）。フォーク PR に対応するには `pull_request_target` イベントを使うが、コードの実行タイミングに注意が必要だ。詳細は GitHub 公式ドキュメントの [Keeping your GitHub Actions and workflows secure](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions) を参照。

---

## まとめ

| ステップ | 内容 |
|---|---|
| 1 | `ANTHROPIC_API_KEY` をリポジトリシークレットに登録 |
| 2 | diff を受け取り Claude API に投げる Python スクリプトを作成 |
| 3 | `pull_request` トリガーの GitHub Actions ワークフローを定義 |
| 4 | モデル選択と diff サイズ制限でコストを制御 |
| 5 | プロンプトを観点・フォーマット・制約の 3 点セットで設計 |

AI レビューは「人間レビュワーの代替」ではなく「レビュー前の前処理」と位置づけることで、チームの生産性向上に自然に溶け込む。まず haiku + 最小プロンプトで動かし、精度やノイズ比を観察しながらチューニングするアプローチが実践的だ。

---

## 参考リンク

- [Anthropic Python SDK — anthropic-sdk-python](https://github.com/anthropic/anthropic-sdk-python) (MIT License)
- [GitHub Actions: Encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [GitHub Actions: Security hardening](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [CodiumAI pr-agent](https://github.com/Codium-ai/pr-agent) (Apache-2.0 License)
- [Anthropic Models Overview](https://docs.anthropic.com/en/docs/about-claude/models)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
