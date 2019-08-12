初回bench score 4100 くらい。

register の高速化をしたいのでパスワードのhash化ロジックを変更. random salt発行をなしに。

image 周りがフルスキャンになっていたので、IDを参照するようにするのが理想っぽいけど、とりあえず name に index貼って違いがどうなるかみてみる。
mysql> create index idx_name on image (name(10));

ここで  "score": 10432 おお。単純に index追加だけでも早くなっている・・・

top での負荷状況は、mysqld がCPU使いまくっている。
  622 mysql     20   0 1617444 337340  15312 S  82.4 16.5  11:30.28 mysqld
 1907 isucon    20   0  351024  22316  12720 S  15.3  1.1   0:22.65 php-fpm

やりたい
* icon が blobdata として入っているので吸い出してnginxで配信する
* 画像のnginxキャッシュで配信。cache control 周り
* /login が結構タイムアウトしている。処理的には重くなさそうなのでDBロックか何かで待たされているのかもしれない。それか単に過負荷
* font js css の配信は？ nginxの設定みると{}だけが書かれているが
* 画像サイズがメッセージ一覧では横100が固定か。不要に大きなものが配信されて帯域食っているかも

iconの吸い出し
```
isucon@iZ6we7qggdtabs3ysvx7mkZ:~/isubata/webapp/php$ for name in `sudo mysql -uroot isubata -e "select name from image"` ; do  sudo mysql -uroot isubata -e "select data from image where name=\"$name\" limit 1" > static/$name  ; done
```


```
isucon@iZ6we7qggdtabs3ysvx7mkZ:~/isubata/webapp/php$ diff  /etc/nginx/sites-available/nginx.php.conf{.org,}
14a15,18
> 	location /icons/ {
> 	   root /home/isucon/isubata/webapp/php/;
>         }
>
```

```
isucon@iZ6we7qggdtabs3ysvx7mkZ:~/isubata/webapp/php$ curl -I http://47.245.35.176/icons/8cbe874a2eac5b9e665add295160d36c56be5df5.png
HTTP/1.1 200 OK
Server: nginx/1.14.0 (Ubuntu)
Date: Mon, 12 Aug 2019 12:41:05 GMT
Content-Type: image/png
Content-Length: 212712
Last-Modified: Mon, 12 Aug 2019 12:24:07 GMT
Connection: keep-alive
ETag: "5d515a67-33ee8"
Accept-Ranges: bytes
```
と帰ってきているが、imagemagick入れてidentityコマンドで確認してみると invalid format となる。1行目に data と入っているのが問題かと思って消してみたがそれでもダメ。 一回設定戻してcurl で面から叩いて取得したのを並べるか。
