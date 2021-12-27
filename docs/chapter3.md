# 3 - 新しいレコードタイプをサポートしよう

このプログラムを使って、`yahoo.com`の検索をしてみましょう。

```rust
let qname = "www.yahoo.com";
```

実行すると次のような出力が得られます。

```text
DnsHeader {
    id: 6666,
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
    answers: 3,
    authoritative_entries: 0,
    resource_entries: 0
}
DnsQuestion {
    name: "www.yahoo.com",
    qtype: A
}
UNKNOWN {
    domain: "www.yahoo.com",
    qtype: 5,
    data_len: 15,
    ttl: 259
}
A {
    domain: "fd-fp3.wg1.b.yahoo.com",
    addr: 46.228.47.115,
    ttl: 19
}
A {
    domain: "fd-fp3.wg1.b.yahoo.com",
    addr: 46.228.47.114,
    ttl: 19
}
```

奇妙なことが起こりました。2つのAレコードの他に、UNKNOWNレコードを取得しています。

クエリタイプ5のUNKNOWNレコードは`CNAME`です。DNSのレコードタイプにはかなりの数があり、その多くは実際には使用されていません。ここではいくつかの重要なものを表にしました。

| ID  | Name  | Description                                              | Encoding                                         |
| --- | ----- | -------------------------------------------------------- | ------------------------------------------------ |
| 1   | A     | エイリアス - 名前からIPアドレスへのマッピングです。              | プリアンブル + IPアドレス(v4)を表す4バイト            |
| 2   | NS    | ネームサーバ - どのDNSサーバーがそのドメインの権威サーバーであるか | プリアンブル + Label Sequence                        |
| 5   | CNAME | Canonical Name - 名前から名前へのマッピングです。(エイリアス)   | プリアンブル + Label Sequence                        |
| 15  | MX    | Mail eXchange - The host of the mail server for a domain | プリアンブル + 2-bytes for priority + Label Sequence |
| 28  | AAAA  | IPv6エイリアス(AレコードのIPv6版)                            | プリアンブル + IPアドレス(v6)を表す16バイト         |

## `QueryType` を拡張して新たなクエリタイプをサポートしよう

早速、コードに追加してみましょう。まず、QueryType enumを更新します。

```rust
#[derive(PartialEq, Eq, Debug, Clone, Hash, Copy)]
pub enum QueryType {
    UNKNOWN(u16),
    A,     // 1
    NS,    // 2
    CNAME, // 5
    MX,    // 15
    AAAA,  // 28
}
```

また、ユーティリティ関数も変更する必要があります。

```rust
impl QueryType {
    pub fn to_num(&self) -> u16 {
        match *self {
            QueryType::UNKNOWN(x) => x,
            QueryType::A => 1,
            QueryType::NS => 2,
            QueryType::CNAME => 5,
            QueryType::MX => 15,
            QueryType::AAAA => 28,
        }
    }

    pub fn from_num(num: u16) -> QueryType {
        match num {
            1 => QueryType::A,
            2 => QueryType::NS,
            5 => QueryType::CNAME,
            15 => QueryType::MX,
            28 => QueryType::AAAA,
            _ => QueryType::UNKNOWN(num),
        }
    }
}
```

## `DnsRecord` に新たなクエリタイプの読み込みをサポートさせよう

では、これらのレコードのデータを保持する方法が必要なので、DnsRecordにいくつかの変更を加えてみましょう。

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
    NS {
        domain: String,
        host: String,
        ttl: u32,
    }, // 2
    CNAME {
        domain: String,
        host: String,
        ttl: u32,
    }, // 5
    MX {
        domain: String,
        priority: u16,
        host: String,
        ttl: u32,
    }, // 15
    AAAA {
        domain: String,
        addr: Ipv6Addr,
        ttl: u32,
    }, // 28
}
```

ここからが仕事の大部分です。レコードを書いたり読んだりする関数を拡張する必要があります。

まず`read`から始めて、レコードの種類ごとにコードを追加していきます。まず最初に、共通のプリアンブルを用意しました。

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
            QueryType::AAAA => {
                let raw_addr1 = buffer.read_u32()?;
                let raw_addr2 = buffer.read_u32()?;
                let raw_addr3 = buffer.read_u32()?;
                let raw_addr4 = buffer.read_u32()?;
                let addr = Ipv6Addr::new(
                    ((raw_addr1 >> 16) & 0xFFFF) as u16,
                    ((raw_addr1 >> 0) & 0xFFFF) as u16,
                    ((raw_addr2 >> 16) & 0xFFFF) as u16,
                    ((raw_addr2 >> 0) & 0xFFFF) as u16,
                    ((raw_addr3 >> 16) & 0xFFFF) as u16,
                    ((raw_addr3 >> 0) & 0xFFFF) as u16,
                    ((raw_addr4 >> 16) & 0xFFFF) as u16,
                    ((raw_addr4 >> 0) & 0xFFFF) as u16,
                );

                Ok(DnsRecord::AAAA {
                    domain: domain,
                    addr: addr,
                    ttl: ttl,
                })
            }
            QueryType::NS => {
                let mut ns = String::new();
                buffer.read_qname(&mut ns)?;

                Ok(DnsRecord::NS {
                    domain: domain,
                    host: ns,
                    ttl: ttl,
                })
            }
            QueryType::CNAME => {
                let mut cname = String::new();
                buffer.read_qname(&mut cname)?;

                Ok(DnsRecord::CNAME {
                    domain: domain,
                    host: cname,
                    ttl: ttl,
                })
            }
            QueryType::MX => {
                let priority = buffer.read_u16()?;
                let mut mx = String::new();
                buffer.read_qname(&mut mx)?;

                Ok(DnsRecord::MX {
                    domain: domain,
                    priority: priority,
                    host: mx,
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

    - snip -
}
```

