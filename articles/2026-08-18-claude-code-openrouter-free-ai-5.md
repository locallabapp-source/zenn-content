---
title: "Claude Code + OpenRouter Free モデルで構築する「ゼロコスト AI レビューパイプライン」— 5 つの設計パターン"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- OpenRouter の `:free` モデル（Qwen3・Gemma3・DeepSeek 等）は商用レベルのコード理解力を持つ
- Claude Code の `custom slash command` + `hooks` 機構を組み合わせると、コストゼロで AI レビューを CI に埋め込める
- 本記事では「差分レビュー」「セキュリティスキャン」「ドキュメント自動生成」「テストヒント生成」「リファクタ提案」の 5 パターンを紹介する

---

## 背景

AI コードレビューは OpenAI・Anthropic の商用 API を使うと、中規模チームでも月 $200〜$500 のコストが発生しやすい。  
一方、2025 年後半から 2026 年にかけて、**OpenRouter の `:free` ティア**（`/models?supported_parameters=free` でリストアップ可能）に本番品質のモデルが急増している。

| モデル識別子 (2026-Q2 時点の例) | 特徴 |
|---|---|
| `qwen/qwen3-235b-a22b:free` | MoE・235B・コード理解力高 |
| `google/gemma-3-27b-it:free` | Google 製・instruction tuning 強 |
| `deepseek/deepseek-r1:free` | 推論系タスクに強い |
| `meta-llama/llama-3.3-70b-instruct:free` | Meta の主力 70B |
| `mistralai/mistral-7b-instruct:free` | 軽量・高速 |

> **注意**: `:free` モデルは利用制限（RPM/TPD）がある。大規模チームでの本番 CI への適用は有料プランへの移行を検討すること。

Claude Code は Anthropic 公式の CLI エージェントだが、その内部で呼び出す LLM を OpenRouter 経由の `:free` モデルに差し替えることが、設定ファイルと環境変数の組み合わせで実現できる。  
本記事では **Claude Code の拡張ポイント**と **OpenRouter Free モデル**を組み合わせた 5 つの実用パターンを解説する。

---

## 前提知識

### Claude Code の拡張ポイント (公式ドキュメントより)

Claude Code には以下の拡張機構がある。

1. **Custom slash commands** (`/.claude/commands/*.md`): `$ARGUMENTS` を使って動的プロンプトを定義できる
2. **Hooks** (`~/.claude/settings.json` の `hooks` セクション): `PreToolUse` / `PostToolUse` / `Stop` 等のライフサイクルで外部スクリプトを呼べる
3. **MCP (Model Context Protocol) サーバ**: stdio/SSE で外部ツールをエージェントに公開できる
4. **CLAUDE.md**: リポジトリのコンテキスト・ルールを Agent に注入できる

### OpenRouter の API 互換性

OpenRouter は OpenAI SDK 互換の REST API を提供している。  
`base_url` を `https://openrouter.ai/api/v1` に差し替え、`api_key` に OpenRouter キーをセットするだけで既存の OpenAI クライアントが動く。

```python
# openai パッケージを使いながら OpenRouter 経由でアクセスする例 (公式ドキュメント掲載パターン)
from openai import OpenAI

client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="<YOUR_OPENROUTER_KEY>",  # 実際のキーは環境変数から取得すること
)

response = client.chat.completions.create(
    model="qwen/qwen3-235b-a22b:free",
    messages=[{"role": "user", "content": "Hello"}],
)
```

---

## パターン 1: git diff を渡してインライン差分レビューを生成する

最もシンプルなユースケース。`git diff HEAD` の出力をプロンプトに詰めて、:free モデルにレビューさせる。

### 実装

