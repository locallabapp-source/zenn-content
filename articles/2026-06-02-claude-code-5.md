---
title: "Claude Code を自律エージェントとして使い倒す5つの設計パターン"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Claude Code はチャット補完ツールではなく「エージェントランタイム」として設計されている
- `CLAUDE.md` による文脈注入・サブエージェント分業・フック機構を組み合わせると自律度が大幅に上がる
- 5 つの設計パターンを実装例とともに解説する

---

## はじめに

Claude Code が正式リリースされてから、多くのエンジニアが「コード補完を速くするツール」として使っている。しかし公式ドキュメントを読み込むと、設計の重心は全く別のところにある。

**Claude Code はエージェントランタイムである。**

ターミナル上で動作し、ファイルシステム・シェル・外部 API を自律的に呼び出し、複数タスクをネストして実行できる。この記事では「ただ質問する」から「自律的に動かす」への移行に必要な 5 つの設計パターンを紹介する。

対象読者は Claude Code をすでに日常利用しており、次のステップを探しているエンジニア。

---

## パターン 1: CLAUDE.md による文脈の永続化

### 問題

Claude Code はセッションをまたいで文脈を保持しない。毎回「このプロジェクトは何をするものか」「どのコマンドでテストを回すか」を説明し直すのは非効率だ。

### 解決策

プロジェクトルートに `CLAUDE.md` を置く。Claude Code は起動時にこのファイルを自動読み込みする。

```markdown
# Project Context

## Stack
- Runtime: Node.js 22 / TypeScript 5.4
- Framework: Next.js 15 (App Router)
- DB: PostgreSQL 16 + Drizzle ORM
- Test: Vitest + Playwright

## 開発コマンド
- `pnpm dev` — 開発サーバー起動 (port 3000)
- `pnpm test` — unit test (Vitest)
- `pnpm test:e2e` — E2E test (Playwright)
- `pnpm lint` — ESLint + Prettier

## コーディング規約
- `any` 型の使用禁止。`unknown` + 型ガードで代替
- Server Component / Client Component を明示的に分離
- DB アクセスは必ず `src/db/` 配下の関数経由

## Agent へのお願い
- 変更前に影響範囲を一言説明してから実装に入ること
- テスト未カバーのコードを追加する場合は必ず test も書くこと
```

### 効果

- セッション開始コストがゼロになる
- 規約の口頭説明が不要になり、新メンバーのオンボーディングにも使える
- `CLAUDE.md` 自体をリポジトリに含めることでチーム共有できる

### 発展: 階層配置

`CLAUDE.md` はサブディレクトリに複数置ける。

```
project/
  CLAUDE.md          ← プロジェクト全体のルール
  src/
    api/
      CLAUDE.md      ← API レイヤー固有のルール
    ui/
      CLAUDE.md      ← UI コンポーネント固有のルール
```

Claude Code はカレントディレクトリから上方向に探索して全階層の `CLAUDE.md` を読み込む。作業ディレクトリを変えるだけで文脈が切り替わる。

---

## パターン 2: サブタスク分解による並列処理

### 問題

「この機能を実装して」という大きな指示は、Claude Code が途中で迷子になりやすい。特に依存関係が複数あるタスクでは顕著だ。

### 解決策

**タスクを明示的に分解してから投入する。**

```
以下のタスクを順番に実行してください:

1. `src/types/user.ts` に `UserRole` 型を追加
   - 値: 'admin' | 'member' | 'viewer'
2. `src/db/users.ts` の `findById` 関数の戻り値に `role` フィールドを追加
3. Drizzle schema (`src/db/schema.ts`) に `role` カラムを追加
   - デフォルト: 'member'
4. マイグレーションファイルを生成 (`pnpm drizzle-kit generate`)
5. 影響を受ける既存テストをすべて修正
6. `src/db/users.test.ts` に role 別クエリのテストを追加

各ステップ完了後に次へ進む前に一言報告してください。
```

### なぜ効果があるか

Claude Code は各ステップを「完了条件付きサブゴール」として扱う。フリーフォームな指示より、条件が明確なタスクのほうがエラー率が下がる。これはエージェント設計の基本原則「具体的な完了条件を持つ小タスクに分解する」そのもの。

### 発展: エージェント間委譲

