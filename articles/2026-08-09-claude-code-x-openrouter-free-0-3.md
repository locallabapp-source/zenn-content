---
title: "Claude Code × OpenRouter Free モデル活用術 — コスト0円で自動化パイプラインを3段階に最適化する"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- OpenRouter の `:free` モデルはレート制限がある一方、**タスク分類さえ正しければ本番品質の自動化**が成立する
- 「重い思考」と「軽い出力」を 3 段階に分離することでコストをゼロに近づけられる
- Claude Code との組み合わせで **タスクルーティング → 実行 → 検証** の全フェーズを無料モデルで賄う構成を解説

---

## 背景: "全部 Opus" はもう終わった

LLM コストの問題はいまだに現場エンジニアを悩ませる。2024 年末〜2025 年にかけて「Claude 3 Opus で全タスクを処理したら月 XX 万円になった」という報告が SNS に溢れたが、その後 OpenRouter を介した **無料モデルの品質向上** により状況は大きく変わった。

本記事では以下を扱う。

1. OpenRouter `:free` モデルの現状と制限の正確な理解
2. **タスク重み分類（3 段階）** の設計パターン
3. Claude Code を "オーケストレーター" に据えた実装例
4. コスト計算と品質トレードオフの実測値（公開情報ベース）

---

## 1. OpenRouter `:free` モデルとは何か

OpenRouter は複数の LLM プロバイダーへの統一 API を提供するルーティングサービスで、URL は `https://openrouter.ai/api/v1`。ChatCompletion 互換の REST API を使うため、`openai` SDK をそのまま流用できる。

`:free` サフィックスが付いたモデル（例: `meta-llama/llama-3.1-8b-instruct:free`、`mistralai/mistral-7b-instruct:free`、`google/gemma-2-9b-it:free` など）は **料金が $0/token** だが、以下の制約がある。

| 制約項目 | 内容 |
|---|---|
| レートリミット | 20 req/min・200 req/day（2025 年時点・モデルによって異なる） |
| コンテキスト長 | モデル依存（8k〜128k） |
| レスポンス保証 | SLA なし（混雑時は遅延・エラー） |
| 利用可能性 | プロバイダー都合で一時停止あり |

重要なのは「**無料 = 低品質とは限らない**」という点だ。Llama 3.1 70B の `:free` 版は有料版と同一ウェイトであり、差異はスループットと優先度のみ。

---

## 2. タスク重み分類（3 段階）設計

### なぜ分類するのか

すべてのタスクに同じモデルを当てる「フラット戦略」は、コストと品質の両方を損なう。以下の分類モデルを導入することで、**無料モデルが処理できるタスクを最大化**しながら、クリティカルな箇所のみ有料モデルにフォールバックできる。

### Tier 0: 軽量・定型（無料モデルで十分）

```
- 変数名・関数名のサジェスト
- コメント生成・JSDoc 補完
- JSON/YAML フォーマット変換
- 短文の翻訳（技術ドキュメント英訳など）
- テストケース名の生成
- コードのリント指摘に対する単純修正
```

**推奨モデル**: `meta-llama/llama-3.1-8b-instruct:free`
**推奨条件**: プロンプト 1,000 トークン以下・出力 500 トークン以下

### Tier 1: 中量・準定型（無料モデルで概ね対応可）

```
- 関数単位のリファクタリング提案
- ユニットテスト生成（既存コードから）
- エラーメッセージの診断と修正案提示
- Markdown ドキュメントの初稿生成
- API レスポンスのスキーマ推論
```

**推奨モデル**: `meta-llama/llama-3.1-70b-instruct:free` または `mistralai/mistral-nemo:free`
**推奨条件**: プロンプト 4,000 トークン以下

### Tier 2: 重量・非定型（無料モデルで試みて、不足なら有料へ）

```
- システム設計レビュー・アーキテクチャ相談
- 複数ファイルにまたがる依存関係の整理
- セキュリティ脆弱性の多角的分析
- 技術選定の根拠作成（比較表+推奨理由）
```

**推奨モデル**: まず `google/gemini-flash-1.5:free` → 不十分なら `anthropic/claude-sonnet-4` 等に昇格

---

## 3. Claude Code でのルーティング実装

Claude Code（CLI ツール）をオーケストレーターとして使う際、モデル選択をタスク種別で自動切り替えする構成例を示す。これはすべて **公開 API と公式 SDK** のみを使った一般的な実装パターンだ。

