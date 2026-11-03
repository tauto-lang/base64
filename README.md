# base64

証明付きの Base64（RFC 4648）。アルファベットとパディングを選べ、どの組み合わせでも往復を両向きに証明している。証明したコードがそのまま実行時に動く。

## 使い方

`src/bytes.taut` がプログラムから呼ぶ入口。

| 関数 | 内容 |
| --- | --- |
| `Base64::encode(t: Text) -> Text` | テキストのバイト列を標準アルファベット・パディング付きでエンコード |
| `Base64::encode_bytes(l: List(Nat), ok: all_bytes(l) = true) -> Text` | バイト列（256 未満の数）を同じくエンコード。`ok` は書いた時点で決まるリストなら `Refl` で示せる |
| `Base64::encode_bytes_checked(l: List(Nat)) -> Text ! {not_a_byte}` | 実行時に確かめてエンコード。最初の 256 以上の数で効果 `not_a_byte(at, n)`（位置と値）を起こす |
| `Base64::decode(t: Text) -> Option(List(Nat))` | パディング必須でデコード。デコード結果は UTF-8 とは限らないのでバイト列で返す |
| `Base64::encode_with(a, pm, t)`、`Base64::encode_bytes_with(a, pm, l, ok)`、`Base64::encode_bytes_checked_with(a, pm, l)`、`Base64::decode_with(a, m, t)` | アルファベット `a`、パディング `pm`、パディング規則 `m` を指定する |

- アルファベット `Alphabet`（`src/alphabet.taut`）: `standard()`（RFC 4648 第 4 節）、`url_safe()`（第 5 節）、`crypt()`、`bcrypt()`、`imap_mutf7()`、`bin_hex()`。独自のものは `alphabet_of("…64 文字…")` で作り、`alphabet_ok(a) = true` を `refl()` で示せば下の定理がすべて使える（64 文字の ASCII が互いに異なり `=` を含まず、デコード表が正しいことを計算で確かめる）。
- エンコードのパディング `Padding`: `Padded`（`=` で 4 記号に揃える）、`Unpadded`（RFC 4648 第 3.2 節）。
- デコードのパディング規則 `PadRule`: `PadRequired`、`PadOptional`（どちらも受け付ける）、`PadForbidden`。

デコードは厳格で、規則に合わないパディング、パディングの後の文字、アルファベット外の文字、最後のグループで使われないビットが 0 でないものを拒否する。

## 構成

- `src/codec.taut`: バイト列と記号（sextet かパッド）の列の変換 `encode_with(pm, l)`、`decode_with(m, s)`。バイトと sextet はビットの並び（上位ビットから）で、RFC 4648 の組み直しをビット単位でそのまま書いてある。
- `src/alphabet.taut`: アルファベットとデコード表。エンコードは sextet の値の位置の文字を読み、デコードは表を 1 回引く。
- `src/text.taut`: 記号の列と ASCII コード列の変換 `encode_text_with(a, pm, l)`、`decode_text_with(a, m, codes)`。
- `src/bytes.taut`: `Text` とバイト列の入口。

## 証明していること

公理は使わない。

- `decode_with_encode_with`（`proof/codec.taut`）: 規則 `m` がパディング `pm` を受け付けるなら（`Takes(m, pm)`）、すべてのバイト列 `l` について `decode_with(m, encode_with(pm, l)) = Some(l)`。
- `encode_with_decode_with`（`proof/codec.taut`）: 逆向き。`decode_with(m, s) = Some(l)` ならば `s` はその書き方での `l` のエンコード。`PadRequired`・`PadForbidden` では、デコードできる入力はバイト列ごとに一つだけ（正準）。
- `code_sextet_code`、`code_sextet_pad`、`code_sextet_sound`（`proof/alphabet.taut`）: `alphabet_ok` を満たすアルファベットでは、sextet は自分の文字から読み戻せ、`=` は文字でなく、文字が読まれる sextet の文字はその文字自身。用意した 6 個が `alphabet_ok` を満たすことも計算で確かめる。
- `decode_text_with_encode_text_with`、`encode_text_with_decode_text_with`（`proof/text.taut`）: ASCII コード列での両向き。
- `Base64::decode_with_encode_bytes_with`、`Base64::encode_bytes_with_decode_with`（`proof/bytes.taut`）: 入口での両向き。`all_bytes(l) = true` を満たす `l` について `Base64::decode_with(a, m, Base64::encode_bytes_with(a, pm, l, ok)) = Some(l)`、デコードできたテキストはそのバイト列のエンコード（`PadOptional` ならパディング付きかパッドなしのどちらか）。`Base64::decode_encode`、`Base64::encode_bytes_decode` は標準の場合。
- `proof/vectors.taut`: RFC 4648 第 10 節のテストベクタ（`""` から `"foobar"` まで）の両方向、パッドなし・URL-safe の例、各規則やアルファベットが拒否する入力。計算（`refl`）で確かめる。

```bash
tauto check proof/codec.taut
tauto check proof/alphabet.taut
tauto check proof/text.taut
tauto check proof/bytes.taut
tauto check proof/vectors.taut
```

## 速さ

`tauto extract src/bytes.taut` で書き出したものを Rust バックエンドでビルドすると、約 1 MB のテキストのエンコードとデコードを合わせて 1.6 秒（評価器 `tauto run` では 27 KB のエンコードに約 3 秒、デコードに約 2.5 秒。書き出したファイルの読み込みに別に約 6 秒）。バイトをビットの構成子で表しリストで運ぶので、機械語の語単位で処理する実装よりはずっと遅い。

## まだ扱っていないこと

- 語単位（3 バイトを 1 つの整数で）の速い実装と、それがここでの定義と一致することの証明。
- ストリーム（入力を分けて渡す）でのエンコードとデコード。

## ライセンス

[MIT](LICENSE-MIT) と [Apache-2.0](LICENSE-APACHE) のデュアルライセンス。どちらかを選んで使える。
