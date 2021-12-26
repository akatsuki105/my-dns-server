# 1 - DNSプロトコルについて

まず、DNSプロトコルについて調べ、その知識をもとに簡単なクライアントを実装してみます。

従来、DNSのパケットはUDPトランスポートを使用して送信され、512バイトに制限されていました。後述しますが、この2つのルールには例外があります。DNSはTCPでも利用でき、eDNSという仕組みを使えばパケットサイズを拡張することができます。しかし、ここではオリジナルの仕様に従うことにします。

DNSは、クエリとレスポンスが同じフォーマットで行われるという点で非常に便利です。つまり、パケットパーサとパケットライタを1つ書いてしまえば、プロトコルの実装は完了してしまうのです。これは、リクエストとレスポンスの構造が異なるほとんどのインターネットプロトコルとは異なります。DNSのパケットは次のようになっています。

| セクション            | サイズ     | Type              | 目的                                                                                                |
| ------------------ | -------- | ----------------- | ------------------------------------------------------------------------------------------------------ |
| Header             | 12 Bytes | Header            | query/responseに関する情報                                                                  |
| Questionセクション   | 可変 | Questions\[\] | 実際には、クエリ名（ドメイン）と関心のあるレコードタイプを示す1つの`Question`のみです。 |
| Answerセクション     | 可変 | Records\[\]  | The relevant records of the requested type.                                                            |
| Authorityセクション  | 可変 | Records\[\]  | ネームサーバー（NSレコード）のリストで、クエリを再帰的に解決するために使用されます。 |
| Additionalセクション | 可変 | Records\[\]  | 役に立ちそうな追加のレコード。例えば、NSレコードに対応するAレコードなどです。 |

基本的には、`Header`,`Question`,`Record`という、3つの異なるオブジェクトをサポートする必要があります。

## ヘッダ(`Header`)のフォーマット

便利なことに、クエスチョン(`Question`)とレコード(`Record`)のリストは、単に個々のインスタンスが一列に追加されたもので、余分なものはありません。各セクションのレコードの数は、ヘッダ(`Header`)によって提供されます。ヘッダ構造は以下のようになります。

| RFC名 | 名称 | サイズ | 説明 |
| -------- | -------------------- | ------------------ | ------------------- |
| ID       | Packet Identifier    | 16 bits | クエリパケットにはランダムな識別子が割り当てられます。レスポンスパケットは同じIDで応答しなければなりません。これは、UDPのステートレスな性質のため、応答を区別するために必要です。       |
| QR       | Query Response       | 1 bit   | 0:クエリ, 1:レスポンス |
| OPCODE   | Operation Code       | 4 bits  | 常に0です。詳細は RFC1035 を参照してください。 |
| AA       | Authoritative Answer | 1 bit   | Set to 1 if the responding server is authoritative - that is, it "owns" - the domain queried. |
| TC       | Truncated Message    | 1 bit   | メッセージの長さが512バイトを超える場合、1に設定されます。従来は、長さの制限が適用されないTCPを使用してクエリを再発行できることを示唆していました。 |
| RD       | Recursion Desired    | 1 bit   | answer が見つからない場合に、サーバーが再帰的にクエリの解決を試みるかどうかを、リクエストの送信者が設定します。 |
| RA       | Recursion Available  | 1 bit   | 再帰的なクエリを許可するかどうかを示すためにサーバーが設定します。 |
| Z        | Reserved             | 3 bits  | 元々は後で使用するために予約されていましたが、現在はDNSSECのクエリに使用されています。 |
| RCODE    | Response Code        | 4 bits  | レスポンスのステータスを示すためにサーバーが設定します。つまり、レスポンスが成功したか失敗したかを示し、失敗した場合は失敗の原因に関する詳細を提供します。 |
| QDCOUNT  | Question Count       | 16 bits | Questionセクションのエントリ数 |
| ANCOUNT  | Answer Count         | 16 bits | Answerセクションのエントリ数 |
| NSCOUNT  | Authority Count      | 16 bits | Authorityセクションのエントリ数 |
| ARCOUNT  | Additional Count     | 16 bits | Additionalセクションのエントリ数 |