Claude Code には `--dangerously-skip-permissions` フラグと組み合わせたヘッドレス実行モードがある。これを使うと、Claude Code 自身が別の Claude Code プロセスをスポーンしてサブタスクを委譲できる。公式ではこれを **multi-agent ネットワーク** と呼んでいる。

実用上は Bash スクリプトや Makefile で複数の `claude` コマンドをパイプしていくシンプルな構成から始めるとよい。

---

## パターン 3: カスタムスラッシュコマンドで作業を定型化

### 問題

毎回同じ種類の作業 (PR レビュー・バグ調査・リリース準備) を自然言語で説明するのは手間だし、品質がばらつく。

### 解決策

`.claude/commands/` ディレクトリにコマンドファイルを置くと、`/project:コマンド名` で呼び出せるカスタムスラッシュコマンドになる。

```
.claude/
  commands/
    review.md
    debug.md
    release-check.md
```

**`.claude/commands/review.md` の例:**

```markdown
# コードレビュー

以下の観点で $ARGUMENTS (指定がなければ `git diff HEAD~1`) を確認してください。

## チェック項目
- [ ] 型安全性: `any` / 型キャストの不適切な使用
- [ ] セキュリティ: SQL インジェクション・XSS・CSRF の可能性
- [ ] パフォーマンス: N+1 クエリ・不要な再レンダリング
- [ ] テスト: 新規ロジックに対応するテストの有無
- [ ] アクセシビリティ: aria 属性・キーボード操作

## 出力フォーマット
Markdown テーブルで指摘事項をまとめ、
重大度を 🔴高 / 🟡中 / 🟢低 で付与してください。
致命的な問題がなければ最後に「✅ LGTM」と書いてください。
```

使い方:

```
/project:review src/api/payment.ts
```

### 効果

- レビュー観点の属人化を防げる
- `$ARGUMENTS` で対象ファイルを動的に渡せる
- `.claude/commands/` ごとリポジトリ管理すればチーム全員が同じコマンドを使える

---

## パターン 4: Hooks でエージェントの行動を制約する

### 問題

「自律的に動く」と「意図しない変更をされる」は表裏一体だ。特に本番環境への影響があるコマンドを Claude Code が勝手に実行するリスクは常に存在する。

### 解決策

Claude Code の Settings JSON に `hooks` を定義すると、特定のツール実行前後にシェルスクリプトを差し込める。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/pre-bash-guard.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/post-write-lint.sh"
          }
        ]
      }
    ]
  }
}
```

**`~/.claude/hooks/pre-bash-guard.sh` の例:**

```bash
#!/usr/bin/env bash
# Claude Code が実行しようとしているコマンドを stdin で受け取る
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // ""')

# 危険なコマンドパターンを検出したら exit 1 でブロック
DANGER_PATTERNS=(
  "rm -rf /"
  "DROP TABLE"
  "kubectl delete"
  "> /dev/null 2>&1 &"   # バックグラウンドプロセスの隠蔽
)

for pattern in "${DANGER_PATTERNS[@]}"; do
  if echo "$COMMAND" | grep -qF "$pattern"; then
    echo "🚨 BLOCKED: 危険なパターンを検出しました: $pattern" >&2
    exit 2  # exit 2 で Claude へエラーメッセージを返せる
  fi
done

exit 0
```

**`~/.claude/hooks/post-write-lint.sh` の例:**

```bash
#!/usr/bin/env bash
INPUT=$(cat)
FILE=$(echo "$INPUT" | jq -r '.tool_input.path // ""')

# TypeScript ファイルが書き込まれたら自動 lint
if [[ "$FILE" == *.ts || "$FILE" == *.tsx ]]; then
  pnpm eslint "$FILE" --fix 2>&1