```bash
#!/usr/bin/env bash
# scripts/ai-review.sh
# 依存: openai Python パッケージ (pip install openai)

set -euo pipefail

DIFF=$(git diff HEAD --unified=5)

if [ -z "$DIFF" ]; then
  echo "No diff found. Exiting."
  exit 0
fi

python3 - <<'PYEOF'
import os, sys
from openai import OpenAI

diff = sys.stdin.read() if not sys.stdin.isatty() else ""

# diff が空なら環境変数経由で受け取る（bashヒアドキュメントの都合上）
import subprocess
diff = subprocess.check_output(["git", "diff", "HEAD", "--unified=5"]).decode()

client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key=os.environ["OPENROUTER_API_KEY"],
)

SYSTEM = """
あなたはシニアエンジニアです。
以下の git diff を確認し、次の観点でレビューコメントを出してください。

1. バグ・ロジックエラーの可能性
2. セキュリティ上の懸念点
3. パフォーマンス改善余地
4. 命名・可読性の改善提案

出力形式:
- ファイルごとにセクション分け
- 問題なければ「✅ 問題なし」と記載
- 深刻度を [HIGH / MEDIUM / LOW] でラベル付け
"""

resp = client.chat.completions.create(
    model="qwen/qwen3-235b-a22b:free",
    messages=[
        {"role": "system", "content": SYSTEM},
        {"role": "user", "content": f"```diff\n{diff}\n```"},
    ],
    max_tokens=2048,
)

print(resp.choices[0].message.content)
PYEOF
```

### GitHub Actions への組み込み例

```yaml
# .github/workflows/ai-review.yml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install deps
        run: pip install openai

      - name: Run AI Review
        env:
          OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
        run: |
          git diff origin/main...HEAD --unified=5 > /tmp/pr.diff
          python3 scripts/ai_review.py /tmp/pr.diff | tee /tmp/review.md

      - name: Post review as PR comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const body = fs.readFileSync('/tmp/review.md', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## 🤖 AI Review (OpenRouter Free)\n\n${body}`,
            });
```

> **コスト**: `:free` モデル使用のため API コストは $0。GitHub Actions の実行時間のみ。

---

## パターン 2: Claude Code Custom Command でその場レビューを呼び出す

Claude Code の `custom slash command` 機能を使い、エディタ作業中に `/review` と入力するだけで差分レビューを走らせる。

### 設定ファイル

```markdown
<!-- .claude/commands/review.md -->
以下のコンテキストで現在の変更点を確認してください。

$ARGUMENTS が指定されている場合はそのファイルのみ、指定なしの場合は `git diff HEAD` 全体を対象にします。

レビュー観点:
1. ロジックの正確性
2. エッジケースの処理漏れ
3. 型安全性 (TypeScript / Rust 等の場合)
4. テストの追加が必要な箇所

出力は日本語で、箇条書きでまとめてください。
深刻な問題があれば冒頭に 🚨 マークを付けてください。
```

`/review src/auth.ts` と入力するだけで該当ファイルのレビューが始まる。

### CLAUDE.md でプロジェクト固有のルールを注入する

```markdown
<!-- CLAUDE.md (リポジトリルートに置く) -->
# Project Review Rules

## セキュリティ必須チェック項目
- SQL クエリはすべてプリペアドステートメントを使用すること
- ユーザー入力の直接 HTML 展開は禁止
- JWT 検証は `exp` クレームを必ず確認すること

## コーディング規約
- 関数の最大行数: 50 行
- ネストの最大深度: 3
- マジックナンバーは const に切り出す

## 使用技術スタック
- Runtime: Node.js 22 / Bun 1.x
- Framework: Next.js 15 App Router
- Database: PostgreSQL (Drizzle ORM)
```

Claude Code はこのファイルを自動的にコンテキストに注入するため、プロジェクト固有のレビュールールを毎回プロンプトに含める必要がなくなる。

---

## パターン 3: セキュリティスキャン特化プロンプト

差分レビューとは別に、**セキュリティ観点に特化したスキャン**を定期実行する。

```python
# scripts/security_scan.py
"""
変更ファイルに対してセキュリティスキャンを実行するスクリプト。
OWASP Top 10 を中心にチェック。
"""

import os
import subprocess
from openai import OpenAI

