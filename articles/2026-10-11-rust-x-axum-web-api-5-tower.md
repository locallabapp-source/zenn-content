---
title: "Rust × axum で学ぶ非同期 Web API 設計：5 つのパターンで理解する Tower ミドルウェア"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- axum は Tower エコシステムの上に乗る薄いルーティングレイヤーである
- `tower::ServiceBuilder` を使えば認証・ロギング・レート制限を **コンポーザブルに** 積み重ねられる
- 本記事では 5 つの実装パターンを通じて「なぜ Tower なのか」を体感する

---

## はじめに：axum を選ぶ理由

Rust の HTTP フレームワーク選定時、`actix-web` と `axum` の二択になることが多い。  
axum の最大の強みは **Tower との完全な親和性** だ。

Tower は `Service` トレイトを中心に構成されたミドルウェアエコシステムで、以下を提供する。

- `tower::Service` — `call(Request) -> Future<Response>` を抽象化するトレイト
- `tower::Layer` — Service を包んで新しい Service を作るトレイト
- `tower::ServiceBuilder` — Layer を積み重ねるビルダー

actix-web がアクターモデルを採用するのに対して、axum は **Tokio 非同期ランタイム + Tower** という組み合わせで「Rust 標準の非同期」に乗り続けるという設計思想を持つ。

---

## 前提：Cargo.toml

```toml
[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
tower = { version = "0.4", features = ["util", "limit", "timeout"] }
tower-http = { version = "0.5", features = ["trace", "cors", "compression-gzip"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

---

## パターン 1 — 最小ルーティング

axum の最小構成を確認する。

```rust
use axum::{routing::get, Router};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/health", get(health_handler));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn health_handler() -> &'static str {
    "OK"
}
```

`axum::serve` は `axum 0.7` で導入された新 API だ。`0.6` 以前の `axum::Server::bind` は削除されているため注意。

---

## パターン 2 — JSON リクエスト / レスポンス

axum は `axum::Json<T>` エクストラクタで Serde の自動デシリアライズを行う。

```rust
use axum::{
    extract::Json,
    http::StatusCode,
    response::IntoResponse,
    routing::post,
    Router,
};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct CreateUserRequest {
    name: String,
    email: String,
}

#[derive(Serialize)]
struct CreateUserResponse {
    id: u64,
    name: String,
}

async fn create_user(
    Json(payload): Json<CreateUserRequest>,
) -> impl IntoResponse {
    // 実際の永続化は省略
    let response = CreateUserResponse {
        id: 42,
        name: payload.name,
    };

    (StatusCode::CREATED, Json(response))
}

pub fn user_router() -> Router {
    Router::new().route("/users", post(create_user))
}
```

`IntoResponse` トレイトを実装した任意の型がレスポンスになれる。  
タプル `(StatusCode, Json<T>)` は標準で `IntoResponse` を実装しているため上記の書き方が成立する。

---

## パターン 3 — State の注入と共有

実用的な API では DB プール等の共有状態が必要になる。  
axum では `State` エクストラクタで **Arc 共有** を安全に扱う。

```rust
use axum::{
    extract::State,
    routing::get,
    Router,
};
use std::sync::Arc;

// アプリ全体で共有するステート
#[derive(Clone)]
struct AppState {
    db_url: String,
    // 実務では sqlx::PgPool 等を入れる
}

async fn get_config(
    State(state): State<Arc<AppState>>,
) -> String {
    format!("connected to: {}", state.db_url)
}

pub fn app(state: Arc<AppState>) -> Router {
    Router::new()
        .route("/config", get(get_config))
        .with_state(state)
}

#[tokio::main]
async fn main() {
    let state = Arc::new(AppState {
        db_url: "postgres://localhost/mydb".to_string(),
    });

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    axum::serve(listener, app(state)).await.unwrap();
}
```

**`Arc<T>` を渡す理由**：`Router::with_state` はリクエストごとに State を clone するため、`Arc` で包んでポインタコピーにとどめることが推奨される。

---

## パターン 4 — ServiceBuilder で 3 層ミドルウェアを積む

いよいよ Tower の本領発揮だ。`tower_http` と組み合わせて、**CORS・タイムアウト・トレーシング** を 1 箇所で宣言する。

```rust
use axum::{routing::get, Router};
use std::time::Duration;
use tower::ServiceBuilder;
use tower_http::{
    cors::{Any, CorsLayer},
    trace::TraceLayer,
};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // tracing の初期化
    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::new(
            std::env::var("RUST_LOG").unwrap_or_else(|_| "info".into()),
        ))
        .with(tracing_subscriber::fmt::layer())
        .init();

    let cors = CorsLayer::new()
        .allow_origin(Any)
        .allow_methods(Any)
        .allow_headers(Any);

    let middleware_stack = ServiceBuilder::new()
        .layer(TraceLayer::new_for_http())       // ① リクエスト/レスポンスの自動ログ
        .layer(cors)                              // ② CORS ヘッダ付与
        .timeout(Duration::from_secs(10));       // ③ 10 秒タイムアウト

    let app = Router::new()
        .route("/", get(|| async { "hello tower" }))
        .layer(middleware_stack);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    axum::serve(listener, app).await.unwrap();
}
```

`ServiceBuilder::layer` の適用順序は **上から下** でリクエストが通過し、**下から上** でレスポンスが戻る。  
つまりトレーシングが最外側に来るため、タイムアウトエラーもログに残る設計になっている。

---

## パターン 5 — カスタムエラーハンドラで API の一貫性を保つ

本番 API では全エラーを統一 JSON 形式で返す必要がある。  
axum の `IntoResponse` トレイトを自前の Error 型に実装すれば `?` 演算子が使えるようになる。

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;

// アプリ固有のエラー型
pub enum AppError {
    NotFound(String),
    Unauthorized,
    InternalServerError(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::NotFound(msg) => (StatusCode::NOT_FOUND, msg),
            AppError::Unauthorized => (
                StatusCode::UNAUTHORIZED,
                "authentication required".to_string(),
            ),
            AppError::InternalServerError(msg) => {
                (StatusCode::INTERNAL_SERVER_ERROR, msg)
            }
        };

        let body = Json(json!({
            "error": {
                "status": status.as_u16(),
                "message": message,
            }
        }));

        (status, body).into_response()
    }
}

// ハンドラ内で ? が使える
async fn get_user(/* ... */) -> Result<Json<serde_json::Value>, AppError> {
    let user_id: u64 = 99;

    // 見つからなかった想定
    if user_id > 50 {
        return Err(AppError::NotFound(format!("user {} not found", user_id)));
    }

    Ok(Json(json!({ "id": user_id })))
}
```

