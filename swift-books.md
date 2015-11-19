# Object Storageの概要
## Object Storageとは何か
オブジェクトストレージとは、データをオブジェクトとして管理するストレージシステムのことである。これは、データをファイル階層として扱うファイルシステムとも、データをセクターとトラックによるブロック構造として扱うブロックストレージとも異なるストレージ形態である。基本的にオブジェクトは、データそのものと、加えて様々なメタデータとユニークなID情報を持っている。通常、NASやCIFSストレージはディレクトリによる階層構造を持っているのに対して、オブジェクトストレージはフラットなオブジェクト構造を持っている。このオブジェクト構造により、オブジェクトの移動が容易となること、オブジェクト毎にメタデータを付与できること、ACL設定が可能であること、オブジェクト数を原則として無制限に拡張可能(ディレクトリのinodeサイズの制限を受けない)であること、ユーザ間での共有が容易であること、などの特徴をもつことができる。

## オブジェクトストレージが適しているデータとその用途
オブジェクトストレージは、分散ストレージであるがゆえに一貫性の担保が非常に難しい。したがって、全ての用途に適用できるわけではない。適用できるデータの例としては、非構造データ、写真、動画、個人データ、学術データなどである。また、そのデータの用途としては、SNSを始めとしたソーシャルコンテンツ、静的なWebコンテンツ、クラウドストレージ、バックアップなどが挙げられる。

# Swiftの概要
## Swiftの特徴
Swiftは、OpenStackのコアコンポーネントの中でオブジェクトストレージを担う部分のプロジェクトである。Keystoneによるマルチテナント化されたアカウント構造と、S3のバケットに相当するコンテナという、いわゆるNASのボリュームにと同等の構造を持っている。オブジェクトの最大サイズは5GBであり、データのストアにはREST APIを利用することができる。また、デフォルトで3冗長にデータが複製され保存される。

## 基本構成
Swiftは、他のOpenStackプロジェクト同様Pythonで開発されている。しかし、バージョン管理は他のコンポーネントとは異なり、独自のバージョン番号により管理されている。そのため、半年ごとのOpenStackのリリース時には、そのタイミングでの最新のSwiftが採用されることになる。

## ファイルシステム
ファイルシステムには、デフォルトでXFSが推奨されている。これは、object-serverにxattrというファイルシステムの拡張属性が利用されているからである。よって、ext4を用いてSwiftを構築することも可能である。

## 基本構造
基本構造として、ProxyノードとStorageノードに分けることができる。

### Proxyノード
Proxyノードは、フロントエンドのサーバとしてREST APIのリクエストを受け付ける。Storageノードは、Proxyノード上でコンシステントハッシングにより、どのノードに配置するかが決定されたうえで、Storageノード上に実データが配置される。

### Storageノード
Storageノードは、内部ではaccount-server、container-server、object-serverの3つが実際には稼働している。account-serverとcontainer-serverはSQLiteが利用され、object-serverはXFSのファイルシステムが利用されている。

#### account-server
account-serverは、Keystoneで管理されているアカウントごとのSQLiteが作成され、その中で各アカウントごとのコンテナ情報が格納されている。

#### container-server
container-serverは、各コンテナごとの実オブジェクトの各種情報が格納されている。各コンテナごとに1つのSQLiteファイルを持つ構造となっている。

#### object-server
object-serverは、実オブジェクトが計算されたハッシュ値に基づき、ノード配置とディレクトリ構造が決定されたうえで、実際にXFSファイルシステム上にストアされる。

### Zone
Swiftクラスターは、ゾーンという論理的なグルーピングがなされており、データのレプリケーションは基本的にはこのゾーンをまたぐ形で行われる。

### Ringファイル
Storageノードの分散配置の構造自体はRingファイルという静的なファイルを全てのノードに配布することで決定される。このファイルの情報に基づき、ノードの拡張、縮退が実施される。データ格納用ディスクは、RAIDを組まず(あるいはSingle Disk RAID0を組む)、1本ずつRingファイルに登録する。つまり1つのディスクが1つのノードのように振る舞うことになる。



# 小規模構成で作るSwiftクラスター(Kilo Release)

