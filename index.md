# localhost の設定を放置して失敗した話
当チームのサーバーでは nginx を使用している。導入当初、稼働確認などで使用するために localhost でアクセスできるように設定していた。
DNS の設定および、テスト用・本番用の各WEB サーバーの設定も完了した後は、稼働確認が必要になることも無くなり localhost にアクセスすることも無くなった。
ただ、今後も稼働確認が必要になるケースも想定されるので、localhost の設定はそのまま放置してあった。
ちなみに、**localhost の root は、本番用のWEBサーバー（www.ドメイン）と同じ場所を使うようにしてあった**。
## エラーの発生
ログのチェックをしている中で、不審なエラーが出るようになった。サーバー上で稼働している WEBサイトでは Google Firebase の Analytics を使用している。ここで使用している APIキー には HTTP リファラーで制限を設けてあるが、この利用制限違反が原因のエラーが発生するようになったのだ。
ログを追っていくと、"localhost" からのアクセスでこのエラーが出ていた。しかもクライアントは Linux 。我々の環境では、サーバーにログインできるのはインフラ担当者のみで、アプリ担当者はログイン禁止にしている。このエラーは有り得ない。まさか、ハッキングされたのでは？と一瞬焦った。
さらにこのエラーを出している元を追跡していったところ、どうやらウェブクローラ [[Google公式：検索が情報を整理する仕組み]](https://www.google.com/intl/ja/search/howsearchworks/crawling-indexing/) だという結論に達した。ウェブクローラが、`hssp://xxx.xxx.xxx.xxx` にアクセスしてきたことで "localhost" の WEBサーバーが使用されていたのだ。

1. ハッカーにやられている訳ではなく、セキュリティ上はなんの問題もない。一般に公開しているページが見られているだけのことである
1. エラーは出ているが、一般のユーザーの利用で出ているエラーではない
1. Analytics のデータは取れないが、むしろウェブクローラのアクセス数はカウントから除外したいくらいなので好都合

と言うことで、実害がないと判断してエラーを放置した。**これがいけなかった**
##問題勃発
最近、Googleで検索していたら当チームのWEBサイトがヒットした。これ自体は喜ぶべきことなのだが、そこでリンクされていたアドレスが**なんと`http://xxx.xxx.xxx.xxx/~`だったのだ！**
ウェブクローラはちゃんと仕事をしていたのだ。
これはヤバい。エラーを放置していたことがバレてしまう。
いや、システムの運用上ヤバいのだ。

1. せっかく本番用のアドレスは https で用意してあるのに http でアクセスされてしまう
1. サーバーは引越すことを想定している。固定アドレスが広まってしまうと困る
1. 一般ユーザーが利用しているときにエラーを出す訳には行かない。HTTP リファラーでの利用制限を前提としている以上、一般ユーザーにlocalhost を使われては非常に困る

なんとかリカバリーしなければ・・・
##対処方法
###reverse proxy
取り急ぎ nginx の localhost に reverse procy を設定した。www.ドメイン のアドレスに飛ばすようにしたのだ
APIキーの HTTP リファラーのエラーは解消したが、これはこれで問題が残る

1. **http://~** のアクセスには対応できるが、**https://~** では別の問題が生じる。**localhost** に対してはCA認証済みの証明書は発行されないのだ。**自己証明書**になってしまうので、ブラウザに「この接続ではプライバシーが保護されません」のエラーが出てしまう。
1. 引き続き`http://xxx.xxx.xxx.xxx/~`でアクセスするとちゃんとページが表示されるので、ウェブクローラは正当なサイトだと認識し続けるのではないか？そうすると、検索結果としてのこのページが表示されたままになる。将来サーバーを引越す場合に障壁になってしまう。

何か別の方法を考えなければ・・・
###Hello World
`http://xxx.xxx.xxx.xxx/~`にアクセスしてきた場合に、Hello world を表示するように変更した。
これなら稼働確認として localhost を使用できるし、意味のないページなのでウェブクローラもいずれは検索結果から除外してくれるのではないか？と期待
しかし、これはこれで問題がある

1. 検索結果からアクセスしてくれた一般ユーザーの方が「なんじゃこりゃ」となる
1. アクセスしても一応正常なステータスでHello World が返ってくるので、ウェブクローラもアクセスしてはダメなサイトだと認識してくれないかもしれない

何か別の方法を考えなければ・・・
###オリジナルの404エラー画面を表示
最終的には`http://xxx.xxx.xxx.xxx/~` や `https://xxx.xxx.xxx.xxx/~` でアクセスされた場合には、オリジナルの404エラーのページを出すように設定した。出しているのはこんなページ。

```
An error occurred.
Sorry, you can not use the IP address for the page you are looking for.
Please try domain name like "https://www.[ドメイン]/~".　＜＝ 実際のURL を記載

Faithfully yours.
```
ご参考までに、オリジナルのエラーページを出す nginx の設定は以下の通り

```server.conf
server {
    listen       443  ssl;
    access_log  /var/log/nginx/localhost.access.log  main;
    error_log  /var/log/nginx/localhost.error.log;
    ssl_certificate /etc/nginx/ssl/nginx.pem;      # 証明書は自己証明書
    ssl_certificate_key /etc/nginx/ssl/nginx.key;
    server_name  localhost;    
    error_page 404 /404.html;                      # オリジナルのエラーページを宣言
    location / {
        return 404;                                # 常に 404 を返す
    }
    location = /404.html {                         # エラーページの格納先
        root   /usr/share/nginx/localhost;         # ここに 404.html を保存しておく
        internal;
    }
}
server {
    listen       80;
    server_name  localhost;
    access_log  /var/log/nginx/localhost.access.log  main;
    error_log  /var/log/nginx/localhost.error.log;
    error_page 404 /404.html;
    location / {
        return 404;
    }
    location = /404.html {
        root   /usr/share/nginx/localhost;
        index  index.html index.htm;
    }
}
```

オリジナルのエラーページなので、

1. 稼働確認や接続テストの場合には正しく動いたかどうか確認ができる
1. nginx オリジナルの50x.html と似ているので、世間的に受け入れられ易いのではないか
1. エラーの理由と対処法を記載してあるので「なんじゃこりゃ」にはならないのではないか
1. エラーで返しているので、アクセスしてはダメなページだとウェブクローラにも判るのではないか

と言うことで、この状態でウェブクローラが検索結果を修正してくれるのを待つことにした。

####あとがき
最初からこういう設定にしておけば良かったと後悔しています。もし、localhostのWEBサーバーを本番と同じ内容で設定している人がいたら、今すぐ変更することをお勧めします。

こちらのwebsiteもよろしくお願いします。
[[初心者のためのWEBシステムのインフラ構築]](https://www.olto3-sugi3.tk/ja/index.html)
