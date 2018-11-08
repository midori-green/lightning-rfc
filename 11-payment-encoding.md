# ボルト #11: ライトニング決済の為のインボイスプロトコル

ライトニング決済のための簡単で、拡張可能なQRコードプロトコル。

# コンテンツ一覧

  * [エンコーディング概要](#encoding-overview)
  * [ヒューマンリーダブルパート](#human-readable-part)
  * [データパート](#data-part)
    * [タグフィールド](#tagged-fields)
  * [支払人／受取人 の相互作用](#payer--payee-interactions)
    * [支払人／受取人 の受取人側の必要条件](#payer--payee-requirements)
  * [実装](#implementation)
  * [例](#examples)
  * [著者](#authors)

# エンコーディング概要

ライトニングインボイスのフォーマットはビットコインのSegwit と同じく [base32エンコーディング](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki)を使う。手入力のために6文字のチェックサムに最適化はされていることがあるが、その場合でもライトニングインボイスに再利用することができる。しかし、ライトニングインボイスの長さを考慮すると、再利用する事例はあまり起こらない。

‘Lightning:’（注：’lightning://’ ではない）BOLT-11のエンコーディングの前につけるか、BIP-21にある’bitcoin:’ をライトニングの鍵とともに使い、値はBOLT-11エンコーディングとするのが望ましい。

## 必要条件

書き手は Bech32（BIP-0173 参照）文字列が90文字を超えるときを除き、Bech32 を使って支払要求をエンコードする必要がある。 読み手はアドレスを（前述の文字数制限を同様の例外として） Bech32 でパースしなければならず、チェックサムが不正なときは読み取り失敗としなければならない。

# ヒューマンリーダブルパート

ライトニングインボイスのヒューマンリーダブルパートは以下の２つから成る:

1. `prefix`: `ln` + BIP-0173 の現状の接頭辞（すなわち、ビットコインメインネットにおける `lnbc`、ビットコインテストネットにおける`lntb`、ビットコイン regtest における`lnbcrt`）
1. `amount`: 任意の`multiplier`文字に続く、任意に設定できる通貨の数量。ここでエンコードされたこの単位は、決済単位の「社交」会である－ビットコインの場合、単位は ’bitcoin’ であり、satoshi ではない。

以下に定義された`multiplier`文字をあげる。

* `m` (ミリ): 0.001 倍
* `u` (ミクロ): 0.000001 倍
* `n` (ナノ): 0.000000001 倍
* `p` (ピコ): 0.000000000001 倍

## 必要条件

書き手:
  - 成功した決済は、使用する通貨の`prefix`がエンコードされてなければならない、
  - 成功した決済が特定の最小額を必要とするならば
    - `amount`を含まなければならない
    - `amount`を先頭のゼロがない10進法の整数としてエンコードしなくてはならない
    - 最大の`multiplier`か、multiplierを省略することにより、可能な限り短く表現すべきである

読み手:
  - `prefix`が認識できなければ失敗としなければならない
  - `amount`が空なら If the `amount` is empty:
    - 額が指定されていないかどうかを知らせるべきである
  - さもなくば:
    - もし`amount`が非アラビア数字か、上記にある`multiplier `以外の文字列が`amount`以降に続いていた場合、失敗としなければならない。
    - `multiplier`が含まれている場合:
      - `amount`に`multiplier`をかけて、支払額を得る。

## 原理

`amount`はヒューマンリーダブルパートにおいて、いくら要求されたかをそれなりに読みやすくて便利な指標でエンコードされている。

寄付アドレスは関連した額がないため、`amount`は任意となる。通常は何が対価として提供されようが、最小の支払は要求される。

# データパート

ライトニングインボイスのデータパートは以下から成る：

1. `timestamp`: 1970年以降からの秒数（35ビット、ビッグエンディアン）
1. ０以上個数のタグパート
1. `signature` 上記のビットコイン形式の署名（520ビット）

## 必要条件

書き手は`timestamp`に1970年１月夜12時（UTC）からの秒数をビッグエンディアンで格納しなくてはならない。書き手は`signature`にヒューマンリーダブルパートにあるSHA2(SHA256)でハッシュ化した有効な512ビットsecp256k1（楕円曲線暗号）署名を格納しなくてはならない。 `signature` は UTF-8のバイトで表され、データパート（署名は除く）と0のビットを次のバイトの境界線として加えたものと、復元ID(0,1,2,3のいずれか)を末尾のバイトに結合したものである。

読みては`signature`が正しいかどうか調べなければならない（下記の`n`個タグ付けされたフィールドを参照）。

## 原理

SHA2はビット領域のハッシュを実は標準でサポートしているが、それは広く使われていないので、`signature`は正確なバイト数を保証している。復元IDは公開鍵の復元を許可しており、従って受取人ノードの正体は暗にほのめかされている。

## タグフィールド

それぞれのタグフィールドは以下の形式をしている：

1. `type` （5ビット）
1. `data_length` （10ビット、ビッグエンディアン）
1. `data` （`data_length` x 5 ビット）

タグフィールドは現在以下のように定義されている:

* `p` (1): `data_length` 52。256-bit の SHA256 ペイメントハッシュ. これのプレイメージは、支払証明を提供する。
* `d` (13): `data_length` 変数。支払目的の短い説明（UTF-8）。例：「1杯のコーヒー」もしくは「漫画を1ページ読んだ」
* `n` (19): `data_length` 53。受取人ノードの33バイトの公開鍵。
* `h` (23): `data_length` 52。256ビットの支払目的の説明(SHA256)。
これは639バイト超の関連した記載を求める為に使われる、but the transport mechanism for the description in that case is transport specific and not defined here.
* `x` (6): `data_length` 変数。 `expiry` は秒数（ビッグエンディアン）で、指定がない場合は3600（１時間）となる。
* `c` (24): `data_length` 変数。 `min_final_cltv_expiry` は経路の最後のHTLCに使われ、指定がない場合は9となる。
* `f` (9): `data_length` 変数で、バージョン依存する。フォールバックオンチェーンアドレス：ビットコインでは、これは5ビットの `version`から始まり、P2PKHかP2SHのwitnessプログラムを含む。
* `r` (3): `data_length` 変数。プライベートな経路の為の他の情報を含む一かそれ以上のエントリー。以下の複数の`r`フィールドがあるかもしれない。
   * `pubkey` (264 ビット)
   * `short_channel_id` (64 ビット)
   * `fee_base_msat` (32 ビット、ビッグエンディアン)
   * `fee_proportional_millionths` (32 ビット、ビッグエンディアン)
   * `cltv_expiry_delta` (16 ビット、ビッグエンディアン)

### 必要条件

書き手はちょうど一つの`p`フィールドを含み、支払と引き換えに与えられ`payment_preimage`のSHA2ハッシュを`payment_hash`に格納しなくてはならない。

書き手はちょうど一つの`d`か、ちょうど一つの`h`フィールドを含まなければならない。含まれていれば、書き手は`d`を支払目的の完成した記述とすべきであり、その際には有効なUTF-8文字列を使わなければならない。含まれていれば、書き手は`h`に含まれるハッシュ化された記述のプレイメージをいくつかの非特定の手段を通して利用可能にしなければならない。記述は支払目的の完全な説明であるべきである。

書き手は`x`フィールドを一つ含ませることができる。

書き手は`c`フィールドを一つ含ませることができる。`c`フィールドは最小の`cltv_expiry`を含まなければならず、これは経路上の最後のHTLCが受け入れる。

書き手は可能であれば最小の`data_length`を`x`と`c`フィールドに使うべきである。

書き手は一つの`n`フィールドを含むことができる。`n`フィールドには、`signature`を作成する為に使われる公開鍵を格納しなくてはならない。

書き手は一つ以上の`f`フィールドを含ませることができる。ビットコイン払いの場合、書き手は`f`フィールドに有効なwitnessバージョンとプログラムか、`17`の後に続く公開鍵のハッシュか、`18`の後に続くスクリプトハッシュを格納しなくてはならない。

書き手は公開鍵に関連した公開チャネルがない場合、最低でも一つの`r`フィールドを含まなければならない。`r`フィールドはパブリックノードから最終目的地に向かうルートを示す一つ以上の整列されたエントリーを含む必要がある。それぞれのエントリーで、`pubkey`はチャネルの始点となるノードIDである。つまり、`short_channel_id`はチャネルを特定する短いチャネルIDのフィールドである。`fee_base_msat`、`fee_proportional_millionths`、`cltv_expiry_delta`は[BOLT #7](07-routing-gossip.md#the-channel_update-message)に詳細がある。書き手は複数のルーティングオプションを提供する一つ以上の`r`フィールドを含む必要がある。

書き手はフィールドデータを5ビットの倍数で詰めなければならない（空いたビットには0を詰める）。

書き手が一つ以上のフィールド型を提供するのなら、最初に最も望ましいフィールドを、続いてそれ以外のフィールドという順番で指定しなければならない。

読み手は不明なフィールド、`version`が不明な`f`フィールド、もしくは`p`、`h`、`n`フィールドで`data_length`でそれぞれ52、 52、53 をもっていないものを無視しなければならない。

読み手は ハッシュ化された説明が`h`フィールドの SHA2と正確に一致するか確かめなけれbあならない。

読み手は有効な`n`フィールドが与えられたならば、署名の復元をせずに、`n`フィールドを署名を検証する為に用いなければならない。

### 原理

型と長さのフォーマットは後方互換性をもったまま将来的な拡張を可能にする。`data_length`はエンコードとデコードを簡単にする為に、常に５ビットの倍数である。将来の変更が予想されるフィールドのため、読み手は異なる長さのフィールドを無視する。

`p`フィールドは現在256ビットのペイメントハッシュをサポートしているが将来の仕様は今と異なる新たな長さを加えられるかもしれない。書き手が新旧両方サポートできるように、古い読み手は正しい長さでないものを無視する。

`d`フィールドはインラインの記述を可能にするが、複雑なものには不十分かもしれない。そのため、`h`フィールドは要約を許可する、though the method
by which the description is served is as-yet unspecified and will
probably be transport dependent. `h` フォーマットは長さを変更することにより変わりうるので、読み手はこれが256バイトでないときに無視する。

`n`フィールドは署名の復元を要求する代わりに、目的地ノードIDを明示的に記述する為に使われる

`x`フィールドは支払が拒否されたときに警告する為に使われる。これは、主に混乱を避ける目的で使われる。通常、必要に応じてオンチェーン決済に十分な時間的猶予を与えるのがほとんどの支払にとって懸命である。

`c`フィールドは入金のHTLCのために特定の最小のCLTV(CheckLockTimeVerify、絶対時間によるロック方法)満了までを期限として、目的地ノードへ経路を与える。Destination nodes may use this to require a higher, more conservative value than the default one, depending on their fee estimation policy and their sensitivity to time locks. 経路中のリモートノードは`channel_update`の中にこれらの`cltv_expiry_delta`を要求し、リモートノードは`cltv_expiry_delta `をいつでも更新することができることを留意すること。

`f`フィールドはオンチェーンのフィードバックを許可するが、これは極小か、時間制約のある支払においては意味をなさない。新しいアドレス形式が今後登場しうるので、複数の`f`フィールドが暗黙的に用意されていることが望ましく、`f`フィールドで19から31のバージョンは読み手に無逸される。

`r`フィールドは制限されたルーティングを支援を許可する。すなわち、プライベートチャネルを使うための最低限の情報のみを許可するように指定されているが、部分的知識による将来のルーティングも支援することができる。

### Security Considerations for Payment Descriptions

Payment descriptions are user-defined and provide a potential avenue for
injection attacks, both in the process of rendering and persistence.

Payment descriptions should always be sanitized before being displayed in
HTML/Javascript contexts, or any other dynamically interpreted rendering
frameworks. Implementers should be extra perceptive to the possibility of
reflected XSS attacks when decoding and displaying payment descriptions. Avoid
optimistically rendering the contents of the payment request until all
validation, verification, and sanitization have been successfully completed.

Furthermore, consider using prepared statements, input validation, and/or
escaping to protect against injection vulnerabilities against persistence
engines that support SQL or other dynamically interpreted querying languages.

* [Stored and Reflected XSS Prevention](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet)
* [DOM-based XSS Prevention](https://www.owasp.org/index.php/DOM_based_XSS_Prevention_Cheat_Sheet)
* [SQL Injection Prevention](https://www.owasp.org/index.php/SQL_Injection_Prevention_Cheat_Sheet)

Don't be like the school of [Little Bobby Tables](https://xkcd.com/327/).

# Payer / Payee Interactions

These are generally defined by the rest of the Lightning BOLT series,
but it's worth noting that [BOLT #5](05-onchain.md) specifies that the payee SHOULD
accept up to twice the expected `amount`, so the payer can make
payments harder to track by adding small variations.

The intent is that the payer recover the payee's node ID from the
signature, and after checking that conditions such as fees,
expiry, and block timeout are acceptable, attempt a payment. It can use `r` fields to
augment its routing information if necessary to reach the final node.

If the payment succeeds but there is a later dispute, the payer can
prove both the signed offer from the payee and the successful
payment.

## Payer / Payee Requirements

A payer SHOULD NOT attempt a payment after the `timestamp` plus
`expiry` has passed. Otherwise, if a Lightning payment fails, a payer
MAY attempt to use the address given in the first `f` field that it
understands for payment. A payer MAY use the sequence of channels
specified by the `r` field to route to the payee. A payer SHOULD consider the
fee amount and payment timeout before initiating payment. A payer
SHOULD use the first `p` field that it did not skip as the payment hash.

A payee SHOULD NOT accept a payment after `timestamp` plus `expiry`.

# Implementation

https://github.com/rustyrussell/lightning-payencode

# Examples

NB: all the following examples are signed with `priv_key`=`e126f68f7eafcc8b74f54d269fe206be715000f94dac067d1c04a8ca3b2db734`.

> ### Please make a donation of any amount using payment_hash 0001020304050607080900010203040506070809000102030405060708090102 to me @03e7156ae33b0a208d0744199163177e909e80176e55d97a2f221ede0f934dd9ad
> lnbc1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdpl2pkx2ctnv5sxxmmwwd5kgetjypeh2ursdae8g6twvus8g6rfwvs8qun0dfjkxaq8rkx3yf5tcsyz3d73gafnh3cax9rn449d9p5uxz9ezhhypd0elx87sjle52x86fux2ypatgddc6k63n7erqz25le42c4u4ecky03ylcqca784w

Breakdown:

* `lnbc`: prefix, lightning on bitcoin mainnet
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `p`: payment hash
  * `p5`: `data_length` (`p` = 1, `5` = 20; 1 * 32 + 20 == 52)
  * `qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypq`: payment hash 0001020304050607080900010203040506070809000102030405060708090102
* `d`: short description
  * `pl`: `data_length` (`p` = 1, `l` = 31; 1 * 32 + 31 == 63)
  * `2pkx2ctnv5sxxmmwwd5kgetjypeh2ursdae8g6twvus8g6rfwvs8qun0dfjkxaq`: 'Please consider supporting this project'
* `8rkx3yf5tcsyz3d73gafnh3cax9rn449d9p5uxz9ezhhypd0elx87sjle52x86fux2ypatgddc6k63n7erqz25le42c4u4ecky03ylcq`: signature
* `ca784w`: Bech32 checksum
* Signature breakdown:
  * `38ec6891345e204145be8a3a99de38e98a39d6a569434e1845c8af7205afcfcc7f425fcd1463e93c32881ead0d6e356d467ec8c02553f9aab15e5738b11f127f` hex of signature data (32-byte r, 32-byte s)
  * `0` (int) recovery flag contained in `signature`
  * `6c6e62630b25fe64410d00004080c1014181c20240004080c1014181c20240004080c1014181c202404081a1fa83632b0b9b29031b7b739b4b232b91039bab83837b93a34b733903a3434b990383937b532b1ba0` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `c3d4e83f646fa79a393d75277b1d858db1d1f7ab7137dcb7835db2ecd518e1c9` hex of SHA256 of the preimage

> ### Please send $3 for a cup of coffee to the same peer, within 1 minute
> lnbc2500u1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdq5xysxxatsyp3k7enxv4jsxqzpuaztrnwngzn3kdzw5hydlzf03qdgm2hdq27cqv3agm2awhz5se903vruatfhq77w3ls4evs3ch9zw97j25emudupq63nyw24cg27h2rspfj9srp

Breakdown:

* `lnbc`: prefix, lightning on bitcoin mainnet
* `2500u`: amount (2500 micro-bitcoin)
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `p`: payment hash...
* `d`: short description
  * `q5`: `data_length` (`q` = 0, `5` = 20; 0 * 32 + 20 == 20)
  * `xysxxatsyp3k7enxv4js`: '1 cup coffee'
* `x`: expiry time
  * `qz`: `data_length` (`q` = 0, `z` = 2; 0 * 32 + 2 == 2)
  * `pu`: 60 seconds (`p` = 1, `u` = 28; 1 * 32 + 28 == 60)
* `aztrnwngzn3kdzw5hydlzf03qdgm2hdq27cqv3agm2awhz5se903vruatfhq77w3ls4evs3ch9zw97j25emudupq63nyw24cg27h2rsp`: signature
* `fj9srp`: Bech32 checksum
* Signature breakdown:
  * `e89639ba6814e36689d4b91bf125f10351b55da057b00647a8dabaeb8a90c95f160f9d5a6e0f79d1fc2b964238b944e2fa4aa677c6f020d466472ab842bd750e` hex of signature data (32-byte r, 32-byte s)
  * `1` (int) recovery flag contained in `signature`
  * `6c6e626332353030750b25fe64410d00004080c1014181c20240004080c1014181c20240004080c1014181c202404081a0a189031bab81031b7b33332b2818020f00` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `3cd6ef07744040556e01be64f68fd9e1565fb47d78c42308b1ee005aca5a0d86` hex of SHA256 of the preimage

> ### Please send 0.0025 BTC for a cup of nonsense (ナンセンス 1杯) to the same peer, within 1 minute
> lnbc2500u1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdpquwpc4curk03c9wlrswe78q4eyqc7d8d0xqzpuyk0sg5g70me25alkluzd2x62aysf2pyy8edtjeevuv4p2d5p76r4zkmneet7uvyakky2zr4cusd45tftc9c5fh0nnqpnl2jfll544esqchsrny

Breakdown:

* `lnbc`: prefix, lightning on bitcoin mainnet
* `2500u`: amount (2500 micro-bitcoin)
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `p`: payment hash...
* `d`: short description
  * `pq`: `data_length` (`p` = 1, `q` = 0; 1 * 32 + 0 == 32)
  * `uwpc4curk03c9wlrswe78q4eyqc7d8d0`: 'ナンセンス 1杯'
* `x`: expiry time
  * `qz`: `data_length` (`q` = 0, `z` = 2; 0 * 32 + 2 == 2)
  * `pu`: 60 seconds (`p` = 1, `u` = 28; 1 * 32 + 28 == 60)
* `yk0sg5g70me25alkluzd2x62aysf2pyy8edtjeevuv4p2d5p76r4zkmneet7uvyakky2zr4cusd45tftc9c5fh0nnqpnl2jfll544esq`: signature
* `chsrny`: Bech32 checksum
* Signature breakdown:
  * `259f04511e7ef2aa77f6ff04d51b4ae9209504843e5ab9672ce32a153681f687515b73ce57ee309db588a10eb8e41b5a2d2bc17144ddf398033faa49ffe95ae6` hex of signature data (32-byte r, 32-byte s)
  * `0` (int) recovery flag contained in `signature`
  * `6c6e626332353030750b25fe64410d00004080c1014181c20240004080c1014181c20240004080c1014181c202404081a1071c1c571c1d9f1c15df1c1d9f1c15c9018f34ed798020f0` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `197a3061f4f333d86669b8054592222b488f3c657a9d3e74f34f586fb3e7931c` hex of SHA256 of the preimage

> ### Now send $24 for an entire list of things (hashed)
> lnbc20m1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqscc6gd6ql3jrc5yzme8v4ntcewwz5cnw92tz0pc8qcuufvq7khhr8wpald05e92xw006sq94mg8v2ndf4sefvf9sygkshp5zfem29trqq2yxxz7

Breakdown:

* `lnbc`: prefix, lightning on bitcoin mainnet
* `20m`: amount (20 milli-bitcoin)
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `p`: payment hash...
* `h`: tagged field: hash of description
  * `p5`: `data_length` (`p` = 1, `5` = 20; 1 * 32 + 20 == 52)
  * `8yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqs`: SHA256 of 'One piece of chocolate cake, one icecream cone, one pickle, one slice of swiss cheese, one slice of salami, one lollypop, one piece of cherry pie, one sausage, one cupcake, and one slice of watermelon'
* `cc6gd6ql3jrc5yzme8v4ntcewwz5cnw92tz0pc8qcuufvq7khhr8wpald05e92xw006sq94mg8v2ndf4sefvf9sygkshp5zfem29trqq`: signature
* `2yxxz7`: Bech32 checksum
* Signature breakdown:
  * `c63486e81f8c878a105bc9d959af1973854c4dc552c4f0e0e0c7389603d6bdc67707bf6be992a8ce7bf50016bb41d8a9b5358652c4960445a170d049ced4558c` hex of signature data (32-byte r, 32-byte s)
  * `0` (int) recovery flag contained in `signature`
  * `6c6e626332306d0b25fe64410d00004080c1014181c20240004080c1014181c20240004080c1014181c202404082e1a1c92db7b3f161a001b7689049eea2701b46f8db7513629edf2408fac7eaedc60800` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `b6025e8a10539dddbcbe6840a9650707ae3f147b8dcfda338561ada710508916` hex of SHA256 of the preimage

> ### The same, on testnet, with a fallback address mk2QpYatsKicvFVuTAQLBryyccRXMUaGHP
> lntb20m1pvjluezhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqfpp3x9et2e20v6pu37c5d9vax37wxq72un98kmzzhznpurw9sgl2v0nklu2g4d0keph5t7tj9tcqd8rexnd07ux4uv2cjvcqwaxgj7v4uwn5wmypjd5n69z2xm3xgksg28nwht7f6zspwp3f9t

Breakdown:

* `lntb`: prefix, lightning on bitcoin testnet
* `20m`: amount (20 milli-bitcoin)
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `h`: tagged field: hash of description...
* `p`: payment hash...
* `f`: tagged field: fallback address
  * `pp`: `data_length` (`p` = 1; 1 * 32 + 1 == 33)
  * `3` = 17, so P2PKH address
  * `x9et2e20v6pu37c5d9vax37wxq72un98`: 160 bit P2PKH address
* `kmzzhznpurw9sgl2v0nklu2g4d0keph5t7tj9tcqd8rexnd07ux4uv2cjvcqwaxgj7v4uwn5wmypjd5n69z2xm3xgksg28nwht7f6zsp`: signature
* `wp3f9t`: Bech32 checksum
* Signature breakdown:
  * `b6c42b8a61e0dc5823ea63e76ff148ab5f6c86f45f9722af0069c7934daff70d5e315893300774c897995e3a7476c8193693d144a36e2645a0851e6ebafc9d0a` hex of signature data (32-byte r, 32-byte s)
  * `1` (int) recovery flag contained in `signature`
  * `6c6e746232306d0b25fe64570d0e496dbd9f8b0d000dbb44824f751380da37c6dba89b14f6f92047d63f576e304021a000081018202830384048000810182028303840480008101820283038404808102421898b95ab2a7b341e47d8a34ace9a3e7181e5726538` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `00c17b39642becc064615ef196a6cc0cce262f1d8dde7b3c23694aeeda473abe` hex of SHA256 of the preimage

> ### On mainnet, with fallback address 1RustyRX2oai4EYYDpQGWvEL62BBGqN9T with extra routing info to go via nodes 029e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255 then 039e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255
> lnbc20m1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqsfpp3qjmp7lwpagxun9pygexvgpjdc4jdj85fr9yq20q82gphp2nflc7jtzrcazrra7wwgzxqc8u7754cdlpfrmccae92qgzqvzq2ps8pqqqqqqpqqqqq9qqqvpeuqafqxu92d8lr6fvg0r5gv0heeeqgcrqlnm6jhphu9y00rrhy4grqszsvpcgpy9qqqqqqgqqqqq7qqzqj9n4evl6mr5aj9f58zp6fyjzup6ywn3x6sk8akg5v4tgn2q8g4fhx05wf6juaxu9760yp46454gpg5mtzgerlzezqcqvjnhjh8z3g2qqdhhwkj

Breakdown:

* `lnbc`: prefix, lightning on bitcoin mainnet
* `20m`: amount (20 milli-bitcoin)
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `p`: payment hash...
* `h`: tagged field: hash of description...
* `f`: tagged field: fallback address
  * `pp`: `data_length` (`p` = 1; 1 * 32 + 1 == 33)
  * `3` = 17, so P2PKH address
  * `qjmp7lwpagxun9pygexvgpjdc4jdj85f`: 160 bit P2PKH address
* `r`: tagged field: route information
  * `9y`: `data_length` (`9` = 5, `y` = 4; 5 * 32 + 4 = 164)
    * `q20q82gphp2nflc7jtzrcazrra7wwgzxqc8u7754cdlpfrmccae92qgzqvzq2ps8pqqqqqqpqqqqq9qqqvpeuqafqxu92d8lr6fvg0r5gv0heeeqgcrqlnm6jhphu9y00rrhy4grqszsvpcgpy9qqqqqqgqqqqq7qqzq`:
      * pubkey: `029e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255`
      * `short_channel_id`: 0102030405060708
      * `fee_base_msat`: 1 millisatoshi
      * `fee_proportional_millionths`: 20
      * `cltv_expiry_delta`: 3
      * pubkey: `039e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255`
      * `short_channel_id`: 030405060708090a
      * `fee_base_msat`: 2 millisatoshi
      * `fee_proportional_millionths`: 30
      * `cltv_expiry_delta`: 4
* `j9n4evl6mr5aj9f58zp6fyjzup6ywn3x6sk8akg5v4tgn2q8g4fhx05wf6juaxu9760yp46454gpg5mtzgerlzezqcqvjnhjh8z3g2qq`: signature
* `dhhwkj`: Bech32 checksum
* Signature breakdown:
  * `91675cb3fad8e9d915343883a49242e074474e26d42c7ed914655689a8074553733e8e4ea5ce9b85f69e40d755a55014536b12323f8b220600c94ef2b9c51428` hex of signature data (32-byte r, 32-byte s)
  * `0` (int) recovery flag contained in `signature`
  * `6c6e626332306d0b25fe64410d00004080c1014181c20240004080c1014181c20240004080c1014181c202404082e1a1c92db7b3f161a001b7689049eea2701b46f8db7513629edf2408fac7eaedc60824218825b0fbee0f506e4ca122326620326e2b26c8f448ca4029e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255010203040506070800000001000000140003039e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255030405060708090a000000020000001e00040` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `ff68246c5ad4b48c90cf8ff3b33b5cea61e62f08d0e67910ffdce1edecade71b` hex of SHA256 of the preimage

> ### On mainnet, with fallback (P2SH) address 3EktnHQD7RiAE6uzMj2ZifT9YgRrkSgzQX
> lnbc20m1pvjluezhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqfppj3a24vwu6r8ejrss3axul8rxldph2q7z9kmrgvr7xlaqm47apw3d48zm203kzcq357a4ls9al2ea73r8jcceyjtya6fu5wzzpe50zrge6ulk4nvjcpxlekvmxl6qcs9j3tz0469gq5g658y

Breakdown:

* `lnbc`: prefix, lightning on bitcoin mainnet
* `20m`: amount (20 milli-bitcoin)
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `h`: tagged field: hash of description...
* `p`: payment hash...
* `f`: tagged field: fallback address
  * `pp`: `data_length` (`p` = 1; 1 * 32 + 1 == 33)
  * `j` = 18, so P2SH address
  * `3a24vwu6r8ejrss3axul8rxldph2q7z9`:  160 bit P2SH address
* `kmrgvr7xlaqm47apw3d48zm203kzcq357a4ls9al2ea73r8jcceyjtya6fu5wzzpe50zrge6ulk4nvjcpxlekvmxl6qcs9j3tz0469gq`: signature
* `5g658y`: Bech32 checksum
* Signature breakdown:
  * `b6c6860fc6ff41bafba1745b538b6a7c6c2c0234f76bf817bf567be88cf2c632492c9dd279470841cd1e21a33ae7ed59b25809bf9b3366fe81881651589f5d15` hex of signature data (32-byte r, 32-byte s)
  * `0` (int) recovery flag contained in `signature`
  * `6c6e626332306d0b25fe64570d0e496dbd9f8b0d000dbb44824f751380da37c6dba89b14f6f92047d63f576e304021a000081018202830384048000810182028303840480008101820283038404808102421947aaab1dcd0cf990e108f4dcf9c66fb437503c228` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `64f1ff500bcc62a1b211cd6db84a1d93d1f77c6a132904465b6ff912420176be` hex of SHA256 of the preimage

> ### On mainnet, with fallback (P2WPKH) address bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4
> lnbc20m1pvjluezhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqfppqw508d6qejxtdg4y5r3zarvary0c5xw7kepvrhrm9s57hejg0p662ur5j5cr03890fa7k2pypgttmh4897d3raaq85a293e9jpuqwl0rnfuwzam7yr8e690nd2ypcq9hlkdwdvycqa0qza8

* `lnbc`: prefix, lightning on bitcoin mainnet
* `20m`: amount (20 milli-bitcoin)
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `h`: tagged field: hash of description...
* `p`: payment hash...
* `f`: tagged field: fallback address
  * `pp`: `data_length` (`p` = 1; 1 * 32 + 1 == 33)
  * `q`: 0, so witness version 0
  * `w508d6qejxtdg4y5r3zarvary0c5xw7k`: 160 bits = P2WPKH.
* `epvrhrm9s57hejg0p662ur5j5cr03890fa7k2pypgttmh4897d3raaq85a293e9jpuqwl0rnfuwzam7yr8e690nd2ypcq9hlkdwdvycq`: signature
* `a0qza8`: Bech32 checksum
* Signature breakdown:
  * `c8583b8f65853d7cc90f0eb4ae0e92a606f89caf4f7d65048142d7bbd4e5f3623ef407a75458e4b20f00efbc734f1c2eefc419f3a2be6d51038016ffb35cd613` hex of signature data (32-byte r, 32-byte s)
  * `0` (int) recovery flag contained in `signature`
  * `6c6e626332306d0b25fe64570d0e496dbd9f8b0d000dbb44824f751380da37c6dba89b14f6f92047d63f576e304021a00008101820283038404800081018202830384048000810182028303840480810242103a8f3b740cc8cb6a2a4a0e22e8d9d191f8a19deb0` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `b3df27aaa01d891cc9de272e7609557bdf4bd6fd836775e4470502f71307b627` hex of SHA256 of the preimage

> ### On mainnet, with fallback (P2WSH) address bc1qrp33g0q5c5txsp9arysrx4k6zdkfs4nce4xj0gdcccefvpysxf3qccfmv3
> lnbc20m1pvjluezhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqfp4qrp33g0q5c5txsp9arysrx4k6zdkfs4nce4xj0gdcccefvpysxf3q28j0v3rwgy9pvjnd48ee2pl8xrpxysd5g44td63g6xcjcu003j3qe8878hluqlvl3km8rm92f5stamd3jw763n3hck0ct7p8wwj463cql26ava

* `lnbc`: prefix, lightning on bitcoin mainnet
* `20m`: amount (20 milli-bitcoin)
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `h`: tagged field: hash of description...
* `p`: payment hash...
* `f`: tagged field: fallback address
  * `p4`: `data_length` (`p` = 1, `4` = 21; 1 * 32 + 21 == 53)
  * `q`: 0, so witness version 0
  * `rp33g0q5c5txsp9arysrx4k6zdkfs4nce4xj0gdcccefvpysxf3q`: 260 bits = P2WSH.
* `28j0v3rwgy9pvjnd48ee2pl8xrpxysd5g44td63g6xcjcu003j3qe8878hluqlvl3km8rm92f5stamd3jw763n3hck0ct7p8wwj463cq`: signature
* `l26ava`: Bech32 checksum
* Signature breakdown:
  * `51e4f6446e410a164a6da9f39507e730c26241b4456ab6ea28d1b12c71ef8ca20c9cfe3dffc07d9f8db671ecaa4d20beedb193bda8ce37c59f85f82773a55d47` hex of signature data (32-byte r, 32-byte s)
  * `0` (int) recovery flag contained in `signature`
  * `6c6e626332306d0b25fe64570d0e496dbd9f8b0d000dbb44824f751380da37c6dba89b14f6f92047d63f576e304021a00008101820283038404800081018202830384048000810182028303840480810243500c318a1e0a628b34025e8c9019ab6d09b64c2b3c66a693d0dc63194b02481931000` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `399a8b167029fda8564fd2e99912236b0b8017e7d17e416ae17307812c92cf42` hex of SHA256 of the preimage

# Authors

[ FIXME: ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).

