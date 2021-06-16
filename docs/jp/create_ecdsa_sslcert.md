## この記事について

この手順は、Let's Encryptを使って楕円曲線暗号化を使ったSSLサーバ証明書の作成方法となります。

## 前提条件

- Webサーバを別途用意していること
- 上記とは別にLet's EncryptのクライアントツールをインストールできるLinuxマシンを用意できること
- 上記Linuxマシンはインターネットアクセスができること(ssh、https)
- SSL証明書を発行するドメイン名を取得済みであること
- 発行したいSSL証明書のFQDNを管理するDNS Zoneを設定できること

## 事前準備

- AWS/Azureにてubuntu EC2 instanceを立てる
- Security groupでinbound 80と443を有効にする
- Route 53にて上記EC2 instanceのIPアドレスをAレコード登録する or

## Let's Encryptを使ったECDSA証明書の作成手順

- ECDSA対応のCSRを作成するため、private keyを作成する

```shell:コマンド
openssl ecparam -out private_prime256v1.key -name prime256v1 -genkey
```

```shell:コマンド
ls -l
drwxrwxr-x 30 ubuntu ubuntu 4096 Oct 15 15:25 certbot
-rw-------  1 ubuntu ubuntu  302 Oct 15 15:27 private_prime256v1.key
```

```shell:コマンド
cat private_prime256v1.key
-----BEGIN EC PARAMETERS-----
BggqhkjOPQMBBw==
-----END EC PARAMETERS-----
-----BEGIN EC PRIVATE KEY-----
MHcCAQEEIKyuRZRPF/fwlsp/ZF/m60SPyueqyvA1SlyZrnnv+VYdoAoGCCqGSM49
AwEHoUQDQgAEoddtY8moERotbWD88AiZFeb5GzgcoCefvU+hORbJlqibSFzD7/O9
g/6v1e28PTpZvEc2Fd0WoJGQburfNQJ+bA==
-----END EC PRIVATE KEY-----
```

EC Parametersってのが表示される  
RSAのprivate keyと比べると表示されるPrivate Keyの行数が少ない

- CSRを作成する
LetsencryptにてCSRを指定して証明書を作成する場合、通常使うBASE64 encodingではなく、derバイナリ形式のCSRが必要となる

```shell:コマンド
ubuntu@ip-10-1-0-79:~$ openssl req -new -key private_prime256v1.key -sha256 -nodes -outform der -out private_prime256v1.der
Can't load /home/ubuntu/.rnd into RNG
140489903141312:error:2406F079:random number generator:RAND_load_file:Cannot open file:../crypto/rand/randfile.c:88:Filename=/home/ubuntu/.rnd
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:JP
State or Province Name (full name) [Some-State]:Tokyo
Locality Name (eg, city) []:Shinagawa
Organization Name (eg, company) [Internet Widgits Pty Ltd]:fujitake
Organizational Unit Name (eg, section) []:CSG
Common Name (e.g. server FQDN or YOUR name) []:sieksfuji9.longnameofdomaintest.tk
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

```shell:コマンド
openssl req -in private_prime256v1.der -inform der -text
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: C = JP, ST = Tokyo, L = Shinagawa, O = fujitake, OU = CSG, CN = sieksfuji9.longnameofdomaintest.tk
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:a1:d7:6d:63:c9:a8:11:1a:2d:6d:60:fc:f0:08:
                    99:15:e6:f9:1b:38:1c:a0:27:9f:bd:4f:a1:39:16:
                    c9:96:a8:9b:48:5c:c3:ef:f3:bd:83:fe:af:d5:ed:
                    bc:3d:3a:59:bc:47:36:15:dd:16:a0:91:90:6e:ea:
                    df:35:02:7e:6c
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        Attributes:
            a0:00
    Signature Algorithm: ecdsa-with-SHA256
         30:45:02:20:28:fb:dc:8b:78:c8:41:a1:fb:81:a0:ec:68:2b:
         a1:91:9d:e6:54:9a:68:c9:ff:38:e5:20:49:52:17:c0:09:e9:
         02:21:00:9b:b7:b5:4f:5d:81:ba:aa:d8:48:1d:34:14:ac:3c:
         1d:3b:b5:79:a8:85:d0:c7:79:5b:e3:74:2e:da:56:fc:72