### 3-1. ルーティングロジックの骨格（Python）

```python
import anthropic
from openai import OpenAI
from enum import Enum

class TaskTier(Enum):
    LIGHT = 0    # Tier 0
    MEDIUM = 1   # Tier 1
    HEAVY = 2    # Tier 2

OPENROUTER_BASE = "https://openrouter.ai/api/v1"

FREE_MODELS = {
    TaskTier.LIGHT:  "meta-llama/llama-3.1-8b-instruct:free",
    TaskTier.MEDIUM: "meta-llama/llama-3.1-70b-instruct:free",
    TaskTier.HEAVY:  "google/gemini-flash-1.5:free",
}

def classify_task(prompt: str) -> TaskTier:
    """
    簡易ヒューリスティックによるタスク分類。
    本番では Tier 0 モデルに分類自体を委譲してもよい。
    """
    token_estimate = len(prompt.split())
    if token_estimate < 200:
        return TaskTier.LIGHT
    elif token_estimate < 1500:
        return TaskTier.MEDIUM
    else:
        return TaskTier.HEAVY

def call_free_model(prompt: str, api_key: str) -> str:
    tier = classify_task(prompt)
    model = FREE_MODELS[tier]

    client = OpenAI(
        base_url=OPENROUTER_BASE,
        api_key=api_key,
    )
    response = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
    )
    return response.choices[0].message.content
```

### 3-2. フォールバック付きラッパー

無料モデルは `429 Too Many Requests` を返すことがある。指数バックオフ + 上位 Tier へのフォールバックを組み込むと安定性が増す。

```python
import time
from openai import RateLimitError

FALLBACK_PAID = "anthropic/claude-haiku-3"  # 万が一のフォールバック

def robust_call(prompt: str, api_key: str, max_retries: int = 3) -> str:
    tier = classify_task(prompt)
    
    for attempt in range(max_retries):
        model = FREE_MODELS.get(tier, FALLBACK_PAID)
        try:
            client = OpenAI(base_url=OPENROUTER_BASE, api_key=api_key)
            resp = client.chat.completions.create(
                model=model,
                messages=[{"role": "user", "content": prompt}],
                timeout=30,
            )
            return resp.choices[0].message.content
        except RateLimitError:
            wait = 2 ** attempt  # 1s, 2s, 4s
            time.sleep(wait)
            # レートリミットなら同 Tier をリトライ（最終回のみ Tier 昇格）
            if attempt == max_retries - 1 and tier != TaskTier.HEAVY:
                tier = TaskTier(tier.value + 1)
    
    raise RuntimeError("全リトライ失敗")
```

### 3-3. Claude Code との統合ポイント

Claude Code は `ANTHROPIC_API_KEY` を参照してデフォルトで Anthropic のモデルを使うが、**サブタスクの処理**には `subprocess` 経由で外部スクリプトを呼び出す構成も有効だ。

```bash
# Claude Code のカスタムコマンド（.claude/commands/ に配置）
# 例: /lightweight-doc コマンドで軽量タスクを無料モデルに委譲
#!/usr/bin/env bash
python3 ./scripts/free_model_runner.py \
  --prompt "$1" \
  --tier light
```

こうすることで Claude Code 自身（sonnet）は「**何をするか決める**」重い思考に専念し、「**実際の出力生成**」は無料モデルへ委譲できる。

---

## 4. 実測値と品質トレードオフ

以下は OpenRouter 公式ドキュメントおよびコミュニティベンチマーク（Hugging Face Open LLM Leaderboard 等）から得られる公開情報をもとにした比較だ。

### コード生成タスク（HumanEval ベース・公開スコア）

| モデル | Pass@1 | コスト | 用途推奨 |
|---|---|---|---|
| Llama 3.1 8B Instruct | ~33% | $0 | Tier 0（定型補完） |
| Llama 3.1 70B Instruct | ~60% | $0（:free） / $0.59/1M tok | Tier 1 |
| Gemini Flash 1.5 | ~72% | $0（:free）/ $0.075/1M tok | Tier 1〜2 |
| Claude Haiku 3 | ~75% | $0.25/1M tok | Tier 2 フォールバック |
| Claude Sonnet 4 | ~90%+ | $3/1M tok | 人間レビュー前の最終確認 |

