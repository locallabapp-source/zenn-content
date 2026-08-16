---
title: "Claude Code × Git Worktree で並列タスク処理を3倍速にする実践ガイド"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- `git worktree` を使うと1リポジトリから複数ブランチを同時チェックアウトできる
- Claude Code セッションを Worktree ごとに分離することで、コンテキスト汚染なしに並列作業が実現する
- セットアップ5分・運用コスト低・特にフロントエンド+バックエンド同時改修やリファクタリング+バグ修正の並走に効く

---

## 背景：「ブランチを切り替えるたびに Claude Code が混乱する」問題

Claude Code を日常的に使い始めると、ある摩擦に気づく。

`feature/A` ブランチで「Stripe の決済フロー」を議論していた Claude Code のセッションを開いたまま、急いで `hotfix/B` ブランチに切り替えて「ログイン周りのバグ」を聞く――このとき Claude Code は直前の会話コンテキストを引き継いでいるため、**엉뚱한**（엉뚱は韓国語なので削除）頓珍漢な返答が混入しやすい。

さらに、`git stash` や `git checkout` によりワーキングツリーが書き変わるので、「さっきまで見ていたファイル」が変わり、Claude Code が参照するファイルもズレる。

**解決策は `git worktree`。** そして「Worktree = 1 Claude Code セッション」というルールを徹底することだ。

---

## git worktree とは

`git worktree` は Git 2.5（2015年）から追加されたコマンド。1つのリポジトリに対して **複数のワーキングディレクトリ** を同時に持てる。

```
my-project/              ← メインワーキングツリー (main ブランチ)
my-project--feat-a/      ← 追加ワーキングツリー (feature/A ブランチ)
my-project--hotfix-b/    ← 追加ワーキングツリー (hotfix/B ブランチ)
```

それぞれが独立したディレクトリなので、**ファイルシステムレベルでブランチが分離** される。`git stash` は不要。`node_modules` も別々に持てる（シンボリックリンクで共有も可）。

### 基本コマンド

```bash
# 新しいブランチを切りながら worktree を追加
git worktree add ../my-project--feat-a -b feature/A

# 既存ブランチの worktree を追加
git worktree add ../my-project--hotfix-b hotfix/B

# worktree 一覧確認
git worktree list

# worktree を削除
git worktree remove ../my-project--feat-a
git worktree prune  # 参照を掃除
```

---

## Claude Code × git worktree の組み合わせ方

### 原則：「1 Worktree = 1 Claude Code セッション」

| Worktree | ブランチ | Claude Code セッション | 担当タスク |
|---|---|---|---|
| `my-project/` | `main` | セッション A | レビュー・マージ管理 |
| `my-project--feat-payment/` | `feature/payment` | セッション B | Stripe 決済フロー実装 |
| `my-project--refactor-db/` | `refactor/db-schema` | セッション C | DB スキーマ整理 |
| `my-project--hotfix-login/` | `hotfix/login-bug` | セッション D | ログインバグ修正 |

各セッションは自分の Worktree ディレクトリ内だけを見るので、コンテキストが交差しない。

### セットアップ手順（5分）

**Step 1: Worktree 用のディレクトリ命名規則を決める**

```bash
# 規則: <repo-name>--<branch-slug>
git worktree add ../my-project--feat-payment -b feature/payment
git worktree add ../my-project--refactor-db  -b refactor/db-schema
```

ハイフン2つ (`--`) で区切るとタブ補完時に視認しやすい。

**Step 2: 各 Worktree を別ターミナルタブで開く**

```bash
# タブ 1
cd ../my-project--feat-payment
claude  # Claude Code 起動

# タブ 2
cd ../my-project--refactor-db
claude  # 別セッションで Claude Code 起動
```

ターミナルマルチプレクサ（tmux / Zellij）を使うと視認性がさらに上がる。

**Step 3: `.gitignore` / `.git/info/exclude` で Worktree ゴミを除外**

Worktree 追加後、メインリポジトリの `.git/worktrees/` に参照ファイルが生成されるが、これは自動管理されるので触らなくてよい。ただし `node_modules` を各 Worktree に置く場合は重複インストールが発生する。

---

## node_modules 重複問題を解消する3つのアプローチ

### アプローチ A: pnpm の `--shared-workspace-lockfile`（最もシンプル）

pnpm を使っている場合、`pnpm-workspace.yaml` を使わなくても、各 Worktree で `pnpm install` するだけで `.pnpm-store` グローバルキャッシュが効く。再インストール時間が大幅短縮。

```bash
# Worktree ごとに実行するだけ
cd ../my-project--feat-payment && pnpm install
```

### アプローチ B: シンボリックリンクで共有