## クエスチョン(`Question`)のフォーマット

`Question`の構造はヘッダよりもシンプルなものになっています。

| フィールド名  | Type           | 説明                                                          |
| ------ | -------------- | -------------------------------------------------------------------- |
| Name   | Label Sequence | ドメイン名。単純な1つの文字列でなく、後述のようにラベルのシーケンスとしてエンコードされます。 |
| Type   | 2-byte Integer | レコードのタイプ                                                   |
| Class  | 2-byte Integer | クラスを表すもので、常に1が設定されています。                              |

厄介なのはドメイン名のエンコーディングで、これについては後ほど説明します。

## レコード(`Record`)のフォーマット

最後に、プロトコルの核となるレコード(`Record`)を紹介します。レコードには多くの種類がありますが、ここではいくつかの重要なものだけを考えます。すべてのレコードには次のようなプリアンブルがあります。

| フィールド名  | Type           | 説明                                                                       |
| ------ | -------------- | --------------------------------------------------------------------------------- |
| Name   | Label Sequence | ドメイン名。単純な1つの文字列でなく、後述のようにラベルのシーケンスとしてエンコードされます。              |
| Type   | 2-byte Integer | レコードのタイプ                                                                  |
| Class  | 2-byte Integer | Classを表すもので、常に1が設定されています。                                           |
| TTL    | 4-byte Integer | レコードがキャッシュされてからリクエリされるまでの時間を表します。 |
| Len    | 2-byte Integer | レコードタイプごとに特有のデータの長さ。                                          |

## 実際に見てみよう

これで、特定のレコードタイプを見る準備が整いました。最も重要な**Aレコード**から始めましょう。

| フィールド名      | Type            | 説明                                        |
| ---------- | --------------- | -------------------------------------------------- |
| Preamble   | Record Preamble | プリアンブル(上述)で、`Len`フィールドは4に設定されています。|
| IP         | 4-byte Integer  | 4バイト整数としてエンコードされたIPアドレス               |

IPが4バイトなので、プリアンブルの`Len`フィールドが4になっています。

ここまではいいとして、実際に`dig`ツールを使って検索してみましょう。

```sh
$ dig +noedns google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +noedns google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 36383
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;google.com.                    IN      A

;; ANSWER SECTION:
google.com.             204     IN      A       172.217.18.142

;; Query time: 0 msec
;; SERVER: 192.168.1.1#53(192.168.1.1)
;; WHEN: Wed Jul 06 13:24:19 CEST 2016
;; MSG SIZE  rcvd: 44
```

オリジナルのフォーマットにこだわるために、`+noedns`フラグを使用しています。上の出力にはいくつかの注意すべき点があります。

- `dig`では、レスポンスパケットのヘッダー、クエスチョン、レスポンスセクションが明示されていることがわかります。
- ヘッダーは0に対応する`OPCODE QUERY`を使用しています。ステータス（`RESCODE`）は`NOERROR`に設定されており、数値的には0になります。IDは36383で、クエリを繰り返すことでランダムに変化します。Query Response（qr）、Recursion Desired（rd）、Recursion Available（ra）フラグが有効になっており、数値的には1になっています。adはDNSSECに関連しているので、今は無視しても大丈夫です。最後に、ヘッダーは1つのクエスチョンと1つのレスポンスのレコードがあることを示しています。
- クエスチョンセクションでは、クエスチョンが表示されます。`IN`はクラスを示し、`A`はAレコードに対するクエリを実行していることを示しています。
- アンサーセクションには、googleのIPを含むアンサーレコードが含まれています。`204`はTTL、`IN`は再びクラス、`A`はレコードタイプを表しています。最後に、google.comのIPアドレスを取得しています。
- 最後の行では、パケットの合計サイズが44バイトであることを示しています。

しかし、まだ見ていない情報があるので、さらに深く掘り下げて、パケットをHexでダンプしたものを見てみましょう。

`netcat`を使ってポートをリッスンし、`dig`にクエリを送信するよう指示することができます。ターミナルウィンドウで次のように実行します。

```sh
$ nc -u -l 1053 > query_packet.txt
```

別のターミナルウィンドウで`dig`を実行すると次のようになります。

