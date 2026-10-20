---
title: "Rust の async/await を完全理解する — 7 つの落とし穴と対処法"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Rust の async/await は「Future を返す糖衣構文」であり、ランタイムがなければ動かない
- `.await` を忘れる・Send 境界が合わない・blocking 呼び出しを混在させる、などの落とし穴がある
- Tokio の `spawn_blocking` と `select!` マクロを正しく使うだけで、大多数の問題は解決できる

---

## はじめに

Rust を書き始めて「async/await を使ってみよう」と思ったとき、コンパイルエラーや意図しない動作に悩んだ経験はないでしょうか。

Rust の非同期処理は **言語コア (Future トレイト) + ランタイム (Tokio など) + エコシステム** の三層構造になっており、それぞれの責任範囲を理解していないと詰まりやすいポイントが多々あります。

本記事では **7 つの典型的な落とし穴** を実際のコード例とともに整理し、「なぜそうなるか」を深掘りします。

---

## 前提知識: Future とは何か

Rust における `Future` トレイトは以下の定義です。

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

ポイントは**「呼び出し側 (ランタイム) が `poll` を呼ぶまで、Future は一切実行されない」**という点です。

```rust
async fn say_hello() {
    println!("hello");
}

fn main() {
    let _fut = say_hello(); // ← ここでは何も表示されない!
}
```

`say_hello()` を呼び出しても、返ってきた `impl Future` を `.await` するか、ランタイムに渡すかしなければ **1 行も実行されません**。これが第一の落とし穴です。

---

## 落とし穴 1: `.await` を忘れる

```rust
#[tokio::main]
async fn main() {
    fetch_data(); // ← .await を忘れると警告が出るだけで値は捨てられる
}

async fn fetch_data() -> String {
    // ...
    "data".to_string()
}
```

Rust コンパイラは `unused_must_use` lint として警告を出しますが、エラーにはなりません。

### 対処

```rust
let result = fetch_data().await;
```

**IDE で `#[must_use]` の警告を全て `deny` にする設定を `lib.rs` に入れておく**と事故を防げます。

```rust
// lib.rs
#![deny(unused_must_use)]
```

---

## 落とし穴 2: async fn 内で blocking 処理を呼ぶ

Tokio の worker thread は非常に少数 (デフォルト: CPU コア数) です。その上で `std::thread::sleep` や重い計算を直接呼ぶと、**全ての非同期タスクが止まります**。

```rust
#[tokio::main]
async fn main() {
    tokio::spawn(async {
        std::thread::sleep(std::time::Duration::from_secs(5)); // 危険!
        println!("done");
    });
}
```

### 対処

`tokio::time::sleep` を使うか、CPU バウンドな処理は `spawn_blocking` に逃がします。

```rust
// I/O 待ちなら tokio のスリープ
tokio::time::sleep(std::time::Duration::from_secs(5)).await;

// CPU バウンド or 外部ブロッキング API
let result = tokio::task::spawn_blocking(|| {
    heavy_computation() // blocking OK
})
.await
.unwrap();
```

`spawn_blocking` は Tokio が管理する**専用のブロッキングスレッドプール**で実行されるため、非同期ワーカーをブロックしません。

---

## 落とし穴 3: `Send` 境界が合わない

`tokio::spawn` でタスクを生成するとき、クロージャは `Send + 'static` を要求します。

```rust
use std::rc::Rc;

#[tokio::main]
async fn main() {
    let data = Rc::new(42); // Rc は Send でない
    tokio::spawn(async move {
        println!("{}", data); // コンパイルエラー!
    });
}
```

```
error[E0277]: `Rc<i32>` cannot be sent between threads safely
```

### 対処

`Rc` → `Arc`、`RefCell` → `Mutex` / `RwLock` に置き換えます。

```rust
use std::sync::Arc;

let data = Arc::new(42);
tokio::spawn(async move {
    println!("{}", data); // OK
});
```

スレッドをまたがない確証があるシングルスレッドユースケースでは `tokio::task::LocalSet` + `spawn_local` を使う方法もあります。

---

## 落とし穴 4: `Mutex` を `.await` をまたいで保持する

```rust
use std::sync::Mutex;
use std::sync::Arc;

async fn increment(counter: Arc<Mutex<u32>>) {
    let mut guard = counter.lock().unwrap();
    some_async_fn().await; // ← ここで guard を保持したままサスペンド → デッドロック or Send エラー
    *guard += 1;
}
```

`std::sync::Mutex` のガードは `Send` ではないため、`.await` ポイントをまたぐと **コンパイルエラー** になります (Tokio の `Send` チェックがある場合)。または実行時デッドロックが起きます。

### 対処

**`tokio::sync::Mutex`** を使います。これは非同期対応の Mutex で、`.await` をまたいでロックを保持できます。

```rust
use tokio::sync::Mutex;
use std::sync::Arc;

async fn increment(counter: Arc<Mutex<u32>>) {
    let mut guard = counter.lock().await; // 非同期ロック
    some_async_fn().await; // OK
    *guard += 1;
}
```

