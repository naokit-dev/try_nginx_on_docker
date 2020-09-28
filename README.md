# とりあえずの`docker-compose up`から入って、webアプリのデプロイに必要なネットワークの基礎設計を学びながら、Dockerを学ぶ

## はじめに

以下のような方とって有益な内容になればと思っています

1. これから初めてのwebアプリを作成する
2. webアプリを作成し、これからデプロイする
3. 初めてデプロイまで到達したが、Nginxが何をしているかとか、設定内容はコピペでよくわかっていない

私は独学プログラミング5ヶ月目で3の状態に近いかと思います
個人的なメッセージとしては是非1.の状態の方により多く、この記事の内容が届くといいなと思っています
**Dockerがなんとなくわかる、便利な気がする！**、**webアプリが動く仕組みに関心を広げる**、そんな気づきを共有できたらいいなと思っています

この記事と同じ内容は、AWS上のEC2を利用したり、契約したVPSを利用するよることで再現可能ですが、dockerを利用すれば**完全無料で挑戦可能**です！より手軽で、予定外の課金に怯える必要はありません



## この記事で知ることができる内容

**ネットワークに視野を広げる**

- webアプリが最低限動作するために必要な構成を知る
- Nginx（エンジンエックスと読みます）の3つの重要な役割, webサーバー, ロードバランサー, リバースプロキシの役割に触れる
- Nginxの基本的な設定を知る


**Dockerの基本を知る**

- 既存の開発環境を簡単に再現できることを知る
- Docker上で開発を行うために最低限必要なコマンドを試すことができる
- docker-compose.ymlに記述された内容や、volumeの仕組みを手を動かして知ることができる
- 複数のdocker-compose.ymlを用意して、異なる環境をシミュレートする (development -> production)

**webアプリの開発環境 - 本番環境での違いを知る**

- 本番環境でアセットコンパイルが必要な理由を知る
- 開発環境でアセットコンパイルが必要でない理由を知る

よって、この記事の最後ではDocker上で、仮想の本番環境で開発環境との違いに触れながら、アプリをデプロイしてみます

**appendix**は補足的内容となっています
その項で知ることのできる内容を初めに書いておきましたので、
改めて知る必要のない内容でしたら読み飛ばして頂いて結構です
もし知らない内容でしたら、実際に手を動かして頭の片隅に留めておくことで、後々役に立つ物があるかもしれません



## 必要なもの、スキル

