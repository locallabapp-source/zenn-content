---
title: "Claude Code + Git Worktree で並列レビューを3倍速にする実践テクニック5選"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- `git worktree` でブランチを並列チェックアウトすると、Claude Code が **同時に複数コンテキストを保持** できる
- PR レビュー待ち時間をほぼゼロにしつつ、LLM コストを抑える構成を解説
- 5 つのテクニックを順番に組み合わせることで、1 人開発でも「セルフレビュー → マージ → 次の実装」のサイクルが 3 倍速になった

---

## 背景

Claude Code は **カレントディレクトリを意識コンテキストの基点** として動作する。  
1 つのリポジトリで複数ブランチを行き来すると、チェックアウトのたびにファイルが切り替わり、Claude が「どの状態を指しているのか」を混同しやすい。

`git worktree` はこの問題をきれいに解決する。**1 リポジトリ・複数ディレクトリ** でそれぞれ異なるブランチを同時に展開できるため、Claude を複数ターミナルで起動すれば、実質的に**ブランチ別に独立したエージェントコンテキスト**を持てる。

本記事では「Claude Code × git worktree」の組み合わせを最大限に活かす 5 つのテクニックを解説する。

---

## 前提知識

### git worktree とは

```bash
# メインの作業ディレクトリ (main ブランチ)
/projects/myapp/           ← git clone 済みのリポジトリ

# worktree で追加した作業ディレクトリ (feature/new-api ブランチ)
/projects/myapp-feat-api/  ← 同じ .git を共有・別ディレクトリ
```

`.git` オブジェクトは **1 つのみ**。ディスク容量をほぼ増やさず、ブランチごとに独立したファイルシステムのビューを持てる。

```bash
# worktree を追加する基本コマンド
git worktree add ../myapp-feat-api feature/new-api

# 一覧表示
git worktree list

# 削除
git worktree remove ../myapp-feat-api
```

---

## テクニック 1: worktree ごとに Claude Code セッションを割り当てる

最も基本的な使い方。ターミナルを 2 枚開き、それぞれ別 worktree をカレントディレクトリにして Claude Code を起動する。

```
ターミナル A: cd /projects/myapp         → Claude (main の不具合修正)
ターミナル B: cd /projects/myapp-feat-api → Claude (新機能実装)
```

**なぜ有効か:**  
Claude Code はプロンプトに添付するファイルをカレントディレクトリから探す。ディレクトリが分かれていれば「どっちのブランチの話か」を明示しなくて済む。コンテキストウィンドウを別ブランチの差分で汚染しない。

---

## テクニック 2: `CLAUDE.md` を worktree 単位で書き分ける

各 worktree のルートに異なる `CLAUDE.md` を置くと、Claude にブランチ固有の指示を与えられる。

```
/projects/myapp/CLAUDE.md           → 「このブランチは hotfix 専用。テストを必ず通してから返答して」
/projects/myapp-feat-api/CLAUDE.md  → 「新機能開発中。OpenAPI スキーマを最初に確認してから提案して」
```

`CLAUDE.md` の一例 (hotfix ブランチ用):

```markdown
# Context
- このブランチは production hotfix 専用です
- 変更ファイルは最小限にしてください
- 提案コードは必ず既存のテストが通ることを確認してください

# Stack
- Runtime: Node.js 22 (LTS)
- Test: Vitest
- CI: GitHub Actions (`.github/workflows/test.yml` 参照)
```

> **注意**: `CLAUDE.md` はリポジトリにコミットしないか `.gitignore` に追加し、worktree 間で上書きされないようにすること。

---

## テクニック 3: `git diff --stat` を Claude のレビュー起点にする

PR をセルフレビューするとき、変更全体を口頭で説明するより `git diff` の出力を直接渡す方が Claude の精度が上がる。

worktree があれば **ブランチを切り替えずに** diff を取れる。

```bash
# feature ブランチと main の差分を取る (worktree を使わない場合はここで stash が必要)
git diff main...feature/new-api --stat

# 出力例
 src/api/routes.ts     | 48 ++++++++++++++++++++++++++++++++++++++----------
 src/api/schema.ts     | 12 ++++++++++++
 tests/api.test.ts     | 31 +++++++++++++++++++++++++++++++
 3 files changed, 91 insertions(+), 10 deletions(-)
```

この出力を Claude に貼り付け、「このコードレビューをしてください。特に routes.ts の変更量が多い点に注目してください」と指示すると、ファイル全文を貼らなくても変更の大局を把握した上でレビューしてくれる。

さらに詳細が必要な場合:

```bash
# 特定ファイルだけ diff を渡す
git diff main...feature/new-api -- src/api/routes.ts
```

---

