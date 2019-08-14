初回bench score 4100 くらい。

register の高速化をしたいのでパスワードのhash化ロジックを変更. random salt発行をなしに。

image 周りがフルスキャンになっていたので、IDを参照するようにするのが理想っぽいけど、とりあえず name に index貼って違いがどうなるかみてみる。
mysql> create index idx_name on image (name(10));

ここで  "score": 10432 おお。単純に index追加だけでも早くなっている・・・（もう一回とったら5000くらいだったけど...）

top での負荷状況は、mysqld がCPU使いまくっている。
  622 mysql     20   0 1617444 337340  15312 S  82.4 16.5  11:30.28 mysqld
 1907 isucon    20   0  351024  22316  12720 S  15.3  1.1   0:22.65 php-fpm

やりたい
* icon が blobdata として入っているので吸い出してnginxで配信する
* 画像のnginxキャッシュで配信。cache control 周り
* /login が結構タイムアウトしている。処理的には重くなさそうなのでDBロックか何かで待たされているのかもしれない。それか単に過負荷
* font js css の配信は？ nginxの設定みると{}だけが書かれているが
* 画像サイズがメッセージ一覧では横100が固定か。不要に大きなものが配信されて帯域食っているかも

iconの吸い出し（blobデータをmysqlから引っ張ってくるのはうまくいかなかったのでlocalネットワーク経由で）
```
$ for name in `sudo mysql -uroot isubata -e "select name from image"` ; do echo $name;  curl  http://localhost/icons/$name --output - > icons/$name  ; done
```


```
isucon@iZ6we7qggdtabs3ysvx7mkZ:~/isubata/webapp/php$ diff  /etc/nginx/sites-available/nginx.php.conf{.org,}
14a15,18
> 	location /icons/ {
> 	   root /home/isucon/isubata/webapp/php/;
>         }
>
```
この時点でのscore
isucon@iZ6we7qggdtabs3ysvx7mkZ:~/isubata/bench$ jq . < result.json  | grep score
  "score": 8702,

上がってねえ〜　キャッシュうまく乗っていないとかもあるのかな。
ただ、iconのタイムアウトは無くなっていたのでOK。

画像サイズを変更。横幅100が使われているのでそれに直す.  => bench の 画像Validationで落ちるので無し
```
 for name in `ls *.*` ; do convert $name -resize 100 ../icons/$name ; done
```

cache control はnginx に画像の設定ブロックに以下の行を追加するだけ
```
expires 30d;
add_header Cache-Control public;
```
benchに影響無し。多分/profileで更新されるからキャッシュが聞いていない => image が同じハッシュ値、つまり同じ中身のファイルなら更新しないようにするようにした。スコアあまり変わらず。
ついでにfaviconやCSSにも同じ設定を入れて起き, iconsをpublic配下に移動


MYSQLチューニング
> transaction-isolation="READ-UNCOMMITTED" を追加
スコア変わらず。むしろ悪化？ よくわかっていないのであとで調べておく。

DBのbuffer pool以外で重要な innodb_flush_log_at_trx_commit=2 を追加
んー　スコア変わらず。そもそもIOがボトルネックになってないか。
典型的な sar の状態は
```
01:53:35 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
01:53:36 PM     all     40.20      0.00      1.01      0.00      0.00     58.79
01:53:36 PM       0     52.53      0.00      2.02      0.00      0.00     45.45
01:53:36 PM       1     27.18      0.00      1.94      0.00      0.00     70.87
```

CPU使い切っていないのでインフラ的な接続数がボトルネックっぽい？
```
isucon@iZ6we7qggdtabs3ysvx7mkZ:~/isubata/webapp/php$ netstat -ant | grep EST -c
439
```
```
isucon@iZ6we7qggdtabs3ysvx7mkZ:~/isubata/webapp/php$ diff /etc/sysctl.conf{.org,}
12c12,13
< net.ipv4.tcp_max_tw_buckets = 5000
---
> # net.ipv4.tcp_max_tw_buckets = 5000
> net.ipv4.tcp_max_tw_buckets = 2000000
14c15
< net.ipv4.tcp_max_syn_backlog = 1024
---
> net.ipv4.tcp_max_syn_backlog = 10000
23a25,29
> net.ipv4.ip_local_port_range = 10000 65000
> net.core.somaxconn = 32768
> net.core.netdev_max_backlog = 8192
> net.ipv4.tcp_tw_reuse = 1
> net.ipv4.tcp_fin_timeout = 10
```

```
isucon@iZ6we7qggdtabs3ysvx7mkZ:~/isubata/webapp/php$ diff /etc/nginx/nginx.conf{.org,}
7c7
< 	worker_connections 768;
---
> 	worker_connections 10000;
```

slow query log を入れる
slow_query_log                = 1
slow_query_log_file           = /var/log/mysql/mysqld-slow.log
long_query_time               = 1
// 他の単にslow query は出ていなかったが、message の count(*)がindex使っていないことが判明
log-queries-not-using-indexes = 1
ここら辺？
```
SELECT COUNT(*) as cnt FROM message WHERE channel_id = '2091'
```
1秒に1回未読とかのリストを取っているのに使っているっぽい。影響度は高くなさそう。
後から考えて、やっぱり他に影響もするので直しておくという気持ちに
```
mysql> CREATE INDEX idx_channel_id ON message(channel_id);
```

さて、そしたら影響度の高いところはどれだ？ という発想になり、一回のbenchでのアクセスログを漁ってみるか
```
isucon@iZ6we7qggdtabs3ysvx7mkZ:~/isubata/webapp/php$ awk -F' ' '{print $6, $7}' /var/log/nginx/access.log | awk -F'/' '{print $1, $2}' | sort | uniq -c
      1
      2 "GET
      3 "GET  add_channel
     62 "GET  channel
    116 "GET  css
     58 "GET  favicon.ico
    416 "GET  fetch
    290 "GET  fonts
     21 "GET  history
   2550 "GET  icons
      1 "GET  initialize
    232 "GET  js
      2 "GET  login
      9 "GET  logout
     24 "GET  message?channel_id=10&last_message_id=0
     13 "GET  message?channel_id=10&last_message_id=9980
　　　中略 10こくらい
     20 "GET  message?channel_id=2&last_message_id=0
     13 "GET  message?channel_id=2&last_message_id=9998
      3 "GET  message?channel_id=5&last_message_id=0
      2 "GET  message?channel_id=5&last_message_id=9994
     16 "GET  profile
      2 "GET  register
     94 "POST  add_channel
    211 "POST  login
     21 "POST  message
     12 "POST  profile
     17 "POST  register
```
ポイント的には、 優先順位は
* login（POSTなので）
* font （数が多い）
* fetch （同様） => とここでレギュレーションを見ると、ここはポイント加算に無関係とのこと
あと 静的ファイルに関しては 302だと 点数が 1/100 らしいので。iconのcacheのチューニング無駄っぽい（負荷低減に役立つので必要ではあるが）。
一旦loginのチューニングやる。
ログインセッションをredisに持たせてDB叩かないようにする。
まずredisのインストールから。
apt install redis 
起動しないのでsudo chown redis /var/run/redis
```
maxmemory 50mb
supervised systemd
bind 127.0.0.1　（::1が入っているとip6見つけられなくてタイムアウトする）
```
コードを書き換えて、bench通る。めっちゃCPU不可落ちた。けどscore上がらない。コネクション数問題か。

ここら辺参考にする
https://blog.yuuk.io/entry/web-operations-isucon
