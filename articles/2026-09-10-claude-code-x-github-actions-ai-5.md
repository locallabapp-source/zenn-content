---
title: "Claude Code × GitHub Actions で実現する AI コードレビュー自動化 — 5 ステップ完全ガイド"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- GitHub Actions の `pull_request` イベントに Claude Code (claude-code-action) を統合し、PR 差分を自動レビューさせる
- Anthropic 公式の `claude-ai/claude-code-action` を使えば 50 行未満のワークフロー定義で動作する
- レビューコメントの粒度・フォーカス領域は `SYSTEM_PROMPT` で制御でき、チーム固有のコーディング規約を注入できる
- 無料枠 (OpenRouter 経由) と組み合わせれば月額コストをほぼゼロに抑えられる

---

## 背景 — なぜ AI コードレビューを CI に組み込むのか

コードレビューにかかる工数は、規模が大きくなるほど指数的に増える。  
1 日に数十本の PR を捌くチームでは、レビュアーの認知負荷が「本当に重要な設計指摘」ではなく「typo 修正・命名の揺れ・未使用 import」の指摘に消費されがちだ。

AI コードレビューを CI パイプラインに組み込む目的は、**人間のレビュアーの注意を高付加価値な部分に集中させること**にある。

```
PR 作成
  └─ (自動) AI が差分をレビュー → "obvious issues" を先に潰す
       └─ (人間) 設計・ドメイン知識が必要な部分のみレビュー
```

---

## 前提知識

| 項目 | 必要なもの |
|---|---|
| GitHub リポジトリ | Actions 有効化済 |
| Anthropic API キー | 環境変数 `ANTHROPIC_API_KEY` として GitHub Secrets に登録済 |
| 対象言語 | 問わない (プロンプトで調整) |
| Actions 経験 | `on:`, `jobs:`, `steps:` の基本が分かる程度 |

---

## Step 1 — GitHub Secrets に API キーを登録する

リポジトリの **Settings → Secrets and variables → Actions → New repository secret** から:

```
Name  : ANTHROPIC_API_KEY
Value : sk-ant-...（Anthropic Console から取得）
```

> **注意**: シークレット名は後述のワークフロー YAML 内で参照するため、名前の一致を確認すること。

OpenRouter 経由で無料モデルを使いたい場合は `OPENROUTER_API_KEY` も追加し、Step 4 で base URL を変更する。

---

## Step 2 — ワークフローファイルを作成する

`.github/workflows/ai-review.yml` を以下の内容で作成する:

```yaml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0          # 差分取得に必要

      - name: Run Claude Code Review
        uses: claude-ai/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          system_prompt: |
            あなたはシニアエンジニアとしてコードレビューを行います。
            以下の観点で指摘してください:
            1. バグ・ロジックエラー (MUST FIX)
            2. セキュリティリスク (MUST FIX)
            3. パフォーマンス上の懸念 (SUGGESTION)
            4. 可読性・命名 (SUGGESTION)
            明らかに問題のない変更には LGTM と返してください。
```

このワークフローは:

1. PR が `opened` または `synchronize`（新コミットが push された）タイミングで起動する
2. `fetch-depth: 0` によりリポジトリの全履歴を取得し、ベースブランチとの差分が正確に計算される
3. `claude-ai/claude-code-action` が差分を取得し、Anthropic API に投げ、結果を PR コメントとして書き込む

---

## Step 3 — system_prompt をチームに最適化する

デフォルトのシステムプロンプトは汎用的すぎる場合が多い。  
チーム固有のコーディング規約を注入することで、AI のレビューコメントを自チームのスタイルに合わせられる。

### 例: TypeScript + React チーム向け

```yaml
system_prompt: |
  あなたはシニアフロントエンドエンジニアです。
  対象スタックは TypeScript / React / Next.js (App Router) です。

  MUST FIX (レビューを通さないブロッカー):
  - `any` 型の使用
  - `useEffect` の deps 配列の欠落
  - `console.log` の残存
  - XSS / CSRF リスク

  SUGGESTION (任意対応):
  - `React.memo` / `useMemo` の最適化機会
  - コンポーネントの責務が大きすぎる場合の分割提案
  - Tailwind クラスの重複

  コード変更が存在しない場合や単純な設定変更のみの場合は LGTM のみ返してください。
  日本語でコメントしてください。
```

### 例: Go バックエンド向け

```yaml
system_prompt: |
  あなたはシニア Go エンジニアです。

  MUST FIX:
  - error を無視している箇所 (_, _ =)
  - goroutine リークの可能性 (context キャンセルなし)
  - nil ポインタデリファレンスのリスク
  - SQL インジェクションの可能性

  SUGGESTION:
  - `defer` の配置が適切か
  - インターフェース設計の改善余地
  - テストカバレッジが明らかに落ちる変更

  コメントは日本語で、指摘箇所はファイル名+行番号で明示してください。
```

---

## Step 4 — OpenRouter 経由でコストを削減する (オプション)

Anthropic API を直接叩くと `claude-3-7-sonnet` クラスのモデルが動作し、PR 1 件あたり数円〜数十円のコストが発生する。  
OpenRouter の `:free` モデル (`qwen/qwen3-235b-a22b:free` 等) を使えば、月数千件の PR でも無料枠内に収められる場合がある。

`claude-code-action` は `base_url` パラメータをサポートしているため:

