---
title: "Claude Code × OpenRouter :free モデルで月0円AI開発環境を構築する5つの設定"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- OpenRouter の `:free` サフィックスモデルを使えば API 費用ゼロで LLM を呼び出せる
- Claude Code のカスタムモデル設定で OpenRouter エンドポイントを向けることができる
- 適切なモデル選定と fallback 設計で、実用レベルの開発支援環境が無料で成立する
- 本記事では環境構築から実践的なモデル選定まで、一般公開情報のみで解説する

---

## はじめに

LLM を日常的な開発支援に使いたいが、API 費用が気になる——そんな悩みを持つ個人開発者やスモールチームは多いはずです。

OpenRouter は複数の LLM プロバイダーへの統一インターフェースを提供するサービスですが、その中に **`:free` サフィックスを持つ無料枠モデル** が存在します。これを Claude Code (Anthropic 製のターミナル AI アシスタント) と組み合わせることで、月額コストゼロの AI 開発支援環境を構築できます。

本記事では、すべて **公開ドキュメントと一般的な設定手順** のみを使って、この環境を 5 つのステップで整える方法を解説します。

---

## 前提知識

| ツール | 概要 | 無料枠 |
|---|---|---|
| **OpenRouter** | LLM 統合 API プロキシ | `:free` モデルは $0/request |
| **Claude Code** | Anthropic 製ターミナル AI | 利用は Anthropic API or カスタムエンドポイント |
| **OpenAI 互換 API** | 多くの OSS ツールが対応する API 仕様 | — |

OpenRouter は OpenAI 互換のエンドポイント (`https://openrouter.ai/api/v1`) を提供しているため、`OPENAI_API_KEY` を差し替えるだけで多くのツールが流用できます。

---

## ① OpenRouter アカウントと API キーの取得

