---
title: "NixOS と systemd-networkd で SoftBank 光 10ギガ の HGW をバイパスする"
description: "SoftBank 光 10ギガ の HGW を NixOS ルーターに置き換える方法。キャプチャした 1 つのパケットから 3 つのトンネルパラメータを読み取り、systemd-networkd で HGW の MAC・DUID・IAID をクローンする手順、そして 40 秒のハングと 4 時間ごとの全断を引き起こした 2 つのバグについて。"
pubDate: "Aug 17 2026"
tags:
 - ルーター
 - NixOS
heroImageId: "821f8dba-079e-4522-b781-88c7f3a58500"
heroImageSource: 'Pixiv'
heroImageSourceUrl: 'https://www.pixiv.net/artworks/137277047'
heroImageAuthor: 'Tinia'
heroImageAuthorUrl: 'https://www.pixiv.net/users/16148853'
---

*以下に登場する個人のアドレス・プレフィックス・MAC はすべて伏せてあります。プレースホルダは `X` または山括弧で示しています。*

## TL;DR

- SoftBank 光 10ギガ は、多くの日本語ブログ記事が言うのとは違い、**DS-Lite でも MAP-E でもありません**。実体は素朴な RFC 2473 の IPv4-in-IPv6 トンネル（`ip6tnl`、Next Header 4）で、**専有**のグローバル IPv4 が割り当てられます。65535 ポートすべてが自分のものです。
- 必要な 3 つのパラメータ（BR アドレス・CE アドレス・グローバル IPv4）は、**キャプチャした 1 つのトンネルパケット**から読み取れます。RADIUS のデコードも、ベンダー辞書も、リバースエンジニアリングも不要です。
- HGW の WAN MAC をクローンし、その DHCPv6 DUID と DHCPv6 IAID を固定する必要があります。MAC をクローンするだけでは足りません。バインディングはこの両方をキーにしています。ここを間違えたせいで、きっかり 4 時間ごとに全断が発生しました。第 9 部を参照してください。
- NixOS では `systemd.network` で実現できます。設定は合計でおよそ 40 行です。
- MSS クランプは任意ではなく必須です。これがないと、IPv4 のみのサイトへの初回接続がすべて 40 秒ハングします。
- HGW はレンタルし続ける必要があります。これはバイパスであって、解約ではありません。

## 背景

私は奈良で SoftBank 光 10ギガ を契約しています（つまり下回りは NTT 西日本、フレッツ光クロスです）。SoftBank から送られてきたのは **ホームゲートウェイ（S）**、型番 `10G E-WMTA1.0` で、中身は Sercomm の **EVO310G** です。これは SoftBank が 2025 年 4 月から配布し始めた新しい一体型の機種で、従来の XG-100NE + 光BBユニット の組み合わせを置き換え、両方の役割を 1 台に統合したものです。

この違いは重要です。というのも既存の解説記事のほぼすべてが XG-100NE を前提にしており、それらの記事にあるテクニックのいくつかは当てはまらないからです。

| | XG-100NE（旧） | ホームゲートウェイ（S）/ EVO310G |
|---|---|---|
| ベンダー | NTT（NEC） | SoftBank（Sercomm） |
| 隠し設定ページ | `http://ntt.setup:8888/t/` | **存在しない** |
| 設定メニュー | `http://ntt.setup/` | `http://192.168.3.1/` |
| 4over6 のプロビジョニング | フレッツ・ジョイント のソフトウェア | 独自の RADIUS クライアント + TFTP |

私のルーターは 4 つのインターフェースを持つ NixOS マシンです。

- `enp1s0f0`、`enp1s0f1` — 10G SFP+
- `enp4s0` — 10G RJ45
- `enp7s0` — 1G RJ45

`enp1s0f1` は `br-lan` にブリッジしています。すべて `systemd.network` で設定し、ワークステーションから `nixos-rebuild --target-host` でデプロイしています。

## 第 1 部: これは実際には何のプロトコルなのか

これは最初に最も時間を無駄にした問いです。ネット上の情報が互いに矛盾しているからです。

**HGW 自身のステータスページには「MAP-E」と表示されます。** だからブロガーたちもそう繰り返します。しかし他の人が公開したパケットキャプチャは素朴な IPIP カプセル化を示しており、これを掘り下げたある研究者は「4rd/SAM」でもないと結論づけました。それは SoftBank が否定している噂です。実体は、JPIX/v6プラス が*固定 IP* 契約で使っているのと同じ RFC 2473 の IPIP トンネルです。

辻褄合わせは簡単です。BBIX は **IPv4 アドレス共有なし**で MAP-E を運用しています。ポートセットの共有がないと、MAP のアルゴリズムは「すべてをカプセル化して BR に送る」に退化します。つまりただのトンネルです。ですから HGW は厳密には嘘をついているわけではなく、MAP-E の面白い部分がオフになっているだけなのです。

DS-Lite は単純に誤りです。経路のどこにも AFTR も CGN もありません。