```yaml
- name: Run Claude Code Review
  uses: claude-ai/claude-code-action@v1
  with:
    anthropic_api_key: ${{ secrets.OPENROUTER_API_KEY }}
    base_url: "https://openrouter.ai/api/v1"
    model: "qwen/qwen3-235b-a22b:free"
    github_token: ${{ secrets.GITHUB_TOKEN }}
    system_prompt: |
      ...（上記と同様）
```

> **注意**: 無料モデルはレート制限が厳しく、大きな差分では応答が遅延または失敗する場合がある。  
> コアな機能リポジトリには有料モデル、実験的リポジトリには無料モデルという使い分けが現実的だ。

---

## Step 5 — 細かいチューニングと運用上の Tips

### 5-1. 大きすぎる PR をスキップする

差分が数千行を超えると API のトークン上限に達してエラーになる。  
あらかじめ差分行数でジョブをスキップする条件を入れておくと安定する:

```yaml
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Check diff size
        id: diff_check
        run: |
          LINES=$(git diff origin/${{ github.base_ref }}...HEAD --stat | tail -1 | grep -oP '\d+ insertion' | grep -oP '\d+' || echo 0)
          echo "lines=$LINES" >> $GITHUB_OUTPUT

      - name: Run Claude Code Review
        if: steps.diff_check.outputs.lines < '2000'
        uses: claude-ai/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          system_prompt: "..."
```

### 5-2. ラベルで AI レビューをオプトアウトする

`no-ai-review` ラベルが付いた PR はスキップするフィルタを追加すると、自動生成コミットやドキュメント修正 PR での無駄な消費を防げる:

```yaml
on:
  pull_request:
    types: [opened, synchronize, labeled, unlabeled]

jobs:
  review:
    if: "!contains(github.event.pull_request.labels.*.name, 'no-ai-review')"
    runs-on: ubuntu-latest
    ...
```

### 5-3. レビューコメントをスレッドにまとめる

デフォルトでは `claude-code-action` は PR 全体コメントとして投稿する。  
`review_type: inline` オプション (v1.2 以降) を有効にすると、差分の該当行に直接インラインコメントとして投稿されるため、視認性が向上する:

```yaml
with:
  review_type: inline
```

### 5-4. 特定ファイルを除外する

ロックファイル・自動生成コード・移行スクリプトなどはレビュー対象から外す:

```yaml
with:
  exclude_patterns: |
    package-lock.json
    yarn.lock
    **/*.generated.ts
    migrations/**
```

---

## 実際に動かした際の挙動イメージ

PR を作成すると、数十秒〜2 分以内に以下のようなコメントがボットアカウント (`github-actions[bot]`) から投稿される:

```
## 🤖 AI Code Review

### 🚨 MUST FIX

**`src/auth/login.ts` L42**
パスワードを平文で `console.log` に出力しています。
本番環境での情報漏洩リスクがあります。直ちに削除してください。

---

### 💡 SUGGESTION

**`src/components/UserList.tsx` L87**
`users.filter(...).map(...)` のチェーンは毎レンダリングで再計算されます。
`useMemo` でメモ化するとパフォーマンスが改善します。

---

### ✅ その他の変更

スタイル修正・テストの追加は問題ありません。LGTM 🎉
```

---

## アーキテクチャ全体像

```
PR push
  │
  ├─ GitHub Actions トリガー
  │     └─ actions/checkout (fetch-depth: 0)
  │          └─ git diff でベースブランチとの差分を取得
  │
  ├─ claude-code-action
  │     ├─ 差分テキスト + system_prompt を Anthropic API へ送信
  │     └─ レスポンスを GitHub Pull Request Review API で投稿
  │
  └─ PR にレビューコメントが追記される
```

`claude-code-action` の内部では `@anthropic-ai/sdk` を Node.js で直接呼び出しており、差分の前処理（バイナリファイルの除外・トークン数の見積もり）も含まれている。  
ソースコードは GitHub で公開されているため、カスタマイズが必要な場合はフォークして使うことも可能だ。

---

## まとめ

| ステップ | 内容 | 所要時間 |
|---|---|---|
| Step 1 | GitHub Secrets に API キー登録 | 2 分 |
| Step 2 | ワークフロー YAML 作成 | 5 分 |
| Step 3 | system_prompt のチューニング | 15〜30 分 |
| Step 4 | OpenRouter 切り替え (任意) | 5 分 |
| Step 5 | 差分サイズ制限・ラベル除外 | 10 分 |

初期設定は 30 分もあれば完了し、翌日から PR ごとに AI の一次レビューが自動化される。  
**人間のレビュアーを「一次フィルター」から解放し、「設計判断・ドメイン知識」に集中させる**のが最大のメリットだ。

また、`system_prompt` を Git 管理することで「チームのコーディング規約がコードとして定義される」副次的な効果もある。  
まずは実験的なリポジトリで試し、チームのフィードバックをもとに `system_prompt` を育てていくのがおすすめだ。

---

## 参考リンク

- [claude-ai/claude-code-action — GitHub](https://github.com/claude-ai/claude-code-action)
- [Anthropic API Reference — Messages](https://docs.anthropic.com/en/api/messages)
- [GitHub Actions — Encrypted Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [OpenRouter — Free Models](https://openrouter.ai/models?q=free)
- [GitHub Pull Request Review API](https://docs.github.com/en/rest/pulls/reviews)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
