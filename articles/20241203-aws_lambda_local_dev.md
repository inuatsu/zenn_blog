---
title: AWS Lambda のローカル開発環境の整備
emoji: "\U0001F4BB"
type: tech
topics:
  - Lambda
  - Docker
published: true
publication_name: simpleform_blog
---

本記事は [SimpleForm Advent Calendar 2024](https://qiita.com/advent-calendar/2024/simpleform) の 3 日目の記事です。

シンプルフォームで開発しているプロダクト SimpleCheck では、[AWS Lambda](https://aws.amazon.com/jp/lambda/) を多数利用しています。
Lambda はサーバの管理不要で処理を実行でき運用は便利ですが、ローカル環境での開発環境を整備するのがやや大変です。
今回は、シンプルフォームで Lambda の開発環境をどのように整備しているかを共有します。

## この記事で言及すること、言及しないこと

### 言及すること

- [Docker Compose](https://docs.docker.com/compose/) を使ってローカル環境で Lambda を起動、実行できる環境を整備する方法
- 複数の Lambda を扱えるようにするため、ローカルドメイン (今回は `.localhost` で終わるドメインを想定) でコンテナにアクセスできるようにする方法

### 言及しないこと

- Lambda 関数を実装する方法
- Lambda 関数を AWS 環境にデプロイする方法

## ローカル環境整備方針

シンプルフォームではコンテナイメージを使って Lambda 関数を作成しています。
コンテナイメージを使う方式の場合、ローカル環境でも Docker で起動する方針が相性が良いと考えられるので、今回は Docker で Lambda を起動する方向で考えます。

Lambda の処理では、データベースなど他のリソースとのアクセスも発生すると想定されます。
Docker Compose で必要なリソースをまとめて起動できるようにし、同一ネットワーク内で起動することでコンテナ間の通信もできるようにする方針で進めることを前提とします。

また、Lambda を複数起動できるようにすることを考えると、ホスト側のポート番号の重複を避けるため、コンテナ側とは別のポート番号にマッピングする必要があります ([AWS ECR Public Gallery に公開されている Lambda 用のイメージ](https://gallery.ecr.aws/lambda/)は 8080 番ポートでエンドポイントを起動しますが、複数の Lambda のコンテナでコンテナ側のポートと同じホスト側のポートにマッピングすることは番号が重複するためできません)。
適当なポート番号を使う場合、ポート番号を覚えるという認知負荷があるため、`.localhost` で終わるドメインでコンテナにアクセスできるようにすることで、認知負荷を下げることを検討します。

## 下準備

Linux だと `.localhost` で終わるドメインは `localhost` と同様 `127.0.0.1` に解決されるものの、macOS では Unknown host となり解決されません。
したがって、macOS の場合はまず `.localhost` で終わるドメインが `127.0.0.1` に解決されるようにする必要があります。
今回は、[Dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html) を使ってこれを実現する方法を紹介します。

1. Dnsmasq を brew でインストールし、dnsmasq を起動

```sh
brew install dnsmasq
sudo brew services start dnsmasq
```

1. `.localhost` へのリクエストを `127.0.0.1` に解決するように Dnsmasq を設定します

```sh
echo "\naddress=/.localhost/127.0.0.1\naddress=/.localhost/::1\n" >> /opt/homebrew/etc/dnsmasq.conf
```

1. macOS が `.localhost` へのリクエストを Dnsmasq に転送するように設定します

```sh
sudo mkdir -p /etc/resolver
sudo sh -c 'echo "nameserver 127.0.0.1\n" > /etc/resolver/localhost'
```

ここまで設定したら、`ping mysql.localhost` など `.localhost` で終わる適当なドメインに ping を実行してみてください。`64 bytes from 127.0.0.1` のような結果が表示されたら、`.localhost` で終わるドメインが `127.0.0.1` に解決されています。

## compose.yml の作成

今回は仮に以下のようなディレクトリ構成になっていると想定します。

```txt
`sample
    |-- common
    |    `-- compose.yml (nginx-proxy と mysql のサービスが定義されている)
    `-- functions
         |-- function1 (Lambda 関数の実装が格納されているディレクトリ)
         |    |-- ...
         |-- function2 (Lambda 関数の実装が格納されているディレクトリ)
         |    |-- ...
         |-- Dockerfile
         `-- compose.yml (function1, 2 のサービスが定義されている)
```

モノレポ的な構成になっていると、nginx-proxy や mysql は色々なところで使われるので、共通的に使うものを格納するディレクトリに格納する選択も採り得るかと思います。
数が少ないと不自然に見えるディレクトリ構成ですが、そのような前提だと考えていただければと思います。

### nginx-proxy と mysql のコンテナ

`.localhost` で終わるドメインが `127.0.0.1` に解決されるようにしたところで、compose.yml を作っていきます。
ドメインでコンテナにアクセスできるようにするには、リバースプロキシを使います。
Docker Hub に [nginx-proxy](https://hub.docker.com/r/nginxproxy/nginx-proxy) というコンテナイメージが公開されており、このイメージを使うと nginx で手軽にリバースプロキシを構築することができます。
まずは、nginx-proxy と mysql のコンテナを起動する compose.yml を作ります。

```yaml:sample/common/compose.yml
services:
  mysql:
    image: mysql:8.0
    command: mysqld --log-error
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "mysql -u $${MYSQL_USER} -p$${MYSQL_PASSWORD} -e 'SELECT version()'",
        ]
      interval: 5s
      timeout: 5s
      retries: 5
      start_interval: 1s
    volumes:
      - mysql:/var/lib/mysql
    ports:
      - "3306:3306"
    environment:
      - MYSQL_DATABASE=test
      - MYSQL_USER=test
      - MYSQL_PASSWORD=test
      - MYSQL_ROOT_PASSWORD=test
      - VIRTUAL_HOST=mysql.localhost
      - VIRTUAL_PORT=3306
    networks:
      default:
        aliases:
          - mysql.localhost
    depends_on:
      - nginx-proxy

  nginx-proxy:
    image: nginxproxy/nginx-proxy:latest
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro

volumes:
  mysql:

networks:
  default:
    name: local_default
```

nginx-proxy サービスはコンテナ側の 80 番ポートをホスト側の 80 番ポートにマッピングします。

mysql サービスの環境変数として `VIRTUAL_HOST` が設定されていますが、ここで設定したドメインでホスト側からアクセスすることができ、`VIRTUAL_PORT` に設定したポートが nginx によりプロキシされます。
また、コンテナ間の通信でも同じドメインでアクセスできるようにする方が都合が良いと考えられます。
そこで、`networks > default > aliases` にも `VIRTUAL_HOST` に設定したものと同じドメインを設定します。
この設定をすることで、mysql サービスはホストからも同一ネットワークに起動している他のサービスからも `http://mysql.localhost:3306` でアクセスできるようになります。

他に以下の設定をしています。

- mysql サービスの [`depends_on`](https://docs.docker.com/reference/compose-file/services/#depends_on) に nginx-proxy を設定 : これにより、mysql サービスを起動すると nginx-proxy サービスも起動するようになります。
- mysql サービスに [`healthcheck`](https://docs.docker.com/reference/compose-file/services/#healthcheck) を設定 : MySQL の起動が完了すると成功する healthcheck を設定しています。これは後ほど Lambda コンテナの起動設定時に使います

### Lambda 関数のコンテナ

次に Lambda 関数を定義する compose.yml を考えます。
function1 と function2 という名前の Lambda 関数が実装されており、function2 を実行する前に function1 を実行しておく必要があるという前提があると仮定します。

```yaml:sample/functions/compose.yml
include:
  - ../common/compose.yml

services:
  function1:
    ports:
      - "11111:8080"
    environment:
      - VIRTUAL_HOST=function1.localhost
      - VIRTUAL_PORT=8080
    networks:
      default:
        aliases:
          - function1.localhost
    depends_on:
      mysql:
        condition: service_healthy

  function2:
    ports:
      - "11112:8080"
    environment:
      - VIRTUAL_HOST=function2.localhost
      - VIRTUAL_PORT=8080
    networks:
      default:
        aliases:
          - function2.localhost
    depends_on:
      - function1
```

[`include`](https://docs.docker.com/compose/how-tos/multiple-compose-files/include/) で定義された compose.yml ファイルに記述されているサービスやネットワークは、この compose.yml の中で利用することができます。
この compose.yml の中で、上の compose.yml で定義した mysql / nginx-proxy のサービスやネットワークを利用できるようにするために記述しておきます。

`VIRTUAL_HOST`、`VIRTUAL_PORT` の環境変数設定や `networks > default > aliases` の値は先ほどの compose.yml と同じ要領で設定します。
コンテナ側はどちらも同じ 8080 番ポートを使うので、被らない適当な番号をそれぞれのホスト側にマッピングします。

function1 は `depends_on` に mysql を設定しており、`service_healthy` という condition を設定しています。
この設定をすると、function1 を起動する際に mysql サービスが起動し (さらに、mysql サービスの依存で nginx-proxy サービスも起動します)、mysql サービスの healthcheck が成功したあとで function1 サービスが起動することになります。
なお、[`depends_on` には `service_started`, `service_healthy`, `service_completed_successfully` のいずれかを設定できます](https://docs.docker.com/compose/how-tos/startup-order/#control-startup) (condition を設定しない場合はデフォルトは `service_started` です)

function2 は function1 のエンドポイントが実行された後に行う処理を実装しているという前提があるので、function2 の `depends_on` には function1 を設定します。

## Docker で起動した Lambda の実行方法

これで compose.yml ファイルの作成は完了です。
実際に Docker で起動した Lambda を実行するには、まずサービスを起動する必要があります。

```sh
cd sample/functions
docker compose up function2 --build
```

上記コマンドを実行すると、function2 の `depends_on` に function1 が、function1 の `depends_on` に mysql が、mysql の `depends_on` に nginx-proxy が設定されているので、nginx-proxy, mysql, function1, function2 が起動します。
function2 の起動が完了したら、以下のコマンドを実行することで function1, function2 の Lambda をローカルで実行できます (`--data` の引数には Lambda に渡したい event ペイロードを適宜セットします)。

```sh
# function1 の実行
curl --location 'http://function1.localhost/2015-03-31/functions/function/invocations' \
--header 'Content-Type: application/json' \
--data '{}
'
# function2 の実行
curl --location 'http://function2.localhost/2015-03-31/functions/function/invocations' \
--header 'Content-Type: application/json' \
--data '{}
'
```

## おわりに

今回紹介した手法を採ることで、コンテナイメージで作成された Lambda 関数をローカル環境でエミュレートすることができるようになり、ローカル環境での Lambda 関数の開発がしやすくなると期待されます。

なお、今回は MySQL にしか言及しませんでしたが、実際には Lambda から RDS のみならず様々なリソースを使うかと思います。
どのようなリソースが必要かによりますが、例えば [MinIO](https://min.io/) (S3 互換のオブジェクトストレージ)、[DynamoDB Local](https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/DynamoDBLocal.html) (ローカル開発用に提供されている DynamoDB)、[LocalStack](https://www.localstack.cloud/) (様々な AWS サービスをエミュレートできるサービス) などを活用することで、今回言及していないリソースもローカル環境で AWS 環境に近い構成を再現できます。

Lambda のローカル開発がしづらくて悩んでいる...という方はぜひ試していただければ幸いです。
