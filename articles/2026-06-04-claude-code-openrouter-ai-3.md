---
title: "Claude Code + OpenRouter 無料モデルで始める AI 駆動開発：3 つの実践パターンと落とし穴"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- OpenRouter の `:free` モデルは**商用利用可・API キー 1 本**でコスト 0 から始められる
- Claude Code のサブエージェントに無料モデルを割り当てることで**コスト構造を劇的に改善**できる
- ただし「無料だから何でも投げる」運用は **レート制限・コンテキスト汚染・品質劣化**の三重苦を招く
- 本記事では「何に無料モデルを使い、何に使ってはいけないか」を 3 パターンで整理する

---

## 背景：「LLM を使いたいが費用が怖い」という壁

AI 駆動開発に踏み出す際、最大の心理的障壁は**コスト不安**だ。Claude Opus や GPT-4o を全作業に投入すると、コード生成・レビュー・テスト生成を繰り返すだけで月数万円を超えることがある。

OpenRouter は複数のモデルプロバイダーを統一エンドポイントで扱えるプロキシサービスで、2025 年時点で **`google/gemini-flash-1.5`・`meta-llama/llama-3.3-70b-instruct`・`mistralai/mistral-7b-instruct` など十数モデルが `:free` バリアントを提供している**。

Claude Code は Anthropic が提供する AI 駆動コーディング環境で、`ANTHROPIC_API_KEY` による Claude 本体利用に加え、**OpenRouter 互換エンドポイントを向けることで任意のモデルをバックエンドに差し替えられる**。

この 2 つを組み合わせると「思考力が必要な工程には Claude、単純作業には `:free` モデル」という**コスト最適化レイヤー**が構築できる。

---

## OpenRouter `:free` モデルの特性を把握する

### 料金と制限

`:free` モデルは文字どおり料金が 0 ドルだが、制限がある。

| 項目 | 内容 |
|---|---|
| レート制限 | モデルによって異なるが **20〜200 req/分**程度が典型 |
| コンテキスト長 | `llama-3.3-70b-instruct:free` は 131K トークンだが実効的には 32K 前後が安定 |
| SLA | ベストエフォート。有料層より遅延が大きい |
| 利用規約 | **プロバイダーのデータポリシーに依存**。送信したプロンプトがトレーニングに使われる可能性あり |

最後の点は重要で、**機密コード・個人情報・API キーを `:free` モデルに渡してはいけない**。コスト 0 の代償がデータだというモデルは一定数存在する。

### 2025 年現在の主要 `:free` モデル比較

