---
title: "Claude Code × OpenRouter :free モデルで月額0円に近づける5つの設定術"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Claude Code は `ANTHROPIC_API_KEY` 以外に **OpenRouter 経由のモデル切り替え**を公式サポートしている
- バックグラウンドタスク・要約・ファイル読み込みに `:free` モデルを割り当てることで **コストを最大80%削減**できる
- 設定は環境変数と `.claude/settings.json` の組み合わせで完結する
- 無料枠モデルの限界を知った上で「使い分け」することが肝

---

## 背景: Claude Code の料金構造を正しく理解する

Claude Code は Anthropic の API を直接叩くため、利用頻度に比例してコストが膨らみやすい。特に以下の操作は**思いのほかトークンを消費**する。

| 操作 | 消費トークン規模 | 備考 |
|---|---|---|
| 長いコードファイルの読み込み | 大 | context に全文が入る |
| 会話履歴の要約 (summarize) | 中 | 自動で走ることがある |
| ツール呼び出し結果の解釈 | 中 | JSON が膨らむ |
| コード補完・生成 | 小〜大 | 指示の粒度次第 |

Anthropic の Sonnet 系モデルはクオリティが高い反面、100万トークンあたり数ドルかかる。**毎日ヘビーに使うと月数万円規模**になるケースも珍しくない。

ここで有効なのが **OpenRouter** の `:free` モデルルーティングだ。

---

## OpenRouter `:free` モデルとは