ただし、**ロックのスコープはなるべく短く**するのが原則です。

```rust
// より良い書き方: await の前後でロック範囲を分ける
async fn increment_v2(counter: Arc<tokio::sync::Mutex<u32>>) {
    some_async_fn().await;
    let mut guard = counter.lock().await;
    *guard += 1;
} // guard をここで drop
```

---

## 落とし穴 5: `select!` の公平性とキャンセル安全性

`tokio::select!` は複数の Future を競争させ、最初に完了したものだけを処理します。

```rust
tokio::select! {
    result = fetch_a() => { /* ... */ }
    result = fetch_b() => { /* ... */ }
}
```

**落とし穴 5-A: 公平性**

デフォルトで `select!` は先に列挙したブランチを**優先して poll** します。高負荷時に後ろのブランチが枯死 (starvation) することがあります。

対処: `biased;` を**外す** (デフォルト挙動はランダム擬似選択なので実は公平)、または `biased;` キーワードを明示して意図的に優先順位をつける。

**落とし穴 5-B: キャンセル安全性**

`select!` でレースに負けたブランチはドロップされます。`tokio::io::AsyncReadExt::read_line` のような「途中まで読んだバッファがある」操作は、キャンセルされると**読んだデータが失われます**。

```rust
// NG: read_line はキャンセル安全でない
select! {
    line = reader.read_line(&mut buf) => { /* ... */ }
    _ = shutdown_signal() => { return; }
}
```

対処: ドキュメントで **「cancel safe」** と明記されている API を使うか、`tokio::sync::mpsc` の `recv()` のようにキャンセル安全な操作に置き換えます。  
Tokio 公式ドキュメントには各 API の cancel safety が明記されています。

---

## 落とし穴 6: 無限 `loop` 内で `.await` を忘れる

```rust
async fn worker() {
    loop {
        process_something(); // async でない処理だけを呼んでいる
    }
}
```

これは **他のタスクに制御を渡すポイントがない**ため、Tokio のシングルスレッドランタイムでは他のタスクが全くスケジュールされません。

### 対処

ループ内に少なくとも 1 つの `.await` ポイントを作ります。意図的に yield したい場合は `tokio::task::yield_now()` が使えます。

```rust
async fn worker() {
    loop {
        process_something();
        tokio::task::yield_now().await; // 他タスクにターンを渡す
    }
}
```

---

## 落とし穴 7: ランタイムのネスト

`#[tokio::main]` 内で別の Tokio ランタイムを作ろうとすると**パニックします**。

```rust
#[tokio::main]
async fn main() {
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async { // パニック! "Cannot start a runtime from within a runtime"
        println!("nested");
    });
}
```

### 対処

- テストコードでは `#[tokio::test]` を使う
- ライブラリ内でランタイムを作る必要がある場合は `tokio::runtime::Handle::try_current()` で既存ランタイムを検出してから判断する

```rust
match tokio::runtime::Handle::try_current() {
    Ok(handle) => {
        // 既存のランタイム上で spawn
        handle.spawn(async { /* ... */ });
    }
    Err(_) => {
        // ランタイムがなければ新規作成
        tokio::runtime::Runtime::new()
            .unwrap()
            .block_on(async { /* ... */ });
    }
}
```

---

## まとめ: 7 つの落とし穴 早見表

| # | 落とし穴 | 対処法 |
|---|---|---|
| 1 | `.await` 忘れ | `#![deny(unused_must_use)]` |
| 2 | blocking 処理の混在 | `spawn_blocking` / `tokio::time::sleep` |
| 3 | `Send` 境界エラー | `Rc` → `Arc`、`RefCell` → `Mutex` |
| 4 | `.await` をまたぐ `std::Mutex` | `tokio::sync::Mutex` に変更 |
| 5 | `select!` のキャンセル安全性 | cancel safe な API を選ぶ |
| 6 | `loop` 内に `.await` なし | `yield_now()` で yield ポイントを追加 |
| 7 | ランタイムのネスト | `Handle::try_current()` で検出 |

Rust の非同期処理は**型システムがコンパイル時に多くの問題を弾いてくれる**のが大きな強みです。エラーメッセージを丁寧に読むことで、ほとんどの問題はコンパイル時に解決できます。

非同期 Rust の世界は深いですが、このチェックリストを手元に置いておくと、詰まったときの調査時間を大幅に短縮できるはずです。

---

## 参考リンク

- [Tokio 公式ドキュメント — Select](https://tokio.rs/tokio/tutorial/select)
- [Tokio 公式ドキュメント — Spawning](https://tokio.rs/tokio/tutorial/spawning)
- [Rust async book](https://rust-lang.github.io/async-book/)
- [Tokio cancel safety ドキュメント](https://docs.rs/tokio/latest/tokio/macro.select.html)
- [std::future::Future — Rust 標準ドキュメント](https://doc.rust-lang.org/std/future/trait.Future.html)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします!