[openrouter.ai](https://openrouter.ai) でサインアップし、ダッシュボードから API キーを発行します。

**:free モデルとは何か**

OpenRouter のモデル一覧で `model-id:free` という形式のモデルが無料枠対象です。たとえば:

```
google/gemma-3-27b-it:free
meta-llama/llama-4-scout:free
mistralai/mistral-7b-instruct:free
qwen/qwen3-8b:free
```

`:free` モデルはレート制限があるものの、費用は一切発生しません。公式ドキュメントによると、無料枠は「低優先度キュー」に回される場合があり、負荷が高い時間帯はレスポンスが遅延することがあります。

> **注意**: `:free` モデルのレート制限は OpenRouter の利用規約で定められており、商用利用の可否はモデルごとのライセンスに依存します。コード生成などの用途では各モデルのライセンス (Apache 2.0 / CC-BY-NC / Llama Community License 等) を必ず確認してください。

---

## ② Claude Code のエンドポイント切り替え

Claude Code は環境変数でベース URL とモデルをオーバーライドできます。公式ドキュメントに記載されている設定項目です。

```bash
# ~/.bashrc または ~/.zshrc に追記
export ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1"
export ANTHROPIC_API_KEY="sk-or-v1-xxxxxxxxxxxx"  # OpenRouter のキー
```

> **セキュリティ注意**: API キーはシェルの設定ファイルにベタ書きせず、`direnv` や OS のキーチェーンを使って管理することを推奨します。シェル設定ファイルを git 管理している場合は `.gitignore` に追加してください。

Claude Code 起動後、利用するモデルを `:free` モデルに切り替えます:

```bash
# Claude Code セッション内で
/model google/gemma-3-27b-it:free
```

あるいは起動引数で指定:

```bash
claude --model google/gemma-3-27b-it:free
```

---

## ③ 2026年版 :free モデル実用選定ガイド

無料モデルを「何でも使えばいい」とすると品質にばらつきが出ます。用途別に使い分けるのが実践的です。

### コード補完・レビュー向け

| モデル | 特徴 |
|---|---|
| `qwen/qwen3-8b:free` | Qwen3 シリーズ。コード品質が高く多言語対応。ライセンス: Apache 2.0 |
| `meta-llama/llama-4-scout:free` | Meta の最新 Llama 4 系。長いコンテキスト対応。ライセンス: Llama 4 Community License |
| `google/gemma-3-27b-it:free` | Google の Gemma 3。instruction following が安定。ライセンス: Gemma Terms of Use |

### 質問応答・ドキュメント要約向け

| モデル | 特徴 |
|---|---|
| `mistralai/mistral-7b-instruct:free` | 軽量で安定。ライセンス: Apache 2.0 |
| `microsoft/phi-3-mini-128k-instruct:free` | 超長文対応 (128K context)。ライセンス: MIT |

### 選定の判断軸

```
1. ライセンス確認 (商用 OK か)
2. コンテキスト長 (8K / 32K / 128K)
3. 言語対応 (日本語の精度差は大きい)
4. レート制限の寛容さ (モデルにより異なる)
```

---

## ④ `.claude/config.json` で fallback 設計を組む

`:free` モデルはレート制限に引っかかることがあります。Claude Code の設定ファイルを使って、**有料モデルへの自動フォールバックなし・複数モデルを手動切替** するワークフローを作ると安定します。

Claude Code は `CLAUDE.md` というプロジェクトルートのファイルを読み込み、AI の振る舞いを変えることができます。

**プロジェクトルートの `CLAUDE.md` 例:**

```markdown
# Project: my-app

## モデル設定ポリシー
- 通常作業: qwen3-8b:free を使用
- コードレビュー: gemma-3-27b-it:free を使用
- レート制限時: mistral-7b-instruct:free に切替

## 禁止事項
- API キーをコードに直書きしない
- テスト未作成のコードを main にマージしない
```

この `CLAUDE.md` はプロジェクトの git 管理に含められ、チーム共有が可能です (キーや秘密情報を書かないことが前提)。

---

## ⑤ OpenRouter の使用量モニタリングと上限設定

無料モデルでも OpenRouter のクレジット消費はゼロですが、**有料モデルへの誤爆を防ぐ** ために Usage Limits を設定しておくと安全です。

OpenRouter ダッシュボードの `Settings > Limits` から:

```
Monthly Limit: $0.00 (ゼロに設定 → 有料モデルへのリクエストをブロック)
```

これにより、設定ミスで課金モデルを呼び出すリスクを排除できます。

また、使用状況は OpenRouter ダッシュボードのアクティビティログで確認できます。モデルごとのリクエスト数・トークン消費をリアルタイムで把握できるので、どの用途でどのモデルが実際に動いているかを把握しやすいです。

---

## 実際の開発フローに組み込む

以上の設定を整えると、以下のようなワークフローが成立します:

```
1. ターミナルで claude コマンド起動
   → OpenRouter :free モデルに接続済み

2. コードレビューを依頼
   → gemma-3-27b-it:free がコードを解析・提案

3. テスト生成を依頼
   → qwen3-8b:free が pytest / vitest のテストを生成

4. レート制限で 429 が返ってきたら
   → /model mistral-7b-instruct:free でモデル切替
   → 数分待ってから戻す

5. 月末に OpenRouter ダッシュボード確認
   → 請求: $0.00 ✅
```

---

## 注意点と限界

### :free モデルで難しいこと

- **長時間の連続セッション**: レート制限でセッションが途切れる
- **最高精度が必要な作業**: Claude 3.7 Sonnet 等の有料モデルには及ばない
- **リアルタイム応答が必要な場面**: 低優先度キューで遅延が出ることがある

### 有料モデルとのハイブリッド戦略

現実的な使い分けは:

| 作業 | 推奨 |
|---|---|
| 日常的なコード補完・質問 | :free モデル |
| 重要な設計レビュー・リファクタ | 有料モデル ($0.01-$0.03 程度) |
| ドキュメント生成・テスト量産 | :free モデル |
| セキュリティレビュー・アーキテクチャ設計 | 有料モデル |

---

## まとめ

5 つのステップを整理します:

1. **OpenRouter アカウント作成** → `:free` モデル一覧の確認
2. **Claude Code のエンドポイント変更** → `ANTHROPIC_BASE_URL` を OpenRouter に向ける
3. **用途別モデル選定** → コード補完は Qwen3/Llama4、要約は Mistral/Phi-3
4. **CLAUDE.md でモデルポリシーを明文化** → チーム共有可能な設定
5. **OpenRouter の月額上限を $0 設定** → 有料モデルへの誤爆防止

無料モデルの品質は 2024〜2026 年にかけて大幅に向上しており、個人開発や PoC フェーズであれば **月 0 円でも十分実用的な AI アシスト開発環境** が成立します。

まずは `:free` モデルで慣れ、本当に有料モデルが必要な作業を見極めてから課金するのがコスト効率の良いアプローチです。

---

## 参考リンク

- [OpenRouter 公式ドキュメント](https://openrouter.ai/docs)
- [OpenRouter モデル一覧](https://openrouter.ai/models)
- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)
- [Qwen3 リリースノート (Hugging Face)](https://huggingface.co/Qwen/Qwen3-8B)
- [Llama 4 Community License](https://llama4.meta.com/llama4-community-license-agreement/)
- [Gemma Terms of Use](https://ai.google.dev/gemma/terms)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
