---
title: "Rust × axum で作る高速 WebSocket サーバー：10 万接続を捌く 5 つの実装パターン"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- axum 0.7 + tokio の非同期ランタイムで WebSocket サーバーを構築する方法を解説
- 接続数スケールのボトルネックになりがちな 5 箇所を実装レベルで解説
- バックプレッシャー・ハートビート・グレースフルシャットダウンまでカバー

---

## はじめに

「WebSocket で 10 万接続を捌きたい」という要件は、チャット・通知基盤・リアルタイムダッシュボードなど多くの SaaS で発生します。Node.js + ws でプロトタイプを作ったあと、Rust + axum に移行するケースが増えています。

本記事では **axum 0.7 系**（2024 年末〜2025 年以降のエコシステム）を前提に、実際に大量接続を意識した実装パターンを 5 つ紹介します。

> **対象読者**: Rust の基礎文法は読める・axum の Hello World 程度は動かしたことがある中級者以上

---

## 前提環境

| ツール | バージョン |
|---|---|
| Rust | 1.78 以上 (2024 edition) |
| axum | 0.7.x |
| tokio | 1.x (multi-thread runtime) |
| tokio-tungstenite | 0.23.x |

`Cargo.toml` の依存関係:

```toml
[dependencies]
axum = { version = "0.7", features = ["ws"] }
tokio = { version = "1", features = ["full"] }
tower = "0.4"
tower-http = { version = "0.5", features = ["trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
dashmap = "5"
```

---

## パターン 1: 最小構成の WebSocket ハンドシェイク

まず axum で WebSocket をアップグレードする最小実装から始めます。

```rust
use axum::{
    extract::ws::{Message, WebSocket, WebSocketUpgrade},
    response::IntoResponse,
    routing::get,
    Router,
};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/ws", get(ws_handler));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn ws_handler(ws: WebSocketUpgrade) -> impl IntoResponse {
    ws.on_upgrade(handle_socket)
}

async fn handle_socket(mut socket: WebSocket) {
    while let Some(Ok(msg)) = socket.recv().await {
        match msg {
            Message::Text(text) => {
                if socket.send(Message::Text(text)).await.is_err() {
                    break;
                }
            }
            Message::Close(_) => break,
            _ => {}
        }
    }
}
```

**ポイント**: `WebSocketUpgrade` は `on_upgrade` 呼び出し時点で HTTP→WS のプロトコルアップグレードを完了します。`handle_socket` は独立した `tokio::task` として spawn されるため、1 接続あたりの処理がブロックしても他接続には影響しません。

---

## パターン 2: 接続管理ストア (`DashMap` による並列安全なルーム管理)

複数接続にブロードキャストするには、接続 ID → `Sender` のマップが必要です。`std::sync::Mutex<HashMap>` は Write ロックが頻発してボトルネックになります。代わりに **`DashMap`** (シャードロック実装) を使います。

```rust
use axum::{
    extract::{ws::{Message, WebSocket, WebSocketUpgrade}, State},
    response::IntoResponse,
    routing::get,
    Router,
};
use dashmap::DashMap;
use std::sync::Arc;
use tokio::sync::broadcast;

type RoomId = String;
type ConnId  = u64;

#[derive(Clone)]
struct AppState {
    // room_id -> broadcast sender
    rooms: Arc<DashMap<RoomId, broadcast::Sender<String>>>,
    next_id: Arc<std::sync::atomic::AtomicU64>,
}

impl AppState {
    fn new() -> Self {
        Self {
            rooms: Arc::new(DashMap::new()),
            next_id: Arc::new(std::sync::atomic::AtomicU64::new(0)),
        }
    }

    fn join_room(&self, room_id: &str) -> broadcast::Receiver<String> {
        self.rooms
            .entry(room_id.to_string())
            .or_insert_with(|| {
                let (tx, _) = broadcast::channel(256); // バッファ 256 メッセージ
                tx
            })
            .subscribe()
    }

    fn broadcast(&self, room_id: &str, msg: String) {
        if let Some(tx) = self.rooms.get(room_id) {
            let _ = tx.send(msg); // 受信者 0 人のエラーは無視
        }
    }
}

async fn ws_handler(
    ws: WebSocketUpgrade,
    State(state): State<AppState>,
) -> impl IntoResponse {
    ws.on_upgrade(move |socket| handle_socket(socket, state, "general".to_string()))
}

async fn handle_socket(mut socket: WebSocket, state: AppState, room_id: RoomId) {
    let conn_id = state.next_id.fetch_add(1, std::sync::atomic::Ordering::Relaxed);
    let mut rx = state.join_room(&room_id);

    loop {
        tokio::select! {
            // クライアントからの受信
            msg = socket.recv() => {
                match msg {
                    Some(Ok(Message::Text(text))) => {
                        state.broadcast(&room_id, format!("[{}] {}", conn_id, text));
                    }
                    Some(Ok(Message::Close(_))) | None => break,
                    _ => {}
                }
            }
            // ブロードキャスト受信 → クライアントへ送信
            Ok(broadcast_msg) = rx.recv() => {
                if socket.send(Message::Text(broadcast_msg)).await.is_err() {
                    break;
                }
            }
        }
    }
}
```