- エディタ（VS Codeで検証しています）
  [Visual Studio Code - Code Editing. Redefined](https://code.visualstudio.com/)

- Linuxの基礎コマンドの知識（cd, ls, vi...くらい、なくてもコピペでなんとかなります）

- Docker desktop
  [Docker Desktop for Mac and Windows | Docker](https://www.docker.com/products/docker-desktop)

- Dockerって何？っていう方は以下がおすすめです

  [【連載】世界一わかりみが深いコンテナ & Docker入門 〜 その1:コンテナってなに？ 〜 | SIOS Tech. Lab](https://tech-lab.sios.jp/archives/18811)
  概念をさらっと理解していただき、ここでは手を動かしてみるというのがおすすめです

アプリの部分はFW（フレームワーク）にRailsを使用しておりますが、
Railsの知識はなくても大丈夫です

（私自身Rails以外の開発経験がないため、
他のFWにおいて不適切な内容があるかもしれません）


## 検証環境
macOS Catalina
docker desktop: 2.3.05
(docker engin: 19.03.12, docker-compose: 1.27.2)
Nginx: 1.18
Rails: 6.03
PostgreSQL: 11.0

## アーキテクチャ（設計）概要

これからDockerで構築する環境では
**Nginxはリバースプロキシとして機能していて、静的コンテンツを`app: Rails`に代わって代理（=プロキシ）配信しており、動的コンテンツへのリクエストのみ`app: Rails`に転送するようになっています。**

というのを少しずつ理解していきたいと思います

img

よく見る構成です（Databaseほか一部省略）
docker上でweb(Nginx), app(rails)というサービスがそれぞれ独立したコンテナで動いていて
docker-composeによってそれぞれの依存関係等が定義されているような理解です




## 目標5分、DockerでRailsの環境構築

Nginx - Railsの環境を構築します
以下の素晴らしい記事を参考にします（笑）
Nginx, Rails 6, PostgreSQL環境（おまけにBootstrapまで）がすぐに構築できます！
少しづつ改善していますので、改善コメントもお待ちしております。

[コマンドひとつ、5分でRails6の開発環境構築 on Docker - Rails6 + Nginx + PostgreSQL + Webpack (Bootstrap install済) - Qiita](https://qiita.com/naokit-dev/items/96cb41361ebc4b7716c0)

上記をベースに今回の記事のために用意したソースコード
https://github.com/naokit-dev/try_nginx_on_docker.git

### ソースコードをgit clone

```
#アプリを配置するディレクトリを作成（アプリケーションルート）
mkdir try_nginx_on_docker

#アプリケーションルートへ移動
cd $_

#ソースコード取得
git clone https://github.com/naokit-dev/try_nginx_on_docker.git

#アプリケーションルートにソースコードを移動
cp -a try_nginx_on_docker/. .                               
rm -rf try_nginx_on_docker 
```

以下のような構成になるかと思います

```
.(try_nginx_on_docker)
├── Dockerfile
├── Gemfile
├── Gemfile.lock
├── README.md
├── docker
│   └── nginx
│       ├── default.conf
│       ├── load_balancer.conf
│       └── static.conf
├── docker-compose.prod.yml
├── docker-compose.yml
├── entrypoint.sh
├── setup.sh
└── temp_files
    ├── copy_application.html.erb
    ├── copy_database.yml
    └── copy_environment.js
```

ソースコードの一部
`docker-compose.yml`
4つのコンテナが定義されています

```yml
version: "3.8"

services:
  web:
    image: nginx:1.18
    ports:
      - "80:80"
    volumes:
      - ./docker/nginx/static.conf:/etc/nginx/conf.d/default.conf
      - public:/myapp/public
      - log:/var/log/nginx
      - /var/www/html
    depends_on:
      - app

  db:
    image: postgres:11.0-alpine
    volumes:
      - postgres:/var/lib/postgresql/data:cached
    ports:
      - "5432:5432"
    environment:
      PGDATA: /var/lib/postgresql/data/pgdata
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-password}
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8 --locale=ja_JP.UTF-8"
      TZ: Asia/Tokyo

  app:
    build:
      context: .
    image: rails_app
    tty: true
    stdin_open: true
    command: bash -c "rm -f tmp/pids/server.pid && ./bin/rails s -p 3000 -b '0.0.0.0'"
    volumes:
      - .:/myapp:cached
      - rails_cache:/myapp/tmp/cache:cached
      - node_modules:/myapp/node_modules:cached
      - yarn_cache:/usr/local/share/.cache/yarn/v6:cached
      - bundle:/bundle:cached
      - public:/myapp/public
      - log:/myapp/log
      - /myapp/tmp/pids
    tmpfs:
      - /tmp
    ports:
      - "3000-3001:3000"
    environment:
      RAILS_ENV: ${RAILS_ENV:-development}
      NODE_ENV: ${NODE_ENV:-development}
      DATABASE_HOST: db
      DATABASE_PORT: 5432
      DATABASE_USER: ${POSTGRES_USER}
      DATABASE_PASSWORD: ${POSTGRES_PASSWORD}
      WEBPACKER_DEV_SERVER_HOST: webpacker
    depends_on:
      - db
      - webpacker

  webpacker:
    image: rails_app
    command: ./bin/webpack-dev-server
    volumes:
      - .:/myapp:cached
      - public:/myapp/public
      - node_modules:/myapp/node_modules:cached
    environment:
      RAILS_ENV: ${RAILS_ENV:-development}
      NODE_ENV: ${NODE_ENV:-development}
      WEBPACKER_DEV_SERVER_HOST: 0.0.0.0
    tty: false
    stdin_open: false
    ports:
      - "3035:3035"

volumes:
  rails_cache:
  node_modules:
  yarn_cache:
  postgres:
  bundle:
  public:
  log:
  html:
```



### 環境構築！

```
source setup.sh
```

正常にセットアップが終われば
アプリケーションルートディレクトリで以下のコマンドでコンテナ立ち上げた後

```
docker-compose up

# バックグラウンドで起動させる場合 -dオプション
docker-compose up　-d
```

ブラウザから`localhost`もしくは`localhost:80`へアクセスすると

` Yay! You’re on Rails!`が確認できるかと思います。
**誰でも簡単に開発環境を構築できます！Dockerのメリット1つ目**

ここで起動しているコンテナを確認してみます
(`-d`オプションを付けずに`docker-compose up`した場合には新しくターミナルを開きます。VS Codeなら`control`+`@`  \*mac環境）

```
docker ps
```

web (Nginx), app(Rails), webpacker(webpack-dev-server), db(PostgreSQL)の4つのコンテナが起動していることだけ確認してください

確認できたら一旦コンテナを終了させておきます

```
docker-compose down 
```





## Nginxで静的コンテンツを配信してみる

まだRailsアプリは使用しません
ここでは以下に挑戦します

- Nginxの最小の設定を確認する

- Docker-composeを利用しつつ、コンテナを単独 (nginxのみ) で起動してみる

- Nginx単独で単純な静的コンテンツ（HTML）を配信してみる

  

### 最もシンプルなNginxの設定

Nginxの設定を変更するため`docker-compose.yml`を編集します

```yml
services:
  web:
    image: nginx:1.18
    ports:
      - "80:80"
    volumes:
    #ここを書き換える./docker/nginx/default.conf... -> ./docker/nginx/static.conf...
      - ./docker/nginx/static.conf:/etc/nginx/conf.d/default.conf 
      - public:/myapp/public
      - log:/var/log/nginx
    depends_on:
      - app
...
```

**Dockerのvolumeについて少し**
`volumes:`の`./docker/nginx/static.conf:/etc/nginx/conf.d/default.conf`は`<host側path>:<container側path>`になっていて
これによってホスト（ローカル）側の`static.conf`をボリュームとしてマウントし、コンテナ内の`default.conf`として扱えるようにしています
ここでは、**ホスト側とコンテナ内では、ストレージが独立して存在するように振る舞われるため、このようなvolumeマウントが必要**ということだけ心に留めてください


`docker/nginx/static.conf`にはNginxの設定が記述されており、中身は以下のようになっています

```yml 
server { #ココから
      listen 80;　# must
      server_name _; # must
      root /var/www/html; # must
      index index.html;
      access_log /var/log/nginx/access.log;
      error_log  /var/log/nginx/error.log;
}　#ココまで、一つのserverブロック = Nginxが扱う一つの仮想サーバーの仕様
```

**server**: "{}"で囲われた内容(serverブロック)をもとに仮想サーバーを定義します

ここでは以下3項目が設定必須です

**listen**: 待ち受けるIP, portを指定（xxx.xxx.xxx.xxx:80, localhost:80, 80）
**server_name**: 仮想サーバに割り当てる名前。Nginxはリクエストに含まれるホスト名(example.com)やIP(xxx.xxx.xxx.xxx)に一致する仮想サーバを検索します。("_"はすべての条件で一致させるの意味です。その他、ワイルドカードや正規表現が利用可能)
**root**: ドキュメントルート、コンテンツが配置されたディレクトリを指定します

ちなみにlogについては、`etc/nginx/nginx.conf`というファイルで上記と同じパスが定義されているので、
ここで記述がなくても、error_logおよびaccess_logともに`/var/log/nginx/`以下に記録されるはずです
例えば`access_log /var/log/nginx/static.access.log;`とすることで、当該の仮想サーバ(serverブロック)固有のログを記録することもできるようです



### Nginxコンテナ単独で起動

先程の`docker-compose up`ではnginx, rails, webpack-dev-server, dbのすべてのコンテナが起動していますが、docker-composeのオプションを使用することで特定のコンテナだけを起動することも可能です

`--no-deps`: コンテナ間の依存関係を無視して起動 (ここではweb: nginxのみ)
`-d`: バックグラウンドでコンテナを起動、シェルは入力を継続できます
`-p`: ポートマッピング<host>:<container>
`web`: composeで定義されたnginxコンテナです

以下のコマンドでNginxコンテナを起動します

```
docker-compose run --no-deps -d -p 80:80 web
```

（ポートマッピングについてはcomposeでも指定しているのですが、改めて指定する必要があり、ホスト側のport 80をwebコンテナのport 80にマッピングしています）

docker-composeをオプション無しで実行したときとの違いを確認します

```
docker ps
```

先ほどと異なり、nginxのコンテナのみが起動していると思います



### HTMLコンテンツを作成

コンテナの中でシェルを呼び出します

```
docker-compose run --no-deps web bash
```

以下webコンテナ内での作業です

```
# index.htmlを作成
touch /var/www/html/index.html

# index.htmlの中身を追加
echo "<h1>I am Nginx</h1>" > /var/www/html/index.html

# index.htmlを確認
cat /var/www/html/index.html 
<h1>I am Nginx</h1>
```

これでコンテナ内のドキュメントルート直下に`index.html`が作成できたので`exit`でシェルを閉じましょう



### 動作確認

ブラウザからlocalhostにアクセスすると、
 以下のようにHTMLとして配信されているのが確認できていると思います

<img>

**ここでのNginxはリクエストに一致するコンテンツをドキュメントルートから探して、一致するものを返すというシンプルな挙動をしています**

確認できたら一旦コンテナを終了させておきます

```
docker-compose down 
```



### Appendix - リクエストに一致する仮想サーバがない場合のNginxの挙動

- Nginxのデフォルトサーバーの概念を知る

Nginxはクライアントからのリクエストに含まれるHostフィールドの情報をもとに、どの仮想サーバーにルーティングするかを定義しています

**では、いずれの仮想サーバもリクエストと一致しない場合はどのような挙動をするのでしょうか？**
設計を考える上で重要そうだったので、ここではそれを確認してみます。

先の設定ファイルのserverブロックで、いずれのリクエストに対しても該当するように`server name`を定義しましたが、これをリクエストと一致しないデタラメな名前に書き換えてみます

```
server_name undefined_server;
```

再びNginxコンテナを起動します

```
docker-compose run --no-deps -p 80:80 web
```

ブラウザからlocalhostにアクセスすると、リクエストに一致する仮想サーバが存在しないにもかかわらず
予想に反して先ほどと同じ"I am Nginx"が表示されると思います

**default server**

Nginxはリクエストがいずれの仮想サーバにも該当しなかった場合、default serverで処理する使用になっており、一番最初、一番上に記述された仮想サーバをdefault serverとして扱う仕様になっています

> In the configuration above, the default server is the first one — which is nginx’s standard default behaviour. It can also be set explicitly which server should be default, with the default_server parameter in the listen directive:
> [How nginx processes a request](https://nginx.org/en/docs/http/request_processing.html)

またはlistenディレクティブに明示的に`default_server`を指定することも可能です

```
listen      80 default_server;
```

今回の実験では"undefined_server"はリクエストに一致しないが、他に一致するものがないので
default serverとしてルーティングされたと考えられます

**いずれの仮想サーバもリクエストと一致しない場合 => default serverにルーティングされる**

うまくバックエンドのサーバーに接続されない場合など、エラーを切り分けるのに役立つ気がします

一旦コンテナも終了させておきましょう

```
docker-compose down 
```



### appendix - Dockerのvolumeを少し理解する

- コンテナの独立性について知る
- コンテナ - コンテナ間でストレージを共有する（永続化して共有する）仕組みとしてvolume、ここでは特にnamed volume, anonymous volumeの違いについて知る

**そもそもvolumeが必要(= 永続化が必要)な意義について**
Dockerではコンテナ内のデータを永続化するためにvolumeを作成し管理します

よくわからないので確認してみます

webコンテナの中でシェルを呼び出します

```
docker-compose run --no-deps web bash
```

以下webコンテナ内での作業です

```
# 検証用のディレクトリを作成　
mkdir /var/www/test

# 検証用のファイルを作成します
touch /var/www/test/index.html

# 存在確認
ls /var/www/test/
```

これで`/var/www/test`は`docker-compose.yml`の中でボリュームとして管理されていないパスであることがポイントです

一旦`exit`でシェルを閉じましょう（コンテナも終了します）

再度webコンテナを起動しシェルを呼び出します

```
docker-compose run --no-deps web bash
```

先程のファイルを探してみます

```
cat /var/www/test/index.html
```

```
ls /var/www
```

いかがでしょうか、
ディレクトリ`/var/www/test`、ファイル`/var/www/test/index.html`ともに見つからないと思います

**コンテナを終了すると、コンテナ内のデータは保持されない**これが原則であり
ボリュームはこの仕組を回避するために利用可能です

`exit`でターミナルを閉じます

すべてのコンテナを停止します

```
docker-compose down
```



**volumeの種類**

Dockerにおけるボリュームには以下のタイプがありますが、コンテナ内のデータを永続化するという点では同じです

1. host volume ?(ちょっと名前がわからないです)
2. anonymous volume (匿名ボリューム?anonymous volumeで通っている気がします)
3. named volume (名前付きボリューム)

docker-compose.ymlを見みてみます

```yml
version: "3.8"

services:
  web:
    image: nginx:1.18
    ports:
      - "80:80"
    volumes:
      - ./docker/nginx/static.conf:/etc/nginx/conf.d/default.conf #host volume
      - public:/myapp/public # named volume
      - log:/var/log/nginx # named volume
      - html:/var/www/html # named volume

...

volumes: # ここで異なるコンテナ間での共有を定義
  public:
  log:
  html:
```

**host volume**
nginxの設定のパートで触れました
`./docker/nginx/static.conf:/etc/nginx/conf.d/default.conf`の部分でホスト側のパスをボリュームとしてマウントします
**ホスト内のファイルをコンテナ側にコピーしているイメージ**です

**named volume**
順番が前後しますが、`html:/var/www/html`の部分
"html"という名前をつけてボリュームをマウントしています
さらに、"services"ブロックと同列の"volumes"ブロックでこの名前をもって定義することで
**複数のコンテナ間でボリュームをシェアすることを可能**にしています

そして、**このボリュームはホスト側からは独立して永続化されます**

最後に**anonymous volume**
公式docではnamed volumeとの違いは**名前があるかないか**のみとありますが
実際に名前がないというより、**named volumeの名前に相当する部分がコンテナごとにハッシュで与えられている**そうです
ちょっとわかりにくいですが、ホスト側をマウントする必要がないが、永続化の必要がある、**かつ複数のコンテナでの共有を想定しない場合**に利用するケースが考えられます
(まだイメージし難いですが、この後のコンテンツでanonymous volumeでないといけない場面に遭遇します）

ここでは少し理解を深めるために検証してみます
もともとnamed volumeとして定義している`/var/www/html`をanonymous volumeに変更して
本項で実施したHTMLファイル作成の手順を繰り返してみます

`docker-compose.yml`

```yml
version: "3.8"

services:
  web:
    image: nginx:1.18
    ports:
      - "80:80"
    volumes:
      - ./docker/nginx/static.conf:/etc/nginx/conf.d/default.conf
      - public:/myapp/public
      - log:/var/log/nginx
      - /var/www/html # コンテナ側のpathのみ指定しanonymous volumeに変更　

...

volumes:
  public:
  log:
  # html:　ここをコメントアウト
```

Nginxをweb serverとして起動

```
docker-compose run --no-deps -d -p 80:80 web
```

シェルを呼び出します

```
docker-compose run --no-deps web bash
```

ここが重要なのですが、別のターミナルでいま起動しているコンテナを確認すると

```
docker ps 
```

**２つのコンテナが起動しており、シェルが動いているコンテナは、ポートマッピングしているコンテナとは別であることがわかります**

このままコンテナ内でHTMLを作成

```
# index.htmlを作成
touch /var/www/html/index.html

# index.htmlの存在を確認
ls /var/www/html
```

さきほどと同様にブラウザから`localhost`にアクセスしてみましょう

するとブラウザは403エラーを示し
Nginxのエラーログを確認すると

```
tail -f 20 /var/log/nginx/error.log
```

```
...directory index of "/var/www/html/" is forbidden...
```

ディレクトリを見つけられないとエラーが記録されています

named volume -> anonymous volumeに変更したことで
**2つのコンテナ間で`/var/www/html/`以下の内容が共有されなくなり**
ローカルからport 80でリクエストを受けたコンテナからは`index.html`を参照することができなくなったことで
このようなエラーが生じていると考えられます

**永続化はするが、他のコンテナとボリュームを共有しない**、その特性にふれることができたかと思います

確認できたら`exit`でシェルを閉じ

毎度ですがコンテナを終了させておきましょう

```
docker-compose down 
```

（変更したdocker-compose.ymlの内容はこのままでも構いません）

...

---

appendixの内容に思ったよりも熱が入ってしまい長くなったので、（私のモチベーション維持のために）一旦ここで区切ります

②に続く





## Nginxでリクエストをバックエンドのサーバーへ転送する

Nginxの配下に複数のサーバーを配置し、受けたリクエストを効率的に分散処理する目的で用いられる設定を
ここではバックエンドサーバー(app: Rails)がひとつなのでロードバランサー的と曖昧な表現にとどめます
Railsは受けたリクエストに応じて動的にHTMLを生成します



### ロードバランサー的なNginxの設定

再び、Nginxの設定を変更するため`docker-compose.yml`を編集します
今度は`load_balancer.conf`を適応させます

```yml
services:
  web:
    image: nginx:1.18
    ports:
      - "80:80"
    volumes:
      - ./docker/nginx/load_balancer.conf:/etc/nginx/conf.d/default.conf #ここを書き換える
      - public:/myapp/public
      - log:/var/log/nginx
    depends_on:
      - app
...
```

docker/nginx/load_balancer.conf

```
upstream puma {
    server app:3000;
    server app:3001;
}

server {
    listen 80;
    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log;
    location / {
      proxy_pass http://puma;
    }
}
```

`static.conf`から３つ変化があります

まず、`upstream <name> {server <host or IP>:<port>}`で**転送先**のバックエンドサーバーを定義します
serverは複数表記可能で<name>でグループ名を定義します
この設定ではラウンドロビン（均等負荷分散）、2つのサーバーで均等に負荷を分散することになります
（転送先の重み付けなどを行うこともできるようです）
またserverのパラメータにはhost:portのほか、ソケットを使用することも可能です

```
upstream puma {
    server app:3000;
    server app:3001;
}
```

２つ目にserverブロック内の、`proxy_pass`でリクエストの転送を宣言します
（後の内容で理解が深まりましたが、ここではこの仮想サバーで受けたリクエストをすべて転送する設定になっています）
`puma`の部分はupstremの<name>にあたります

```
proxy_pass http://puma;
```

3つめに、Nginx自身はドキュメントルートを持ちません
Nginxはバックエンドサーバーに代わって代理でコンテンツを配信しているように振る舞いますが（プロキシ）
Nginx自身が直接コンテンツを配信しないのでドキュメントルートが必要ないということかと思います

> 参考 - Nginx公式-
> [Using nginx as HTTP load balancer](https://nginx.org/en/docs/http/load_balancing.html)



### Rails appで適当な動的コンテンツを作成

**ルーティング**`config/routes.rb`

```ruby
Rails.application.routes.draw do
  root "try_nginx#index" #追記
end
```

**コントローラー:** `app/controllers/try_nginx_controller.rb`(新規作成)

```
docker-compose run app rails g controller try_nginx index
```

以下の内容で生成されます

```ruby
class TryNginxController < ApplicationController
  def index
  end
end
```

**ビュー:** `app/views/try_nginx/index.html.erb`

すごく適当です
`rails g controller`で生成された`index.html`を利用します
`<%= Time.now %>`でRailsが現在時刻を動的に生成し表示します

```erb
<h1>I am Rails</h1>
<p><%= Time.now %></p>
```

もう一つ仕掛けを
`app/javascript/stylesheets/application.scss`を編集しpタグにスタイルをあてます

```scss
p {
  color: red;
}
```

あと少し、
config/environments/development.rbに以下の一行を挿入します
（ホストのホワイトリストのようですが、これがないとエラーが出ます。必要な理由はいまいちわかっていません）

```
config.hosts << "puma"
```

> 参考: [cloud9 - Upgraded Rails to 6, getting Blocked host Error - Stack Overflow](https://stackoverflow.com/questions/53878453/upgraded-rails-to-6-getting-blocked-host-error)

これで下準備は終了です！



### 動作確認

今回はすべてのコンテナを起動します

```
docker-compose up
```

ブラウザで`localhost`もしくは`localhost:80`にアクセスすると

<img>

`I am Rails`とともに、現在時刻が赤字で表示されると思います
この時刻はリロードするたびに、バックエンドのRails appが生成しているという点で動的コンテンツと位置づけています

**Nginxがlocalhost:80で受けたリクエストをバックエンドのRailsアプリに転送し、Railsが生成した動的コンテナをNginxがクライアントに返す**という挙動が確認できたかと思います



### appendix - ブラウザはSCSSを読める？

**ブラウザはCSSしか読めないはずです**
思い出してほしいのですが、スタイルを記述したのは`application.scss`
なぜ、先程の動作確認でスタイルが適応されていたのでしょうか？

**開発環境ではCSSやJavaScriptをブラウザが読める形に随時変換（コンパイル）する仕組みがあります**

Railsでの開発環境のことしかわかりませんが
Rails6ではwebpackerというgemに含まれるwebpack-dev-serverがこの働きをしています
webpack-dev-serverはディレクトリを監視し、変更を検出すると自動でコンパイルし、**ブラウザが読めるように提供します**

やや曖昧な表現にしたのは理由があって、それをコンテナに入って確認してみます

```
docker-compose exec app bash
```

```
ls  /myapp/public

404.html
422.html
500.html
apple-touch-icon-precomposed.png
apple-touch-icon.png
favicon.ico
packs
robots.txt
```

```
ls  /myapp/public/packs/

manifest.json
```

`public/`にも, `public/packs/`にもコンパイルされたファイル*.cssは存在していないのです
あるのはwebpackがコンパイルする際の設計図にあたる`manifest.json`のみ

これについて、**webpack-dev-serverは、CSSやJavaScriptを随時自動コンパイルするが、
ファイルの実体を出力しているわけではなく、メモリ上にコンパイルした結果を出力しているとのことです**

ソースが随時変更される開発環境においてはとても便利な仕組みに感じますが、
デプロイ環境では頻繁にソースに手が加えられるわけではないのでこの仕組は不要かと思います
よって、この機能は開発環境でのみ有効になるように設定されています

こういった開発環境と、デプロイ環境での挙動の違いは、初めてアプリをデプロイする際に躓いたポイントでもあるので
なんとなく頭の片隅に留めていただけるといいかと思います



## リバースプロキシとしてのNginx





```
docker-compose exec app rails assets:precompile
```

### 参考

[nginxについてまとめ(設定編) - Qiita](https://qiita.com/morrr/items/7c97f0d2e46f7a8ec967)

[リバースプロキシ（Reverse Proxy）：90秒の動画で学ぶITキーワード - ＠IT](https://www.atmarkit.co.jp/ait/articles/1608/25/news034.html)

[リバースプロキシって何？触りだけ学んだサーバー/インフラ入門 - Qiita](https://qiita.com/growsic/items/fead30272a5fa374ac7b)

## Troubleshoot

### Railsアプリの起動時に以下のエラーが表示される

> ActionView::Template::Error (Webpacker can't find application in /myapp/public/packs/manifest.json. Possible causes:
>
> 1. You want to set webpacker.yml value of compile to true for your environment
>    unless you are using the `webpack -w` or the webpack-dev-server.
> 2. webpack has not yet re-run to reflect updates.
> 3. You have misconfigured Webpacker's config/webpacker.yml file.
> 4. Your webpack configuration is not creating a manifest.
>
> ...

初回起動時に絞って考えると幾つか原因が考えられます

- yarn, webpackerのインストールがうまくできていない

  ```
  # local
  rails webpacker:install
  
  # docker
  docker-compose run app rails webpacker:install
  ```

  

- 上記は問題なく、webpack-dev-server（のコンテナ）は起動しているが、`manifest`の中身に問題がある。（空の場合がある）

  ```
  # local
  rails webpacker:compile
  
  # docker
  docker-compose run  app rails webpacker:compile
  ```

- Docker環境に起因 (development環境)
  `app: Rails`と`webpacker: webpack-dev-server`間でボリュームを共有していない

  以下のような状況で再現性が得られることがわかりました。

  

  1. Railsコンテナと、webpack-dev-serverコンテナは`volumes: - .:/myapp`でローカルのディレクトリをマウントしている。
  2. 名前付きボリューム`volumes: - public:/myapp/public`でNginxとRailsはボリュームを共有している。
  3. この状態で、2.は1.から切り離されたボリュームとなり、ローカルとコンテナ内は同期されない。
  4. 結果webpack-dev-serverコンテナから、`/myapp/public`が見えない。

  

  解決

  名前付きボリュームをNginx - Rails - webpack-dev-serverの全てで共有する


  docker-compose.yml

  ```yml
  version: "3.8"
  
  services:
    web:
      image: nginx:1.18
      volumes:
        - public:/myapp/public
        
  ...
  
    app:
      build:
        context: .
      image: rails_app
      volumes:
        - .:/myapp:cached
        - public:/myapp/public
        
  ...
  
    webpacker:
      image: rails_app
      command: ./bin/webpack-dev-server
      volumes:
        - .:/myapp:cached
        - public:/myapp/public #これが必要
  
  
  volumes:
    public:
  
  ```

  production環境ではそもそも`webpack-dev-server`を使用せず、precompileする方法をとると思うのでこの限りでは無いです

```
curl -v localhost:80
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 80 (#0)
> GET / HTTP/1.1
> Host: localhost
> User-Agent: curl/7.64.1
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.18.0
< Date: Tue, 22 Sep 2020 19:26:03 GMT
< Content-Type: text/html; charset=utf-8
< Transfer-Encoding: chunked
< Connection: keep-alive
< X-Frame-Options: SAMEORIGIN
< X-XSS-Protection: 1; mode=block
< X-Content-Type-Options: nosniff
< X-Download-Options: noopen
< X-Permitted-Cross-Domain-Policies: none
< Referrer-Policy: strict-origin-when-cross-origin
< ETag: W/"37063413ebb7a2403534fb24e66a04fa"
< Cache-Control: max-age=0, private, must-revalidate
< Set-Cookie: _myapp_session=qD5omg%2BFRQgHLZ9z%2FqQFa3sG3Dxa7eozevGQrI%2FtAfDQGDWQfhx6BDpfTCV%2BedwZw8YFgx1qcejO6LY4Y1ernGx7qxU34WVWAwqHs9HL1I9cyDcOmnI18gmnUUH66GGKCQ4KOrB%2Bg%2F2LQvyfHxTPx6UwvkATVnkabqqi8BNEP9z%2BphprvOaoLCR50GXMIesPnrK%2Fo%2F5q7OpXrtSs7U1HvF7MUFlpQHr0uAUXE%2Fb%2BI7p8iXXoL5Cnsax4KyN2lqQLk%2BWvEyZkgt3gHkFCR13a3tTK7qWmgw%3D%3D--DL%2Fn4F6dHnpQu4Up--%2BO9c9yWtlKaTtIuWyH2XJA%3D%3D; path=/; HttpOnly
< X-Request-Id: 9fb7ab2f-0956-417e-80b0-637d1b0ab0a0
< X-Runtime: 0.142430
< 
```

#### RHEL/CentOS

/etc/nginx/nginx.conf