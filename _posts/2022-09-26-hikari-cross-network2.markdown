---
title: "光クロスで快適自宅サーバー環境構築 Linuxルーター構築編"
category: "雑記"
tags: ["ネットワーク", "光クロス", "Linux"]
date: 2022-09-26 00:00:00 +09:00

permalink: /posts/20220926-hikari-cross-network2
---

> 大幅にシンプルになった[2024年版の記事](2024-07-21-hikari-cross-systemd-networkd)があります。
> こちらもどうぞ！

[ISP選び編はこちら](20220811-hikari-cross-network1)

こうしてISPを選び、家までNGNのネットワークが来たところで、いよいよルーターの構築をします。  
とはいえ、大した内容ではない上に知見もほとんど存在しないので詳しい方にはきっと退屈です。

## なぜLinuxルーターなのか？

安いし勉強になるからです！  
家に転がってた i7 8700 メモリ 16GB のサブPCに Intel の 10GbE NIC を買って挿してルーターということにしました。  
オーバースペックすぎるんですが NIC 代2万円だけで 10GbE のルーターが構築できました。

## 全体的な構成に関するお詫び

今回の構成では Ubuntu 22.04 を使用しましたが、 **`systemd-networkd` は使用していません**。期待外れですみません…  
通信ができるようになるのを急いでいたのと、フックやDHCPの機能がよくわからなかったためなのですが、ドキュメントを読んだ感じだと `systemd-networkd` をメインに使用することもおそらく可能です。  
来年までにはオーバースペックではないルーター用PCを用意する予定なのでそのときにでもチャレンジしてみようと思います。

## インターフェース構成

ifupdown 化するのにあたり名前変更はしていません。

- `enp1s0f0` WAN 側
- `enp1s0f1` LAN 側
- `mape0@enp1s0f0` MAP-E(固定IP) トンネル

## 事前準備

netplan を削除し、 ifupdown を導入します

```sh
sudo apt update
sudo apt install ifupdown net-tools
sudo apt purge network-manager netplan
```

だったと思います(記憶が怪しい)

## IPv6 を考える

今回実現するのは IPoE + MAP-E(固定IP) による IPv4･IPv6 のデュアルスタック環境です。  
IPv4 については IPv6 のパケットに包んだ上でルーターから出ていくことになるため、 WAN 側では IPv6 のパケットしかやり取りしないことになります。

IPoE では、 終端装置に紐づいたフレッツの契約に紐づいた ISP(VNE) のプレフィクスが割り振られるため、 VNE を変更するかフレッツの契約が変更されるまで同じプレフィクスが割り振られ、 ISP から送られる設定情報にも記載されています。  
そのため本当のところ IP アドレスを取得する必要はないのですが、今回は NGN との接続確立の検知なども兼ねて DHCPv6 を使用しています。

### 流れ

流れとしては、

1. DHCPv6 により WAN 側からプレフィクスを取得する
    - 光クロスではRAは利用できません。RAプロキシも不要です。
2. 取得が完了したらそのプレフィクスを radvd で LAN 側に配布する
3. フィルタリングとかルーティングをこういい感じにする
    - アドレス変換は不要なため、フィルタリングをしなければ本当にルーティングさせるだけです
    - 今回は IPv4 と同時に設定を行うようにしたため後述します

となります。

### DHCPv6 の導入･設定

今回は dhcpcd を使用しました。

```
sudo apt install dhcpcd5
sudo nano /etc/dhcpcd.conf
```

```
duid
noipv4
noipv6rs
waitip 6
ipv6only

# WANのIPv6用 DHCPv6-DP
interface enp1s0f0
  ipv6rs
  iaid 1
  ia_pd 1 enp1s0f1
```

おまじないレベルの内容ですね。  
これで LAN のインターフェースに払い出された IPv6 アドレスがセットされます。

### radvd の導入･設定


```
sudo apt install radvd
sudo nano /etc/radvd.conf
```

```
interface enp1s0f1 {
  AdvOtherConfigFlag on;
  AdvSendAdvert on;
  prefix ::/64 { };
  RDNSS **** { };
};
```

これも同じくおまじないレベルの内容ですね…。  
これで LAN のインターフェースにセットされた IPv6 アドレスの広告が始まります。  
ポイントとしては、 `RDNSS` に DNS のアドレスを設定していることです。

