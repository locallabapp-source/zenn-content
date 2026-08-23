---
title: "Claude Code を 5 つの設定で劇的に使いやすくする — `.claude/` カスタマイズ完全ガイド 2026"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- `CLAUDE.md` にプロジェクト仕様を書くと Claude Code が文脈を保持する
- `.claude/commands/` にスラッシュコマンドを追加すると繰り返し作業が 1 行になる
- `settings.json` の `allowedTools` でツール許可を細かく制御できる
- `hooks` でコミット前自動フォーマット・テスト実行が組み込める
- これら 5 つを組み合わせると「毎回同じ説明をする」無駄が消える

---

## なぜ Claude Code の設定が重要か

Claude Code は対話型 AI コーディングツールだが、何も設定しないまま使うと毎セッションで「このプロジェクトは Next.js 15 App Router を使っています」「Rust の edition は 2021 です」といった前提を何度も伝え直すことになる。

Anthropic が 2025 年後半に整備した `.claude/` ディレクトリ仕様を活用すると、この問題が解消される。本記事では 5 つの設定ファイル・機能を順番に解説する。

---

## 設定 1 — `CLAUDE.md` でプロジェクト文脈を固定する

`CLAUDE.md` はリポジトリルートに置くマークダウンファイルで、Claude Code がセッション開始時に自動で読み込む。

```markdown
# プロジェクト概要

Next.js 15 (App Router) + TypeScript 5.5 + Tailwind CSS v4。
バックエンドは Rust (axum 0.8 / tokio 1.x)。

## コーディング規約

- コンポーネントは `src/components/<FeatureName>/index.tsx` に配置
- Server Component がデフォルト、クライアント処理が必要な場合のみ `"use client"` を付与
- エラーハンドリングは `Result<T, AppError>` パターンを統一

## よく使うコマンド

| 目的 | コマンド |
|---|---|
| 開発サーバ起動 | `pnpm dev` |
| テスト全実行 | `pnpm test` |
| Rust ビルド | `cargo build --release` |

## 絶対にやってはいけないこと

- `any` 型の使用
- `console.log` をコミットに含める
- `unsafe {}` ブロックの無断使用
```

### ポイント

- **箇条書き中心**で書くと Claude が要点を取りこぼしにくい
- 「やってはいけないこと」セクションは特に効果的。禁止事項を明示すると提案コードの品質が上がる
- 1 ファイルで長くなりすぎる場合は `.claude/README.md` に分割する (サブディレクトリの `CLAUDE.md` もネストして読まれる)

---

## 設定 2 — `.claude/commands/` でスラッシュコマンドを自作する

`.claude/commands/` 以下に `.md` ファイルを置くと、Claude Code の `/` コマンドとして呼び出せるカスタムプロンプトになる。

**ファイル構成例**

```
.claude/
└── commands/
    ├── review.md
    ├── gen-test.md
    └── explain-error.md
```

**`.claude/commands/review.md`**

```markdown
あなたはシニアエンジニアとして、以下のコードレビューを行ってください。

観点:
1. バグ・ロジックエラーの可能性
2. パフォーマンス上の問題 (不要な再レンダリング / O(n²) アルゴリズム等)
3. 型安全性 (TypeScript / Rust)
4. プロジェクトのコーディング規約 (CLAUDE.md 参照) との整合性

フォーマット:
- 問題点は重大度 🔴🟡🟢 で分類
- 各指摘に「なぜ問題か」「どう直すか」を併記
- 問題がなければ「LGTM」と 1 行で返す
```

このファイルを保存した後、Claude Code のプロンプトで `/review` と入力するだけでこのプロンプトが展開される。

**`.claude/commands/gen-test.md`** (引数付き例)

```markdown
以下のファイル・関数に対するユニットテストを生成してください: $ARGUMENTS

要件:
- テストフレームワーク: vitest (TypeScript) または cargo test (Rust)
- 正常系・異常系・境界値を網羅
- モックは最小限にとどめ、実際の動作に近い形で書く
- テストファイルは `<対象ファイル>.test.ts` または `tests/<モジュール名>.rs` に配置
```

