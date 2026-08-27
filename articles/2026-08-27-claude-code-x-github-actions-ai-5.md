---
title: "Claude Code × GitHub Actions で実現する AI レビューパイプライン — 5 ステップで導入するコードレビュー自動化"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Claude Code を GitHub Actions 上で動かすことで PR のコードレビューを自動化できる
- 設定ファイル 3 枚 + ワークフロー 1 本で完結する最小構成を解説
- ハルシネーションを減らす prompt 設計と、コスト爆発を防ぐ guard の実践的な書き方も紹介

---

## 背景 — なぜ AI レビューを CI に組み込むのか

コードレビューは品質ゲートとして不可欠な一方、**レビュアーのボトルネック**になりやすい。特に小規模チームでは 1 人が複数 PR をさばかなければならず、深夜の PR が翌朝まで積み残される、というサイクルに悩むエンジニアは多い。

AI レビューを CI に組み込むと次の 2 点が解消する。

1. **即時フィードバック** — `git push` から数分でレビューコメントが届く
2. **機械的指摘のオフロード** — typo / unused import / 明らかな N+1 を人間が指摘しなくて済む

重要な前提として、AI レビューは**人間レビューの代替ではなくサポート**である。設計の妥当性・ビジネスロジックの正確性は引き続き人間が判断する。

---

## 構成概要

```
GitHub PR 作成 / 更新
    │
    ▼
GitHub Actions Workflow (pr-review.yml)
    │
    ├─ diff 取得 (git diff --unified=5 origin/main...HEAD)
    │
    ├─ Claude API 呼び出し (claude-code-sdk / REST)
    │
    └─ PR コメント投稿 (GitHub CLI / REST)
```

使用技術スタック:

| 項目 | 選択 |
|---|---|
| LLM | Anthropic Claude 3.5 Haiku (コスト重視) |
| ランタイム | GitHub Actions (ubuntu-latest) |
| 言語 | Bash + jq (追加依存ゼロ) |
| PR コメント | `gh pr review --comment` |

なぜ Haiku か? Sonnet は高精度だがコストが高く、大型 PR で diff が数千行になると 1 回の呼び出しで $0.1〜0.5 かかる。Haiku は精度が多少落ちるが**速くて安く**、日常的な CI 用途には十分である。

---

## ステップ 1 — シークレット設定

GitHub リポジトリの **Settings → Secrets and variables → Actions** に以下を登録する。

| シークレット名 | 値 |
|---|---|
| `ANTHROPIC_API_KEY` | Anthropic Console で発行したキー |

> **注意**: キーを YAML にハードコードしないこと。`${{ secrets.ANTHROPIC_API_KEY }}` の形式でのみ参照する。

---

## ステップ 2 — ワークフローファイル

`.github/workflows/pr-review.yml` を作成する。

