---
title: "Rust × axum で作る高速 REST API入門 — 5 つの実装パターンで理解する非同期サーバー設計"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Rust の async/await と **axum 0.8** を組み合わせると、Node.js 比で **2〜5 倍の RPS** を低メモリで達成できる
- ルーティング・ミドルウェア・状態共有・エラーハンドリング・graceful shutdown の 5 パターンを最小コードで解説
- tokio + Tower エコシステムの理解が axum を使いこなす鍵

---

## 背景

Web バックエンドに Rust を採用する事例が 2025〜2026 年にかけて急増しています。Discord・Cloudflare Workers・Fly.io などが本番で Rust を使うと公表したことで、「Rust = システムプログラミング専用」というイメージは過去のものになりつつあります。

その中心にいるフレームワークが **axum**（Tokio チームが開発・MIT ライセンス）です。axum は Actix-web のような独自 Actor モデルではなく、**Tower サービス抽象**の上に薄いルーティング層を載せた設計で、ミドルウェアの再利用性が非常に高いのが特徴です。

本記事では、axum 0.8 系を使って REST API を作るときに必ず遭遇する **5 つの実装パターン** を、コピペして動くコードとともに解説します。

---

## 前提環境

```
Rust 1.78+
axum 0.8
tokio 1.x (features = ["full"])
serde / serde_json
tower-http 0.6
```

`Cargo.toml` の最小構成:

```toml
[dependencies]
axum      = "0.8"
tokio     = { version = "1", features = ["full"] }
serde     = { version = "1", features = ["derive"] }
serde_json = "1"
tower-http = { version = "0.6", features = ["trace", "cors"] }
tracing-subscriber = "0.3"
```

---

## パターン 1: 基本ルーティングと Path / Query パラメータ抽出

axum のルーターは `Router::new().route(path, method_handler)` で構築します。パスパラメータは `Path<T>`、クエリパラメータは `Query<T>` エクストラクタで型安全に受け取れます。

```rust
use axum::{
    extract::{Path, Query},
    routing::get,
    Json, Router,
};
use serde::{Deserialize, Serialize};

#[derive(Serialize)]
struct Item {
    id: u32,
    name: String,
}

#[derive(Deserialize)]
struct Pagination {
    page: Option<u32>,
    per_page: Option<u32>,
}

// GET /items/:id
async fn get_item(Path(id): Path<u32>) -> Json<Item> {
    Json(Item {
        id,
        name: format!("item-{id}"),
    })
}

// GET /items?page=1&per_page=20
async fn list_items(Query(params): Query<Pagination>) -> Json<Vec<Item>> {
    let page = params.page.unwrap_or(1);
    let per_page = params.per_page.unwrap_or(20);
    let items = (0..per_page)
        .map(|i| Item {
            id: (page - 1) * per_page + i,
            name: format!("item-{}", (page - 1) * per_page + i),
        })
        .collect();
    Json(items)
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/items/:id", get(get_item))
        .route("/items", get(list_items));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

**ポイント:** axum 0.7 以降は `axum::Server` が廃止され `axum::serve(listener, app)` に統一されました。0.6 系のサンプルをそのまま使うとコンパイルエラーになるので注意。

---

## パターン 2: 状態共有 — `Arc<AppState>` パターン

複数のハンドラから DB コネクションプールや設定値を共有する場合、`State` エクストラクタを使います。スレッドセーフにするため `Arc` でラップするのが定石です。

```rust
use axum::{extract::State, routing::get, Router};
use std::sync::Arc;

struct AppState {
    db_pool: String,      // 実際は sqlx::PgPool など
    app_name: String,
}

async fn health(State(state): State<Arc<AppState>>) -> String {
    format!("{} is running (db: {})", state.app_name, state.db_pool)
}