[OpenRouter](https://openrouter.ai) は複数 LLM プロバイダーへの統一 API エンドポイントを提供するサービスで、モデル名末尾に `:free` を付けると**無料枠で試せるモデル**にルーティングされる。

2026年時点での主な `:free` モデル例:

```
google/gemini-2.0-flash-exp:free
meta-llama/llama-3.1-8b-instruct:free
qwen/qwen3-14b:free
mistralai/mistral-7b-instruct:free
nousresearch/hermes-3-llama-3.1-405b:free
```

> **注意**: `:free` モデルはレート制限が厳しく、コンテキスト長や応答速度に制限がある場合がある。本番・クリティカルなコード生成には Sonnet/Opus を使い続けることを推奨する。

---

## 設定1: OpenRouter をベースURLとして登録する

Claude Code は `ANTHROPIC_BASE_URL` 環境変数でエンドポイントを上書きできる。

```bash
# ~/.zshrc または ~/.bashrc に追記
export ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1"
export ANTHROPIC_API_KEY="sk-or-v1-xxxxxxxxxxxxxxxx"  # OpenRouter の API キー
```

これだけで Claude Code のすべてのリクエストが OpenRouter 経由になる。

> **補足**: OpenRouter の API は Anthropic API と互換性があるため、Claude Code 側のコードは無修正で動く。

---

## 設定2: `.claude/settings.json` でモデルを明示指定する

プロジェクトルートに `.claude/settings.json` を置くと、そのプロジェクト内でのデフォルトモデルを固定できる。

```json
{
  "model": "anthropic/claude-sonnet-4-5",
  "smallModel": "google/gemini-2.0-flash-exp:free",
  "largeModel": "anthropic/claude-sonnet-4-5"
}
```

ここで重要なのが **`smallModel`** キーだ。Claude Code は内部的に「軽量タスク」と「重量タスク」を分けており、ファイルの要約・検索結果の整形・単純なリネームなどの補助処理には `smallModel` を優先して使う。

`smallModel` を `:free` モデルにするだけで、**実際の開発会話コストの2〜3割をオフロード**できる。

---

## 設定3: コンテキスト圧縮を積極的に使う

Claude Code には `/compact` コマンドがあり、会話履歴をその場で圧縮・要約してコンテキストを節約できる。

```
/compact
```

長時間のセッションで気づかないうちに数万トークンのコンテキストが積み上がることがある。作業の区切り（機能実装完了・PR 作成後など）で `/compact` を打つ癖をつけると、**1セッションあたりのトークン消費を30〜50%削減**できる。

圧縮後は重要な決定事項・TODO を `CLAUDE.md` に書き出しておくと情報ロストを防げる。

```markdown
<!-- CLAUDE.md の例 -->
## 現在の作業状況
- `/api/users` の CRUD 実装完了
- 次: バリデーション追加（Zod を使う予定）
- 注意: `UserRepository` は DI コンテナ経由でしか取得しない設計
```

---

## 設定4: `--model` フラグでワンショット切り替え

会話の途中でモデルを変えたいケース（「この部分はクオリティが重要だから Sonnet で」など）は、`/model` コマンドで動的に切り替えられる。

```
/model anthropic/claude-sonnet-4-5
```

```
/model google/gemini-2.0-flash-exp:free
```

**使い分けの実例:**

| ユースケース | 推奨モデル |
|---|---|
| アーキテクチャの設計相談 | Sonnet / claude-3-7-sonnet |
| ファイルリネーム・フォーマット修正 | `:free` モデル |
| テストコードの雛形生成 | `:free` モデル |
| セキュリティ関連のコードレビュー | Sonnet 以上 |
| ドキュメント文章の推敲 | `:free` モデル（Qwen3 系は日本語が強い）|

---

## 設定5: `CLAUDE.md` でシステムプロンプトを最適化する

意外と盲点なのが**システムプロンプトのトークン数**だ。`CLAUDE.md` が肥大化するとすべてのターンに余計なトークンが消費される。

推奨する最適化:

```markdown
<!-- 肥大化しやすいアンチパターン -->
## ルール
- TypeScriptを使用すること
- ESLintを使用すること
- Prettierを使用すること
- テストはJestを使用すること
- ...（20行続く）

<!-- トークン効率の良い書き方 -->
## Stack
TypeScript + ESLint + Prettier + Jest + Prisma + Next.js App Router
```

箇条書きを1行に圧縮するだけで、100ターンのセッションでは**数千トークンの節約**になる。

また、プロジェクト固有の背景情報（ERD・API仕様など）は `CLAUDE.md` に直書きせず、**ファイルパスへの参照**に留めると良い。

```markdown
## 参考ファイル
- DB スキーマ: `prisma/schema.prisma`
- API 仕様: `docs/api.md`
```

こうすると Claude Code が必要なときだけファイルを読み込むため、常にコンテキストに展開されることを防げる。

---

## `:free` モデルの限界と注意点

`:free` モデルを使う上で知っておくべき制約をまとめる。

### レート制限
OpenRouter の `:free` モデルは**1分あたりのリクエスト数**が有料モデルより厳しく制限されている。高頻度な自動処理（CI/CDパイプライン内のコードレビューなど）には向かない。

### コンテキスト長
モデルによってコンテキストウィンドウが大幅に異なる。`mistral-7b-instruct:free` は 32k、`llama-3.1-8b:free` は 128k など、事前確認が必要。

### 整合性のある日本語出力
英語の OSS コードに対するコメント生成や英語ドキュメントの翻訳は `:free` モデルでも十分だが、**日本語の文脈を深く理解したコードレビュー**は Qwen3 系以外では品質が落ちる場合がある。

### 推論能力
複雑なバグの根本原因分析・マルチファイルリファクタリングの設計判断などは、Sonnet クラスのモデルを引き続き使うことを推奨する。

---

## 実際のコスト感（参考値）

以下は筆者環境での実測ではなく、**公開されているモデル単価からの試算**である。

| 構成 | 月間100万トークン時の概算コスト |
|---|---|
| Sonnet 100% | 約 $3〜5 |
| Sonnet 70% + `:free` 30% | 約 $2.1〜3.5 |
| Sonnet 40% + `:free` 60% | 約 $1.2〜2 |

> 出典: [OpenRouter Pricing ページ](https://openrouter.ai/models)、[Anthropic API Pricing](https://www.anthropic.com/api) (2026年時点の公開単価をもとに試算)

**個人の趣味開発・スタートアップの初期フェーズ**なら、`:free` モデルへの積極的なオフロードで実質コストを半減以下にすることは現実的だ。

---

## まとめ

Claude Code のコスト最適化は「使わない」ではなく「使い分ける」が正解だ。

1. **OpenRouter をベース URL に設定**して切り替えの土台を作る
2. **`smallModel` に `:free` モデルを割り当て**て補助処理をオフロード
3. **`/compact` を習慣化**してコンテキストを常にリーン状態に保つ
4. **作業種別でモデルをその場で切り替え**てコスト対効果を最大化
5. **`CLAUDE.md` を最小限に保つ**ことでシステムプロンプストコストを削る

LLM コストは「気づいたら高くなっていた」になりがちなサンクコストの典型だ。設定に15分かけるだけで毎月の請求が変わるなら、早めに対処したほうが良い。

---

## 参考リンク

- [Claude Code 公式ドキュメント — Model Configuration](https://docs.anthropic.com/en/docs/claude-code)
- [OpenRouter — Models List](https://openrouter.ai/models)
- [OpenRouter — Quickstart](https://openrouter.ai/docs/quickstart)
- [Anthropic API Pricing](https://www.anthropic.com/api)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
