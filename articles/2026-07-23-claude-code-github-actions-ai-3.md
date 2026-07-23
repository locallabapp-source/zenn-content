---
title: "Claude Code + GitHub Actions で実現する AI レビュー自動化：3 ステップで導入できる実践ガイド"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Claude Code の `claude` CLI を GitHub Actions から呼び出し、PR レビューを自動化できる
- 設定ファイル 3 つ・ステップ 3 つで最小構成が動く
- レビューコメントの品質を上げる `CLAUDE.md` の書き方も解説

---

## なぜ AI コードレビューを自動化するのか

コードレビューには時間がかかる。特にチームが小さいほど、レビュー待ちがボトルネックになりやすい。

Anthropic が提供する **Claude Code**（`@anthropic-ai/claude-code`）は CLI ツールとして配布されており、ローカルだけでなく CI/CD パイプラインにも組み込める。GitHub Actions と組み合わせれば、PR が作成された瞬間に Claude が差分を読んで「初回レビュー」を自動投稿してくれる環境を構築できる。

人間レビュアーが見る前に：

- 明らかなバグやタイポを潰す
- コーディング規約の逸脱を指摘する
- テスト漏れを検出する

この「0 次レビュー」があるだけで、人間レビュアーの集中を「設計判断」に向けられる。

---

## 前提知識

- GitHub Actions の基本（YAML・`on:` トリガー・`secrets` の使い方）
- Node.js / npm の基礎
- Claude API キーを取得済み（[https://console.anthropic.com](https://console.anthropic.com)）

---

## 全体アーキテクチャ

```
PR 作成 / 更新
    │
    ▼
GitHub Actions (ubuntu-latest)
    │
    ├─ [1] リポジトリ checkout
    ├─ [2] Node.js セットアップ + claude CLI インストール
    ├─ [3] git diff で差分取得
    ├─ [4] claude CLI へ差分をパイプ
    └─ [5] GitHub API でレビューコメント投稿
```

外部への通信は `api.anthropic.com` のみ。差分テキストをそのまま Claude に渡すだけなので、シークレットは `ANTHROPIC_API_KEY` 1 つで済む。

---

## ステップ 1：ワークフローファイルを作成する

`.github/workflows/ai-review.yml` を作成する。

```yaml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write

jobs:
  ai-review:
    runs-on: ubuntu-latest
    # ドラフト PR はスキップ（任意）
    if: github.event.pull_request.draft == false

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0   # git diff のために全履歴が必要

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install claude CLI
        run: npm install -g @anthropic-ai/claude-code

      - name: Generate diff
        id: diff
        run: |
          git diff origin/${{ github.base_ref }}...HEAD \
            -- '*.ts' '*.tsx' '*.js' '*.jsx' '*.py' '*.go' '*.rs' \
            > diff.txt
          echo "lines=$(wc -l < diff.txt)" >> $GITHUB_OUTPUT

      - name: Run AI review
        if: steps.diff.outputs.lines > '0'
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          cat diff.txt | claude --print \
            --system-prompt "$(cat .github/review-prompt.md)" \
            > review.md

      - name: Post review comment
        if: steps.diff.outputs.lines > '0'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const body = fs.readFileSync('review.md', 'utf8');
            await github.rest.pulls.createReview({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number,
              body: `## 🤖 AI レビュー (Claude)\n\n${body}`,
              event: 'COMMENT'
            });
```

### ポイント解説

| 設定 | 理由 |
|---|---|
| `fetch-depth: 0` | `git diff origin/main...HEAD` には全履歴が必要 |
| `permissions: pull-requests: write` | レビューコメントの投稿に必要な最小権限 |
| `-- '*.ts' '*.tsx' ...` | 拡張子フィルタで差分サイズを削減・コストを抑える |
| `if: draft == false` | ドラフト PR への無駄なレビューを防ぐ |

---

## ステップ 2：レビュープロンプトを作成する

`.github/review-prompt.md` を作成する。このファイルがレビュー品質の心臓部になる。

```markdown
あなたは経験豊富なソフトウェアエンジニアです。
提供された git diff を読み、以下の観点でコードレビューを行ってください。

## レビュー観点（優先順）

1. **バグ・ロジックエラー**
   - 境界値・null/undefined チェック漏れ
   - 非同期処理の await 忘れ・競合状態
   - 型の不一致・暗黙のキャスト

2. **セキュリティ**
   - インジェクション（SQL・コマンド・XSS）
   - 認証・認可の抜け
   - 機密情報のハードコード

3. **パフォーマンス**
   - N+1 クエリ
   - ループ内での不要な計算
   - 大きなオブジェクトの不必要なコピー

4. **可読性・保守性**
   - 関数・変数の命名
   - 複雑すぎるネスト
   - コメントと実装の乖離

## 出力フォーマット

### ✅ 良い点
（差分の中で特筆すべき良いコードがあれば）

### 🚨 必須修正
（バグ・セキュリティ問題など、マージ前に直すべきもの）

ファイル名と行番号を明記：
- `src/api/users.ts:42` — `userId` が null の場合に ...

### ⚠️ 提案
（パフォーマンス・可読性の改善提案。マージを妨げないもの）

### 📝 備考
（差分から読み取れる文脈・前提の確認など）

---

差分がテストファイルのみの場合は「テストのみの変更です」と 1 行で返してください。
差分が空の場合は「レビュー対象の差分がありません」と 1 行で返してください。
```

---

## ステップ 3：CLAUDE.md でプロジェクト固有知識を渡す

Claude Code はリポジトリルートの `CLAUDE.md` を自動的に読み込む。ここにプロジェクト固有の制約を書くことで、レビューの精度が大きく上がる。

```markdown
# プロジェクト規約