```sh
$ dig +retry=0 -p 1053 @127.0.0.1 +noedns google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +retry=0 -p 1053 @127.0.0.1 +noedns google.com
; (1 server found)
;; global options: +cmd
;; connection timed out; no servers could be reached
```

`dig`はレスポンスを受け取らないとタイムアウトするので、この場合は失敗が予想されます。

実際、これは失敗するので、終了します。

この時点で`netcat`は`Ctrl+C`で終了しても大丈夫です。なぜなら`query_packet.txt`には、`dig`からのクエリパケットが残ります。

このクエリパケットを使って、レスポンスパケットを次のように記録することができます。

```sh
$ nc -u 8.8.8.8 53 < query_packet.txt > response_packet.txt
```

1秒待ってから、`Ctrl+C`でキャンセルします。これで、パケットを検査する準備ができました。

```sh
$ hexdump -C query_packet.txt
00000000  86 2a 01 20 00 01 00 00  00 00 00 00 06 67 6f 6f  |.*. .........goo|
00000010  67 6c 65 03 63 6f 6d 00  00 01 00 01              |gle.com.....|
0000001c

$ hexdump -C response_packet.txt
00000000  86 2a 81 80 00 01 00 01  00 00 00 00 06 67 6f 6f  |.*...........goo|
00000010  67 6c 65 03 63 6f 6d 00  00 01 00 01 c0 0c 00 01  |gle.com.........|
00000020  00 01 00 00 01 25 00 04  d8 3a d3 8e              |.....%...:..|
0000002c
```

このことを理解できるかどうか見てみましょう。

先ほどから、ヘッダーは12バイトであることがわかっています。クエリパケットの場合、ヘッダのバイトは次のようになります: `86 2a 01 20 00 01 00 00 00 00 00 00`

末尾の8バイトは各セクションの長さに対応しており、実際にコンテンツを持っているのは、1つのエントリを持つクエスチョンセクションだけであることがわかります。

さらに興味深いのは、最初の4バイトで、これはヘッダのさまざまなフィールドに対応しています。

まず、2バイトのIDがありますが、これはクエリパケットとレスポンスパケットの両方で同じであるはずです。実際、この例では、クエリとレスポンスの両方のHexダンプで`86 2a`に設定されていることがわかります。

解析が難しいのは、残りの2バイトです。

ビットごとに意味を持っているため、理解するためには、バイナリ表記に変換する必要があります。クエリパケットの`01 20`から始めると、（最上位ビットから順に）次のようになります。

```
0 0 0 0 0 0 0 1  0 0 1 0 0 0 0 0
- -+-+-+- - - -  - -+-+- -+-+-+-
Q    O    A T R  R   Z      R
R    P    A C D  A          C
     C                      O
     O                      D
     D                      E
     E
```

`Z`セクションの`DNSSEC`関連のビットを除いて、これは予想通りです。

`QR`はクエリなので0となり、`OPCODE`も標準的なルックアップなので0です。

`AA`,`TC`,`RA`のフラグは`RD`がセットされている間はクエリには関係ありません。それは`dig`はデフォルトで再帰的ルックアップを要求するからです。最後に、`RCODE`もクエリには使用されません。

次にレスポンスパケットの後半2バイト(`81 80`)を見ていきましょう。

```
1 0 0 0 0 0 0 1  1 0 0 0 0 0 0 0
- -+-+-+- - - -  - -+-+- -+-+-+-
Q    O    A T R  R   Z      R
R    P    A C D  A          C
     C                      O
     O                      D
     D                      E
     E
```

これはレスポンスなので`QR`がセットされており、サーバーが再帰をサポートしていることを示すために`RA`が設定されています。

レスポンスの残りの8バイトを見ると、1つのクエスチョンに加えて、1つのアンサーレコードを持っていることがわかります。

ヘッダーの直後にはクエスチョンがあります。バイトごとに分解してみましょう。

```
                    query name              type   class
       -----------------------------------  -----  -----
HEX    06 67 6f 6f 67 6c 65 03 63 6f 6d 00  00 01  00 01
ASCII     g  o  o  g  l  e     c  o  m
DEC    6                    3           0       1      1
```