SECURITY_SYSTEM_PROMPT = """
あなたはセキュリティエンジニアです。
提供されたコードを OWASP Top 10 (2021) の観点でスキャンしてください。

チェック項目:
- A01: Broken Access Control (アクセス制御の欠陥)
- A02: Cryptographic Failures (暗号化の失敗)
- A03: Injection (インジェクション: SQL / Command / XSS)
- A04: Insecure Design (セキュアでない設計)
- A05: Security Misconfiguration (セキュリティの設定ミス)
- A06: Vulnerable Components (脆弱なコンポーネントの使用)
- A07: Authentication Failures (認証の失敗)
- A08: Software Integrity Failures (ソフトウェアとデータの整合性の欠陥)
- A09: Logging Failures (ログ記録と監視の失敗)
- A10: SSRF (サーバーサイドリクエストフォージェリ)

出力フォーマット:
## スキャン結果サマリ
- 検出件数: X 件 (HIGH: N / MEDIUM: N / LOW: N)

## 詳細
### [HIGH] <問題タイトル>
- 場所: <ファイル名>:<行番号付近>
- 問題: <説明>
- 推奨修正: <具体的な修正方法>
"""


def scan_file(filepath: str) -> str:
    with open(filepath) as f:
        code = f.read()

    client = OpenAI(
        base_url="https://openrouter.ai/api/v1",
        api_key=os.environ["OPENROUTER_API_KEY"],
    )

    resp = client.chat.completions.create(
        model="deepseek/deepseek-r1:free",  # 推論系タスクは DeepSeek R1 が得意
        messages=[
            {"role": "system", "content": SECURITY_SYSTEM_PROMPT},
            {
                "role": "user",
                "content": f"ファイル: {filepath}\n\n```\n{code}\n```",
            },
        ],
        max_tokens=3000,
    )

    return resp.choices[0].message.content


if __name__ == "__main__":
    # 変更されたファイルのリストを git から取得
    changed = subprocess.check_output(
        ["git", "diff", "--name-only", "HEAD"],
    ).decode().splitlines()

    # ソースファイルのみ対象（設定ファイル・ロックファイルは除外）
    targets = [
        f for f in changed
        if f.endswith((".ts", ".tsx", ".js", ".py", ".go", ".rs"))
    ]

    for filepath in targets:
        print(f"\n{'='*60}")
        print(f"Scanning: {filepath}")
        print('='*60)
        result = scan_file(filepath)
        print(result)
```

---

## パターン 4: JSDoc / TSDoc 自動生成

未ドキュメントの関数・クラスを検出し、ドキュメントコメントを自動生成するパターン。

```typescript
// scripts/gen-docs.ts
// 実行: bun run scripts/gen-docs.ts src/utils/

import { OpenAI } from "openai";
import { readFileSync, writeFileSync } from "fs";
import { glob } from "glob";

const client = new OpenAI({
  baseURL: "https://openrouter.ai/api/v1",
  apiKey: process.env.OPENROUTER_API_KEY!,
});

const SYSTEM = `
あなたは TypeScript の技術文書エキスパートです。
提供された TypeScript コードを解析し、JSDoc / TSDoc コメントが未記載の
関数・クラス・インターフェースに対してドキュメントコメントを追記してください。

ルール:
- 既存のコメントは変更しない
- @param / @returns / @throws / @example を適切に付与する
- 型情報から推測可能な場合は @param の型は省略してもよい
- @example は実際に動く最小コードを書く
- 出力はコメントを追記したファイル全体を返す
`;

async function generateDocs(filepath: string): Promise<void> {
  const source = readFileSync(filepath, "utf-8");

  const resp = await client.chat.completions.create({
    model: "google/gemma-3-27b-it:free",
    messages: [
      { role: "system", content: SYSTEM },
      {
        role: "user",
        content: `ファイル: ${filepath}\n\n\`\`\`typescript\n${source}\n\`\`\``,
      },
    ],
    max_tokens: 4096,
  });

  const generated = resp.choices[0].message.content ?? "";

  // コードブロックを抽出
  const match = generated.match(/```typescript\n([\s\S]+?)\n```/);
  if (match) {
    writeFileSync(filepath, match[1]);
    console.log(`✅ Docs generated: ${filepath}`);
  } else {
    console.warn(`⚠️  No code block found in response for ${filepath}`);
  }
}

