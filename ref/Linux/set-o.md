# set -o は何ですか

## 質問

set -o は何ですか

## 回答

`set -o` は**シェルオプションを設定・表示する**コマンドです。

### 基本的な使い方

**1. 現在の設定を表示**
```bash
set -o
```
すべてのシェルオプションとその on/off 状態が表示されます。

**2. オプションを有効にする**
```bash
set -o オプション名
```

**3. オプションを無効にする**
```bash
set +o オプション名
```

### 短縮形との対応

一部のオプションには短縮形があります:

| 長い形式 | 短縮形 | 説明 |
|---------|--------|------|
| `set -o nounset` | `set -u` | 未定義変数の使用を禁止 |
| `set -o errexit` | `set -e` | エラー発生時にスクリプトを終了 |
| `set -o xtrace`  | `set -x` | 実行コマンドを表示（デバッグ用） |
| `set -o verbose` | `set -v` | 読み込んだ行を表示 |

### よく使うオプション

#### nounset (set -u)
未定義変数を参照するとエラーになります。

```bash
set -o nounset
echo $UNDEFINED_VAR  # エラーで終了
```

#### errexit (set -e)
コマンドが失敗したらスクリプトが終了します。

```bash
set -o errexit
false  # ここでスクリプトが終了
echo "ここは実行されない"
```

#### pipefail
パイプラインの途中でエラーが起きても検出します。

```bash
set -o pipefail
false | true  # 終了コード: 1（パイプ全体のエラーを検出）
```

#### xtrace (set -x)
実行されるコマンドを表示します（デバッグに便利）。

```bash
set -o xtrace
greeting="Hello"
echo "$greeting, World!"  # + echo 'Hello, World!' と表示される
set +o xtrace
```

#### noclobber
リダイレクトでの既存ファイル上書きを防ぎます。

```bash
set -o noclobber
echo "test" > existing.txt  # エラー: ファイルが既に存在
echo "test" >| existing.txt  # 強制上書き（>| を使用）
```

### 実用例

スクリプトの先頭でよく使われる設定:

```bash
#!/bin/bash
set -euo pipefail

# -e: コマンドがエラーを返したら即座に終了
# -u: 未定義変数を使ったらエラー
# -o pipefail: パイプラインのどこかが失敗したら全体を失敗とみなす
```

この組み合わせは**堅牢なシェルスクリプト**を書くための標準的な設定として広く推奨されています。

### まとめ

- `set -o` は引数なしで**現在の設定を表示**
- `set -o オプション名` で**有効化**
- `set +o オプション名` で**無効化**
- 一部のオプションには短縮形（`-e`, `-u`, `-x` など）がある
- スクリプトの品質向上のため、`set -euo pipefail` がよく使われる



# shopt

## 質問

shopt は何ですか

## 回答

`shopt` は bash 固有の**シェルオプションを設定・表示する**コマンドです。`set -o` とは別のオプション群を扱います。## shopt とは

`shopt` は **bash 固有のシェルオプション**を設定・表示するコマンドです。`set -o` とは異なるオプション群を扱います。

### 基本的な使い方

```bash
shopt              # 全オプションの状態を表示
shopt -s option    # オプションを有効にする
shopt -u option    # オプションを無効にする
shopt -q option    # 静かに確認（終了ステータスで判定）
```

### set -o との違い

| | set -o | shopt |
|---|---|---|
| **由来** | POSIX標準 | bash固有 |
| **互換性** | 他のシェルでも使える | bashでのみ |
| **用途** | 基本的な動作制御 | bash拡張機能 |
| **有効化** | `set -o option` | `shopt -s option` |
| **無効化** | `set +o option` | `shopt -u option` |

### よく使われる shopt オプション

**dotglob** - 隠しファイルもワイルドカード展開に含める
```bash
shopt -s dotglob
ls *  # .hidden ファイルも表示される
```

**nullglob** - パターンにマッチするものがない場合は空文字列に
```bash
shopt -s nullglob
for file in *.nonexistent; do
    echo "$file"  # 何も出力されない（エラーにならない）
done
```

**extglob** - 拡張パターンマッチング
```bash
shopt -s extglob
ls !(*.txt)      # .txt 以外のファイル
ls +(*.jpg|*.png)  # .jpg または .png のファイル
```

**globstar** - `**` で再帰的にディレクトリを検索
```bash
shopt -s globstar
ls **/*.txt  # サブディレクトリも含めて .txt ファイルを検索
```

**nocaseglob** - 大文字小文字を区別しないマッチング
```bash
shopt -s nocaseglob
ls *.TXT  # abc.txt, ABC.txt, aBc.TxT などすべてマッチ
```

### 実用例

```bash
#!/bin/bash

# エラー処理（set -o を使用）
set -euo pipefail

# ファイル操作を便利に（shopt を使用）
shopt -s nullglob   # マッチなしで空にする
shopt -s dotglob    # 隠しファイルも処理対象
shopt -s globstar   # ** で再帰検索

# すべての .txt ファイルを処理
for file in **/*.txt; do
    echo "Processing: $file"
done
```

### 使い分けの目安

- **エラー処理、デバッグ** → `set -o`
- **ファイル名展開、パターンマッチング** → `shopt`
- **移植性が必要** → `set -o`
- **bash専用の便利機能** → `shopt`