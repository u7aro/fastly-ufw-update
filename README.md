# fastly-allow-ufw

https://api.fastly.com/public-ip-list
で公開されている Fastly のIPアドレスリストを定期的に取得し、
それらのIPアドレスから本サーバへ sftp 接続できるよう、
ufw のルールを更新する。

## 動作確認

ufw を有効にする。
```
sudo ufw enable
```

本スクリプトをダウンロードして、ufw にダミー用のルールを設定する。

```
git clone https://github.com/emon/fastly-ufw-update.git
cd fastly-ufw-update/
./test/ufw-test-setup.sh	# ufw にテスト用のダミーのルールを設定する (sudo 権限が必要)
```
ダミーのルールとして、
- fastly の本物のIPアドレスの一部
- fastly とは関係のないIPアドレス
を登録している。

`ufw-update.sh` は引数にサブコマンド名を取る。
```
cd script
./ufw-update.sh
Usage: ./ufw-update.sh [command]
 show        - show local and fastly's latest rules
 show local  - show local ufw rules
 show remote - show fastly's latest rules
 diff        - diff local and remote
 apply       - apply latest rules
```

### ufw-update.sh show
ローカル(ufw)の設定と、fastly の公開している最新情報を、それぞれ表示する。
下記でローカル設定として表示されているのは、上記の `ufw-test-setup.sh` で登録したダミーのルール。

```
./ufw-update.sh show
# local ufw rules
103.244.50.0/24
103.245.222.0/23
103.245.224.0/24
104.156.80.0/20
130.54.0.0/16
133.3.0.0/16
146.75.0.0/16
151.101.0.0/16
157.52.64.0/18
167.82.0.0/17
23.235.32.0/20
43.249.72.0/22
8.8.8.8
# fastly's latest rules
23.235.32.0/20
43.249.72.0/22
103.244.50.0/24
103.245.222.0/23
103.245.224.0/24
104.156.80.0/20
146.75.0.0/16
151.101.0.0/16
157.52.64.0/18
167.82.0.0/17
167.82.128.0/20
167.82.160.0/20
167.82.224.0/20
172.111.64.0/18
185.31.16.0/22
199.27.72.0/21
199.232.0.0/16
```

### ufw-update.sh diff
ローカル設定と、fastly の公開している最新情報との差分を表示する。
```
./ufw-update.sh diff
@@ -5,2 +4,0 @@
-130.54.0.0/16
-133.3.0.0/16
@@ -10,0 +9,7 @@
+167.82.128.0/20
+167.82.160.0/20
+167.82.224.0/20
+172.111.64.0/18
+185.31.16.0/22
+199.232.0.0/16
+199.27.72.0/21
@@ -13 +17,0 @@
-8.8.8.8
```

### ufw-update.sh apply

diff で表示されている差分を適用する。
環境変数 `DEBUG_LEVEL` に `1` を設定すると `ufw` での登録内容を表示する。

```
DEBUG_LEVEL=1 ./ufw-update.sh apply
sudo ufw delete allow in from 130.54.0.0/16 to any port ssh
Rule deleted
sudo ufw delete allow in from 133.3.0.0/16 to any port ssh
Rule deleted
sudo ufw allow in from 167.82.128.0/20 to any port ssh comment from https://api.fastly.com/public-ip-list
Rule added
sudo ufw allow in from 167.82.160.0/20 to any port ssh comment from https://api.fastly.com/public-ip-list
Rule added
sudo ufw allow in from 167.82.224.0/20 to any port ssh comment from https://api.fastly.com/public-ip-list
Rule added
sudo ufw allow in from 172.111.64.0/18 to any port ssh comment from https://api.fastly.com/public-ip-list
Rule added
sudo ufw allow in from 185.31.16.0/22 to any port ssh comment from https://api.fastly.com/public-ip-list
Rule added
sudo ufw allow in from 199.232.0.0/16 to any port ssh comment from https://api.fastly.com/public-ip-list
Rule added
sudo ufw allow in from 199.27.72.0/21 to any port ssh comment from https://api.fastly.com/public-ip-list
Rule added
sudo ufw delete allow in from 8.8.8.8 to any port ssh
Rule deleted
```

再度実行すると差分が無いため、何も表示されない。
```
DEBUG_LEVEL=1 ./ufw-update.sh apply
```

## cron での実行

`sudo` を NOPASS で利用できる必要がある。

`ufw-update.sh` と同一ディレクトリ内の `jq.py` が実行できる必要がある。`jq.py` は python3.8 で動作を確認した。jquery をパースするためだけに利用している。

## rsyslog の設定

```
sudo cp fastly-allow-ufw/rsyslog.d/30-fastly-allow-ufw.conf /etc/rsyslog.d/
sudo systemctl restart rsyslog.service
```

下記のようなログが残る。

```log:/var/log/fastly-ufw-update.log
[debug] ./ufw-update.sh show
[debug] finished

[debug] ./ufw-update.sh diff
[debug] finished

[debug] ./ufw-update.sh apply
[info] sudo ufw delete allow in from 130.54.0.0/16 to any port ssh
[info] sudo ufw delete allow in from 133.3.0.0/16 to any port ssh
[info] sudo ufw allow in from 167.82.128.0/20 to any port ssh comment from https://api.fastly.com/public-ip-list
[info] sudo ufw allow in from 167.82.160.0/20 to any port ssh comment from https://api.fastly.com/public-ip-list
[info] sudo ufw allow in from 167.82.224.0/20 to any port ssh comment from https://api.fastly.com/public-ip-list
[info] sudo ufw allow in from 172.111.64.0/18 to any port ssh comment from https://api.fastly.com/public-ip-list
[info] sudo ufw allow in from 185.31.16.0/22 to any port ssh comment from https://api.fastly.com/public-ip-list
[info] sudo ufw allow in from 199.232.0.0/16 to any port ssh comment from https://api.fastly.com/public-ip-list
[info] sudo ufw allow in from 199.27.72.0/21 to any port ssh comment from https://api.fastly.com/public-ip-list
[info] sudo ufw delete allow in from 8.8.8.8 to any port ssh
[debug] finished

[debug] ./ufw-update.sh apply
[debug] finished
```
