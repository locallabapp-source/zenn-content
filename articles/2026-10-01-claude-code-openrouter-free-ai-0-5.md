---
title: "Claude Code + OpenRouter :free モデルで作る AIコーディング環境——月0円で動かす5つの設定"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Claude Code は `ANTHROPIC_BASE_URL` を差し替えるだけで OpenRouter 経由の任意モデルを使える
- OpenRouter の `:free` サフィックスモデルを選べばトークンコスト **$0**
- モデル切替・フォールバック・コンテキスト圧縮を組み合わせれば実用レベルに達する
- 本記事では無料で動かすための **5 つの設定ポイント** を具体的に解説する

---

## はじめに

Claude Code は Anthropic の公式 CLI ベースのコーディングエージェントです。デフォルトでは `api.anthropic.com` に向いており、Claude 3.5 Sonnet / Claude 3 Opus などを呼び出します。高品質ですが、ヘビーユースすると API コストが積み上がります。

OpenRouter は 200 以上のモデルへの統一エンドポイントを提供するプロキシサービスです。多くのモデルに `:free` バリアントが存在し、レート制限はあるものの **トークン単価 $0** で利用できます。

この 2 つを組み合わせると「コーディングエージェントを毎日動かしつつコストをほぼゼロに抑える」構成が実現できます。本記事では、その具体的な設定を 5 つに分けて解説します。

---

## 前提知識

