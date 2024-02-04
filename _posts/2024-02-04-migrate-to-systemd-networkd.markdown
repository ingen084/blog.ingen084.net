---
title: "Linuxルーターを systemd-networkd に移行してみる"
category: "雑記"
tags: ["ネットワーク", "光クロス", "Linux"]
date: 2024-02-05 00:00:00 +09:00
---

明けましておめでとうございます。  
といいながら 2024年も1ヶ月が経過してしまいました。

今回は年末に [光クロスで快適自宅サーバー環境構築 Linuxルーター構築編](/posts/20220926-hikari-cross-network2) で構築した Linux ルーターを systemd-networkd で構成しようとチャレンジしたのでその経過を貼ってみたいと思います。

## netplan ではありません

ちょうど systemd 本を読んだりしていたので、 今回は netplan ではなくsystemd-networkd を直接利用しました。

## パッケージのインストール

`systemd-networkd` は最初から入っていて `systemd` 本体と一緒になっているはずので基本的には不要です。  
ただし、 `NetworkManager` と `netplan`, `cloudinit` あたりは競合してしまうので削除するか無効化しておきましょう。  
あとは多分僕と同じことしてる人は `systemd-networkd` が disable されてるはずなので enable しておきましょう。

## 設定ファイルを書く

前回の記事では `/etc/network/interfaces` に記述していましたが、 `systemd-networkd` では `/etc/systemd/network/` 内にファイルを作ります。  
前回の記事のフック同様ファイル名順に処理されますが、拡張子が `.network` の場合はネットワークの設定、 `.netdev` の場合は VLAN やトンネルのような仮想デバイスの設定になるようです。

### 今回のネットワーク構成

改めて僕の現環境のIF構成をあげてみます。

- `enp1s0f0` WAN 側
- `enp1s0f1` LAN 側
- `mape0@enp1s0f0` MAP-E(固定IP) トンネル
- `vrf10` IPv6 のフルルートが載っている VRF
- `enp1s0f1.100@enp1s0f1` IPv6 インターネット直結の VLAN
- `ip6gre-*@enp1s0f0` BGP 接続のための GRE トンネル

フルルートを受け取って経路を広報しているためいくつかデバイスが増えています。  
細かい話は次の記事に回すとして、大きな構成は変わっていませんが MAP-E に加えて VLAN と VRF、 GRE トンネルが増えていますので、その定義を書くことになります。

### 物理的なネットワークの設定

とりあえず物理的なデバイスである LAN と WAN のネットワーク設定を書きます。  
インターフェイスの名前はデバイスの場所などに従ってついているのですが、がどうやってつくのかとかは systemd 本に書いてあったので読むと良いとおもいます。

> `/etc/systemd/network/10-enp1s0f0.network`

```ini
[Match]
Name=enp1s0f0

[Network]
DHCP=ipv6
Address=固定IPv6アドレス+インターフェースID/127
#Tunnel=mape0
```

`Match` 内で設定対象の NIC を選んで `Network` に内容を書くみたいです。  
細かいことは man に書いてあります(僕は英語読めないのでWeb経由で翻訳使って読んでますが)。  
ここでは DHCP6 を使ってアドレスを取得することと、 MAP-E のインターフェイスIDをセットしたアドレスを割り当てています。  
なんか Token を指定する方法もあったはずなので動的IPの方はその方法も試してみてもいいかもしれません。  
`Tunnel` については今後定義するトンネルの親となるデバイスとして定義しています。(コメントアウトしている理由については後述)

> `/etc/systemd/network/11-enp1s0f1.network`

```ini
[Match]
Name=enp1s0f1

[Network]
DHCP=no
Address=10.0.0.1/16
DHCPServer=false
IPv6SendRA=yes
DHCPv6PrefixDelegation=yes
DNS=***
DNS=***
MulticastDNS=true
VLAN=enp1s0f1.100
```

LAN 側については今回 DHCP(IPv4) のサーバーは DHCP は使用しないため `no` に、 `isc-dhcp-server` を続投したため `DHCPServer` も無効化しています。  
`IPv6SendRA` を yes にすることで `enp1s0f0` の DHCP で取得したアドレスを RA で配布することができます。  
`VLAN` については上記の `Tunnel` と同様後で定義する VLAN デバイスの親となるデバイスとして定義しています。

