# 5 - 再帰問い合わせ

サーバーは動いていますが、実際にルックアップを実行するために別のサーバーに依存しているのは煩わしく、使い勝手もよくありません。

今こそ、名前が実際にどのように解決されるのか、その詳細を掘り下げる良い機会です。

何の事前情報も知られていない、つまりキャッシュがないと仮定した場合、クエスチョンはまずインターネットの13のルートサーバーの1つに発行されます。

なぜ13台なのでしょうか？ それは、512バイトのDNSパケットに収まる数だからです。（厳密に言えば、14個分のスペースがありますが、若干の余裕を残しています）

インターネット全体を扱うには13台では少なすぎると思われるかもしれません。その通りで、論理的には13台ですが、実際にはもっと多くのサーバーが存在します。詳細は、[こちら](http://www.root-servers.org/)をご覧ください。

どのリゾルバも、事前にこの13台のサーバを知っておく必要があります。これらすべてのサーバーをBind形式で記述したファイルが用意されており、[`named.root`](https://www.internic.net/domain/named.root)と呼ばれています。これらのサーバーにはすべて同じ情報が含まれているので、まずはその中からランダムに1つ選んでみましょう。

`named.root`を見ると、`a.root-servers.net`のIPアドレスが`198.41.0.4`であることがわかります。先にそれを使って、`www.google.com` に対する最初のクエリを行います。

```sh
$ dig +norecurse @198.41.0.4 www.google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +norecurse @198.41.0.4 www.google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 64866
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 13, ADDITIONAL: 16

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.google.com.			IN	A

;; AUTHORITY SECTION:
com.			172800	IN	NS	e.gtld-servers.net.
com.			172800	IN	NS	b.gtld-servers.net.
com.			172800	IN	NS	j.gtld-servers.net.
com.			172800	IN	NS	m.gtld-servers.net.
com.			172800	IN	NS	i.gtld-servers.net.
com.			172800	IN	NS	f.gtld-servers.net.
com.			172800	IN	NS	a.gtld-servers.net.
com.			172800	IN	NS	g.gtld-servers.net.
com.			172800	IN	NS	h.gtld-servers.net.
com.			172800	IN	NS	l.gtld-servers.net.
com.			172800	IN	NS	k.gtld-servers.net.
com.			172800	IN	NS	c.gtld-servers.net.
com.			172800	IN	NS	d.gtld-servers.net.

;; ADDITIONAL SECTION:
e.gtld-servers.net.	172800	IN	A	192.12.94.30
b.gtld-servers.net.	172800	IN	A	192.33.14.30
b.gtld-servers.net.	172800	IN	AAAA	2001:503:231d::2:30
j.gtld-servers.net.	172800	IN	A	192.48.79.30
m.gtld-servers.net.	172800	IN	A	192.55.83.30
i.gtld-servers.net.	172800	IN	A	192.43.172.30
f.gtld-servers.net.	172800	IN	A	192.35.51.30
a.gtld-servers.net.	172800	IN	A	192.5.6.30
a.gtld-servers.net.	172800	IN	AAAA	2001:503:a83e::2:30
g.gtld-servers.net.	172800	IN	A	192.42.93.30
h.gtld-servers.net.	172800	IN	A	192.54.112.30
l.gtld-servers.net.	172800	IN	A	192.41.162.30
k.gtld-servers.net.	172800	IN	A	192.52.178.30
c.gtld-servers.net.	172800	IN	A	192.26.92.30
d.gtld-servers.net.	172800	IN	A	192.31.80.30

;; Query time: 24 msec
;; SERVER: 198.41.0.4#53(198.41.0.4)
;; WHEN: Fri Jul 08 14:09:20 CEST 2016
;; MSG SIZE  rcvd: 531
```

ルートサーバーは `www.google.com` を知りませんが、`com` は知っているので、私たちの返事は次にどこに行けばいいかを教えてくれます。注意すべき点がいくつかあります。

- 私たちは、AuthorityセクションにあるNSレコードのセットを提供されています。NSレコードは、ドメインを扱うネームサーバーの名前を教えてくれます。
- サーバーは、NSレコードに対応するAレコードを渡してくれるので、2回目のルックアップを行う必要はありません。
- 実際には`com`のクエリを実行したわけではなく、`www.google.com`です。しかし、NSレコードはすべて`com`を参照しています。

その結果からサーバーを選んで次に進みましょう。`a.gtld-servers.net`の`192.5.6.30`は、他のものと同様に良いと思われます。

```sh
$ dig +norecurse @192.5.6.30 www.google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +norecurse @192.5.6.30 www.google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16229
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 4, ADDITIONAL: 5

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.google.com.			IN	A

;; AUTHORITY SECTION:
google.com.		172800	IN	NS	ns2.google.com.
google.com.		172800	IN	NS	ns1.google.com.
google.com.		172800	IN	NS	ns3.google.com.
google.com.		172800	IN	NS	ns4.google.com.

;; ADDITIONAL SECTION:
ns2.google.com.		172800	IN	A	216.239.34.10
ns1.google.com.		172800	IN	A	216.239.32.10
ns3.google.com.		172800	IN	A	216.239.36.10
ns4.google.com.		172800	IN	A	216.239.38.10

;; Query time: 114 msec
;; SERVER: 192.5.6.30#53(192.5.6.30)
;; WHEN: Fri Jul 08 14:13:26 CEST 2016
;; MSG SIZE  rcvd: 179
```

まだ `www.google.com` には達していませんが、少なくとも `google.com` のドメインを扱うサーバーのセットができたことになります。`216.239.32.10`にクエリを送信して、もう一度試してみましょう。

```sh
$ dig +norecurse @216.239.32.10 www.google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +norecurse @216.239.32.10 www.google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 20432
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;www.google.com.            IN  A

;; ANSWER SECTION:
www.google.com.     300 IN  A   216.58.211.132

;; Query time: 10 msec
;; SERVER: 216.239.32.10#53(216.239.32.10)
;; WHEN: Fri Jul 08 14:15:11 CEST 2016
;; MSG SIZE  rcvd: 48
```

`www.google.com` のIPを取得できました。

ここまでの流れをおさらいしておきましょう。

- `a.root-servers.net`は、`com`を扱う`a.gtld-servers.net`をチェックするように指示しています。
- `a.gtld-servers.net`は、`google.com`を扱う`ns1.google.com`をチェックするように指示しています。
- `ns1.google.com`から`www.google.com`のIPアドレスを取得しています。

これは典型的な例で、キャッシュがなくてもほとんどのルックアップは3つのステップしか必要ありません。

サブドメイン用のネームサーバを用意し、さらにサブドメイン用のネームサーバを用意することも可能です。しかし、実際には、DNSサーバはキャッシュを保持しており、ほとんどのTLDは事前に知られています。

つまり、ほとんどのクエリでは、サーバーによるルックアップはたかだか2回しか必要なく、大抵の場合は1回か0回です。

## リカーシブルルックアップのために`DnsPacket`を拡張しよう

その前に、`DnsPacket`のユーティリティ関数がいくつか必要になります。

```rust
impl DnsPacket {

    - snip -

    /// パケットからランダムにAレコードを選ぶことができるのは便利です。
    /// 1つの名前に対して複数のIPを取得した場合、どのIPを選ぶかは重要ではありませんので、そのような場合にはランダムに選ぶことができます。
    pub fn get_random_a(&self) -> Option<Ipv4Addr> {
        self.answers
            .iter()
            .filter_map(|record| match record {
                DnsRecord::A { addr, .. } => Some(*addr),
                _ => None,
            })
            .next()
    }

    /// Authorityセクションのすべてのネームサーバのイテレータを返すヘルパー関数で、(domain, host)のタプルとして表されます。
    fn get_ns<'a>(&'a self, qname: &'a str) -> impl Iterator<Item = (&'a str, &'a str)> {
        self.authorities
            .iter()
            // 実際には、これらは常にNSレコードであり、しっかりとしたパッケージになっています。
            // NSレコードを、必要なデータだけを持つタプルに変換して、作業しやすくします。
            .filter_map(|record| match record {
                DnsRecord::NS { domain, host, .. } => Some((domain.as_str(), host.as_str())),
                _ => None,
            })
            // 問い合わせに対して権威のないサーバーを除外します。
            .filter(move |(domain, _)| qname.ends_with(*domain))
    }

    /// ここでは、ネームサーバがNSクエリに応答する際に、対応するAレコードを束ねることが多いことを利用して、可能であればNSレコードの実際のIPを返す関数 を実装しています
    pub fn get_resolved_ns(&self, qname: &str) -> Option<Ipv4Addr> {
        // Authorityセクションのネームサーバのイテレータを取得します
        self.get_ns(qname)
            // 次に、追加セクションで一致するAレコードを探す必要があります。
            // 最初の有効なレコードが欲しいだけなので、一致するレコードのストリームを作ればいいのです。
            .flat_map(|(_, host)| {
                self.resources
                    .iter()
                    // 現在処理しているNSレコードのホストとドメインが一致するAレコードをフィルタリングします。
                    .filter_map(move |record| match record {
                        DnsRecord::A { domain, addr, .. } if domain == host => Some(addr),
                        _ => None,
                    })
            })
            .map(|addr| *addr)
            // 最後に、最初の有効なエントリを選びます。
            .next()
    }

    /// しかし、すべてのネームサーバーがそのように親切なわけではありません。
    /// 場合によっては、追加セクションにAレコードが存在せず、中途半端な状態で再度ルックアップを行わなければならないこともあります。 
    /// このため、適切なネームサーバーのホスト名を返す方法を紹介します。
    pub fn get_unresolved_ns<'a>(&'a self, qname: &'a str) -> Option<&'a str> {
        // Authorityセクションのネームサーバのイテレータを取得します
        self.get_ns(qname)
            .map(|(_, host)| host)
            // 最後に、最初の有効なエントリを選びます。
            .next()
    }

} // End of DnsPacket
```

## リカーシブルルックアップを実装しよう

早速、新しい`recursive_lookup`関数の説明に移ります。

```rust
fn recursive_lookup(qname: &str, qtype: QueryType) -> Result<DnsPacket> {
    // 今のところ、常に *a.root-servers.net* から始めています。
    let mut ns = "198.41.0.4".parse::<Ipv4Addr>().unwrap();

    // 任意の数のステップを要する可能性があるため、拘束力のないループに入ります。
    loop {
        println!("attempting lookup of {:?} {} with ns {}", qtype, qname, ns);

        // 次のステップでは、アクティブサーバーにクエリを送信します。
        let ns_copy = ns;

        let server = (ns_copy, 53);
        let response = lookup(qname, qtype, server)?;

        // Answerセクションに入力があり、エラーがなければ完了です。
        if !response.answers.is_empty() && response.header.rescode == ResultCode::NOERROR {
            return Ok(response);
        }

        // また、`NXDOMAIN`という返事が返ってくることもありますが、これは権威あるネームサーバーが、その名前が存在しないことを伝える手段です。
        if response.header.rescode == ResultCode::NXDOMAIN {
            return Ok(response);
        }

        // そうでない場合は、NSと追加セクションの対応するAレコードに基づいて、新しいネームサーバーを見つけようとします。
        // これが成功すれば、ネームサーバを切り替えてループを再試行します。
        if let Some(new_ns) = response.get_resolved_ns(qname) {
            ns = new_ns;

            continue;
        }

        // そうでない場合は、NSレコードのIPを解決する必要があります。
        // NSレコードが存在しない場合は、最後のサーバーが教えてくれたものを使用します。
        let new_ns_name = match response.get_unresolved_ns(qname) {
            Some(x) => x,
            None => return Ok(response),
        };

        // ここでは、現在のルックアップシーケンスの途中で別のルックアップシーケンスを開始します。
        // うまくいけば、適切なネームサーバーのIPを得ることができます。
        let recursive_response = recursive_lookup(&new_ns_name, QueryType::A)?;

        // 最後に、その結果からランダムなIPを選び、ループを再開します。
        // そのようなレコードがない場合は、最後に得られた結果を再び返します。
        if let Some(new_ns) = recursive_response.get_random_a() {
            ns = new_ns;
        } else {
            return Ok(response);
        }
    }
}
```

また、どのサーバを使用するかを渡す必要があるため、`lookup`関数を少し変更する必要があります。関数シグネチャにパラメータ`server`を追加し、[第4章](chapter4.md)で使用したハードコードされた変数`server`を削除しました。

```rust
fn lookup(qname: &str, qtype: QueryType, server: (Ipv4Addr, u16)) -> Result<DnsPacket> {
```

## リカーシブルルックアップを試してみよう

あとは、`handle_query`関数を`recursive_lookup`を使うように変更するだけです。

```rust
fn handle_query(socket: &UdpSocket) -> Result<()> {

    - snip -

            println!("Received query: {:?}", question);
            if let Ok(result) = recursive_lookup(&question.name, question.qtype) {
                packet.questions.push(question.clone());
                packet.header.rescode = result.header.rescode;

    - snip -

}
```

実行してみましょう。

```sh
$ dig @127.0.0.1 -p 2053 www.google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> @127.0.0.1 -p 2053 www.google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41892
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;www.google.com.			IN	A

;; ANSWER SECTION:
www.google.com.		300	IN	A	216.58.211.132

;; Query time: 76 msec
;; SERVER: 127.0.0.1#2053(127.0.0.1)
;; WHEN: Fri Jul 08 14:31:39 CEST 2016
;; MSG SIZE  rcvd: 62
```

サーバーのウィンドウを見ると、次のようになっています。

```sh
Received query: DnsQuestion { name: "www.google.com", qtype: A }
attempting lookup of A www.google.com with ns 198.41.0.4
attempting lookup of A www.google.com with ns 192.12.94.30
attempting lookup of A www.google.com with ns 216.239.34.10
Answer: A { domain: "www.google.com", addr: 216.58.211.132, ttl: 300 }
```

この結果はこの章の最初に行った手動での作業で得られたものと同じです。これで、ルートサーバのリストからドメインを解決することができました。これで、最適ではないものの、完全に機能するDNSサーバーを手に入れることができました。

もっとうまくやれることはたくさんあります。例えば、このサーバには本当の意味での同時性はありません。TCP経由でクエリを送信することも受信することもできません。また、自分のゾーンをホストしたり、権威サーバーとして機能させたりすることもできません。また、DNSSECがサポートされていないため、悪意のあるサーバーが他人のドメインに関するレコードを返すDNSポイズニング攻撃を受ける可能性があります。

これらの問題の多くは、私自身のプロジェクトである [hermes](https://github.com/EmilHernvall/hermes) で修正されているので、よかったらみていただければ幸いです。

いずれにしても、DNSがどのように機能しているのか、少しでも理解していただけたら幸いです。

