---
title: "Claude Code + OpenRouter 無料モデルで作る AI コーディング環境：コスト 0 円で始める 5 つの構成パターン"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- OpenRouter の `:free` サフィックスモデルは API キー 1 本で複数の最新 LLM を無料で呼び出せる
- Claude Code の `ANTHROPIC_BASE_URL` を OpenRouter エンドポイントに向けることで、Claude Code UI のまま別モデルを使える
- 用途別に「コード補完 / レビュー / ドキュメント生成 / テスト生成 / リファクタ」の 5 パターンを使い分けると費用ゼロで実用レベルの AI 開発支援が実現する

---

## はじめに

「AI コーディング支援を使いたいが、API 費用がかさむ」という悩みをよく耳にします。OpenAI の GPT-4o や Anthropic の Claude 3.5 Sonnet はそれぞれ強力ですが、ヘビーな自動化パイプラインで回し続けると月数万円のコストが現実になります。

この記事では **OpenRouter の `:free` モデル群** と **Claude Code の柔軟なエンドポイント設定** を組み合わせて、ほぼゼロ円から始める AI 開発環境の構成パターンを 5 つ紹介します。

> 本記事で扱う技術はすべて公開 API / OSS のみです。

---

## 前提知識

### OpenRouter とは

[OpenRouter](https://openrouter.ai) は、OpenAI 互換 API 形式で複数の LLM プロバイダにアクセスできるゲートウェイサービスです。モデル名の末尾に `:free` をつけると、レート制限はあるものの**クレジット消費なし**で呼び出せるモデルが複数あります（2026 年時点）。

代表的な `:free` モデル例：

| モデル | 得意領域 |
|---|---|
| `google/gemini-2.0-flash-exp:free` | 高速・長コンテキスト |
| `meta-llama/llama-3.3-70b-instruct:free` | コード生成・汎用 |
| `mistralai/mistral-7b-instruct:free` | 軽量・低レイテンシ |
| `qwen/qwen3-235b-a22b:free` | コーディング特化・長思考 |
| `deepseek/deepseek-r1:free` | 数学・推論 |

レート制限は 1 分あたりのリクエスト数（RPM）と 1 日あたりのリクエスト数（RPD）で設定されており、モデルにより異なります。自動化パイプラインを組む場合は [OpenRouter の Rate Limits ページ](https://openrouter.ai/docs/limits) で最新値を確認してください。

### Claude Code とは

Anthropic が公式提供する CLI ツールで、ターミナル上で Claude と対話しながらコード編集・ファイル操作・コマンド実行ができます。`npm install -g @anthropic-ai/claude-code` でインストールし、`claude` コマンドで起動します（公式ドキュメント参照）。

---

## OpenRouter を Claude Code から使う基本設定

Claude Code は `ANTHROPIC_BASE_URL` 環境変数を参照します。これを OpenRouter エンドポイントに向けるだけで、Claude Code の UI・コンテキスト管理・ツール呼び出しをそのまま使いながら、裏側のモデルを差し替えられます。

```bash
# シェル設定ファイル（~/.bashrc, ~/.zshrc 等）に追記
export ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1"
export ANTHROPIC_API_KEY="sk-or-v1-xxxxxxxxxxxxxxxx"  # OpenRouter の API キー
```

起動時にモデルを指定する場合：

```bash
claude --model "qwen/qwen3-235b-a22b:free"
```

`claude.json`（プロジェクトルートまたは `~/.claude.json`）でもモデルをデフォルト設定できます：

```json
{
  "model": "meta-llama/llama-3.3-70b-instruct:free",
  "fallbackModel": "google/gemini-2.0-flash-exp:free"
}
```

> **注意**: OpenRouter は Anthropic API と完全互換ではなく、`tool_use` / `computer_use` など一部機能はモデル依存です。コア会話機能は動作しますが、高度なツール呼び出しは実験的扱いになります。

---

## 5 つの構成パターン

### パターン 1: コード補完 — 「速さ最優先」構成

**用途**: 既存コードへの追記・補完・短いスニペット生成  
**推奨モデル**: `google/gemini-2.0-flash-exp:free` または `mistralai/mistral-7b-instruct:free`  
**理由**: Gemini Flash は応答速度が速く、軽量な補完タスクに向く。コンテキストウィンドウが大きいためファイル全体を渡しても詰まりにくい。

```bash
# セッション開始例
export ANTHROPIC_MODEL="google/gemini-2.0-flash-exp:free"
claude --model "$ANTHROPIC_MODEL"
```

プロンプト例（Claude Code 内で使用）：

```
# この関数の境界値チェックを追加して、既存のスタイルに合わせてください
```

**ポイント**: 補完用途では思考系モデル（DeepSeek R1 等）は不要。速度 < 精度のトレードオフを意識して軽量モデルを選択する。

---

### パターン 2: コードレビュー — 「品質重視」構成

**用途**: PR 前の自己レビュー・セキュリティホールの検出・パフォーマンス問題の指摘  
**推奨モデル**: `qwen/qwen3-235b-a22b:free` または `deepseek/deepseek-r1:free`  
**理由**: 大型モデルは「なぜ問題か」の説明が詳細で、単なる「ここを直せ」以上の教育的フィードバックが得られる。

```bash
# git diff をそのまま渡す使用例（シェルスクリプト）
git diff HEAD~1 | claude --model "qwen/qwen3-235b-a22b:free" \
  --message "以下の diff をレビューしてください。
バグ・セキュリティリスク・パフォーマンス問題を優先度順にリストアップし、
修正案のコードスニペットも添えてください。
<diff>
$(cat /dev/stdin)
</diff>"
```

**ポイント**: レート制限に引っかかりやすいのでループ処理では `sleep 3` 等を挟む。1 ファイルずつレビューするほうがコンテキストが締まって精度が上がる傾向がある。

---

### パターン 3: ドキュメント自動生成 — 「量産」構成

**用途**: JSDoc / rustdoc / Python docstring の自動生成・README 更新  
**推奨モデル**: `meta-llama/llama-3.3-70b-instruct:free`  
**理由**: ドキュメント生成はトークン出力量が多い。70B クラスはコスト無料でもそれなりの品質があり、定型フォーマットへの追従が安定している。

```python
# Claude Code をサブプロセスから呼ぶ Python スクリプト例（概念コード）
import subprocess, pathlib

def generate_docstring(source_file: str, model: str = "meta-llama/llama-3.3-70b-instruct:free"):
    code = pathlib.Path(source_file).read_text()
    prompt = f"""以下の Python ファイルの全関数・クラスに Google スタイルの docstring を追加してください。
既存のコードロジックは変更せず、docstring のみを追記してください。

{code}"""
    result = subprocess.run(
        ["claude", "--model", model, "--print", "--message", prompt],
        capture_output=True, text=True
    )
    return result.stdout

# 使用例
updated = generate_docstring("src/utils.py")
pathlib.Path("src/utils_documented.py").write_text(updated)
```

**ポイント**: `--print` フラグ（非対話モード）を使うとパイプライン組み込みが簡単。ただし Claude Code の非対話モードサポートはバージョンに依存するため公式 changelog を確認すること。

---

### パターン 4: テスト生成 — 「推論強め」構成

**用途**: ユニットテスト・エッジケーステストの自動生成  
**推奨モデル**: `deepseek/deepseek-r1:free`  
**理由**: R1 系モデルは Chain-of-Thought 推論が強く「このケースでは例外が発生するはず」のような境界値の洗い出しが得意。テスト生成は推論品質が出力品質に直結する。

プロンプト設計例：

```
以下の関数に対して pytest のユニットテストを書いてください。

要件:
1. 正常系 3 ケース以上
2. 異常系 (例外・エッジケース) 3 ケース以上
3. テスト関数名は test_<関数名>_<シナリオ> の命名規則
4. `pytest.mark.parametrize` を活用して重複を減らす
5. モックが必要な外部依存は `unittest.mock` を使う

対象コード:
```python
# <ここに関数を貼り付け>
```
```

**ポイント**: R1 の `<think>` タグ内の思考過程が長くなることがある。Claude Code のストリーミング表示で長い思考ブロックが続いても辛抱強く待つ。結果品質は高い。

---

### パターン 5: リファクタリング — 「フォールバック付き」構成

**用途**: レガシーコードのモダン化・型注釈追加・関数分割  
**モデル戦略**: プライマリ `qwen/qwen3-235b-a22b:free` → フォールバック `meta-llama/llama-3.3-70b-instruct:free`  
**理由**: リファクタは最も高品質な判断が求められるタスク。ただし大型モデルはレート制限に引っかかりやすいためフォールバックを用意する。

```bash
#!/bin/bash
# refactor-with-fallback.sh

FILE="$1"
PRIMARY_MODEL="qwen/qwen3-235b-a22b:free"
FALLBACK_MODEL="meta-llama/llama-3.3-70b-instruct:free"

PROMPT="以下のコードをリファクタしてください。
- 関数が 30 行を超えていれば適切に分割
- 型注釈を可能な限り追加
- マジックナンバーを定数化
- 変数名が曖昧なものはより明確な名前に変更
変更箇所には // REFACTORED: <理由> のコメントを添えること。

$(cat "$FILE")"

run_claude() {
  local model="$1"
  claude --model "$model" --message "$PROMPT" 2>&1
  return $?
}

OUTPUT=$(run_claude "$PRIMARY_MODEL")
if [ $? -ne 0 ]; then
  echo "Primary model failed, falling back to $FALLBACK_MODEL..."
  OUTPUT=$(run_claude "$FALLBACK_MODEL")
fi

echo "$OUTPUT"
```

---

## モデル選定早見表

| タスク | 推奨モデル | 優先指標 |
|---|---|---|
| コード補完 | `gemini-2.0-flash-exp:free` | 速度 |
| コードレビュー | `qwen/qwen3-235b-a22b:free` | 品質・説明力 |
| ドキュメント生成 | `llama-3.3-70b-instruct:free` | 出力量・安定性 |
| テスト生成 | `deepseek/deepseek-r1:free` | 推論・網羅性 |
| リファクタ | `qwen/qwen3-235b-a22b:free` + fallback | 判断精度 |

---

## ハマりやすいポイント 3 選

### 1. レート制限 (429 エラー)

`:free` モデルは RPM / RPD 制限が厳しく、連続リクエストで頻繁に 429 が返ります。

```python
import time, random

def call_with_retry(fn, max_retries=3, base_wait=5):
    for attempt in range(max_retries):
        try:
            return fn()
        except Exception as e:
            if "429" in str(e) and attempt < max_retries - 1:
                wait = base_wait * (2 ** attempt) + random.uniform(0, 1)
                print(f"Rate limited. Waiting {wait:.1f}s...")
                time.sleep(wait)
            else:
                raise
```

Exponential backoff + jitter が定石です。

### 2. コンテキスト長の超過

モデルによってコンテキストウィンドウが異なります。大きなファイルを渡すときは `wc -l` でおおまかに行数確認してから送ること。目安として 1,000 行 ≒ 約 15,000〜30,000 トークン。

### 3. ツール呼び出しの互換性

Claude Code の `computer_use` や高度な `tool_use` は、OpenRouter 経由では**モデル側が未対応**な場合があります。ファイル編集・コマンド実行など Claude Code 固有の操作ツールが動かない場合は、`--print` による非対話出力に切り替えてシェル側で処理するほうが安定します。

---

## コスト比較：有料プランとの使い分け

| シナリオ | 推奨構成 |
|---|---|
| 個人学習・試作 | `:free` モデル 100% |
| チーム開発の補助 (低頻度) | `:free` メイン + 月数 $ のクレジット補充 |
| CI/CD 組み込みの自動 PR レビュー | `:free` でレート制限内に収める / 超える場合は有料モデルを混在 |
| 本番プロダクションのコア機能 | 有料モデル (SLA・レート上限の保証が必要) |

`:free` モデルは**非同期・非クリティカルな用途**に絞ると費用対効果が最大化します。

---

## まとめ

- OpenRouter の `:free` モデルは、正しく使えばゼロ円で実用的な AI コーディング支援が得られる
- Claude Code の `ANTHROPIC_BASE_URL` 差し替えで UI を変えずにバックエンドを切り替えられる
- タスク特性（速度 / 推論 / 出力量）に応じてモデルを使い分けるのが鍵
- レート制限・ツール互換性の制約を理解した上で設計すると安定運用できる

まずは `gemini-2.0-flash-exp:free` 一本から始めて、タスクごとの当たりはずれを感じながら最適モデルを探っていくのが現実的なアプローチです。

---

## 参考リンク

- [OpenRouter 公式ドキュメント](https://openrouter.ai/docs)
- [OpenRouter モデル一覧・レート制限](https://openrouter.ai/models)
- [Claude Code 公式ドキュメント (Anthropic)](https://docs.anthropic.com/ja/docs/claude-code)
- [Claude Code GitHub](https://github.com/anthropics/claude-code)
- [Qwen3 技術レポート (arxiv)](https://arxiv.org/abs/2505.09388)
- [DeepSeek R1 技術レポート (arxiv)](https://arxiv.org/abs/2501.12948)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