先ほどの表で説明したように、クエリ名(ドメイン名)、タイプ、クラスの3つの部分で構成されています。

しかし、名前のエンコード方法には興味深い点があり、ドットがありません。

DNSは各名前を`labels`という文字列の配列にエンコードし、各ラベルの前にはその長さを示す1バイトが付けられます。

上の例では、"google"は6バイトなので`0x06`が、"com"は3バイトなので`0x03`が先頭に付けられています。`["google", "com"]` 

最後に、すべての名前は、長さが0のラベル、つまりヌルバイトで締めくくられます。 簡単そうに見えますよね？ しかし、すぐに分かるように、これには変わった点があります。

クエリパケットはこれで終わりですが、レスポンスパケットにはデコードすべきデータが残っています。残っているデータは、google.com の対応するIPアドレスを示す1つのAレコードです。

```
      name     type   class         ttl        len      ip
      ------  ------  ------  --------------  ------  --------------
HEX   c0  0c  00  01  00  01  00  00  01  25  00  04  d8  3a  d3  8e
DEC   192 12    1       1           293         4     216 58  211 142
```

ほとんどが予想通りです。

TypeはAレコードを表す`1`、クラスは`IN`を表す`1`となっています。

この場合のTTLは293で正常な値であり、データの長さは4で先程の説明通りです。

そして最後にgoogleのIPは`216.58.211.142`であることがわかりました。

では、名前欄には何が起こっているのでしょうか？ 先ほど学んだラベルはどこにあるのでしょうか？

1つのパケットが512バイトというDNSの元々のサイズ制限のため、何らかの圧縮が必要でした。必要な容量のほとんどはドメイン名のためのものであり、同じ名前の部分は繰り返し使われる傾向があるため、明らかにスペースを節約できる機会があります。例えば、次のようなDNSクエリを考えてみましょう。

```sh
$ dig @a.root-servers.net com

- snip -

;; AUTHORITY SECTION:
com.                172800  IN  NS      e.gtld-servers.net.
com.                172800  IN  NS      b.gtld-servers.net.
com.                172800  IN  NS      j.gtld-servers.net.
com.                172800  IN  NS      m.gtld-servers.net.
com.                172800  IN  NS      i.gtld-servers.net.
com.                172800  IN  NS      f.gtld-servers.net.
com.                172800  IN  NS      a.gtld-servers.net.
com.                172800  IN  NS      g.gtld-servers.net.
com.                172800  IN  NS      h.gtld-servers.net.
com.                172800  IN  NS      l.gtld-servers.net.
com.                172800  IN  NS      k.gtld-servers.net.
com.                172800  IN  NS      c.gtld-servers.net.
com.                172800  IN  NS      d.gtld-servers.net.

;; ADDITIONAL SECTION:
e.gtld-servers.net. 172800  IN  A       192.12.94.30
b.gtld-servers.net. 172800  IN  A       192.33.14.30
b.gtld-servers.net. 172800  IN  AAAA    2001:503:231d::2:30
j.gtld-servers.net. 172800  IN  A       192.48.79.30
m.gtld-servers.net. 172800  IN  A       192.55.83.30
i.gtld-servers.net. 172800  IN  A       192.43.172.30
f.gtld-servers.net. 172800  IN  A       192.35.51.30
a.gtld-servers.net. 172800  IN  A       192.5.6.30
a.gtld-servers.net. 172800  IN  AAAA    2001:503:a83e::2:30
g.gtld-servers.net. 172800  IN  A       192.42.93.30
h.gtld-servers.net. 172800  IN  A       192.54.112.30
l.gtld-servers.net. 172800  IN  A       192.41.162.30
k.gtld-servers.net. 172800  IN  A       192.52.178.30
c.gtld-servers.net. 172800  IN  A       192.26.92.30
d.gtld-servers.net. 172800  IN  A       192.31.80.30

- snip -
```

ここでは、インターネットのルートサーバのひとつに、`.com`TLDを扱うネームサーバを問い合わせています。(TLD=トップレベルドメイン)

`gtld-servers.net.`が何度も登場していることに注目してください。これを毎回パケットに含めずに一度だけ含めればいいのであればサイズの削減にかなり効果的です。