**ポイント**:
- `tokio::sync::broadcast` は 1 対多配信に最適。チャネルのバッファを超えると古いメッセージが drop されます（後述のバックプレッシャー対策）。
- `DashMap` は内部でシャードに分割してロック粒度を下げるため、多数のルームへの同時 read/write に強い。

---

## パターン 3: バックプレッシャー制御

大量接続時に「送信が追いつかない」クライアントにメッセージを詰め込み続けると、メモリが枯渇します。**バックプレッシャー** を実装してスローなクライアントを適切に切断します。

```rust
use tokio::time::{timeout, Duration};

async fn send_with_backpressure(
    socket: &mut WebSocket,
    msg: Message,
    deadline: Duration,
) -> bool {
    match timeout(deadline, socket.send(msg)).await {
        Ok(Ok(())) => true,
        Ok(Err(_)) => {
            tracing::warn!("send error: client disconnected");
            false
        }
        Err(_elapsed) => {
            tracing::warn!("send timeout: dropping slow client");
            false
        }
    }
}
```

呼び出し側:

```rust
Ok(broadcast_msg) = rx.recv() => {
    let ok = send_with_backpressure(
        &mut socket,
        Message::Text(broadcast_msg),
        Duration::from_millis(200), // 200ms で応答しないなら切断
    ).await;
    if !ok { break; }
}
```

**ポイント**:
- `timeout` ラッパーは `tokio::time::timeout` で実装。async fn の中でも `await` ポイントとして機能します。
- 適切な deadline は用途依存。ゲームサーバーなら 16ms、通知基盤なら 1-2s が目安。

---

## パターン 4: ハートビート (Ping/Pong) による死活監視

TCP レベルでは切断を検知できないケースがあります（NAT の沈黙・モバイルスリープなど）。定期的な Ping で死んでいる接続を刈り取ります。

```rust
use tokio::time::{interval, Duration};

async fn handle_socket_with_heartbeat(mut socket: WebSocket, state: AppState, room_id: RoomId) {
    let mut rx = state.join_room(&room_id);
    let mut heartbeat = interval(Duration::from_secs(30));
    let mut awaiting_pong = false;

    loop {
        tokio::select! {
            _ = heartbeat.tick() => {
                if awaiting_pong {
                    // 前回の Ping に Pong が来ていない → 切断
                    tracing::info!("heartbeat timeout, closing");
                    break;
                }
                if socket.send(Message::Ping(vec![])).await.is_err() {
                    break;
                }
                awaiting_pong = true;
            }

            msg = socket.recv() => {
                match msg {
                    Some(Ok(Message::Pong(_))) => {
                        awaiting_pong = false; // 生きている
                    }
                    Some(Ok(Message::Text(text))) => {
                        state.broadcast(&room_id, text.into());
                    }
                    Some(Ok(Message::Close(_))) | None => break,
                    _ => {}
                }
            }

            Ok(bcast) = rx.recv() => {
                if socket.send(Message::Text(bcast)).await.is_err() {
                    break;
                }
            }
        }
    }
}
```

**ポイント**:
- axum の WebSocket 実装は RFC 6455 準拠で、サーバー送信の `Ping` に対してクライアントは自動で `Pong` を返すことが仕様上期待されます。
- `awaiting_pong` フラグで「前回 Ping に応答があったか」を管理します。30 秒タイマーを 2 周しても応答がなければ切断。

---

