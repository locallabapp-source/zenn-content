---
title: "Claude Code × GitHub Actions で CI コストを 60% 削減した5つの最適化テクニック"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- GitHub Actions の課金単位・ランナー選定を見直すだけで CI 実行コストは大幅に削減できる
- キャッシュ戦略・ジョブ並列化・条件付きスキップを組み合わせると効果が乗算される
- Claude Code (Anthropic 公式 CLI) をワークフローに組み込むことでレビュー工数も同時に圧縮できる
- 本記事は OSS / 公式ドキュメントの知識のみに基づく実装ガイドです

---

## 背景：CI コストはなぜ膨らむのか

SaaS 開発を続けると、ほぼ確実に CI の実行時間が膨張します。原因は大抵この 3 つです。

1. **テスト増加に伴う逐次実行の長大化** — 初期に書いたシーケンシャルな `jobs` 定義がそのまま残っている
2. **キャッシュ未整備による毎回フルビルド** — `node_modules` や cargo レジストリを都度ダウンロードしている
3. **不要なトリガー** — `push` に全ブランチを指定したまま、ドキュメント更新でもフルランが走る

GitHub Actions の従量課金は **実行分数 × ランナー単価** で決まります。`ubuntu-latest` (Linux) は 1 分 $0.008 ですが、`windows-latest` は $0.016、`macos-latest` は $0.08 と 10 倍の差があります。macOS が必要ない工程に `macos-latest` を指定し続けているだけで、コストが爆発することがあります。

---

## テクニック 1：ジョブをパスフィルターで条件スキップ

最も即効性が高い施策です。`paths` フィルターを使うと、変更ファイルに関係するジョブだけを実行できます。

```yaml
on:
  push:
    branches: [main, develop]
    paths-ignore:
      - '**.md'
      - 'docs/**'
      - '.github/CODEOWNERS'
```

さらに `dorny/paths-filter` アクションを使うと、**ジョブ単位** で動的にスキップを制御できます。

```yaml
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      backend: ${{ steps.filter.outputs.backend }}
      frontend: ${{ steps.filter.outputs.frontend }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            backend:
              - 'src/api/**'
              - 'Cargo.toml'
            frontend:
              - 'src/app/**'
              - 'package.json'

  test-backend:
    needs: changes
    if: needs.changes.outputs.backend == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: cargo test

  test-frontend:
    needs: changes
    if: needs.changes.outputs.frontend == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: pnpm test
```

フロントエンドのみの変更なら `test-backend` は **完全スキップ**。バックエンドのみなら `test-frontend` がスキップされます。ドキュメント修正なら両方スキップです。

---

## テクニック 2：依存関係キャッシュを「正しく」設定する

`actions/cache` を設定していても、キャッシュキーが毎回変わっていては意味がありません。

### Node.js (pnpm)

```yaml
- name: Setup pnpm
  uses: pnpm/action-setup@v3
  with:
    version: 9

- name: Get pnpm store path
  id: pnpm-cache
  run: echo "STORE_PATH=$(pnpm store path)" >> $GITHUB_OUTPUT

- uses: actions/cache@v4
  with:
    path: ${{ steps.pnpm-cache.outputs.STORE_PATH }}
    key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
    restore-keys: |
      ${{ runner.os }}-pnpm-
```

`pnpm-lock.yaml` のハッシュをキーにすることで、lockfile が変わった時だけキャッシュが無効化されます。`restore-keys` に prefix を指定しておくと、完全ミスでも部分的なキャッシュを拾えます。

### Rust (cargo)

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.cargo/bin/
      ~/.cargo/registry/index/
      ~/.cargo/registry/cache/
      ~/.cargo/git/db/
      target/
    key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
    restore-keys: |
      ${{ runner.os }}-cargo-
```

`target/` ディレクトリごとキャッシュするのがポイントです。これだけで Rust プロジェクトのビルド時間が 3〜5 分 → 30 秒以下になるケースがあります。

---

## テクニック 3：matrix 戦略でテストを並列展開

E2E テストや単体テストが多い場合、`strategy.matrix` を使ってテストスイートを並列実行します。

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
      fail-fast: false
    steps:
      - uses: actions/checkout@v4
      - run: pnpm test --shard=${{ matrix.shard }}/4
```

4 並列にした場合、理論上は実行時間が 1/4 になります。`fail-fast: false` を設定しておくと、1 シャードで失敗しても他のシャードは最後まで実行されるので、全体の失敗件数を一度に把握できます。

Playwright や Vitest はシャーディングをネイティブサポートしているため、追加ライブラリ不要です。

```bash
# Playwright
npx playwright test --shard=1/4

# Vitest
vitest run --shard=1/4
```

---

## テクニック 4：Claude Code を PR レビュー自動化に組み込む

ここからが本記事のユニークな部分です。