これを実現する1つの方法は、パケットパーサに別の位置にジャンプして、そこで名前を読み終えるように指示する「ジャンプ指示文」を含めることです。結果的には、それがまさに私たちが見ているレスポンスパケットの内容です。

先ほど、各ラベルの前には1バイトの長さがあると述べました。さらに考慮しなければならないのは、`Len`の2つの最上位ビットが設定されている場合、代わりに長さのバイトの後に2つ目のバイトが続くことが予想されることです。

この2つのバイトを合わせて、2つのMSBを取り除いたものがジャンプ位置を示します。上の例では、`0xC00C`となっています。上位2ビットのビットパターンを16進数で表すと`0xC000`なので、このマスクで2バイトをXORしてアンセットすれば、ジャンプ位置が12だとわかります。`0xC00C ^ 0xC000 = 0x000C = 12`

したがって、パケットのバイト12にジャンプして、そこからドメイン名を読み取るのがここでの挙動になります。

DNSヘッダの長さがたまたま12バイトだったことを思い出すと、パケットのクエスチョン部分が始まるところから読み始めるように指示していることがわかります。

質問はクエリドメイン（この場合は "google.com"）で始まるので、これは理にかなっています。名前の読み取りが終わると、前回の続きから解析を再開し、レコードタイプに進みます。

## `BytePacketBuffer`

これでようやく、実装を始めるのに十分な知識が得られました。まず最初に必要なのは、パケットを操作する便利なメソッドです。ここでは、`BytePacketBuffer`という構造体を使います。

