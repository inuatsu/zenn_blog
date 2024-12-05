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
Lambda はサーバの管理不要で処理を実行でき運用は便利ですが、ローカル環境での開発環境を整備するのがややたいへんです。
今回は、シンプルフォームで Lambda の開発環境をどのように整備しているかを共有します。

## この記事で言及すること、言及しないこと

### 言及すること

- [Docker Compose](https://docs.docker.com/compose/) を使ってローカル環境で Lambda を起動、実行できる環境を整備する方法
- 複数の Lambda を扱えるようにするため、ローカルドメイン (今回はトップレベルが `.localhost` のドメインを想定) でコンテナへアクセスできるようにする方法

### 言及しないこと

- Lambda 関数を実装する方法
- Lambda 関数を AWS 環境にデプロイする方法

## ローカル環境整備方針

シンプルフォームではコンテナイメージを使って Lambda 関数を作成しています。
コンテナイメージを使う方式の場合、ローカル環境でも Docker で起動する方針と相性が良いと考えられるので、今回は Docker で Lambda を起動する方向で考えます。

Lambda の処理では、データベースなどほかのリソースとのアクセスも発生すると想定されます。
Docker Compose を使って必要なリソースをまとめて起動し、同一ネットワーク内で起動することによりコンテナ間も通信可能にする方針で進めることを前提とします。

ところで、[AWS ECR Public Gallery に公開されている Lambda 用のイメージ](https://gallery.ecr.aws/lambda/)は 8080 番ポートでエンドポイントを起動します。
Lambda を複数起動できるようにすることを考えると、ホスト側のポート番号の重複を避けるため、コンテナ側とは別のポート番号にマッピングする必要があります。
ランダムなポート番号を使う場合、ポート番号を覚えるという認知負荷があります。
トップレベルが `.localhost` のドメインでコンテナへアクセス可能にすることで、認知負荷を下げることを検討します。

## 下準備

Linux だとトップレベルが `.localhost` のドメインは `localhost` と同様 `127.0.0.1` に解決されますが、macOS では Unknown host となり解決されません。
したがって、macOS の場合はまず トップレベルが `.localhost` のドメインが `127.0.0.1` に解決されるよう設定する必要があります。
今回は、[Dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html) を使ってこれを実現する方法を紹介します。

1. Dnsmasq を brew でインストールし、dnsmasq を起動

```sh
brew install dnsmasq
sudo brew services start dnsmasq
```

1. `.localhost` へのリクエストを `127.0.0.1` に解決するように Dnsmasq を設定

```sh
echo "\naddress=/.localhost/127.0.0.1\naddress=/.localhost/::1\n" >> /opt/homebrew/etc/dnsmasq.conf
```

1. macOS が `.localhost` へのリクエストを Dnsmasq に転送するように設定

```sh
sudo mkdir -p /etc/resolver
sudo sh -c 'echo "nameserver 127.0.0.1\n" > /etc/resolver/localhost'
```

ここまで設定したら、`ping mysql.localhost` などトップレベルが `.localhost` である適当なドメインに ping を実行してみてください。
`64 bytes from 127.0.0.1` のような結果が表示されたら、トップレベルが `.localhost` のドメインが `127.0.0.1` に解決されています。

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

モノレポ的な構成になっていると、`nginx-proxy` や `mysql` はいろいろなところで使われるので、共通的に使うものを格納するディレクトリに格納する選択も採り得ると考えます。
数が少ないと不自然に見えるディレクトリ構成ですが、そのような前提だと考えていただければ幸いです。

### `nginx-proxy` と `mysql` のコンテナ

トップレベルが `.localhost` のドメインが `127.0.0.1` に解決されるよう設定したところで、compose.yml を書いていきます。
ドメインでコンテナへアクセスできるようにするには、リバースプロキシを使います。
Docker Hub に [nginx-proxy](https://hub.docker.com/r/nginxproxy/nginx-proxy) というコンテナイメージが公開されており、このイメージを使うと nginx で手軽にリバースプロキシを構築できます。
まずは、`nginx-proxy` と `mysql` のコンテナを起動する compose.yml を書きます。

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

`nginx-proxy` サービスはコンテナ側の 80 番ポートをホスト側の 80 番ポートにマッピングします。

`mysql` サービスの環境変数として `VIRTUAL_HOST` が設定されています。
ここに設定したドメインでホスト側からアクセスでき、`VIRTUAL_PORT` に設定したポートが nginx によりプロキシされます。
また、コンテナ間の通信でも同じドメインでアクセスできるようにする方が便利だと考えられます。
そこで、`networks > default > aliases` にも `VIRTUAL_HOST` に設定したものと同じドメインを設定します。
この設定をすることで、`mysql` サービスはホストからも同一ネットワークに起動しているほかのサービスからも `http://mysql.localhost:3306` でアクセスできます。

他に以下の設定をしています。

- `mysql` サービスの [`depends_on`](https://docs.docker.com/reference/compose-file/services/#depends_on) に `nginx-proxy` を設定 : これにより、`mysql` サービスを起動すると `nginx-proxy` サービスも起動する
- `mysql` サービスに [`healthcheck`](https://docs.docker.com/reference/compose-file/services/#healthcheck) を設定 : MySQL の起動が完了すると成功する healthcheck を設定。のちほど Lambda コンテナの起動設定時に利用する

### Lambda 関数のコンテナ

次に Lambda 関数を定義する compose.yml を考えます。
function1 と function2 という名前の Lambda 関数があり、function2 の実行前に function1 を実行しておく必要があると仮定します。

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

[`include`](https://docs.docker.com/compose/how-tos/multiple-compose-files/include/) で定義された compose.yml ファイルに記述されているサービスやネットワークは、この compose.yml の中で利用できます。
この compose.yml の中で、上の compose.yml で定義した `mysql` / `nginx-proxy` のサービスやネットワークを利用できるようにするため記述しておきます。

`VIRTUAL_HOST`、`VIRTUAL_PORT` の環境変数設定や `networks > default > aliases` の値は先ほどの compose.yml と同じ要領で設定します。
コンテナ側はどちらも同じ 8080 番ポートを使うので、被らない適当な番号をそれぞれのホスト側にマッピングします。

function1 は `depends_on` に `mysql` を設定しており、`service_healthy` という condition を設定しています。
この設定をすると、function1 起動の際に `mysql` サービスが起動し、`mysql` サービスの healthcheck が成功したあとで function1 サービスが起動することになります。
また、`mysql` サービスの依存で `nginx-proxy` サービスも起動します。
なお、[`depends_on` には `service_started`, `service_healthy`, `service_completed_successfully` のいずれかを設定できます](https://docs.docker.com/compose/how-tos/startup-order/#control-startup)。
condition を設定しない場合のデフォルトは `service_started` です。

function2 は function1 のエンドポイントが実行された後に行う処理を実装している前提があるので、function2 の `depends_on` には function1 を設定します。

## Docker で起動した Lambda の実行方法

これで compose.yml ファイルの作成は完了です。
実際に Docker で起動した Lambda を実行するには、まずサービスを起動する必要があります。

```sh
cd sample/functions
docker compose up function2 --build
```

上記コマンドを実行すると、以下の依存関係があるので `nginx-proxy`, `mysql`, `function1`, `function2` が起動します。

- `function2` -- (`depends_on`) --> `function1`
- `function1` -- (`depends_on`) --> `mysql`
- `mysql` -- (`depends_on`) --> `nginx-proxy`

function2 の起動が完了したら、以下のコマンドを実行することで function1, function2 の Lambda をローカルで実行できます。
なお、`--data` の引数には Lambda に渡したい event ペイロードを適宜セットします。

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

## 終わりに

今回は、コンテナイメージで作成された Lambda 関数をローカル環境でエミュレートする方法を紹介しました。
この手法を採ることで、ローカル環境での Lambda 関数の開発がしやすくなると期待されます。

なお、今回は MySQL にしか言及しませんでしたが、実際には Lambda から RDS のみならずさまざまなリソースを使うことが想定されます。
どのようなリソースが必要かによりますが、たとえば以下のようなソリューションを活用することで、今回言及していないリソースもローカル環境で AWS 環境に近い構成を実現できます。

- [MinIO](https://min.io/) (S3 互換のオブジェクトストレージ)
- [DynamoDB Local](https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/DynamoDBLocal.html) (ローカル開発用に提供されている DynamoDB)
- [LocalStack](https://www.localstack.cloud/) (さまざまな AWS サービスをエミュレートできるサービス)

Lambda のローカル開発がしづらくて悩んでいる...という方はぜひ試していただければ幸いです。
