---
title: "Rust の async/await を完全理解する — 5 つの「なぜ？」で紐解くゼロコスト非同期設計"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Rust の async/await は「ゼロコスト抽象化」として実装された状態機械コンパイルである
- ランタイムは言語仕様に含まれず、Tokio / async-std / smol を選択できる
- `Future` トレイトの Poll モデルを理解すると、スタックレスコルーチンの本質が見えてくる
- `Pin<P>` が必要な理由は「自己参照構造体の移動を禁止する」という純粋な安全保証から来ている
- `Send` + `Sync` 境界を正しく扱えば、データ競合なしの並行処理がコンパイル時に保証される

---

## はじめに

非同期プログラミングは現代の Web バックエンドやネットワークサービスには不可欠です。Go の goroutine、JavaScript の Promise、Python の asyncio と並んで、Rust の async/await はここ数年で急速に成熟しました。

しかし「なんとなく動く」レベルで使っているエンジニアが多いのも事実です。本記事では **「なぜそう設計されているのか」** という問いを 5 つ立て、Rust の非同期の本質に迫ります。

対象読者は「Rust の基本的な所有権・借用は理解している。async/await は書いたことあるが、`Pin` や `Waker` でつまずいた」くらいのレベル感を想定しています。

---

## なぜ① ランタイムが標準ライブラリに含まれないのか？

Go や JavaScript には言語組み込みの非同期ランタイムがあります。Go は goroutine スケジューラが runtime パッケージに同梱され、Node.js は libuv がイベントループを担います。

Rust は意図的にこれを **言語仕様から切り離しました**。

```rust
// これだけでは動かない — ランタイムがない
#[tokio::main]
async fn main() {
    println!("Hello, async!");
}
```

`#[tokio::main]` というマクロで初めてスケジューラが起動されます。これを外すと `async fn main()` はコンパイルエラーになります（正確には「async fn を呼ぶ executor が存在しない」）。

### なぜ切り離したのか

Rust の用途はサーバーサイド Web だけではありません。

| 用途 | 最適なランタイム |
|---|---|
| 高スループット Web サーバー | Tokio（マルチスレッド work-stealing） |
| 組み込み/WASM | embassy、smol（ヒープなし対応） |
| コマンドラインツール | async-std（軽量） |
| ゲームエンジン | 独自スケジューラ |

「すべてのユースケースに最適なランタイムは存在しない」という判断から、ランタイムを **コアから外してエコシステムに委ねた** わけです。

`std::future::Future` トレイトだけが標準ライブラリに含まれ、その実行は外部ランタイムが担う——この分離こそ Rust 非同期の根幹です。

---

## なぜ② Future はステートマシンにコンパイルされるのか？

他の言語の非同期モデルと大きく異なるのが、Rust には **スタックを持つコルーチン（スタックフルコルーチン）がない** という点です。

```rust
async fn fetch_user(id: u64) -> User {
    let raw = http_get(format!("/users/{}", id)).await;  // 中断点①
    let profile = db_query(raw.user_id).await;           // 中断点②
    User::from(raw, profile)
}
```

このコードをコンパイラは以下のような enum に変換します（概念的擬似コード）:

```rust
enum FetchUserFuture {
    State0 { id: u64 },
    State1 { raw: RawUser, inner: HttpGetFuture },
    State2 { raw: RawUser, profile: Profile, inner: DbQueryFuture },
    Done,
}

impl Future for FetchUserFuture {
    type Output = User;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<User> {
        match self.get_mut() {
            Self::State0 { id } => {
                // http_get を起動して State1 へ遷移
            }
            Self::State1 { raw, inner } => {
                // inner を poll → Ready なら State2 へ
            }
            Self::State2 { raw, profile, inner } => {
                // inner を poll → Ready なら User を返す
            }
            Self::Done => panic!("polled after completion"),
        }
    }
}
```

### ゼロコスト性

「ゼロコスト抽象化」と言われる所以がここにあります。

- **ヒープ確保なし**: 状態は enum として stack または呼び出し元の Future 内に格納
- **仮想関数テーブルなし**: 具体的な型が静的ディスパッチされる（`Box<dyn Future>` を使わない限り）
- **スタック切り替えなし**: OS スレッドの context switch に比べてコストゼロ

Go の goroutine は **スタックフルコルーチン** であり、1 goroutine あたり初期 8KB 程度のスタック領域を確保します。100 万 goroutine = 約 8GB のスタック。一方 Rust の Future は実際に必要な変数しかサイズを持ちません。

---

## なぜ③ `Pin<P>` が必要なのか？