```rust
pub struct BytePacketBuffer {
    pub buf: [u8; 512],
    pub pos: usize,
}

impl BytePacketBuffer {

    /// これにより、パケットの内容を保持するための新鮮なバッファと、現在の位置を追跡するためのフィールドが得られます。
    pub fn new() -> BytePacketBuffer {
        BytePacketBuffer {
            buf: [0; 512],
            pos: 0,
        }
    }

    /// バッファの現在の位置
    fn pos(&self) -> usize {
        self.pos
    }

    /// バッファの位置を特定のステップ数だけ前に進める
    fn step(&mut self, steps: usize) -> Result<()> {
        self.pos += steps;

        Ok(())
    }

    /// バッファの位置を変更する
    fn seek(&mut self, pos: usize) -> Result<()> {
        self.pos = pos;

        Ok(())
    }

    /// バッファを1バイトを読み込んで、位置を1ステップ前進させる
    fn read(&mut self) -> Result<u8> {
        if self.pos >= 512 {
            return Err("End of buffer".into());
        }
        let res = self.buf[self.pos];
        self.pos += 1;

        Ok(res)
    }

    /// バッファの位置を変えずに1バイトを取得する
    fn get(&mut self, pos: usize) -> Result<u8> {
        if pos >= 512 {
            return Err("End of buffer".into());
        }
        Ok(self.buf[pos])
    }

    /// バッファから`len`バイトだけデータを取得する(位置は変えない)
    fn get_range(&mut self, start: usize, len: usize) -> Result<&[u8]> {
        if start + len >= 512 {
            return Err("End of buffer".into());
        }
        Ok(&self.buf[start..start + len as usize])
    }

    /// バッファを2バイトを読み込んで、位置を2ステップ前進させる
    fn read_u16(&mut self) -> Result<u16> {
        let res = ((self.read()? as u16) << 8) | (self.read()? as u16);

        Ok(res)
    }

    /// バッファを4バイトを読み込んで、位置を4ステップ前進させる
    fn read_u32(&mut self) -> Result<u32> {
        let res = ((self.read()? as u32) << 24)
            | ((self.read()? as u32) << 16)
            | ((self.read()? as u32) << 8)
            | ((self.read()? as u32) << 0);

        Ok(res)
    }


    /// クエリ名を読み取ります。
    ///
    /// トリッキーな部分としてラベルを考慮してドメイン名を読んでいます。
    /// つまり、[3]www[6]google[3]com[0]のようにデータを取得して、連結して`www.google.com`として出力します。
    fn read_qname(&mut self, outstr: &mut String) -> Result<()> {
        // ジャンプがあるかもしれないので、構造体の中での位置ではなく、ローカルでの位置を把握します。
        // これにより、この変数を使って現在のクエリ名読み取りの進捗状況を把握しながら、共有位置を現在のqnameを過ぎた地点に移動させることができます。
        let mut pos = self.pos();

        // ジャンプしたかどうかの追跡
        let mut jumped = false;
        let max_jumps = 5;
        let mut jumps_performed = 0;

        // 各ラベルに付加するデリミタです。ドメイン名の先頭にドットは必要ないので、ここでは空にしておき、最初の繰り返しの最後に`.`に設定します。
        let mut delim = "";
        loop {
            // Dnsパケットは信頼されていないデータなので、猜疑心を持つ必要があります。誰かがジャンプ命令にサイクルを入れてパケットを作ることができます。これはそのようなパケットを防ぐためのものです。
            if jumps_performed > max_jumps {
                return Err(format!("Limit of {} jumps exceeded", max_jumps).into());
            }

            // この時点で、私たちは常にラベルの先頭にいます。ラベルは長さのバイトで始まることを思い出してください。
            let len = self.get(pos)?;

            // lenの最上位2ビットがセットされている場合は、パケット内の他のオフセットへのジャンプを表します:
            if (len & 0xC0) == 0xC0 {
                // バッファの位置を、現在のラベルを過ぎた地点に更新します。これ以上、触る必要はありません。
                if !jumped {
                    self.seek(pos + 2)?;
                }

                // 別のバイトを読み、オフセットを計算し、ローカルの位置変数を更新してジャンプを実行します。
                let b2 = self.get(pos + 1)? as u16;
                let offset = (((len as u16) ^ 0xC0) << 8) | b2;
                pos = offset as usize;

                // ジャンプが行われたことを示しておきます。
                jumped = true;
                jumps_performed += 1;

                continue;
            }

            // 基本シナリオでは、1つのラベルを読み取り、それを出力に追加しています。
            else {
                // 1バイト前に移動して、長さのバイトを越えます。
                pos += 1;

                // ドメイン名は長さ0の空のラベルで終端されるので、長さが0であれば終了します。
                if len == 0 {
                    break;
                }

                // 最初にデリミタを出力バッファに追加します。
                outstr.push_str(delim);

                // このラベルの実際のASCIIバイトを抽出して、出力バッファに追加します。
                let str_buffer = self.get_range(pos, len as usize)?;
                outstr.push_str(&String::from_utf8_lossy(str_buffer).to_lowercase());

                delim = ".";

                // ラベルの長さ分だけ前進します。
                pos += len as usize;
            }
        }

        if !jumped {
            self.seek(pos)?;
        }

        Ok(())
    }
}
```

## `ResultCode`

ヘッダに移る前に、`rescode`フィールドの値を表す列挙型を追加します。

```rust
#[derive(Copy, Clone, Debug, PartialEq, Eq)]
pub enum ResultCode {
    NOERROR = 0,
    FORMERR = 1,
    SERVFAIL = 2,
    NXDOMAIN = 3,
    NOTIMP = 4,
    REFUSED = 5,
}

impl ResultCode {
    pub fn from_num(num: u8) -> ResultCode {
        match num {
            1 => ResultCode::FORMERR,
            2 => ResultCode::SERVFAIL,
            3 => ResultCode::NXDOMAIN,
            4 => ResultCode::NOTIMP,
            5 => ResultCode::REFUSED,
            0 | _ => ResultCode::NOERROR,
        }
    }
}
```

## `DnsHeader`

これでヘッダに取り掛かることができます。このように表現します。

```rust
#[derive(Clone, Debug)]
pub struct DnsHeader {
    pub id: u16, // 16 bits

    pub recursion_desired: bool,    // 1 bit
    pub truncated_message: bool,    // 1 bit
    pub authoritative_answer: bool, // 1 bit
    pub opcode: u8,                 // 4 bits
    pub response: bool,             // 1 bit

    pub rescode: ResultCode,       // 4 bits
    pub checking_disabled: bool,   // 1 bit
    pub authed_data: bool,         // 1 bit
    pub z: bool,                   // 1 bit
    pub recursion_available: bool, // 1 bit

    pub questions: u16,             // 16 bits
    pub answers: u16,               // 16 bits
    pub authoritative_entries: u16, // 16 bits
    pub resource_entries: u16,      // 16 bits
}
```

