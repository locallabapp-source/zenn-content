---
title: "axum で 308 Permanent Redirect を実装する 5 パターンと 3 つの落とし穴"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- リダイレクトの移行は、POST などメソッドを保持したい場合に **301 ではなく 308** を選ぶのが安全です。
- axum では `Redirect::permanent()` が 308 を返します (`Redirect::temporary()` は 307、`Redirect::to()` は 303)。
- この記事では次の 5 パターンをコード付きで整理します。
  1. 単発ルートの 308
  2. クエリ文字列を保持したままのパス prefix 移行
  3. `www` から apex への正規化ミドルウェア
  4. テーブル駆動の大量リダイレクト
  5. `oneshot` によるテスト
- 最後に、ブラウザキャッシュなど本番投入前に知っておきたい落とし穴を 3 つ紹介します。

対象読者は Rust と axum の基本 (Router / handler) が分かる中級者です。

## 1. なぜ 301 ではなく 308 なのか

恒久的なリダイレクトには、歴史的に 301 が使われてきました。ただし 301 には仕様上のあいまいさがあります。**POST が 301 で GET に書き換えられるクライアント実装が存在する**ためです。

| ステータス | 恒久/一時 | メソッド保持 | axum のコンストラクタ |
|---|---|---|---|
| 301 Moved Permanently | 恒久 | 保証なし (GET に変わることがある) | 標準では提供なし |
| 302 Found | 一時 | 保証なし | 標準では提供なし |
| 303 See Other | 一時 | GET に変更 | `Redirect::to` |
| 307 Temporary Redirect | 一時 | **保持** | `Redirect::temporary` |
| 308 Permanent Redirect | 恒久 | **保持** | `Redirect::permanent` |

308 は 301 の「メソッドとボディを保持する版」にあたります。API エンドポイントの URL 移行では、POST が GET に化けない 308 のほうが事故が起きにくくなります。

検索エンジンの扱いについては、Google Search Central のドキュメントで 301 と 308 は同じ「恒久的リダイレクト」のシグナルとして扱われると説明されています。SEO 目的でも、308 を選んで不利になることは基本的にありません。詳細は参考リンクの公式ドキュメントを確認してください。

## 2. 準備

検証環境は次のとおりです。以降のコードは axum 0.7 系を前提に書いています。

```toml
[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }

[dev-dependencies]
tower = { version = "0.5", features = ["util"] }
```

axum、tokio、tower はいずれも MIT ライセンスです。

## 3. パターン 1: 単発ルートの 308

最もシンプルな形です。

```rust
use axum::{response::Redirect, routing::get, Router};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/old", get(|| async { Redirect::permanent("/new") }))
        .route("/new", get(|| async { "new page" }));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

動作確認は `curl` で行います。

```bash
curl -i http://127.0.0.1:3000/old
# HTTP/1.1 308 Permanent Redirect
# location: /new
```

注意点が 1 つあります。上のコードは `get` だけを登録しているため、`POST /old` は 405 になります。メソッド保持の 308 の効果を活かすには、旧エンドポイントで受けたいメソッドを `post(...)` や `any(...)` で登録する必要があります。

```rust
use axum::routing::any;