## パターン 5: グレースフルシャットダウン

OS シグナル (`SIGTERM`) を受け取ったとき、接続中の全クライアントに `Close` フレームを送ってから終了します。

```rust
use tokio::signal;

#[tokio::main]
async fn main() {
    let state = AppState::new();
    // シャットダウン通知用チャネル
    let (shutdown_tx, _) = broadcast::channel::<()>(1);

    let app = Router::new()
        .route("/ws", get(ws_handler))
        .with_state((state.clone(), shutdown_tx.clone()));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal(shutdown_tx))
        .await
        .unwrap();
}

async fn shutdown_signal(tx: broadcast::Sender<()>) {
    let ctrl_c = async {
        signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install signal handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {}
        _ = terminate => {}
    }

    tracing::info!("shutdown signal received");
    let _ = tx.send(()); // 全ハンドラに通知
}
```

ハンドラ側でシャットダウンを購読:

```rust
async fn handle_socket_graceful(
    mut socket: WebSocket,
    mut shutdown_rx: broadcast::Receiver<()>,
) {
    loop {
        tokio::select! {
            _ = shutdown_rx.recv() => {
                let _ = socket.send(Message::Close(None)).await;
                break;
            }
            msg = socket.recv() => {
                // 通常処理…
                if msg.is_none() { break; }
            }
        }
    }
}
```

---

## 5 パターン まとめ対比

| # | パターン | 主な効果 | 使うべき状況 |
|---|---|---|---|
| 1 | 最小 WS アップグレード | 動作確認・PoC | プロトタイプ |
| 2 | DashMap + broadcast | ルーム別ブロードキャスト | チャット・通知基盤 |
| 3 | バックプレッシャー制御 | メモリ枯渇防止 | 高負荷・不特定多数接続 |
| 4 | Ping/Pong ハートビート | ゾンビ接続刈り取り | モバイル・NAT 越え |
| 5 | グレースフルシャットダウン | デプロイ時の接続断を最小化 | 本番運用全般 |

---

## ベンチマーク参考値 (公開情報より)

TechEmpower Benchmark (Round 22 公開値) において Rust 系フレームワークは Node.js 比で **平均スループット 3〜8 倍** の数値が報告されています。axum 自体は actix-web と同等クラスで、JSON レスポンスで **150,000〜200,000 req/s** 以上のスループットが計測されています。

WebSocket 接続数については、Linux カーネルのデフォルト `ulimit -n`（1024）を `fs.file-max` と `nofile` 設定で拡張することが必須です:

```bash
# /etc/sysctl.conf
fs.file-max = 1048576
net.core.somaxconn = 65535

# /etc/security/limits.conf
* soft nofile 1048576
* hard nofile 1048576
```

これにより理論上 10 万接続はシングルサーバー (数 GB RAM) でも到達可能な数値です。実測値はアプリケーションのメッセージ頻度・ペイロードサイズに依存します。

---

## まとめ

axum + tokio による WebSocket サーバーの大規模接続対応は、以下の 5 つを組み合わせることで実現できます:

1. **`WebSocketUpgrade` + spawn 分離**: 接続ごとに独立した async task
2. **`DashMap` + `broadcast::channel`**: ロック競合を下げたルーム管理
3. **`timeout` ラッパー**: スロークライアントによるメモリ枯渇を防止
4. **Ping/Pong ハートビート**: ゾンビ接続を定期的に刈り取り
5. **グレースフルシャットダウン**: デプロイ時の接続断を最小化

Rust のゼロコスト抽象化と tokio の非同期ランタイムにより、Go や Node.js に比べてメモリフットプリントを抑えながら高接続数を達成できます。プロトタイプは数百行で書けるので、次の WebSocket 基盤候補として検討してみてください。

---

## 参考リンク

- [axum 公式ドキュメント (docs.rs)](https://docs.rs/axum/latest/axum/)
- [axum WebSocket example (GitHub)](https://github.com/tokio-rs/axum/tree/main/examples/websockets)
- [tokio::sync::broadcast (docs.rs)](https://docs.rs/tokio/latest/tokio/sync/broadcast/index.html)
- [DashMap (docs.rs)](https://docs.rs/dashmap/latest/dashmap/)
- [TechEmpower Web Framework Benchmarks](https://www.techempower.com/benchmarks/)
- [RFC 6455 - The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
