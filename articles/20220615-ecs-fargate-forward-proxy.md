---
title: ECS Fargate 上でフォワードプロキシサーバを構築する (前編)
emoji: "\U0001F310"
type: tech
topics:
  - aws
  - ecs
published: true
published_at: "2022-06-15 07:00"
publication_name: simpleform_blog
---

今回は、ECS Fargate 上でフォワードプロキシサーバを構築する方法を紹介します。
構成としては、NLB を置き、ECS (Fargate) 上で動いている Squid コンテナを NLB のターゲットとする形です。
NLB のドメインでフォワードプロキシサーバを指定して使うことを想定しています。

## 想定しているユースケース

プライベートサブネット内の Lambda や ECS で任意の Web サイトのクローラを作るケースを想定します。

プライベートサブネットからインターネットへアクセスするので、目的のサイトにアクセスする際のソース IP アドレスは NAT Gateway に割り当てた Elastic IP で固定されます。
可能な限りソース IP アドレスを分散させたいと考えましたが、無料のプロキシサーバは不安定かつセキュリティ面で不安が残り、一方で有料のプロキシサーバはトラフィックが増えると比較的良いお値段がします。

それゆえ、AWS 上でフォワードプロキシサーバを構築できないかという発想に至りました。
AWS 上で構築するということは、AWS が保有している IP アドレスしか利用できないという制約はあるものの、その制約内である程度のソース IP アドレスの分散はできると考えられます。

想定している構成を図にまとめると、以下のようになります。

![フォワードプロキシサーバ構成図](/images/20220615-ecs-fargate-forward-proxy_structure.drawio.png)

## この構成を選定した理由

### ECS

Google で検索すると、EC2 でフォワードプロキシサーバを構築している記事はいくつか見かけました。

