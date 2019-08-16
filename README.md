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

cache control はnginx に画像の設定ブロックに以下の行を追加するだけ。当時はCDNが外に挟まっていて、CacheControl publicがないときかないというところで足したチームが多かった見たい（ここら辺カンニングした）
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

ここら辺途中現実逃避。問題の本質から目を背けてMYSQLさらにみる。

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

nginx の multi_accept on という設定がコメントアウトされていたので入れる。

php fpm 設定のプロセス数が5で固定されていたので、20にとりあえず増やした。
```
$ grep max  /home/isucon/local/php/etc/isubata.php-fpm.conf
pm.max_children = 20
```
```
 2482 isucon    20   0  348988  17520  10112 S   2.0  0.9   0:00.53 php-fpm
 2490 isucon    20   0  348988  17128   9720 S   2.0  0.8   0:00.55 php-fpm
```
1プロセスあたりの消費CPUとメモリがそれぞれ 2%と1.7% くらい。nginxだけならば40までは余裕を持って増やせそう。
ここでbenchとってみると
  "score": 43517,　とやっといいスコアがでた。


ここでDBのサーバを別に立ち上げ、構成変更を試す
```
apt update
apt install git
apt install ansible
```
site.ymlにcommonだけ入れて、/etc/ansible/hosts　に127.0.0.1を追加（必要ない気がするが何も書いてないとダメなのか？）
```
ansible-playbook site.yml --connection=local
```
isuconユーザが作られるので、移動して、
`git clone https://github.com/isucon/isucon7-qualify.git isubata`
こちらのユーザで、今度はsite.ymlにmysqlだけ入れて流し込み => table などはいらなかったので一回drop table。ちゃんと流し込む中身見た方が良い。
元のサーバでmysqldump
```
 mysqldump -uisucon -p isubata > /tmp/mysql_dump_out
 scp /tmp/mysql_dump_out root@47.245.57.148:/tmp/
```
dbサーバで
```
mysql -uisucon -p isubata < /tmp/mysql_dump_out
```
データ入ったので繋ぎ直し。
なんか転送遅いなと思ったらGlobal回線使っていた！ 172.24.50.40 に接続すべし。 => 140Mb出てるやん・・・・
webサーバのenv書き換えて再読み込みさせる
```
$ cat ~/env.sh
ISUBATA_DB_HOST=172.24.50.40
ISUBATA_DB_USER=isucon
ISUBATA_DB_PASSWORD=isucon
```
dbサーバ側で、同じようにmysqld.cnf設定を変えて、
```
innodb_flush_log_at_trx_commit=2
slow_query_log                = 1
slow_query_log_file           = /var/log/mysql/mysqld-slow.log
long_query_time               = 1
log-queries-not-using-indexes = 1
innodb_flush_method = O_DIRECT
```
bindのところをコメントアウトしておく。restartも忘れずに

webサーバでmysqld潰しておく
```
isucon@iZ6we7qggdtabs3ysvx7mkZ:~/isubata/webapp/php$ sudo systemctl stop mysql
isucon@iZ6we7qggdtabs3ysvx7mkZ:~/isubata/webapp/php$ sudo systemctl disabled  mysql
Unknown operation disabled.
isucon@iZ6we7qggdtabs3ysvx7mkZ:~/isubata/webapp/php$ sudo systemctl disable  mysql
```
良さそう。

ここでbenchとってみるとCPUとメモリがちょっと空いているのでphpプロセスもうちょっと増やせそう
%Cpu(s): 80.3 us, 14.3 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  5.4 si,  0.0 st
KiB Mem :  1009028 total,   376164 free,   402048 used,   230816 buff/cache

 "score": 62462,


ネットワーク調査のコマンドもここらでみておく
```
 sudo apt install vnstat
```
```
 vnstat -l -i eth0
```
-l はライブのl。以下の値は scpでmysqldump_out を送り込んでいる時のもの。帯域が5Mb以下の指定なのでそれよりは出ているっぽい。
```
isucon@iZ6we7qggdtabs3ysvx7mkZ:~/isubata/bench$ vnstat  -l
Monitoring eth0...    (press CTRL-C to stop)

   rx:      173 kbit/s   302 p/s          tx:     6.98 Mbit/s   578 p/s
```