// 指定ディレクトリ以下の .ts ファイルを処理
const files = await glob(process.argv[2] ?? "src/**/*.ts", {
  ignore: ["**/*.test.ts", "**/*.d.ts", "node_modules/**"],
});

for (const file of files) {
  await generateDocs(file);
  // Rate limit 対策: 1 秒待機
  await new Promise((r) => setTimeout(r, 1000));
}
```

---

## パターン 5: テストヒント生成 + リファクタ提案の並列実行

複数の :free モデルを**並列呼び出し**して異なる観点のフィードバックを集める。異なるモデルにそれぞれ「テストヒント」「リファクタ提案」を割り当て、結果をマージする。

```python
# scripts/parallel_analysis.py
"""
テストヒント生成とリファクタ提案を異なるモデルで並列実行し、
レポートを生成する。
"""

import asyncio
import os
from openai import AsyncOpenAI

client = AsyncOpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key=os.environ["OPENROUTER_API_KEY"],
)


async def generate_test_hints(code: str, filename: str) -> str:
    """テストケースのヒントを生成する (Llama 3.3 70B 使用)"""
    resp = await client.chat.completions.create(
        model="meta-llama/llama-3.3-70b-instruct:free",
        messages=[
            {
                "role": "system",
                "content": (
                    "あなたはテスト設計の専門家です。"
                    "提供されたコードに対して、Vitest / Jest 形式のテストケースヒントを生成してください。"
                    "正常系・異常系・境界値テストを網羅すること。"
                    "実際のテストコードではなく、テストすべき観点のリストを出力してください。"
                ),
            },
            {"role": "user", "content": f"```typescript\n{code}\n```"},
        ],
        max_tokens=1024,
    )
    return resp.choices[0].message.content or ""


async def generate_refactor_suggestions(code: str, filename: str) -> str:
    """リファクタリング提案を生成する (Qwen3 使用)"""
    resp = await client.chat.completions.create(
        model="qwen/qwen3-235b-a22b:free",
        messages=[
            {
                "role": "system",
                "content": (
                    "あなたはリファクタリングの専門家です。"
                    "SOLID 原則・DRY 原則・YAGNI 原則に基づいてコードを評価し、"
                    "具体的な改善提案を出してください。"
                    "改善後のコードスニペットも示すこと（関数単位でよい）。"
                ),
            },
            {"role": "user", "content": f"```typescript\n{code}\n```"},
        ],
        max_tokens=2048,
    )
    return resp.choices[0].message.content or ""


async def analyze_file(filepath: str) -> dict:
    """テストヒントとリファクタ提案を並列実行"""
    with open(filepath) as f:
        code = f.read()

    test_task = asyncio.create_task(generate_test_hints(code, filepath))
    refactor_task = asyncio.create_task(generate_refactor_suggestions(code, filepath))

    test_hints, refactor_suggestions = await asyncio.gather(
        test_task, refactor_task
    )

    return {
        "file": filepath,
        "test_hints": test_hints,
        "refactor_suggestions": refactor_suggestions,
    }


async def main():
    import sys
    import subprocess

    targets = sys.argv[1:] or subprocess.check_output(
        ["git", "diff", "--name-only", "HEAD"]
    ).decode().splitlines()

    targets = [f for f in targets if f.endswith((".ts", ".tsx", ".py"))]

    results = await asyncio.gather(*[analyze_file(f) for f in targets])

    for result in results:
        print(f"\n{'#' * 60}")
        print(f"# {result['file']}")
        print(f"{'#' * 60}")
        print("\n## 📋 テストヒント")
        print(result["test_hints"])
        print("\n## 🔧 リファクタリング提案")
        print(result["refactor_suggestions"])