`$ARGUMENTS` はコマンド実行時に引数を受け取るプレースホルダー。`/gen-test src/utils/formatter.ts` のように使う。

---

## 設定 3 — `settings.json` でツール許可を細粒度に制御する

`.claude/settings.json` でツールのアクセス権限を管理できる。デフォルトは対話確認が多いが、信頼できる操作をあらかじめ許可すると自動化が進む。

```json
{
  "allowedTools": [
    "Bash(git diff:*)",
    "Bash(git log:*)",
    "Bash(pnpm test:*)",
    "Bash(cargo test:*)",
    "Bash(cargo clippy:*)",
    "Read",
    "Write",
    "Edit"
  ],
  "deniedTools": [
    "Bash(rm -rf:*)",
    "Bash(curl:*)",
    "WebFetch"
  ]
}
```

### allowedTools のパターン記法

| パターン | 意味 |
|---|---|
| `"Bash(git diff:*)"` | `git diff` から始まるコマンドをすべて許可 |
| `"Bash(pnpm test:*)"` | `pnpm test` から始まるコマンドをすべて許可 |
| `"Read"` | ファイル読み込み操作を全許可 |
| `"Write"` | ファイル書き込み操作を全許可 |

`deniedTools` は `allowedTools` より優先される。破壊的な操作 (`rm -rf`) や外部通信 (`curl` / `WebFetch`) を明示的に拒否することで、意図しないサイドエフェクトを防げる。

### チームで共有する際の注意

`.claude/settings.json` はリポジトリにコミットしてチーム全体に適用できるが、各開発者のローカル設定を上書きしたい場合は `~/.claude/settings.json` (ユーザーグローバル設定) が優先される仕様を理解しておくこと。

---

## 設定 4 — `hooks` でコミット前処理を自動化する

`settings.json` の `hooks` セクションを使うと、Claude Code が特定のアクション (ファイル保存・コマンド実行前後) にシェルスクリプトを自動実行できる。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "pnpm lint --fix $CLAUDE_TOOL_OUTPUT_FILE 2>&1 | head -20"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash(git commit:*)",
        "hooks": [
          {
            "type": "command",
            "command": "pnpm test --run 2>&1 | tail -20"
          }
        ]
      }
    ]
  }
}
```

**動作イメージ**

1. Claude Code がファイルを書き換える (`Write` / `Edit`)
   → 自動で `pnpm lint --fix` が走り、フォーマットが整う
2. `git commit` コマンドを実行しようとする
   → 事前に `pnpm test --run` が走り、テストが通ってからコミット

CI と同じ品質ゲートをローカルの Claude Code 操作に組み込めるのが強み。

### hooks で使える環境変数

| 変数 | 内容 |
|---|---|
| `$CLAUDE_TOOL_OUTPUT_FILE` | 直前に書き込まれたファイルパス |
| `$CLAUDE_HOOK_EVENT` | イベント名 (`PostToolUse` 等) |
| `$CLAUDE_TOOL_NAME` | 実行されたツール名 |

---

## 設定 5 — `.claude/` サブディレクトリ CLAUDE.md でモジュール別コンテキストを管理する

モノレポや複数サービスが混在するリポジトリでは、ルートの `CLAUDE.md` だけでは文脈が発散する。サブディレクトリに `CLAUDE.md` を置くと、そのディレクトリ内で Claude Code が作業する際に追加コンテキストとして読み込まれる。

```
repo-root/
├── CLAUDE.md              ← 全体共通設定
├── apps/
│   ├── web/
│   │   └── CLAUDE.md      ← Next.js フロントエンド固有の設定
│   └── api/
│       └── CLAUDE.md      ← Rust axum バックエンド固有の設定
└── packages/
    └── ui/
        └── CLAUDE.md      ← shadcn/ui コンポーネントライブラリ固有の設定