ブランチ間の依存関係が同一の場合は有効。ただし依存が違う場合は干渉するので注意。

```bash
cd ../my-project--feat-payment
ln -s ../my-project/node_modules ./node_modules
```

### アプローチ C: Docker / devcontainer で完全隔離

最もクリーンだが、セットアップコストが高い。長期並走が必要な場合に採用。

---

## 実践パターン3選

### パターン 1: フロントエンド + バックエンド 同時改修

```
my-project--feat-ui/       ← Next.js 側を Claude Code で改修
my-project--feat-api/      ← FastAPI 側を別 Claude Code で改修
```

UI と API の I/F を `openapi.yaml` で合意してから並走させる。Claude Code には「openapi.yaml の `/auth/login` に合わせて実装して」と伝えれば I/F の齟齬が出にくい。

### パターン 2: リファクタリング中にバグが飛び込んできた

```
my-project--refactor/   ← リファクタリング進行中（中断しない）
my-project--hotfix/     ← 緊急バグを別 Claude Code で即修正 → main にマージ
```

`stash → checkout → 戻る` が不要になるのでリファクタリングの文脈がリセットされない。

### パターン 3: 複数バージョンの仕様比較

```
my-project--v2-design-a/   ← 設計案A を Claude Code で実装させて動作確認
my-project--v2-design-b/   ← 設計案B を別 Claude Code で並行実装
```

どちらが優れているかを同時に動かして比較できる。スパイク（捨てコード）に最適。

---

## よくあるハマりどころと対処法

### ① 同じブランチを2つの Worktree でチェックアウトできない

Git の仕様上、同一ブランチを複数 Worktree に同時チェックアウトはできない。

```bash
# エラー例
$ git worktree add ../my-project--main main
fatal: 'main' is already checked out
```

**対処**: 作業ブランチを先に切ってから Worktree を追加する。`main` を直接使わず `feature/` か `hotfix/` ブランチを必ず経由する運用が自然に整う。

### ② Worktree 削除後に `.git/worktrees/` に残骸が残る

```bash
git worktree prune
```

`prune` を定期実行する習慣を作る。`git worktree list` で `prunable` マークが付いたものを掃除できる。

### ③ Claude Code が「このファイルはどこにあるの?」と混乱する

Worktree ディレクトリ直下で `claude` を起動すれば、Claude Code はそのディレクトリをルートとして認識する。親ディレクトリから起動すると複数 Worktree を誤って横断参照することがあるため、**必ず Worktree ディレクトリに `cd` してから起動**する。

### ④ `.env` ファイルの扱い

`.env` はリポジトリに含めない前提で、各 Worktree に個別にコピーするか、シンボリックリンクで共有する。

```bash
# メイン Worktree の .env をリンク
cd ../my-project--feat-payment
ln -s ../my-project/.env .env
```

---

## 速度感の実測イメージ

厳密なベンチマーク数値は環境依存だが、以下のような体感変化が報告されている（GitHub Discussions / Zenn コメント等より）。

| 作業パターン | worktree なし | worktree あり |
|---|---|---|
| hotfix 対応の割り込みコスト | stash + checkout + 戻す = 5〜10分のコンテキスト回復 | 新タブを開くだけ = ほぼ 0 分 |
| Claude Code の的外れ回答率 | コンテキスト汚染で体感 20〜30% 増加 | ほぼゼロ（セッション分離） |
| 並列タスクのスループット | 直列 1タスクずつ | 3〜4 タスク同時進行 |

「3倍速」というのはあくまで体感・構造的な並列化の話であり、Claude Code の推論速度自体は変わらない。**人間のコンテキストスイッチコストと待ち時間が削減される**という意味での高速化だ。

---

## まとめ

- `git worktree` はブランチをファイルシステム単位で分離する Git の公式機能（追加ツール不要）
- `1 Worktree = 1 Claude Code セッション` を徹底するとコンテキスト汚染が消える
- ホットフィックス割り込み・並列フィーチャー開発・設計案の並行検討など、あらゆる「並列作業シーン」で効く
- セットアップは5分・学習コストは低い

Claude Code を使い始めると「もっと速く・もっと正確に」と欲が出てくる。`git worktree` はその欲を満たすための**地味だが確実な基盤整備**だ。まず1本だけ Worktree を追加して、並列作業の感触をつかんでみてほしい。

---

## 参考リンク

- [Git 公式ドキュメント: git-worktree](https://git-scm.com/docs/git-worktree)
- [GitHub Blog: Highlights from Git 2.5](https://github.blog/open-source/git/highlights-from-git-2-5/)
- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)
- [pnpm: Store 管理の仕組み](https://pnpm.io/cli/store)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