php fpm周りのチューニングをもうちょっと。
unix sock で行けるようにするため、mkdir /var/run/php/ ; sudo chmod 777 /var/run/php/
```
listen = "/var/run/php/php-fpm.sock"
```
nginxの設定を、どっちが適用されているかまだわかっていないがとりあえず書き換え（あとでもうちょっと調べる）
```
        index index.php;
        location / {
               if (!-f $request_filename) {
                       rewrite ^(.+)$ /index.php$1 last;
               }
                proxy_set_header Host $http_host;
                proxy_pass http://unix:/var/run/php/php-fpm.sock;
        }

         location ~ [^/]\.php(/|$) {
                 root           /home/isucon/isubata/webapp/php;
                 include        fastcgi_params;
                 fastcgi_index  index.php;
                 fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
                 fastcgi_param  SCRIPT_NAME     $fastcgi_script_name;
                 fastcgi_pass   unix:/var/run/php/php-fpm.sock;
        }
```
特にスコア変わらず。メモリ使用料などは？  1.7~1.9% くらい。あんまり変わってないな。unix socketで楽になるという説はよくわからん。調べよう。
nginx自体はほとんどCPU, mem共に使っていない 3パーセント程度なので、これをMySQLと相乗りさせて2台Webサーバにする案良さそう。

jsコードを読むと、ページ読み込み時に /message を読み込んでいる。そのあと、fetchをして、current channel に未読があったらmessageをまた叩く。
なので unread がたまるのがどういう状況かというのが重要。

とりあえずmessageエンドポイントの理解と合わせて がN+1になっているのを直す（ボトルネックではないのでスコアにほぼ影響なしのはず。学習のための感じ）
 "score": 66352,

history が重いので直す。動線はなさそうなのだが、負荷を上げる判定に使っているっぽい。
この二つのN+1を直したら劇的にMySQLの負荷がへり（CPU 0.3%, MEM25%）、これDBの構成分ける必要なかったのでは状態になった。
core は69000くらい
またもう一回撮ってみたら  "score": 84034 。


fetch と message のハックはやめて（解説読んだがfetch遅くするといいというはよくわからん）
リソース使い切っていないのでnginx fpm のチューニングもうちょっとやってみる
pm.max_requests = 10000 をfpm設定に入れる

max children を 100まで増やしたが、意外と used が少ないように見える。CachedにはNginxのファイルくらいしかないのでもうちょっと減らしても良さそうか？
KiB Mem :  1009028 total,    86336 free,   634444 used,   288248 buff/cache
```
$ du -h ../public/icons/
25M	../public/icons/
```
かなり少ないので大丈夫。
150まで増やして以下の感じ。
KiB Mem :  1009028 total,    77296 free,   769048 used,   162684 buff/cache

200まで増やして
KiB Mem :  1009028 total,    65076 free,   886248 used,    57704 buff/cache
流石にvim開くのも重くなってきた。

  "score": 31195, スコア落ちたので攻めすぎた。CPUの方がサチっていたか？ 見逃した。
100子だと
%Cpu(s): 82.3 us, 12.8 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  4.9 si,  0.0 st
KiB Mem :  1009028 total,   236780 free,   586904 used,   185344 buff/cache
こんなもんかなあ・・・

slow pathと言われてしまっている history と message をもうちょいなんとかしてみるか count が遅いとかいう解説を見た。


ここまでで考えたisucon戦略としてこうしたらいいのでは？ というやつ
* まずアプリケーションをみんなで使ってみて理解を深める
* レギュレーションを読み込んで、変えていいところ変えちゃダメなところ、特にポイント計算について読み込む
* benchを回していかにも軽いはずなのにタイムアウトするものについてはコードとDB設計みて潰す
* 基礎的なOSとミドルウェアチューニング
* slow query log(indexないものも出す), アクセスログ
* ボトルネックとなるリソースを緩和させるためにインフラ構成を見直す
* vnstat, sar, top, iostat などを使って、モニタリングしてボトルネックをあぶり出す
* ポイント計算のロジックから、どのエンドポイントが早くなればいいのかあぶり出す。また、そのエンドポイントが呼ばれる条件をjsやbenchから調査する。
