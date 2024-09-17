# nginxを使用したWEBサーバチューニングの学習

## 環境
```
PC: MacBook Pro
チップ: Apple M1
macOS: Sonoma 14.3.1
```

## 手順
1. Homebrewでnginxをインストールする。
```
brew install nginx
```
※ `llvm`のインストール(というかビルド)にとんでもなく時間がかかった。2時間経っても`llvm`のビルドが完了しなかったのでnginxのインストールを中断し、以下でHomebrewのキャッシュを掃除してからnginxをインストールしなおしたら40分くらいで無事に完了した。
```
brew cleanup
```
2. nginxを起動する
```
brew services start nginx
```
ブラウザで`http://localhost:8080`にアクセスすると、

> Welcome to nginx!

などと表示されるのを確認する。

3. nginxの設定ファイルを編集
`/usr/local/etc/nginx/`に`nginx.conf`ファイルがあるので、以下を追記することで、ブラウザでnginxのステータスページを確認できるようにする。
```
http {
    server {
        location /nginx_status {
            stub_status on;
            access_log off;
            allow 127.0.0.1;
            deny all;
        }
    }
}
```
4. nginxを再起動
```
brew services restart nginx
```
ブラウザで`http://localhost:8080/nginx_status`にアクセスし、ステータスページを確認する。

## ステータスページ
ブラウザで`http://localhost:8080/nginx_status`にアクセスすると、以下のようにステータスが表示される。
```
Active connections: 2 
server accepts handled requests
 2 2 1 
Reading: 0 Writing: 1 Waiting: 1 
```
### 1. Active connections: 2
現在nginxが処理しているアクティブな接続の数を示している。この場合、2つのアクティブな接続があることを意味する。  
アクティブな接続には、リクエストの処理中、リクエストの読み込み中、レスポンスの書き込み中、待機状態の接続が含まれる。

### 2. server accepts handled requests
```
 2 2 1
```
- 2: nginxがサーバーの起動以来、受け入れた接続（accept）の総数
- 2: 受け入れた接続のうち、正常に処理された（handled）接続の数
- 1: 処理されたリクエストの数

通常であれば`accept`と`handled`は同じ値になる。

### 3. Reading: 0 Writing: 1 Waiting: 1
- Reading: クライアントからのリクエストヘッダを読み込んでいる接続数
- Writing: クライアントへのレスポンスを書き込んでいる接続数
- Waiting: keep-alive接続で、次のリクエストを待っている接続数

## 試験ツール
試験ツールを回してみて、ステータスの変化を確認してみる。`Apache Bench`を使用してみた。
```
ab -n 1000 -c 100 http://localhost:8080/
```
- `-n`: リクエスト回数
- `c`: 同時接続数

`Apache Bench`の実行後、ステータスページは以下のように変化した。
```
Active connections: 2 
server accepts handled requests
 1004 1004 1002 
Reading: 0 Writing: 1 Waiting: 1 
```

## チューニング
### worker_processes
nginxが同時に処理できるプロセス数。`auto`と設定することで、使用しているシステムのCPUコア数に基づいてnginxが自動的に適切なワーカー数を設定してくれる。

#### チューニングの判断基準の例
- サーバーが遅くて`Active connections`の数値が高い場合
  - ワーカー数が不足している可能性があるので、サーバーのコア数を最大値として`worker_processes`の値を増やす対策を提案できるかも。

サーバーのコア数を確認するコマンド:
```
sysctl -n hw.ncpu
```
---
### worker_connections
1つのワーカープロセスが同時に処理できる接続数。

#### チューニングの判断基準の例
- `Active connections`が非常に多く、`Reading`や`Writing`が停滞している場合
  - 多くの接続を処理しきれていない可能性があるので、`worker_connections`の値を増やす対策を提案できるかも。
---
### keepalive_timeout
クライアントが接続を維持する時間。`keep-alive接続`により、クライアントは同じ接続を使って複数のリクエストを送信でき、パフォーマンスが向上するが、長時間接続を維持するとリソースが浪費される。

#### チューニングの判断基準の例
- `Waiting`が多い場合
  - keep-alive接続が過剰に残されている可能性があるため、`keepalive_timeout`の値を短く設定し、未使用の接続をすぐに閉じ、新しい接続を効率的に処理するよう提案できるかも。
- リクエストが少なく、各接続で処理されるデータが多い場合
  - `keepalive_timeout`の値を長めに設定することで、再接続のオーバーヘッドを減らす提案ができるかも。
---
### gzip
`gzip`を有効にすると、サーバーからクライアントに送信されるデータが圧縮され、ネットワーク帯域幅を節約できる。ただし、圧縮にはCPUリソースが必要になるため、負荷が高い環境では調整が必要。

#### チューニングの判断基準の例
- `Writing`が非常に多い場合
  - クライアントへのレスポンスが遅れている可能性があるため、`gzip`を有効にすることで、データ転送量を減らして、レスポンスを効率化する提案ができるかも。
- CPU使用率が高い場合
  - `gzip`を無効にするか、圧縮対象のファイルを大きなファイルのみに制限するなどの提案ができるかも。

`gzip`を有効にし、ファイルサイズが1000バイト以上のデータにのみ圧縮を適用する設定:
```
gzip on;
gzip_min_length 1000;
```
## nginxを停止
- 全ての接続を強制的に終了してnginxを即座に停止させる
```
nginx -s stop
```
- 接続中のリクエストを処理し終わってからnginxを停止させる
```
nginx -s quit
```