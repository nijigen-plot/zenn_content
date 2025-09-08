---
title: "TP-Link Wi-Fiルーターを使ってスマホから家のサーバーにSSH接続する"
emoji: "📱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["VPN", "SSH", "tplink"]
published: true
---

## 目的

スマートフォン(Android)端末から、家のLinuxサーバーにSSH接続し、作業を行うことです。

![](/images/tplink_vpn_server/202509086.png)


セキュリティの観点からVPN経由でのSSH接続を行います。

TP-Link製のWi-Fiルーターでは、VPNサーバーを簡単に立てられるのでその機能を使いました。

https://www.tp-link.com/jp/support/faq/3220/


## TP-Link設定

TP-Linkルーターの管理画面から設定を行います。

https://www.tp-link.com/jp/support/faq/87/

![](/images/tplink_vpn_server/20250908.png)
*管理画面*

### 動的DNSサービス

お家のインターネット回線のグローバルIPが動的な場合（個人契約はほとんど動的だと思います）、定期的にグローバルIPが変わってしまうので、[TP-Linkの動的DNSサービス](https://www.tp-link.com/jp/support/faq/4509/)を使ってドメインネームとグローバルIPを紐づけます。

https://www.tp-link.com/jp/support/faq/4509/

![](/images/tplink_vpn_server/2025-09-08_154434.png)
*詳細設定→動的 DNS*

上記画像の文章の通りTP-Link IDでログインを行ったあと動的DNSの画面に戻ると、設定ができます。

![](/images/tplink_vpn_server/202509081.png)

ステータスが**occupied**になっていればサブドメインの競合なく動的DNSが完了しています。
気になる場合はpingコマンド等で設定した動的DNSが家のグローバルIPに紐づいているか確認するとよいです。

```bash
$ ping xxxxxxxxxx.tplinkdns.com
PING xxxxxxxxxx.tplinkdns.com (xxx.xxx.xxx.xxx) 56(84) bytes of data.
64 bytes from xxx.com (xxx.xxx.xxx.xxx): icmp_seq=1 ttl=64 time=1.62 ms
64 bytes from xxx.com (xxx.xxx.xxx.xxx): icmp_seq=2 ttl=64 time=0.997 ms
64 bytes from xxx.com (xxx.xxx.xxx.xxx): icmp_seq=3 ttl=64 time=0.720 ms
64 bytes from xxx.com (xxx.xxx.xxx.xxx): icmp_seq=4 ttl=64 time=0.902 ms
```

### VPNサーバー

OpenVPNによるVPNサーバーを構築します。

![](/images/tplink_vpn_server/202509082.png)
*VPNサーバー→OpenVPN*

https://www.tp-link.com/jp/support/faq/1544/


1. 証明書を生成
2. OpenVPN 有効にチェックを入れて項目の設定を行う
3. 設定ファイルを出力

の順番に行っていきます。

**証明書を生成**は「生成する」のボタンを押して待てばよいだけです。

**OpenVPN 有効にチェックを入れて項目の設定を行う**も、チェック後、特にこだわりが無ければそのままで問題ありません。

![](/images/tplink_vpn_server/202509083.png)


**設定ファイルを出力**すると、ダウンロードが始まります。

![](/images/tplink_vpn_server/202509084.png)

#### OpenVPN-Config.ovpn設定

OpenVPN-Config.ovpnを開くと以下のようになっています。

```:OpenVPN-Config.ovpn
client
dev tun
proto udp
float
nobind
cipher AES-128-CBC
comp-lzo adaptive
resolv-retry infinite
remote-cert-tls server
persist-key
remote xxx.xxx.xxx.xxx 1194
<ca>
以下証明書･鍵情報
```

以下のように変更して保存します。([こちらの記事](https://qiita.com/yuya_hatano/items/fa613eff68576f9db35c)を参考にしました)

```diff:OpenVPN-Config.ovpn
client
dev tun
proto udp
float
nobind
- cipher AES-128-CBC
+ data-ciphers-fallback AES-128-CBC
comp-lzo adaptive
resolv-retry infinite
remote-cert-tls server
persist-key
- remote xxx.xxx.xxx.xxx 1194
+ remote xxxxxxxxxx.tplinkdns.com 1194
+ tls-cert-profile insecure
+ tls-version-min 1.0
<ca>
以下証明書･鍵情報
```

#### VPNサーバーを経由しない場合どうなるか

単純にSSHの為のポートをインターネットに解放した場合、世界中から攻撃が来るようです。（特に22番そのままだと沢山来る）

https://www.reddit.com/r/linuxquestions/comments/1gaufdm/whats_the_point_of_ssh_over_vpn_connection/?tl=ja

### 接続先サーバーローカルIPアドレス予約

接続先サーバーのローカルIPアドレスを予約し、固定します。

![](/images/tplink_vpn_server/202509085.png)
*詳細設定→ネットワーク→DHCPサーバーからMACアドレスを指定してIPアドレスを予約*


## スマートフォン設定

### VPN

OpenVPNによるVPNサーバーに接続するためのアプリをインストールします。

私はOpenVPN Connectを使いました。

https://play.google.com/store/apps/details?id=net.openvpn.openvpn&hl=ja&pli=1


[VPNサーバー設定](#openvpn-configovpn設定)で出力した`OpenVPN-Config.ovpn`をスマートフォンに転送し、**Upload File**で選択します。

転送方法についてはできるだけファイルを外に出さない方法が好ましいと思います。私は家のNAS経由で行いました。

![](/images/tplink_vpn_server/Screenshot_20250904-205917.png)
*BROWSEからファイルを選択*

設定が適切であればそのまま繋がります。

![](/images/tplink_vpn_server/Screenshot_20250904-212418~2.png)

この時、Wi-FiルーターのWi-Fiを使っていると失敗するので、キャリア回線で試すのが良いと思います。

![](/images/tplink_vpn_server/Screenshot_20250908-162110.png)

### SSH

ターミナルエミュレーターのTermuxアプリをインストールします。

https://play.google.com/store/apps/details?id=com.termux&hl=ja

![](/images/tplink_vpn_server/Screenshot_20250908-165109.png)
*開くと最初こんなかんじ*

SSH鍵を発行します。

```bash:Termux Terminal
$ pkg update
$ pkg install openssh
$ ssh -V
$ ssh-keygen -t ed25519
$ cat ~/.ssh/id_ed25519.pub
```

SSH Configファイルを作ります。その後は左へスワイプし新しいセッションを開始します。

```bash:~/.ssh/config
Host xxxxxxxxxx
    HostName 接続先サーバーのローカルIP
    User yyyy
    IdentityFile ~/.ssh/id_ed25519
```


接続先サーバーの`~/.ssh/authorized_keys`にTermuxで発行した`id_ed25519.pub`の情報を入れます

```bash:接続先 Terminal
$ vi ~/.ssh/authorized_keys
ssh-ed25519 AAAA~~~~~~~~~~~ xxxxxxx@localhost
```


Termuxから`ssh`コマンドでサーバーへ接続します。

![](/images/tplink_vpn_server/Screenshot_20250908-171056.png)

接続できました！

これで外から家のサーバーにSSHすることができますね。
