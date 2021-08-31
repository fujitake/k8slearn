# テスト用SSL/TLS証明書作成方法

## Let's Encryptを使ったSSL/TLS証明書の作成方法

自分でドメイン名を有することが、以下のステップを実施する条件となる。
また、以下のサーバは、実際に構築しようしているドメイン名と同じに指定してアクセスを確認する必要がある


1) aws等でLinuxを起動
2) 上記サーバが、以下の2つの条件を満たすように設定
　・対象のdomain nameとしてアクセスできること
　・tcp 80/443でアクセスできること(おそらく実際は80のみ)
2) git clone https://github.com/certbot/certbot を実行
3) cd certbot
4) ./certbot-auto --help で動作することを確認
5) $ ./certbot-auto certonly --standalone -t
　不足しているパッケージ等の自動インストールが行われる
　その後、 email addressを聞かれる
　次にTerms of Serviceに同意することを確認させる
　次に、上記email address宛にマーケティングメッセージの送信を希望するか確認される
　最後にドメイン名を入力
6) Congratulations! と表記されたらOK、そうではない場合、エラー内容を確認し、再度実行
7) VANTIQで利用する際は、fullchain.pemとprivkey.pemを secrets.yaml で指定 (cert.pemを使うとブラウザは動作するが、VANTIQ間での利用ができない)

参考情報
https://qiita.com/HeRo/items/f9eb8d8a08d4d5b63ee9

#### Notes
同じドメイン名で更新すると1回目と違い、fullchain1.pemという感じで、末尾に1という数字が付与される。おそらく1回目の更新という意味かと。
ちなみにrenewで作成ではなく、新しく作ったec2インスタンスにてcertbot-auto certonly --standalone -tで作成した



## 自己証明書(オレオレ証明書)を使った場合の注意点

自己証明書(オレオレ証明書)を使った場合、postman、curlなどアクセスできないケースが大半である
#### VANTIQからオレオレ証明書のVANTIQ環境にRemote source:
ssl handshake error

####postman
ssl handshake後にエラーが出力される(実際は存在しているVANTIQ Topic)
```
404 Not Found
{
    "error": "Resource not found: /api/v1/resources/topics//MyTopic"
}
```

#### curl
下記のようなエラーが出力される
```
curl: (60) SSL certificate problem: self signed certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

####Python
Verifyする場合：`requests.exceptions.SSLError: HTTPSConnectionPool(host='tkeksfuji7.longnameofdomaintest.tk', port=443): Max retries exceeded with url: /api/v1/resources/topics//start (Caused by SSLError(SSLCertVerificationError(1, '[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: self signed certificate (_ssl.c:1056)')))`

Verifyしない場合：
`/Library/Frameworks/Python.framework/Versions/3.7/lib/python3.7/site-packages/urllib3/connectionpool.py:851: InsecureRequestWarning: Unverified HTTPS request is being made. Adding certificate verification is strongly advised. See: https://urllib3.readthedocs.io/en/latest/advanced-usage.html#ssl-warnings InsecureRequestWarning)`