## 言語・ランタイム
- TypeScript 5.x / Node.js 20
- React 18 + Next.js 14 App Router

## コーディング規約
- `any` 型は原則禁止。使う場合は `// eslint-disable-next-line` コメントと理由を添える
- コンポーネントは Named export のみ（default export 禁止）
- データ取得は Server Component 内で行う（Client Component でのフェッチは原則禁止）

## エラーハンドリング
- API ルートでは必ず `try/catch` + `NextResponse.json({ error: '...' }, { status: 5xx })` を返す
- ユーザー向けエラーメッセージに内部スタックトレースを含めない

## テスト
- ビジネスロジックを含む関数には Vitest によるユニットテストが必須
- テストファイルは `__tests__/` ではなく `*.test.ts` の命名規則

## セキュリティ
- `process.env` へのアクセスは `src/config/env.ts` を経由する（直接参照禁止）
- Prisma の生クエリ（`$queryRaw`）は使用禁止
```

`CLAUDE.md` に書いた内容はすべてのセッション（ローカル・CI 両方）で有効なので、ここをメンテするだけで AI レビューの「チューニング」ができる。

---

## 差分サイズとコストの管理

差分が大きすぎると API のコンテキスト上限に達したり、コストが膨らんだりする。以下の対策が有効。

### 拡張子フィルタ（再掲）

```bash
git diff origin/${{ github.base_ref }}...HEAD \
  -- '*.ts' '*.tsx' '*.py' '*.go' '*.rs'
```

ロック系ファイル（`package-lock.json` / `go.sum` / `Cargo.lock`）は自動的に除外される。

### 差分行数で打ち切る

```yaml
- name: Truncate large diffs
  run: |
    head -n 2000 diff.txt > diff_truncated.txt
    mv diff_truncated.txt diff.txt
```

2,000 行（約 100KB 程度）を目安にすると、`claude-haiku-3-5` でも 1 回 $0.01 以下に収まりやすい。

### モデルを指定する

コストを抑えたい場合は `--model` オプションで軽量モデルを指定できる。

```bash
claude --print --model claude-haiku-3-5 \
  --system-prompt "$(cat .github/review-prompt.md)" \
  < diff.txt > review.md
```

| モデル | 精度 | コスト感 | ユースケース |
|---|---|---|---|
| `claude-opus-4` | ◎ | 高 | 重要リポジトリのリリースブランチ |
| `claude-sonnet-4` | ○ | 中 | デイリーの PR |
| `claude-haiku-3-5` | △ | 低 | 量の多い小規模修正 |

---

## よくあるトラブルと対処法

### `ANTHROPIC_API_KEY` が認識されない

`Settings → Secrets and variables → Actions → New repository secret` から `ANTHROPIC_API_KEY` を登録しているか確認する。環境変数名のタイポ（`ANTHROPIC_API_KEYS` など）も多い。

### `git diff` が空になる

`fetch-depth: 0` を忘れると `origin/main` のコミット情報がなく、diff が空になる。`actions/checkout@v4` の `with:` に `fetch-depth: 0` を追加する。

### レビューコメントが重複投稿される

`synchronize` イベント（push のたびに発火）と `opened` イベントを両方指定しているのが原因。重複を防ぐには「最新のレビューコメントを更新する」方式に切り替えるか、Bot の既存コメントを削除してから新規投稿するロジックを追加する。

```javascript
// actions/github-script 内で既存コメントを削除する例
const comments = await github.rest.issues.listComments({
  owner: context.repo.owner,
  repo: context.repo.repo,
  issue_number: context.issue.number,
});
const botComments = comments.data.filter(
  c => c.user.type === 'Bot' && c.body.includes('🤖 AI レビュー')
);
for (const c of botComments) {
  await github.rest.issues.deleteComment({
    owner: context.repo.owner,
    repo: context.repo.repo,
    comment_id: c.id,
  });
}
```

### PR に大量のノイズが出る

レビュープロンプトの「提案」セクションが饒舌になりやすい。`review-prompt.md` に次のような制約を加えると落ち着く。

```
提案は最大 5 件まで。軽微なスタイルの指摘は省略してください。
```

---

## 発展：セキュリティスキャンとの併用

Claude レビューはあくまで「最初の目」。`npm audit` や `trivy`、`semgrep` などの静的解析ツールと組み合わせるのが現実的な運用。

```yaml
jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Trivy
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

  ai-review:
    needs: security-scan   # セキュリティスキャンが通過してから AI レビュー
    # ...
```

`needs:` で依存関係を設定すると、重大な脆弱性があるときは AI レビューがそもそも走らないため、トークンを無駄遣いしない。

---

## まとめ

今回紹介した構成で実現できることを整理する。

| 目的 | 手段 |
|---|---|
| AI によるゼロ次レビュー | `claude --print < diff.txt` |
| プロジェクト規約の注入 | `CLAUDE.md` + `--system-prompt` |
| コスト最適化 | 拡張子フィルタ・差分打ち切り・モデル指定 |
| ノイズ削減 | プロンプトの出力件数制限 |
| 重複投稿防止 | 既存 Bot コメント削除ロジック |

**3 ファイル**（`ai-review.yml` / `review-prompt.md` / `CLAUDE.md`）を追加するだけで動く最小構成から始めて、運用しながらプロンプトを育てていくのが最もうまくいくパターンだ。

---

## 参考リンク

- [Claude Code 公式ドキュメント](https://docs.anthropic.com/ja/docs/claude-code/overview)
- [GitHub Actions: permissions for pull-requests](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/controlling-permissions-for-github_token)
- [actions/github-script](https://github.com/actions/github-script)
- [aquasecurity/trivy-action](https://github.com/aquasecurity/trivy-action)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
