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
```
benchに影響無し。多分/profileで更新されるからキャッシュが聞いていない
