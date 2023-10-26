# Lets EncryptでEC2を無料でSSL化

# 問題点
EC2単体を利用してデプロイされたDockerアプリケーションをSSL化する際に、public subnetでALSを使用していないのでACMでSSL化できない。

# アプローチ
Let's Encryptを使用してSSL化

# 動作環境
- EC2
- Amazon linux 2023
- nginx

# 具体的な方法
## 1. 必要なパッケージのインストール

Let's Encryptを使用するためにはCertbotを実行する必要があります。
しかしながら、EC2を起動した際にデフォルトで選択されているAmazon linux 2023ではCertbotをyumでインストールすることができなくなりました（Amazon linux 2であれば直接Certbotをyumでインストールできます）
そのため、pythonをインストールしてからpipでインストールする必要があります。

~~~bash
sudo yum install -y python3 augeas-libs pip
~~~

## 2. Certbotのインストール

先ほどインストールしたpythonを使用して、Certbotをインストールします。

~~~bash
python3 -m venv /opt/certbot/
/opt/certbot/bin/pip install --upgrade pip
/opt/certbot/bin/pip install certbot
ln -s /opt/certbot/bin/certbot /usr/bin/certbot
~~~

## 3. Webサーバーの停止

nginxの設定ファイルを操作する必要があるため、一旦サーバーを停止します。

nginx
~~~bash
systemctl stop nginx
~~~

## 4. Certbotの実行

サーバーを停止したことが確認できたら、インストールしたCertbotを実行します。
いくつか質問がされますので、以下の手順に沿って回答してください。

~~~bash
certbot certonly --standalone -d backend.milklog.jp
~~~

### 質問の内容と解答例

1) 証明書の期限切れを通知するメールアドレス
    → メールアドレスを入力します。

2) 利用規約に同意しますか？
    → Y（同意します）

3) メールアドレス情報を共有しますか？
    → N（共有しません）

4) 証明書に記載するドメイン名を入力してください。
    → example.com（SSL化したいドメイン名を入力）

Successfully received certificate. と出力されれば完了です。
これで SSL 証明書が取得できました。
証明書は/etc/letsencrypt/live/{ドメイン名}配下に作成されます。

## 5. Nginxの設定

Cerbotを使用してSSL証明書を取得することができたので、次はサーバーにSSLを設定します。
nanoを使用してnginxの設定ファイルを操作します。
vimに慣れている方はnanoの部分をviにすることでvimで編集することができるようになります。

~~~bash
sudo nano /etc/nginx/sites-available/default
~~~

### nanoの操作方法
    1. Ctrl + Oキーを押してファイルを保存するプロンプトを表示します。
    2. エンターキーを押して、変更を保存します。
    3. Ctrl + Xキーを押して、nanoエディタを終了します。

### vimの操作方法
    1. i(insert)キーを押して編集モードにします。
    2. コードを編集します。
    3. escキーを押して編集モードを終了します。
    4. :wでファイルを上書き保存します。
    5. :qでvimを終了します。

:wqでも保存して終了することができます。

## 6. 以下の内容をファイルに追加/修正してください

~~~
server {
    ...
    server_name backend.milklog.jp;
    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/backend.milklog.jp/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/backend.milklog.jp/privkey.pem;
    ...
}
~~~

追加、修正したら以下のコマンドを実行して正常に動作しているかを確認します。
~~~bash
sudo nginx -t
~~~

問題がなければ設定を反映するためにサーバー（今回はnginxを使用しています）を再起動します。
~~~bash
sudo systemctl reload nginx
~~~

## 7. Webサーバーの起動:
~~~bash
systemctl start nginx
~~~

## 8. 証明書の更新の自動化

Let's Encryptで作成したSSL証明書は3か月という期限が存在し、それを過ぎると自動で無効になってしまいます。
なので、SSL証明書を再取得するcroneを作成しましょう。

~~~bash
sudo yum install cronie-noanacron
systemctl enable crond
systemctl start crond
crontab -e
~~~

crontab -eでcronの設定を開くことができるので、以下の行をcrontabファイルに追加してください
~~~
30 1 * * * root certbot renew --post-hook "systemctl reload nginx" --no-self-upgrade
~~~

# ソース
[Amazon Linux 2023 に Let’s Encrypt をインストールする](https://softwarenote.info/p3954/)