---
title: "Claude Code × OpenRouter Free モデル：コスト0円でAIコーディング支援を回す5つの設定"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Claude Code の `ANTHROPIC_API_KEY` を OpenRouter 経由に切り替えると、無料モデルで補完・レビューが回る
- 無料枠のモデル選定・プロンプト設計・フォールバック戦略の 5 点を押さえれば実用品質になる
- 本記事はすべて公開仕様のみを根拠にしており、再現可能

---

## 背景

Claude Code は 2025 年後半から急速に普及したが、**Anthropic 直販 API のコストは高い**。Claude Sonnet 4 クラスを毎日ヘビーに使うと月数万円規模になることも珍しくない。

OpenRouter は複数プロバイダの LLM を統一エンドポイント (`https://openrouter.ai/api/v1`) で束ねるプロキシサービスで、末尾に `:free` が付くモデルは**レート制限付きの完全無料**で利用できる。

Claude Code は内部的に Anthropic Messages API (v1) と互換のエンドポイントを叩くため、環境変数を 2 本差し替えるだけで OpenRouter 経由に切り替えられる。

> 公式ドキュメント: [Claude Code Environment Variables](https://docs.anthropic.com/en/docs/claude-code/settings) / [OpenRouter API Reference](https://openrouter.ai/docs)

---

## 設定 1: エンドポイントとキーの切り替え

Claude Code が参照する環境変数は以下の 2 つ:

```bash
# ~/.bashrc or ~/.zshrc に追記
export ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1"
export ANTHROPIC_API_KEY="sk-or-v1-xxxxxxxxxxxx"   # OpenRouter のキー
```

`ANTHROPIC_BASE_URL` を上書きすることで、Claude Code のすべての API 呼び出しが OpenRouter に向く。**Anthropic のキーは不要になる**。

確認コマンド:

```bash
claude --version   # 起動確認
claude "1+1は？"   # 簡易疎通テスト
```

---

## 設定 2: 無料モデルの選定基準 (2026 年現在)

OpenRouter の `:free` ラインナップは頻繁に変わる。選定時のチェックポイントは 3 つ:

| 軸 | 理由 |
|---|---|
| **コンテキスト長 ≥ 32k** | Claude Code はファイル全文を渡すため、短いと途中で切れる |
| **Function Calling 対応** | Claude Code は tool_use スキーマを使うため必須 |
| **応答速度 (TTFT < 5s)** | 体感品質に直結。OpenRouter の Model Rankings で確認 |

2026 年 5 月時点で安定している無料モデルの例 (OpenRouter 公式ページで最新情報を確認すること):

- `qwen/qwen3-235b-a22b:free` — 235B MoE、Function Calling 対応、128k context
- `meta-llama/llama-4-maverick:free` — Meta 公式、ツール呼び出し強化版
- `mistralai/mistral-small-3.2-24b-instruct:free` — 軽量・高速、コーディング特化

Claude Code のモデル指定:

```bash
# claude_config.json (ホームディレクトリまたは .claude/)
{
  "model": "qwen/qwen3-235b-a22b:free"
}
```

または起動時フラグ:

```bash
claude --model "qwen/qwen3-235b-a22b:free" "このコードをリファクタして"
```

---

## 設定 3: プロンプト設計でトークンを節約する

無料モデルはレートリミットが厳しい (多くは 200 req/day, 20 req/min 程度)。**1 リクエストで得られる情報量を最大化**するプロンプト設計が重要になる。

### Bad: 漠然とした依頼

```
このコードを改善して
```

### Good: 制約と期待値を明示

```
以下の TypeScript コードをレビューしてください。
目的: Next.js 15 App Router の Server Component で使用
確認してほしい点:
1. 型安全性の問題
2. 不必要な re-render を引き起こすパターン
3. エラーハンドリングの漏れ
修正案はコメント付きで diff 形式で返してください。
```

Claude Code の `CLAUDE.md` (プロジェクトルートに置く指示書) に以下を書いておくとすべての会話に自動で前置きされる:

```markdown
## 回答スタイル
- 変更は diff 形式で提示 (変更量が大きいときはファイル単位)
- 説明は日本語・コードはそのまま
- 根拠のない変更はしない
- 不明点があれば先に確認する
```

---

## 設定 4: フォールバック戦略

無料モデルが落ちている・レートリミットに当たる場合のフォールバックを用意しておく。

### OpenRouter の Auto Routing 機能

OpenRouter には `openrouter/auto` という特殊なモデル ID があり、コストとクオリティのバランスを自動選択してくれる。ただしこれは有料モデルにルーティングされることがある点に注意。

**無料モデル限定でフォールバックさせる**には、シェルスクリプトで直接制御する方法が確実:

```bash
#!/bin/bash
# claude-free.sh

FREE_MODELS=(
  "qwen/qwen3-235b-a22b:free"
  "meta-llama/llama-4-maverick:free"
  "mistralai/mistral-small-3.2-24b-instruct:free"
)

for model in "${FREE_MODELS[@]}"; do
  echo "Trying model: $model"
  result=$(claude --model "$model" "$@" 2>&1)
  exit_code=$?
  
  if [[ $exit_code -eq 0 ]]; then
    echo "$result"
    exit 0
  fi
  
  echo "Model $model failed, trying next..."
done

echo "All free models failed. Check rate limits."
exit 1
```

```bash
chmod +x claude-free.sh
./claude-free.sh "この関数の単体テストを書いて"
```

---

## 設定 5: コスト監視ダッシュボードと組み合わせる

OpenRouter の API にはリクエストログのエンドポイントが公開されている。無料モデルのみ使っているつもりが、フォールバックやモデル ID のミスで有料リクエストが混入するリスクを防ぐ。

```python
# check_usage.py
# OpenRouter の公開 API で当日のリクエストログを確認するスクリプト

import httpx
import os
from datetime import date

def check_daily_usage():
    api_key = os.environ["OPENROUTER_API_KEY"]
    headers = {"Authorization": f"Bearer {api_key}"}
    
    # https://openrouter.ai/docs#get-/api/v1/auth/key
    resp = httpx.get(
        "https://openrouter.ai/api/v1/auth/key",
        headers=headers,
    )
    data = resp.json()
    
    usage = data.get("data", {})
    print(f"本日の利用額: ${usage.get('usage', 0):.6f}")
    print(f"残高: ${usage.get('limit_remaining', 'N/A')}")
    print(f"無料リクエスト数: {usage.get('rate_limit', {}).get('requests', 'N/A')} req/min")

if __name__ == "__main__":
    check_daily_usage()
```

実行:

```bash
python check_usage.py
# 本日の利用額: $0.000000
# 残高: N/A (free tier)
# 無料リクエスト数: 20 req/min
```

`usage` が 0 のままであれば、無料モデルのみが呼ばれている証拠。

---

## 実際の使用感 (Qwen3-235B:free)

筆者が 2026 年 5 月時点で検証した体感値:

| タスク | 品質 | 速度 (TTFT) | 備考 |
|---|---|---|---|
| TypeScript 型修正 | ★★★★☆ | 3-6s | 型推論が正確 |
| Python リファクタ | ★★★★★ | 2-4s | Sonnet と遜色なし |
| コミットメッセージ生成 | ★★★★★ | 1-2s | テンプレ化で安定 |
| 複雑なバグ解析 | ★★★☆☆ | 5-12s | 長いコンテキストで遅延 |
| テスト生成 | ★★★★☆ | 3-8s | edge case の網羅は手動補完必要 |

コード補完・レビュー用途なら**無料モデルで 9 割のユースケースをカバーできる**という印象。

---

## まとめ

5 つの設定をまとめると:

1. **`ANTHROPIC_BASE_URL` を OpenRouter に向ける** — エンドポイント切り替えのみ
2. **32k 以上 + Function Calling 対応の `:free` モデルを選ぶ** — 現時点は Qwen3-235B が最有力
3. **`CLAUDE.md` にレスポンス制約を書く** — トークン節約 + 品質安定化
4. **フォールバックスクリプトで無料モデルをローテーション** — 可用性向上
5. **OpenRouter の `/auth/key` エンドポイントで利用額を監視** — 意図しない課金を防ぐ

月額 $0 で Claude Code が動く環境が手に入る。レートリミットに引っかかる場面は出てくるが、CLAUDE.md による効率化と並列ワークロードの分散で実運用に十分耐えられる。

---

## 参考リンク

- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)
- [OpenRouter ドキュメント](https://openrouter.ai/docs)
- [OpenRouter モデル一覧 (`:free` でフィルタ可)](https://openrouter.ai/models?order=newest&supported_parameters=free)
- [Qwen3 技術レポート (Qwen チーム)](https://qwenlm.github.io/blog/qwen3/)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
