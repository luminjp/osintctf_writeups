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


## Chalenge 14 by solved @ykame

>Our junior developer Henry Lopez cannot find his assets. Can you help him please. His credentials are given below: Username: henry.lopez Password: xxxxxxxxxxxxxx

パスワードはマスクしました。

``` bash
$ dig cryptorama.cloud +short
54.176.12.255
$ openssl s_client cryptorama.cloud:443
CONNECTED(00000003)
Can't use SSL_get_servername
depth=2 C = US, O = Internet Security Research Group, CN = ISRG Root X1
verify return:1
depth=1 C = US, O = Let's Encrypt, CN = R3
verify return:1
depth=0 CN = assets.cryptorama.cloud
verify return:1
---
Certificate chain
 0 s:CN = assets.cryptorama.cloud
   i:C = US, O = Let's Encrypt, CN = R3
 1 s:C = US, O = Let's Encrypt, CN = R3
   i:C = US, O = Internet Security Research Group, CN = ISRG Root X1
 2 s:C = US, O = Internet Security Research Group, CN = ISRG Root X1
   i:O = Digital Signature Trust Co., CN = DST Root CA X3
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIFNTCCBB2gAwIBAgISA+nxepGKTXeOQCkAMr57EyecMA0GCSqGSIb3DQEBCwUA
MDIxCzAJBgNVBAYTAlVTMRYwFAYDVQQKEw1MZXQncyBFbmNyeXB0MQswCQYDVQQD
EwJSMzAeFw0yMjA4MTEwNjIwMjFaFw0yMjExMDkwNjIwMjBaMCIxIDAeBgNVBAMT
F2Fzc2V0cy5jcnlwdG9yYW1hLmNsb3VkMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A
MIIBCgKCAQEAqmaxsBoKzFbJ5WWGFQlXDtvzfvShJcYXySbSNFJhaR+/nDblR8t9
cBsjtU2duPTUykam7CfSVBHUN3jKxzU/DcrtKULv2PfEc/EX47dkWHIPGUE5J2Ig
2Cn7vvFjySkXH2VYypdTvqbXpwjBMQ+NEao++7VInSRHSHFJ7MQnOHITtPRKN5sP
3XNRFYrr84dlZUup+cT4f6rY6cSpGI6yvBNdgfkCBN8H6+w4rdfe0QT2pg+5nKbR
+gfis0rwEo923VTwNsb8vwcrHkWWKk67nzSkRukStHpvknaRI+vOYMPhsrx90MBS
sz3obHbAULuXq5043Retjzbey1S/QVURswIDAQABo4ICUzCCAk8wDgYDVR0PAQH/
BAQDAgWgMB0GA1UdJQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAMBgNVHRMBAf8E
AjAAMB0GA1UdDgQWBBTOVp+Coi4KDESQ/KffiW37AgbxHzAfBgNVHSMEGDAWgBQU
LrMXt1hWy65QCUDmH6+dixTCxjBVBggrBgEFBQcBAQRJMEcwIQYIKwYBBQUHMAGG
FWh0dHA6Ly9yMy5vLmxlbmNyLm9yZzAiBggrBgEFBQcwAoYWaHR0cDovL3IzLmku
bGVuY3Iub3JnLzAiBgNVHREEGzAZghdhc3NldHMuY3J5cHRvcmFtYS5jbG91ZDBM
BgNVHSAERTBDMAgGBmeBDAECATA3BgsrBgEEAYLfEwEBATAoMCYGCCsGAQUFBwIB
FhpodHRwOi8vY3BzLmxldHNlbmNyeXB0Lm9yZzCCAQUGCisGAQQB1nkCBAIEgfYE
gfMA8QB3AN+lXqtogk8fbK3uuF9OPlrqzaISpGpejjsSwCBEXCpzAAABgovGjxAA
AAQDAEgwRgIhAOLylImohl5hjqF7GMSVnwPRfbPA1PZDrfZ03TyA5admAiEAkCug
ym6fhcOz5wb3k72OCb9yqbf8P/L9G2/hQF5S8NUAdgBGpVXrdfqRIDC1oolp9PN9
ESxBdL79SbiFq/L8cP5tRwAAAYKLxo9OAAAEAwBHMEUCIQC2xVvMxKtaPzD6xZO+
vFjZxMt4RsUgeNM6EqZr8CwHzQIgeqMcasr+qMkkEOFWefLHfloVHuYBiG3/WWUu
nhe/F6MwDQYJKoZIhvcNAQELBQADggEBACuBsfMQTR6yHrJEilCEjEEwgOJ0lzkm
4fKsOoA7I8YQzSmHww5is5nGyR9Rp9tJq6/WRMJA+OiLQBwCSTVPG6lrmD0945/9
wFYuERlud2OtHKlfbBm8XlicZF3qXX/DHF7glAh36c6qXz/ScN64Q8e20sepW2YR
moqOAORZOtuJTWwrhJGzAxsyqqNpL3z7eeVS0kzNrlrrLYIx/gwrQU8VTl4xuk+3
5rA7EWKXC0JKL2q4UUxShMBurZIBgoix4H9x1jC9Z7pbG0ujsWER5TZOHePMd2cg
D/3lLLvlbcpdDKajiYchW91xxJNi9P90vb/BcR4PvnidZsxB3+tK2fM=
-----END CERTIFICATE-----
subject=CN = assets.cryptorama.cloud

issuer=C = US, O = Let's Encrypt, CN = R3

---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 4589 bytes and written 363 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 2048 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
---
Post-Handshake New Session Ticket arrived:
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID: FAA15A0C3ED8A27205C96F84AF9F56705EF276C065561C4560A85B092478AA66
    Session-ID-ctx:
    Resumption PSK: 6CCE80043AEA4E99B82CACAF5AE6DED7BE96F352473E1E296F4C3EF72A79507F89F704B11560CA964F55AA61749E8572
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 86400 (seconds)
    TLS session ticket:
    0000 - 19 81 bb be 92 70 1f 99-5e 06 18 2d a7 83 34 af   .....p..^..-..4.
    0010 - d2 44 64 f0 68 82 0c fb-15 5b 42 98 bf d3 d6 4f   .Dd.h....[B....O

    Start Time: 1660731455
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
    Max Early Data: 0
---
read R BLOCK
---
Post-Handshake New Session Ticket arrived:
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID: C335CEE7DE601A71548D7F3605E6AA0DA47450525D079E4475D188A613DDBDCB
    Session-ID-ctx:
    Resumption PSK: 43FE72347E37ED725022881B56FC11AFECD13A0662165481E85003B7DE0AD9184417EB4C2E8302F0AC3529792D86C56F
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 86400 (seconds)
    TLS session ticket:
    0000 - 89 54 6f 32 6d 5d 62 af-25 e5 e6 6b 17 cf 90 3b   .To2m]b.%..k...;
    0010 - 51 28 d1 c0 f1 62 cb 60-d1 1c b2 8f 70 c8 06 d2   Q(...b.`....p...

    Start Time: 1660731455
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
    Max Early Data: 0
---
read R BLOCK

```
openssl コマンドでcryptorama.cloiudに接続すると。

assets.cryptorama.cloud を発見しました。


このサイトは、Snip-ITの Version v5.4.3 - build 6812 (master) を使っています。
古いものを使っているので脆弱性を探します。

https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=snipe

このバージョンということは、5.4.4で修正されているのが怪しいです。

CVE-2022-1511は 5.4.4で修正されています。

https://nvd.nist.gov/vuln/detail/CVE-2022-1511

攻撃の詳細は以下のURLに詳しく乗っています。

https://huntr.dev/bounties/4a1723e9-5bc4-4c4b-bceb-1c45964cc71d/

Requested Assetsのアクセス制限不備の脆弱性ですので。脆弱なURLにアクセスしますとフラグがあります。

hxxps://assets.cryptorama.cloud/hardware/requested

```
Ariane Lee flag:{p@tcH_aLL_vULn3r@biL1ti3s}
```
