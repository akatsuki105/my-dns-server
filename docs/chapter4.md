# 4 - 簡単なDNSサーバの構築

ここまで来たら、いよいよ実際のサーバーを作ってみましょう。実際のDNSサーバーには2つの種類があります。

- 権威サーバ: 1つまたは複数の「ゾーン」をホストするDNSサーバーです。例えば、google.comというゾーンの権限を持つサーバーは、ns1.google.com、ns2.google.com、ns3.google.com、ns4.google.comとなります。
- キャッシュサーバ: DNSサーバーは、要求されたレコードをすでに知っているかどうかを確認するためにキャッシュをチェックし、知らない場合は再帰検索を行ってレコードを見つけ出すことで、DNSルックアップを処理します。これには、ホームルーターで稼動しているDNSサーバー、ISPがDHCPで割り当てているDNSサーバー、GoogleのパブリックDNSサーバー`8.8.8.8`と`8.8.4.4`が含まれます。

1つのサーバーが両方の役割を果たすことは技術的には可能ですが、実際にはこの2つの役割は相互に排他的であることが一般的です。

これは、パケットヘッダのRD(`Recursion Desired`)とRA(`Recursion Available`)というフラグの意味も説明しています。スタブリゾルバがキャッシュサーバに問い合わせをすると、RDフラグが設定され、サーバはそのような問い合わせを許可しているので、ルックアップを実行してRAフラグを設定した応答を送信します。

これは権威サーバでは機能しません。権威サーバはホストされているゾーンに関連するクエリにのみ応答しますので、RDフラグが設定されているクエリにはエラーレスポンスを送信します。

実際に検証してみましょう。まず、`8.8.8.8`を使って`yahoo.com`を調べてみましょう。

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

これは期待通りの動作です。次に、同じクエリを`google.com`ゾーンをホストしているサーバーの1つに送信してみましょう。

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

レスポンスのステータスが「REFUSED」となっていることに注目してください。

また、クエリにRDフラグが設定されているにもかかわらず、サーバがレスポンスにRAフラグを設定していないことを`dig`が警告しています。しかし、同じサーバを`google.com`にも使用することができます。

TODO
