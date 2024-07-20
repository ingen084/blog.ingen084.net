---
title: "【24.04LTS版】 systemd-networkd で光クロスのLinuxルーターを作る"
category: "雑記"
tags: ["ネットワーク", "光クロス", "Linux"]
date: 2024-07-21 00:00:00 +09:00
---

タイトルが胡散臭すぎる。

この度めでたくルーターPCの再構築を行い、 Ubuntu Server 22.04 LTS から 24.04 LTS に更新しました。  
[前回の記事](2024-02-04-migrate-to-systemd-networkd) では systemd-networkd への移行を試みましたが、バグのため失敗する…と言った悲しい感じで終了しましたが、今回は無事成功しましたので設定など載せてみようと思います。

## インターフェース構成

環境自体は前回と変わらず

- `enp1s0f0` WAN 側
- `enp1s0f1` LAN 側
- `mape0@enp1s0f0` MAP-E(固定IP) トンネル

です。

## OSのインストール

今回は普通に USB メモリに rufus でインストーラーイメージを焼き行いました。  
ここで SSH の設定もやっておくと楽で良いですね。  
minimum としてインストールするのと、パッケージは更新しておきたいので既存(仮設)NW に接続した状態でセットアップを続けます。  
ここでの設定は cloud-init に書き込まれ、ネットワーク周りは netplan -> systemd-networkd という経路で反映される？ようです。間違っていたら教えてください。

### パッケージのインストールと更新

minimum では冗談抜きに本当に何もないのでいくつか必要なパッケージを入れておきます。  

- `nano` or `vim`
- `ping`
- `traceroute` or `mtr`
- `nftables` or `iptables`

辺りは入れておきましょう。  
あと大抵はパッケージが古いので `apt update && apt upgrade` もお忘れ無きよう。

## ルーター化に向けた準備

インストーラーで LAN の設定を行っていますので、DHCP でアドレスを取得する設定が入っています。  

`/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`

```yaml
network: {config: disabled}
```

無効化しておきましょう。  

`/etc/netplan/50-cloud-init.yaml`

```yaml
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    #ethernets:
    #    enp1s0f0:
    #        dhcp4: true
    #    enp1s0f1:
    #        dhcp4: true
    #    enp3s0:
    #        dhcp4: true
    version: 2
```

とはいえ雑にこれでいいかも。知らんけど。

## systemd-networkd 設定の流し込み

### WAN 側

`/etc/systemd/network/10-enp1s0f0.network`

```ini
[Match]
Name=enp1s0f0

[Network]
DHCP=ipv6
Address={払い出されたprefix}:{インターフェースID}/128
Tunnel=mape0

[DHCPv6]
DUIDType=link-layer
```

DHCPv6 でアドレスを取得するだけです。  
が、MAP-E での通信のためインターフェイスにアドレスを割り当てる必要があります。  
DHCP でわざわざアドレスを拾ってくる意味がまるで無いのでこの辺りスクリプトで良い感じにした方が良いかもしれません。

良いソリューションがあれば教えてください。

### LAN 側

`/etc/systemd/network/11-enp1s0f1.network`

```ini
[Match]
Name=enp1s0f1

[Network]
DHCP=no
Address=10.0.0.1/16
DHCPServer=true
IPv6SendRA=yes
DHCPv6PrefixDelegation=yes
MulticastDNS=true

[DHCPServer]
PoolOffset=2
PoolSize=47
EmitDNS=yes
DNS=10.0.0.1
EmitRouter=yes
```

DHCPv6-PD で取得したアドレスを RA で配布するのと共に(IPv4 の) DHCP サーバーも実行させます。  
DNS が 10.0.0.1 なのは別途 Unbound をセットアップしたからです。

### MAP-E トンネル

`/etc/systemd/network/30-mape0.netdev`

```ini
[NetDev]
Name=mape0
Kind=ip6tnl

[Tunnel]
Mode=ipip6
Local={払い出されたprefix}:{インターフェースID}
Remote={BRアドレス}
EncapsulationLimit=none
```

`/etc/systemd/network/30-mape0.network`

```ini
[Match]
Name=mape0

[Network]
BindCarrier=enp1s0f0
DefaultRouteOnDevice=yes
DHCP=no
IPForward=ipv4
LinkLocalAddressing=no
MulticastDNS=no
LLMNR=no

[Address]
Address={固定IPv4アドレス}/32
```

`enp1s0f0` を親に各種設定を行い、デフォルトルートとしての設定を行います。  
特筆すべき点はあまりないですね。22.04 LTS ではここでトンネルが作成できず苦しんでいましたが 24.04 LTS では難なく作成できました。

ちなみにウチは固定IPアドレスなので難しいことなく設定できていますが、固定IPでない場合

- 割り当てられたプレフィックスからのパブリックIPアドレスの算出
- 利用可能ポートの算出と iptables への設定

が別途必要になるので僕の環境のように脳死で設定できません。  
設定料金を払うと思って固定IPにしましょう(ぉぃ

## iptables の設定

スムーズにトンネルが設定できたとはいえ iptables の設定は自分で流し込む必要があるため `networkd-dispatcher` は今のところ健在です。  
NAT のための内容については[前々回の記事](20220926-hikari-cross-network2#nat-の設定を行う)が参考になると思います。

`/etc/networkd-dispatcher/routable.d/iptables`

```bash
#!/bin/bash

if [ "$IFACE" = "enp1s0f0" ]; then
  ip6tables-restore < /etc/ip6tables.rules
  exit
fi

if [ "$IFACE" = "mape0" ]; then
  iptables-restore < /etc/iptables.rules
  exit
fi
```

これで構築完了です！

## 課題

まれにあると言われる prefix の変更に対応できません。  
ルーターPCの再構築を行うことになった原因も prefix の変更が元で、色々触りすぎてよくわからなくなっていたため思い切って再構築することになりました。

systemd-networkd 上の設定だけでなく iptables の設定にもアドレスで制限している箇所があるためなのですが、これを解決するにはどうにかして

- 払い出された prefix の取得
- そこからの各種アドレスの設定

が必要になります。  
RA でアドレス設定されるときは Token として指定する方法があるようなのですが、man 力が試される…。

まあもしこれをすべて自力で対応するとなるとトンネルの設定などは明らかに自分で設定した方が楽なので、そういう方向にシフトしていってしまうかもしれません。  
また何かしたら記事にしますのでいんげんくんの今後にご期待ください。

## さいごに

課題はあるものの、めっちゃシンプルになった！！！  
ほぼデフォルトのパッケージだけで実現できるのでめちゃくちゃ楽でした。

しかし vlan でネットワークを分離したり複雑な構成になっていくと `systemd-networkd` ではカバーしきれない構成になっていくことが目に見えているので、現状のシンプルな構成はやめて複雑になっていきそうだな…と思います。  
もしかして bird に寄せたりできるのかな…色々試行錯誤してみようと思います。

光クロス導入したいけどルーターが高い…と思ったそこのあなた！おすすめですよ！
