# 2 - スタブリゾルバを作ろう

DNSパケットの解析に成功したことには少し満足していますが、単にディスクから読み取っただけではあまり意味がありません。

次のステップとして、スタブリゾルバを構築するために使用します。スタブリゾルバとは、再帰検索をサポートしていないDNSクライアントで、再帰検索をサポートしているDNSサーバでのみ動作します。

後に、サーバーを必要としない実際の再帰リゾルバを実装します。

## `BytePacketBuffer` を書き込み用に拡張しよう

クエリを提供するためには、パケットを読むだけではなく、書き込むこともできなければなりません。そのためには、`BytePacketBuffer`を拡張していくつかのメソッドを追加する必要があります。

```rust
impl BytePacketBuffer {

    - snip -

    fn write(&mut self, val: u8) -> Result<()> {
        if self.pos >= 512 {
            return Err("End of buffer".into());
        }
        self.buf[self.pos] = val;
        self.pos += 1;
        Ok(())
    }

    fn write_u8(&mut self, val: u8) -> Result<()> {
        self.write(val)?;

        Ok(())
    }

    fn write_u16(&mut self, val: u16) -> Result<()> {
        self.write((val >> 8) as u8)?;
        self.write((val & 0xFF) as u8)?;

        Ok(())
    }

    fn write_u32(&mut self, val: u32) -> Result<()> {
        self.write(((val >> 24) & 0xFF) as u8)?;
        self.write(((val >> 16) & 0xFF) as u8)?;
        self.write(((val >> 8) & 0xFF) as u8)?;
        self.write(((val >> 0) & 0xFF) as u8)?;

        Ok(())
    }
```

また、クエリ名をラベル形式で書くための関数も必要です。

```rust
    fn write_qname(&mut self, qname: &str) -> Result<()> {
        for label in qname.split('.') {
            let len = label.len();
            if len > 0x3f {
                return Err("Single label exceeds 63 characters of length".into());
            }

            self.write_u8(len as u8)?;
            for b in label.as_bytes() {
                self.write_u8(*b)?;
            }
        }

        self.write_u8(0)?;

        Ok(())
    }

} // End of BytePacketBuffer
```

## `DnsHeader` を書き込み用に拡張しよう

上で定義した新しい関数を使って、プロトコル表現構造体を拡張することができます。`DnsHeader`から始めていきましょう。

```rust
impl DnsHeader {

    - snip -

    pub fn write(&self, buffer: &mut BytePacketBuffer) -> Result<()> {
        buffer.write_u16(self.id)?;

        buffer.write_u8(
            (self.recursion_desired as u8)
                | ((self.truncated_message as u8) << 1)
                | ((self.authoritative_answer as u8) << 2)
                | (self.opcode << 3)
                | ((self.response as u8) << 7) as u8,
        )?;

        buffer.write_u8(
            (self.rescode as u8)
                | ((self.checking_disabled as u8) << 4)
                | ((self.authed_data as u8) << 5)
                | ((self.z as u8) << 6)
                | ((self.recursion_available as u8) << 7),
        )?;

        buffer.write_u16(self.questions)?;
        buffer.write_u16(self.answers)?;
        buffer.write_u16(self.authoritative_entries)?;
        buffer.write_u16(self.resource_entries)?;

        Ok(())
    }

}
```

## `DnsQuestion` を書き込み用に拡張しよう

次は `DnsQuestion` です。

```rust
impl DnsQuestion {

    - snip -

    pub fn write(&self, buffer: &mut BytePacketBuffer) -> Result<()> {
        buffer.write_qname(&self.name)?;

        let typenum = self.qtype.to_num();
        buffer.write_u16(typenum)?;
        buffer.write_u16(1)?;

        Ok(())
    }

}
```

## `DnsRecord` を書き込み用に拡張しよう

`DnsRecord` も今のところ非常にコンパクトにまとまっていますが、最終的には様々なレコードタイプを扱うためにかなりのコードを追加します。

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
            DnsRecord::UNKNOWN { .. } => {
                println!("Skipping record: {:?}", self);
            }
        }

        Ok(buffer.pos() - start_pos)
    }

}
```

## `DnsPacket` を書き込み用に拡張しよう

以上のことを `DnsPacket` でまとめてみました。

```rust
impl DnsPacket {

    - snip -

    pub fn write(&mut self, buffer: &mut BytePacketBuffer) -> Result<()> {
        self.header.questions = self.questions.len() as u16;
        self.header.answers = self.answers.len() as u16;
        self.header.authoritative_entries = self.authorities.len() as u16;
        self.header.resource_entries = self.resources.len() as u16;

        self.header.write(buffer)?;

        for question in &self.questions {
            question.write(buffer)?;
        }
        for rec in &self.answers {
            rec.write(buffer)?;
        }
        for rec in &self.authorities {
            rec.write(buffer)?;
        }
        for rec in &self.resources {
            rec.write(buffer)?;
        }

        Ok(())
    }

}
```

## スタブリゾルバの実装

スタブリゾルバを実装する準備ができました。Rustには便利な`UDPSocket`があり、ほとんどの作業をしてくれます。

```rust
fn main() -> Result<()> {
    // Perform an A query for google.com
    let qname = "google.com";
    let qtype = QueryType::A;

    // Using googles public DNS server
    let server = ("8.8.8.8", 53);

    // Bind a UDP socket to an arbitrary port
    let socket = UdpSocket::bind(("0.0.0.0", 43210))?;

    // Build our query packet. It's important that we remember to set the
    // `recursion_desired` flag. As noted earlier, the packet id is arbitrary.
    let mut packet = DnsPacket::new();

    packet.header.id = 6666;
    packet.header.questions = 1;
    packet.header.recursion_desired = true;
    packet
        .questions
        .push(DnsQuestion::new(qname.to_string(), qtype));

    // Use our new write method to write the packet to a buffer...
    let mut req_buffer = BytePacketBuffer::new();
    packet.write(&mut req_buffer)?;

    // ...and send it off to the server using our socket:
    socket.send_to(&req_buffer.buf[0..req_buffer.pos], server)?;

    // To prepare for receiving the response, we'll create a new `BytePacketBuffer`,
    // and ask the socket to write the response directly into our buffer.
    let mut res_buffer = BytePacketBuffer::new();
    socket.recv_from(&mut res_buffer.buf)?;

    // As per the previous section, `DnsPacket::from_buffer()` is then used to
    // actually parse the packet after which we can print the response.
    let res_packet = DnsPacket::from_buffer(&mut res_buffer)?;
    println!("{:#?}", res_packet.header);

    for q in res_packet.questions {
        println!("{:#?}", q);
    }
    for rec in res_packet.answers {
        println!("{:#?}", rec);
    }
    for rec in res_packet.authorities {
        println!("{:#?}", rec);
    }
    for rec in res_packet.resources {
        println!("{:#?}", rec);
    }

    Ok(())
}
```

実行すると次のような出力が得られます。

```
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
    addr: 216.58.209.110,
    ttl: 79
}
```

[次の章](chapter3.md)では、新たなレコードタイプの実装について説明します。