ちょっとした言葉の羅列ですが、特に複雑な記録があるわけではなく、それらをまとめて見ることで、少し扱いにくい印象を受けます。

## 書き込みメソッドの追加による`BytePacketBuffer`の拡張

レコードの書き込みに移る前に、`BytePacketBuffer`にさらに2つの関数を追加する必要があります。

```rust
impl BytePacketBuffer {

    - snip -

    fn set(&mut self, pos: usize, val: u8) -> Result<()> {
        self.buf[pos] = val;

        Ok(())
    }

    fn set_u16(&mut self, pos: usize, val: u16) -> Result<()> {
        self.set(pos, (val >> 8) as u8)?;
        self.set(pos + 1, (val & 0xFF) as u8)?;

        Ok(())
    }

}
```

レコードのラベルを書くときには、必要なバイト数を事前に知ることができません。

なぜなら、サイズを圧縮するためにジャンプを使用することになるかもしれないからです。これを解決するために、サイズを`0`にして書いておき、戻ってから必要なサイズを記入します。

## `DnsRecord` に新たなクエリタイプの書き込みをサポートさせよう

`DnsRecord::write`を次のように更新します。

```rust
impl DnsRecord {

    - snip -

    pub fn write(&self, buffer: &mut BytePacketBuffer) -> Result<usize> {
        let start_pos = buffer.pos();

        match *self {
            DnsRecord::A {
                ref domain,
                ref addr,
                ttl,
            } => {
                buffer.write_qname(domain)?;
                buffer.write_u16(QueryType::A.to_num())?;
                buffer.write_u16(1)?;
                buffer.write_u32(ttl)?;
                buffer.write_u16(4)?;

                let octets = addr.octets();
                buffer.write_u8(octets[0])?;
                buffer.write_u8(octets[1])?;
                buffer.write_u8(octets[2])?;
                buffer.write_u8(octets[3])?;
            }
            DnsRecord::NS {
                ref domain,
                ref host,
                ttl,
            } => {
                buffer.write_qname(domain)?;
                buffer.write_u16(QueryType::NS.to_num())?;
                buffer.write_u16(1)?;
                buffer.write_u32(ttl)?;

                let pos = buffer.pos();
                buffer.write_u16(0)?;

                buffer.write_qname(host)?;

                let size = buffer.pos() - (pos + 2);
                buffer.set_u16(pos, size as u16)?;
            }
            DnsRecord::CNAME {
                ref domain,
                ref host,
                ttl,
            } => {
                buffer.write_qname(domain)?;
                buffer.write_u16(QueryType::CNAME.to_num())?;
                buffer.write_u16(1)?;
                buffer.write_u32(ttl)?;

                let pos = buffer.pos();
                buffer.write_u16(0)?;

                buffer.write_qname(host)?;

                let size = buffer.pos() - (pos + 2);
                buffer.set_u16(pos, size as u16)?;
            }
            DnsRecord::MX {
                ref domain,
                priority,
                ref host,
                ttl,
            } => {
                buffer.write_qname(domain)?;
                buffer.write_u16(QueryType::MX.to_num())?;
                buffer.write_u16(1)?;
                buffer.write_u32(ttl)?;

                let pos = buffer.pos();
                buffer.write_u16(0)?;

                buffer.write_u16(priority)?;
                buffer.write_qname(host)?;

                let size = buffer.pos() - (pos + 2);
                buffer.set_u16(pos, size as u16)?;
            }
            DnsRecord::AAAA {
                ref domain,
                ref addr,
                ttl,
            } => {
                buffer.write_qname(domain)?;
                buffer.write_u16(QueryType::AAAA.to_num())?;
                buffer.write_u16(1)?;
                buffer.write_u32(ttl)?;
                buffer.write_u16(16)?;

                for octet in &addr.segments() {
                    buffer.write_u16(*octet)?;
                }
            }
            DnsRecord::UNKNOWN { .. } => {
                println!("Skipping record: {:?}", self);
            }
        }

        Ok(buffer.pos() - start_pos)
    }
}
```

今の時点では未使用の部分もあるので、かなり余分なコードですが、サーバーを書いたときに役に立つでしょう。

## 新しいレコードタイプをテストする

再び、`yahoo.com`のクエリを試してみましょう。出力は次のようになります。

```text
DnsHeader {
    id: 6666,
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
    answers: 3,
    authoritative_entries: 0,
    resource_entries: 0
}
DnsQuestion {
    name: "www.yahoo.com",
    qtype: A
}
CNAME {
    domain: "www.yahoo.com",
    host: "fd-fp3.wg1.b.yahoo.com",
    ttl: 3
}
A {
    domain: "fd-fp3.wg1.b.yahoo.com",
    addr: 46.228.47.115,
    ttl: 19
}
A {
    domain: "fd-fp3.wg1.b.yahoo.com",
    addr: 46.228.47.114,
    ttl: 19
}
```

念のため、MXのルックアップもしてみましょう。

```rust
let qname = "yahoo.com";
let qtype = QueryType::MX;
```

```text
- snip -
DnsQuestion {
    name: "yahoo.com",
    qtype: MX
}
MX {
    domain: "yahoo.com",
    priority: 1,
    host: "mta6.am0.yahoodns.net",
    ttl: 1794
}
MX {
    domain: "yahoo.com",
    priority: 1,
    host: "mta7.am0.yahoodns.net",
    ttl: 1794
}
MX {
    domain: "yahoo.com",
    priority: 1,
    host: "mta5.am0.yahoodns.net",
    ttl: 1794
}
```

[次の章](chapter4.md)ではDNSサーバを構築します。