以下は OpenRouter 公式ドキュメントおよびコミュニティベンチマークを参照した概観だ（数値は時期により変動するため、[openrouter.ai/models](https://openrouter.ai/models) で最新値を確認すること）。

| モデル ID (`:free` 付き) | 特徴 | 得意タスク |
|---|---|---|
| `google/gemini-flash-1.5:free` | 高速・1M コンテキスト | ドキュメント読み込み・要約 |
| `meta-llama/llama-3.3-70b-instruct:free` | コード品質が高い | コード補完・リファクタ提案 |
| `mistralai/mistral-7b-instruct:free` | 軽量・低遅延 | 単純フォーマット変換 |
| `deepseek/deepseek-r1:free` | 推論特化 | アルゴリズム設計・デバッグ |
| `qwen/qwen3-8b:free` | 多言語対応 | 日本語コメント生成・翻訳 |

---

## 実践パターン 3 選

### パターン 1：「コード生成は Claude・説明文生成は `:free`」の役割分担

最も安全な入門パターン。**ロジック生成**には Claude を、**ドキュメント・コメント・コミットメッセージ生成**には `:free` モデルを使う。

#### 具体的なワークフロー

```bash
# Claude Code (Claude 本体) で実装
claude "この関数の Rust 実装をしてください。エラー型は thiserror を使うこと"

# コミットメッセージは OpenRouter :free 経由で生成
curl https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/llama-3.3-70b-instruct:free",
    "messages": [
      {
        "role": "user",
        "content": "以下の git diff を見て Conventional Commits 形式のコミットメッセージを 1 行で生成してください:\n\n<diff>"
      }
    ]
  }'
```

コミットメッセージにはビジネスロジックが含まれないため、`:free` モデルでも品質面・機密面の問題が起きにくい。

#### 期待できる効果

実装系タスクに Claude を集中させ、記述系タスクを `:free` に移すことで、**同じ開発体験を維持したままコストを 30〜60% 削減**できるケースが多い（タスク比率次第）。

---

### パターン 2：「ファイル読み込みは Gemini Flash・決定は Claude」の並列調査

Gemini Flash `:free` は 1M コンテキストが強みで、**大量ドキュメントの要約・検索・抽出**に向いている。

#### ユースケース：依存ライブラリの CHANGELOG 要約

新バージョンへの移行前に複数ライブラリの CHANGELOG を読み込んで「破壊的変更だけ抽出」する作業は、人間がやると 30 分以上かかる。

```python
import httpx
import asyncio

async def summarize_changelog(changelog_text: str, model: str) -> str:
    response = await httpx.AsyncClient().post(
        "https://openrouter.ai/api/v1/chat/completions",
        headers={"Authorization": f"Bearer {OPENROUTER_API_KEY}"},
        json={
            "model": model,
            "messages": [
                {
                    "role": "user",
                    "content": (
                        "以下の CHANGELOG から「破壊的変更 (BREAKING CHANGE)」のみを"
                        "箇条書きで抽出してください。なければ「なし」と返してください。\n\n"
                        + changelog_text
                    ),
                }
            ],
        },
        timeout=60,
    )
    return response.json()["choices"][0]["message"]["content"]

# 複数ライブラリを並列処理
async def main():
    changelogs = {
        "axum": AXUM_CHANGELOG,
        "tower": TOWER_CHANGELOG,
        "tokio": TOKIO_CHANGELOG,
    }
    tasks = {
        name: summarize_changelog(text, "google/gemini-flash-1.5:free")
        for name, text in changelogs.items()
    }
    results = await asyncio.gather(*tasks.values())
    for name, result in zip(tasks.keys(), results):
        print(f"=== {name} ===\n{result}\n")

asyncio.run(main())
```

CHANGELOG 本文はオープンな情報なので機密リスクはゼロ。並列リクエストでも `:free` のレート制限内に収まるよう、`asyncio.Semaphore` で同時実行数を制御するとよい。

#### 注意点

Gemini Flash `:free` は高速だが、**コードを生成させると細部が粗くなる**ことがある。抽出・要約・分類のような「読んで整理する」タスクに限定するのが無難だ。

---

### パターン 3：「DeepSeek R1 でアルゴリズム設計の壁打ち」

DeepSeek R1 は推論（thinking）に特化したモデルで、`:free` バリアントでも**アルゴリズム設計・計算量解析・データ構造の選定**に強い。Claude Code のセッションを開く前に「設計の壁打ち相手」として使う。

#### 具体的なプロンプト例

```
以下の問題を解く関数を設計してください（コードは書かなくていい。設計のみ）:

- 入力: N 件のイベントログ (timestamp, user_id, event_type)
- 要件: 直近 7 日間に「purchase」した user_id を O(N) で抽出したい
- 制約: メモリは 1GB 以内

計算量・データ構造の選択理由・ edge case を説明してください。
```

設計段階のやり取りにはプロダクションコードが含まれないため `:free` モデルに渡しやすい。R1 の応答を見て「方針が固まったら Claude で実装」という流れにする。

#### なぜ Claude に全部やらせないのか？

Claude Sonnet はコード生成・設計・実装すべてで高水準だが、**設計フェーズは反復回数が多い**。「○○だとどうなる？」「N が 10M になったら？」という試行錯誤を Claude で繰り返すとトークン消費が激しい。R1 で方針を固めてから Claude に「実装してください」と 1 発投げる方がコスパがよい。

---

## 落とし穴：やってはいけない 3 つの運用

### ❌ 落とし穴 1：`:free` モデルに秘密情報を渡す

繰り返しになるが、**`:free` モデルはデータポリシーが有料層より緩いことが多い**。

絶対に渡してはいけないもの：
- API キー・シークレット
- データベース接続文字列
- 顧客の個人情報
- 自社の未公開ビジネスロジック

「ファイル全体を貼り付けてレビューしてほしい」という使い方は危険。`.env` が含まれていないか、コードに認証情報がハードコードされていないかを必ず確認する。

### ❌ 落とし穴 2：レート制限を無視した全自動化

`:free` モデルは **1 分あたりのリクエスト数に厳しい上限**がある。CI/CD パイプラインに組み込んで並列 PR ごとに呼び出すと、すぐに `429 Too Many Requests` を返すようになる。

対策：

```python
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=4, max=60),
)
async def call_free_model(prompt: str) -> str:
    # OpenRouter API 呼び出し
    ...

# 同時実行数を制限
semaphore = asyncio.Semaphore(3)  # 同時 3 req まで

async def safe_call(prompt: str) -> str:
    async with semaphore:
        return await call_free_model(prompt)
```

`tenacity` による指数バックオフと `asyncio.Semaphore` による同時実行制限の組み合わせが定番だ。

### ❌ 落とし穴 3：無料モデルの出力を信頼しすぎる

`:free` モデルは有料モデルより**ハルシネーション率が高く、コードの微細なバグを見落としやすい**。

「コミットメッセージ生成」「要約」のような**人間がすぐ確認できる出力**には使えるが、「ユニットテストの生成」「セキュリティチェック」のような**高精度が求められる出力**を無批判に採用してはいけない。

ガイドライン：
- `:free` モデルの出力はかならず「人間またはより強いモデルがレビューする」
- セキュリティに関わるコード (認証・暗号化・権限チェック) は Claude Sonnet 以上を使う
- `:free` モデルが生成したテストコードはそのまま CI に入れず、実装者が内容を読んでから追加する

---

## コスト試算：実際どれくらい変わるか

以下は仮想的な 1 日の開発タスク分布と、モデル割り当てを変えた場合のコスト比較だ（トークン単価は 2025 年 5 月時点の OpenRouter 掲載値をベースにした概算）。

| タスク | 1 日の想定トークン | All Claude Sonnet | 分担後 |
|---|---|---|---|
| 機能実装コード生成 | 200K | $0.60 | $0.60 (Claude) |
| ドキュメント・コメント生成 | 150K | $0.45 | $0 (`:free`) |
| CHANGELOG 要約 | 300K | $0.90 | $0 (`:free`) |
| 設計壁打ち (30 往復) | 100K | $0.30 | $0 (`:free`) |
| **合計** | 750K | **$2.25/日** | **$0.60/日** |

もちろん実際のタスク比率やモデルのバージョンアップで変わるが、**「コア実装以外を `:free` に移す」だけで 1 日コストが 1/3 以下になる**というオーダー感は参考になる。

---

## OpenRouter を Claude Code から使う設定

Claude Code は `ANTHROPIC_API_KEY` を使うが、OpenRouter のエンドポイントを直接叩くスクリプトを横に置くことで補完できる。以下は `.zshrc` や `.bashrc` への追加例（`:free` モデル呼び出し用のシェル関数）：

```bash
# OpenRouter :free モデルへの簡易呼び出し関数
or_free() {
  local model="${1:-meta-llama/llama-3.3-70b-instruct:free}"
  local prompt="$2"
  curl -s https://openrouter.ai/api/v1/chat/completions \
    -H "Authorization: Bearer $OPENROUTER_API_KEY" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg m "$model" --arg p "$prompt" \
      '{model: $m, messages: [{role: "user", content: $p}]}')" \
    | jq -r '.choices[0].message.content'
}

# 使い方
# or_free "google/gemini-flash-1.5:free" "$(git diff HEAD~1 | head -200) をもとにコミットメッセージを作って"
```

`jq` が入っていない環境は `brew install jq` / `apt install jq` で準備する。

---

## まとめ

| | 推奨モデル層 |
|---|---|
| 機能実装・設計確定・セキュリティ | Claude Sonnet 以上 |
| ドキュメント・コミットメッセージ | `:free` (llama-3.3-70b 等) |
| 大量ドキュメント読み込み・要約 | `:free` (Gemini Flash) |
| アルゴリズム設計の壁打ち | `:free` (DeepSeek R1) |
| 秘密情報を含む処理 | `:free` **絶対 NG** |

「LLM をふんだんに使いたいがコストが怖い」という状況の突破口として、**タスクを性質で分類して `:free` モデルを戦略的に活用する**アプローチは即日導入できる現実解だ。まずはコミットメッセージ生成から試してみることをお勧めする。

---

## 参考リンク

- [OpenRouter — モデル一覧](https://openrouter.ai/models)
- [OpenRouter API ドキュメント](https://openrouter.ai/docs)
- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)
- [DeepSeek R1 論文 (arXiv)](https://arxiv.org/abs/2501.12948)
- [tenacity — Python リトライライブラリ](https://tenacity.readthedocs.io/)
- [Conventional Commits 仕様](https://www.conventionalcommits.org/)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