### 仮想デバイスの定義

#### VRF

> `/etc/systemd/network/20-vrf10.netdev`

```ini
[NetDev]
Name=vrf10
Kind=vrf

[VRF]
TableId=10
```

フルルートを流し込むための VRF(分離されたルートテーブル) を定義しています。  
デバイスの定義では `Kind` でどの種類のデバイスかどうかを定義します。  
この場合、テーブルIDを `10` として定義しています(そのまんま)。

#### VLAN

> `/etc/systemd/network/21-enp1s0f1.100.netdev`

```ini
[NetDev]
Name=enp1s0f1.100
Kind=vlan

[VLAN]
Id=100
```

VLAN デバイスの定義をしています。  
VLAN ID は `100` ということになります(これもマジでそのまんま)。

#### MAP-E

> `/etc/systemd/network/30-mape0.netdev`

```ini
[NetDev]
Name=mape0
Kind=ip6tnl

[Tunnel]
Mode=ipip6
Local=自分のIPアドレス(+インターフェイスID)
Remote=BRアドレス
EncapsulationLimit=none
```

IPv6 上のトンネルは `ip6tnl` で定義します。  
内容を見ると `ip` コマンドで作成したときと大差ないのがわかりますね。

### しかし…

**このトンネルは作成に失敗します！**  
すでに ip コマンドなどでトンネルを作成した場合はあたかも正常に反映できたかのような顔をするのですが、再起動後などトンネルが作成されていない状況では失敗します！めっちゃ罠。  
これはトンネルの設定が悪いとかではなく、GRE トンネルでも同様に失敗することがわかっています。

## というわけで…

完全に乗り換えることはできませんでした！  
Ubuntu Server 22.04 LTS の `systemd-networkd` には不具合があるようで？うまくトンネルが作成できません。  
あたらしい debian では成功しているとのことなので、 Ubuntu 24.04 LTS になったら完全に乗り換えてみることにしたいと思います。

## フックで設定する

dhcpcd のときと同様にフックを元にトンネルを作成するようにしてみます。  
`systemd-networkd` にもフックが存在しており、 `networkd-dispatcher` でフックを実行させることができます。  
無効化している場合はフックが実行されませんので、 `systemctl` コマンドで有効化しておきましょう。

実行されるファイルは `/etc/networkd-dispatcher/` 下の条件ごとのディレクトリに保存します。  
`carrier` `degraded` `dormant` `no-carrier` `off` `routable` がありますが、まあとりあえず今回はコマンドが実行できれば良いので、 `carrier` にしました。 `routable` でも良いように見えるかもしれませんが、 DHCP の更新のたびに実行されてしまうためエラーにならないようにスクリプトを考えないといけません。

> `/etc/networkd-dispatcher/carrier.d/setup-mape`

```sh
#!/bin/bash

if [ "$IFACE" != "enp1s0f0" ]; then
  exit
fi

# 固定IP
curl 'http://****.jp/update?user=****&pass=****'
ip -6 tunnel add mape0 mode ip4ip6 remote BRアドレス local インターフェイスID dev enp1s0f0
ip link set dev mape0 up
ip route add default dev mape0
ip -4 address add 固定IP dev mape0
```

というわけで `enp1s0f0` が利用可能になったときにトンネルの作成を行うようにしました。

## さいごに

結果として dhcpcd と radvd をなくすことができました。  
(ただし VLAN 側の RA のために radvd は続投しています)

が、現状の Ubuntu の LTS ではまだ使い物にならなさそうな感じですね。  
GRE トンネルも全部定義してやりたいのですが、きっと 24.04 では良い感じに使えるようになっているはずなのでそのときにまたチャレンジしてみようと思います。

それはそれとして、上でも述べたように現状 IPv6 のフルルートを貰って経路を広報しているのですが bird の設定にくっっっっっっっっっっっっそ苦労したためそのあたりの知見を残すという目的も含めて次回の記事にしてみたいと思います。  
余裕があれば今月中に…。

おやすみ！(4時半)