## はじめに
OpenStack Swiftを構築しようと思い、[公式ドキュメント](http://docs.openstack.org/kilo/install-guide/install/apt/content/ch_swift.html)を見ながら構築したら出来なかった。構築出来なかった原因は、`container-reconciler`が`memcached`を必要とすることが公式ドキュメントのどこにも記載されていなかったためと、rootデバイスしかない仮想VM上で構築しようとしたためである。

memcachedに関しては、以下のとおりissueが存在する。
[container-reconciler need memcache in storage node](https://bugs.launchpad.net/openstack-manuals/+bug/1464939)

この点を踏まえて、改めて構築の手順を以下に記すことにする。公式ドキュメントそのままの手順で問題ない箇所については手順を省略する。

## 構成
OSはUbuntu 14.04.3を利用し、かつ、SwiftはKiloリリース時点のものを用いた。が、なぜかUbuntuのkiloリリース時点のSwiftのバージョンが`2.3`ではなく`2.2.2`になっていた。これはCanonicalのパッケージメンテナがサボってるからなのだろうか??? 加えて言うと、認証には同じくKiloリリースのKeystoneを使った。申し訳ないが、Keystoneの構築手順まで記すと非常に長くなるため割愛する。

account-server/container-serverとobject-serverを別のVM上に構築しているため以下の様な構成となっている。また、Regionは1つ、Zoneは5つで構築した。各Zoneに付き、account-server/container-serverを1VMとobject-serverを1VMの計2VM使っている。

| Component                         | Num of VMs| IP address       |
|:----------------------------------|:---------:|:-----------------|
| proxy-server                      | 1         | 192.168.0.2      |
| account-server & container-server | 5         | 192.168.0.[3-7]  |
| object-server                     | 5         | 192.168.0.[8-12] |
| keystone                          | 1         | 192.168.0.13     |

## KeystoneにおけるSwiftユーザーの作成Endpointの登録
Kiloリリースを用いるため基本的には`v3 API`と`python-openstackclient`を利用する。

Swiftユーザの作成、Swiftユーザーのadminロールへの紐付け、Swiftサービスの作成は、公式ドキュメントの通りである。

[To configure prerequisites](http://docs.openstack.org/kilo/install-guide/install/apt/content/swift-install-controller-node.html)

API Endpointの登録

```shell-session
$ openstack endpoint create \
  --publicurl 'http://192.168.0.2:8080/v1/AUTH_%(tenant_id)s' \
  --internalurl 'http://192.168.0.2:8080/v1/AUTH_%(tenant_id)s' \
  --adminurl http://192.168.0.2:8080 \
  --region RegionOne \
  object-store
+--------------+----------------------------------------------+
| Field        | Value                                        |
+--------------+----------------------------------------------+
| adminurl     | http://192.168.0.2:8080/                      |
| id           | af534fb8b7ff40a6acf725437c586ebe             |
| internalurl  | http://192.168.0.2:8080/v1/AUTH_%(tenant_id)s |
| publicurl    | http://192.168.0.2:8080/v1/AUTH_%(tenant_id)s |
| region       | RegionOne                                    |
| service_id   | 75ef509da2c340499d454ae96a2c5c34             |
| service_name | swift                                        |
| service_type | object-store                                 |
+--------------+----------------------------------------------+
```

ここで注意しなければならない点は、Endpointのアドレスである。公式ドキュメントでは`controller`と記載がある。これはSwiftのproxy-serverがcontrollerで稼働していることが前提となっている。proxy-serverを別サーバとして構築、あるいは、前段にロードバランサを用いる場合、このEndpointに記載するアドレスはproxy-server、もしくは、ロードバランサのIPアドレスである。

上記では、proxy-serverのIP addressを用いて設定している。

## proxy-serverの設定
パッケージのインストール、GitHubからのサンプルconfの取得は公式ドキュメントの通りである。

[To install and configure the controller node components](http://docs.openstack.org/kilo/install-guide/install/apt/content/swift-install-controller-node.html)

取得したサンプルを元に`/etc/swift/proxy-server.conf`を編集する。
`[DEFAULT]`、`[pipeline:main]`、`[app:proxy-server]`、`[filter:keystoneauth]`は公式ドキュメントの通りで問題ない。認証項目に関してのみ注意が必要である。

```
[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
...
auth_uri = http://192.168.0.13:5000
auth_url = http://192.168.0.13:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = swift
password = SWIFT_PASS
delay_auth_decision = true
```

公式ドキュメントでは、auth_uriとauth_urlに`controller`と記載があるが、この項目は認証に関する項目であるため、keystoneが稼働しているサーバのアドレスに変更する必要がある。

## account-server、container-server、object-serverの下準備
Swiftを構築するにあたって、account-server、container-server、object-serverではroot領域とは別デバイスのパーティションが必要となる。例えば、object-serverではobject-replicatorというプロセスがデータ領域としての追加デバイスがマウントされているかどうかのチェックをしている。root領域しかない場合、プロセスの起動に失敗するため注意が必要である。

sdb以降のデバイスを確保できる場合はそれを使ってXFSでフォーマットする。無い場合は、ループバックデバイスを用いる方法がある。

## ループバックデバイスを使って普通のファイルをXFSファイルシステムとしてマウントする
ちなみに、以下の方法はdevstackのソースコードをベースにしている。

## XFSパッケージのインストール

`xfsprogs`というパッケージが必要になる。

```swift

$ sudo apt-get install -y xfsprogs
```

## ループバックデバイスとなるファイルの作成
ファイルを配置するディレクトリの作成と、`touch`によるファイル作成を行う。

```shell-session
$ sudo mkdir -p /srv/node/images
$ sudo touch /srv/node/images/file.img
```

`truncate`コマンドによりSparseファイルを作成する。

```shell-session
$ sudo truncate -s 50G /srv/node/images/file.img
```

## ファイルシステムのフォーマット

XFSでフォーマットする

```shell-session
$ sudo /sbin/mkfs.xfs -f -i size=1024 /srv/node/images/file.img
meta-data=/srv/node/images/file.img isize=1024   agcount=4, agsize=3276800 blks
         =                       sectsz=512   attr=2, projid32bit=0
data     =                       bsize=4096   blocks=13107200, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0
log      =internal log           bsize=4096   blocks=6400, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

fileコマンドで確認すると以下のとおりになる。

```shell-session
$ file /srv/node/images/file.img
/srv/node/images/file.img: SGI XFS filesystem data (blksz 4096, inosz 1024, v2 dirs)
```

## マウント

マウント先のディレクトリを作成する。

```shell-session
$ sudo mkdir -p /srv/node/disk1
```

マウントする。

```shell-session
$ sudo mount -t xfs -o loop,noatime,nodiratime,nobarrier,logbufs=8 /srv/node/images/file.img /srv/node/disk1
```

`df`コマンドで確認する。

```shell-session
$ df -h | grep loop0
/dev/loop0       50G   33M   50G   1% /srv/node/disk1
```

これで、ループバックデバイスのマウントが完了する。

rsyncの設定まではaccount-server & container-serverのホストとobject-serverのホストで共通である。

[To configure prerequisites](http://docs.openstack.org/kilo/install-guide/install/apt/content/swift-install-storage-node.html)

## account-server、container-serverの設定
account-server、container-serverの稼働するホストで各々必要なパッケージをインストールする。注意する点は、`memcached`をインストールして起動させておく必要がある点である。

```shell-session
# apt-get install swift swift-account swift-container memcached
```

`/etc/swift`に移動し、`account-server.conf-sample`、`container-server.conf-sample`、`container-reconciler.conf-sample`をGitHubから取得してくる。object-server関連のconfを取得する必要はない。むしろ、object-server関連のconfが存在すると、最後にプロセス起動を`swift-init`を用いて行う際に、object-serverのプロセスも稼働してしまう。

```shell-session
# curl -o /etc/swift/account-server.conf \
  https://git.openstack.org/cgit/openstack/swift/plain/etc/account-server.conf-sample?h=stable/kilo
# curl -o /etc/swift/container-server.conf \
  https://git.openstack.org/cgit/openstack/swift/plain/etc/container-server.conf-sample?h=stable/kilo
# curl -o /etc/swift/container-reconciler.conf \
  https://git.openstack.org/cgit/openstack/swift/plain/etc/container-reconciler.conf-sample?h=stable/kilo
```

その後の手順は、公式ドキュメントの通りである。

最後に、`memcached`は再起動しておこう。

```shell-session
# service memcached restart
```

## object-serverの設定
object-serverの稼働するホストで各々必要なパッケージをインストールする。

```shell-session
# apt-get install swift swift-account swift-object
```

`/etc/swift`に移動し、`object-server.conf-sample`、`object-expirer.conf-sample`をGitHubから取得してくる。こちらでは、逆に、account-server、container-server関連のconfを取得する必要はない。

```shell-session
# curl -o /etc/swift/object-server.conf \
  https://git.openstack.org/cgit/openstack/swift/plain/etc/object-server.conf-sample?h=stable/kilo
# curl -o /etc/swift/object-expirer.conf \
  https://git.openstack.org/cgit/openstack/swift/plain/etc/object-expirer.conf-sample?h=stable/kilo
```

こちらも、その後の手順は、公式ドキュメントの通りである。

## ringの設定
proxy-serverのホスト上でringファイルを生成する。このタイミングで、クラスターのパーティション数や冗長度が決定される。クラスターの規模によりパーティション数や冗長度は慎重に決める必要がある。パーティション数に関する知見はGREEさんのこの記事がとても役に立つ。

[OpenStack Swiftによる画像ストレージの運用](http://labs.gree.jp/blog/2014/12/11746/)

今回は、冗長度3、パーティション数は2^17とする。proxy-serverで以下を実行する。コマンドオプションについては、公式ドキュメントを参照する。

[Create initial rings](http://docs.openstack.org/kilo/install-guide/install/apt/content/swift-initial-rings.html)

```shell-session
# cd /etc/swift
# rm -f *.builder *.ring.gz backups/*.builder backups/*.ring.gz
# swift-ring-builder account.builder create 17 3 1
# swift-ring-builder account.builder add r1z1-192.168.0.3:6002/disk1 100
# swift-ring-builder account.builder add r1z2-192.168.0.4:6002/disk1 100
# swift-ring-builder account.builder add r1z3-192.168.0.5:6002/disk1 100
# swift-ring-builder account.builder add r1z4-192.168.0.6:6002/disk1 100
# swift-ring-builder account.builder add r1z5-192.168.0.7:6002/disk1 100
# swift-ring-builder account.builder
# swift-ring-builder account.builder rebalance

# swift-ring-builder container.builder create 17 3 1
# swift-ring-builder container.builder add r1z1-192.168.0.3:6001/disk1 100
# swift-ring-builder container.builder add r1z2-192.168.0.4:6001/disk1 100
# swift-ring-builder container.builder add r1z3-192.168.0.5:6001/disk1 100
# swift-ring-builder container.builder add r1z4-192.168.0.6:6001/disk1 100
# swift-ring-builder container.builder add r1z5-192.168.0.7:6001/disk1 100
# swift-ring-builder container.builder
# swift-ring-builder container.builder rebalance

# swift-ring-builder object.builder create 17 3 1
# swift-ring-builder object.builder add r1z1-192.168.0.8:6000/disk1 100
# swift-ring-builder object.builder add r1z2-192.168.0.9:6000/disk1 100
# swift-ring-builder object.builder add r1z3-192.168.0.10:6000/disk1 100
# swift-ring-builder object.builder add r1z4-192.168.0.11:6000/disk1 100
# swift-ring-builder object.builder add r1z5-192.168.0.12:6000/disk1 100
# swift-ring-builder object.builder
# swift-ring-builder object.builder rebalance
```

完了したら、`account.ring.gz`、`container.ring.gz`、`object.ring.gz`を各ノードに配置する。

## プロセスの起動
ここは公式ドキュメントの通りで問題ない。

[Configure hashes and default storage policy](http://docs.openstack.org/kilo/install-guide/install/apt/content/swift-finalize-installation.html)

proxy-serverのホストでは、`proxy-server`と`memcached`が起動していれば良い。それ以外のホストでは、`swift-init`がよしなにやってくれるはずである。

## 構築できているかどうかの確認
KiloリリースであるためKeystoneのAPIはv3を利用している。そのため、swiftコマンドを実行する際には、`-V 3`というオプションが必要になる。

```shell-session
$ source demo-openrc.sh
$ swift -V 3 stat
                        Account: AUTH_25e9c03ea9824a6e8d24a60ac5e72c98
                     Containers: 0
                        Objects: 0
                          Bytes: 0
Containers in policy "policy-0": 0
   Objects in policy "policy-0": 0
     Bytes in policy "policy-0": 0
    X-Account-Project-Domain-Id: default
                     Connection: keep-alive
                    X-Timestamp: 1441783575.55310
                     X-Trans-Id: tx2260ad3b3ed840f99d075-0056091154
                   Content-Type: text/plain; charset=utf-8
                  Accept-Ranges: bytes
```

このようにプロンプトが返ってくれば問題ない。プロンプトが少し時間がたっても返ってこない場合、どこかで設定が間違っているはずである。その際には、まず`--debug`オプションを付けて再度実行してみてほしい。

```shell-session
$ swift --debug -V 3 stat
```

内部で実行されているREST APIの詳細が分かる。あとは、各ホストのログを確認していけば良い。

以上で、構築は完了である。各々の構成によって注意すべき点は変わってくると思うが、今回は、自分がハマった点を踏まえて手順を作成してみた。





# CLIによるクラスターの操作


# 運用と管理