fi
```

### Hooks の設計思想

| Hook タイミング | 主な用途 |
|---|---|
| `PreToolUse` | 危険なコマンドのブロック・ドライラン確認 |
| `PostToolUse` | lint / format / テスト自動実行 |
| `Notification` | Slack / LINE 通知 |
| `Stop` | エージェント終了時のクリーンアップ |

`exit 0` で続行、`exit 1` で警告（実行は継続）、`exit 2` でエラーとして Claude に通知 + 実行ブロック、という 3 段階になっている。

---

## パターン 5: 長期タスクを --print モードで CI に組み込む

### 問題

Claude Code はインタラクティブな対話ツールとして使うのが基本だが、「毎朝 PR のリストを確認して優先度付けする」「毎週月曜に依存関係の脆弱性レポートを作る」といった定期処理に組み込みたいケースがある。

### 解決策

`--print` フラグ (`-p`) を使うと非インタラクティブモードで動作し、出力を stdout に吐き出して終了する。CI や cron と相性がよい。

```bash
# 脆弱性レポートの自動生成例 (cron で週次実行)
claude \
  --print \
  --model claude-opus-4-5 \
  "package.json と pnpm-lock.yaml を読んで、\
   既知の脆弱性がある依存関係を調べ、\
   severity / CVE 番号 / 推奨バージョンを Markdown テーブルで報告してください。" \
  > reports/vuln-$(date +%Y%m%d).md
```

GitHub Actions との統合例:

```yaml
name: Weekly Dep Review

on:
  schedule:
    - cron: '0 9 * * 1'  # 毎週月曜 09:00 UTC

jobs:
  dep-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Run dep review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude --print \
            "依存関係の更新が必要なパッケージを package.json から抽出し、\
             breaking change が予想されるものに ⚠️ を付けてください。" \
            > dep-review.md

      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: dep-review
          path: dep-review.md
```

### 注意点

- `--print` モードでは人間による承認プロンプトが表示されない
- ファイル書き込みを伴う操作は `--allowedTools` で明示的に許可する
- CI 環境では `ANTHROPIC_API_KEY` をシークレット管理する（環境変数直書きは禁止）
- コスト管理のため `--max-turns` でループ上限を設定することを推奨

```bash
claude \
  --print \
  --max-turns 10 \
  --allowedTools "Read,Bash(read-only)" \
  "..."
```

---

## 5 パターンの組み合わせ例

実際には各パターンを組み合わせることで効果が最大化される。

```
CLAUDE.md        → プロジェクト文脈の永続化
  ↓
カスタムコマンド  → 作業の定型化 (/project:review 等)
  ↓
タスク分解       → 大きい作業を確実に完遂
  ↓
Hooks            → 危険な操作をブロック & lint 自動実行
  ↓
--print モード   → CI/cron で定期レポート生成
```

たとえば「新機能ブランチのマージ前チェック」を自動化するなら:

1. `CLAUDE.md` でプロジェクト規約を定義済み
2. `/project:review` でレビュー観点を定型化済み
3. `PostToolUse` Hook で ESLint を自動実行
4. `--print` モードで CI から呼び出してレポートを PR コメントに投稿

この 4 層が揃うと、PR レビューの初期スクリーニングはほぼ無人で回せる。

---

## まとめ

| パターン | 解決する問題 | 実装コスト |
|---|---|---|
| CLAUDE.md 階層配置 | 文脈の毎回説明が不要に | 低 |
| タスク分解 | 大きな指示での迷子防止 | 低 |
| カスタムスラッシュコマンド | 作業の属人化解消 | 低 |
| Hooks による制約 | 意図しない操作のブロック | 中 |
| --print + CI 統合 | 定期作業の自動化 | 中 |

Claude Code を「聞くと答えるツール」から「設計して動かすランタイム」に切り替えると、1 日の開発速度の感覚が変わる。特に `CLAUDE.md` と Hooks は今日から導入できる投資対効果の高いパターンだ。

公式ドキュメント (https://docs.anthropic.com/ja/docs/claude-code/overview) は継続的に更新されており、Hooks の仕様は 2025 年に入ってから急速に充実した。定期的に確認することを勧める。

---

## 参考リンク

- [Claude Code 公式ドキュメント (Anthropic)](https://docs.anthropic.com/ja/docs/claude-code/overview)
- [Claude Code Hooks リファレンス](https://docs.anthropic.com/ja/docs/claude-code/hooks)
- [CLAUDE.md ガイド](https://docs.anthropic.com/ja/docs/claude-code/memory)
- [カスタムスラッシュコマンド仕様](https://docs.anthropic.com/ja/docs/claude-code/slash-commands)
- [Claude Code GitHub Actions インテグレーション](https://docs.anthropic.com/ja/docs/claude-code/github-actions)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします！