Rust の非同期で最もつまずきやすいのが `Pin` です。なぜ移動を禁止する必要があるのでしょうか。

### 自己参照構造体の問題

先ほどのステートマシン enum を思い出してください。`State1` には `raw: RawUser` と、`raw` を参照している `inner: HttpGetFuture` が **同時に存在する可能性があります**。

```rust
struct SelfRef {
    data: String,
    ptr: *const String, // data を指すポインタ（自己参照）
}
```

Rust の値は **移動（move）できます**。`let a = b;` で `b` のメモリ上のアドレスが変わります。もし `ptr` が移動前の `data` のアドレスを指していたら、**ダングリングポインタ** になります。

```
移動前:       移動後:
addr 0x100    addr 0x200
┌──────┐       ┌──────┐
│ data │  →→→  │ data │
│ ptr  │       │ ptr  │  ← まだ 0x100 を指している！（壊れた）
└──────┘       └──────┘
```

### Pin が提供する保証

`Pin<P>` は「このポインタの指す値を移動させない」というコンパイル時保証です。

```rust
use std::pin::Pin;

// Pin<Box<T>> は T を heap に固定し、移動を禁止する
let pinned: Pin<Box<MySelfRefFuture>> = Box::pin(my_future);
```

- `Pin<&mut T>` を受け取った関数は `T` を移動できない
- `Unpin` トレイトを実装した型（`i32`、`String` など自己参照しない大半の型）は `Pin` の制約が緩和される
- 自己参照する async-generated Future は自動的に `!Unpin`（Unpin でない）になる

実用上は **`Box::pin()`** や **`tokio::pin!()`** マクロを使えば問題なく動き、`Pin` の低レベルな操作は `unsafe` の世界に閉じ込められます。

---

## なぜ④ `Waker` は callback ではなく trait なのか？

`Future::poll()` は `Ready` または `Pending` を返します。`Pending` を返した Future は「準備できたら再度 poll して」とランタイムに伝える必要があります。その仕組みが `Waker` です。

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

`Context` の中に `Waker` が入っており、`Waker::wake()` を呼ぶと executor がその Future を再度 poll します。

### callback ではなく Waker にした理由

他の非同期モデルでは callback が主流です（`on_ready(|result| { ... })`）。Rust が Waker を採用した理由:

1. **コンポジションが容易**: 複数の Future を束ねる `join!` や `select!` は、子 Future の Waker を自分の Waker でラップするだけでよい
2. **型消去が最小限**: Waker は `RawWaker` + vtable というシンプルな構造で、ランタイム依存を切り離せる
3. **所有権が明確**: callback クロージャは生存期間の追跡が難しいが、Waker は `Clone` + `Send` な値

実際のエコシステムでは、`futures::task::ArcWake` トレイトを使うと Waker の実装が格段に楽になります:

```rust
use futures::task::{ArcWake, waker_ref};
use std::sync::{Arc, Mutex};
use std::task::Context;

struct MyTask {
    future: Mutex<Pin<Box<dyn Future<Output = ()> + Send>>>,
}

impl ArcWake for MyTask {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        // 自分自身をキューに積み直す
        schedule(arc_self.clone());
    }
}
```

---

## なぜ⑤ `Send` 境界がそんなに重要なのか？

マルチスレッドのランタイム（Tokio デフォルト）で async を書いていると、しばしば以下のコンパイルエラーに遭遇します:

```
error[E0277]: `Rc<i32>` cannot be sent between threads safely
   --> src/main.rs:12:5
    |
12  |     tokio::spawn(async {
    |     ^^^^^^^^^^^^ `Rc<i32>` cannot be sent between threads safely
```

Tokio の `spawn` は `F: Future + Send + 'static` を要求します。なぜなら work-stealing スケジューラは **poll するスレッドが poll ごとに変わる可能性がある** からです。

```
スレッド A → Future を poll（State0）
スレッド B → Future を poll（State1）  ← 別スレッドに移った
```

この移動が安全であるためには、Future が保持するすべての値が `Send` でなければなりません。

### よくある落とし穴と解決策

```rust
// NG: Rc は !Send
use std::rc::Rc;
async fn bad() {
    let x = Rc::new(42);
    some_async_fn().await;  // ← この await 点をまたいで Rc を保持している
    println!("{}", x);
}

// OK: Arc は Send
use std::sync::Arc;
async fn good() {
    let x = Arc::new(42);
    some_async_fn().await;
    println!("{}", x);
}

// OK: await より前にドロップ
async fn also_good() {
    let x = Rc::new(42);
    println!("{}", x);
    drop(x);               // ← await の前にドロップ
    some_async_fn().await;
}
```