> **注意**: HumanEval は単純な関数補完タスクであり、実業務の複雑度とは乖離がある。上記はあくまで傾向の参考値。

### 無料モデルを使うべきでないケース

以下のタスクは無料モデルに委譲しないことを推奨する。

- **セキュリティ関連のコードレビュー**（見落としリスクが高い）
- **法的文書・契約書関連の解釈**（ハルシネーションが致命的）
- **データベーススキーマの破壊的マイグレーション提案**（誤回答のコストが大きい）

---

## 5. コスト計算シミュレーション

1 日 100 件のタスクが発生する開発チームを想定する。

```
タスク内訳（仮定）:
- Tier 0 (軽量): 60件/日 → 無料モデル → $0
- Tier 1 (中量): 30件/日 → 無料モデル → $0
- Tier 2 (重量): 10件/日 → 無料モデル試行 → 失敗2件のみ有料フォールバック

有料フォールバック見積もり:
- 2件 × 平均 2,000 tokens (入力) × $0.003/1k tok = 約 $0.012/日
- 月換算: 約 $0.36/月（≒ 54円/月）
```

フラット戦略で全件を Claude Haiku 3 に投げた場合：

```
100件/日 × 2,000 tokens × $0.00025/tok = $0.05/日
月換算: $1.5/月（≒ 225円/月）
```

数字だけ見ると差が小さいが、スケールすると意味がある。1,000 req/日になれば：

- 無料戦略: 〜 $3.6/月
- フラット Claude Haiku: 〜 $22.5/月
- フラット Claude Sonnet 4: 〜 $270/月

---

## 6. 注意点とベストプラクティス

### 無料モデルのレートリミットを把握する

2025 年時点で多くの `:free` モデルは **1 分あたり 20 リクエスト**制限がある。バースト的に叩くパイプラインは意図せず上限に当たるため、**キュー + スロットリング**は必須。

```python
import asyncio

sem = asyncio.Semaphore(15)  # 20 を超えないよう 15 に設定

async def safe_call(prompt: str, api_key: str):
    async with sem:
        return await asyncio.to_thread(robust_call, prompt, api_key)
```

### プロンプトキャッシュと組み合わせる

OpenRouter は Anthropic の **Prompt Caching** をサポートしており、`cache_control` を埋め込むと繰り返し現れるシステムプロンプトが再利用される。無料モデルには適用されないが、フォールバック先の有料モデル呼び出し時にキャッシュが効くため組み合わせ効果がある。

### モデル可用性の監視

`:free` モデルは予告なく一時停止することがある。`/api/v1/models` エンドポイントで利用可能なモデル一覧を定期取得し、**代替候補を事前にリスト化**しておくことを推奨する。

```python
import requests

def get_available_free_models(api_key: str) -> list[str]:
    resp = requests.get(
        "https://openrouter.ai/api/v1/models",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    models = resp.json()["data"]
    return [
        m["id"] for m in models
        if m["id"].endswith(":free") and m.get("context_length", 0) >= 8192
    ]
```

---

## まとめ

| ポイント | 内容 |
|---|---|
| タスクを 3 段階に分類する | 軽量・中量・重量で無料モデルの適用範囲が明確になる |
| フォールバック戦略を持つ | レートリミットやモデル停止に備えた Tier 昇格ロジックを実装 |
| Claude Code はオーケストレーターに | 「何をするか」の判断にのみ高品質モデルを使い、出力生成は委譲 |
| コストはゼロに近づくが SLA は消える | 無料モデルはレイテンシ・可用性を犠牲にしている点を忘れずに |
| 品質クリティカルなタスクは除外する | セキュリティ・法務・DB 破壊的変更は有料モデルを使うべき |

3 段階ルーティングの導入は、コードを書いてから **数時間以内に効果が出る**実践的な最適化だ。特に CI/CD パイプライン内の LLM 呼び出しや、夜間バッチの定型レポート生成などに最初に適用すると費用対効果を実感しやすい。

---

## 参考リンク

- [OpenRouter Docs — Models](https://openrouter.ai/docs/models)
- [OpenRouter Docs — Rate Limits](https://openrouter.ai/docs/limits)
- [Open LLM Leaderboard (Hugging Face)](https://huggingface.co/spaces/open-llm-leaderboard/open_llm_leaderboard)
- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)
- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [OpenAI Python SDK (openai-python)](https://github.com/openai/openai-python)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