```

**`apps/api/CLAUDE.md` の例**

```markdown
# API サービス固有設定

このディレクトリは Rust (axum 0.8) で書かれた REST API です。

## ディレクトリ構造

```
src/
├── main.rs         // エントリポイント
├── routes/         // ハンドラ
├── models/         // DB モデル (sqlx)
└── errors.rs       // AppError 定義
```

## DB マイグレーション

`sqlx migrate run` で適用。マイグレーションファイルは `migrations/` に追加。

## エラーハンドリング規約

`AppError` に variant を追加して `impl IntoResponse` を更新すること。
`unwrap()` / `expect()` は src 内では禁止 (tests/ 内は許可)。
```

ルートの `CLAUDE.md` → サブディレクトリの `CLAUDE.md` の順で読み込まれ、より具体的な設定が後から上書き・補完される形になる。

---

## 5 つの設定をまとめて適用する最小テンプレート

```
.claude/
├── commands/
│   ├── review.md
│   ├── gen-test.md
│   └── summarize-pr.md
└── settings.json

CLAUDE.md
```

**セットアップ手順**

```bash
# 1. ディレクトリ作成
mkdir -p .claude/commands

# 2. settings.json の初期化
cat > .claude/settings.json << 'EOF'
{
  "allowedTools": ["Read", "Write", "Edit", "Bash(git diff:*)", "Bash(git log:*)"],
  "deniedTools": ["Bash(rm -rf:*)", "WebFetch"]
}
EOF

# 3. CLAUDE.md を作成 (プロジェクトに合わせて編集)
touch CLAUDE.md
touch .claude/commands/review.md
```

---

## よくある落とし穴と対処法

### CLAUDE.md が長すぎてコンテキストを圧迫する

Claude Code は CLAUDE.md を毎セッション読み込むため、長すぎると他の作業への利用可能コンテキストが減る。**目安は 200 行以内**。長くなる場合は「コーディング規約は `docs/CONTRIBUTING.md` を参照」と参照先を書くにとどめるとよい。

### hooks のコマンドが失敗してもブロックされる

`hooks` の `command` が非ゼロ終了するとデフォルトでツール実行がブロックされる。副作用として Claude Code の操作が止まることがある。フォーマッターのようにエラーを無視してよい場合は `command || true` でラップする。

```json
"command": "pnpm lint --fix $CLAUDE_TOOL_OUTPUT_FILE || true"
```

### `.claude/` を `.gitignore` に入れてしまう

チームで同じ設定を共有するには `.claude/` をリポジトリに含める必要がある。個人設定 (API キーなど機密情報) は `~/.claude/` のグローバル設定に書き、チーム共通設定だけをリポジトリの `.claude/` に置くのがベストプラクティス。

---

## まとめ

| 設定ファイル | 主な効果 |
|---|---|
| `CLAUDE.md` | プロジェクト文脈の自動注入・規約の固定 |
| `.claude/commands/*.md` | 繰り返しプロンプトのスラッシュコマンド化 |
| `.claude/settings.json` (allowedTools) | ツール許可の細粒度制御 |
| `.claude/settings.json` (hooks) | 保存時フォーマット・コミット前テストの自動化 |
| サブディレクトリ `CLAUDE.md` | モノレポ・マルチサービスの文脈分離 |

5 つを組み合わせることで「Claude Code に毎回同じ説明をする」コストがほぼゼロになる。特に `CLAUDE.md` + カスタムコマンドの組み合わせは即効性が高く、今日から試せる。

---

## 参考リンク

- [Claude Code — Overview | Anthropic Documentation](https://docs.anthropic.com/claude-code/overview)
- [CLAUDE.md best practices | Anthropic Documentation](https://docs.anthropic.com/claude-code/memory)
- [Claude Code settings reference](https://docs.anthropic.com/claude-code/settings)
- [Claude Code hooks](https://docs.anthropic.com/claude-code/hooks)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