## テクニック 4: worktree + `tmux` で Claude のペインを固定配置する

画面レイアウトを固定すると「どのペインがどのブランチ」を瞬時に把握できる。  
以下は `tmux` の設定例 (`.tmux.conf` または起動スクリプト):

```bash
#!/usr/bin/env bash
# claude-worktree-session.sh
SESSION="claude-dev"

tmux new-session -d -s $SESSION -n main -x 220 -y 50

# ペイン 0: main ブランチ作業
tmux send-keys -t $SESSION "cd /projects/myapp && claude" C-m

# ペイン 1: feature ブランチ作業 (垂直分割)
tmux split-window -h -t $SESSION
tmux send-keys -t $SESSION "cd /projects/myapp-feat-api && claude" C-m

# ペイン 2: git log / テスト結果確認用 (水平分割)
tmux split-window -v -t $SESSION:0.1
tmux send-keys -t $SESSION "cd /projects/myapp && git log --oneline --all --graph" C-m

tmux attach -t $SESSION
```

起動すると以下のレイアウトになる:

```
┌──────────────────────┬──────────────────────┐
│  Claude (main)       │  Claude (feature)    │
│  /projects/myapp     │  /projects/myapp-    │
│                      │        feat-api      │
│                      ├──────────────────────┤
│                      │  git log / test      │
└──────────────────────┴──────────────────────┘
```

---

## テクニック 5: worktree を使ったコストを抑えた並列レビューフロー

Claude Code を使うとトークンコストが気になる。worktree を使うと「不要なコンテキストを持たせない」設計ができる。

### 悪い例 (コスト高):

1. main ブランチで Claude を起動
2. `git checkout feature/new-api` してから「このコードを見て」と依頼
3. Claude はチェックアウト前後のファイルを混同しがち → 訂正のやり取りでトークン消費

### 良い例 (コスト低):

1. `git worktree add ../myapp-feat-api feature/new-api`
2. **新しいターミナルで** `cd ../myapp-feat-api && claude`
3. Claude は最初からfeature ブランチのファイル構成だけを見る → 無駄なやり取りがない

さらに、**レビューが終わったら worktree を即座に削除**することで、誤って古いコンテキストを引き継ぐリスクをゼロにできる。

```bash
# レビュー完了後のクリーンアップ
git worktree remove ../myapp-feat-api
# ブランチはリモートに push 済みなので手元に残す必要なし
```

---

## 実際の並列レビューワークフロー (まとめ)

以下のフローで 1 人開発でも高速サイクルを回せる:

```
1. main で作業中に PR 案が出てきたら
   └─ git worktree add ../myapp-review-pr123 feature/pr-123

2. 新ターミナルで Claude Code を起動
   └─ cd ../myapp-review-pr123 && claude

3. Claude に git diff の結果 + CLAUDE.md の指示でレビュー依頼

4. 修正があれば worktree ディレクトリでコミット
   └─ main の作業は別ターミナルで継続 (コンテキスト汚染なし)

5. PR マージ後に worktree 削除
   └─ git worktree remove ../myapp-review-pr123

6. 次の PR が来たら 1 に戻る
```

**結果として:**  
- ブランチ切り替えによる「あれ、いまどっちのブランチだっけ」がなくなる
- Claude のコンテキストが汚染されず、レビューの訂正ラリーが激減
- tmux 固定レイアウトで「左が本番修正・右が機能開発」を視覚的に分離できる

---

## まとめ

| テクニック | 効果 |
|---|---|
| worktree × Claude Code セッション分離 | コンテキスト汚染ゼロ |
| ブランチ別 `CLAUDE.md` | 指示の繰り返しなし |
| `git diff` を起点にしたレビュー依頼 | トークン節約 |
| `tmux` 固定レイアウト | 「どっちのブランチ」混乱なし |
| レビュー後即 worktree 削除 | 古いコンテキスト引き継ぎリスクなし |

`git worktree` は Git 2.5 (2015 年) から正式搭載されているにも関わらず、まだ活用している人が少ない機能の一つ。  
Claude Code のような **ディレクトリコンテキスト依存のツール** との相性が非常に良いので、ぜひ試してみてほしい。

---

## 参考リンク

- [git-worktree 公式ドキュメント](https://git-scm.com/docs/git-worktree)
- [Claude Code 公式ドキュメント — CLAUDE.md の使い方](https://docs.anthropic.com/ja/docs/claude-code/memory)
- [tmux チートシート (GitHub: tmux/tmux)](https://github.com/tmux/tmux/wiki/Getting-Started)
- [Pro Git Book — 7.5 Git のさまざまなツール — ワークツリー](https://git-scm.com/book/ja/v2)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
