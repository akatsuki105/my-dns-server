# 4 - 簡単なDNSサーバの構築

ここまで来たら、いよいよ実際のサーバーを作ってみましょう。実際のDNSサーバーには2つの種類があります。

- **権威サーバ**: 1つまたは複数のゾーンを管理するDNSサーバーです。例えば、`google.com`というゾーンの権限を持つサーバーは、`ns1.google.com`、`ns2.google.com`、`ns3.google.com`、`ns4.google.com`となります。
- **キャッシュサーバ**: DNSサーバーは、要求されたレコードをすでに知っているかどうかを確認するためにキャッシュをチェックし、知らない場合は再帰検索を行ってレコードを見つけ出すことで、DNSルックアップを処理します。これには、ホームルーターで稼動しているDNSサーバー、ISPがDHCPで割り当てているDNSサーバー、GoogleのパブリックDNSサーバー`8.8.8.8`と`8.8.4.4`が含まれます。

> ゾーン: 自分の管理する範囲内の情報。例えば、権威サーバのns1.google.comが持っている、google.comについての情報(IPアドレスなど)

1つのサーバーが両方の役割を果たすことは技術的には可能ですが、実際にはこの2つの役割は別々のサーバで行われることが一般的です。

これは、パケットヘッダの`RD`(Recursion Desired)と`RA`(Recursion Available)というフラグの意味も説明しています。

スタブリゾルバがキャッシュサーバに問い合わせをすると、RDフラグがセットされ、サーバはそのようなクエリを許可しているので、ルックアップを実行してRAフラグをセットした応答を送信します。

これは権威サーバでは機能しません。権威サーバは管理しているゾーンに関連するクエリにのみ応答しますので、RDフラグがセットされているクエリにはエラーレスポンスを送信します。

実際に検証してみましょう。まず、`8.8.8.8`(キャッシュサーバ)を使って`yahoo.com`を調べてみましょう。

```sh
$ dig @8.8.8.8 yahoo.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +recurse @8.8.8.8 yahoo.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53231
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;yahoo.com.			IN	A

;; ANSWER SECTION:
yahoo.com.		1051	IN	A	98.138.253.109
yahoo.com.		1051	IN	A	98.139.183.24
yahoo.com.		1051	IN	A	206.190.36.45

;; Query time: 1 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Fri Jul 08 11:43:55 CEST 2016
;; MSG SIZE  rcvd: 86
```

これは期待通りの動作です。次に、同じクエリを`google.com`ゾーンを管理しているサーバ(権威サーバ)の1つに送信してみましょう。

```sh
$ dig @ns1.google.com yahoo.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +recurse @ns1.google.com yahoo.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: REFUSED, id: 12034
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;yahoo.com.			IN	A

;; Query time: 10 msec
;; SERVER: 216.239.32.10#53(216.239.32.10)
;; WHEN: Fri Jul 08 11:44:07 CEST 2016
;; MSG SIZE  rcvd: 27
```

レスポンスのステータスが`REFUSED`となっていることに注目してください。

また、クエリにRDフラグがセットされているにもかかわらず、サーバがレスポンスにRAフラグをセットしていないことを`dig`が警告しています。しかし、同じサーバを`google.com`にも使用することができます。

```sh
$ dig @ns1.google.com google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +recurse @ns1.google.com google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 28058
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;google.com.			IN	A

;; ANSWER SECTION:
google.com.		300	IN	A	216.58.211.142

;; Query time: 10 msec
;; SERVER: 216.239.32.10#53(216.239.32.10)
;; WHEN: Fri Jul 08 11:46:27 CEST 2016
;; MSG SIZE  rcvd: 44
```

今回は権威サーバのゾーンに対してのクエリなのでエラーはありません。

しかし、`dig`はまだ再帰機能が利用できないことを警告しています。この場合、`+norecurse`を使って明示的に再帰性の設定を解除すれば、警告は消えます。

```sh
$ dig +norecurse @ns1.google.com google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +norecurse @ns1.google.com google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 15850
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;google.com.			IN	A

;; ANSWER SECTION:
google.com.		300	IN	A	216.58.211.142

;; Query time: 10 msec
;; SERVER: 216.239.32.10#53(216.239.32.10)
;; WHEN: Fri Jul 08 11:47:52 CEST 2016
;; MSG SIZE  rcvd: 44
```

この最後のクエリは、キャッシュサーバが名前を再帰的に解決する際に送信することが期待されるタイプのクエリです。

独自のサーバを書く最初の試みとして、クエリを別のキャッシュサーバ、つまり「DNSプロキシサーバ」に転送するだけのサーバを実装することで、よりシンプルなものにします。

今までの章で、難しい作業のほとんどはすでに終わっているので、むしろすぐにできる作業です。

## ルックアップの処理を別の関数に切り出す

```rust
/// キャッシュサーバに問い合わせを行う
fn lookup(qname: &str, qtype: QueryType) -> Result<DnsPacket> {
    // Forward queries to Google's public DNS
    let server = ("8.8.8.8", 53);

    let socket = UdpSocket::bind(("0.0.0.0", 43210))?;

    let mut packet = DnsPacket::new();

    packet.header.id = 6666;
    packet.header.questions = 1;
    packet.header.recursion_desired = true;
    packet
        .questions
        .push(DnsQuestion::new(qname.to_string(), qtype));

    let mut req_buffer = BytePacketBuffer::new();
    packet.write(&mut req_buffer)?;
    socket.send_to(&req_buffer.buf[0..req_buffer.pos], server)?;

    let mut res_buffer = BytePacketBuffer::new();
    socket.recv_from(&mut res_buffer.buf)?;

    DnsPacket::from_buffer(&mut res_buffer)
}
```