コンパイラは `async fn` を状態機械に変換する際、**どの変数が await をまたいで生存しているか** を正確に追跡します。`Rc` を await 前にドロップすれば、Future の enum フィールドに入らないので `Send` 境界を満たせます。

---

## 実践: Tokio での基本パターン 4 選

以上の理解を踏まえ、日常的に使う 4 パターンを示します。

### 1. 並行実行 — `tokio::join!`

```rust
use tokio::join;

async fn get_name() -> String { "Alice".into() }
async fn get_age() -> u32 { 30 }

#[tokio::main]
async fn main() {
    let (name, age) = join!(get_name(), get_age());
    println!("{} is {}", name, age);
}
```

`join!` は両方の Future を **同一タスク内で並行** に進めます。`tokio::spawn` と異なりスレッドをまたがりません。

### 2. 競争実行 — `tokio::select!`

```rust
use tokio::{time, select};
use std::time::Duration;

async fn fetch() -> &'static str { "data" }

#[tokio::main]
async fn main() {
    select! {
        result = fetch() => println!("Got: {}", result),
        _ = time::sleep(Duration::from_millis(100)) => println!("Timeout"),
    }
}
```

`select!` は **先に完了した Future の結果だけ**を使い、残りは drop します。

### 3. バックグラウンドタスク — `tokio::spawn`

```rust
use tokio::task::JoinHandle;

async fn background_work() -> u32 { 42 }

#[tokio::main]
async fn main() {
    let handle: JoinHandle<u32> = tokio::spawn(background_work());
    // 他の処理
    let result = handle.await.unwrap();
    println!("Result: {}", result);
}
```

`JoinHandle` は `Future<Output = Result<T, JoinError>>` です。`await` でパニックやキャンセルを検知できます。

### 4. チャネル通信 — `tokio::sync::mpsc`

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<u32>(32);

    tokio::spawn(async move {
        for i in 0..5 {
            tx.send(i).await.unwrap();
        }
    });

    while let Some(val) = rx.recv().await {
        println!("Received: {}", val);
    }
}
```

`mpsc::channel` はバッファ付き非同期チャネルです。`send().await` は **バッファが満杯のとき自動的に yield** します。

---

## Tokio vs async-std vs smol — 何を選ぶか

| | Tokio | async-std | smol |
|---|---|---|---|
| スレッドモデル | work-stealing マルチスレッド | work-stealing マルチスレッド | thread-per-core または シングルスレッド |
| ヒープ使用 | 多め（高機能） | 中程度 | 最小限 |
| エコシステム | 最大（reqwest / axum / tonic 等） | 中程度 | 小さいが growing |
| 組み込み/WASM | △（embassy の方が適切） | △ | ○ |
| 公式ドキュメント | 充実（tokio.rs tutorials） | 充実 | 最小限 |

**2026 年時点での推奨**: Web バックエンドなら **Tokio 一択**。組み込みや WASM は **embassy** または **smol**。学習目的なら **async-std**（std に近い API）。

---

## まとめ

5 つの「なぜ」を振り返ります。

| 問い | 答えの核心 |
|---|---|
| なぜランタイムが標準ライブラリにないのか | 用途多様性・ゼロコスト原則 |
| なぜ Future はステートマシンになるのか | スタックフルコルーチンのヒープ/切り替えコストを避けるため |
| なぜ `Pin` が必要なのか | 自己参照構造体の移動によるダングリングポインタを防ぐため |
| なぜ `Waker` は trait なのか | ランタイム非依存のコンポジションを実現するため |
| なぜ `Send` 境界が厳しいのか | work-stealing スケジューラはスレッドをまたいで poll するため |

Rust の非同期は「難しい」と言われますが、その難しさの多くは **他の言語が暗黙に隠しているコストや危険を Rust が型システムで表面に出している** ことに由来します。一度 Poll モデルを腹落ちさせると、コンパイルエラーのメッセージが「なるほど、そういうことか」と読めるようになります。

---

## 参考リンク

- [The Rust Async Book](https://rust-lang.github.io/async-book/) — async/await の公式ガイド
- [Tokio Tutorial](https://tokio.rs/tokio/tutorial) — Tokio の公式チュートリアル（日本語訳あり）
- [Jon Gjengset "Rust for Rustaceans" Chapter 8](https://nostarch.com/rust-rustaceans) — 非同期の内部実装の最良の解説
- [withoutboats blog: "Why async Rust?"](https://without.boats/blog/why-async-rust/) — 設計判断の背景
- [embassy: async for embedded](https://embassy.dev/) — 組み込み向け async ランタイム

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
