# Recon Village CTF 2022 @DEFCON30

簡潔に書いていますが、いろいろと試行錯誤した結果の解答への道筋です。

チームで解いているので、経過はチームメンバーで情報共有しています。

## Challenge 3


>On 11 August, what was the primary email domain from which spam was reported as coming from 185.129.62.62 and 107.189.28.253 (two bot IPs)?

spamなので、いくつかあるspam情報公開サイトから
stop forum spam で探して
https://www.stopforumspam.com/country/dk

該当IPアドレスより

11-Aug-22 17:14	185.129.62.62	vy4	 gt69@hiroyuki4010.yoshito33.inwebmail.fun

```
flag:{hiroyuki4010.yoshito33.inwebmail.fun}
```

## Challenge 5

>You're investigating a missing person who went missing following a party in 2019. While working through case notes you've come across the following NTIyMyBTb3V0aCBCcmFlc3dvb2QgQm91bGV2YXJkLCBIb3VzdG9u What date was the pool party?

謎の文字列がBase64だった

```bash
$ echo NTIyMyBTb3V0aCBCcmFlc3dvb2QgQm91bGV2YXJkLCBIb3VzdG9u|base64 -d 
5223 South Braeswood Boulevard, Houston
```


"5223 South Braeswood Boulevard" でGoogle検索して。一番最後にPastebinに招待メールを発見

https://pastebin.com/BNwRBAX4

```
Thank you for your RSVP to our mf pool party THIS Saturday. As a reminder
```

とあるので、

```
Date: Thu, 18 Apr 2019 07:57:24 -0500
```

の２日後で

```
flag:{20/04/2019}
```


## Challenge 11

>Hi I'm Eva Hesington. Remember me from last year. I am the founder of Cryptorama. Thanks for your support we have been able to scale the business a lot. I cannot thank the open source community enough. Using Open source tools and platforms, our business has grown and our tech department is now running strong. We could not have done it without these Open Source tools and community. You can visit out website to find out more.

CryptoRamaのWebページを https://cryptorama.cloud/ に発見。
https://cryptorama.cloud/dist/js/main.min.js
に
```
        // make sure this code is removed in the production release.
        // Token for prod server: d0_N0t_cOd3_&_c0mM3nt

```

とあったので、
```
flag:{d0_N0t_cOd3_&_c0mM3nt}
```


## Challenge 12

>We at CryptoRama are very concerned about sharing sensitive information to the outside world and even on the inside. We use secret sharing services. But looks like someone has been manipulating our secrets application and hindering our progress. Can you please check.

CryptoRama を githubで検索すると、

https://github.com/CryptoRamaLtd

が出てきます。

https://github.com/CryptoRamaLtd/snappass が個々のsecret をやり取りするためのツールのようです。


```
Hosted at: https://secrets.cryptorama.cloud
```

とあるので、ここにアクセスします。メッセージを入れて、URLを入れると暗号化メッセージにアクセスできるURLを生成するようです。
１回開くとメッセージは消えてしまいます。

ドキュメントに怪しい記述が、あります。これは乱数の種っぽいですね。

``SECRET_KEY``: unique key that's used to sign key. This should
be kept secret.  See the `Flask Documentation`__ for more information.


https://github.com/pinterest/snappass/commit/aeeedf3b77aaa568a695a87d51eda97821ef39af

コミットログをよく読んで脆弱性を作っているところを探します。
以下の記述がありました。

```
- SECRET_KEY=6ardCD6XQ49FLrxY6fd7pB3DeeNmzn8Y
```


main.pyから必要な部分を取り出してコードを書きます。

```python
import os
import sys
import uuid
import base64
from cryptography.fernet import Fernet
from flask import abort, Flask, render_template, request, jsonify


key = os.environ.get('SECRET_KEY', 'Secret Key')
encryption_key = base64.b64encode(key.encode("utf-8"))

print (encryption_key)
print (uuid.uuid4().hex)
```

```bash
$ export SECRET_KEY=6ardCD6XQ49FLrxY6fd7pB3DeeNmzn8Y
$ python3 Challenge12.py
b'NmFyZENENlhRNDlGTHJ4WTZmZDdwQjNEZWVObXpuOFk='
ddebc64cca8a498d865f49fc7e3b2a9b
$ python3 Challenge12.py
b'NmFyZENENlhRNDlGTHJ4WTZmZDdwQjNEZWVObXpuOFk='
675156cd4583467ea148de4f4d09edc9
$ python3 Challenge12.py
b'NmFyZENENlhRNDlGTHJ4WTZmZDdwQjNEZWVObXpuOFk='
31e2de70f8eb46acb5c0a6c1786aef4a

```

NmFyZENENlhRNDlGTHJ4WTZmZDdwQjNEZWVObXpuOFk=
が固定鍵です。
uuidは正常に生成されているので予測できません。


https://secrets.cryptorama.cloud/RV5e5fa1b0219a49fc8692c772aab1ba62%7C20220815040742~wuBIAkY29ZHDkOYQijAVeinbpw49HBVD7uxDfNFHsn4%3D

ソースコードを読んでこのURLの構造を調べました。

https://secrets.cryptorama.cloud/RV+uuid.uuid4().hex+"%7C"+datetime+'~'+base64(secret_key)

になっています。

必要な値は。
* uuid
* datetime
* secret_key

です。
secret_keyはスクリプトで固定値が得られました。
datetimeは予測するとして、uuidは正常に作られているため予測ができません。


CryptoRamaの公開レポジトリで、変なレポジトリがあります。

https://github.com/CryptoRamaLtd/Progress-Tracker

ファイルは１つで１分ごとにコミットしてます。
README.md
Check Status Commit 6fd79f3f896344d5a99c250af786006d

桁数的にUUIDです。

あとは、正解のURLを組み立てるだけです。


https://secrets.cryptorama.cloud/RV6fd79f3f896344d5a99c250af786006d%7C20220815054328~NmFyZENENlhRNDlGTHJ4WTZmZDdwQjNEZWVObXpuOFk%3D

発見できないので、秒は00にしました。

https://secrets.cryptorama.cloud/RV6fd79f3f896344d5a99c250af786006d%7C20220815054300~NmFyZENENlhRNDlGTHJ4WTZmZDdwQjNEZWVObXpuOFk%3D

```
flag:{$h@r3_pa$$w0Rd5_w1tH_c@ut10n}
```