実装には多くのビット操作が必要です。

```rust
impl DnsHeader {
    pub fn new() -> DnsHeader {
        DnsHeader {
            id: 0,

            recursion_desired: false,
            truncated_message: false,
            authoritative_answer: false,
            opcode: 0,
            response: false,

            rescode: ResultCode::NOERROR,
            checking_disabled: false,
            authed_data: false,
            z: false,
            recursion_available: false,

            questions: 0,
            answers: 0,
            authoritative_entries: 0,
            resource_entries: 0,
        }
    }

    pub fn read(&mut self, buffer: &mut BytePacketBuffer) -> Result<()> {
        self.id = buffer.read_u16()?;

        let flags = buffer.read_u16()?;
        let a = (flags >> 8) as u8;
        let b = (flags & 0xFF) as u8;
        self.recursion_desired = (a & (1 << 0)) > 0;
        self.truncated_message = (a & (1 << 1)) > 0;
        self.authoritative_answer = (a & (1 << 2)) > 0;
        self.opcode = (a >> 3) & 0x0F;
        self.response = (a & (1 << 7)) > 0;

        self.rescode = ResultCode::from_num(b & 0x0F);
        self.checking_disabled = (b & (1 << 4)) > 0;
        self.authed_data = (b & (1 << 5)) > 0;
        self.z = (b & (1 << 6)) > 0;
        self.recursion_available = (b & (1 << 7)) > 0;

        self.questions = buffer.read_u16()?;
        self.answers = buffer.read_u16()?;
        self.authoritative_entries = buffer.read_u16()?;
        self.resource_entries = buffer.read_u16()?;

        // Return the constant header size
        Ok(())
    }
}
```

## `QueryType`

パケットのクエスチョン部分に移る前に、クエリされるレコードタイプを表現する方法が必要です。

```rust
#[derive(PartialEq, Eq, Debug, Clone, Hash, Copy)]
pub enum QueryType {
    UNKNOWN(u16),
    A, // 1
}

impl QueryType {
    pub fn to_num(&self) -> u16 {
        match *self {
            QueryType::UNKNOWN(x) => x,
            QueryType::A => 1,
        }
    }

    pub fn from_num(num: u16) -> QueryType {
        match num {
            1 => QueryType::A,
            _ => QueryType::UNKNOWN(num),
        }
    }
}
```

## `DnsQuestion`

この列挙型を使うことで、後から簡単にレコードタイプを追加することができます。さて、質問のエントリーです。

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct DnsQuestion {
    pub name: String,
    pub qtype: QueryType,
}

impl DnsQuestion {
    pub fn new(name: String, qtype: QueryType) -> DnsQuestion {
        DnsQuestion {
            name: name,
            qtype: qtype,
        }
    }

    pub fn read(&mut self, buffer: &mut BytePacketBuffer) -> Result<()> {
        buffer.read_qname(&mut self.name)?;
        self.qtype = QueryType::from_num(buffer.read_u16()?); // qtype
        let _ = buffer.read_u16()?; // class

        Ok(())
    }
}
```

ドメイン名を`BytePacketBuffer`構造体の一部として読み込むという大変な作業を行った結果、非常にコンパクトであることがわかりました。

## `DnsRecord`

もちろん、実際のDNSレコードを表現する方法も必要ですが、ここでも簡単に拡張できるようにenumを使用します。

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, PartialOrd, Ord)]
#[allow(dead_code)]
pub enum DnsRecord {
    UNKNOWN {
        domain: String,
        qtype: u16,
        data_len: u16,
        ttl: u32,
    }, // 0
    A {
        domain: String,
        addr: Ipv4Addr,
        ttl: u32,
    }, // 1
}
```

レコードの種類はたくさんあるので、まだ出会っていないレコードの種類を記録する機能を追加します。

また、この列挙型を使うことで、後から簡単に新しいレコードを追加することができます。`DnsRecord`の実際の実装は次のようになっています。