たぶんこれで IPv6 だけでの通信はできるようになったと思います。簡単ですね。

## IPv4 を考える

IPv4 はトンネルを作成する必要がある上、 NAT の設定を行う必要があるため、考えることが増えます。

### 流れ

IPv6 アドレスが割り当てられたのをトリガーとして、

1. インターフェースIDをセットする
2. アップデートURLへリクエストを行い、IPv6アドレスを向こうのサーバーに登録させる
3. IPIPトンネルを作成
4. DHCP サーバーで各端末にローカルIPアドレスを割り当てる
5. iptables の設定を流し込む

を行います。

### トリガーの設定

アドレスの割当をトリガーとするため dhcpcd のフックを設定していきます。  
パスは `/usr/lib/dhcpcd/dhcpcd-hooks/` です。

#### テンプレート

```sh
#!/bin/bash

if [ "$interface" != "enp1s0f0" ]; then
  exit
fi

case "$reason" in
  "REBIND6") 割当時の処理;;
  "STOP6") 終了時の処理;;
esac
```

よくわかってないのですが、こんな感じのテンプレートを作成してみました。  
これで割り当て時と終了時に任意の処理を実行することができます。  
ファイルの名前順に実行されるので、 `90-` とかから始めると安心できそうです。

### インターフェースIDのセット

```sh
ip -6 a add "${new_dhcp6_ia_pd1_prefix1}[インターフェースID]" dev "$interface"
```

ここでIPコマンドを使ってインターフェースIDとなるIPアドレスを追加します。  
これがないとうまくルーティングできずトンネルが確立できない認識だけど、なくても動くかもしれない…。

### アップデートURLへリクエスト

アップデートURLへリクエストを行うことにより、アクセス元のIPアドレスが契約者のものであることを BR に登録します。  
transix(固定IP) も v6プラス(固定IP) も同様のエンドポイントが存在します。  
先述の通り、 IPv6 アドレスは半固定のため基本は不要なのですが、変化した際に面倒なので設定しています。  
ユーザー名･パスワードやエンドポイントについてはISP契約時に渡されるドキュメントに記載があります。

```sh
curl 'http://***.jp/update?user=***&pass=***'
```

ネットで調べているとここで名前解決ができないケースも見られますが、デフォルトで存在しているフックでリゾルバは設定されているはずなので疎通できるはずです。

### IPIP トンネルを作成

ここでトンネルを作成し、 IPv4 での通信を行えるようにします。  
いくつかパターンを貼ってみますが参考にならないかもしれません…。

#### DS-Lite の場合

```sh
ip -6 tunnel add dslite0 mode ip4ip6 remote [BRアドレス] local "${new_dhcp6_ia_pd1_prefix1}[インターフェースID]" dev "$interface"
ip link set dev dslite0 up
ip route add default dev dslite0
```

前回の投稿でも解説した通り、ポートなどの設定は不要です。

#### transix 固定IPの場合

```sh
ip -6 tunnel add ipip0 mode ipip6 remote [BRアドレス] local "${new_dhcp6_ia_pd1_prefix1}[インターフェースID]" dev "$interface"
ip link set dev ipip0 up
ip route add default dev ipip0
ip -4 a add dev ipip0 [グローバルIP]
```

こちらはただの IPIP のためグローバルIPをセットしておきます。

#### MAP-E の場合