| 項目 | バージョン / 確認コマンド |
|---|---|
| Node.js | 18 以上 (`node -v`) |
| Claude Code | `npm i -g @anthropic-ai/claude-code` 最新版 |
| OpenRouter アカウント | [openrouter.ai](https://openrouter.ai) で無料登録 |

Claude Code の環境変数は以下の 2 つが核です。

```bash
ANTHROPIC_BASE_URL   # API エンドポイント
ANTHROPIC_API_KEY    # APIキー（OpenRouter の場合は OR のキーを渡す）
```

OpenRouter のエンドポイントは `https://openrouter.ai/api/v1` で、Anthropic Messages API と互換があります。

---

## 設定 1: エンドポイントを OpenRouter に向ける

最もシンプルな設定です。シェルの起動ファイル (`.zshrc` / `.bashrc`) に追記します。

```bash
export ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1"
export ANTHROPIC_API_KEY="sk-or-v1-xxxxxxxxxxxxxxxxxxxx"  # OpenRouter のキー
```

設定後:

```bash
source ~/.zshrc
claude --version  # 動作確認
```

これだけで `claude` コマンドが OpenRouter 経由でモデルを呼び出すようになります。

> **注意**: `ANTHROPIC_API_KEY` に設定するのは OpenRouter のキーです。Anthropic のキーではありません。

---

## 設定 2: :free モデルを指定する

OpenRouter でモデルを明示指定するには `ANTHROPIC_MODEL` 環境変数または `--model` フラグを使います。

### 2026年5月時点の代表的な :free モデル

| モデル名 (OpenRouter 識別子) | コンテキスト長 | 特徴 |
|---|---|---|
| `qwen/qwen3-235b-a22b:free` | 40k | Qwen3 最大クラス・思考モード対応 |
| `qwen/qwen3-30b-a3b:free` | 40k | 軽量・高速 |
| `google/gemini-2.0-flash-exp:free` | 1M | 超長コンテキスト |
| `meta-llama/llama-4-maverick:free` | 128k | Llama 4 系 |
| `microsoft/phi-4-reasoning-plus:free` | 32k | 推論特化 |
| `deepseek/deepseek-r1:free` | 64k | 推論モデル |
| `mistralai/mistral-small-3.1-24b-instruct:free` | 128k | バランス型 |

`:free` のモデルは OpenRouter の [models ページ](https://openrouter.ai/models?q=:free) でフィルタできます。

### 実際の指定方法

```bash
# 一時的に使いたいとき
claude --model "qwen/qwen3-235b-a22b:free" "このコードをリファクタして"

# 常用モデルを固定したいとき
export ANTHROPIC_MODEL="qwen/qwen3-235b-a22b:free"
```

---

## 設定 3: .claude/settings.json でプロジェクト別設定を管理する

リポジトリルートに `.claude/settings.json` を置くと、プロジェクト固有の設定を環境変数に依存せず管理できます。

```json
{
  "model": "qwen/qwen3-235b-a22b:free",
  "maxTokens": 8192,
  "temperature": 0,
  "systemPrompt": "あなたは TypeScript と Rust に精通したシニアエンジニアです。コードレビューと実装提案を行ってください。"
}
```

| キー | 説明 |
|---|---|
| `model` | 使用モデル識別子 |
| `maxTokens` | 出力トークン上限 (無料モデルは低めに抑えると Rate Limit 回避になる) |
| `temperature` | 0 にするとコーディング用途では安定する |
| `systemPrompt` | プロジェクト文脈をここで注入 |

チームリポジトリに `.claude/settings.json` をコミットすることで、チームメンバー全員が同じモデル設定で Claude Code を使えるようになります。

---

## 設定 4: コンテキスト圧縮で Rate Limit を回避する

:free モデルは RPM (requests per minute) や TPM (tokens per minute) に制限があります。長時間セッションを続けると `429 Too Many Requests` が発生します。

Claude Code には `/compact` コマンドがあり、会話履歴をサマリに圧縮してコンテキスト使用量を削減できます。

```
> /compact
```

実行すると Claude Code が現在の会話を要約し、短縮されたコンテキストで続行します。大きなコードベースを扱うセッションでは **30 分に 1 回程度 `/compact` を挟む** と安定します。

### `.claude/settings.json` で自動圧縮を設定

```json
{
  "model": "qwen/qwen3-235b-a22b:free",
  "autoCompact": true,
  "autoCompactThreshold": 0.8
}
```

`autoCompactThreshold` はコンテキストウィンドウの何割を使ったら自動圧縮するかの割合です (0〜1)。

---

## 設定 5: モデルフォールバックスクリプトを書く

:free モデルは混雑時に高レイテンシや一時的な 503 が発生します。複数モデルを順番に試すシェルスクリプトを用意しておくと実用度が上がります。

```bash
#!/bin/bash
# claude-free.sh  —  :free モデルを順番にフォールバックしながら claude を実行する

FREE_MODELS=(
  "qwen/qwen3-235b-a22b:free"
  "google/gemini-2.0-flash-exp:free"
  "meta-llama/llama-4-maverick:free"
  "mistralai/mistral-small-3.1-24b-instruct:free"
)

for MODEL in "${FREE_MODELS[@]}"; do
  echo "→ trying model: $MODEL"
  claude --model "$MODEL" "$@" && exit 0
  echo "  failed, trying next..."
done

echo "すべての :free モデルで失敗しました"
exit 1
```

使い方:

```bash
chmod +x claude-free.sh
./claude-free.sh "src/utils.ts を見てリファクタ案を出して"
```

`claude` が終了コード 0 を返したら成功とみなして次のモデルには進みません。簡易的ですが実用上は十分機能します。

---

## :free モデルの品質について

「無料なのに実用レベルか?」という疑問は当然です。実際に使った印象をまとめます。

### 向いているタスク

- **コードレビュー・改善提案**: 構造的な問題を指摘する精度は高い
- **ドキュメント生成**: README や JSDoc の自動生成は品質安定
- **単一ファイルのリファクタ**: 小〜中規模ならほぼ問題ない
- **テスト生成**: ユニットテストの雛形作成は十分実用的

### 苦手なタスク

- **マルチファイルにまたがる大規模リファクタ**: コンテキスト長と品質のトレードオフが出る
- **ドメイン固有の複雑なビジネスロジック**: 独自概念が多いとハルシネーションが増える
- **リアルタイム性が必要な対話**: :free は TPM 制限で応答が数秒〜数十秒遅れることがある

大規模タスクには有料モデルを使い、**ルーティンな作業だけを :free に流す** 使い分けが現実的です。

---

## まとめ

| 設定 | 効果 |
|---|---|
| ① `ANTHROPIC_BASE_URL` を OpenRouter に向ける | Claude Code のバックエンドをゼロコスト化 |
| ② `:free` モデルを明示指定 | $0 のモデルを確実に選択 |
| ③ `.claude/settings.json` でプロジェクト管理 | チーム共有・バージョン管理対応 |
| ④ `/compact` でコンテキスト圧縮 | Rate Limit 回避・長時間セッション安定 |
| ⑤ フォールバックスクリプト | 混雑時の可用性確保 |

Claude Code は CLI ツールとして設計されているため、エンドポイント差し替えの自由度が高く、OpenRouter との組み合わせは非常に相性が良いです。まず `:free` モデルで動かしてみて、品質に不満があるタスクだけ有料モデルに切り替える運用が費用対効果の面でおすすめです。

---

## 参考リンク

- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)
- [OpenRouter — モデル一覧](https://openrouter.ai/models)
- [OpenRouter API ドキュメント](https://openrouter.ai/docs)
- [Qwen3 技術レポート (Hugging Face)](https://huggingface.co/Qwen/Qwen3-235B-A22B)
- [Claude Code GitHub Issues — openrouter integration](https://github.com/anthropics/claude-code/issues)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