#[tokio::main]
async fn main() {
    let state = Arc::new(AppState {
        db_pool: "postgres://localhost/mydb".into(),
        app_name: "my-api".into(),
    });

    let app = Router::new()
        .route("/health", get(health))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

**ポイント:** `Router::with_state()` で状態を注入するとハンドラ関数の型が `State<Arc<AppState>>` として静的チェックされます。実行時エラーではなくコンパイルエラーで気付けるのが Rust らしい強みです。

---

## パターン 3: エラーハンドリング — `impl IntoResponse` で型を統一

axum のハンドラは戻り値が `IntoResponse` を実装していれば何でも返せます。アプリ固有のエラー型を定義して `IntoResponse` を実装するパターンが最もスッキリします。

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;

// アプリ固有エラー型
enum AppError {
    NotFound(String),
    Internal(String),
    BadRequest(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::NotFound(msg)   => (StatusCode::NOT_FOUND, msg),
            AppError::Internal(msg)   => (StatusCode::INTERNAL_SERVER_ERROR, msg),
            AppError::BadRequest(msg) => (StatusCode::BAD_REQUEST, msg),
        };
        (status, Json(json!({ "error": message }))).into_response()
    }
}

// 使用例: ハンドラが Result<Json<T>, AppError> を返す
async fn get_user(axum::extract::Path(id): axum::extract::Path<u32>)
    -> Result<Json<serde_json::Value>, AppError>
{
    if id == 0 {
        return Err(AppError::BadRequest("id must be > 0".into()));
    }
    if id > 1000 {
        return Err(AppError::NotFound(format!("user {id} not found")));
    }
    Ok(Json(json!({ "id": id, "name": format!("user-{id}") })))
}
```

`anyhow::Error` を `AppError::Internal` に変換する `From` 実装も合わせて書いておくと、`?` 演算子で既存の anyhow ベースコードと統合できます。

```rust
impl From<anyhow::Error> for AppError {
    fn from(e: anyhow::Error) -> Self {
        AppError::Internal(e.to_string())
    }
}
```

---

## パターン 4: ミドルウェア — Tower + tower-http で横断処理

認証・ロギング・CORS・レートリミットなどの横断処理は Tower ミドルウェアとして `layer()` で追加します。tower-http クレートが豊富なビルトインを提供しています。

```rust
use axum::{routing::get, Router};
use tower_http::{
    cors::{Any, CorsLayer},
    trace::TraceLayer,
};
use tracing_subscriber::EnvFilter;

async fn hello() -> &'static str {
    "Hello, world!"
}

#[tokio::main]
async fn main() {
    // tracing 初期化 (RUST_LOG=info で実行)
    tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::from_default_env())
        .init();

    let cors = CorsLayer::new()
        .allow_origin(Any)
        .allow_methods(Any)
        .allow_headers(Any);

    let app = Router::new()
        .route("/", get(hello))
        .layer(TraceLayer::new_for_http())  // リクエスト/レスポンスを自動ログ
        .layer(cors);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

**カスタムミドルウェアを書く場合** は `tower::Service` トレイトを実装するか、より簡単な `axum::middleware::from_fn()` を使います:

```rust
use axum::{
    middleware::{self, Next},
    extract::Request,
    response::Response,
    http::HeaderMap,
};

async fn auth_middleware(
    headers: HeaderMap,
    request: Request,
    next: Next,
) -> Response {
    if headers.get("x-api-key").and_then(|v| v.to_str().ok()) == Some("secret") {
        next.run(request).await
    } else {
        axum::http::StatusCode::UNAUTHORIZED.into_response()
    }
}

// 特定ルートにのみ認証を適用
let protected = Router::new()
    .route("/admin", get(admin_handler))
    .layer(middleware::from_fn(auth_middleware));
```

`layer()` は **後から書いたものが外側（先に実行）** になるので、ロギングを最外層に置く場合は最後に `.layer(TraceLayer::new_for_http())` を書くのが正しい順序です。

---

## パターン 5: Graceful Shutdown — シグナルを受けて安全に終了

本番環境では SIGTERM を受けたときに処理中のリクエストを完了させてから終了する必要があります。axum 0.7+ では `serve().with_graceful_shutdown()` が用意されています。