```rust
impl DnsRecord {
    pub fn read(buffer: &mut BytePacketBuffer) -> Result<DnsRecord> {
        let mut domain = String::new();
        buffer.read_qname(&mut domain)?;

        let qtype_num = buffer.read_u16()?;
        let qtype = QueryType::from_num(qtype_num);
        let _ = buffer.read_u16()?;
        let ttl = buffer.read_u32()?;
        let data_len = buffer.read_u16()?;

        match qtype {
            QueryType::A => {
                let raw_addr = buffer.read_u32()?;
                let addr = Ipv4Addr::new(
                    ((raw_addr >> 24) & 0xFF) as u8,
                    ((raw_addr >> 16) & 0xFF) as u8,
                    ((raw_addr >> 8) & 0xFF) as u8,
                    ((raw_addr >> 0) & 0xFF) as u8,
                );

                Ok(DnsRecord::A {
                    domain: domain,
                    addr: addr,
                    ttl: ttl,
                })
            }
            QueryType::UNKNOWN(_) => {
                buffer.step(data_len as usize)?;

                Ok(DnsRecord::UNKNOWN {
                    domain: domain,
                    qtype: qtype_num,
                    data_len: data_len,
                    ttl: ttl,
                })
            }
        }
    }
}
```

## `DnsPacket`

最後に、`DnsPacket`という構造体にまとめてみましょう。

```rust
#[derive(Clone, Debug)]
pub struct DnsPacket {
    pub header: DnsHeader,
    pub questions: Vec<DnsQuestion>,
    pub answers: Vec<DnsRecord>,
    pub authorities: Vec<DnsRecord>,
    pub resources: Vec<DnsRecord>,
}

impl DnsPacket {
    pub fn new() -> DnsPacket {
        DnsPacket {
            header: DnsHeader::new(),
            questions: Vec::new(),
            answers: Vec::new(),
            authorities: Vec::new(),
            resources: Vec::new(),
        }
    }

    pub fn from_buffer(buffer: &mut BytePacketBuffer) -> Result<DnsPacket> {
        let mut result = DnsPacket::new();
        result.header.read(buffer)?;

        for _ in 0..result.header.questions {
            let mut question = DnsQuestion::new("".to_string(), QueryType::UNKNOWN(0));
            question.read(buffer)?;
            result.questions.push(question);
        }

        for _ in 0..result.header.answers {
            let rec = DnsRecord::read(buffer)?;
            result.answers.push(rec);
        }
        for _ in 0..result.header.authoritative_entries {
            let rec = DnsRecord::read(buffer)?;
            result.authorities.push(rec);
        }
        for _ in 0..result.header.resource_entries {
            let rec = DnsRecord::read(buffer)?;
            result.resources.push(rec);
        }

        Ok(result)
    }
}
```

## まとめ

```rust
fn main() -> Result<()> {
    let mut f = File::open("response_packet.txt")?;
    let mut buffer = BytePacketBuffer::new();
    f.read(&mut buffer.buf)?;

    let packet = DnsPacket::from_buffer(&mut buffer)?;
    println!("{:#?}", packet.header);

    for q in packet.questions {
        println!("{:#?}", q);
    }
    for rec in packet.answers {
        println!("{:#?}", rec);
    }
    for rec in packet.authorities {
        println!("{:#?}", rec);
    }
    for rec in packet.resources {
        println!("{:#?}", rec);
    }

    Ok(())
}
```

実行すると次のような出力が得られます。

```
DnsHeader {
    id: 34346,
    recursion_desired: true,
    truncated_message: false,
    authoritative_answer: false,
    opcode: 0,
    response: true,
    rescode: NOERROR,
    checking_disabled: false,
    authed_data: false,
    z: false,
    recursion_available: true,
    questions: 1,
    answers: 1,
    authoritative_entries: 0,
    resource_entries: 0
}
DnsQuestion {
    name: "google.com",
    qtype: A
}
A {
    domain: "google.com",
    addr: 216.58.211.142,
    ttl: 293
}
```

[次の章](chapter2.md)では、ネットワーク接続を追加します。

