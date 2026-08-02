---
title: "Claude Code × OpenRouter Free モデル活用術：コスト0円でAIコーディングを回す5つの設定"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- OpenRouterの `:free` サフィックスモデルを使えばAPI費用ゼロでAIコーディングが回せる
- Claude Codeの `ANTHROPIC_BASE_URL` をOpenRouterに向けるだけで切り替え可能
- モデル選定・コンテキスト管理・フォールバック設定の3点が品質維持のカギ
- 2026年時点で実用的なfreeモデル5本を比較検証

---

## はじめに：AIコーディングの従量課金問題

Claude CodeやContinueなどのAIコーディングアシスタントは、本格的に使い始めると**月数千〜数万円のAPI費用**がかかるようになる。特にコンテキストが長くなりがちなリファクタリングや大規模コードレビューでは、Opus系モデルは1セッションで数ドルを消費することも珍しくない。

OpenRouterが提供する `:free` モデルは、モデルプロバイダーがマーケティング・評価目的で無償提供しているAPIエンドポイントだ。レートリミットはあるものの、**個人開発・検証用途であれば十分な速度**が出る。

この記事では、Claude CodeをOpenRouterのfreeモデルに繋げて運用する具体的な方法と、実際に使える5つのモデルの特性を整理する。

---

## OpenRouter `:free` モデルとは

OpenRouterのモデルIDには `:free` というサフィックスが付くバリアントが存在する。

```
meta-llama/llama-3.3-70b-instruct:free
qwen/qwen3-235b-a22b:free
google/gemma-3-27b-it:free
microsoft/phi-4:free
mistralai/mistral-7b-instruct:free
```

**通常モデルとの違い：**

| 項目 | 通常モデル | :free モデル |
|------|-----------|-------------|
| 料金 | 従量課金 | 無料 |
| レートリミット | 緩やか | 厳しめ（20〜200 req/min程度） |
| 優先度 | 高 | 低（混雑時に遅延） |
| コンテキスト長 | フルサポート | モデル依存 |
| 機能 | フル | 一部制限あり |

重要なのは**モデルのウェイト自体は通常版と同一**であること。品質が落ちるわけではなく、単に「お試し枠のAPIキャパシティを使っている」という位置づけだ。

---

## 設定1：Claude CodeをOpenRouterに向ける

Claude Codeは `ANTHROPIC_BASE_URL` と `ANTHROPIC_API_KEY` の2つの環境変数を上書きすることで、任意のOpenAI互換エンドポイントを向けられる。

```bash
export ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1"
export ANTHROPIC_API_KEY="sk-or-v1-xxxxxxxxxxxx"  # OpenRouterのAPIキー
```

シェルの設定ファイル（`.zshrc` / `.bashrc`）に書いておくと毎回設定不要になる。

```bash
# ~/.zshrc
export ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1"
export ANTHROPIC_API_KEY="${OPENROUTER_API_KEY}"  # 別名で管理する場合
```

> **注意**：`ANTHROPIC_API_KEY` の変数名はそのままにする。Claude Codeが内部でこの変数名を参照しているためで、`OPENROUTER_API_KEY` に変えると認識されない。

---

## 設定2：モデルを明示指定する

Claude Codeのデフォルトモデルは `claude-opus-4` や `claude-sonnet-4` を参照しようとするが、OpenRouter経由では**存在しないモデルIDへのリクエストが400エラーになる**。

`.claude/settings.json` またはプロジェクトルートの `CLAUDE.md` に使用モデルを明示する：

```json
// .claude/settings.json
{
  "model": "meta-llama/llama-3.3-70b-instruct:free"
}
```

あるいはCLIオプションで都度指定：

```bash
claude --model "qwen/qwen3-235b-a22b:free" "この関数をリファクタリングして"
```

---

## 設定3：コンテキスト長を意識した使い方

freeモデルの最大の制約はコンテキスト長のキャパシティだ。通常版と比べて**ウィンドウサイズが制限される場合がある**ほか、長いプロンプトはキューイングされやすい。

