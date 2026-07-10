# 2つの Copy タスクの前者と後者で from と into は別のメソッド？ですか

## 質問

```groovy
task extractBinary(type: Copy) {
    from rpmExtracted
    into "${binaryDir}/dest"
    duplicatesStrategy 'exclude'
}
```

と

```groovy
task extractBinary(type: Copy) {
    from(rpmExtracted) {
        into "${binaryDir}/dest"
    }
    duplicatesStrategy 'exclude'
}
```

は、前者と後者で from と into は別のメソッド？ですか


## 回答

結論から言うと：

**どちらも同じ `from()` と `into()` メソッドですが、
呼び出される対象（CopySpec）が異なるため挙動が変わります。**

つまり、**メソッドが違うのではなく、
“どのオブジェクトに対して呼ばれているか” が違うだけ**です。

---

### 1. Gradle の Copy タスクは CopySpec のツリー構造を持つ

Gradle の `Copy` タスクは内部に以下のような階層構造を持ちます。

```
Copy task
└── root CopySpec（タスク全体の設定）
      ├── child CopySpec（from A の設定）
      ├── child CopySpec（from B の設定）
      └── ...
```

`from { ... }` を書くと **サブ CopySpec** が作成され、
そのブロック内で呼ばれた `into()` は **そのサブ CopySpec の設定**になります。

---

### 2. パターン A（トップレベルに from と into）

```groovy
task extractBinary(type: Copy) {
    from rpmExtracted
    into "${binaryDir}/dest"
    duplicatesStrategy 'exclude'
}
```

これはすべて **root（メイン）CopySpec** に適用されます。

* `from` → rootSpec にコピー元を追加
* `into` → rootSpec にコピー先を設定

**結果：すべての from が同じ into を共有する**

---

### 3. パターン B（from のブロック内で into を設定）

```groovy
task extractBinary(type: Copy) {
    from(rpmExtracted) {
        into "${binaryDir}/dest"
    }
    duplicatesStrategy 'exclude'
}
```

ここでは

* `from(rpmExtracted)` により **サブ CopySpec が生成される**
* `{ ... }` の中の `into` は **そのサブ CopySpec にだけ適用される**

**結果：この from のみ専用の into が適用される**

---

### 4. なぜ違う挙動になるのか？

理由は **Groovy DSL の delegate（委譲）メカニズム**です。

* トップレベルでは
  → `into()` は task の root CopySpec に対して実行される
* `from { ... }` ブロック内では
  → `delegate` が「子 CopySpec」になる
  → `into()` はサブ CopySpec に対する呼び出しになる

**メソッドは同じでも、呼び出し対象が違うため結果が変わる**という仕組み。

---

### 5. まとめ

* `from()` と `into()` 自体は同じメソッドである
* しかし、適用される CopySpec が

  * トップレベル → rootSpec
  * from ブロック内 → childSpec
    と変わるため挙動が異なる
* これは Gradle の CopySpec の階層構造と
  Groovy DSL の delegate によるもの


#### Configuration フェーズ（定義フェーズ）

* task extractBinary(type: Copy) の部分で Copy オブジェクト（Task オブジェクトのサブクラス）を生成。
* task { ... } の部分（Groovy クロー ジャ）が実行され、Copy オブジェクトの詳細な定義がされる
* from, into, duplicatesStrategy などは、属性を設定するようなメソッド。

#### Execution フェーズ（実行フェーズ）

* 実際の処理は Configuration フェーズで定義された Copy オブジェクトを使って、特殊な方法で内部処理が行われます。
* doLast { ... } や Copy タスクの実処理が実行される


# Gradle の書き方は「内部仕様」によって決まるが、どのように理解すればいいか

## 質問

```groovy
task extractBinary(type: Copy) {
    from rpmExtracted
    into "${binaryDir}/dest"
    duplicatesStrategy 'exclude'
}
```

と

```groovy
task extractBinary(type: Copy) {
    from(rpmExtracted) {
        into "${binaryDir}/dest"
    }
    duplicatesStrategy 'exclude'
}
```

は、その内部的な処理の仕様によって書き方が決まるということですか


