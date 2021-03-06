# BOLT #0: イントロダクションと目次

ようこそ！　これはライトニングの技術の基礎（BOLT）のドキュメントです。このドキュメントは相互協力によりオフチェーンでビットコインを移動するために用いられるレイヤー２プロトコルについて書かれています。レイヤー２プロトコルは、必要に応じてオンチェーントランザクションによる強制力を利用します。

（ライトニングの実装に必要な）いくつかの要件は目立ちません。従って、私達は動機づけや、背後の結果に対して考えることを強調します。短すぎるのはわかっています。紛らわしかったり間違っていたりする箇所があれば、改善にご協力ください。

これはバージョン0です。

1. [ボルト #1](01-messaging.md): プロトコルの基礎
2. [ボルト #2](02-peer-protocol.md): ピアプロトコルとチャネルの管理
3. [ボルト #3](03-transactions.md): ビットコインのトランザクションとスクリプトのフォーマット
4. [ボルト #4](04-onion-routing.md): オニオンルーティングプロトコル
5. [ボルト #5](05-onchain.md): オンチェーントランザクションを扱う際の勧告
7. [ボルト #7](07-routing-gossip.md): P2Pノードとチャネルの発見
8. [ボルト #8](08-transport.md): 暗号化と認証されたトランスポート
9. [ボルト #9](09-features.md): Assigned Feature Flags
10. [ボルト #10](10-dns-bootstrap.md): DNSの立ち上げとノード位置の補助
11. [ボルト #11](11-payment-encoding.md): ライトニング決済の為のインボイスプロトコル

## スパーク: ライトニングの簡単な紹介

ライトニングとは、複数のチャネルによるネットワークを使って、ビットコインの高速な決済を可能にするプロトコルです。

### チャネル

ライトニングは *チャネル* を立ち上げることによって機能します。チャネルとは、二人の参加者が作るいくらかのビットコイン（例：0.1ビットコイン）が入金されたライトニング決済チャネルのことです。チャネルはビットコインネットワークに内包されており、出金には参加者双方の署名が必要です。

まず、それぞれの参加者は全てのビットコイン（例えば 0.1 ビットコイン）を自分に払い戻すトランザクションを保有します。その後それらの資金を分ける新たなビットコイントランザクションに署名します。例えば0.09ビットコインを相手に送る、0.01 ビットコインを相手に送るなどです。そうして以前のビットコイントランザクションを使われていなかったものとして取り消します。

[ボルト #2: チャネルの立ち上げ](02-peer-protocol.md#channel-establishment)により詳しく書かれており、[ボルト #3: トランザクションアウトプットのファンディング](03-transactions.md#funding-transaction-output)にチャネルを作るためのビットコイントランザクションのフォーマットが書かれています。[ボルト #5: 推奨されるオンチェーントランザクションの保有方法](05-onchain.md)にそれぞれの参加者が不賛成か失敗したときに双方が署名した際のトランザクションが消費される必要条件が書かれています。

### 条件付決済

ライトニングチャネルは二人の参加者間の決済だけを許可しますが、チャネル同士を接続し合うことによってネットワークを形成できます。そのネットワークを通じてネットワーク内の全ての参加者に対する決済が可能となります。これは条件付決済という技術を必要とします。条件付支払によって、チャネルに例えば「６時間以内にその秘密を公開すれば、あなたは0.01ビットコインを受け取れる」などの条件を付与することができます。受取人が秘密を提供すれば、ビットコイントランザクションは一つの秘密が不足した条件付決済に置き換えられ、ファンドに受取人のアウトプットが追加されます。

[BOLT #2: HTLC の追加](02-peer-protocol.md#adding-an-htlc-update_add_htlc)に参加者が条件付決済を行うためのコマンドがあり、[ボルト #3: Commitment Transaction](03-transactions.md#commitment-transaction)にビットコイントランザクションの完全なフォーマットがあります。

### フォワーディング

そのような条件付決済は別の参加者に時間制限付きで安全に転送（フォワーディング）することができます。例えば、「５時間以内に秘密を公開すれば、0.01ビットコインが得られる」などです。これにより、仲介者を信頼すること無く、チャネルがネットワークに繋がることを可能としています。

[ボルト #2:HTLCフォワーディング](02-peer-protocol.md#forwarding-htlcs)にフォワーディングペイメントの詳細があり、[ボルト #4: パケット構造](04-onion-routing.md#packet-structure)に支払いの指示がどうやって運ばれるのかが書かれています。

### ネットワークトポロジー

決済をするためには、参加者はどのチャネルを経由するのか知っておく必要があります。参加者は互いに自己のチャネル、ノード構築・更新を伝えあいます。

[ボルト #7: P2Pノードとチャネルの発見](07-routing-gossip.md)に通信プロトコルの詳細があり、[ボルト #10: DNSの立ち上げと補助されたノード位置](10-dns-bootstrap.md)にネットワークの初期構築について書かれています。

### ペイメントインボイス

参加者は支払を作成する為のインボイスを受け取ります。

[ボルト #11: ライトニングペイメントの為のインボイスプロトコル](11-payment-encoding.md)に支払人が事後に支払の成功を証明するための支払の目標と目的のプロトコルがあります。

## 用語集

* *お知らせ*:
   * ゴシップメッセージは *ピアの* 間で *チャネル* や *ノード* の発見を支援するために送られます。

* `chain_hash`:
   * 対象となるブロックチェーンを特定するハッシュ（普通はジェネシスハッシュ）。
     これはいくつかのブロックチェーン上で、 *ノード* に *チャネル* の参照をすることを許可する。
     ノードは自分が知らない`chain_hash`が含まれるメッセージを無視する。
     `bitcoin-cli`と違って、ハッシュは逆にされずに直接使用される。

     ビットコインブロックチェーンのメインチェーンでは、`chain_hash`値は以下でなければならない（16進数エンコード済み）。
     `6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000 `

* *チャネル*:
   * 2つの *ピアの* 間で相互に交換される高速なオフチェーンメソッド。
   ファンドを処理するには、ピアは *コミットメントトランザクション* を更新するための署名を交換しなくてはならない。
   * _チャネルのクローズメソッド： 相互クローズ、取消トランザクションクローズ、一方的クローズ_
   * _関連情報： ルート_

* *トランザクションの終了*:
   * トランザクションは _相互クローズ_ の一部を生成します。クロージングトランザクションは _コミットメントトランザクション_ と似ていますが、保留された支払はありません。
   * _関連： コミットメントトランザクション、ファンディングトランザクション、ペナルティトランザクション_

* *コミット番号*:
   * *コミットメントトランザクション* のための48ビットのインクリメントされるカウンタです。ある *チャネル* 内のそれぞれの  *ピア* のカウンタはそれぞれ独立しており、0から数え始めます。
   * _詳細: コミットメントトランザクション_
   * _関連: クロージングトランザクション、ファンディングトランザクション、ペナルティトランザクション_

* *コミットメント取消秘密鍵*:
   * すべての *コミットメントトランザクション* は、他の *ピア* に全てのアウトプットを即座に消費させることができるユニークなコミットメント取消秘密鍵をもっている。
    この鍵を公開するということは、古いコミットメントトランザクションを取消す方法を提供することと等しい。
    取消をサポートするため、コミットメントトランザクションに対応するそれぞれのアウトプットはコミットメント取消公開鍵を参照している。
   * _詳細: コミットメントトランザクション_
   * _原典: パーコミットメントシークレット_

* *コミットメントトランザクション*:
   * *ファンディングトランザクション* を消費するトランザクション。
   それぞれの *ピア* はこのトランザクションに使われる相手方ピアの署名を保有する。これにより、双方は常に消費可能なコミットメントトランザクションを持つ。
     新たなコミットメントトランザクションが成立すると、古いものは *取り消される* 。
   * _参考: コミットメント番号、コミットメント取消秘密鍵、HTLC、パーコミットメントシークレット_
   * _関連: クロージングトランザクション、ファンディングトランザクション、ペナルティトランザクション_
   * _種類: 取り消されたコミットメントトランザクション_

* *終点ノード*:
   * _起点ノード_ からいくつかの _ホップ_ を経由した決済経路での最後の受取人。チェーンにおける最後の *受取ピア* でもある。
   * _カテゴリ: ノード_
   * _関連: 起点ノード、処理ノード_

* *ファンディングトランザクション*:
   * *チャネル* 上で双方の *ピアに* 支払う取り消しできないオンチェーンのトランザクション。
   双方の同意にのみにより消費可能。
   * _関連: クロージングトランザクション、コミットメントトランザクション、ペナルティトランザクション_

* *ホップ*:
   * ノードのこと。通常は、*起点ノード*と*終点ノード*中間に存在するノードを意味する。
   * _See category: ノード_

* *HTLC*: Hashed Time Locked Contract.
   * 2つの*ピア*間で行われる条件付決済。受取人は署名と*ペイメントプレイメージ*を公開することによって支払いを行う。
     これによって、支払人は支払後の所与の時間内、ペイメントプレイメージを使って支払を作成することにより前の支払を取消すことができる。
   これは*コミットメントトランザクション*として実装されている。
   * _詳細: コミットメントトランザクション_
   * _参考: ペイメントハッシュ、ペイメントプレイメージ_

* *インボイス*: ライトニングネットワークのファンドの要求。もしかすると決済の種類、決済額、有効期限などの情報を含んでいるかもしれない。
       これはビットコイン形式のアドレスというよりは、ライトニングネットワークの決済方法である。

* *It's ok to be odd*:
   * A rule applied to some numeric fields that indicates either optional or
     compulsory support for features. Even numbers indicate that both endpoints
     MUST support the feature in question, while odd numbers indicate
     that the feature MAY be disregarded by the other endpoint.

* *MSAT*:
   * ミリサトシ(1サトシの10のマイナス3乗)。フィールド名によく使われる。

* *相互クローズ*:
   * *チャネル*を協同的にクローズすること。それぞれの*ピア*にファンディングトランザクションを無制限に消費できるアウトプットを
       ブロードキャストすることによって実現される（アウトプットが小さすぎない限りそれは含まれない）。
   * _関連: 取消トランザクションクローズ、一方的クローズ_

* *ノード*:
   * ライトニングネットワークの一部を構成するコンピューター等のデバイス。
   * _関連: ピア_
   * _種類: 終点ノード、ホップ、起点ノード、処理ノード、受取ノード、送付ノード_

* *起点ノード*:
   * _終点ノードに_向けた、いくつかの_ホップ_を経由した決済のパケットの始まりである_ノード_。チェーンにおける最初の_送付ピア_でもある。
   * _カテゴリ: ノード_
   * _関連: 終点ノード、処理ノード_

* *ペイメントハッシュ*:
   * *ペイメントプレイメージ*  のハッシュ。*HTLC*はペイメントハッシュを含んでいる。
   * _詳細: HTLC_
   * _原典: ペイメントプレイメージ_

* *ペイメントプレイメージ*:
   * シークレットを知る唯一の者である最後の受取人が有する支払を受け取ったことの証明。
    最後の受取人はファンドを解放する為にプレイメージを解放する。ペイメントプレイメージは*HTLC*の*ペイメントハッシュ*としてハッシュ化されている。
   * _詳細: HTLC_
   * _派生: ペイメントハッシュ_

* *ピア*:
   * 通信しあっている2つの*ノード*。
      * 2つのピアはチャネルを開く前にゴシップを送り合うかもしれない。
      * 2つのピアはトランザクションを送るためのチャネルを立ち上げるかもしれない。
   * _関連: ノード_

* *ペナルティトランザクション*:
   * コミットメント取消秘密鍵を用いて、全ての取消コミットメントトランザクションのアウトプットを消費するトランザクションのこと。
    ピアは相手が*コミットメント取消秘密鍵*をブロードキャストすることによって「だまそうと」した場合にこれを使う。
   * _関連: クロージングトランザクション、コミットメントトランザクション、ファンディングトランザクション_

* *パーコミットメントシークレット*:
   * すべての*コミットメントトランザクション*デバイスはパーコミットメントシークレットからその鍵を引き出す。
     この鍵は、圧縮された以前の全てのコミットメントに対する一連のパーコミットメントシークレットにする為に生成されたものである。
   * _詳細: コミットメントトランザクション_
   * _原典: コミットメント取消秘密鍵_

* *処理ノード*:
   * 支払経路を構築する為に、起点ノードから開始され、終点ノードに送られるパケットを処理するノード。
     _送付ピア_がパケットを送付し、_受取ピア_がメッセージを受信する為に処理ノードが用いられる。
   * _カテゴリ: ノード_
   * _関連: 終点ノード、起点ノード_

* *受取ノード*:
   * メッセージ受け取る*ノード*。
   * _カテゴリ: ノード_
   * _関連: 送付ノード_

* *受取ピア*:
   * 接続された*ピア*から直接メッセージを受け取る*ノード*。
   * _カテゴリ: ピア_
   * _関連: 送付ピア_

* *取消コミットメントトランザクション*:
   * 新しいコミットメントトランザクションが作成されたために取り消された古い*コミットメントトランザクション*。
   * _カテゴリ: コミットメントトランザクション_

* *取消トランザクションクローズ*:
   * *取消コミットメントトランザクション*をブロードキャストすることにより*チャネル*を無効化してクローズすること。
    *ピア*は*コミットメント取消秘密鍵*を知っているので、*ペナルティトランザクション*を生成可能。
   * _See 関連: 相互クローズ、一方的クローズ_

* *ルート*: 支払を可能にするライトニングネットワーク上の経路。
    *起点ノード*から１以上の*ホップ*を通って*終点ノードに*たどり着く。
  * _関連: チャネル_

* *送付ノード*:
   * メッセージを送付するノード。
   * _カテゴリ: ノード_
   * _関連: 受取ノード_

* *送付ピア*:
   * 接続した*ピア*に直接メッセージを送付する*ノード*。
   * _カテゴリ: ピア_
   * _関連: 受取ピア_.

* *一方的クローズ*:
   * *コミットメントトランザクション*をブロードキャストすることによる非協力的なチャネルのクローズ。
    *相互クローズトランザクション*より大きな（すなわちより非効率な）トランザクションであり、
       コミットメントをブロードキャストした*ピア*は従前に作成した自己のアウトプットの一部にアクセスできない。
   * _関連: 相互クローズ、取消トランザクションクローズ_

## テーマソング

      Why this network could be democratic...
      Numismatic...
      Cryptographic!
      Why it could be released Lightning!
      (Release Lightning!)


      We'll have some timelocked contracts with hashed pubkeys, oh yeah.
      (Keep talking, whoa keep talkin')
      We'll segregate the witness for trustless starts, oh yeah.
      (I'll get the money, I've got to get the money)
      With dynamic onion routes, they'll be shakin' in their boots;
      You know that's just the truth, we'll be scaling through the roof.
      Release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)


      [Chorus:]
      Oh released Lightning, it's better than a debit card..
      (Release Lightning, go release Lightning!)
      With released Lightning, micropayments just ain't hard...
      (Release Lightning, go release Lightning!)
      Then kaboom: we'll hit the moon -- release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)


      We'll have QR codes, and smartphone apps, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      P2P messaging, and passive incomes, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      Outsourced closure watch, gives me feelings in my crotch.
      You'll know it's not a brag when the repo gets a tag:
      Released Lightning.


      [Chorus]
      [Instrumental, ~1m10s]
      [Chorus]
      (Lightning! Lightning! Lightning! Lightning!
       Lightning! Lightning! Lightning! Lightning!)


      C'mon guys, let's get to work!


   -- Anthony Towns <aj@erisian.com.au>

## クレジット

[FIXME: 著者リスト作成]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
このコンテンツは、[Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/)でライセンスされています。