@[card](https://qiita.com/miyabiz/items/06a70f879c53fd07e635)
@[card](https://nopipi.hatenablog.com/entry/2022/02/13/100000)

ただ、EC2 で構築する場合サーバのメンテナンスが面倒だなと感じるのが正直なところです。
ECS Fargate で構築すれば、同じ構成のコンテナを何台立てるのも楽ですし、定期的にコンテナを落として IP ローテーションをするといったことも難なくできそうです。
というわけで、ECS Fargate を前提に検討しました。

なお、コンテナで Squid を動かす、ECS で Squid を動かすといったテーマを取り上げた記事はいくつか見かけました。
しかし、今回のように `NLB -> ECS Fargate (Squid)` という構成に言及した記事は私が調べた限りでは見つけられませんでした。

### NLB

ECS で IP ローテーションする場合、パブリックサブネットに配置して IP 自動割り当てを有効化するのが手っ取り早いと思われます。
ただし、IP が自動割り当てされるということは、接続するフォワードプロキシサーバの IP アドレスが毎回変わることになります。

それだとクローラを開発する際にプロキシの設定が難しいので、何かしらのドメインでアクセスができるようになるのが望ましいです。
したがって、ECS の前にロードバランサを挟もうという考えになるのですが、ALB と NLB どちらを使うかが問題になります。

普段 ALB を使うのに慣れているので、今回も ALB でと思っていたのですが、実験してみるとうまくいきませんでした。
ALB 自体がリバースプロキシのしくみですので、フォワードプロキシサーバを作りたいという今回の要件に合わないようです。

そのため、NLB を利用することにしました。

### Squid

次に、フォワードプロキシサーバを立てるのに使うソフトウェアを選定する必要があります。

今回、候補としては nginx と Squid を検討していました。Squid は使ったことがなく、まず nginx を検討したのですが、調べてみると 2 点不便な点がありました。

1. nginx はデフォルトでは HTTPS 通信をフォワードプロキシできないため、ソースコードをダウンロードしたうえで別途パッチを適用する必要がある[^1]
1. nginx はデフォルトでは Digest 認証に対応していないため、ソースコードをダウンロードしたうえで別途 Digest 認証モジュールを追加してコンパイルする必要がある[^2]

2 点目に関しては、上に書いたように ECS の IP 自動割り当てを有効化するためパブリックサブネットに配置する場合、サーバが全世界に公開されてしまうため、何かしらの認証を設けたいと考えていました。
Basic 認証よりはまだ Digest 認証の方がマシかなということで Digest 認証を使おうと思っていたのですが、nginx はデフォルトだと対応していないと知って意外に思いました。

Squid は上記 2 点いずれもデフォルトで対応しています。

パッチ適用や外部モジュール追加をしても良いのですが、管理がやや煩雑になりそうな気がしたため、Squid を利用することにしました。

## 実装

### Squid の設定ファイル

まずは Squid の設定ファイルから見ていきます。

:::details 長いので、全体をご覧になりたい方はアコーディオンを開いてください。

```conf:squid.conf
auth_param digest program /usr/lib/squid/digest_file_auth -c /etc/squid/password
auth_param digest children 20 startup=0 idle=1
auth_param digest realm MyRealm
auth_param digest nonce_garbage_interval 5 minutes
auth_param digest nonce_max_duration 30 minutes
auth_param digest nonce_max_count 50

acl digest_user proxy_auth REQUIRED

http_access allow digest_user

# Squid normally listens to port 3128
http_port 35201 require-proxy-header

# Add NLB IP addresses to ACL
acl nlb_ip src xxx.xxx.xxx.xxx/xx

# Add client IP addresses to ACL
acl client_ip src xxx.xxx.xxx.xxx/32

# Permit using proxy protocol
proxy_protocol_access allow nlb_ip
proxy_protocol_access allow client_ip

# Permit http access
http_access allow nlb_ip
http_access allow client_ip

acl SSL_ports port 443
acl Safe_ports port 80  # http
acl Safe_ports port 21  # ftp
acl Safe_ports port 443  # https
acl Safe_ports port 70  # gopher
acl Safe_ports port 210  # wais
acl Safe_ports port 1025-65535 # unregistered ports
acl Safe_ports port 280  # http-mgmt
acl Safe_ports port 488  # gss-http
acl Safe_ports port 591  # filemaker
acl Safe_ports port 777  # multiling http
acl CONNECT method CONNECT

#
# Recommended minimum Access Permission configuration:
#
# Deny requests to certain unsafe ports
http_access deny !Safe_ports

# Deny CONNECT to other than secure SSL ports
http_access deny CONNECT !SSL_ports

# Only allow cachemgr access from localhost
http_access allow localhost manager
http_access deny manager

# We strongly recommend the following be uncommented to protect innocent
# web applications running on the proxy server who think the only
# one who can access services on "localhost" is a local user
http_access deny to_localhost

# And finally deny all other access to this proxy
http_access deny all

# Set PID file to a place the default squid user can write to
pid_filename /var/run/squid/${service_name}.pid

# Change log format
logformat timefm %{%Y/%m/%d %H:%M:%S}tl %ts.%03tu %6tr %>a %Ss/%03>Hs %<st %rm %ru %[un %Sh/%<a %mt

# Disable cache
cache deny all

# Hide hostname
visible_hostname unknown

# Hide source IP address
forwarded_for off

# Prevent the fact that accessing via proxy from being known to target
request_header_access X-Forwarded-For deny all
request_header_access Via deny all
request_header_access Cache-Control deny all

# Don't display the version on the error page.
httpd_suppress_version_string on

# Use stdio and redirect access log to stdout and daemon log to stderr
access_log stdio:/proc/self/fd/1 timefm

# Leave coredumps in the first cache dir
coredump_dir /var/cache/squid
```

:::

デフォルトの設定と異なる箇所のみ解説します。

#### Digest 認証の設定

```
auth_param digest program /usr/lib/squid/digest_file_auth -c /etc/squid/password
auth_param digest children 20 startup=0 idle=1
auth_param digest realm MyRealm
auth_param digest nonce_garbage_interval 5 minutes
auth_param digest nonce_max_duration 30 minutes
auth_param digest nonce_max_count 50

acl digest_user proxy_auth REQUIRED

http_access allow digest_user
```

- `/usr/lib/squid/digest_file_auth` は認証用のプログラムで、OS により場所が異なる。のちほど触れるが、今回は `debian:bullseye-slim` を使っており、その場合はこの場所に配置される
- `/etc/squid/password` はパスワードファイルの保管場所である。保管場所を変更する場合は適宜変更すること
- `MyRealm` は認証領域の指定で、任意の文字列を指定可能
- `digest_user` は ACL の名前で、任意の文字列を指定可能
- そのほか、各種パラメータの意味は公式ドキュメントを参照

@[card](http://www.squid-cache.org/Doc/config/auth_param/)

#### Listen するポートの指定

```conf
# Squid normally listens to port 3128
http_port 35201 require-proxy-header
```

- Squid はデフォルトでは 3128 番ポートで Listen するが、パブリックサブネットに配置する都合上、デフォルトとは異なるポートで Listen するのが望ましいと考える
- NLB を通った通信のアクセス元は、通常 ECS は知ることができない[^3]。ECS がアクセス元を知ることができるようにするためには NLB において Proxy Protocol を有効化する必要がある。Squid 側でも Proxy Protocol を必須にするため `require-proxy-header` という指定を付ける必要がある[^4]

#### 許可するアクセス元 IP アドレスの設定

```conf
# Add NLB IP addresses to ACL
acl nlb_ip src xxx.xxx.xxx.xxx/xx

# Add client IP addresses to ACL
acl client_ip src xxx.xxx.xxx.xxx/xx

# Permit using proxy protocol
proxy_protocol_access allow nlb_ip
proxy_protocol_access allow client_ip

# Permit http access
http_access allow nlb_ip
http_access allow client_ip
```

- ECS をパブリックサブネットに配置するため、Squid 側でアクセス元 IP アドレスの制限をかけておくのが非常に重要。NLB が配置されている VPC の CIDR (`nlb_ip`) と、NLB のアクセス元の IP アドレス (`client_ip`) を設定し、いずれも Proxy Protocol を利用許可する。`xxx.xxx.xxx.xxx/xx` としている箇所は適切な IP アドレスを記述すること[^5]
- 複数のアクセス元 IP アドレスを設定したい場合、`acl client_ip src xxx.xxx.xxx.xxx/xx` を必要な行数だけ増やす

:::message
Squid の設定ファイルは上から読まれるため、順番が大事です。
`http_access deny all` が登場するまでにアクセス許可される条件に当てはまらないと、アクセス拒否されます。
そのため、Digest 認証の設定や許可するアクセス元 IP アドレスの設定は `http_access deny all` の行より上に書く必要があります。

:::

#### そのほか細かな設定

```conf
# Change log format
logformat timefm %{%Y/%m/%d %H:%M:%S}tl %ts.%03tu %6tr %>a %Ss/%03>Hs %<st %rm %ru %[un %Sh/%<a %mt

# Use stdio and redirect access log to stdout and daemon log to stderr
access_log stdio:/proc/self/fd/1 timefm

# Disable cache
cache deny all

# Hide hostname
visible_hostname unknown

# Hide source IP address
forwarded_for off

# Prevent the fact that accessing via proxy from being known to target
request_header_access X-Forwarded-For deny all
request_header_access Via deny all
request_header_access Cache-Control deny all

# Don't display the version on the error page.
httpd_suppress_version_string on
```

- デフォルトのログのフォーマットには時刻やユーザーエージェントなどがないため、少し設定を変更する
- ECS では標準出力に出力された内容が CloudWatch Logs にログとして残るため、ログの出力先を標準出力に変更する[^6]
- 今回はキャッシュしたいという要件はないので、キャッシュは無効にする
- アクセス元を隠しておきたいため、ホスト名やソース IP アドレスを隠匿する設定を入れる
- デフォルトではエラーページに Squid のバージョンが表示される。悪意のある攻撃者に利用される恐れがあるため、これも表示されないようにする[^7]

### Squid の起動スクリプト

```sh:start_squid.sh
#!/bin/sh

set -e

SQUID=$(/usr/bin/which squid)

# Prepare the cache using Squid.
echo "Initializing cache..."
"$SQUID" -z

# Give the Squid cache some time to rebuild.
sleep 5

# Launch squid
echo "Starting Squid..."
exec "$SQUID" -NYCd 1
```

Squid の起動スクリプトは上記のようにしました。使っているオプションは以下の通りです[^8]。

- `-z` : キャッシュディレクトリを作成するオプションで、最初の起動時には実行する
- `-N` : デーモンモードで起動しないオプション。Docker でデーモンモードで起動すると即終了してしまうため、このオプションを付ける
- `-C` : fatal signal をキャッチしない設定をするオプション
- `-Y` : ファストリロード時に `UDP_HIT` か `UDP_MISS_NOFETCH` のみ返すようにする設定をするオプション
- `-d` : デバッグも標準エラー出力へ流すようにするオプション

### Dockerfile

```dockerfile:Dockerfile
FROM debian:bullseye-slim

ARG PROXY_USERNAME
ARG PROXY_PASSWORD

ENV TZ=Asia/Tokyo

RUN apt-get update && apt-get install -y --no-install-recommends squid tzdata
RUN cp /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

RUN groupadd squid
RUN useradd -g squid -d /home/squid squid
RUN mkdir /var/cache/squid
RUN mkdir /var/run/squid

RUN echo | awk -v username=${PROXY_USERNAME} -v realm=MyRealm -v hash="$( echo -n "${PROXY_USERNAME}:MyRealm:${PROXY_PASSWORD}" | md5sum | cut -d ' ' -f 1 )" '{print ""username":"realm":"hash""}' > /etc/squid/password

RUN chown -R squid:squid /var/log/squid
RUN chown -R squid:squid /var/cache/squid

COPY start-squid.sh /usr/local/bin/
COPY squid.conf /etc/squid/

RUN chmod 755 /usr/local/bin/start-squid.sh
RUN chmod 755 /etc/squid/squid.conf
RUN chown -R squid:squid /etc/squid/password
RUN chown -R squid:squid /var/run/squid

USER squid

CMD ["/usr/local/bin/start-squid.sh"]
```

Dockerfile は上記のようにしました。ポイントは以下のとおりです。

- セキュリティの観点から、Squid の Digest 認証のユーザー名とパスワードは build-arg としてビルド時のみ保持するように渡す。`build-arg` として渡された値はビルドされたコンテナ内には残らないため、環境変数などで渡すより安全性が高いため
- セキュリティの観点から、`squid` ユーザーを作成して Squid 起動に必要なファイルやディレクトリのみにアクセス権を渡し、コンテナ起動時は `root` ではなく `squid` ユーザーとして起動する
- 日本時間でログが出るようにするため、タイムゾーンを日本標準時に設定する
- MD5 暗号化されたパスワードファイルは awk, md5sum, cut コマンドを使って生成。Dockerfile では対話型プロンプトでパスワードを与えるのが面倒であるため、htdigest は使わない

---

このあたりまで準備できれば、あとは ECS Fargate にデプロイするだけというところですが、長くなってきたので続きは次回の記事に書きます。

次回は、ECS Fargate へのデプロイおよび関連するリソースの設定周りを解説し、実際に起動したフォワードプロキシサーバにアクセスしてフォワードプロキシとして機能していることの確認まで行います。

[^1]: 詳細は <https://fujiu.hatenablog.com/entry/2020/03/15/010353> をご参照ください

[^2]: 詳細は <https://qiita.com/heiwa_pinf/items/72bd8569320f9362a5b3> をご参照ください

[^3]: Proxy Protocol に関しては <https://dev.classmethod.jp/articles/nlb-meets-proxy-protocol-v2/> をご参照ください

[^4]: 公式ドキュメントでは <http://www.squid-cache.org/Doc/config/http_port/> に `require-proxy-header` のパラメータの説明があります

[^5]: NLB 経由で Squid サーバにアクセスする場合の設定のアクセス制限の方法については <https://blog.serverworks.co.jp/tech/2018/04/13/clb-proxyprotocol/> を参考にしました

[^6]: ログのフォーマット、出力先については <https://blog.mmmcorp.co.jp/blog/2018/02/17/squid_ecs/#outline__4> を参考にしました

[^7]: アクセス元情報の隠匿、エラーページでの Squid バージョンの非表示などは <https://dev.classmethod.jp/articles/redundant-proxy-servers-using-squid-and-nlb-and-efs/#toc-9> を参考にしました

[^8]: <https://blog.mmmcorp.co.jp/blog/2018/02/17/squid_ecs/#outline__7> の内容を拝借しました。オプションの説明は <https://linux.die.net/man/8/squid> を参考にしました