**実用上のコツ：**

```bash
# 対象ファイルを絞り込んでコンテキストを小さくする
claude --model "meta-llama/llama-3.3-70b-instruct:free" \
  "src/auth/login.ts の型エラーを直して" \
  --add-file src/auth/login.ts \
  --add-file src/types/user.ts
# ❌ プロジェクト全体を渡さない
# ✅ 関連ファイル2〜3本に絞る
```

また、**長い会話セッションはこまめに `/clear` でリセット**することで、コンテキスト超過によるエラーを防げる。

---

## 設定4：タスク別モデル使い分け戦略

2026年時点でOpenRouterで使えるfreeモデルを用途別に整理する。

### Llama 3.3 70B Instruct `:free`
**`meta-llama/llama-3.3-70b-instruct:free`**

- **得意**: 汎用コーディング・コードレビュー・リファクタリング
- **コンテキスト**: 128K tokens
- **特性**: Metaのオープンウェイトモデルの中では最も安定した出力品質。英語・日本語ともに実用的なレスポンスが出る。Claude Codeとの相性が良く「まずこれを試す」的な立ち位置。
- **レートリミット**: 20 req/min（2026年5月時点）

```bash
# 主力として使う
alias cc-free='claude --model meta-llama/llama-3.3-70b-instruct:free'
```

### Qwen3 235B A22B `:free`
**`qwen/qwen3-235b-a22b:free`**

- **得意**: 複雑なロジック・アルゴリズム・数学的推論が絡むコード
- **コンテキスト**: 32K tokens（freeプランでの制限）
- **特性**: MoEアーキテクチャで235Bパラメータを持ちながらアクティブは22B。**thinking モードが使える**のが特徴で、`/no_think` タグで高速モードにも切り替えられる。コンテキストが短めなので長いファイルには向かない。
- **用途向け**: 設計レビュー・複雑なバグ調査

```bash
# 複雑な問題には思考モードを使う
alias cc-think='claude --model qwen/qwen3-235b-a22b:free'
```

### Gemma 3 27B IT `:free`
**`google/gemma-3-27b-it:free`**

- **得意**: 短いスニペット生成・テスト自動生成・ドキュメント生成
- **コンテキスト**: 128K tokens
- **特性**: Googleの軽量モデル。速度が速く**レスポンスタイムが安定している**のが利点。単純なboilerplate生成や定型的なコード補完に向いている。
- **注意**: 指示への追従性がLlama 3.3より若干弱いため、プロンプトを明確に書く必要がある。

### Phi-4 `:free`
**`microsoft/phi-4:free`**

- **得意**: 小規模なロジック検証・SQL・シェルスクリプト
- **コンテキスト**: 16K tokens
- **特性**: Microsoftの14Bモデル。コンテキストが短いため使いどころは限られるが、**SQLクエリ最適化やシェルスクリプト生成の品質**が同サイズ帯では高い。

### Mistral 7B Instruct `:free`
**`mistralai/mistral-7b-instruct:free`**

- **得意**: 簡単なコード変換・フォーマット整理
- **コンテキスト**: 32K tokens
- **特性**: 最も軽量なため**レートリミットに引っかかりにくい**。複雑なタスクには不向きだが、import文の整理・変数名リネーム・コメント追加などの機械的な変換に使える。
- **用途向け**: CI上での軽量チェック

---

## 設定5：フォールバックとエラーハンドリング

freeモデルはレートリミットに頻繁に当たる。**429エラー対策として複数モデルのローテーション**を組むのが実用的だ。

シンプルなシェルスクリプトでのフォールバック例：