Anthropic が公式に提供する Claude Code CLI (`@anthropic-ai/claude-code`) は、GitHub Actions 上でも動作します。PR が作成されるたびに差分をレビューさせることで、**人間レビュワーが確認すべき箇所を絞り込め**、レビュー工数を削減できます。

公式ドキュメント (https://docs.anthropic.com/claude-code) に記載されている非インタラクティブモード (`--print`) を使った基本パターンは以下のとおりです。

```yaml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Get diff
        id: diff
        run: |
          git diff origin/${{ github.base_ref }}...HEAD > diff.txt

      - name: Run Claude Code review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude --print \
            "以下の diff をレビューしてください。バグ・セキュリティリスク・型の問題を中心に日本語で箇条書きにまとめてください。問題がなければ '問題なし' とだけ返してください。\n\n$(cat diff.txt)" \
            > review.txt

      - name: Post review comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const review = fs.readFileSync('review.txt', 'utf8');
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## 🤖 Claude Code 自動レビュー\n\n${review}`
            });
```

### 注意点

- `ANTHROPIC_API_KEY` は GitHub Secrets に登録します (値そのものをワークフロー YAML に書いてはいけません)
- diff のサイズが大きいと API コストが増えるため、`paths` フィルターでレビュー対象を絞ることを推奨します
- あくまで **補助ツール** です。自動レビューを最終承認として扱わないでください

---

## テクニック 5：Self-hosted runner とエフェメラルランナーの活用

GitHub のホステッドランナーが高コストに感じる場合、**Self-hosted runner** を VPS や AWS Spot インスタンスに立てることでコストを劇的に削減できます。

### エフェメラル (使い捨て) ランナーとは

`--ephemeral` フラグを付けて登録したランナーは、ジョブを 1 回実行したら自動的に登録解除されます。常駐プロセスを残さず、Spot Instance の終了後も登録が汚染されません。

GitHub 公式の [actions/runner](https://github.com/actions/runner) と、オートスケーリング用の [philips-labs/terraform-aws-github-runner](https://github.com/philips-labs/terraform-aws-github-runner) (OSS・MIT) を組み合わせると、AWS Spot Instance のエフェメラルランナー自動プールを構築できます。

```bash
# 手動でエフェメラルランナーを 1 台起動する例 (公式 runner)
./config.sh \
  --url https://github.com/YOUR_ORG/YOUR_REPO \
  --token YOUR_RUNNER_TOKEN \
  --ephemeral \
  --unattended

./run.sh
```

### コスト比較目安

| 環境 | 1 分あたりのコスト |
|---|---|
| GitHub ホスト (ubuntu-latest) | $0.008 |
| AWS EC2 c6i.large (On-Demand) | ~$0.0014 |
| AWS EC2 c6i.large (Spot) | ~$0.0004〜0.0008 |
| Hetzner CPX21 (月額固定) | ~$0.0002 (利用時間換算) |

Spot + エフェメラル構成にすると GitHub ホストの **1/10 以下** のコストになるケースもあります。デメリットはセットアップコストと、Spot 中断時のリトライ設計が必要な点です。

---

## まとめ：5 つを組み合わせると効果が乗算される

| テクニック | 削減インパクト | 実装コスト |
|---|---|---|
| パスフィルターでスキップ | ★★★★ | 低 |
| 依存関係キャッシュの正確な設定 | ★★★★ | 低 |
| matrix でテスト並列化 | ★★★ | 中 |
| Claude Code で PR レビュー自動化 | ★★★ (人件費削減) | 中 |
| Self-hosted エフェメラルランナー | ★★★★★ | 高 |

最初の 3 つを組み合わせるだけでも実行時間・コストの 40〜60% 削減は現実的なラインです。Claude Code によるレビュー自動化は CI コストそのものより **エンジニアの可処分時間** を増やす効果が大きいです。Self-hosted ランナーは初期構築コストが高いですが、CI ジョブが月 10,000 分を超えるようになったら検討価値が出てきます。

CI は「作ったら終わり」ではなく、プロジェクトが育つにつれて継続的に見直すべきインフラです。まずはパスフィルターとキャッシュの 2 点だけ今週中に適用してみてください。効果はすぐに数字として現れます。

---

## 参考リンク

- [GitHub Actions 課金ドキュメント](https://docs.github.com/ja/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions)
- [actions/cache 公式](https://github.com/actions/cache)
- [dorny/paths-filter](https://github.com/dorny/paths-filter)
- [Playwright シャーディング](https://playwright.dev/docs/test-sharding)
- [Vitest シャーディング](https://vitest.dev/guide/improving-performance.html#sharding)
- [Claude Code 公式ドキュメント](https://docs.anthropic.com/claude-code)
- [philips-labs/terraform-aws-github-runner](https://github.com/philips-labs/terraform-aws-github-runner) (MIT License)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