let app = Router::new().route(
    "/api/old-endpoint",
    any(|| async { Redirect::permanent("/api/new-endpoint") }),
);
```

## 4. パターン 2: クエリ文字列を保持したままパス prefix を移行

`/v1/*` を `/v2/*` に移すとき、クエリ文字列を落とすと検索 URL などが壊れます。`OriginalUri` を使えば、元のパスとクエリをそのまま扱えます。

```rust
use axum::{extract::OriginalUri, response::Redirect, routing::get, Router};

async fn v1_to_v2(OriginalUri(uri): OriginalUri) -> Redirect {
    let pq = uri
        .path_and_query()
        .map(|p| p.as_str())
        .unwrap_or("/");

    let target = match pq.strip_prefix("/v1/") {
        Some(rest) => format!("/v2/{rest}"),
        None => "/v2/".to_string(),
    };
    Redirect::permanent(&target)
}

fn app() -> Router {
    Router::new().route("/v1/*rest", get(v1_to_v2))
}
```

`/v1/items?page=2&sort=desc` は `/v2/items?page=2&sort=desc` へ 308 で飛びます。

> axum 0.8 以降はワイルドカードの記法が `/{*rest}` に変わっています。バージョンに合わせて書き換えてください。

`path_and_query` の文字列を丸ごと扱っているため、パーセントエンコードされたパスもデコードされずに渡ります。リダイレクト先でエンコードが壊れる心配がありません。

## 5. パターン 3: `www` から apex への正規化ミドルウェア

ドメインの正規化は、複数のルートにまたがる横断的な関心事なので、ミドルウェアで書くのが適切です。

```rust
use axum::{
    extract::Request,
    http::header,
    middleware::{self, Next},
    response::{IntoResponse, Redirect, Response},
    routing::get,
    Router,
};

async fn canonical_host(req: Request, next: Next) -> Response {
    let host = req
        .headers()
        .get(header::HOST)
        .and_then(|v| v.to_str().ok())
        .unwrap_or("")
        .to_owned();

    if let Some(bare) = host.strip_prefix("www.") {
        let pq = req
            .uri()
            .path_and_query()
            .map(|p| p.as_str())
            .unwrap_or("/");
        return Redirect::permanent(&format!("https://{bare}{pq}")).into_response();
    }
    next.run(req).await
}

fn app() -> Router {
    Router::new()
        .route("/", get(|| async { "home" }))
        .layer(middleware::from_fn(canonical_host))
}
```

リバースプロキシ (Traefik / nginx など) の背後で動かす場合、`Host` ヘッダがそのまま届くとは限りません。プロキシの設定次第で、元のホストは `X-Forwarded-Host` に入ります。信頼できるプロキシから来たヘッダだけを参照する設計にしてください。ヘッダを無条件に信用すると、Open Redirect の温床になります。

また、`https://` を固定で書いているのは、本番が常に HTTPS である前提だからです。HTTP から HTTPS への変換をプロキシ側で行っているなら、アプリ側ではホスト正規化だけに責務を絞ると見通しがよくなります。

## 6. パターン 4: テーブル駆動で大量のリダイレクトを登録

ブログ記事の URL 構造を変えたときなど、旧 URL が数十から数百件に及ぶことがあります。ハンドラを 1 つずつ書くのは現実的ではないため、定数テーブルをループで登録します。

```rust
use axum::{response::Redirect, routing::get, Router};

const MOVED: &[(&str, &str)] = &[
    ("/blog/hello-world", "/articles/hello-world"),
    ("/blog/rust-async", "/articles/rust-async-intro"),
    ("/about-us", "/about"),
];

fn with_redirects(mut router: Router) -> Router {
    for &(from, to) in MOVED {
        router = router.route(
            from,
            get(move || async move { Redirect::permanent(to) }),
        );
    }
    router
}

fn app() -> Router {
    with_redirects(Router::new().route("/about", get(|| async { "about" })))
}
```

`to` は `&'static str` (`Copy`) なので、`move` クロージャに安全に取り込めます。件数が増えたら、次の 2 つを検討してください。

- テーブルを CSV や TOML に外出しし、`include_str!` でビルド時に埋め込む
- 登録時に **リダイレクトチェーン** (A→B→C) と **ループ** (A→B→A) を検出するテストを書く

リダイレクトチェーンはクロール効率を下げます。チェーンやループの検出は、1 回書いておけば後の移行作業が楽になります。

```rust
#[test]
fn no_redirect_chain() {
    let froms: std::collections::HashSet<_> = MOVED.iter().map(|(f, _)| *f).collect();
    for (from, to) in MOVED {
        assert!(!froms.contains(to), "chain detected: {from} -> {to}");
    }
}
```

## 7. パターン 5: `oneshot` でリダイレクトをテストする

サーバーを実際に起動しなくても、`tower::ServiceExt::oneshot` で Router に直接リクエストを流せます。

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use axum::{
        body::Body,
        http::{header, Request, StatusCode},
    };
    use tower::ServiceExt; // oneshot

    #[tokio::test]
    async fn old_path_returns_308_with_location() {
        let res = app()
            .oneshot(
                Request::builder()
                    .uri("/about-us")
                    .body(Body::empty())
                    .unwrap(),
            )
            .await
            .unwrap();

        assert_eq!(res.status(), StatusCode::PERMANENT_REDIRECT);
        assert_eq!(res.headers()[header::LOCATION], "/about");
    }
}
```

`StatusCode::PERMANENT_REDIRECT` が 308 です。`StatusCode::MOVED_PERMANENTLY` (301) と取り違えないよう注意してください。

テストしておくと安心なケースを表にまとめます。

| ケース | 期待値 |
|---|---|
| 旧 URL (GET) | 308 + `Location` が新 URL |
| 旧 URL + クエリ | クエリが保持される |
| 旧 URL (POST) | 308 (405 にならない) |
| 新 URL | 200 (リダイレクトしない) |
| 存在しない URL | 404 |

## 8. 本番投入前の 3 つの落とし穴

### 落とし穴 1: 308 はブラウザに強くキャッシュされる

恒久リダイレクトは、ブラウザが長期間キャッシュすることがあります。**リダイレクト先を間違えたまま公開すると、利用者のブラウザ側に残り、サーバーを直しても反映されない**事態になります。

対策は次のとおりです。

- 最初は `Redirect::temporary` (307) でデプロイして挙動を確認し、問題がなければ 308 に切り替える
- 開発中は `curl -i` で確認する (ブラウザのキャッシュに影響されない)
- ブラウザで確認するなら、DevTools の「Disable cache」を有効にするか、シークレットウィンドウを使う

### 落とし穴 2: 相対パスの `Location` とプロキシ

`Redirect::permanent("/new")` は相対参照の `Location` を返します。RFC 9110 では相対参照も許可されており、多くのクライアントは問題なく解決します。ただし、中間のプロキシや一部のクライアントとの相性が気になる場合は、絶対 URL を組み立てて返すと安全です。その際は **ベース URL を設定値として持ち、リクエストヘッダから組み立てない**のが無難です。

### 落とし穴 3: リダイレクトのチェーンとループ

移行を何度か繰り返すと、`A→B→C` のようにリダイレクトが連鎖しがちです。ブラウザやクローラーは、一定回数以上のリダイレクトで追跡を打ち切ります。移行のたびに、**旧 URL から最終 URL へ 1 ホップで飛ぶ**ように、テーブルを更新してください。パターン 4 のチェーン検出テストが役立ちます。

## 9. どのパターンを選ぶか

| 状況 | 推奨パターン |
|---|---|
| 1〜2 本の URL を移すだけ | パターン 1 |
| API のバージョン prefix を変える | パターン 2 |
| ドメイン正規化 (www 有無) | パターン 3 (またはプロキシ側) |
| 記事 URL を大量に付け替える | パターン 4 |
| どの場合でも | パターン 5 のテストを書く |

ドメイン正規化や HTTPS 強制は、リバースプロキシ側 (Traefik や nginx のミドルウェア) で行う選択肢もあります。**アプリ側に持たせるのは、ビジネスロジックに近い URL の付け替え**に限り、インフラ寄りの正規化はプロキシに寄せると