これにより全エンドポイントが `{"error": {"status": 404, "message": "..."}}` という一貫した形式を返す。  
エラーの追加は `AppError` の enum に variant を足すだけで済む。

---

## Tower ミドルウェアの仕組みを 1 分で理解する

```
Request
   │
   ▼
┌──────────────────────────────┐
│  TraceLayer (最外)            │
│  ┌────────────────────────┐  │
│  │  CorsLayer             │  │
│  │  ┌──────────────────┐  │  │
│  │  │  TimeoutLayer    │  │  │
│  │  │  ┌────────────┐  │  │  │
│  │  │  │  Router    │  │  │  │
│  │  │  │ (ハンドラ)  │  │  │  │
│  │  │  └────────────┘  │  │  │
│  │  └──────────────────┘  │  │
│  └────────────────────────┘  │
└──────────────────────────────┘
   │
   ▼
Response
```

`tower::Service` の定義は以下のとおりシンプルだ。

```rust
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;
    fn call(&mut self, req: Request) -> Self::Future;
}
```

`Layer` はこの `Service` を受け取り、別の `Service` を返す。ロシア人形のように何層でも包める設計だ。

---

## 5 パターンの使いどころまとめ

| パターン | 使いどころ | 主要 API |
|---|---|---|
| 1. 最小ルーティング | ヘルスチェック・プロトタイプ | `Router::new`, `routing::get` |
| 2. JSON 入出力 | REST API エンドポイント | `Json<T>`, `IntoResponse` |
| 3. State 注入 | DB プール・設定共有 | `State<T>`, `Router::with_state` |
| 4. ミドルウェアスタック | CORS・ログ・タイムアウト | `ServiceBuilder`, `tower-http` |
| 5. カスタムエラー | API 一貫性・エラーハンドリング | `IntoResponse for AppError` |

---

## よくあるハマりどころ 3 選

### ① Handler の型推論エラー

```
error[E0277]: the trait bound `...: Handler<_, _, _>` is not satisfied
```

ハンドラの引数がすべて「axum が知っているエクストラクタ」でなければならない。  
独自の型を引数にしたい場合は `FromRequest` / `FromRequestParts` を実装する。

### ② State の clone コスト

`Arc<T>` を使わずに `T: Clone` だけにすると、リクエストのたびに T 全体が deep clone される。  
`sqlx::PgPool` などは内部でコネクションプールを持つため `Arc` なしでも問題ないが、自前の大きな構造体は必ず `Arc` で包む。

### ③ axum 0.6 → 0.7 のマイグレーション

- `axum::Server` が削除 → `tokio::net::TcpListener` + `axum::serve` に変更
- `FromRequest` の分割 (`FromRequest` / `FromRequestParts`) が 0.7 で整理された
- `TypedHeader` が `axum-extra` に移動した

---

## まとめ

1. axum は Tower エコシステムの上に乗る「薄いルーター」であり、Tower への理解が直接設計力に繋がる
2. `ServiceBuilder` による宣言的ミドルウェア合成は、actix-web のマクロベース設定より **型安全かつ composable**
3. `IntoResponse` を自前の Error 型に実装するだけで、全体の API 一貫性が確保できる

Tower の `Service` / `Layer` の概念さえつかめば、`tonic` (gRPC) や `hyper` 直使いへの移行コストも大幅に下がる。  
まず本記事の 5 パターンを手元で動かして体感してほしい。

---

## 参考リンク

- [axum 公式ドキュメント](https://docs.rs/axum/latest/axum/)
- [tower 公式ドキュメント](https://docs.rs/tower/latest/tower/)
- [tower-http 公式ドキュメント](https://docs.rs/tower-http/latest/tower_http/)
- [tokio 公式チュートリアル — mini-redis](https://tokio.rs/tokio/tutorial)
- [axum 0.6 → 0.7 マイグレーションガイド](https://github.com/tokio-rs/axum/blob/main/axum/CHANGELOG.md)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
