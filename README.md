初回bench score 4100 くらい。

register の高速化をしたいのでパスワードのhash化ロジックを変更. random salt発行をなしに。

image 周りがフルスキャンになっていたので、IDを参照するようにするのが理想っぽいけど、とりあえず name に index貼って違いがどうなるかみてみる。
mysql> create index idx_name on image (name(10));

ここで  "score": 10432 おお。単純に index追加だけでも早くなっている・・・