## 回答

— どう理解すればよいか

### 結論

はい。Gradle の `Copy` タスクにおける
**`from` / `into` の書き方の違いは、Gradle の内部仕様（CopySpec の階層構造 + Groovy DSL の delegate 仕様）によって決まります。**

しかし、コードの意味を理解するときは、
**「実際の処理」ではなく「仕様としての意味」で読む**とスッキリします。

---

### 1. Gradle のコードは「命令」ではなく「設定」として読む

一般的なコードは「上から順に実行される命令」ですが、
Gradle の `build.gradle` は **DSL による設定**です。

```groovy
task extractBinary(type: Copy) {
    from rpmExtracted
    into "${binaryDir}/dest"
}
```

これは

> 「`extractBinary` という Copy task に対して
> `from` と `into` を設定している」

と読むのが正しい。

コピー処理が実行されるのは Gradle 実行時であり、
スクリプトは **設定を宣言しているだけ**です。

---

### 2. 違いが生まれる理由：CopySpec の階層構造

`Copy` タスクは内部的に次のような階層を持ちます：

```
Copy Task
└── Root CopySpec  ← タスク全体
      ├── Sub CopySpec (from A)
      ├── Sub CopySpec (from B)
      └── ...
```

この階層があるため、

* **トップレベル `from` / `into`**
  → rootSpec へ設定

* **from { into } の中の `into`**
  → subSpec（その `from` 専用の CopySpec）に設定

という動作になります。

---

### 3. 「内部仕様でそうなる」→ ではどう理解する？

Gradle DSL のようなコードは、次の **3 つのレイヤー**に分けて理解できます。

#### ① 仕様レベル（重要）

ユーザーが従うべき公式の意味：

* トップレベルの `into` → すべての `from` に適用
* `from { into }` → その `from` にだけ適用
* `from` を複数書くと Sub CopySpec が複数できる

これは完全に **Gradle の公開仕様として保証されている挙動**です。

#### ② 実装レベル（理由付け）

* CopySpec の階層構造
* Groovy DSL の delegate によるメソッド解決

挙動の違いはこの **内部構造**によって説明できます。

#### ③ ソースコードレベル（追う必要なし）

Decorated クラス／内部フィールドなど
ここまで知る必要はありません。

---

### 4. 実例での「意味」の読み取り方

#### パターン A

```groovy
task extractBinary(type: Copy) {
    from rpmExtracted
    into "${binaryDir}/dest"
}
```

**仕様としての意味**

> 「`rpmExtracted` 以下のファイルを
> `binaryDir/dest` にコピーする。」

→ すべて rootSpec へ設定されている。

---

#### パターン B

```groovy
task extractBinary(type: Copy) {
    from(rpmExtracted) {
        into "${binaryDir}/dest"
    }
}
```

**仕様としての意味**

> 「`rpmExtracted` をコピーするための
> 1 個の subSpec を作り、
> そこに専用の into を設定している。」

→ subSpec に対して設定されている。

実装の違いは内部仕様によりますが、
**読者が意識すべきは「どの Spec に対して設定されているか」だけ**です。

---

### 5. 「メタっぽい」処理の理解方法（指針）

内部がメタ的だからといって、コードが理解しにくくなるときは
次の 3 点を意識すると整理できます。

#### ① 公開仕様を“正”とする

Gradle ドキュメントに従う。

#### ② 自分の中で 1 つのモデルを持つ

例：

```
rootSpec にタスク全体の設定
from(...) で subSpec が作られる
from { ... } ブロック内では subSpec を設定する
```

#### ③ 実際の Spec ツリーをダンプして確認する

dump 関数などで現実の挙動を観察し、理解を補強する。

---

### 6. まとめ

* `from` / `into` の挙動の違いは
  **Gradle の CopySpec 内部構造と Groovy DSL の delegate** により発生する
* しかしコードを読むときは **「設定としてどういう意味を持つか」** だけ読めばよい
* DSL は「命令」ではなく「宣言的な設定」
* 仕様レベル → 実装レベル の順で理解すると混乱しない