見つけた中で最も役立った確認は、日本の IPv4-over-IPv6 トンネル向けの OpenWrt ヘルパーである [`luci-app-fleth`](https://github.com/makeding/luci-app-fleth) でした。ここには DS-Lite・MAP-E・固定IP の 3 つのカテゴリがあり、`SoftBank 光`（1G と 10G の両方）を **固定IP** の下に、その `IPIP6H` プロトコルで扱うものとして列挙しています。これは、*MAP-E でも DS-Lite でもない* と明確に述べている、メンテナンスされた対応表です。

知っておく価値がもう一つ。**フレッツ光クロス は PPPoE を一切提供していません。** フォールバック経路はありません。トンネルが動かなければ、手元に残るのは IPv6 だけです。

### 不幸中の幸い

IPv6 は何のトリックもなく素の DHCPv6-PD で問題なく動くので、**トンネルが壊れても IPv6 のインターネットは生きたまま残ります**。`cache.nixos.org` も `github.com` も AAAA レコードを持っています。これは意味のあるセーフティネットになりました。

## 第 2 部: 何を取り出す必要があるか

| 値 | 在り処 |
|---|---|
| グローバル IPv4 | HGW の設定メニュー |
| CE IPv6（トンネルのローカル） | HGW の設定メニュー |
| WAN MAC | 本体のラベル |
| **BR IPv6（トンネルのリモート）** | **どこにもない — キャプチャするしかない** |
| **DHCPv6 DUID** | **どこにもない — キャプチャするしかない** |
| **DHCPv6 IAID** | **どこにもない — キャプチャするしかない** |
| IA_PD の T1/T2/各ライフタイム | 同じキャプチャから — 記録しておく価値あり |

最初の 3 つはタダで手に入ります。残りは、ONU と HGW の間の回線上に自分を割り込ませる必要があります。

**IAID は誰もが見落とすもので、これを飛ばしたせいで 4 時間の全断を 2 回食らいました。** DHCPv6 のバインディングは DUID 単独ではなく **DUID + IAID** をキーにします。DUID を完璧にしても IAID を間違えれば、サーバーはどちらも送らなかったのと全く同じくらい徹底的にあなたを無視します。第 9 部を参照してください。

## 第 3 部: キャプチャ

### トポロジー

当初は、キャプチャ中も家がオンラインのままでいられるよう、空きポートを一時的なアップリンクに使うつもりでした。しかしこう気づきました。**`nixos-rebuild --target-host` はワークステーション側でビルドし、クロージャを LAN 経由で送り込む。ルーターの再設定にインターネットは一切不要なのだ、と。** ワークステーションをスマホにテザリングするだけで十分で、これで両方の RJ45 ポートをタップ用に空けられました。

```
ONU ──────► enp4s0 (10G) ┐
                         ├─ br-tap  (IPなし・STPなし)
HGW WAN ◄── enp7s0 (1G)  ┘

switch ◄─── enp1s0f1 ──── br-lan
```

ONU のケーブルは初日に恒久的な定位置（`enp4s0`）に挿し、それ以降は二度と動かしません。切り替え時には HGW を抜くだけです。ブリッジをまたぐ速度の不一致は問題ありません。HGW の WAN ポートはオートネゴシエーションで速度を落としますし、DHCPv6/RADIUS は数百バイトです。

代償として、その間 LAN はインターネットにつながりません。20 分ほど見ておきましょう。

### タップの設定

```nix
systemd.network.netdevs."05-br-tap" = {
  netdevConfig = { Name = "br-tap"; Kind = "bridge"; };
  bridgeConfig = { STP = false; VLANFiltering = false; };
};

systemd.network.networks."05-tap-members" = {
  matchConfig.Name = "enp4s0 enp7s0";
  networkConfig = { Bridge = "br-tap"; LinkLocalAddressing = "no"; };
  linkConfig.RequiredForOnline = "no";
};

systemd.network.networks."05-br-tap" = {
  matchConfig.Name = "br-tap";
  networkConfig = { DHCP = "no"; LinkLocalAddressing = "no"; IPv6AcceptRA = false; };
  linkConfig.RequiredForOnline = "no";
};

environment.systemPackages = with pkgs; [ tcpdump tshark ethtool ];
boot.blacklistedKernelModules = [ "br_netfilter" ];
```

`br_netfilter` をブラックリストに入れるのは重要です。これがロードされると nftables のルールがブリッジされたフレームを検査し始め、HGW のトラフィックを黙って飲み込んでしまうことがあります。これは見た目がまさに「タップが壊れている」と同じになります。

> **まだ** MAC をクローンしないでください。`enp4s0` がブリッジポートである間は自分自身のアドレスを持たず、フレームを改変せず転送します。これがタップを透過的にしている点です。今クローンすると、同じ MAC を持つ 2 台のデバイスが 1 つのセグメントに存在することになり、ブリッジの FDB を掻き乱します。

### BR アドレスを手に入れる

ここは考えすぎた部分です。誰もが RADIUS の `Access-Accept` を盗聴してベンダー属性 204 と 207 を 16 進デコードする方法を説明しています。その必要はありません。**1 つのトンネルパケットにすべてが入っています。**

```
$ sudo tcpdump -nn -i br-tap -c 20 'ip6 proto 4'

IP6 2400:2000:4:0:a000::XXXX > 2400:2650:XXXX:XXXX:1111:1111:1111:1111: \
    IP 198.51.100.42.44424 > <MY_IPV4>.10000: Flags [S], seq ..., length 0
    ^^^^^^^^^^^^^^^^^^^^^^   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    BR = トンネルの Remote   CE = トンネルの Local     内側: あなたの IPv4
```

HGW の再起動は不要です。このトラフィックは常に流れています。CE アドレスと IPv4 を HGW の設定メニューと突き合わせてください。3 つが一致すれば完了です。

### キャプチャがタダで教えてくれたこと

**IPv4 は本当に専有です。** 私のキャプチャのほとんどは受信側のスキャントラフィックでした。世界中のホストからポート 10000、8181、63500、3128、31038、19998 への SYN です。それ自体は退屈ですが、良い結果でもあります。ポート共有の MAP-E 構成なら、割り当てられたポート範囲に着信するパケットしか見えないはずです。低いポートも高いポートも任意に届くということは、65535 すべてが自分のものだということです。これで、誰かのブログ記事を信用せずとも、自分の回線については MAP-E かどうかの問題が経験的に決着します。

> これはインターネットの背景放射のようなものです。攻撃者があなたのデバイスを踏み台にして他人や、あなたの他のデバイスを攻撃するかもしれません。公開側ではファイアウォールを有効にしておきましょう。

**BR のブロック。** 私のものは `2400:2000:4:0:a000::/64` にあります。他の人はその同じ /64 から `::1919` や `::1999` を公開しています。ホストは違えど同じ BBIX のブロックで、地域ごとの BR という説明と整合します。あなたのものは違うはずなので、自分でキャプチャしてください。

**HGW はトンネル越しに自分の親元へ通信します。** `221.111.x.x` のホストに対する `10gewmta.sc` の TFTP 読み取り要求です。機種の独立した裏付けであり、この機器のプロビジョニングが純粋に RADIUS 主導ではない証拠でもあります。

**スキャナーには DROP ではなく RST で応答します。** これは、ルーターがその IPv4 を引き継いだ瞬間に、活発に探られている公開アドレスも引き継ぐのだという、良い注意喚起になります。

### DHCPv6 のアイデンティティを手に入れる: DUID *と* IAID

こちらは HGW の電源再投入が必要です。DHCPv6 は起動時にしか走らないからです。

```sh
sudo tcpdump -nn -i enp4s0 -w /tmp/sb.pcap \
  'udp port 546 or udp port 547 or udp port 1812 or udp port 1813'
# HGW の電源を入れ直し、完全に起動しきるまで待って Ctrl-C

tshark -r /tmp/sb.pcap -Y 'dhcpv6.msgtype == 1' -V
```

Client Identifier だけを grep しないでください。Solicit 全体をダンプします。ここから 3 つのものが必要です。

**1. DUID:**

```
Client Identifier
    Option: Client Identifier (1)
    Length: 10
    DUID: 00030001982cc6XXXXXX
    DUID Type: link-layer address (3)
    Hardware type: Ethernet (1)
    Link-layer address: 98:2c:c6:XX:XX:XX
    Link-layer address (Ethernet): SernetTechno_XX:XX:XX
Reconfigure Accept
    Option: Reconfigure Accept (20)
```

WAN MAC を包む DUID-LL（type 3）です。`98:2c:c6` は Sercomm の OUI で、EVO310G であることを裏付けます。

**2. IAID:**

```
Identity Association for Prefix Delegation
    Option: Identity Association for Prefix Delegation (25)
    Length: 12
    IAID: 00000001
    T1: 0
    T2: 0
```

`IAID: 00000001` です。systemd-networkd はインターフェース名のハッシュから IAID を導出するため、あなたの値は HGW のものとは無関係な任意の値になります。**サーバーは DUID + IAID でバインディングを引きます。** DUID が完璧でも IAID が合っていなければ、得られるものは何もありません。

また、Solicit が **IA_PD のみ**を運んでいることにも注意してください。IA_NA はありません。これに合わせて `UseAddress = false` にしないと、この回線でサーバーが払い出さないものを要求することになります。

**3. リースのライフタイム**、Reply から取得します。

```sh
tshark -r /tmp/sb.pcap -Y 'dhcpv6.msgtype == 7' -V | grep -A12 'Prefix Delegation'
```

私の場合: T1 7200、T2 10800、preferred 12600、**valid 14400** です。この数字は書き留めておいてください。第 9 部の障害における導火線の長さであり、事前に知っていれば、わけの分からない全断を 5 分の診断に変えてくれます。

### DUID のバイト配置の罠

これは私の人生を 4 時間ずつ 2 回も焼き尽くしたので、独立した見出しを与えます。

systemd-networkd における `DUIDRawData` は、**2 バイトの type フィールドより後ろのペイロード**です。type そのものは networkd が `DUIDType` から導出して書き込みます。ここに完全な DUID を渡すと、networkd は自分の type フィールドをその上に重ねてしまいます。

```
HGW    (Length: 10)  00 03  0001982cc6XXXXXX
Router (Length: 12)  00 03  00030001982cc6XXXXXX      ← 間違い
                     ^^^^^  ^^^^^^^^^^^^^^^^^^^^^^
                     型     DUIDRawData に入れた値
                 networkd が
                 前置する
```

| | バイト |
|---|---|
| 回線上の完全な DUID | `0003` `0001` `982cc6XXXXXX` |
| `DUIDType = "link-layer"` が供給する | `0003` |
| `DUIDRawData` に入れるべき値 | `0001982cc6XXXXXX` — **8 バイト** |

残る `00:01` は **hardware type**（Ethernet）であって、DUID の type の一部ではありません。値のパターンは同じ、フィールドは別、1 バイトペアだけずれている — だからこそ間違えやすく、いったん書いてしまうとほとんど気づけないのです。

最速の確認は、16 進を 1 桁も読む前に長さを見ることです。`Length: 10` なら正解、`Length: 12` ならペイロードが長すぎます。

**なぜ MAC だけでなくこれらすべてが必要なのか。** NTT の DHCPv6-PD サーバーは DUID + IAID をキーにします。systemd-networkd の既定は DUID-EN（ベンダーベース）に名前ハッシュの IAID を組み合わせたもので、どんな MAC を提示しようと `/56` は得られません。DUID は MAC を*含んでいる*ので、MAC・DUID・IAID を固定するのは冗長ではなく整合的です。

その Solicit に `Reconfigure Accept` があることにも注意してください。あとで効いてくるので覚えておきましょう。

何かを撤去する前に、pcap をルーターの外へコピーしておいてください。トンネルが上がらず HGW がもう経路にいない午前 2 時に、きっと欲しくなります。私は自分のものに 3 回も戻りました。

## 第 4 部: NixOS の設定

```nix
{ lib, pkgs, ... }:
let
  wanIf   = "enp4s0";
  hgwMac  = "98:2c:c6:XX:XX:XX";
  # ペイロードのみ — networkd が 2 バイトの type フィールドを自分で前置する。
  # キャプチャした DUID は 00:03:00:01:98:2c:c6:XX:XX:XX; 先頭の 00:03 を落とす。
  hgwDuid = "00:01:98:2c:c6:XX:XX:XX";
  ceAddr  = "2400:2650:XXXX:XXXX:1111:1111:1111:1111";
  brAddr  = "2400:2000:4:0:a000::XXXX";
  myV4    = "<MY_IPV4>";
in
{
  boot.kernelModules = [ "ip6_tunnel" ];

  # WAN: MAC をクローン、DUID を固定、RA + PD を受け取り、トンネルを接続
  systemd.network.networks."10-wan" = {
    matchConfig.Name = wanIf;
    linkConfig.MACAddress = hgwMac;
    networkConfig = {
      IPv6AcceptRA = true;
      DHCP = "ipv6";
      LinkLocalAddressing = "ipv6";
      Tunnel = [ "sbtun" ];
    };
    address = [ "${ceAddr}/64" ];
    dhcpV6Config = {
      DUIDType = "link-layer";
      DUIDRawData = hgwDuid;
      IAID = 1;                 # HGW の Solicit から — networkd の既定ではない
      PrefixDelegationHint = "::/56";
      UseAddress = false;       # HGW は IA_PD のみを要求、IA_NA なし
      UseDNS = false;
      WithoutRA = "solicit";
    };
  };

  # トンネル本体
  systemd.network.netdevs."20-sbtun" = {
    netdevConfig = { Name = "sbtun"; Kind = "ip6tnl"; MTUBytes = "1460"; };
    tunnelConfig = {
      Mode = "ipip6";
      Local = ceAddr;
      Remote = brAddr;
      EncapsulationLimit = "none";
    };
  };

  systemd.network.networks."20-sbtun" = {
    matchConfig.Name = "sbtun";
    address = [ "${myV4}/32" ];
    routes = [ { Destination = "0.0.0.0/0"; Scope = "link"; } ];
  };
}
```

### MAC は `.link` ファイルではなく `.network` の `[Link]` で設定する

これは強調しておく価値があります。`.link` ファイルは **udev がデバイス追加時に**適用するため、`nixos-rebuild switch` では既に存在するインターフェースに再適用されません。`udevadm trigger --action=add` か再起動が必要です。`.network` ファイル内の `[Link] MACAddress=` は networkd 自身が適用し、リロードで反映されます。

「なぜ MAC が変わらないのか」を、アップリンクなしでデバッグするのは悲惨です。古い `.link` ファイルは `.network` の設定を黙って上書きして勝つので、`.link` を使うガイドに従っていたなら削除してください。

MAC が変わってもインターフェース名は変わりません。`enp4s0` はアドレスではなく PCI パスから導出されるからです。

### `EncapsulationLimit = "none"` は任意ではない

既定ではカーネルが、トンネルのカプセル化上限を運ぶ IPv6 Destination Options ヘッダーを付けます。一部の BR はそれを破棄します。`ip -d link show sbtun` で確認してください。`encaplimit none` ではなく `encaplimit 4` と出ていたら、それがバグです。これは「トンネルは上がっているのに何も通らない」の典型的な原因です。

（`ip -d` はモードを `ip4ip6` と報告します。これは `Mode=ipip6` として設定したものと同じで、エラーではありません。）

## 第 5 部: ファイアウォール — 本当に肝心な部分

元々の設定はこうでした。

```nix
networking.firewall = {
  enable = true;
  trustedInterfaces = [ "br-lan" "tailscale0" ];
  checkReversePath = "loose";
};
```

`allowedTCPPorts` がどこにもないので、input チェインは信頼されていないすべてのインターフェースで既定として drop します。Netdata は `0.0.0.0:19999` にバインドされていましたが、到達できるのは LAN と Tailscale からだけでした。これは問題なく持ちこたえました。

切り替え時に変わることが 3 つあります。

**1. `nat.externalInterface` は当然ながら `sbtun` に移す必要があります。** `enp4s0` でマスカレードすると、IPv6 とカプセル化されたフレームしか運んでいないインターフェースに適用されてしまい、LAN のトラフィックは RFC1918 の送信元アドレスのままトンネルを出て消えてしまいます。

**2. 既定の拒否はトンネルを黙って殺します。** カプセル化された戻りトラフィックは next header 4 の IPv6 パケットとして届きますが、netfilter の input フックはカーネルがそれをデカプセル化器に渡す*前*に走ります。drop ポリシーがそれを飲み込みます。`sbtun` は UP と表示され、RX は 0 のままです。プロトコル 4 を明示的に許可する必要があります。

```nix
networking.nftables.enable = true;
networking.firewall.extraInputRules = ''
  # proto 4 = BBIX の BR からの IPv4-in-IPv6（RFC 2473）
  ip6 saddr ${brAddr} meta l4proto 4 accept
'';
```

> **落とし穴:** `meta l4proto ipencap` はコンパイルできません。`Error: Could not resolve protocol name` が出ます。nftables は `l4proto` の名前を、`/etc/protocols` *ではなく*、小さな組み込みテーブル（`tcp`、`udp`、`icmp`、`icmpv6`、`sctp`、`dccp`、`ah`、`esp`、`comp`、`udplite`）から解決します。このリストの外にあるものはすべて数値で書かなければなりません。同じ罠が GRE（47）と 6in4（41）にも当てはまります。

**3. LAN は IPv6 ファイアウォールを完全に失います。** これが本当の露出であり、設定上は見えません。`/56` を委任して RA を送るようになると、すべての LAN デバイスがグローバルにルーティング可能なアドレスを持ちます。そして `networking.firewall` は既定で *input* チェインしかフィルタしません。転送されるトラフィックはフィルタされずに通ります。NAT が偶然何かを守ってくれるということもありません。

私のキャプチャで、いかに多くのスキャナーが既にその IPv4 を叩いていたかを踏まえると、こうします。

```nix
networking.firewall.filterForward = true;
networking.firewall.extraForwardRules = ''
  iifname "br-lan" accept
  ct state established,related accept
'';
```

`checkReversePath = "loose"` のままにしてください。厳格な RPF と、オンリンクのデフォルト経路を持つポイントツーポイントトンネル上の `/32` は相性が悪いのです。

> ファイアウォールは LAN の内側からではなく外側から検証してください
>
> ```sh
> nmap -Pn -p 22,19999,80,443 <MY_IPV4>
> nmap -6 -Pn -p 22,19999 <LAN デバイスのグローバル IPv6>
> ```
>
> 2 つ目こそが本当に肝心なテストで、しかもみんなが飛ばすものです。両方ともスマホのテザリングから実行してください。NAT ループバックは設定するまで存在しないので、内側からテストしても何の証明にもなりません。

## 第 6 部: インターネットなしでのデプロイ

崩す価値のある前提。**ルーターの再ビルドにインターネットは一切不要です。** `nixos-rebuild --target-host` は評価とビルドを完全にワークステーション側で行い、クロージャを LAN 上の SSH 経由で送り込みます。ルーターはただの受け手です。

```sh
# 一度だけ、オンライン中に — 入力を取得してクロージャを事前ビルドする
nix flake archive
nix build .#nixosConfigurations.router.config.system.build.toplevel

# その後はオフラインでも安全
nixos-rebuild switch --flake .#router --target-host root@<router> --fast
```

ネットワークが必要なのは*新しい依存を追加する*ときだけです。設定だけの変更はローカルストアから再ビルドされます。

安全な反復ループ。

```sh
# 1. ブートローダーに触れずに適用する
nixos-rebuild test --flake .#router --target-host root@<router>

# 2. デッドマンスイッチを仕掛ける — 再起動で最後の正常状態に戻る
ssh root@<router> 'systemd-run --on-active=10min --unit=deploy-watchdog systemctl reboot'

# 3. うまくいったら
ssh root@<router> 'systemctl stop deploy-watchdog.timer'
nixos-rebuild switch --flake .#router --target-host root@<router>
```

`test` はブートエントリを更新しないので、再起動が*そのまま*ロールバックになります。書くべきロールバックロジックはありません。手で組みたくなければ、`magicRollback = true` を付けた `deploy-rs` が同じ段取りを自動化してくれます。

ループを大幅に短くするものがあと 2 つあります。

- **トンネルのテストのために再ビルドしないでください。** `ip` コマンド 5 つで済みます。パケットが流れるまでシェルで反復し、*それから* 一度だけ Nix に落とし込みます。
- **リスクのある設定にはスペシャライゼーションを使いましょう。** そうすればブートの既定は正常な状態のままになり、電源の再投入が脱出口になります。

## 第 7 部: 切り替えチェックリスト

HGW を抜く前に:

- `ip6 proto 4` の送信元から BR IPv6 を記録した
- CE IPv6 を**一字一句そのまま**コピーした — インターフェース ID を勝手に作らない
- グローバル IPv4 が HGW の設定メニューと一致している
- WAN MAC を記録した（元に戻すため、焼き込みの MAC も `ethtool -P` で保存した）
- DHCPv6 の Client Identifier のバイトを記録した — **ペイロードは 10 バイトではなく 8 バイト**
- **DHCPv6 IAID を記録した**（IA_PD オプション、同じ Solicit 内）
- IA_PD の T1/T2/preferred/valid の各ライフタイムを控えた — valid ライフタイムが導火線の長さ
- `ls /etc/systemd/network/*.link` — WAN インターフェースをリネームする野良ファイルがない
- pcap をルーターの外へコピーした
- `ss -tlnp` で `0.0.0.0` / `::` にバインドされているものを監査した
- クロージャをワークステーションで事前ビルドした
- スマホをテザリングした

アクティベーション後の検証順序。

```sh
ip link show enp4s0 | grep ether     # クローンした MAC
networkctl status enp4s0             # /56 が届いた
ip -6 addr show enp4s0 | grep 1111   # ceAddr がある
ip -d link show sbtun                # encaplimit none、local/remote が正しい
ip -s link show sbtun                # TX だけでなく RX が増えている
ip route get 8.8.8.8                 # dev sbtun
ping -c3 8.8.8.8
```

診断のショートカット。

- **`/56` が来ない** → DUID か MAC の不一致。`journalctl -u systemd-networkd -b | grep -i dhcp` を確認。
- **`/56` は来るが IPv4 がない** → `EncapsulationLimit` が抜けているか、`Local=` が `ceAddr` と厳密に一致していない。
- **TX は増えるが RX が横ばい** → ファイアウォールがプロトコル 4 を飲み込んでいるか、BR が送信元アドレスを拒否している。
- **`ping google.com` ではなく `ping 8.8.8.8` でテストしてください。** 両方のインターフェースで `UseDNS = false` にしていると、トンネルが動くかどうかとは無関係に名前解決が失敗します。

## 第 8 部: 最初のバグ — IPv4 のみのサイトへの初回リクエストが毎回ハングした

症状: AAAA レコードのないサイトを開くと約 40 秒ハングします。どのマシンでも、再試行は一瞬でした。DNS の問題に見えますが、違いました。

> このブログはちょうど AAAA レコードを持っていないので、このブログで試せます。

手がかりは、「そのサイトが IPv6 を持っているか」と完全に連動していたことです。デュアルスタックのサイトは問題ありませんでした。その経路はカプセル化のない MTU 1500 のネイティブな `enp4s0` だからです。IPv4 だけが 1460 の `sbtun` を通ります。

ハング中の `tcpdump -nn -i sbtun`:

```
SYN         options [mss 1460, ...]        ← バグはまさにここ
SYN-ACK     options [mss 1460, ...]
ACK
PSH 1:1561  (ClientHello)                  ← 送信は正常
ACK 1409 / ACK 1561                        ← サーバーは全て受信した

PSH seq 5793:5830, length 37               ← 37 バイト、正常に到着
ACK 1, sack {5793:5830}                    ← 「バイト 1 と 5793-5830 を受信済み」
```

バイト 1〜5792 — ServerHello と証明書チェーン — は決して届きませんでした。届いたのはわずか 37 バイトの末尾だけです。**小さいパケットは通り、フルサイズのものは消え、ICMP もない。** その後、約 63 秒のウィンドウの停止、FIN、そして RST が続きます。

`sack` ブロックが決め手です。ACK がまだ 1 で止まっているのに高い範囲の SACK が見えたら、それは遅いサーバーではなく PMTU ブラックホールを見ているのです。

仕組み: クライアントは自分の 1500 バイトの MTU から MSS 1460 を広告します。サーバーは 1460 バイトのペイロードを送ります。BR はそれを 40 バイトの IPv6 ヘッダーで包まなければならず — 1540 バイト — これが経路 MTU を超えるので破棄します。ICMP Fragmentation Needed が生成されるかどうかは問題ではありません。インターネットの十分に多くの部分が ICMP を破棄するので、送信者には届かないのです。

再試行が成功するのは、Linux が最初の停止のあとに PMTU エントリをキャッシュするからです。つまり宛先ごと・マシンごと・キャッシュのライフタイムごとに 40 秒のタイムアウトを 1 回だけ支払うことになり、これがまさに「初回はハング、2 回目は一瞬」というパターンです。

修正。

```nix
networking.nftables.tables.clamp = {
  family = "inet";
  content = ''
    chain forward {
      type filter hook forward priority mangle; policy accept;
      tcp flags syn / syn,rst oifname "sbtun" tcp option maxseg size set rt mtu
      tcp flags syn / syn,rst iifname "sbtun" tcp option maxseg size set rt mtu
    }
    chain output {
      type route hook output priority mangle; policy accept;
      tcp flags syn / syn,rst oifname "sbtun" tcp option maxseg size set rt mtu
    }
  '';
};

boot.kernel.sysctl."net.ipv4.tcp_mtu_probing" = 1;
```

意図的な選択が 3 つあります。

- `size set rt mtu` は 1420 をハードコードするのではなく経路 MTU から導出します。あとで `MTUBytes` を変えても自動的に追従します。
- forward に `oifname` と `iifname` の**両方**を指定します。前者は自分が広告する値を直し、後者は*サーバー*の広告が LAN のクライアントに届く前に書き換えるので、どちらの端も超過できなくなります。
- ルーター自身から発するトラフィックのために、`type route hook output` を持つ別の `output` チェインを用意します。

`tcpdump -i sbtun 'tcp[tcpflags] & tcp-syn != 0'` で検証します。SYN は今度は `mss 1420` を運んでいるはずです。先に `ip route flush cache` を実行してください。さもないと、修正ではなくキャッシュされた PMTU をテストすることになります。

## 第 9 部: 2 つ目のバグ — きっかり 4 時間ごとの全断

これは私をほとんど打ち負かしたもので、修正はたった一つ欠けていたフィールドでした。

### 何が起きたか

切り替えからおよそ 4 時間後、すべてのインターネットが止まりました。劣化ではなく、消失です。

```
9: sbtun@enp4s0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1460 ...
    RX:  bytes packets errors dropped  missed   mcast
             0       0      0       0       0       0
    TX:  bytes packets errors dropped carrier collsns
        269687    4060    164     164     164       0
```

RX はきっかり 0、TX には carrier エラー。ONU とルーターを再起動しても効果はありませんでした。

- BR への `ping6` が失敗した
- **Router Advertisement が一切届かない**（`tcpdump -nn -i enp4s0 'icmp6 && ip6[40] == 134'` → 沈黙）
- 動的な SLAAC アドレスがない
- networkd のジャーナルに DHCP・renew・rebind の言及が何もない
- フィルタなしの `tcpdump -i enp4s0` は大量のトラフィックを示した — しかし **100% 送信方向**

復旧方法: HGW を ONU に挿し直し、起動させ、ルーターをつなぎ直す。設定変更は一切なし。

### 突破口になった手がかり

再び起きました。しかも、ルーターが起動してから 4 時間後ではなく、**HGW が最後に接続してからきっかり 4 時間後**でした。

この基点がすべてです。4 時間 = 14400 秒 = キャプチャから得た IA_PD の **valid ライフタイム**。私が見ていたカウントダウンは、*HGW* が確立したリースのものだったのです。私のルーターはそれを一度も更新しませんでした。そもそも一度も持っていなかったからです。

つまり、私はずっと引き継いだバインディングに乗っていたのです。期限が切れると、加入者セッションが上流で切断されました。これが、単にトンネルではなく **RA が止まった**理由です。RA は要求なしのマルチキャストでクライアント側の状態を必要としないので、これは私の以前のどの仮説にも当てはまらなかった観察でした。HGW をつなぎ直すとバインディングが再確立され、同じ 4 時間の導火線が再び始まりました。

### なぜルーターは自分のリースを一度も得られなかったのか

キャプチャを走らせながら `networkctl reconfigure enp4s0`:

```
1   0.000000 fe80::9a2c:... → ff02::1:2  DHCPv6 Solicit XID: 0x6f1749 CID: 000300030001982cc6XXXXXX
2   1.070431 fe80::9a2c:... → ff02::1:2  DHCPv6 Solicit XID: 0x6f1749 CID: 000300030001982cc6XXXXXX
3   3.184200 ...
4   7.368770 ...
5  15.473139 ...
6  31.498594 ...
7  61.963827 ...
8 121.580047 ...
```

1/2/4/8/16/32/64/128 秒で再送バックオフし、**Reply はゼロ**。サーバーは認識できないクライアントを完全に無視していました。

欠陥は 2 つ、どちらもクライアントのアイデンティティにありました。

**`CID: 000300030001982cc6XXXXXX` — `0003` が 2 回ある 12 バイト。** 第 3 部の DUID バイト配置の罠です。完全な 10 バイトの DUID を `DUIDRawData` に入れてしまい、networkd が自分の type フィールドを前置したのです。

**IAID の不一致。** DUID を取り出したところで止めてしまい、同じ Solicit の数行下にある HGW の `IAID: 00000001` に気づいていませんでした。networkd は既定でインターフェース名をハッシュします。バインディングは DUID **と** IAID をキーにします。

### 修正

```nix
dhcpV6Config = {
  DUIDType = "link-layer";
  DUIDRawData = "00:01:98:2c:c6:XX:XX:XX";   # 10 バイトではなく 8 バイト
  IAID = 1;
  PrefixDelegationHint = "::/56";
  UseAddress = false;
  UseDNS = false;
  WithoutRA = "solicit";
};
```

4 時間待つ前に検証しましょう。

```sh
sudo tcpdump -nn -i enp4s0 -w /tmp/v2.pcap 'udp port 546 or udp port 547' &
networkctl reconfigure enp4s0
sleep 20 && sudo pkill tcpdump
tshark -r /tmp/v2.pcap
```

成功は `Solicit → Advertise → Request → Reply` で、Client Identifier が `Length: 10` になっていることです。応答のない Solicit が 8 回続くなら、まだ認識されていません。

その後、`journalctl -u systemd-networkd` がようやく DHCPv6 に言及するようになり、`ip -6 addr show enp4s0` が静的なエントリと並んで動的なエントリを表示します。ルーターは自分のリースを保持し、T1（7200 秒）で更新するようになったので、4 時間の導火線は消えました。

### 同じログが露わにした 3 つ目のバグ

```
enp4s0: Interface name change detected, renamed to eth0.
```

以前の試行で残った `.link` ファイルが WAN インターフェースをリネームしていました。まだ何も壊してはいませんでした — networkd は古い名前でマッチし続けていたので — が、いずれ `matchConfig.Name = "enp4s0"` は何にもマッチしなくなっていたはずです。野良ファイルを確認して削除しましょう。

```sh
ls -la /etc/systemd/network/*.link
```

MAC は第 4 部のとおり、`.network` の `[Link]` に置きます。

### 診断上の教訓

**静的な `address =` 行は、実際には設定されていないのにインターフェースが設定済みに見せかけます。**

私は `ip -6 addr show enp4s0` が CE アドレスを表示するのを見て何時間も安心し、それを DHCPv6 が動いた証拠だと思い込んでいました。違いました。そのアドレスは `.network` ファイルにハードコードされていて — networkd は DHCPv6 パケットが 1 つでも応答されたかどうかに関係なくそれを設定します。私は自分の設定を自分に読み返していただけだったのです。

本当のシグナルはこうです。有限のライフタイムを持つ*2 つ目の*動的アドレス、`networkctl status` に現れる委任されたプレフィックス、そしてそもそもジャーナルに DHCPv6 が現れること。ログからの完全な欠落こそ、最後ではなく最初に追うべきものでした。

肝に銘じる価値のある系。**別のデバイスの時計に基準を置いた全断は、あなたのバグの時計ではありません。** 最初の全断が 3 時間に見えたのは、自分の切り替えから測っていたからです。実際には HGW の最後のプロビジョニングから 4 時間でした。間違ったイベントから測ったせいで、決して適用されない T2 の rebind タイマーを追いかける羽目になりました。

## 第 10 部: もう一つの説明のつかない現象

最初のアクティベーション: 疎通なし。再起動: すべて動く。同じ generation、設定変更なし。

最も可能性の高い説明: `ip6tnl` の netdev が、`enp4s0` に `ceAddr` が設定される前に作成された、というものです。`ip6tnl` は作成時に `local` をバインドしますが、networkd は netdev の作成を下位リンクのアドレス設定より後に順序付けるとは限りません。そのためトンネルが誤ったアドレス（あるいは何もなし）を送信元として上がってしまい、BR はそれを無視しました。再起動はたまたまそれを正しい順序に直列化しただけです。

もしこれが原因なら、リンクのネゴシエーションが遅いコールドブートで再発します。`ip -d link show sbtun` を確認してください。`local` が CE アドレスと厳密に一致していなければ、運で動いているだけです。決定的な修正はおそらくこれです。

```nix
systemd.network.netdevs."20-sbtun".tunnelConfig.Independent = true;
```

これはトンネルの作成を下位リンクの状態から切り離します。**まだ未検証**です。再起動で抜け出せる `test` アクティベーションで試してください。

別の説明: HGW がまだ BBIX のセッションを保持していて、再起動が単に経過時間を稼いだだけ、というものです。既に抜いてあったことを考えると可能性は低いですが、これなら設定上の欠陥が一切ないまま同一の症状を生み出しえます。

## 注意点

- **HGW はレンタルし続け、しかも手の届く場所に置いておく必要があります。** IPv6高速ハイブリッド はレンタルに同梱されているので、解約するとサービスが死にます。しかも第 9 部が示すように、これは唯一の復旧手段でもあります。BR のバインディングをゼロから再確立できる唯一のデバイスなのです。不便な場所に置かないでください。
- **何日にもわたるデバッグの尾を見込んでおきましょう。** 私の場合、安定するまでに 3 つの別々のバグがあり、そのうち一つは 4 時間周期でしか姿を現しませんでした。接続が必要になる前日に切り替えるのはやめましょう。
- **ひかり電話 / ホワイト光電話 は使えなくなります。** 電話を残したままルーターをバイパスする方法はありません。同じクローン MAC を持つ 2 台のデバイスを ONU に同時につなぐことはできないからです。
- **これはほぼ確実に SoftBank の規約の範囲外です。** サポートは受けられません。
- **地域が重要です。** 公開されている報告はほとんどが XG-100NE の 東日本 のものです。私は EVO310G の 西日本 です。アーキテクチャは同じですが、アドレスは違います。自分でキャプチャしてください。
- **ファームウェアはこれらのどれでも変えうります。** パラメータが HGW によって動的に取得されるのには理由があるのです。

## 参考文献

- [`makeding/luci-app-fleth`](https://github.com/makeding/luci-app-fleth) — OpenWrt のヘルパー。その ISP 対応表は、SoftBank が MAP-E でも DS-Lite でもなく 固定IP/IPIP6 であるという、最も明確な公開情報です
- [Missing's Blog — SoftBank 光・10ギガ移除NTT路由器直接桥接ONU](https://blog.missing233.com/2023/08/13/softbank-hikari-research/) と [設定ガイドの続編](https://blog.missing233.com/2023/09/16/softbank-hikari-openwrt-configuration/) — 元祖のリバースエンジニアリング。RADIUS VSA 204/207 のデコードと 8 時間の DHCPv6 問題を含みます
- [zenn.dev/zyun — ソフトバンク光の10Gプラン](https://zenn.dev/zyun/scraps/d6d3781094804a) — HGW の「MAP-E」というラベルにもかかわらず素朴な IPIP を示すパケットキャプチャ
- [塩の惑星 — 大容量回線ソフトバンク光10GでIPv6,IPv4の自宅サーバ運用をしてみた](https://corkborg.github.io/home-server-with-softbank-hikari-10g/) — 専有 IPv4 という発見。XG-100NE 側からのもの
- [RFC 2473](https://datatracker.ietf.org/doc/html/rfc2473) — Generic Packet Tunneling in IPv6