-----BEGIN CERTIFICATE REQUEST-----
MIIBOjCB4QIBADB/MQswCQYDVQQGEwJKUDEOMAwGA1UECAwFVG9reW8xEjAQBgNV
BAcMCVNoaW5hZ2F3YTERMA8GA1UECgwIZnVqaXRha2UxDDAKBgNVBAsMA0NTRzEr
MCkGA1UEAwwic2lla3NmdWppOS5sb25nbmFtZW9mZG9tYWludGVzdC50azBZMBMG
ByqGSM49AgEGCCqGSM49AwEHA0IABKHXbWPJqBEaLW1g/PAImRXm+Rs4HKAnn71P
oTkWyZaom0hcw+/zvYP+r9XtvD06WbxHNhXdFqCRkG7q3zUCfmygADAKBggqhkjO
PQQDAgNIADBFAiAo+9yLeMhBofuBoOxoK6GRneZUmmjJ/zjlIElSF8AJ6QIhAJu3
tU9dgbqq2EgdNBSsPB07tXmohdDHeVvjdC7aVvxy
-----END CERTIFICATE REQUEST-----
```


- Letsencryptのクライアントをインストール

```shell:コマンド
git clone https://github.com/certbot/certbot
```

- Letencryptの初期設定を行う

```shell:コマンド
cd certbot/
./certbot-auto help
Requesting to rerun ./certbot-auto with root privileges...
Bootstrapping dependencies for Debian-based OSes... (you can skip this with --no-bootstrap)
Get:1 http://ap-southeast-1.ec2.archive.ubuntu.com/ubuntu bionic InRelease [242 kB]
Get:2 http://ap-southeast-1.ec2.archive.ubuntu.com/ubuntu bionic-updates InRelease [88.7 kB]
Get:3 http://ap-southeast-1.ec2.archive.ubuntu.com/ubuntu bionic-backports InRelease [74.6 kB]
Get:4 http://ap-southeast-1.ec2.archive.ubuntu.com/ubuntu bionic-updates/main amd64 Packages [1713 kB]
Get:5 http://ap-southeast-1.ec2.archive.ubuntu.com/ubuntu bionic-updates/universe amd64 Packages [1678 kB]      
Get:6 http://ap-southeast-1.ec2.archive.ubuntu.com/ubuntu bionic-updates/multiverse amd64 Packages [31.6 kB]
Get:7 http://security.ubuntu.com/ubuntu bionic-security InRelease [88.7 kB]                           
Fetched 3917 kB in 1s (3330 kB/s)                                                     
## 途中の記載を省略したが、パッケージのインストール及び作業可否の質問などが表示されるので、読んで対応すること
Installation succeeded.
```


- 作成したCSRを使い証明書を作成する

この時、standaloneモードを使うので、このコマンドを実行するサーバがインターネットアクセス可能であり、かつ80/443でアクセスできる必要がある

```shell:コマンド
./certbot-auto certonly --standalone -t --csr ~/private_prime256v1.der
Requesting to rerun ./certbot-auto with root privileges...
./certbot-auto has insecure permissions!
To learn how to fix them, visit https://community.letsencrypt.org/t/certbot-auto-deployment-best-practices/91979/
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator standalone, Installer None
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): tfujitake@gmail.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
agree in order to register with the ACME server at
https://acme-v02.api.letsencrypt.org/directory
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(A)gree/(C)ancel: a

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing, once your first certificate is successfully issued, to
share your email address with the Electronic Frontier Foundation, a founding
partner of the Let's Encrypt project and the non-profit organization that
develops Certbot? We'd like to send you email about our work encrypting the web,
EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: y
Performing the following challenges:
http-01 challenge for sieksfuji9.longnameofdomaintest.tk
Waiting for verification...
Cleaning up challenges
Server issued certificate; certificate written to /home/ubuntu/certbot/0000_cert.pem
Cert chain written to <fdopen>
Cert chain written to <fdopen>
Subscribe to the EFF mailing list (email: tfujitake@gmail.com).

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /home/ubuntu/certbot/0001_chain.pem
   Your cert will expire on 2021-01-14. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot-auto
   again. To non-interactively renew *all* of your certificates, run
   "certbot-auto renew"
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```


- ファイルを確認する

Dir certbot配下で実行すると下記のようになってしまうので、注意が必要  
通常のcertbot-autoコマンドでは、 /etc/letsencrypt にDirが作成され証明書が置かれるが、CSRを指定するとcurrent dirに証明書が作られる

```shell:コマンド
ls -l
total 352
-rw-r--r-- 1 root   root    1688 Oct 16 18:18 0000_cert.pem
-rw-r--r-- 1 root   root    1647 Oct 16 18:18 0000_chain.pem
-rw-r--r-- 1 root   root    3335 Oct 16 18:18 0001_chain.pem
-rw-rw-r-- 1 ubuntu ubuntu 12737 Oct 15 15:25 AUTHORS.md
lrwxrwxrwx 1 ubuntu ubuntu    20 Oct 15 15:25 CHANGELOG.md -> certbot/CHANGELOG.md
-rw-rw-r-- 1 ubuntu ubuntu   103 Oct 15 15:25 CODE_OF_CONDUCT.md
-rw-rw-r-- 1 ubuntu ubuntu  1481 Oct 15 15:25 CONTRIBUTING.md
-rw-rw-r-- 1 ubuntu ubuntu   555 Oct 15 15:25 Dockerfile-dev
-rw-rw-r-- 1 ubuntu ubuntu   835 Oct 15 15:25 ISSUE_TEMPLATE.md
-rw-rw-r-- 1 ubuntu ubuntu 11456 Oct 15 15:25 LICENSE.txt
lrwxrwxrwx 1 ubuntu ubuntu    18 Oct 15 15:25 README.rst -> certbot/README.rst
drwxrwxr-x 6 ubuntu ubuntu  4096 Oct 15 15:25 acme
drwxrwxr-x 6 ubuntu ubuntu  4096 Oct 15 15:25 certbot
drwxrwxr-x 4 ubuntu ubuntu  4096 Oct 15 15:25 certbot-apache
-rwxrwxr-x 1 ubuntu ubuntu 78696 Oct 15 15:25 certbot-auto
drwxrwxr-x 5 ubuntu ubuntu  4096 Oct 15 15:25 certbot-ci
drwxrwxr-x 4 ubuntu ubuntu  4096 Oct 15 15:25 certbot-compatibility-test
drwxrwxr-x 6 ubuntu ubuntu  4096 Oct 15 15:25 certbot-dns-cloudflare
drwxrwxr-x 6 ubuntu ubuntu  4096 Oct 15 15:25 certbot-dns-cloudxns
```


- 作成した証明書の内容を確認する

```shell:コマンド
openssl x509 -in 0000_cert.pem -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            04:e3:d9:f9:3b:d8:8b:c2:01:e7:0a:70:68:be:90:21:19:39
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = US, O = Let's Encrypt, CN = Let's Encrypt Authority X3
        Validity
            Not Before: Oct 16 17:18:04 2020 GMT
            Not After : Jan 14 17:18:04 2021 GMT
        Subject: CN = sieksfuji9.longnameofdomaintest.tk
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:a1:d7:6d:63:c9:a8:11:1a:2d:6d:60:fc:f0:08:
                    99:15:e6:f9:1b:38:1c:a0:27:9f:bd:4f:a1:39:16:
                    c9:96:a8:9b:48:5c:c3:ef:f3:bd:83:fe:af:d5:ed:
                    bc:3d:3a:59:bc:47:36:15:dd:16:a0:91:90:6e:ea:
                    df:35:02:7e:6c
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature
            X509v3 Extended Key Usage:
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Key Identifier:
                62:16:63:24:33:C8:C8:89:FF:45:1C:A0:21:AA:10:5D:2E:00:17:AA
            X509v3 Authority Key Identifier:
                keyid:A8:4A:6A:63:04:7D:DD:BA:E6:D1:39:B7:A6:45:65:EF:F3:A8:EC:A1

            Authority Information Access:
                OCSP - URI:http://ocsp.int-x3.letsencrypt.org
                CA Issuers - URI:http://cert.int-x3.letsencrypt.org/

            X509v3 Subject Alternative Name:
                DNS:sieksfuji9.longnameofdomaintest.tk
            X509v3 Certificate Policies:
                Policy: 2.23.140.1.2.1
                Policy: 1.3.6.1.4.1.44947.1.1.1
                  CPS: http://cps.letsencrypt.org

            CT Precertificate SCTs:
                Signed Certificate Timestamp:
                    Version   : v1 (0x0)
                    Log ID    : 44:94:65:2E:B0:EE:CE:AF:C4:40:07:D8:A8:FE:28:C0:
                                DA:E6:82:BE:D8:CB:31:B5:3F:D3:33:96:B5:B6:81:A8
                    Timestamp : Oct 16 18:18:04.872 2020 GMT
                    Extensions: none
                    Signature : ecdsa-with-SHA256
                                30:44:02:20:54:B7:FA:28:4F:25:58:3D:3D:85:DC:EB:
                                12:09:0F:2A:9D:7F:CD:32:BC:A5:22:28:4C:F0:8B:BC:
                                55:A3:95:C7:02:20:3B:34:4F:CC:66:68:C6:A1:8C:22:
                                FD:12:4D:04:0C:67:3B:46:90:7A:D0:51:6D:16:B0:40:
                                C2:D4:FA:A3:7C:CB
                Signed Certificate Timestamp:
                    Version   : v1 (0x0)
                    Log ID    : 7D:3E:F2:F8:8F:FF:88:55:68:24:C2:C0:CA:9E:52:89:
                                79:2B:C5:0E:78:09:7F:2E:6A:97:68:99:7E:22:F0:D7
                    Timestamp : Oct 16 18:18:04.908 2020 GMT
                    Extensions: none
                    Signature : ecdsa-with-SHA256
                                30:45:02:21:00:BA:4A:5E:8A:06:5D:64:27:C9:94:71:
                                AA:A3:DE:F8:D5:B4:11:9A:67:09:46:BB:BB:0F:67:CE:
                                CF:2E:4C:B4:14:02:20:1C:30:BC:90:2A:38:56:6C:80:
                                F6:12:E0:B2:61:07:A4:8A:38:6C:2F:98:95:D7:57:0B:
                                66:2F:D1:E0:6D:8C:F0
    Signature Algorithm: sha256WithRSAEncryption
         47:d3:ee:3d:ec:8f:92:65:bc:ce:36:f7:b8:d9:eb:4b:0a:30:
         ca:74:83:4f:eb:bb:de:f1:5f:a3:6f:78:bd:5d:a6:88:dd:1e:
         7b:6a:4b:76:97:d9:c2:43:97:73:08:57:78:54:1d:d5:16:f5:
         d4:cb:88:32:0a:7f:54:2c:d9:73:4a:39:27:01:de:7f:a0:2c:
         51:46:85:94:b1:71:66:7f:04:95:59:27:57:3a:29:d9:b9:d9:
         22:f4:3e:eb:3c:cf:6d:b4:2b:9d:e9:c8:a8:5e:63:ea:d5:a7:
         cd:d7:80:5e:66:8f:e9:bc:f4:45:97:ff:f2:aa:c4:20:56:1c:
         b1:78:4f:ff:62:52:16:f7:71:67:ea:15:91:52:82:67:6c:f9:
         e8:64:0a:8a:ac:e9:33:67:da:ae:0f:e3:bf:ef:d8:b6:43:7a:
         e8:6e:92:93:0b:48:37:e7:0a:78:bf:60:6d:44:5b:90:84:ac:
         48:3d:08:88:f6:11:74:26:6f:3a:7e:51:1a:be:3f:9d:9a:91:
         b5:25:f3:65:4e:6e:9f:36:b3:53:a2:7a:d3:f1:b9:41:a2:a3:
         3c:8c:84:b2:78:02:57:a3:ab:04:0a:ab:0a:77:39:20:99:ea:
         9b:ee:b0:60:23:35:30:aa:4e:d2:36:65:d2:8f:30:c5:0c:df:
         98:4a:df:0d