if __name__ == "__main__":
    asyncio.run(main())
```

### 並列実行の効果

| 実行方法 | 1 ファイルあたりの待機時間 |
|---|---|
| 直列 (テスト → リファクタ) | ~10〜15 秒 |
| 並列 (`asyncio.gather`) | ~5〜8 秒 (最も遅い方に合わせる) |

---

## 5 パターン全体のまとめ

| パターン | 使用モデル推奨 | 用途 | トリガー例 |
|---|---|---|---|
| 1. 差分レビュー | `qwen3-235b:free` | PR 単位の網羅レビュー | GitHub Actions (PR open) |
| 2. Custom Command | Claude Code 標準 + CLAUDE.md | エディタ内その場確認 | `/review` |
| 3. セキュリティスキャン | `deepseek-r1:free` | OWASP Top 10 チェック | git pre-push hook |
| 4. JSDoc 自動生成 | `gemma-3-27b:free` | ドキュメント整備 | 手動 or 定期バッチ |
| 5. テスト + リファクタ並列 | `llama-3.3-70b:free` + `qwen3-235b:free` | 品質改善 | PR open or 手動 |

---

## 運用上の注意点

### `:free` モデルのレート制限

OpenRouter の `:free` モデルには **RPM (Requests Per Minute)** と **TPD (Tokens Per Day)** の制限がある。  
大量のファイルを一度に処理する場合は以下の対策を取ること。

```python
import time

# シンプルなレート制限対策
for filepath in targets:
    result = process(filepath)
    time.sleep(1.0)  # 1 リクエスト/秒に制限
```

### モデルの可用性

`:free` モデルは OpenRouter の混雑状況によって**レスポンス時間が変動**する（数秒〜数十秒）。  
CI の `timeout` は余裕を持って設定すること（推奨: 120 秒以上）。

### コンテキスト長の上限

大きなファイルはコンテキスト長の上限に引っかかる場合がある。  
その場合は関数単位に分割してリクエストするか、`git diff --unified=3` で差分を圧縮する。

```python
# ファイルが大きい場合は関数単位に分割する簡易例
import ast

def split_by_functions(source: str) -> list[str]:
    """Python ソースを関数単位に分割する"""
    tree = ast.parse(source)
    functions = []
    for node in ast.walk(tree):
        if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef)):
            functions.append(ast.get_source_segment(source, node) or "")
    return functions
```

---

## まとめ

本記事では Claude Code の拡張機構と OpenRouter `:free` モデルを組み合わせた 5 つのパターンを紹介した。

**コスト $0 で実現できること**:
- PR 単位の AI コードレビュー
- OWASP ベースのセキュリティスキャン
- JSDoc / TSDoc 自動生成
- テストヒント・リファクタ提案の並列取得

**今後のステップ**:
- 検出された問題の自動 GitHub Issue 起票
- MCP サーバとして既存ツール (ESLint / Semgrep) との連携
- レビュー精度の週次集計とモデルローテーション

AI レビューはあくまでも**人間のレビューを補助するもの**。False Positive も発生するため、最終判断は必ず人間が行うようにしよう。

---

## 参考リンク

- [OpenRouter Documentation — Free Models](https://openrouter.ai/docs/overview/models)
- [Claude Code — Custom Slash Commands (Anthropic 公式)](https://docs.anthropic.com/en/docs/claude-code/slash-commands)
- [Claude Code — Hooks (Anthropic 公式)](https://docs.anthropic.com/en/docs/claude-code/hooks)
- [Claude Code — CLAUDE.md (Anthropic 公式)](https://docs.anthropic.com/en/docs/claude-code/memory)
- [OWASP Top 10 2021](https://owasp.org/www-project-top-ten/)
- [OpenAI Python SDK — Base URL 変更](https://github.com/openai/openai-python)
- [asyncio.gather — Python 公式ドキュメント](https://docs.python.org/3/library/asyncio-task.html#asyncio.gather)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
