---
title: "Claude Code + OpenRouter で月額コストを90%削減した設定術：無料モデル徹底活用ガイド2026"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- OpenRouter の `:free` モデルを優先指定することで LLM API コストをほぼゼロにできる
- Claude Code の `ANTHROPIC_API_KEY` を OpenRouter のプロキシ URL に向けるだけで設定完了
- モデル選定・タスク種別ごとの使い分け戦略を組み合わせると品質を落とさず運用できる

---

## 背景：「Claude Code 便利すぎて API 代が怖い」問題

Claude Code を使い込み始めると、すぐに気になるのが API コストだ。Claude 3.7 Sonnet を毎日ガッツリ使うと、個人開発者でも月に数千〜1万円以上の請求が飛んでくることがある。

そこで注目したいのが **OpenRouter**だ。OpenRouter は数十種類の LLM を単一エンドポイントで切り替えられるプロキシサービスで、**`:free` サフィックス付きモデルを無料で利用できる枠** を提供している（レート制限あり）。

この記事では次の 3 点を解説する。

1. OpenRouter の `:free` モデルとは何か
2. Claude Code を OpenRouter 経由で動かす設定手順
3. タスク別のモデル選定戦略

---

## OpenRouter の `:free` モデルとは

OpenRouter では一部モデルが `<model-id>:free` という識別子で提供されており、**API キーあり・クレジット消費なし** で呼び出せる。2026 年時点の代表的な無料モデルは以下の通り（公式サイト openrouter.ai で常に最新リストを確認してほしい）。

| モデル ID（例） | 提供元 | 特徴 |
|---|---|---|
| `google/gemini-2.0-flash-exp:free` | Google | 高速・長コンテキスト |
| `meta-llama/llama-3.3-70b-instruct:free` | Meta | 指示追従性が高い |
| `qwen/qwen3-235b-a22b:free` | Alibaba | 超大規模 MoE |
| `mistralai/mistral-7b-instruct:free` | Mistral | 軽量・安定 |
| `deepseek/deepseek-chat:free` | DeepSeek | コード生成が得意 |

> ⚠️ `:free` モデルはレート制限（RPM / TPM）が有料枠より厳しく、混雑時にレスポンスが遅延することがある。本番の時間制約が厳しいタスクには注意。

---

## Claude Code を OpenRouter プロキシに向ける

### 仕組み

Claude Code は内部的に Anthropic SDK を使って `api.anthropic.com` にリクエストを送る。OpenRouter は **Anthropic 互換のエンドポイント** を `https://openrouter.ai/api/v1` として提供しているため、ベース URL と API キーを差し替えるだけでモデルを切り替えられる。

### 環境変数で設定（シェル）

```bash
export ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1"
export ANTHROPIC_API_KEY="sk-or-v1-xxxxxxxxxxxx"   # OpenRouter で発行したキー
```

これだけで `claude` コマンドが OpenRouter 経由に切り替わる。

> **注意**: OpenRouter のキーは `sk-or-v1-` プレフィックスで始まる。Anthropic のキー（`sk-ant-`）とは別物。

### モデルを明示指定する場合

Claude Code の `--model` フラグや `CLAUDE_MODEL` 環境変数で指定できる（バージョンによって異なる）。

```bash
# Qwen3 無料枠を指定して起動
ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1" \
ANTHROPIC_API_KEY="sk-or-v1-xxxx" \
claude --model "qwen/qwen3-235b-a22b:free"
```

### `.env` ファイルに書いてプロジェクトごとに管理

```dotenv
# .env.openrouter（.gitignore に追加必須）
ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1
ANTHROPIC_API_KEY=sk-or-v1-xxxxxxxxxxxx
```

direnv を使うと `.envrc` にプロジェクトを切り替えるだけで自動ロードできる。

```bash
# .envrc
dotenv .env.openrouter
```

---

## タスク別モデル選定戦略

「無料だからクオリティが低い」という先入観は今や通用しない。用途に応じてモデルを使い分けることで、有料モデルとほぼ同等の成果を出せる。

### 🟢 `:free` モデルで十分なタスク

| タスク | 推奨モデル（例） | 理由 |
|---|---|---|
| コードの説明・コメント生成 | `deepseek/deepseek-chat:free` | コード理解が得意 |
| 設計ドキュメントのドラフト | `qwen/qwen3-235b-a22b:free` | 長文生成が安定 |
| テストケースの列挙 | `meta-llama/llama-3.3-70b-instruct:free` | 指示追従が確実 |
| リファクタリング提案 | `google/gemini-2.0-flash-exp:free` | 速い・長コンテキスト |
| Git コミットメッセージ生成 | `mistralai/mistral-7b-instruct:free` | 軽量で十分 |

