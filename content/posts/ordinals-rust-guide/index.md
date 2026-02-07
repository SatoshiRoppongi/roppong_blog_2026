---
title: "Ordinalsプロジェクトから学ぶRustプログラミング完全ガイド"
date: 2026-02-07
draft: false
description: "Bitcoin OrdinalsのRust実装を題材に、Rustプログラミングをゼロから学び、オープンソースへコントリビュートできるレベルまで到達するための完全ガイド"
summary: "Ordinals/ordプロジェクトのRust実装を解説しながら、Rustの言語仕様から実践的なOSS開発まで体系的に学べる超大作ガイド"
tags: ["Rust", "Bitcoin", "Ordinals", "オープンソース", "プログラミング"]
categories: ["技術"]
showToc: true
---

## Part 1: 導入

---

## 1. イントロダクション

### この記事の目的と学習ゴール

この記事は、Bitcoin OrdinalsプロジェクトのRust実装である[ord](https://github.com/ordinals/ord)を題材に、Rustプログラミングをゼロから学び、最終的にはオープンソースプロジェクトにコントリビュートできるレベルを目指す完全ガイドです。

この記事を読み終えると、以下のことができるようになります。

1. **Rustの基礎から応用まで習得できる** - 変数宣言から並行処理まで、実践的なコードを通じて学ぶ
2. **Ordinal理論とInscriptionの仕組みを深く理解できる** - Bitcoinの最小単位satoshiへの番号付けと、NFTの仕組み
3. **ordプロジェクトにコントリビュートできる状態になる** - コードを読み、テストを書き、PRを作成できる

### ordプロジェクトとは

ordは、Casey Rodarmor氏が開発したBitcoin Ordinalsプロトコルのリファレンス実装です。主に以下の3つの機能を提供します。

```
┌─────────────────────────────────────────────────────────────┐
│                        ord                                   │
├─────────────────┬─────────────────┬─────────────────────────┤
│   Bitcoin Index │  Block Explorer │        Wallet           │
│                 │                 │                         │
│ Ordinal番号の   │ Inscription/Rune│ Ordinalsの送受信と      │
│ トラッキング    │ の閲覧          │ Inscriptionの作成       │
└─────────────────┴─────────────────┴─────────────────────────┘
```

- **Bitcoin Index**: 各satoshiにOrdinal番号を付与し、その移動を追跡
- **Block Explorer**: InscriptionやRunesをブラウザで閲覧できるWebインターフェース
- **Wallet**: Ordinalsの送受信、Inscriptionの作成が可能なウォレット機能

### 対象読者

この記事は以下のような方を想定しています。

- TypeScript/JavaScript、Python等のプログラミング経験がある
- ブロックチェーンの基本概念（ブロック、トランザクション、UTXO）は理解している
- Rustは未経験、または入門レベル

### 環境構築

ordプロジェクトをビルドし、開発を行うための環境を構築しましょう。

#### 1. Rustのインストール

```bash
# rustupをインストール
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# パスを通す
source $HOME/.cargo/env

# バージョン確認
rustc --version
cargo --version
```

#### 2. ordリポジトリのクローン

```bash
git clone https://github.com/ordinals/ord.git
cd ord
```

#### 3. ビルドと実行

```bash
# デバッグビルド
cargo build

# リリースビルド（最適化あり）
cargo build --release

# テストの実行
cargo test
```

#### 4. Bitcoin Core（参考情報）

ordを実際に動作させるにはBitcoin Coreが必要ですが、この記事ではコードリーディングと開発に焦点を当てるため、Bitcoin Coreのセットアップは省略します。詳細は[公式ドキュメント](https://docs.ordinals.com)を参照してください。

---

## 2. Rustの基礎文法とordのエントリーポイント

### 変数宣言: let と mut

Rustでは変数は**デフォルトで不変（immutable）**です。これはTypeScriptの `const` に似ていますが、Rustではこれがデフォルトの挙動です。

```rust
// 不変の変数（再代入不可）
let x = 5;
// x = 6;  // コンパイルエラー！

// 可変の変数（再代入可能）
let mut y = 5;
y = 6;  // OK
```

なぜデフォルトで不変なのでしょうか？ それは**バグを防ぐため**です。意図しない変更が起きないことをコンパイラが保証してくれます。

TypeScriptとの比較:
| TypeScript | Rust |
|------------|------|
| `const x = 5` | `let x = 5` |
| `let x = 5` | `let mut x = 5` |

### 関数定義

Rustの関数は `fn` キーワードで定義します。

```rust
// 引数と戻り値の型は必須
fn add(a: i32, b: i32) -> i32 {
    a + b  // セミコロンなしで式を返す
}

// 戻り値がない場合（Unit型 ()を返す）
fn greet(name: &str) {
    println!("Hello, {}!", name);
}
```

重要なポイント:
- 引数の型注釈は**必須**
- 戻り値の型は `->` の後に記述
- 関数の最後の式は `return` なしで返せる（セミコロンなし）

### main関数とordのエントリーポイント

すべてのRustプログラムは `main` 関数から始まります。ordのエントリーポイントを見てみましょう。

**src/lib.rs:258-331**
```rust
pub fn main() {
    env_logger::init();

    ctrlc::set_handler(move || {
        if SHUTTING_DOWN.fetch_or(true, atomic::Ordering::Relaxed) {
            process::exit(1);
        }

        eprintln!("Shutting down gracefully. Press <CTRL-C> again to shutdown immediately.");

        LISTENERS
            .lock()
            .unwrap()
            .iter()
            .for_each(|handle| handle.graceful_shutdown(Some(Duration::from_millis(100))));

        gracefully_shut_down_indexer();
    })
    .expect("Error setting <CTRL-C> handler");

    let args = Arguments::parse();

    let format = args.options.format;

    match args.run() {
        Err(err) => {
            eprintln!("error: {err}");
            // エラーハンドリング...
            process::exit(1);
        }
        Ok(output) => {
            if let Some(output) = output {
                output.print(format.unwrap_or_default());
            }
            gracefully_shut_down_indexer();
        }
    }
}
```

この短いコードの中に、Rustの重要な概念がたくさん詰まっています。

1. **ロガーの初期化**: `env_logger::init()` で環境変数ベースのロガーを初期化
2. **Ctrl+Cハンドラ**: `ctrlc::set_handler` でシグナルハンドラを設定
3. **コマンドライン引数の解析**: `Arguments::parse()` でCLI引数をパース
4. **エラーハンドリング**: `match` 式で `Result` 型を処理

### match式

`match` はRustの最も強力な制御フロー構文の一つです。TypeScriptの `switch` に似ていますが、はるかに強力です。

```rust
match args.run() {
    Err(err) => {
        eprintln!("error: {err}");
        process::exit(1);
    }
    Ok(output) => {
        if let Some(output) = output {
            output.print(format.unwrap_or_default());
        }
    }
}
```

`match` の特徴:
- **網羅性チェック**: すべてのパターンをカバーしないとコンパイルエラー
- **パターンマッチング**: 値の構造を分解しながらマッチング
- **式として使用可能**: 結果を変数に代入できる

### if let構文

`if let` は、1つのパターンだけをマッチさせたい場合に便利です。

```rust
// matchを使う場合
match output {
    Some(value) => println!("{}", value),
    None => {},  // 何もしないケースも書く必要がある
}

// if letを使う場合（より簡潔）
if let Some(value) = output {
    println!("{}", value);
}
```

`if let Some(output) = output` は「`output` が `Some` の場合、中身を `output` として取り出す」という意味です。シャドーイング（同名の変数で上書き）が使われています。

### モジュールのインポート

ordの `src/lib.rs` の冒頭を見ると、多くの `use` 文があります。

```rust
use {
    self::{
        arguments::Arguments,
        blocktime::Blocktime,
        // ...
    },
    anyhow::{Context, Error, anyhow, bail, ensure},
    bitcoin::{
        Amount, Block, KnownHrp, Network, OutPoint, // ...
    },
    std::{
        collections::{BTreeMap, BTreeSet, HashSet},
        env,
        // ...
    },
};
```

Rustでは:
- `use` でモジュールや型をスコープに持ち込む
- `self::` は現在のモジュール内の項目を指す
- `{}` で複数の項目をまとめてインポート
- `::` はパスの区切り（TypeScriptの `/` や `.` に相当）

---

## Part 2: データモデリング

---

## 3. 構造体とデータモデリング - Satoshiの表現

### Ordinal理論の基礎

Ordinal理論は、Bitcoinの最小単位であるsatoshi（sat）に一意の番号を付与する方法論です。

```
1 BTC = 100,000,000 satoshi

最初のブロック（ジェネシスブロック）の報酬:
  sat 0 ~ sat 4,999,999,999 (50億sat = 50 BTC)

2番目のブロックの報酬:
  sat 5,000,000,000 ~ sat 9,999,999,999

...以降、マイニングされた順に番号が付与される
```

Bitcoinの総供給量は約2100万BTCなので、最大で約2.1京（2,099,999,997,690,000）個のsatoshiが存在します。

### struct定義とタプル構造体

Rustの構造体（struct）は、関連するデータをまとめる型です。ordでは `Sat` 型がsatoshiを表現します。

**crates/ordinals/src/sat.rs:1-10**
```rust
use {super::*, std::num::ParseFloatError};

#[derive(Copy, Clone, Eq, PartialEq, Debug, Display, Ord, PartialOrd, Deserialize, Serialize)]
#[serde(transparent)]
pub struct Sat(pub u64);

impl Sat {
    pub const LAST: Self = Self(Self::SUPPLY - 1);
    pub const SUPPLY: u64 = 2099999997690000;
    // ...
}
```

`Sat(pub u64)` は**タプル構造体**と呼ばれる形式です。これは**Newtype Pattern**（新しい型パターン）の典型例です。

#### Newtype Patternとは？

`u64` を直接使う代わりに `Sat` という型でラップすることで:

1. **型安全性**: `Sat` と他の `u64` 値を間違えて混ぜることを防ぐ
2. **意味の明確化**: 「これはsatoshi番号である」ことを型で表現
3. **専用メソッドの追加**: `Sat` 固有の操作を定義できる

TypeScriptでの類似パターン（ただし実行時チェックなし）:
```typescript
type Sat = number & { readonly __brand: unique symbol };
```

### #[derive(...)]マクロ

`#[derive(...)]` は、コンパイラに自動実装を依頼するマクロです。

```rust
#[derive(Copy, Clone, Eq, PartialEq, Debug, Display, Ord, PartialOrd, Deserialize, Serialize)]
pub struct Sat(pub u64);
```

各トレイトの意味:

| トレイト | 機能 | TypeScript相当 |
|---------|------|----------------|
| `Copy` | ビット単位のコピーが可能 | (プリミティブの挙動) |
| `Clone` | `.clone()` でコピー作成 | (スプレッド演算子) |
| `Eq`, `PartialEq` | `==` で比較可能 | `===` |
| `Debug` | `{:?}` でデバッグ出力 | `console.log` |
| `Display` | `{}` でフォーマット | `.toString()` |
| `Ord`, `PartialOrd` | `<`, `>` で比較可能 | 比較演算子 |
| `Serialize`, `Deserialize` | JSON等との相互変換 | JSON.stringify/parse |

### 定数の定義

```rust
impl Sat {
    pub const LAST: Self = Self(Self::SUPPLY - 1);
    pub const SUPPLY: u64 = 2099999997690000;
}
```

- `const`: コンパイル時に値が決まる定数
- `Self`: その `impl` ブロックの型自身（ここでは `Sat`）
- `SUPPLY`: Bitcoinで生成される最大のsatoshi数

約2100万BTCは `2099999997690000` satoshi です。なぜ `2100000000000000`（= 21,000,000 × 100,000,000）ではないのでしょうか？

これはBitcoinの半減期の仕組みによります。

```
報酬は4年ごとに半減:
  Epoch 0: 50 BTC/ブロック × 210,000ブロック = 10,500,000 BTC
  Epoch 1: 25 BTC/ブロック × 210,000ブロック =  5,250,000 BTC
  Epoch 2: 12.5 BTC/ブロック × 210,000ブロック = 2,625,000 BTC
  ...
  合計: 20,999,999.9769 BTC (= 2,099,999,997,690,000 sat)
```

---

## 4. 実装ブロックとメソッド - Satの計算ロジック

### implブロック

Rustでは、構造体のメソッドは `impl` ブロックで定義します。

```rust
impl Sat {
    // メソッド定義
}
```

`impl` ブロックは複数定義可能で、関連する機能ごとに分けることもできます。

### selfとSelf

- `self`: メソッドのレシーバ（インスタンス自身）
- `Self`: `impl` ブロックの型（ここでは `Sat`）

```rust
impl Sat {
    pub fn n(self) -> u64 {
        self.0  // タプル構造体の中身にアクセス
    }

    pub fn height(self) -> Height {
        self.epoch().starting_height()
            + u32::try_from(self.epoch_position() / self.epoch().subsidy()).unwrap()
    }
}
```

`self` を値で受け取る（`&self` ではなく `self`）のは、`Sat` が `Copy` トレイトを実装しているからです。`Copy` 型は自動的にコピーされるため、所有権を気にする必要がありません。

### Epoch、Height、Degreeの概念

ordでは、satoshiを様々な視点から表現するための概念があります。

```
┌─────────────────────────────────────────────────────────────────┐
│                       Sat (satoshi番号)                          │
│                           例: 0                                  │
├─────────────────────────────────────────────────────────────────┤
│  Height: 0         │ ブロック高                                   │
│  Epoch: 0          │ 半減期（0 = 最初の4年間）                      │
│  Period: 0         │ 難易度調整期間（2016ブロックごと）              │
│  Degree: 0°0′0″0‴  │ サイクル°エポックオフセット′周期オフセット″位置‴│
└─────────────────────────────────────────────────────────────────┘
```

**crates/ordinals/src/sat.rs:18-27**
```rust
pub fn height(self) -> Height {
    self.epoch().starting_height()
        + u32::try_from(self.epoch_position() / self.epoch().subsidy()).unwrap()
}

pub fn cycle(self) -> u32 {
    Epoch::from(self).0 / CYCLE_EPOCHS
}
```

`height` メソッドの計算ロジック:
1. そのsatが属するエポックの開始ブロック高を取得
2. エポック内での位置（epoch_position）をそのエポックの報酬で割って、ブロック数を算出
3. 両者を足し合わせる

### ビット演算

**crates/ordinals/src/epoch.rs:45-51**
```rust
pub fn subsidy(self) -> u64 {
    if self < Self::FIRST_POST_SUBSIDY {
        (50 * COIN_VALUE) >> self.0
    } else {
        0
    }
}
```

`>>` は右シフト演算子です。ビットを右にシフトすることで2で割る効果があります。

```rust
50 * COIN_VALUE >> 0  // = 5,000,000,000 (50 BTC)
50 * COIN_VALUE >> 1  // = 2,500,000,000 (25 BTC) - 1回目の半減
50 * COIN_VALUE >> 2  // = 1,250,000,000 (12.5 BTC) - 2回目の半減
```

この実装は、半減期の計算を非常にエレガントに表現しています。

### メソッドチェーン

Rustでもメソッドチェーンは自然に書けます。

```rust
pub fn rarity(self) -> Rarity {
    self.into()  // Sat -> Rarity への変換
}

// 使用例
let sat = Sat(0);
let rarity = sat.rarity();  // Mythic
```

---

## 5. 列挙型（Enum）とパターンマッチング - Rarityの表現

### enum定義

Rustの `enum` は、TypeScriptの `enum` よりもはるかに強力です。各バリアントがデータを持つことができます。

**crates/ordinals/src/rarity.rs:1-14**
```rust
#[derive(
    Clone, Copy, Debug, DeserializeFromStr, Eq, Hash, PartialEq, PartialOrd, SerializeDisplay,
)]
pub enum Rarity {
    Common,
    Uncommon,
    Rare,
    Epic,
    Legendary,
    Mythic,
}
```

### Ordinalのレアリティ分類

Ordinal理論では、satoshiの希少性を以下のように分類します。

```
┌─────────────┬──────────────────────────────────────┬─────────────────┐
│ レアリティ   │ 条件                                  │ 総数            │
├─────────────┼──────────────────────────────────────┼─────────────────┤
│ Common      │ ブロック最初のsat以外                  │ 2,099,999,990,760,000 │
│ Uncommon    │ 各ブロックの最初のsat                  │ 6,926,535       │
│ Rare        │ 難易度調整期間の最初のsat              │ 3,432           │
│ Epic        │ 半減期の最初のsat                      │ 27              │
│ Legendary   │ サイクルの最初のsat                    │ 5               │
│ Mythic      │ ジェネシスブロックの最初のsat（sat 0） │ 1               │
└─────────────┴──────────────────────────────────────┴─────────────────┘
```

**crates/ordinals/src/rarity.rs:25-35**
```rust
pub fn supply(self) -> u64 {
    match self {
        Self::Common => 2_099_999_990_760_000,
        Self::Uncommon => 6_926_535,
        Self::Rare => 3_432,
        Self::Epic => 27,
        Self::Legendary => 5,
        Self::Mythic => 1,
    }
}
```

### パターンマッチングの詳細

**crates/ordinals/src/rarity.rs:76-99**
```rust
impl From<Sat> for Rarity {
    fn from(sat: Sat) -> Self {
        let Degree {
            hour,
            minute,
            second,
            third,
        } = sat.degree();

        if hour == 0 && minute == 0 && second == 0 && third == 0 {
            Self::Mythic
        } else if minute == 0 && second == 0 && third == 0 {
            Self::Legendary
        } else if minute == 0 && third == 0 {
            Self::Epic
        } else if second == 0 && third == 0 {
            Self::Rare
        } else if third == 0 {
            Self::Uncommon
        } else {
            Self::Common
        }
    }
}
```

この実装では、`Degree` 構造体を分解（デストラクチャリング）しています。

```rust
let Degree { hour, minute, second, third } = sat.degree();
```

これはTypeScriptの分割代入に相当します:
```typescript
const { hour, minute, second, third } = sat.degree();
```

### enumのメソッド

enumにもメソッドを定義できます。

**crates/ordinals/src/rarity.rs:59-74**
```rust
impl Display for Rarity {
    fn fmt(&self, f: &mut Formatter) -> fmt::Result {
        write!(
            f,
            "{}",
            match self {
                Self::Common => "common",
                Self::Uncommon => "uncommon",
                Self::Rare => "rare",
                Self::Epic => "epic",
                Self::Legendary => "legendary",
                Self::Mythic => "mythic",
            }
        )
    }
}
```

`match` 式の結果を `write!` マクロに直接渡しています。`match` は式なので、値を返すことができます。

---

## 6. トレイトと型変換 - FromとIntoの実装

### トレイトの概念

トレイトは、TypeScriptの `interface` に似た概念ですが、より強力です。

```
TypeScript Interface vs Rust Trait
┌─────────────────────────────────────────────────────────────────┐
│ TypeScript:                                                      │
│   interface Printable {                                          │
│     print(): void;                                               │
│   }                                                              │
│   class Document implements Printable { ... }                    │
├─────────────────────────────────────────────────────────────────┤
│ Rust:                                                            │
│   trait Printable {                                              │
│     fn print(&self);                                             │
│   }                                                              │
│   impl Printable for Document { ... }                            │
└─────────────────────────────────────────────────────────────────┘
```

重要な違い:
- Rustのトレイトは**既存の型に後から実装できる**
- デフォルト実装を持てる
- 関連型やジェネリクスを使える

### From/Intoトレイト

`From` と `Into` は型変換のための標準トレイトです。

```rust
// Fromを実装すると
impl From<Sat> for Epoch { ... }

// 以下のように使える
let epoch = Epoch::from(sat);  // From
let epoch: Epoch = sat.into();  // Into（自動的に実装される）
```

**crates/ordinals/src/epoch.rs:70-142**
```rust
impl From<Sat> for Epoch {
    fn from(sat: Sat) -> Self {
        if sat < Self::STARTING_SATS[1] {
            Epoch(0)
        } else if sat < Self::STARTING_SATS[2] {
            Epoch(1)
        } else if sat < Self::STARTING_SATS[3] {
            Epoch(2)
        }
        // ... 各エポックの境界値でチェック
        else {
            Epoch(33)
        }
    }
}
```

なぜ `binary_search` ではなく手動で比較しているのでしょうか？ これはパフォーマンス最適化の一例で、エポック数が固定（34個）であり、線形探索でも十分高速だからです。

### FromStrトレイト

文字列からのパース用トレイトです。

**crates/ordinals/src/rarity.rs:101-115**
```rust
impl FromStr for Rarity {
    type Err = String;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        match s {
            "common" => Ok(Self::Common),
            "uncommon" => Ok(Self::Uncommon),
            "rare" => Ok(Self::Rare),
            "epic" => Ok(Self::Epic),
            "legendary" => Ok(Self::Legendary),
            "mythic" => Ok(Self::Mythic),
            _ => Err(format!("invalid rarity `{s}`")),
        }
    }
}
```

`type Err = String` は**関連型**です。この実装により:

```rust
let rarity: Rarity = "rare".parse().unwrap();  // Ok(Rarity::Rare)
let rarity: Rarity = "invalid".parse();        // Err("invalid rarity `invalid`")
```

### Displayトレイト

`Display` トレイトを実装すると、`{}` フォーマットや `to_string()` が使えます。

```rust
impl Display for Rarity {
    fn fmt(&self, f: &mut Formatter) -> fmt::Result {
        write!(f, "{}", match self {
            Self::Common => "common",
            // ...
        })
    }
}

// 使用例
let rarity = Rarity::Rare;
println!("{}", rarity);      // "rare"
let s = rarity.to_string();  // "rare"
```

### TryFromトレイト

`From` が常に成功する変換なのに対し、`TryFrom` は失敗する可能性のある変換用です。

**crates/ordinals/src/rarity.rs:43-57**
```rust
impl TryFrom<u8> for Rarity {
    type Error = u8;

    fn try_from(rarity: u8) -> Result<Self, u8> {
        match rarity {
            0 => Ok(Self::Common),
            1 => Ok(Self::Uncommon),
            2 => Ok(Self::Rare),
            3 => Ok(Self::Epic),
            4 => Ok(Self::Legendary),
            5 => Ok(Self::Mythic),
            n => Err(n),  // 無効な値はエラー
        }
    }
}
```

使用例:
```rust
let rarity = Rarity::try_from(2)?;  // Ok(Rarity::Rare)
let rarity = Rarity::try_from(99);  // Err(99)
```

---

## Part 3: エラー処理と所有権

---

## 7. Result/Optionによるエラーハンドリング

### ResultとOption

Rustにはnullがありません。代わりに以下の2つの型を使います。

```
┌─────────────────────────────────────────────────────────────────┐
│ Option<T>                                                        │
│   - Some(T): 値がある                                             │
│   - None: 値がない                                                │
│   用途: nullableな値、検索結果など                                 │
├─────────────────────────────────────────────────────────────────┤
│ Result<T, E>                                                     │
│   - Ok(T): 成功、値はT                                            │
│   - Err(E): 失敗、エラーはE                                        │
│   用途: 失敗する可能性のある操作                                    │
└─────────────────────────────────────────────────────────────────┘
```

TypeScriptとの比較:
```typescript
// TypeScript
function find(id: number): User | null { ... }
function parse(s: string): User { throw new Error(...); }

// Rust
fn find(id: u32) -> Option<User> { ... }
fn parse(s: &str) -> Result<User, ParseError> { ... }
```

### ? 演算子

`?` 演算子は、エラー処理を簡潔に書くための糖衣構文です。

```rust
// ?を使わない場合
fn read_file() -> Result<String, io::Error> {
    let file = match File::open("hello.txt") {
        Ok(f) => f,
        Err(e) => return Err(e),
    };
    // ...
}

// ?を使う場合
fn read_file() -> Result<String, io::Error> {
    let file = File::open("hello.txt")?;  // エラーなら即座にreturn Err
    // ...
}
```

### varintモジュールの実例

可変長整数（varint）のエンコード/デコードを見てみましょう。

**crates/ordinals/src/varint.rs:1-46**
```rust
pub fn encode_to_vec(mut n: u128, v: &mut Vec<u8>) {
    while n >> 7 > 0 {
        v.push(n.to_le_bytes()[0] | 0b1000_0000);
        n >>= 7;
    }
    v.push(n.to_le_bytes()[0]);
}

pub fn decode(buffer: &[u8]) -> Result<(u128, usize), Error> {
    let mut n = 0u128;

    for (i, &byte) in buffer.iter().enumerate() {
        if i > 18 {
            return Err(Error::Overlong);
        }

        let value = u128::from(byte) & 0b0111_1111;

        if i == 18 && value & 0b0111_1100 != 0 {
            return Err(Error::Overflow);
        }

        n |= value << (7 * i);

        if byte & 0b1000_0000 == 0 {
            return Ok((n, i + 1));
        }
    }

    Err(Error::Unterminated)
}
```

`decode` 関数の戻り値 `Result<(u128, usize), Error>` は:
- 成功時: デコードされた値と、消費したバイト数のタプル
- 失敗時: エラーの種類

### エラー型の定義

**crates/ordinals/src/varint.rs:41-58**
```rust
#[derive(PartialEq, Debug)]
pub enum Error {
    Overlong,     // 長すぎる
    Overflow,     // オーバーフロー
    Unterminated, // 終端がない
}

impl Display for Error {
    fn fmt(&self, f: &mut Formatter) -> fmt::Result {
        match self {
            Self::Overlong => write!(f, "too long"),
            Self::Overflow => write!(f, "overflow"),
            Self::Unterminated => write!(f, "unterminated"),
        }
    }
}

impl std::error::Error for Error {}
```

エラー型は通常:
1. `enum` で定義
2. `Display` を実装してエラーメッセージを提供
3. `std::error::Error` を実装して標準エラートレイトに準拠

### map、and_then、unwrap_or

`Option` と `Result` には便利なメソッドがあります。

```rust
// map: 値がある場合に変換
let x: Option<i32> = Some(5);
let y: Option<String> = x.map(|n| n.to_string());  // Some("5")

// and_then: 値がある場合に別のOption/Resultを返す関数を適用
let x: Option<&str> = Some("42");
let y: Option<i32> = x.and_then(|s| s.parse().ok());  // Some(42)

// unwrap_or: 値がない場合のデフォルト値
let x: Option<i32> = None;
let y: i32 = x.unwrap_or(0);  // 0
```

ordでの実例:

**src/inscriptions/inscription.rs:259-278**
```rust
pub fn pointer(&self) -> Option<u64> {
    let value = self.pointer.as_ref()?;  // Noneなら早期リターン

    if value.iter().skip(8).copied().any(|byte| byte != 0) {
        return None;
    }

    let pointer = [
        value.first().copied().unwrap_or(0),
        value.get(1).copied().unwrap_or(0),
        value.get(2).copied().unwrap_or(0),
        // ...
    ];

    Some(u64::from_le_bytes(pointer))
}
```

---

## 8. 所有権とライフタイム - Inscriptionの実装

### 所有権の3ルール

Rustの最も特徴的な概念である所有権システムには、3つの基本ルールがあります。

```
┌─────────────────────────────────────────────────────────────────┐
│ ルール1: 各値は1つの変数（オーナー）に所有される                    │
│ ルール2: 値は同時に1つのオーナーしか持てない                        │
│ ルール3: オーナーがスコープを抜けると、値は破棄される                │
└─────────────────────────────────────────────────────────────────┘
```

```rust
let s1 = String::from("hello");
let s2 = s1;  // 所有権がs1からs2に移動（move）
// println!("{}", s1);  // コンパイルエラー！s1はもう使えない
println!("{}", s2);  // OK
```

なぜこのようなルールがあるのでしょうか？ **メモリ安全性をコンパイル時に保証するため**です。これにより、ダングリングポインタやデータ競合を防ぎます。

### 借用（&T と &mut T）

所有権を移動せずに値を参照するには**借用**を使います。

```rust
// 不変の借用（&T）- 複数同時に可能
let s = String::from("hello");
let r1 = &s;  // 借用
let r2 = &s;  // 複数OK
println!("{}, {}", r1, r2);

// 可変の借用（&mut T）- 1つだけ
let mut s = String::from("hello");
let r = &mut s;
r.push_str(", world");
println!("{}", r);
```

重要なルール:
- 不変の借用は同時に複数可能
- 可変の借用は同時に1つだけ
- 不変と可変の借用は同時に存在できない

### Inscription構造体

**src/inscriptions/inscription.rs:15-31**
```rust
#[derive(Debug, PartialEq, Clone, Serialize, Deserialize, Eq, Default)]
pub struct Inscription {
    pub body: Option<Vec<u8>>,
    pub content_encoding: Option<Vec<u8>>,
    pub content_type: Option<Vec<u8>>,
    pub delegate: Option<Vec<u8>>,
    pub duplicate_field: bool,
    pub incomplete_field: bool,
    pub metadata: Option<Vec<u8>>,
    pub metaprotocol: Option<Vec<u8>>,
    pub parents: Vec<Vec<u8>>,
    pub pointer: Option<Vec<u8>>,
    pub properties: Option<Vec<u8>>,
    pub property_encoding: Option<Vec<u8>>,
    pub rune: Option<Vec<u8>>,
    pub unrecognized_even_field: bool,
}
```

この構造体は多くの `Option<Vec<u8>>` フィールドを持っています。これは:
- `Option`: フィールドが存在しない可能性がある
- `Vec<u8>`: 可変長のバイト列

### 借用を使ったメソッド

**src/inscriptions/inscription.rs:219-232**
```rust
pub fn body(&self) -> Option<&[u8]> {
    Some(self.body.as_ref()?)
}

pub fn into_body(self) -> Option<Vec<u8>> {
    self.body
}

pub fn content_length(&self) -> Option<usize> {
    Some(self.body()?.len())
}
```

- `&self`: 構造体を借用してメソッドを呼ぶ
- `self`: 構造体の所有権を取得（ムーブ）

`body()` は `Option<&[u8]>` を返します。これは:
- 元の `Vec<u8>` をコピーせずに参照を返す
- 呼び出し元は借用として受け取る

`into_body()` は `Option<Vec<u8>>` を返し、所有権を移動します。

### ライフタイム注釈 'a

ライフタイムは、参照が有効な期間を示します。

```rust
// ライフタイム注釈なし（単純なケース）
fn first_word(s: &str) -> &str {
    &s[..s.find(' ').unwrap_or(s.len())]
}

// ライフタイム注釈あり（複雑なケース）
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

`'a` は「この参照は少なくとも `'a` の期間は有効」という意味です。

### Cow<'a, T>

`Cow`（Clone on Write）は、借用か所有のどちらかを保持できるスマートポインタです。

**src/inscriptions/inscription.rs:318-355**
```rust
fn properties_cbor(&self) -> Option<Cow<[u8]>> {
    let value = self.properties.as_deref()?;

    if let Some(encoding) = &self.property_encoding {
        if encoding != BROTLI.as_bytes() {
            return None;
        }

        // 圧縮データを展開 → 新しいVecを作成
        let mut value = Vec::new();
        // ... decompression ...
        Some(Cow::Owned(value))  // 所有
    } else {
        Some(Cow::Borrowed(value))  // 借用
    }
}
```

`Cow<[u8]>` は:
- `Cow::Borrowed(&[u8])`: 既存データへの参照
- `Cow::Owned(Vec<u8>)`: 新しく作成したデータ

これにより、必要な時だけメモリ確保を行う効率的な実装が可能になります。

---

## Part 4: プロジェクト構造

---

## 9. モジュールシステムとプロジェクト構造

### mod と use

Rustのモジュールシステムは、コードを整理するための仕組みです。

**src/lib.rs:116-144**
```rust
pub mod api;
pub mod arguments;
mod blocktime;
pub mod chain;
pub mod decimal;
mod deserialize_from_str;
mod error;
mod fee_rate;
mod fund_raw_transaction;
pub mod index;
mod inscriptions;
// ...
```

- `mod name;`: `name.rs` または `name/mod.rs` をモジュールとして読み込む
- `pub mod`: 外部から見えるモジュール
- `mod`（pubなし）: クレート内部でのみ使用

### pub と可視性

Rustの可視性システム:

```rust
// 完全にプライベート（デフォルト）
fn private_fn() {}

// 同じクレート内で公開
pub(crate) fn crate_fn() {}

// 親モジュールまで公開
pub(super) fn super_fn() {}

// 完全に公開
pub fn public_fn() {}
```

**src/lib.rs:98-107**
```rust
pub use self::{
    chain::Chain,
    fee_rate::FeeRate,
    index::{Index, RuneEntry},
    inscriptions::{Envelope, Inscription, InscriptionId, ParsedEnvelope, RawEnvelope},
    object::Object,
    options::Options,
    properties::{Attributes, Item, Properties, Trait, Traits},
    wallet::transaction_builder::{Target, TransactionBuilder},
};
```

`pub use` は**再エクスポート**です。内部の型を外部からより簡単にアクセスできるようにします。

### ワークスペースとcrates/

ordプロジェクトはCargoワークスペースを使用しています。

**Cargo.toml:19-20**
```toml
[workspace]
members = [".", "crates/*"]
```

```
ord/
├── Cargo.toml          # ワークスペースルート
├── src/                # メインクレート
│   └── lib.rs
├── crates/             # サブクレート
│   ├── ordinals/       # Ordinal理論の実装
│   │   └── src/
│   │       ├── sat.rs
│   │       ├── rarity.rs
│   │       └── ...
│   └── mockcore/       # テスト用モック
└── tests/              # 統合テスト
```

サブクレートを使う利点:
- **コンパイル時間の短縮**: 変更のあるクレートだけ再コンパイル
- **明確な境界**: 依存関係を制限できる
- **再利用性**: 他のプロジェクトで使用可能

### Cargo.tomlの読み方

**Cargo.toml:1-28**
```toml
[package]
name = "ord"
description = "◉ Ordinal wallet and block explorer"
version = "0.24.2"
autotests = false
autobins = false

authors.workspace = true
edition.workspace = true
homepage.workspace = true
license.workspace = true
repository.workspace = true
rust-version.workspace = true
```

ワークスペース設定の継承:
```toml
[workspace.package]
authors = ["The Ord Maintainers"]
edition = "2024"
homepage = "https://github.com/ordinals/ord"
license = "CC0-1.0"
repository = "https://github.com/ordinals/ord"
rust-version = "1.89.0"
```

---

## 10. ジェネリクスと型パラメータ

### ジェネリック構造体<T>

ジェネリクスは、型をパラメータ化する機能です。

**src/inscriptions/envelope.rs:20-27**
```rust
#[derive(Default, PartialEq, Clone, Serialize, Deserialize, Debug, Eq)]
pub struct Envelope<T> {
    pub input: u32,
    pub offset: u32,
    pub payload: T,
    pub pushnum: bool,
    pub stutter: bool,
}
```

`Envelope<T>` は、任意の型 `T` を `payload` として持てます。

```rust
// 具体的な型を指定して使用
pub type RawEnvelope = Envelope<Vec<Vec<u8>>>;
pub type ParsedEnvelope = Envelope<Inscription>;
```

TypeScriptとの比較:
```typescript
interface Envelope<T> {
    input: number;
    offset: number;
    payload: T;
    pushnum: boolean;
    stutter: boolean;
}

type RawEnvelope = Envelope<number[][]>;
type ParsedEnvelope = Envelope<Inscription>;
```

### トレイト境界

ジェネリック型に制約を付けることができます。

```rust
// Displayを実装している型のみ受け付ける
fn print<T: Display>(value: T) {
    println!("{}", value);
}

// 複数のトレイト境界
fn process<T: Display + Clone>(value: T) {
    let copy = value.clone();
    println!("{}", copy);
}
```

### where句

複雑なトレイト境界は `where` 句で書くと読みやすくなります。

```rust
fn complex_function<T, U>(t: T, u: U)
where
    T: Display + Clone,
    U: Debug + Default,
{
    // ...
}
```

---

## 11. イテレータとクロージャ

### Iteratorトレイト

Rustのイテレータは遅延評価され、非常に効率的です。

```rust
let v = vec![1, 2, 3, 4, 5];

// イテレータを作成（まだ何も実行されない）
let iter = v.iter();

// 値を消費する時に初めて評価される
for x in iter {
    println!("{}", x);
}
```

### クロージャ構文

クロージャは匿名関数です。

```rust
// 基本形
let add = |a, b| a + b;
let result = add(1, 2);  // 3

// 型注釈付き
let add: fn(i32, i32) -> i32 = |a, b| a + b;

// 複数行
let process = |x| {
    let y = x * 2;
    y + 1
};
```

### map、filter、collect

**src/inscriptions/envelope.rs:96-102**
```rust
impl ParsedEnvelope {
    pub fn from_transaction(transaction: &Transaction) -> Vec<Self> {
        RawEnvelope::from_transaction(transaction)
            .into_iter()
            .map(|envelope| envelope.into())
            .collect()
    }
}
```

この関数は:
1. `RawEnvelope::from_transaction` でRawEnvelopeのVecを取得
2. `.into_iter()` で所有権を取得するイテレータに変換
3. `.map()` で各要素を変換
4. `.collect()` でVecに集約

TypeScriptとの比較:
```typescript
function fromTransaction(transaction: Transaction): ParsedEnvelope[] {
    return RawEnvelope.fromTransaction(transaction)
        .map(envelope => envelope.into());
}
```

### イテレータチェーンの実例

**src/inscriptions/envelope.rs:29-93**
```rust
impl From<RawEnvelope> for ParsedEnvelope {
    fn from(envelope: RawEnvelope) -> Self {
        let body = envelope
            .payload
            .iter()
            .enumerate()
            .position(|(i, push)| i % 2 == 0 && push.is_empty());

        let mut fields: BTreeMap<&[u8], Vec<&[u8]>> = BTreeMap::new();

        for item in envelope.payload[..body.unwrap_or(envelope.payload.len())].chunks(2) {
            match item {
                [key, value] => fields.entry(key).or_default().push(value),
                _ => incomplete_field = true,
            }
        }

        // ...
    }
}
```

使われているイテレータメソッド:
- `.iter()`: 不変参照のイテレータ
- `.enumerate()`: インデックス付きタプルにする
- `.position()`: 条件を満たす最初の位置を探す
- `.chunks(2)`: 2要素ずつのスライスに分割

---

## Part 5: 実践とコントリビューション

---

## 12. テストの書き方 - TDDアプローチ

### #[test]属性

Rustでは `#[test]` 属性をつけた関数がテストになります。

**crates/ordinals/src/sat.rs:391-395**
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn n() {
        assert_eq!(Sat(1).n(), 1);
        assert_eq!(Sat(100).n(), 100);
    }

    #[test]
    fn height() {
        assert_eq!(Sat(0).height(), 0);
        assert_eq!(Sat(1).height(), 0);
        assert_eq!(Sat(Epoch(0).subsidy()).height(), 1);
        // ...
    }
}
```

### #[cfg(test)]

`#[cfg(test)]` は**条件付きコンパイル**属性です。この属性がついたコードは、テスト時のみコンパイルされます。

```rust
#[cfg(test)]
mod tests {
    // テストコードはテスト時のみコンパイルされる
}
```

### assert マクロ

```rust
assert!(condition);                    // 条件が真であることを確認
assert_eq!(left, right);               // 等値を確認
assert_ne!(left, right);               // 不等を確認
assert!(condition, "error message");   // メッセージ付き
```

### テストの実例

**crates/ordinals/src/sat.rs:764-777**
```rust
#[test]
fn common() {
    #[track_caller]
    fn case(n: u64) {
        assert_eq!(Sat(n).common(), Sat(n).rarity() == Rarity::Common);
    }

    case(0);
    case(1);
    case(50 * COIN_VALUE - 1);
    case(50 * COIN_VALUE);
    case(50 * COIN_VALUE + 1);
    case(2067187500000000 - 1);
    case(2067187500000000);
    case(2067187500000000 + 1);
}
```

`#[track_caller]` は、テスト失敗時にヘルパー関数ではなく呼び出し元の行番号を表示します。

### ユニットテストと統合テスト

```
プロジェクト構造:
ord/
├── src/
│   └── lib.rs          # #[cfg(test)] mod tests { ... }
├── tests/              # 統合テスト
│   └── lib.rs
└── crates/
    └── mockcore/       # テスト用モッククレート
```

- **ユニットテスト**: 同じファイル内のprivate関数もテスト可能
- **統合テスト**: tests/ディレクトリに配置、public APIのみテスト

### mockcoreの仕組み

ordプロジェクトでは、Bitcoin Core RPCをモックする `mockcore` クレートを使用しています。

```rust
// 実際のBitcoin Coreを起動せずにテスト可能
let mock = mockcore::spawn();
let client = mock.client();
// テストを実行
```

---

## 13. 並行処理とスレッド

### Arc（Atomic Reference Counting）

`Arc` は、複数のスレッドから安全に参照できるスマートポインタです。

```rust
use std::sync::Arc;

let data = Arc::new(vec![1, 2, 3]);

// 別スレッドにクローンを渡す
let data_clone = Arc::clone(&data);
std::thread::spawn(move || {
    println!("{:?}", data_clone);
});
```

### Mutex

`Mutex` は、データへの排他的アクセスを提供します。

**src/lib.rs:154-156**
```rust
static SHUTTING_DOWN: AtomicBool = AtomicBool::new(false);
static LISTENERS: Mutex<Vec<axum_server::Handle>> = Mutex::new(Vec::new());
static INDEXER: Mutex<Option<thread::JoinHandle<()>>> = Mutex::new(None);
```

`static` 変数はプログラム全体で共有されるため、スレッドセーフである必要があります。

### AtomicBool

アトミック型は、ロックなしでスレッドセーフな操作を提供します。

**src/lib.rs:225-231**
```rust
pub fn cancel_shutdown() {
    SHUTTING_DOWN.store(false, atomic::Ordering::Relaxed);
}

pub fn shut_down() {
    SHUTTING_DOWN.store(true, atomic::Ordering::Relaxed);
}
```

`Ordering::Relaxed` は最も軽量なメモリオーダリングで、単一のフラグの読み書きには十分です。

### スレッドセーフティ

Rustでは、コンパイラがスレッドセーフティを保証します。

```rust
// Send: 別スレッドに所有権を移動できる
// Sync: 複数スレッドから参照できる

// Arc<Mutex<T>> は Send + Sync
// 生ポインタは !Send + !Sync（スレッド安全ではない）
```

---

## 14. 外部クレートとエコシステム

### 主要クレート解説

ordプロジェクトで使用されている主要なクレートを見てみましょう。

**Cargo.toml:46-100**（抜粋）

| クレート | 用途 |
|---------|------|
| `bitcoin` | Bitcoin関連の型と操作 |
| `redb` | 組み込みデータベース |
| `axum` | Webフレームワーク |
| `clap` | コマンドライン引数パーサ |
| `serde` | シリアライゼーション |
| `tokio` | 非同期ランタイム |

#### bitcoin クレート

```rust
use bitcoin::{
    Amount, Block, Network, OutPoint, Transaction, Txid,
    blockdata::constants::{DIFFCHANGE_INTERVAL, SUBSIDY_HALVING_INTERVAL},
};

let amount = Amount::from_sat(1000);
let network = Network::Bitcoin;  // Mainnet
```

#### clap クレート

コマンドライン引数を宣言的に定義:

```rust
use clap::Parser;

#[derive(Parser)]
struct Args {
    #[arg(short, long)]
    verbose: bool,

    #[arg(short, long, default_value = "mainnet")]
    network: String,
}
```

#### serde クレート

JSON等との相互変換:

```rust
#[derive(Serialize, Deserialize)]
struct User {
    name: String,
    age: u32,
}

let json = serde_json::to_string(&user)?;
let user: User = serde_json::from_str(&json)?;
```

### Cargo.tomlの読み方（詳細）

```toml
[dependencies]
# 基本的な依存関係
anyhow = { version = "1.0.90", features = ["backtrace"] }

# ワークスペースから継承
bitcoin.workspace = true

# ローカルパス依存
ordinals = { version = "0.0.15", path = "crates/ordinals" }

[dev-dependencies]
# テスト時のみの依存関係
mockcore = { path = "crates/mockcore" }
pretty_assertions.workspace = true
```

フィーチャーフラグ:
```toml
bitcoin = { version = "0.32.5", features = ["rand"] }
```

特定の機能を有効/無効にできます。

---

## 15. ordへのコントリビューション実践

### Good First Issueの見つけ方

1. GitHubの[Issues](https://github.com/ordinals/ord/issues)ページにアクセス
2. `good first issue` ラベルでフィルタ
3. 興味のあるIssueを選び、コメントで着手を宣言

### 開発環境のセットアップ

```bash
# リポジトリをフォーク＆クローン
git clone https://github.com/YOUR_USERNAME/ord.git
cd ord

# ビルド確認
cargo build

# テスト実行
cargo test
```

### cargo fmt

コードフォーマッタを実行:

```bash
# フォーマットチェック
cargo fmt -- --check

# 自動フォーマット
cargo fmt --all
```

### cargo clippy

Lintツールを実行:

```bash
# 警告をエラーとして扱う（CIと同じ設定）
cargo clippy --all --all-targets -- --deny warnings
```

### justfile

ordプロジェクトでは `just` コマンドランナーを使用しています。

**justfile:6-8**
```makefile
ci: clippy forbid
  cargo fmt -- --check
  cargo test --all
  cargo test --all -- --ignored
```

```bash
# CIと同じチェックを実行
just ci
```

### PRの作成方法

1. **ブランチを作成**
```bash
git checkout -b fix-issue-123
```

2. **変更を実装**
```bash
# コードを編集
# テストを追加
cargo test
```

3. **フォーマットとLint**
```bash
cargo fmt --all
cargo clippy --all --all-targets -- --deny warnings
```

4. **コミット**
```bash
git add -A
git commit -m "Fix issue #123: description"
```

5. **プッシュ＆PR作成**
```bash
git push origin fix-issue-123
# GitHubでPRを作成
```

### コントリビューションガイドライン

**CONTRIBUTING**ファイルより:
> Unless you explicitly state otherwise, any contribution intentionally
> submitted for inclusion in the work by you shall be licensed as in
> LICENSE, without any additional terms or conditions.

コントリビュートするコードは、プロジェクトと同じライセンス（CC0-1.0）で提供されます。

---

## 16. まとめと次のステップ

### 学習内容の総復習

この記事で学んだ内容を振り返りましょう。

```
Part 1: 導入
├── Rustの基本文法（let, mut, fn, match, if let）
└── ordのエントリーポイント

Part 2: データモデリング
├── 構造体とNewtype Pattern（Sat）
├── 列挙型とパターンマッチング（Rarity）
└── トレイトと型変換（From, Into, Display）

Part 3: エラー処理と所有権
├── Result/Option
├── 所有権と借用
└── ライフタイム

Part 4: プロジェクト構造
├── モジュールシステム
├── ジェネリクス
└── イテレータとクロージャ

Part 5: 実践
├── テストの書き方
├── 並行処理
├── 外部クレート
└── コントリビューション方法
```

### 理解度チェックリスト

以下の項目が理解できているか確認しましょう。

**基礎**
- [ ] `let` と `let mut` の違い
- [ ] 関数の定義方法と戻り値
- [ ] `match` 式の使い方

**データ型**
- [ ] 構造体の定義とインスタンス化
- [ ] 列挙型（enum）の定義とパターンマッチング
- [ ] Newtype Patternの目的

**トレイト**
- [ ] トレイトの概念
- [ ] `From`/`Into`/`Display` の実装方法
- [ ] `derive` マクロの役割

**エラー処理**
- [ ] `Option<T>` と `Result<T, E>` の違い
- [ ] `?` 演算子の動作
- [ ] `map`/`and_then`/`unwrap_or` の使い方

**所有権**
- [ ] 所有権の3ルール
- [ ] 借用（`&T`と`&mut T`）
- [ ] ライフタイムの基本

**プロジェクト構造**
- [ ] `mod` と `use` の使い方
- [ ] ジェネリクス `<T>`
- [ ] イテレータチェーン

**実践**
- [ ] テストの書き方
- [ ] `cargo fmt` と `cargo clippy`
- [ ] PRの作成フロー

### 推奨リソースとコミュニティ

**公式ドキュメント**
- [The Rust Programming Language](https://doc.rust-lang.org/book/) - 公式チュートリアル
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/) - 例で学ぶRust
- [Ordinals Documentation](https://docs.ordinals.com/) - Ordinals公式ドキュメント

**コミュニティ**
- [Rust Users Forum](https://users.rust-lang.org/) - Rust公式フォーラム
- [Ordinals Discord](https://discord.gg/ordinals) - Ordinals Discord

**次のステップ**
1. ordリポジトリをフォークして、実際にコードを読んでみる
2. `cargo test` を実行して、テストがどのように書かれているか確認
3. Good First Issueを探して、小さなコントリビュートから始める

### おわりに

この記事では、Bitcoin OrdinalsプロジェクトのRust実装を題材に、Rustプログラミングの基礎から実践までを解説しました。

Rustは学習曲線が急ですが、一度習得すれば非常に強力なツールになります。所有権システムやトレイトなど、最初は難しく感じる概念も、実際のコードを読みながら学ぶことで理解が深まります。

ordプロジェクトは、Rust学習の教材として非常に優れています。明確なドメインモデル、テストの充実、活発なコミュニティなど、OSS開発を学ぶのに最適な環境が整っています。

ぜひ、この記事をきっかけにRustとOrdinalsの世界に飛び込んでみてください。

---

*この記事の内容は ord v0.24.2 時点のコードに基づいています。最新のコードはGitHubリポジトリを参照してください。*