## サーバを実装しよう

それでは、サーバーのコードを書きましょう。まず、いくつかの項目を整理する必要があります。

```rust
/// やってきたパケットを1つを処理(クエリを見て、レスポンスを返す)します
fn handle_query(socket: &UdpSocket) -> Result<()> {
    // ソケットの準備ができたら、さっそくやってきたパケットを読み込んでみましょう。

    // まずパケットを読み込むためのバッファを作成します
    let mut req_buffer = BytePacketBuffer::new();

    // 関数`recv_from`は、提供されたバッファにデータを書き込みます。そして，読み込んだデータの長さと送信元アドレスを返します。
    // 今回は長さには興味がありませんが、後で返信を送るためには送信元アドレスを保存しておく必要があります。
    let (_, src) = socket.recv_from(&mut req_buffer.buf)?;

    // バッファの内容を読み取って、高レベルな構造体`DnsPacket`を生成する
    let mut request = DnsPacket::from_buffer(&mut req_buffer)?;

    // レスポンスとして送るパケットを作成、初期化します
    let mut packet = DnsPacket::new();
    packet.header.id = request.header.id;
    packet.header.recursion_desired = true;
    packet.header.recursion_available = true;
    packet.header.response = true;

    // 通常のケースでは、ちょうど1つのクエスチョンがクエリパケットに存在します。
    if let Some(question) = request.questions.pop() {
        println!("Received query: {:?}", question);

        // 全てのセットアップが完了し、期待通りの結果が得られれば、クエリをターゲットサーバに転送することができます。
        // クエリが失敗する可能性もありますが、その場合にはクライアントにその旨を伝えるために`SERVFAIL`というレスポンスコードが設定されます。
        // 逆に、すべてが計画通りに進んだ場合は、クエスチョンとレスポンスのレコードがレスポンスパケットにコピーされます。
        if let Ok(result) = lookup(&question.name, question.qtype) { // Googleのキャッシュサーバにクエリを転送
            packet.questions.push(question);
            packet.header.rescode = result.header.rescode;

            for rec in result.answers {
                println!("Answer: {:?}", rec);
                packet.answers.push(rec);
            }
            for rec in result.authorities {
                println!("Authority: {:?}", rec);
                packet.authorities.push(rec);
            }
            for rec in result.resources {
                println!("Resource: {:?}", rec);
                packet.resources.push(rec);
            }
        } else {
            packet.header.rescode = ResultCode::SERVFAIL;
        }
    }
    // 任意の送信者からの入力データが信頼できない可能性があることを念頭に置き、クエスチョンが実際に存在するかどうかを確認する必要があります。もしそうでなければ、送信者が何か間違っていることを示すために `FORMERR` を返します。
    else {
        packet.header.rescode = ResultCode::FORMERR;
    }

    // あとはレスポンスをエンコードして送信するだけです。
    let mut res_buffer = BytePacketBuffer::new();
    packet.write(&mut res_buffer)?;

    let len = res_buffer.pos();
    let data = res_buffer.get_range(0, len)?;

    socket.send_to(data, src)?;

    Ok(())
}

fn main() -> Result<()> {
    // ポート2053にUDPソケットをバインド
    let socket = UdpSocket::bind(("0.0.0.0", 2053))?;

    // 今のところ、クエリは順次処理されているので、リクエストを処理するための無限ループが発生しています。
    loop {
        match handle_query(&socket) {
            Ok(_) => {},
            Err(e) => eprintln!("An error occurred: {}", e),
        }
    }
}
```

エラー処理のための`match`イディオムがここで何度も使われていますが、これはリクエストループの終了を何としても避けたいからです。

これは少し冗長で、通常は代わりに`try!`を使用したいところです。残念ながら、ここでは`Result`を返さない`main`関数の中にいるため、この方法は使えません。

これでひと段落です。

では、実際に実行してみましょう。一方のターミナルでサーバを起動し、もう一方のターミナルで`dig`を使って検索を行います。

```sh
$ dig @127.0.0.1 -p 2053 google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> @127.0.0.1 -p 2053 google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 47200
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;google.com.			IN	A

;; ANSWER SECTION:
google.com.		68	IN	A	216.58.211.142

;; Query time: 1 msec
;; SERVER: 127.0.0.1#2053(127.0.0.1)
;; WHEN: Fri Jul 08 12:07:44 CEST 2016
;; MSG SIZE  rcvd: 54
```

サーバーを起動したターミナルを見ると、出力は次のようになります。

```sh
Received query: DnsQuestion { name: "google.com", qtype: A }
Answer: A { domain: "google.com", addr: 216.58.211.142, ttl: 96 }
```

成功です。800行足らずのコードで、複数の異なるレコードタイプのクエリに応答できるDNSサーバーを構築できました。

[次の章](chapter5.md)では、既存のリゾルバへの依存を解消します。