### 🟡 有料モデルを推奨するタスク

| タスク | 理由 |
|---|---|
| 複雑なバグ調査・原因特定 | 多段推論が必要・無料枠だと途中で打ち切られることがある |
| セキュリティレビュー | 誤検知・見落としのリスクを下げたい |
| 顧客向けコードの品質保証 | レート制限による待機を避けたい |

### フォールバック設計

OpenRouter では `models` フィールドに複数モデルをリスト指定すると、**順番にフォールバック** する機能がある（API リクエストレベルの機能）。

```json
{
  "models": [
    "qwen/qwen3-235b-a22b:free",
    "meta-llama/llama-3.3-70b-instruct:free",
    "anthropic/claude-3-haiku"
  ],
  "messages": [...]
}
```

Claude Code の MCP や外部スクリプトでリクエストを組み立てる場合に応用できる。

---

## コスト試算：実際にどれくらい減るのか

あくまで概算だが、Claude 3.7 Sonnet を 1 日 50 リクエスト・平均 2,000 トークン/リクエストで使い続けると:

| プラン | 月額コスト（概算） |
|---|---|
| Anthropic 直接（Sonnet） | 約 $15〜20 |
| OpenRouter 有料モデル（Sonnet 同等） | 約 $10〜15（ルーティング手数料込み） |
| OpenRouter `:free` モデル主体 | **$0〜2**（レート超過時のみ有料モデルへフォールバック） |

**差分は月額 $13〜18 程度**。個人開発者なら年間 $150〜200 の節約になる計算だ。

---

## 注意点とハマりどころ

### 1. コンテキストウィンドウの差異

`:free` モデルは有料版より最大コンテキスト長が短い場合がある。大きなコードベースを一括で渡すような使い方では `context_length_exceeded` エラーが出ることがある。

**対策**: ファイルを分割してチャンク処理する、または長コンテキスト対応の Gemini 系を選ぶ。

### 2. レスポンスの遅延・タイムアウト

無料枠はリクエストが集中すると 10〜30 秒待たされることがある。Claude Code のデフォルトタイムアウトを伸ばす設定を合わせて調整しておくと安心だ。

```bash
# 例：CLAUDE_TIMEOUT（秒）環境変数（バージョンによる）
export CLAUDE_TIMEOUT=60
```

### 3. System Prompt の互換性

モデルによって `system` ロールの扱いが異なる。Llama 系は `[INST]` 形式に内部変換されるが、OpenRouter 側でほぼ吸収してくれる。ただし複雑なシステムプロンプトを使う場合は動作確認を推奨。

### 4. モデルの廃止・名称変更

`:free` モデルのラインナップは月単位で変わることがある。`openrouter.ai/models?q=free` で最新リストを確認してから設定すること。

---

## まとめ

| 設定ステップ | 内容 |
|---|---|
| ① OpenRouter アカウント作成 | `openrouter.ai` で無料登録 |
| ② API キー発行 | ダッシュボードから `sk-or-v1-` キーを取得 |
| ③ 環境変数を差し替え | `ANTHROPIC_BASE_URL` と `ANTHROPIC_API_KEY` を上書き |
| ④ モデル選定 | タスク種別に応じて `:free` モデルを指定 |
| ⑤ フォールバック設定 | 品質が重要なタスクは有料モデルへの自動切替を組む |

ポイントは「全タスクを無料モデルに寄せる」のではなく、**コスト感度が低いルーティンタスクに `:free` を充て、判断力が必要なタスクには予算をかける** という使い分け。これだけで月額コストは大幅に下がる。

OpenRouter の公式ドキュメント（`openrouter.ai/docs`）と、Claude Code のリリースノートを定期的にチェックしながら設定を最適化していこう。

---

## 参考リンク

- [OpenRouter 公式ドキュメント](https://openrouter.ai/docs)
- [OpenRouter モデル一覧（Free フィルタ）](https://openrouter.ai/models?q=free)
- [Claude Code 公式ドキュメント（Anthropic）](https://docs.anthropic.com/ja/docs/claude-code)
- [Qwen3 技術レポート（Alibaba）](https://qwenlm.github.io/blog/qwen3/)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