```yaml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]
    # ドラフト PR はスキップ
    branches:
      - main
      - develop

jobs:
  review:
    runs-on: ubuntu-latest
    # GITHUB_TOKEN に PR コメント権限を付与
    permissions:
      contents: read
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # 全履歴が必要

      - name: Install dependencies
        run: |
          sudo apt-get install -y jq
          # gh CLI は ubuntu-latest に同梱済み

      - name: Get diff
        id: diff
        run: |
          git fetch origin ${{ github.base_ref }}
          DIFF=$(git diff --unified=5 origin/${{ github.base_ref }}...HEAD \
            -- '*.ts' '*.tsx' '*.js' '*.py' '*.go' '*.rs' \
            | head -c 12000)   # 12KB キャップ (トークン爆発防止)
          echo "diff<<EOF" >> $GITHUB_OUTPUT
          echo "$DIFF" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Review with Claude
        id: ai_review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          DIFF='${{ steps.diff.outputs.diff }}'

          PAYLOAD=$(jq -n \
            --arg diff "$DIFF" \
            '{
              model: "claude-haiku-4-5",
              max_tokens: 1024,
              messages: [
                {
                  role: "user",
                  content: ("あなたは熟練したコードレビュアーです。\n以下の git diff をレビューし、問題点を箇条書きで簡潔に日本語で指摘してください。\n指摘は「重大 🔴」「改善提案 🟡」「軽微 🟢」の 3 段階で分類すること。\n問題がない場合は「LGTM ✅」とだけ返してください。\n\n```diff\n" + $diff + "\n```")
                }
              ]
            }')

          RESPONSE=$(curl -s https://api.anthropic.com/v1/messages \
            -H "x-api-key: $ANTHROPIC_API_KEY" \
            -H "anthropic-version: 2023-06-01" \
            -H "content-type: application/json" \
            -d "$PAYLOAD")

          REVIEW=$(echo "$RESPONSE" | jq -r '.content[0].text')
          echo "review<<EOF" >> $GITHUB_OUTPUT
          echo "$REVIEW" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Post PR comment
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          BODY="## 🤖 AI Code Review\n\n${{ steps.ai_review.outputs.review }}\n\n---\n_このコメントは AI により自動生成されました。最終判断は人間のレビュアーが行います。_"
          gh pr comment ${{ github.event.pull_request.number }} \
            --body "$BODY" \
            --repo ${{ github.repository }}
```

---

## ステップ 3 — diff のトークン上限ガード

大型 PR (マイグレーションファイル追加・自動生成コードなど) では diff が数万行になることがある。前のステップで `head -c 12000` を使ったが、より精密に制御するには以下のパターンが有効だ。

```bash
#!/usr/bin/env bash
# scripts/trim-diff.sh

MAX_BYTES=10000

RAW_DIFF=$(git diff --unified=3 origin/${BASE_BRANCH}...HEAD \
  -- '*.ts' '*.tsx' '*.js' '*.py' '*.go' '*.rs' \
  ':!*.lock' ':!*-lock.json' ':!*.min.js' ':!dist/*' ':!*.generated.*')

BYTE_COUNT=${#RAW_DIFF}

if [ "$BYTE_COUNT" -gt "$MAX_BYTES" ]; then
  TRIMMED="${RAW_DIFF:0:$MAX_BYTES}"
  echo "${TRIMMED}"
  echo ""
  echo "⚠️  diff が ${BYTE_COUNT} bytes のため ${MAX_BYTES} bytes に切り詰めました。全差分は GitHub 上で確認してください。"
else
  echo "${RAW_DIFF}"
fi
```

除外パターンのポイント:

| パターン | 理由 |
|---|---|
| `*.lock` / `*-lock.json` | パッケージロックは自動生成で人間が読まない |
| `*.min.js` | minified コードは AI がレビューしても無意味 |
| `dist/*` | ビルド成果物 |
| `*.generated.*` | protobuf / GraphQL 自動生成 |

---

## ステップ 4 — Prompt チューニング

「どんな指摘でもいいので返してください」という素朴な prompt は **ハルシネーションが多くノイジー**になりやすい。実運用で有効だった制約を 3 つ紹介する。

### 4-1. 役割定義を具体化する

```
あなたは TypeScript / React を専門とするシニアエンジニアです。
セキュリティ (XSS / CSRF / SQL インジェクション)、パフォーマンス、可読性の観点からレビューしてください。
```

ジェネリックな「熟練エンジニア」より、**言語と観点を明示**した方が的外れな指摘が減る。

### 4-2. 出力フォーマットを固定する

```
出力は以下の JSON 配列のみ返してください (他の文章は一切含めないこと):
[
  {
    "severity": "critical|warning|info",
    "file": "src/foo.ts",
    "line_hint": "L23-L27",
    "message": "..."
  }
]
問題がなければ空配列 [] を返してください。
```

JSON で返させると、後続の jq 処理で**重大度別にフィルタリング**できる。`critical` だけを PR ブロック条件にする、といった運用が可能になる。

### 4-3. 禁止事項を明示する