```rust
use axum::{routing::get, Router};
use tokio::signal;

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install SIGTERM handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }

    tracing::info!("shutdown signal received, starting graceful shutdown");
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "ok" }));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}
```

Kubernetes や systemd 環境では `SIGTERM` を受けてから `terminationGracePeriodSeconds`（デフォルト 30 秒）以内に終了することが求められます。このパターンを入れておくだけで `502 Bad Gateway` の頻度が大きく下がります。

---

## 5 パターンをまとめて組み合わせる

実際のプロジェクトでは上記を 1 つの `main.rs` に統合します。構成イメージ:

```
src/
├── main.rs          ... サーバー起動 + graceful shutdown
├── state.rs         ... AppState 定義 (DB プール等)
├── error.rs         ... AppError + IntoResponse 実装
├── routes/
│   ├── mod.rs       ... Router を組み立て返す関数
│   ├── items.rs     ... /items 系ハンドラ
│   └── users.rs     ... /users 系ハンドラ
└── middleware/
    └── auth.rs      ... from_fn ミドルウェア
```

`routes/mod.rs` で `Router::nest()` を使うとプレフィックスごとにルーターを分割できます:

```rust
pub fn app_router(state: Arc<AppState>) -> Router {
    Router::new()
        .nest("/api/v1/items", items::router())
        .nest("/api/v1/users", users::router())
        .layer(TraceLayer::new_for_http())
        .layer(CorsLayer::new().allow_origin(Any))
        .with_state(state)
}
```

---

## パフォーマンス特性と使いどころ

| フレームワーク | 言語 | 目安 RPS (単純 JSON echo) | メモリ (idle) |
|---|---|---|---|
| axum 0.8 | Rust | 〜 200,000 | 5–10 MB |
| Actix-web 4 | Rust | 〜 220,000 | 8–15 MB |
| Fastify 5 | Node.js | 〜 80,000 | 60–80 MB |
| FastAPI | Python | 〜 20,000 | 80–120 MB |

> ※ 数値は公開ベンチマーク (TechEmpower Framework Benchmarks Round 22、Rust-lang 公式ブログ等) を参考にした概算。実際のアプリはDB アクセスやビジネスロジックによって大幅に変わります。

axum が特に輝く場面:
- **大量の同時接続** (WebSocket チャット・SSE ストリーミング)
- **エッジ環境** (メモリ制約が厳しい VPS・Fargate Spot)
- **型安全を最優先したい API** (コンパイル時に抜け漏れを検出)

逆に、プロトタイプ速度優先・チームに Rust 経験者がいない場合は Fastify や Hono (TypeScript) から始めて、ボトルネックが見えてから Rust に移行するアプローチも現実的です。

---

## まとめ

| パターン | 使う API |
|---|---|
| ルーティング + パラメータ | `Router`, `Path<T>`, `Query<T>` |
| 状態共有 | `State<Arc<T>>`, `.with_state()` |
| エラーハンドリング | `impl IntoResponse for AppError` |
| ミドルウェア | `.layer()`, `from_fn()`, tower-http |
| Graceful shutdown | `.with_graceful_shutdown()`, `tokio::signal` |

axum は「薄いラッパーとして Tower を露出する」設計思想なので、最初は「なぜ自分でサービスを合成しないといけないのか」と戸惑うかもしれません。しかし Tower の概念を一度掴むと、認証・レートリミット・タイムアウトを **組み合わせ可能なブロック**として扱えるようになり、設計の自由度が格段に上がります。

5 つのパターンを手元で動かして、まず `/health` エンドポイントを完成させることから始めてみてください。

---

## 参考リンク

- [axum 公式ドキュメント (docs.rs)](https://docs.rs/axum/latest/axum/)
- [axum GitHub (tokio-rs/axum) — MIT License](https://github.com/tokio-rs/axum)
- [tower-http — MIT License](https://github.com/tower-rs/tower-http)
- [TechEmpower Framework Benchmarks](https://www.techempower.com/benchmarks/)
- [tokio チュートリアル](https://tokio.rs/tokio/tutorial)
- [Rust async book](https://rust-lang.github.io/async-book/)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
