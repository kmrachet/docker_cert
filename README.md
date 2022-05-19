# ルート証明書入りDocekrイメージの作り方
## 手動でやる方法
1. ルート証明書`XXXX.crt`をダウンロードする
2. Docker Desktopを起動する<br>
(Docker Engineを常駐させている場合は不要)
3. コマンドプロンプトを起動し、ルート証明書があるフォルダに移動<br>
```
cd C:\Users\user1\Downloads
```
`user1`にはログインしているアカウント名を入れる。ルート証明書の場所は任意で
4. ルート証明書を入れたいDockerイメージからコンテナを立ち上げる<br>
```
docker container run -d -e PASSWORD=password -p 8787:8787 --rm --name certplus ykunisato/paper-r:latest
```
    - `-d`：Dockerコンテナを起動するときにデタッチする
    - `-e`：環境変数を`変数名=値`の書式で引数に指定する
    - `-p`：`ローカルマシン側のポート番号:Dockerコンテナ側のポート番号`の書式でポート番号を引数に指定する
    - `--rm`：Dockerコンテナが停止したとき自動で削除する
    - `--name`：Dockerコンテナの名前を引数に指定する
    - `ykunisato/paper-r:latest`：ルート証明書をインストールしたいDockerリポジトリやイメージ、タグを指定する
5. ルート証明書をコンテナにコピーし設定を更新<br>
以下の作業が必要
    - Dockerコンテナ内にルート証明書`XXXX.crt`をコピーする
    - ルート証明書を`/usr/share/ca-certificates`ディレクトリに作成した新規ディレクトリ`xxxx`内に配置する
    - 証明書の設定ファイル`/etc/ca-certificates.conf`に証明書の場所情報(`/usr/share/ca-certificates`より下のルート証明書のパス)を追記
    - 証明書の設定ファイルに基づいて設定を更新する

    上記をまとめたものが↓
```
docker container exec -d certplus bash -c \
"mkdir /usr/share/ca-certificates/xxxx && echo xxxx/XXXX.crt >> /etc/ca-certificates.conf" && \
docker container cp XXXX.crt certplus:/usr/share/ca-certificates/xxxx/. && \
docker container exec -d certplus update-ca-certificates
```

    - `docker container exec`：DockerコンテナにLinuxコマンドを実行させる。
    - `mkdir`：ディレクトリを作成する。
    - `echo hoge >> tmp.conf`：`hoge`を`tmp.conf`ファイルの末尾に追記する。
    - `docker container cp`：ローカルマシンのファイルをコンテナ内にコピーする。
    - `update-ca-certificates`：証明書情報を`/etc/ca-certificates.conf`に記載されたとおりに更新する。
6. Dockerコンテナからイメージを作成する<br>
```
docker container commit certplus user1/paper-r:latest-cert
```
Dockerイメージは`作成したユーザ名/コンテナ名:タグ名`が通例
7. Dockerイメージ作成用のコンテナを停止する<br>
```
docker container stop certplus
```
`certplus`コンテナを立ち上げるときに`--rm`オプションを指定しているため`stop`コマンドのみで削除される。`--rm`オプションを指定していないコンテナを削除する場合は`docker container rm コンテナ名`で。
8. 作成したDockerイメージからコンテナを立ち上げる<br>
Docker DesktopでOK。`user1/paper-r:latest-cert`イメージをRUN→オプションでポート番号とマウント先のフォルダを指定する。`PASSWORD`環境変数は先に指定しているため不要。<br>
コマンドプロンプトで実行する場合は以下<br>
```
docker container run -d -p 8787:8787 -v "%cd%":/home/rstudio user1/paper-r:latest-cert
```

## Dockerfileを使う方法
既存のDockerリポジトリから新しいイメージを作成するための設定ファイルを使ってイメージを作成する。<br>
Dockerコンテナのデフォルトユーザが`root`じゃないとうまくいかない。デフォルトユーザの確認は「手動でやる方法」の1.～4.でDockerコンテナを起動→`docker exec -it コンテナ名 bash`でコンテナにアタッチしたあとターミナルで`whoami`コマンドを入力する。

1. 「手動でやる方法」1.～2.を実行する
2. ローカルマシンのフォルダ構成を以下のようにする

    Dockerfileとルート証明書を同じフォルダに入れておく(以下例)。

```
Downloads
└ certplus
　 ├ Dockerfile
　 └ XXXX.crt
```

3. Docekrfileを作成

    テキストエディタ(メモ帳等)で以下をコピペしたDockerfileを作成する。Dockerfileには拡張子をつけない。

```
FROM ykunisato/paper-r:latest

COPY XXXX.crt /usr/share/ca-certificates/xxxx/
RUN echo "xxxx/XXXX.crt" >> /etc/ca-certificates.conf && update-ca-certificates
ENV PASSWORD=password
```

    - `FROM`：ベースとなるDockerリポジトリ/イメージとタグを指定する。
    - `COPY`：Dockerfileと同じフォルダ内の指定したファイルをDockerコンテナ内にコピーする。
    - `RUN`：Dockerコンテナの中でLinuxコマンドを実行する。
    - `ENV`：環境変数を設定する。

4. DockerfileからDockerイメージを構築する<br>
```
docker image build -t user1/paper-r:latest-cert .
```
    - `-t`：構築するDockerイメージ名、タグ名を引数に指定する。
5. 構築したDockerイメージからコンテナを立ち上げる<br>
「手動でやる方法」 8.に同じ。