```
以下は指摘しないでください:
- セミコロンの有無 (ESLint が担当)
- import の順序 (Prettier が担当)
- コメントの有無・文体
```

Linter / Formatter が既にカバーしている点を AI に重複指摘させない。**ツールの役割分担**を prompt に書くことでノイズが減る。

---

## ステップ 5 — コスト監視

Claude API の費用は**トークン量 × 呼び出し回数**に比例する。アクティブなリポジトリでは 1 日 50〜100 PR という規模になることがあり、無制限に動かすと月数万円になるケースがある。

### 5-1. concurrency でキューイング制限

```yaml
concurrency:
  group: ai-review-${{ github.ref }}
  cancel-in-progress: true
```

同一ブランチへの連続 push で古いジョブをキャンセルし、**最新コミットのみレビュー**する。

### 5-2. ラベルで opt-in 制御

```yaml
on:
  pull_request:
    types: [labeled]

jobs:
  review:
    if: contains(github.event.pull_request.labels.*.name, 'ai-review')
```

`ai-review` ラベルが付いた PR だけを対象にすることで、**意図しない呼び出しを防ぐ**。

### 5-3. monthly spend アラート

Anthropic Console の **Usage Limits** でソフトリミット (例: $20/月) を設定しておく。ハードリミット到達前にメール通知が来るので予算オーバーを防げる。

---

## 実運用で得られた知見

### ✅ うまくいったこと

- **型安全性の問題** (`as any` の乱用・型アサーション漏れ) の指摘精度が高い
- **未使用変数・デッドコード**の検出は Haiku でも十分
- レビュアーが「これは言いにくいな…」と感じがちな **命名規則の一貫性**を機械的に指摘できる

### ⚠️ 注意が必要なこと

- **ドメイン固有ロジック**の妥当性判断は苦手 (「この計算式が業務要件に合っているか」は AI には判断できない)
- diff の冒頭しか渡っていない場合、**コンテキスト不足**で的外れな指摘が増える
- 同一ファイルを毎 PR 変更するケースで**繰り返し同じ指摘**が来ることがある → システム prompt に `以前の指摘の繰り返しは避けてください` を追加で抑制可

### コスト実績の目安

小規模チーム (PR 10本/日, 平均 diff 3KB) の場合:

| モデル | 1 PR あたりコスト | 月額概算 |
|---|---|---|
| claude-haiku-4-5 | ~$0.002 | ~$0.6 |
| claude-3-5-sonnet | ~$0.02 | ~$6 |

Haiku で十分な品質が出るなら Sonnet は不要。diff 全体を渡す必要がある大規模リポジトリのみ Sonnet を使うという使い分けが現実的だ。

---

## まとめ

| ステップ | 内容 |
|---|---|
| 1 | シークレット登録 (`ANTHROPIC_API_KEY`) |
| 2 | ワークフロー作成 (diff 取得 → API 呼び出し → PR コメント) |
| 3 | diff トークンキャップで自動生成ファイルを除外 |
| 4 | Prompt に役割・フォーマット・禁止事項を明示 |
| 5 | concurrency / ラベル / spend アラートでコスト管理 |

AI コードレビューは**「レビュアーの代替」ではなく「1 次フィルタ」**として使うのが正解だ。機械的・反復的な指摘を AI にオフロードすることで、人間のレビュアーは設計や業務ロジックの深いレビューに集中できる。

まずは小規模なリポジトリで Haiku + ラベル opt-in 構成で試してみてほしい。

---

## 参考リンク

- [Anthropic Messages API リファレンス](https://docs.anthropic.com/en/api/messages)
- [GitHub Actions: 暗号化シークレット](https://docs.github.com/ja/actions/security-guides/encrypted-secrets)
- [GitHub CLI `gh pr comment`](https://cli.github.com/manual/gh_pr_comment)
- [Claude モデル一覧と料金](https://docs.anthropic.com/en/docs/about-claude/models)
- [actions/checkout fetch-depth オプション](https://github.com/actions/checkout#usage)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