ぶっちゃけすごい大変なんですが、[このあたり](https://benedicam-te.blogspot.com/2020/05/map-e-router-by-debian-box-iptables.html)を参考にするといいかも知れません。  
結局IPアドレスが固定な上に使えるポートが少ないだけなので僕は利用していません。

#### MAP-E 固定IPの場合

```sh
ip -6 tunnel add mape0 mode ip4ip6 remote [BRアドレス] local "${new_dhcp6_ia_pd1_prefix1}[インターフェースID]" dev "$interface"
ip link set dev mape0 up
ip route add default dev mape0
ip -4 a add dev mape0 [固定IP]
```

現行僕が利用している設定です。

### DHCP サーバーで各端末にローカルIPアドレスを割り当てる

[とても参考になる記事](https://qiita.com/iedred7584/items/2bb0d4f7424857eb9f4c)です。以上です。

さすがにアレなので僕の設定を貼っておきます。

```
option domain-name "ingen.work";
max-lease-time 7200;

subnet 10.0.0.0 netmask 255.255.0.0 {
  range 10.0.0.2 10.0.0.99;
  option routers 10.0.0.1;
}
```

まあ普通ですね。  
特徴といえばDNSリゾルバのアドレスを配布していないことで、RAで配布した IPv6 のリゾルバを利用してもらっています。  
なので何も設定していなければうちの環境では IPv4 だけではインターネットに接続できないということになりますね。

#### デフォルトゲートウェイとしてのIPアドレスを設定する

このままではルーターのIPアドレスが存在しないため、 DHCP で配布したIPアドレスに合わせ設定します。 `/etc/network/interfaces` に追記するだけです。  
とはいえさっきのフックでなんとかすればいいだけなので本当は ifupdown も不要になりますね(？)

```
auto lo
iface lo inet loopback

auto enp1s0f1
iface enp1s0f1 inet static
 address 10.0.100.1/16
iface enp1s0f1 inet static
 address 10.0.0.1/16
```

`10.0.100.1` に関しては従来のフレッツ光ネクストと同一のLAN内で運用していた時期があり、その際に使用していたIPアドレスです。

### NAT の設定を行う

iptables を使用することにしました。 nftables よくわかんない… iptables の内部で使われてるからいいよね…。

`iptables-restore` を使用して例のフックで読み込みます。

```
iptables-restore < /etc/iptables.up.rules
ip6tables-restore < /etc/ip6tables.up.rules
```

#### IPv6

なんかいい感じにブロックしつつ RA のパケットなど、必要なものを通すようにします。  
結構沼ったので検索していい感じだったものを使用しています。(参考元ページ紛失)

#### IPv4

NAPT の設定などを行います。  
最低限の構成としてはこんな感じでしょうか。実際に動かしているものではないのでちゃんと機能するかはわかりませんが。

```sh
*nat
:PREROUTING ACCEPT
:INPUT ACCEPT
:POSTROUTING ACCEPT
:OUTPUT ACCEPT
-A POSTROUTING -o mape0 -s 10.0.0.0/16 -j MASQUERADE
COMMIT

*filter
:INPUT DROP
:FORWARD DROP
:OUTPUT ACCEPT

-A INPUT -i lo -j ACCEPT
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A INPUT -i enp1s0f1 -j ACCEPT
-A INPUT -p icmp -j ACCEPT

-A FORWARD -i enp1s0f1 -o mape0 -s 10.0.0.0/16 -j ACCEPT
-A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# block outgoing local packets
-A OUTPUT -o mape0 -d 10.0.0.0/8 -j DROP
-A OUTPUT -o mape0 -d 176.16.0.0/12 -j DROP
-A OUTPUT -o mape0 -d 192.168.0.0/16 -j DROP
-A OUTPUT -o mape0 -d 127.0.0.0/8 -j DROP
COMMIT
```

### ポート開放をする

IPv4 のポート開放の例です。  
結構詰まったので参考になれば幸いです。

`[転送先]` にはサーバーのローカルIPアドレスを入れます。(例: `10.0.0.100`)

#### *nat

```sh
-A PREROUTING -d [グローバルIP] -p tcp -m multiport --dports [カンマ区切りポート] -j DNAT --to-destination [転送先]
```

**内外問わず**、ルーターのグローバルIPに向けた特定のポートへの接続を特定のホストに転送します。

```sh
-A POSTROUTING -o enp1s0f1 -s 10.0.0.0/16 -d [転送先] -p tcp -m multiport --dports [カンマ区切りポート] -j SNAT --to-source [グローバルIP]
```

**内部からの**特定のポートへの接続のソースを自身のグローバルIPアドレスにします。  
これにより、内部からルーターのグローバルIPに向けて接続を行った際、転送先から内部に直接応答してしまいコネクションが確立できないといったことを防ぎます。(ヘアピン NAT を実現します)

#### *filter

```sh
-A FORWARD -o enp1s0f1 -p tcp -d [転送先] -m multiport --dports [カンマ区切りポート] -j ACCEPT
```

上記の設定で宛先を変化させた転送を許可します。

ポートを開放する際は基本この3つを増やす形になります(間違ってたら教えてください)。  
スクリプトにまとめて簡単に設定を追加できるようにできるといいですね。

## さいごに

ありがとうございました。多分これで全部です。  
抜けている所があればちょくちょく追記していきたいと思います。  
眠い…。
