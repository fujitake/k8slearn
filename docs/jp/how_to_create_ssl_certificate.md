# テスト用SSL/TLS証明書作成方法

## 条件

本手順で作業を行うための条件は以下の通り。

- ドメイン名を所有していること
- 上記ドメインのDNSサーバの設定変更ができること
- 証明書を取得するFQDNを一時的に作業を行う端末に設定できること

## Let's Encryptを使ったSSL/TLS証明書の作成方法

aws等でLinuxを起動

上記サーバが`証明書発行対象のFQDNとしてアクセスできること`かつ`tcp 80でアクセス可能なこと`を満たす

以下のコマンドにて`certbot`をインストール

```sh:
snap install --classic certbot
```

以下のコマンドを実行し、証明書を作成

```sh:
sudo certbot certonly -d "your.domain.name","your.domain.name" --server https://acme-v02.api.letsencrypt.org/directory --register-unsafely-without-email
```

次のようなメッセージが表示されるので`1`を入力し`enter`を押下

```sh:
Saving debug log to /var/log/letsencrypt/letsencrypt.log

How would you like to authenticate with the ACME CA?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: Spin up a temporary webserver (standalone)
2: Place files in webroot directory (webroot)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel): 1
```

Terms of Serviceを読んで同意する、`y`を入力して`enter`を押下

```sh:
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
agree in order to register with the ACME server. Do you agree?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: y
```

適切に処理が行われれば、下記のように成功する。

```sh:
Account registered.
Requesting a certificate for your.domain.name

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/your.domain.name/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/your.domain.name/privkey.pem
This certificate expires on 2022-02-11.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

## 参考情報

Vantiq Private Cloudで利用する際は、`fullchain.pem`と`privkey.pem`を使用する
