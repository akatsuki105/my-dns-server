# 3 - 新しいレコードタイプをサポートしよう

Let's use our program to do a lookup for ''yahoo.com''.

```rust
let qname = "www.yahoo.com";
```

Running it yields:

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
| 1   | A     | Alias - 名前からIPアドレスへのマッピングです。                    | Preamble + Four bytes for IPv4 adress            |
| 2   | NS    | Name Server - The DNS server address for a domain        | Preamble + Label Sequence                        |
| 5   | CNAME | Canonical Name - 名前から名前へのマッピングです。(エイリアス)                     | Preamble + Label Sequence                        |
| 15  | MX    | Mail eXchange - The host of the mail server for a domain | Preamble + 2-bytes for priority + Label Sequence |
| 28  | AAAA  | IPv6 alias                                               | Premable + Sixteen bytes for IPv6 adress         |

## Extending QueryType with more record types

TODO

