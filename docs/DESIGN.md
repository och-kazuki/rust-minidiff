## 全体方針

* **core（差分計算）** と **render（表示）** を分離する
  → diffのアルゴリズム変えても、表示は壊れにくい
* 差分結果は「編集操作」よりも「表示しやすい列」を持つ
  → `Equal/Add/Remove` の並びを返す（Changeはレンダラ側でまとめてもOK）

---

## 1) 差分データ構造

### 1.1 生の差分（最小）

まずはこれで十分ばい。

```rust
/// 行単位の差分。AとBの行列を比較した結果のストリーム。
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum DiffOp<'a> {
    Equal { a: &'a str, b: &'a str }, // 通常 a==b。前処理で等価扱いになった時もここ
    Add   { b: &'a str },             // Bにのみ存在
    Remove{ a: &'a str },             // Aにのみ存在
}
```

* **参照（`&str`）で持つ**：コピーを増やさんで済む
* 入力は `Vec<String>` を保持しといて、`&str` はそこから借りる

「Change」を入れたい気持ちは分かるけど、初期は `Remove + Add` で表現しておくのが堅い。
レンダリング段階で「連続した Remove と Add をまとめて Changeっぽく見せる」ことができるけんね。

### 1.2 レンダ用に“ハンク”を持つ（context表示するなら）

`--context N` をやるなら、差分列をそのまま出すより **ハンク** にしたほうがラク。

```rust
#[derive(Debug, Clone)]
pub struct Hunk<'a> {
    pub a_start: usize, // 1-based line number
    pub b_start: usize,
    pub a_len: usize,
    pub b_len: usize,
    pub lines: Vec<HunkLine<'a>>,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum HunkLine<'a> {
    Context(&'a str), // ' '
    Add(&'a str),     // '+'
    Remove(&'a str),  // '-'
}
```

* coreは `Vec<DiffOp>` を返すだけでもいい
* render前に **hunk builder**（`Vec<DiffOp>`→`Vec<Hunk>`）を挟むのが綺麗

---

## 2) モジュール分割（おすすめ）

```
src/
  main.rs
  lib.rs
  cli/
    mod.rs
  io/
    mod.rs
  diff/
    mod.rs
    lcs.rs
  model/
    mod.rs
  render/
    mod.rs
    unified.rs
```

### 各責務

* `model`: 差分データ構造（`DiffOp`, `Hunk` など）
* `cli`: `clap`で引数を構造体にする（入出力やロジックに触らない）
* `io`: 読み込み・前処理（EOL正規化、ignore系）
* `diff`: 差分計算（アルゴリズムはここ）

  * `lcs.rs`: 最初はLCSでOK
* `render`: 表示（unified/色/行番号）

  * `unified.rs`: 出力テキストを組み立てる

---

## 3) 公開API（lib.rsの入口）設計

`main.rs` は薄くして、`lib.rs` に「実行ユースケース」を置くとテストしやすい。

```rust
pub struct AppConfig {
    pub path_a: String,
    pub path_b: String,
    pub context: usize,
    pub color: ColorMode,
    pub ignore_eol: bool,
    pub ignore_trailing_space: bool,
    pub ignore_case: bool,
}

pub enum ColorMode { Auto, Always, Never }

pub struct RunResult {
    pub has_diff: bool,
    pub output: String,
}

pub fn run(cfg: AppConfig) -> anyhow::Result<RunResult> {
    // 1) io: read + preprocess -> Vec<String> lines_a/lines_b
    // 2) diff: compute -> Vec<DiffOp>
    // 3) render: ops -> output String
}
```

`main.rs` は `run()` の結果で終了コードを決めるだけ。

* diffなし: `0`
* diffあり: `1`

---

## 4) データの流れ（パイプライン）

1. **Input**: `pathA/pathB` or `-`
2. **Read**: `Vec<String>`（生行）
3. **Normalize/Filter**: 比較用に変換（EOL無視、大小無視など）

   * コツ: 「表示用の元行」と「比較用の正規化行」を分けると綺麗
     例：`struct Line { raw: String, norm: String }`
4. **Diff**: `Vec<DiffOp>`（`&str`参照）
5. **Build hunks**（contextが必要な場合）
6. **Render**: `String`

---

## 5) “前処理”の設計（ここでハマりやすい）

無視系（ignore）のせいで

* 表示は元の文字列
* 比較は正規化文字列

ってしたくなる。そこで、次の形が扱いやすいばい。

```rust
pub struct Line {
    pub raw: String,   // 表示用
    pub key: String,   // 比較用（正規化済み）
}
```

diff計算は `key` 同士で比較、レンダは `raw` を表示。
`DiffOp` には `raw` の `&str` を入れる（keyは内部だけでOK）。

---

## 6) アルゴリズム差し替えできる形（diff::Engine）

後でMyersに替えたくなった時のために、traitで隠すのが吉。

```rust
pub trait DiffEngine {
    fn diff<'a>(&self, a: &'a [Line], b: &'a [Line]) -> Vec<DiffOp<'a>>;
}

pub struct LcsEngine;
```

最初は `LcsEngine` だけ実装で十分。

---

## 7) 最初のマイルストーン（内部設計としての到達点）

* `io::read_lines` が `Vec<Line>` を返す
* `diff::LcsEngine` が `Vec<DiffOp>` を返す
* `render::unified` が `String` を返す
* `run()` が組み立て、`main.rs` が終了コードを返す