```bash
#!/bin/bash
# claude-free.sh: freeモデルを順番に試すラッパー

MODELS=(
  "meta-llama/llama-3.3-70b-instruct:free"
  "qwen/qwen3-235b-a22b:free"
  "google/gemma-3-27b-it:free"
)

PROMPT="$1"

for model in "${MODELS[@]}"; do
  echo "Trying: $model"
  result=$(claude --model "$model" "$PROMPT" 2>&1)
  exit_code=$?

  if [ $exit_code -eq 0 ]; then
    echo "$result"
    exit 0
  fi

  if echo "$result" | grep -q "429\|rate limit\|Rate limit"; then
    echo "Rate limited on $model, trying next..."
    sleep 2
    continue
  fi

  # 429以外のエラーは即座に表示して終了
  echo "Error: $result"
  exit $exit_code
done

echo "All free models are rate limited. Try again later."
exit 1
```

```bash
chmod +x claude-free.sh
./claude-free.sh "この関数のテストを書いて"
```

---

## 実際の品質感：ユースケース別まとめ

実際に各モデルを使ってみると、**タスクの複雑さ×コンテキスト長の2軸**でモデルを選ぶのが最も効率的だと分かる。

```
                  コンテキスト長
                短い (<8K)    長い (>32K)
              ┌─────────────┬────────────────┐
複雑なタスク  │  Qwen3-235B │  Llama 3.3 70B │
              ├─────────────┼────────────────┤
シンプルなタスク│  Phi-4 / Mistral7B │  Gemma 3 27B  │
              └─────────────┴────────────────┘
```

### ✅ freeモデルで十分なケース
- 既存コードへのコメント追加・JSDoc生成
- 単一ファイルのリファクタリング（200行以下）
- テストケースのスケルトン生成
- SQLクエリの最適化提案
- エラーメッセージのデバッグ支援

### ⚠️ 有料モデルが欲しくなるケース
- 複数ファイルをまたぐ設計レビュー（コンテキスト不足）
- 長時間のペアプログラミングセッション（レートリミット）
- 厳密な型整合性チェック（精度が落ちる場合あり）
- 本番環境へのコードコミット前の最終確認

---

## よくあるトラブルと対処

### エラー：`model not found`

OpenRouterのモデルIDは定期的に追加・廃止される。使いたいモデルが見つからない場合は [openrouter.ai/models](https://openrouter.ai/models) でfreeフィルタをかけて最新リストを確認する。

```bash
# OpenRouter APIでfreeモデル一覧を取得
curl https://openrouter.ai/api/v1/models \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  | jq '.data[] | select(.pricing.prompt == "0") | .id'
```

### エラー：`context_length_exceeded`

コンテキストウィンドウを超えた場合は、添付ファイルを減らす・会話履歴をクリアする・よりコンテキストが長いモデルに切り替える、の3択で対処する。

### レスポンスが遅い

freeモデルのリクエストは**共有キューに入るため、夜間（UTC）の混雑時間帯は遅延する**。日本時間の早朝〜昼間（UTC 0:00〜8:00）が比較的速い。

---

## まとめ

OpenRouterの `:free` モデルを活用することで、Claude CodeベースのAIコーディングのAPIコストをゼロに抑えながら実用的な開発支援を受けられる。

**要点の整理：**

1. `ANTHROPIC_BASE_URL` をOpenRouterに向けるだけで切り替え可能
2. モデルIDを明示指定しないとエラーになる
3. コンテキストは必要最小限に絞る
4. タスク複雑度×コンテキスト長でモデルを使い分ける
5. レートリミット対策にフォールバックスクリプトを用意する

完全無料とはいえ**品質は本家Claudeよりは落ちる**ため、本番コードのコミット前は必ず人間がレビューする、というルールは維持したい。それを守れば、個人開発や社内ツールのプロトタイピングではコスト0で十分なAI支援が手に入る。

---

## 参考リンク

- [OpenRouter Documentation — Models](https://openrouter.ai/docs/models)
- [Claude Code Documentation — Environment Variables](https://docs.anthropic.com/en/docs/claude-code/settings)
- [Meta Llama 3.3 Release Notes](https://ai.meta.com/blog/llama-3-3/)
- [Qwen3 Technical Report (arXiv)](https://arxiv.org/abs/2505.09388)
- [OpenRouter Pricing Page](https://openrouter.ai/models)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