-----BEGIN CERTIFICATE-----
MIIEsDCCA5igAwIBAgISBOPZ+TvYi8IB5wpwaL6QIRk5MA0GCSqGSIb3DQEBCwUA
MEoxCzAJBgNVBAYTAlVTMRYwFAYDVQQKEw1MZXQncyBFbmNyeXB0MSMwIQYDVQQD
ExpMZXQncyBFbmNyeXB0IEF1dGhvcml0eSBYMzAeFw0yMDEwMTYxNzE4MDRaFw0y
MTAxMTQxNzE4MDRaMC0xKzApBgNVBAMTInNpZWtzZnVqaTkubG9uZ25hbWVvZmRv
bWFpbnRlc3QudGswWTATBgcqhkjOPQIBBggqhkjOPQMBBwNCAASh121jyagRGi1t
YPzwCJkV5vkbOBygJ5+9T6E5FsmWqJtIXMPv872D/q/V7bw9Olm8RzYV3RagkZBu
6t81An5so4ICdjCCAnIwDgYDVR0PAQH/BAQDAgeAMB0GA1UdJQQWMBQGCCsGAQUF
BwMBBggrBgEFBQcDAjAMBgNVHRMBAf8EAjAAMB0GA1UdDgQWBBRiFmMkM8jIif9F
HKAhqhBdLgAXqjAfBgNVHSMEGDAWgBSoSmpjBH3duubRObemRWXv86jsoTBvBggr
BgEFBQcBAQRjMGEwLgYIKwYBBQUHMAGGImh0dHA6Ly9vY3NwLmludC14My5sZXRz
ZW5jcnlwdC5vcmcwLwYIKwYBBQUHMAKGI2h0dHA6Ly9jZXJ0LmludC14My5sZXRz
ZW5jcnlwdC5vcmcvMC0GA1UdEQQmMCSCInNpZWtzZnVqaTkubG9uZ25hbWVvZmRv
bWFpbnRlc3QudGswTAYDVR0gBEUwQzAIBgZngQwBAgEwNwYLKwYBBAGC3xMBAQEw
KDAmBggrBgEFBQcCARYaaHR0cDovL2Nwcy5sZXRzZW5jcnlwdC5vcmcwggEDBgor
BgEEAdZ5AgQCBIH0BIHxAO8AdQBElGUusO7Or8RAB9io/ijA2uaCvtjLMbU/0zOW
tbaBqAAAAXUyohbIAAAEAwBGMEQCIFS3+ihPJVg9PYXc6xIJDyqdf80yvKUiKEzw
i7xVo5XHAiA7NE/MZmjGoYwi/RJNBAxnO0aQetBRbRawQMLU+qN8ywB2AH0+8viP
/4hVaCTCwMqeUol5K8UOeAl/LmqXaJl+IvDXAAABdTKiFuwAAAQDAEcwRQIhALpK
XooGXWQnyZRxqqPe+NW0EZpnCUa7uw9nzs8uTLQUAiAcMLyQKjhWbID2EuCyYQek
ijhsL5iV11cLZi/R4G2M8DANBgkqhkiG9w0BAQsFAAOCAQEAR9PuPeyPkmW8zjb3
uNnrSwowynSDT+u73vFfo294vV2miN0ee2pLdpfZwkOXcwhXeFQd1Rb11MuIMgp/
VCzZc0o5JwHef6AsUUaFlLFxZn8ElVknVzop2bnZIvQ+6zzPbbQrnenIqF5j6tWn
zdeAXmaP6bz0RZf/8qrEIFYcsXhP/2JSFvdxZ+oVkVKCZ2z56GQKiqzpM2farg/j
v+/YtkN66G6SkwtIN+cKeL9gbURbkISsSD0IiPYRdCZvOn5RGr4/nZqRtSXzZU5u
nzazU6J60/G5QaKjPIyEsngCV6OrBAqrCnc5IJnqm+6wYCM1MKpO0jZl0o8wxQzf
mErfDQ==
-----END CERTIFICATE-----
```


## 備考
- 暗号化スイートの確認方法  
curl -s -k -v --tlsv1.2 https://www.yahoo.co.jp
