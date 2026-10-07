# HashiCorp Vault Workshop

[Vault](https://www.vaultproject.io/) は HashiCorp が中心に開発をする OSS のシークレット管理ツールです。Vault を利用することで既存の Static なシークレット管理のみならず、クラウド、データベースや SSH などの様々なシークレットを動的に発行することができます。Vault はマルチプラットフォームでかつ全ての機能を HTTP API で提供しているため、環境やクライアントを問わず利用することができます。

本ワークショップは OSS の機能を中心に様々なユースケースに合わせたハンズオンを用意しています。本ドキュメントは、従来はページごとに分かれていたハンズオンを 1 つのファイルにまとめたフラット版です。

## Pre-requisite

* 環境
	* macOS or Linux(Ubuntu 推奨)

* ソフトウェア
	* Vault
	* Docker
	* MySQL クライアント
	* Java 12(いつか直します...)
	* jq, watch, wget, curl
	* vagrant(SSH の章のみ必要)
	* minikube(Kubernetes の章のみ必要)
	* helm(Kubernetes の章のみ必要)

* アカウント
	* GitHub
	* AWS / Azure / GCP

## Vault 概要の学習

* こちらのビデオをご覧ください。

[HashiCorp Vault で始めるクラウドセキュリティ対策](https://www.youtube.com/watch?v=PJaNVSEXcUA&t=1s)

## 資料

* [Vault Overview](https://docs.google.com/presentation/d/14YmrOLYirdWbDg5AwhuIEqJSrYoroQUQ8ETd6qwxe6M/edit?usp=sharing)

## お勧めの進め方

初めて Vault を扱う人は下記の順番で消化すると一通りの Vault の使い方が掴めるためお勧めです。

1. 初めての Vault
2. Databases Secret Engine
3. 認証とポリシー & トークン
4. Auth Method のいずれか
5. Public Clouds Secret Engine のいずれか
6. Transit

## その他の参考リンク

このフラット版には、以下の外部ホスティングされたコンテンツは含まれていません。必要に応じて個別に参照してください。

* [Auth Method: OIDC](https://learn.hashicorp.com/vault/operations/oidc-auth)
* [Auth Method: GitHub](https://learn.hashicorp.com/vault/getting-started/authentication)
* [HashiCorp Nomad との連携機能](https://github.com/hashicorp-japan/nomad-workshop/blob/master/contents/nomad-vault.md)
* [Enterprise 機能の紹介](https://docs.google.com/presentation/d/1dtoRmLxySDL8PTEe_X51BQNIXn19H_910StO2DlFkLI/edit?usp=sharing)
* [Vault Ops Workshop](https://docs.google.com/document/d/1KWl3Krv3L4A0UQmw5deanXHGKr5Mu8kKoTMGNEyAgTM/edit#heading=h.wr5wzikn620)

## 目次

- [初めての Vault](#初めての-vault)
- [Secret Engine: KV](#secret-engine-kv)
- [Secret Engine: Databases](#secret-engine-databases)
- [Vault のポリシーを使ってアクセス制御する](#vault-のポリシーを使ってアクセス制御する)
- [Policy のエクササイズ](#policy-のエクササイズ)
- [Token について](#token-について)
- [AppRole による認証](#approle-による認証)
- [Vault AWS auth demo](#vault-aws-auth-demo)
- [AWS のシークレットエンジンを試す](#aws-のシークレットエンジンを試す)
- [Vault を PKI エンジンとして扱う](#vault-を-pki-エンジンとして扱う)
- [Transit シークレットエンジンで Vault を Encryption as a Sevice として使う](#transit-シークレットエンジンで-vault-を-encryption-as-a-sevice-として使う)
- [Transform Secret Engine を試す](#transform-secret-engine-を試す)
- [Response Wrapping を使ってシークレットをセキュアに渡して取得する。](#response-wrapping-を使ってシークレットをセキュアに渡して取得する)

---

## 初めての Vault

ここではまず Vault のインストール、unseal と初めてのシークレットを作ってみます。

### Vault のインストール

[こちら](https://www.vaultproject.io/downloads.html)の Web サイトからご自身の OS に合ったものをダウンロードしてください。

```
wget https://releases.hashicorp.com/vault/1.3.0/vault_1.3.0_linux_amd64.zip
```

パスを通します。以下は macOS の例ですが、OS にあった手順で`vault`コマンドにパスを通します。

```shell
unzip vault*.zip
chmod +x vault
mv vault /usr/local/bin
```

新しい端末を立ち上げ、Vault のバージョンを確認します。

```console
$ vault -version                                                                       
Vault v1.1.1+ent ('7a8b0b75453b40e25efdaf67871464d2dcf17a46')
```

これでインストールは完了です。

### 初めてのシークレット

次に Vault サーバを立ち上げ、Generic なシークレットを Vault に保存して取り出してみます。

```console
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault server -dev
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: false, enabled: false
                 Storage: inmem
                 Version: Vault v1.1.1+ent
             Version Sha: 7a8b0b75453b40e25efdaf67871464d2dcf17a46

WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: CNmWA769OVVTcyOptf3mFDPW5sVHOE4fw0yRnV7Tl74=
Root Token: s.rAc6mBZgrNwPxSky2dBJkgSd 
```

途中で出力される`Root Token`は後で使いますのでメモしてとっておきましょう。`-dev`モードで起動すると、データーストレージのコンフィグレーション等を行うことなく、プリセットされた設定で手軽に起動することが出来ます。クラスタ構成やデータストレージなど細かな設定が必要な場合には利用しません。また、デフォルトだとデータはインメモリに保存されるため、起動毎にデータが消滅します。

では、先ほど取得したトークンでログインしてみます。

```console
$ vault login                                                                                             
Token (will be hidden):
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.rAc6mBZgrNwPxSky2dBJkgSd
token_accessor       SgvDbZAFk0RU7bHvAtNCpw3B
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

現在有効になっているシークレットエンジンを見てみます。現在使っているトークンは root 権限と紐づいているため、現在有効になっている全てのシークレットにアクセスすることが可能です。

```console
$ vault secrets list 
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_65e8821b    per-token private secret storage
identity/     identity     identity_03927077     identity store
secret/       kv           kv_9d34f5e6           key/value secret storage
sys/          system       system_b2dfb5a6       system endpoints used for control, policy and debugging
```

`kv`シークレットエンジンを使って、簡単なシークレットを Vault に保存して取り出してみます。

```console
$ vault kv list secret/                         
No value found at secret/metadata

$ vault kv put secret/mypassword password=p@SSW0d
Key              Value
---              -----
created_time     2019-07-12T02:20:57.871216Z
deletion_time    n/a
destroyed        false
version          1

$ vault kv list secret/
Keys
----
mypassword

$ vault kv get secret/mypassword
====== Metadata ======
Key              Value
---              -----
created_time     2019-07-12T02:20:57.871216Z
deletion_time    n/a
destroyed        false
version          1

====== Data ======
Key         Value
---         -----
password    p@SSW0d
```

また、Vault の CLI は API への HTTPS のアクセスをラップしているため、全ての CLI での操作は API への curl のリクエストに変換できます。`-output-curl-string`を使うだけです。

```console
$ vault kv list -output-curl-string secret/
curl -H "X-Vault-Token: $(vault print token)" http://127.0.0.1:8200/v1/kv/metadata?list=true
```

curl コマンドを使ったリクエストが表示されました。アプリなどのクライアントから Vault の API を呼ぶ時などに記述方法に迷った時などに便利です。

また、デフォルトではテーブル形式ですが様々なフォーマットで出力を得られます。

```console
$ vault kv get -format=yaml secret/mypassword    
data:
  name: kabu
  password: passwd
lease_duration: 2764800
lease_id: ""
renewable: false
request_id: 33de9c9d-1455-ca31-5571-84d69d0a0b77
warnings: null

$ vault kv get -format=json secret/mypassword                            
{
  "request_id": "15a27428-e566-186b-3a47-b66c727f5f02",
  "lease_id": "",
  "lease_duration": 2764800,
  "renewable": false,
  "data": {
    "name": "kabu",
    "password": "passwd"
  },
  "warnings": null
}
```

特定のフィールドのデータを抽出することもできます。

```console
$ vault kv get -format=json -field=password secret/mypassword
"p@SSW0d"
```

さて、Vault にデータを put して get 出来ました。以降のセッションでその他のシークレットを扱っていきますが、「認証され」「ポシリーに基づいたトークンを取得し」「トークンを利用してシークレットにアクセスする」これが基本の流れです。

### Vault のコンフィグレーション

一旦 Vault のサーバを停止し、次は Vault のコンフィグレーションを作成し、起動してみます。Vault のコンフィグレーションは`HashiCorp Configuration Language`で記述します。

デスクトップに任意のフォルダーを作って、以下のファイルを作成します。ファイル名は`vault-local-config.hcl`とします。

```shell 
$ #for MacOS
$ mkdir vault-workshop
$ cd vault-workshop

$ DIR=$(pwd)
$ cat > vault-local-config.hcl <<EOF
storage "file" {
   path = "${DIR}/vaultdata"
}

listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = 1
}

ui = true
disable_mlock = true
EOF
```

```shell
$ #for Windows
$ mkdir vault-workshop
$ cd vault-workshop

$ $DIR=(pwd).Path
$ @"
storage "file" {
   path = "${DIR}\vaultdata"
}

listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = 1
}

ui = true
disable_mlock = true
"@ | Out-File -FilePath vault-local-config.hcl -Encoding Ascii

```

ここではストレージ、リスナーと UI の最低限の設定をしています。その他にも[様々な設定](https://www.vaultproject.io/docs/configuration/)が出来ます。

ストレージのタイプは複数選択できますが、ここではローカルファイルを使います。実際の運用で可用性などを考慮する場合は Consul など HA の機能が盛り込まれたストレージを使うべきです。このコンフィグを使って Vault を再度起動してみましょう。

>下記のコマンドで起動時に"Error initializing core: Failed to lock memory: cannot allocate memory"のエラーが出る場合は以下の 1 行を vault-local-config.hcl に追記してください。
> `disable_mlock  = true`

```console
$ vault server -config vault-local-config.hcl
WARNING! mlock is not supported on this system! An mlockall(2)-like syscall to
prevent memory from being swapped to disk is not supported on this system. For
better security, only run Vault on systems where this call is supported. If
you are running Vault in a Docker container, provide the IPC_LOCK cap to the
container.
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: false, enabled: false
                 Storage: file
                 Version: Vault v1.1.1+ent
             Version Sha: 7a8b0b75453b40e25efdaf67871464d2dcf17a46

==> Vault server started! Log data will stream in below:
```
今回はプロダクションモードで起動しています。先ほどと違い、`Root Token`, `Unseal Key`は出力されません。Vault を利用するまでに`init`と`unseal`という処理が必要です。

### Vault の初期化処理

別の端末を立ち上げて以下のコマンドを実行してください。GUI でも同様のことが出来ますが、このハンズオンでは全て CLI を使います。

```console
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault operator init > vault-keys
$ cat vault-keys
Unseal Key 1: E9wz16Q+6K8sHdV0G1IZNw4/xBC3b0lm28Hz0K/MyfM1
Unseal Key 2: FmP/bBJqArQ30wPDYS8GNfFUKKgUu141LtVNThrX8YyT
Unseal Key 3: K2zppWuRaDcCCCqb8NznfDw1Fp4bRXwslRoR4eTd7igz
Unseal Key 4: uxpETuMXmdwPm4AUcrusWwuHvn52A8XfGXPXwRBGajOF
Unseal Key 5: e3DwN3SOnSh/boJmCav4Ve8FOD3oSLjwywNwy+P5qrcx

Initial Root Token: s.51du1iIeam79Q5fBRBALVhRB
```

init の処理をすると、Vault を`unseal`するためのキーと`Initial Root Token`が生成されます。試しにこの状態でログインしてみます。

```console
$ vault login                                                                         
Token (will be hidden):
Error authenticating: error looking up token: Error making API request.

URL: GET http://127.0.0.1:8200/v1/auth/token/lookup-self
Code: 503. Errors:

* error performing token check: Vault is sealed
```

エラーになるはずです。Vault では`sealed`という状態になっているといかに強力な権限のあるトークンを使ったとしてもいかなる操作も受け付けません。`unseal`の処理は`Unseal Key`を使います。

デフォルトだと 5 つのキーが生成され、そのうち 3 つのキーが集まると`unseal`されます。5 つの`Unseal Key`の任意の 3 つを使ってみましょう。`vault operator unseal`コマンドを 3 度実行します。

```console
$ vault operator unseal                                                        
Unseal Key (will be hidden):
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    1/3
Unseal Nonce       5ab14385-6ea9-f09b-4429-b6942c3cc005
Version            1.1.1+ent
HA Enabled         false

$ vault operator unseal
Unseal Key (will be hidden):
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    2/3
Unseal Nonce       5ab14385-6ea9-f09b-4429-b6942c3cc005
Version            1.1.1+ent
HA Enabled         false

$ vault operator unseal
Unseal Key (will be hidden):
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    5
Threshold       3
Version         1.1.1+ent
Cluster Name    vault-cluster-a1cd882e
Cluster ID      3f7c2734-ec50-8834-e6c9-7a1c35726d4f
HA Enabled      false
``` 

3 回目の出力で`Sealed`が`false`に変化したことがわかるでしょう。この状態で再度ログインします。

```console
$ vault login
Token (will be hidden):
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.51du1iIeam79Q5fBRBALVhRB
token_accessor       z28eqFezRCtIlaH33OSnhEGt
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

これでログインは成功です。以降の章ではこの環境を使ってハンズオンを進めていきます。

### 参考リンク
* [アーキテクチャ](https://www.vaultproject.io/docs/internals/architecture.html)
* [コンフィグレーション](https://www.vaultproject.io/docs/configuration/)
* [Seal](https://www.vaultproject.io/docs/configuration/seal/index.html)
* [シャミアの秘密鍵分散法](http://ohta-lab.jp/users/mitsugu/research/SSS/main.html)
* [vault server command](https://www.vaultproject.io/docs/commands/server.html)

---

## Secret Engine: KV

ここでは非常にシンプルな Key Value Store 型のシークレットエンジンを使ってみます。KV シークレットエンジンは`-dev`モードだとデフォルトでオンになっていますが、プロダクションモードだと明示的にオンにする必要があります。

### KV シークレットエンジンを有効化

Vault では各シークレットエンジンを有効化するために`enable`の処理を行います。`enable`は特定の権限を持ったトークンのみが実施できるようにすべきですが、ここでは root token を使います。ポリシーについては後ほど扱います。

```console
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault secrets enable -path=kv -version=2 kv
Success! Enabled the kv secrets engine at: kv/

$ vault secrets list
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_e3aa0798    per-token private secret storage
identity/     identity     identity_86c0240d     identity store
kv/           kv           kv_12159ddb           n/a
sys/          system       system_ae51ee57       system endpoints used for control, policy and debugging
```

`kv`が有効化され、`kv/`が API のエンドポイントとしてマウントされました。以降はこのパスを利用して KV データを扱っていきます。

### KV データのライフサイクル

先ほどと同様、データを put してみましょう。

```console
$ vault kv put kv/iam name=kabu password=passwd
$ vault kv get kv/iam                                            
====== Data ======
Key         Value
---         -----
name        kabu
password    passwd
```

#### データの更新
データの更新には 2 通りの方法があります。

まずは上書きしてデータのバージョンを上げる方法です。
```console
$ vault kv enable-versioning kv
$ vault kv get kv/iam
====== Metadata ======
Key              Value
---              -----
created_time     2019-07-12T06:00:44.023139Z
deletion_time    n/a
destroyed        false
version          1

====== Data ======
Key         Value
---         -----
name        kabu
password    passwd
```

`enable-versioning`をするとメタデータが付与され、バージョン管理されます。データを上書きしてバージョン 2 を作ってみます。

```console
$ vault kv put kv/iam name=kabu-2 password=passwd
Key              Value
---              -----
created_time     2019-07-12T06:08:03.871067Z
deletion_time    n/a
destroyed        false
version          2

$ vault kv get kv/iam
====== Metadata ======
Key              Value
---              -----
created_time     2019-07-12T06:08:03.871067Z
deletion_time    n/a
destroyed        false
version          2

====== Data ======
Key         Value
---         -----
name        kabu-2
password    passwd
```

データが上書きされてバージョン 2 のデータが生成されました。古いバージョンのデータは`-version`オプションを付与することで参照できます。

```console
$ vault kv get -version=1 kv/iam
====== Metadata ======
Key              Value
---              -----
created_time     2019-07-12T06:13:49.354811Z
deletion_time    n/a
destroyed        false
version          1

====== Data ======
Key         Value
---         -----
name        kabu
password    passwd
```

古いバージョンのデータを削除する際は以下の手順です。

```console
$ vault kv destroy -versions=1 kv/iam
$ vault kv get -version=1 kv/iam
====== Metadata ======
Key              Value
---              -----
created_time     2019-07-12T06:13:49.354811Z
deletion_time    n/a
destroyed        true
version          1

$ vault kv get -version=2 kv/iam
====== Metadata ======
Key              Value
---              -----
created_time     2019-07-12T06:14:08.207969Z
deletion_time    n/a
destroyed        false
version          2

====== Data ======
Key         Value
---         -----
name        kabu-2
password    passwd
```

二つ目の更新の方法は`-patch`オプションを付与する方法です。先ほどの上書きの方法だと、キーを忘れてアップデートするとどうなるか試してみましょう。

```console
$ vault kv put kv/iam password=passwd-2

Key              Value
---              -----
created_time     2019-07-12T06:19:28.028113Z
deletion_time    n/a
destroyed        false
version          3

$ vault kv get kv/iam
====== Metadata ======
Key              Value
---              -----
created_time     2019-07-12T06:19:28.028113Z
deletion_time    n/a
destroyed        false
version          3

====== Data ======
Key         Value
---         -----
password    passwd-2
```

このようにキーの存在ごと上書きされてしまいます。つぎに`patch`オプションを使ってみます。まずはデータを戻します。

```console
$ vault kv put kv/iam name=kabu-2 password=passwd
Key              Value
---              -----
created_time     2019-07-12T06:21:38.791328Z
deletion_time    n/a
destroyed        false
version          4

$ vault kv get kv/iam
====== Metadata ======
Key              Value
---              -----
created_time     2019-07-12T06:21:38.791328Z
deletion_time    n/a
destroyed        false
version          4

====== Data ======
Key         Value
---         -----
name        kabu-2
password    passwd
```

データの一部を更新してみましょう。

```console
$ vault kv patch kv/iam password=passwd-2
$ vault kv get kv/iam
====== Metadata ======
Key              Value
---              -----
created_time     2019-07-12T06:25:39.688255Z
deletion_time    n/a
destroyed        false
version          5

====== Data ======
Key         Value
---         -----
name        kabu-2
password    passwd-2
```
`patch`を使うとデータの一部のみを更新できます。

最後にデータを削除します。

```console
$ vault kv delete kv/iam
$ vault kv metadata delete kv/iam
```

### 参考リンク
* [Vault KV Secret Engine](https://www.vaultproject.io/docs/secrets/kv/kv-v2.html)
* [vault kv command](https://www.vaultproject.io/docs/commands/kv/patch.html)
* [API Document](https://www.vaultproject.io/api/secret/kv/index.html)

---

## Secret Engine: Databases

ここではデータベースのシークレットエンジンを扱い、MySQL データベースのシークレットを生成してみます。データベースのシークレットエンジンでは、特定の権限を与えたデータベースユーザを動的に生成、削除することが出来ます。

これにより、複数のクライアントが同じシークレットを使いまわすことを防いだり、たとえシークレットが漏れても即座に破棄するなどの運用が可能になります。

Vault はデフォルトでは以下のような Database に対応しています。
* Cassandra
* Influxdb
* HanaDB
* MongoDB
* MSSQL
* MySQL/MariaDB
* PostgreSQL
* Oracle

### Database シークレットエンジンの有効化

KV と同様`database`シークレットエンジンを`enable`します。

```console
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault secrets enable -path=database database
Success! Enabled the database secrets engine at: database/
```
マウントされた`database/`のエンドポイントを使うことでデータベースシークレットエンジンに対する様々な操作が可能です。

### Database の動的シークレットの発行

流れとしては以下の通りです。
* Vault に特権ユーザのクレデンシャルとデータベースの接続先を登録する
* ロールを定義し、Vault が発行するデータベースユーザの設定を行う
	* データベースに対する権限
	* Time to Live
* クライアントから Vault に対してシークレットの発行を依頼する

#### MySQL の準備

ローカルの Docker 上で MySQL を起動してください。

```shell
$ docker run --name mysql -e MYSQL_ROOT_PASSWORD=rooooot -p 3306:3306 -d mysql:5.7.22
```

root でログインをしたら、サンプルのデータを投入します。パスワードは`rooooot`です。

```shell
$ mysql -u root -p -h 127.0.0.1 --ssl-mode=DISABLED
```

```mysql
mysql> create database handson;
mysql> use handson;
mysql> create table products (id int, name varchar(50), price varchar(50));
mysql> insert into products (id, name, price) values (1, "Nice hoodie", "1580");
```

これで MySQL の準備は完了です。

<details><summary>Dockerではなくローカルで起動の場合はこちら</summary>

```console
$ sudo mysql.server start
Password:
Starting MySQL
.Logging to '/usr/local/var/mysql/Takayukis-MacBook-Pro.local.err'.
 SUCCESS!
 ```

> root ユーザのパスワードが設定されていない場合、以下のコマンドで変更してください。
> ```console
> $ mysql -u root
> ```
> 
> ```mysql
> mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'rooooot';
> Query OK, 0 rows affected (0.00 sec)
> 
> mysql> exit
> ```
> 
> ```console
> $ sudo mysql.server restart
> ```

ログインを試してみてください。
```console
$ mysql -u root -p
```
</details>

#### Vault の設定

まずはデータベースへのコネクションの設定を Vault に行います。これ以降 Vault はこのパラメータを使ってユーザを払い出します。そのため強い権限のユーザを登録する必要があります。

```shell
$ vault write database/config/mysql-handson-db \
  plugin_name=mysql-legacy-database-plugin \
  connection_url="{{username}}:{{password}}@tcp(127.0.0.1:3306)/" \
  allowed_roles="role-handson" \
  username="root" \
  password="rooooot"
```

>ここで MySQL への Access Denied でエラーになる方は下記のコマンドを実行してください。
>ここでの MySQL の再起動方法は OS によって異なります。
>```
>sudo mysql -u root -p
>mysql> USE mysql;
>mysql> UPDATE user SET plugin='mysql_native_password' WHERE User='root';
>mysql> FLUSH PRIVILEGES;
>mysql> exit;
>service mysql restart
>mysql login -p root
>mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'rooooot'; 
>service mysql restart
>```
`database/config/*****`はコンフィグの名前、任意に指定可能です。`plugin_name`は別途説明します。`allowed_roles`はこれから作成するユーザのロールの名前です。`allowed_roles`は List 型になっており、一つのコンフィグに複数のロールを紐づけることが可能です。

次にロールの定義をします。

```shell
$ vault write database/roles/role-handson \
  db_name=mysql-handson-db \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" \
  default_ttl="1h" \
  max_ttl="24h"
```

```console
$ vault list database/roles                         
Keys
----
role-handson
```

次に`/database/creds`のエンドポイントを使って、ロール名`role-handson`に基づいたシークレットを発行します。このエンドポイントが使われない限りシークレットは発行されません。通常この処理はクライアントから実行します。

```console
$ vault read database/creds/role-handson
Key                Value
---                -----
lease_id           database/creds/role-handson/nN0DRYCywFdU5Hjin0xLSGGs
lease_duration     1h
lease_renewable    true
password           A1a-P0cpPpKeKzdtv6hP
username           v-role-YpuDx1rjz
```

#### MySQL にアクセスして権限を試す

次に発行したユーザを使って MySQL サーバにアクセスしてみます。

```console 
$ mysql -u <USERNAME_GEN_BY_VAULT>  -h 127.0.0.1 -p handson
Enter password: <PASSWORD__GEN_BY_VAULT>
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 6
Server version: 5.7.25 Homebrew

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```

投入したデータを参照してみます。

```mysql
mysql> use handson;
mysql> show tables;
+-------------------+
| Tables_in_handson |
+-------------------+
| products          |
+-------------------+

mysql> select * from products;
+------+--------------+-------+
| id   | name         | price |
+------+--------------+-------+
|    1 | Nice hoodie  | 1580  |
+------+--------------+-------+

mysql> insert into products (id, name, price) values (1, "aaa", "bbb");
ERROR 1142 (42000): INSERT command denied to user 'v-role-InoM8WOwU'@'localhost' for table 'product'

mysql> create table test (id int, name varchar(10), price varchar(10));
ERROR 1142 (42000): CREATE command denied to user 'v-role-InoM8WOwU'@'localhost' for table 'test'
```

`select`の処理は実行出来ますが、そのほかの`insert`や`create table`の処理は権限上実行不可能なことがわかります。次はもう少し権限を絞ってみましょう。現在のロールは`GRANT SELECT ON *.*`とある通り、全てのデータベースと全てのテーブルに対して`select`の権限を与えています。

```mysql
mysql> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+---------------------------+
| Tables_in_mysql           |
+---------------------------+
| columns_priv              |
| db                        |
| engine_cost               |
| event                     |
| func                      |
| general_log               |
| gtid_executed             |
| help_category             |
| help_keyword              |
~~~~~~~~~~~
```

次は該当のテーブルだけにアクセスできるロールを作ってみましょう。

```shell 
$ vault write database/config/mysql-handson-db \
  plugin_name=mysql-legacy-database-plugin \
  connection_url="{{username}}:{{password}}@tcp(127.0.0.1:3306)/" \
  allowed_roles="role-handson","role-handson-2" \
  username="root" \
  password="rooooot"
```

```shell
$ vault write database/roles/role-handson-2 \
  db_name=mysql-handson-db \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON handson.products TO '{{name}}'@'%';" \
  default_ttl="1h" \
  max_ttl="24h"
```

`allowed_roles`に`role-handson-2`を追加し、`role-handson-2`を作成しています。`GRANT SELECT ON handson.products`としています。

このロールを使ってユーザを発行してログインしてみます。

```console
$ vault read database/creds/role-handson-2
Key                Value
---                -----
lease_id           database/creds/role-handson-2/KXAWRvI0aawT9KObG3fVGLJo
lease_duration     1h
lease_renewable    true
password           A1a-8aBlTXjRSu9eR3y1
username           v-role-Ync7153K8

$ mysql -u <USERNAME_GEN_BY_VAULT> -h 127.0.0.1 -p                  
Enter password: <PASSWORD__GEN_BY_VAULT>
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 7
Server version: 5.7.25 Homebrew

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

```mysql
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| handson            |
+--------------------+
2 rows in set (0.01 sec)
```

`handson`という名前のデータベースのみ権限があることがわかります。このような形でロールを定義し、必要な権限のユーザを必要な時に動的に生成することができます。次はシークレットの破棄を扱います。

#### 動的シークレットの破棄

一つは TTL を設定した自動破棄です。短い TTL を設定した新しいロールを作ってみます。

```shell
$ vault write database/config/mysql-handson-db \
  plugin_name=mysql-legacy-database-plugin \
  connection_url="{{username}}:{{password}}@tcp(127.0.0.1:3306)/" \
  allowed_roles="role-handson","role-handson-2","role-handson-3" \
  username="root" \
  password="rooooot"
```
```shell
$ vault write database/roles/role-handson-3 \
  db_name=mysql-handson-db \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON handson.products TO '{{name}}'@'%';" \
  default_ttl="120s" \
  max_ttl="360s"
Success! Data written to: database/roles/role-handson
```

`default_ttl`のパラメータに 120 秒を指定しています。`default_ttl`は生成した時の TTL、`max_ttl`は`renew`できる最大の TTL です。

このロールを利用して Short Lived なユーザを発行します。

```console
$ vault read database/creds/role-handson-3
Key                Value
---                -----
lease_id           database/creds/role-handson-3/H3y6DjZBGztisnO3B3DqzgkA
lease_duration     120s
lease_renewable    true
password           A1a-0VP1UDi5BPEMnPnZ
username           v-role-bnsYTFQAj
```

`lease_duration`が設定した TTL の 120 秒になっています。これを使ってまずは試しにログインしてみます。

```console
$ mysql -u <USERNAME_GEN_BY_VAULT> -h 127.0.0.1 -p           
Enter password: <PASSWORD__GEN_BY_VAULT>
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 16
Server version: 5.7.25 Homebrew

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> exit
Bye¥
```

120 秒後に再度ログインします。

```console
$ mysql -u <USERNAME_GEN_BY_VAULT> -h 127.0.0.1 -p        
Enter password: <PASSWORD__GEN_BY_VAULT>
ERROR 1045 (28000): Access denied for user 'v-role-bnsYTFQAj'@'localhost' (using password: YES)
```

ユーザが破棄され、利用不可能になりました。

2 つ目の方法は`revoke`コマンドを使って明示的に破棄する方法です。`role-handson-2`のロールを使って新規のユーザを払い出します。払い出された`lease_id`をメモっておいてください。revoke の際に使用します。

```console
$ vault read database/creds/role-handson-2
Key                Value
---                -----
lease_id           database/creds/role-handson-2/JSnf6zV2jTrRJmI66Hfz189K
lease_duration     1h
lease_renewable    true
password           A1a-TaZktgzsQw4FfIT8
username           v-role-jklQMrcJa

$ mysql -u <USERNAME_GEN_BY_VAULT> -p -h 127.0.0.1 -p handson
Enter password: <PASSWORD__GEN_BY_VAULT>
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 22
Server version: 5.7.25 Homebrew

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

`revoke`コマンドを実行してみます。

```console
$ vault lease revoke database/creds/role-handson-2/JSnf6zV2jTrRJmI66Hfz189K
All revocation operations queued successfully!

$ mysql -u <USERNAME_GEN_BY_VAULT> -p -h 127.0.0.1 -p handson
Enter password: <PASSWORD__GEN_BY_VAULT>
ERROR 1045 (28000): Access denied for user 'v-role-jklQMrcJa'@'localhost' (using password: YES)
```

revoke され、ログインが出来なくなりました。このように Vault ではシークレットを動的に生成し、短い時間でユーザを細かく破棄し、クレデンシャルをセキュアに保つ運用が簡単に実現できます。

### Root ユーザのパスワードローテーション

Vault には Root ユーザの権限を持たせる必要があるため、Root ユーザのパスワードの扱いは非常にセンシティブです。Vault にはコンフィグレーションとして登録したデータベースのルートユーザのパスワードをローテーションさせる API を持っています。これを使ってこまめに Root のパスワードをリフレッシュできます。

**Vault によって Root パスワードのローテーションを行った後は Root のパスワードは Vault しか扱うことができません。そのため通常別の特権ユーザを準備してから行います。** 以下の手順はデフォルトのルートユーザをローテーションさせる手順のため、実施後 Vault からしか使えなくなります。こちらを実行するかはお任せします。

まず、`root_rotation_statements`のパラメータをコンフィグに追加してローテーションの API が呼ばれた時に実施する処理を記述します。

```shell
$ vault write database/config/mysql-handson-db \
  plugin_name=mysql-legacy-database-plugin \
  connection_url="{{username}}:{{password}}@tcp(127.0.0.1:3306)/" \
  allowed_roles="role-handson","role-handson-2","role-handson-3" \
  username="root" \
  password="rooooot" \
  root_rotation_statements="SET PASSWORD = PASSWORD('{{password}}')"
``` 

その後、`rotate-root`の API を実行するだけです。

```console
$ vault write -force database/rotate-root/mysql-handson-db
Success! Data written to: database/rotate-root/mysql-handson-db

$ mysql -u root -p -h 127.0.0.1
Enter password: rooooot
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
```

古い Root パスワードは破棄され、利用不可能となりました。

### 参考リンク
* [API Documents](https://www.vaultproject.io/api/secret/databases/index.html)
* [Lease, Renew, and Revoke](https://www.vaultproject.io/docs/concepts/lease.html)

---

## Vault のポリシーを使ってアクセス制御する

ここでは Vault がサポートするいくつかの認証プロバイダーとの連携と、ポリシーによるアクセスコントロールを試してみます。これらの機能を使うことでクライアントとなるユーザ、ツールやアプリに対してどのリソースに対して、どの権限を与えるかというアイデンティティベースのセキュリティを設定することが出来ます。

ここまで Root Token を利用して様々なシークレットを扱ってきましたが、実際の運用では強力な権限を持つ Root Token は保持をせずに必要な時のみ生成します。通常、最低限の権限のユーザを作成し Vault を利用していきます。また認証も直接トークンで行うのではなく信頼できる認証プロバイダに委託することがベターです。

ここではその一部の方法とポリシーの設定を扱います。

### 初めてのポリシー

まず、プリセットされるポリシー一覧を確認してみましょう。ポリシーを管理するエンドポイントは`sys/policy`と`sys/policies`です。`sys`のエンドポイントには[その他にも様々な機能](https://www.vaultproject.io/api/system/index.html)が用意されています。

```console
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault secrets enable -path=kv -version=2 kv
$ vault secrets enable -path=database database
$ vault list sys/policy

Keys
----
default
root
```

```console
$ vault read sys/policy/default

Key      Value
---      -----
name     default
rules    # Allow tokens to look up their own properties
path "auth/token/lookup-self" {
    capabilities = ["read"]
}

# Allow tokens to renew themselves
path "auth/token/renew-self" {
    capabilities = ["update"]
}

# Allow tokens to revoke themselves
path "auth/token/revoke-self" {
    capabilities = ["update"]
}
~~~~
```

`path`と指定されているのが各エンドポイントで`capablities`が各エンドポイントに対する権限を現しています。試しに`default`の権限を持つトークンを発行してみましょう。`default`にはこの前に作成した`database`への権限はないので`database`のパスへの如何なる操作もできないはずです。

```console
$ vault token create -policy=default
Key                  Value
---                  -----
token                s.acBPCz3lfDryfVr01RgwyTqK
token_accessor       DnUd62Wcfwbg6eDX5Mhha0jf
token_duration       768h
token_renewable      true
token_policies       ["default"]
identity_policies    []
policies             ["default"]
```

`default`の権限を持ったトークンを生成しました。このトークンをコピーします。Token を環境変数にセットしておきましょう。

```shell
$ export DEFAULT_TOKEN=s.acBPCz3lfDryfVr01RgwyTqK
$ export ROOT_TOKEN=s.51du1iIeam79Q5fBRBALVhRB
```

`database`エンドポイントにアクセスしましょう。権限がないため`permission denied`が発生します。

```console
$ VAULT_TOKEN=$DEFAULT_TOKEN vault list database/roles
Error listing database/roles/: Error making API request.

URL: GET http://127.0.0.1:8200/v1/database/roles?list=true
Code: 403. Errors:

* 1 error occurred:
	* permission denied
```

### ポリシーを作る

ポリシーは Vault のコンフィグレーションと同様`HCL`で記述します。

```shell
$ cd /path/to/vault-workshop
$ cat > my-first-policy.hcl <<EOF
path "database/*" {
  capabilities = [ "read", "list"]
}
EOF
```

作ったら`vault policy write`のコマンドでポリシーを作成します。ポリシーの作成は Root Token で実施します。

```console
$ VAULT_TOKEN=$ROOT_TOKEN vault policy write my-policy my-first-policy.hcl
Success! Uploaded policy: my-policy

$ vault policy list           
default
my-policy
root

$ vault policy read my-policy
path "database/*" {
  capabilities = [ "read", "list"]
}
```

新しいポリシーができました。このポリシーと紐づけられたトークンは`database`エンドポイントへの`read`, `list`の権限を与えられます。ではトークンを発行してみます。

```console
$ VAULT_TOKEN=$ROOT_TOKEN vault token create -policy=my-policy 
Key                  Value
---                  -----
token                s.bA9M42W41G7tF90REMDCtMeO
token_accessor       LfQCnqPOJHGqO8TplfSjTNFs
token_duration       768h
token_renewable      true
token_policies       ["default" "my-policy"]
identity_policies    []
policies             ["default" "my-policy"]
```

Vault にこのトークンを使って以下のコマンドを実行してください。

```shell
$ export MY_TOKEN=s.bA9M42W41G7tF90REMDCtMeO
```

```console
$ VAULT_TOKEN=$MY_TOKEN vault list database/roles       
Keys
----
role-handson
role-handson-2
role-handson-3

$ VAULT_TOKEN=$MY_TOKEN vault read database/roles/role-handson
Key                      Value
---                      -----
creation_statements      [CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';]
db_name                  my-mysql-database
default_ttl              1h
max_ttl                  24h
renew_statements         []
revocation_statements    []
rollbakc_statements      []

$ VAULT_TOKEN=$MY_TOKEN vault kv list kv/
Error making API request.

URL: GET http://127.0.0.1:8200/v1/sys/internal/ui/mounts/kv
Code: 403. Errors:

* preflight capability check returned 403, please ensure client's policies grant access to path "kv/"
```

Database のエンドポイントの read, list 出来てきますが kv エンドポイントには権限がないことがわかります。

次に Database エンドポイントに write の処理をしてみましょう。

```shell
$ VAULT_TOKEN=$MY_TOKEN vault write database/roles/role-handson-4 \
    db_name=mysql-handson-db \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON handson.product TO '{{name}}'@'%';" \
    default_ttl="30s" \
    max_ttl="30s"
```

エラーが出るはずです。

```
Error writing data to database/roles/role-handson-4: Error making API request.
URL: PUT http://127.0.0.1:8200/v1/database/roles/role-handson-4
Code: 403. Errors:

* 1 error occurred:
	* permission denied
```

ポリシーに設定した通り、`database`に対する`read`, `list`の処理が成功しましたが`write`の処理、`kv`に対する処理はエラーが発生したことがわかります。`deny by default`というルールのもと、指定したもの以外は全て`deny`となります。

もう少し細かいポリシーに変更してみましょう。

[ドキュメント](https://www.vaultproject.io/docs/concepts/policies.html)を見ながら`database/roles`以下の直下のすべてのリソースに対して`create`,`read`,`list`の権限があるが、`database/roles/role-handson`だけには一切アクセスできないコンフィグファイルを作ってみてください。

正解は[こちら](https://raw.githubusercontent.com/tkaburagi/vault-configs/master/policies/my-first-policy.hcl)です。

```console
$ VAULT_TOKEN=$ROOT_TOKEN vault policy write my-policy my-first-policy.hcl
$ VAULT_TOKEN=$ROOT_TOKEN vault token create -policy=my-policy -ttl=20m
```

```shell
$ export MY_TOKEN=<TOKEN_ABOVE>
```

```
$ VAULT_TOKEN=$MY_TOKEN vault list database/roles
Keys
----
role-handson
role-handson-2
role-handson-3

$ VAULT_TOKEN=$MY_TOKEN vault read database/roles/role-handson
Error reading database/roles/role-handson: Error making API request.

URL: GET http://127.0.0.1:8200/v1/database/roles/role-handson
Code: 403. Errors:

* 1 error occurred:
	* permission denied

$ VAULT_TOKEN=$MY_TOKEN vault read database/roles/role-handson-2
Key                      Value
---                      -----
creation_statements      [CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON handson.product TO '{{name}}'@'%';]
db_name                  mysql-handson-db
default_ttl              1h
max_ttl                  24h
renew_statements         []
revocation_statements    []
rollback_statements      []
```

以上のようになれば OK です。

ここまではトークン発行の権限を持つユーザから直接トークンを CLI を使って発行してきました。通常クライアントから Vault を利用する際は信頼する認証プロバイダと Vault を連携させプロバイダで認証をし適切なトークンを発行するといったワークフローを簡単に実現できます。以降の章ではその方法をいくつか紹介していきます。

### 参考リンク
* [Policy API Document](https://www.vaultproject.io/api/system/policy.html)
* [Authentication](https://www.vaultproject.io/docs/concepts/auth.html)
* [Policies](https://www.vaultproject.io/docs/concepts/policies.html)
* [OIDC Provider Configuration](https://www.vaultproject.io/docs/auth/jwt_oidc_providers.html)

---

## Policy のエクササイズ

Policy は Vault にとって非常に大切なものです。
まず、Vault は全て Path ベースになっています。Policy も当然 Path ベースになります。

Policy によって、各 Path に対して細かいアクセス制御（**Capabilities**)を設定できます。

Capabilities には以下のようなものがあります。

Capabilities  | 内容  |  対応する HTTP API method
--|---|--
create | データの作成を許可  | `POST` `PUT`
read  | データの読み取りを許可  | `GET`
update | データの変更を許可 | `POST` `PUT`
delete | データの削除を許可  | `DELETE`
list  | Path にあるデータのリストを表示  | `LIST`
sudo  | `root-protected`の Path へのアクセスを許可  | n/a
deny  | 全てのアクセスを禁止  | n/a

上記のうち、**sudo**と**deny**は特殊な Capabilities です。特に sudo は Vault の管理 API などへのアクセスをコントロールするので、主に管理者向けの Capabilities となります。

`root-protected`の Path の一覧は[こちら](https://learn.hashicorp.com/vault/identity-access-management/iam-policies#root-protected-api-endpoints)

---
### エクササイズの事前準備


それでは、実際に Policy を触ってみましょう。まずは環境を構築します。

#### Secret engine のマウント

これからのエクササイズで利用する Secret engine を設定します。

```console
vault secrets enable -path=kv_training kv
```

これで、Vault 上の`kv_training`という Path に KV の Secret engine がマウントされました。この Engine を使って、この後のエクササイズを行います。

---
### Ex1. Producer と Consumer

**シナリオ：**

登場人物  | 役割  | アクセス制限
--|---|--
Producer  | Secret を設定する  | KV へ Secret を書き込みたい 。複数のユーザへのユニークな Secret を書き込みたい。
Consumer  | Secret を利用する  | KV から Secret を読み出したい。ただし、Secret の変更や、許可されていない Secret へのアクセスは出来ない。

#### Policy の準備

まず、Producer 側の Policy を準備します。Policy は、Vault 上の Path に対し、どのような Capabilities を許可するか（または許可しないか）を記述します。

以下のようなファイルを作成し、`producer.hcl`として保存して下さい。

```hcl
$ cat <<EOF> producer.hcl

path "kv_training"
{
	capabilities = [ "list" ]
}

path "kv_training/*"
{
	capabilities = [ "create", "read", "update", "delete", "list" ]
}
EOF
```

この Policy では、
- kv_training 内の Secret のリストを表示できる
- kv_training/　以下の全ての Path に対して書き込み・読み取り・修正・削除ができる

という制御になります。

それでは次に Consumer の Policy を consumer.hcl という名前で作成します。

```hcl
$ cat <<EOF> consumer.hcl
path "kv_training"
{
	capabilities = [ "list" ]
}

path "kv_training/consumer_*"
{
	capabilities = [ "read" ]
}
EOF
```

この Policy では、

- kv_training 内の Secret のリストを表示できる
- kv_training 以下の`consumer_`で始まる Secret に対してのみ読み取りが出来る

という制御になります。

それではこれらの Policy を Vault に設定します。Policy の作成は、`vault policy write`コマンドで行います。

```console
$ vault policy write producer producer.hcl
Success! Uploaded policy: producer

$ vault policy write consumer consumer.hcl
Success! Uploaded policy: consumer
```

Success と表示されれば正常に Vault に Policy が書き込まれました。

この後のエクササイズの為に、Consumer と Producer それぞれのための Token を作成しておきましょう。Token の作成は、`vault token create`コマンドを使います。実際の Token は`s.esHB5Ggj0JoRxkInQrm9eia6`のようなランダムな文字列ですが、ここではエクササイズを簡単にするために、それぞれ`producer_token`と`consumer_token`と簡単な Token 名に設定しています（本番環境などでは決して真似しないで下さい）。


```console
$ vault token create -policy=producer -id producer_token
WARNING! The following warnings were returned from Vault:

  * Supplying a custom ID for the token uses the weaker SHA1 hashing instead
  of the more secure SHA2-256 HMAC for token obfuscation. SHA1 hashed tokens
  on the wire leads to less secure lookups.

Key                  Value
---                  -----
token                producer_token
token_accessor       ej7jPSlmXYBOkpILCQ42j3Kk
token_duration       768h
token_renewable      true
token_policies       ["default" "producer"]
identity_policies    []
policies             ["default" "producer"]

$ vault token create -policy=consumer -id consumer_token
WARNING! The following warnings were returned from Vault:

  * Supplying a custom ID for the token uses the weaker SHA1 hashing instead
  of the more secure SHA2-256 HMAC for token obfuscation. SHA1 hashed tokens
  on the wire leads to less secure lookups.

Key                  Value
---                  -----
token                consumer_token
token_accessor       QZdNTXBir9JjcQHJySlCm2Q7
token_duration       768h
token_renewable      true
token_policies       ["consumer" "default"]
identity_policies    []
policies             ["consumer" "default"]
```

Vault が Warning を吐いていますが、気にせず先に進みましょう。

---
#### Producer による Secret 作成

それでは Producer Token を使って、Secret を書き込みます。まずは Producer Token で login します。

```console
$ vault login producer_token
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                producer_token
token_accessor       ej7jPSlmXYBOkpILCQ42j3Kk
token_duration       767h53m20s
token_renewable      true
token_policies       ["default" "producer"]
identity_policies    []
policies             ["default" "producer"]
```

この状態で、kv_training 以下の Secret を表示してみましょう。

```console
$ vault list kv_training
No value found at kv_training/
```

もちろんまだ何も入っていません。それではいくつかの Secret を書き込んでみましょう。以下のコマンドを順に実行してください。

```shell
vault write kv_training/consumer_username key=consumer
vault write kv_training/consumer_password key=P@ssword
vault write kv_training/trainer_username key=trainer
vault write kv_training/trainer_password key=S3CR3T
```

これで、`consumer_`で始まるものと`trainer_`で始まる、4 つの Secret が書き込まれました。

念の為、ちゃんと書き込まれたかチェック知てみましょう。

```console
$ vault list kv_training
Keys
----
consumer_password
consumer_username
trainer_password
trainer_username
```

これで Producer の仕事は終わりです。

---
#### Consumer による Secret の読み出し

次に Consumer になりきって、先程 Producer が作成した自分専用の Secret を読み出せるか試してみます。まず Consumer Token を使って、Consumer としてログインします。

```console
$ vault login consumer_token
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                consumer_token
token_accessor       QZdNTXBir9JjcQHJySlCm2Q7
token_duration       767h40m28s
token_renewable      true
token_policies       ["consumer" "default"]
identity_policies    []
policies             ["consumer" "default"]
```

この状態で`kv_secret`内の Secret を表示してみましょう。

```console
$ vault list kv_training
Keys
----
consumer_password
consumer_username
trainer_password
trainer_username
```

Consumer Policy では、kv_training 内の Secret のリスト表示は許可されているので、どのような Secret があるかは表示できました。

では、自分用の Secret を読み出して見ましょう。

```console
$ vault read kv_training/consumer_password
Key                 Value
---                 -----
refresh_interval    768h
key                 P@ssword

$ vault read kv_training/consumer_username
Key                 Value
---                 -----
refresh_interval    768h
key                 consumer
```

このように、`consumer_`で始まる自分用の Secret は読み出せました。次に`trainer`の Secret を読み出してみましょう。

```console
$ vault read kv_training/trainer_password
Error reading kv_training/trainer_password: Error making API request.

URL: GET http://127.0.0.1:8200/v1/kv_training/trainer_password
Code: 403. Errors:

* 1 error occurred:
	* permission denied


$ vault read kv_training/trainer_username
Error reading kv_training/trainer_username: Error making API request.

URL: GET http://127.0.0.1:8200/v1/kv_training/trainer_username
Code: 403. Errors:

* 1 error occurred:
	* permission denied
```

Policy によって制限されているため読み出しは Error になります。また、Producer の用に Secret を書き込もうとしても Error になるはずです。

```console
$ vault write kv_training/consumer_password key=NEWP@ssword
Error writing data to kv_training/consumer_password: Error making API request.

URL: PUT http://127.0.0.1:8200/v1/kv_training/consumer_password
Code: 403. Errors:

* 1 error occurred:
	* permission denied
```

時間があれば、Policy や Secret に変更を加えて色々試してみて下さい。

これで、このエクササイズは終わりです。

---
### Ex2. より細かい Policy による制御

ここでは Policy による、より細かい制御をやってみます。具体的には、以下のような制御を追加します。

- Secret に必須のパラメータを設定する
- Secret の Key のパラメータに入れても良い値を指定する
- Secret の Key のパラメータに入れてはいけない値を指定する

それでは一つづつやっていきましょう。

---
#### Secret に必須のパラメータを設定する

**前のエクササイズで Consumer token でログインしている場合は、root もしくは Policy を変更できる権限の Token でログインし直して下さい。**

まず新たに Policy を作ります。
この Policy では、`required_parameters`で必須のパラメータを指定しています。この例では、この Secret には`username`と`password`という 2 つのパラメータがないと Error にする設定になります。

```console
# Policyの作成

$ cat <<EOF>> producer2.hcl

path "kv_training"
{
    capabilities = [ "list" ]
}    

path "kv_training/*"
{
    capabilities = [ "create", "read", "update", "delete", "list" ]
    required_parameters = [ "username", "password" ]
}
EOF
```

つぎにこの Policy を Vault に設定して、Token を作成し、その Token でログインします（以下の実行ログでは、これら 3 つを順番に行なっています)。

```
# Policyの登録

$ vault policy write producer2 producer2.hcl
Success! Uploaded policy: producer2

# Tokenの作成

$ vault token create -policy=producer2 -id=producer2_token
WARNING! The following warnings were returned from Vault:

  * Supplying a custom ID for the token uses the weaker SHA1 hashing instead
  of the more secure SHA2-256 HMAC for token obfuscation. SHA1 hashed tokens
  on the wire leads to less secure lookups.

Key                  Value
---                  -----
token                producer2_token
token_accessor       qF6HtnPM27VI4vAyU2mPMCBp
token_duration       768h
token_renewable      true
token_policies       ["default" "producer2"]
identity_policies    []
policies             ["default" "producer2"]

# 作成したTokenでログイン

$ vault login producer2_token
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                producer2_token
token_accessor       qF6HtnPM27VI4vAyU2mPMCBp
token_duration       767h56m53s
token_renewable      true
token_policies       ["default" "producer2"]
identity_policies    []
policies             ["default" "producer2"]

```

ここであえて、必須パラメータを 1 つ足りない Secret を書き込んでみます。

```console
$ vault write kv_training/new_secret username="foo"
Error writing data to kv_training/new_secret: Error making API request.

URL: PUT http://127.0.0.1:8200/v1/kv_training/new_secret
Code: 403. Errors:

* 1 error occurred:
	* permission denied
```

予想通り Error になりました。では、Policy で指定されている 2 つのパラメータで書き込んでみます。

```console
$ vault write kv_training/new_secret username="foo" password="bar"
Success! Data written to: kv_training/new_secret
```

予想通り成功しました。

> **注意：　上記の通り write については想定どおりの挙動をしますが、現時点（Vault 1.3）では Read についても Policy の制限が継承されてしまい Read ができないという Bug があります。**

---
#### Secret の Key のパラメータに入れても良い値を指定する

WIP

---
#### Secret の Key のパラメータに入れてはいけない値を指定する

WIP

---

## Token について

Token は Vault とやり取りする上でコアとなるものです。つまり、Token がなければ Vault とやり取りすることはできません。

この Token をいかに取得するか、というメカニズムを提供するのが Auth Method になります。
Token は通常は次のようなテキスト形式で表されます。

`s.RmHqNz3ssJMOBZsU1ldBMCi3`

Token には様々な種類のタイプや特徴がありますので、いくつか紹介していきます。

### Root token

Root token は Vault が初期化されたときに作成される特別な Token です。
また、他の Token と違い、TTL が無期限であることも特徴です。

Root という名前が示すとおり、Vault 上のどのオペレーションも行なうことができます。よって、Root Token は管理者により Vault の最初の設定のときにだけ使用し、あらかたの設定が終わったら破棄してください。再度 Root Token が必要になった際は、再作成することを推奨します。[Root Token の再作成方法はこちら](https://learn.hashicorp.com/vault/operations/ops-generate-root)

### 他の Token

Root Token の他に Vault には以下の２種類の Token のタイプがあります。

1. Service token
2. Batch token

Service Token は Vault が開発された時から存在しているもので、今もほとんどのユーザーが利用しています。また、Token に関する全ての機能（Renewal, Revokation, Child token の作成など）が使えます。ただ、その多機能ぶりと同時に処理は少々重くなります。

Batch Token は Vault 1.0 からサポートされた新しいタイプの Token です。軽量版 Service Token とも言えるもので、少量の情報だけを保持している Token になります。特徴としては、Token は全てインメモリに保存されます。よって、Vault がダウンすると Batch Token は全て失われてしまいます。その代わり非常に軽量なので、大量の Request への対応やスケーラビリティに向いています。

以下か大まかな違いのリストです。使われている用語については追々学んでいくので今はそこまで気にしないでください。

機能  | Service Token  |  Batch Token
--|---|--
Root token として使えるか？  | Yes  |  No
Child token を作れるか？  |  Yes |  No
Renew できるか？  |  Yes |  No
Max TTL が設定できるか？  |  Yes |  No
Periodic の設定ができるか？  |  Yes |  No
Accessor を持てるか？  | Yes  |  No
Cubbyhole を持てるか？  |  Yes |  No
Parent が Revoke されたら | Revoke される | 動かなくなる
動的 Secret のリースの管理  | 自分自身  | Parent
Performance Replication で使えるか？  | No  | Yes
Performance stand-by node で使えるか？  | No  |  Yes
コスト  | ヘビー  |  ライト

#### Service token

Service Token は階層構造で構成されます。
つまり、各 Token は Child Token を作成することで、Parent Token となります（もちろん Token を作成できるポリシーが必須）。Parent Token が Revoke されると、全ての Child token と付随している Lease は Revoke されます。
ただし、Orphan token は Parent なしで、自らの TTL で存在できます。

Service token には様々な機能があります。非常に豊富なので、ここでは良く使われるものを見ていきます。

>**注意**　Auth method によって設定できないものもあります。詳細は各 Auth method のドキュメントを参照ください。

##### 様々な Service token

Token  | 内容 | 作成方法
--|---|---
通常の Token  | MaxTTL の期間内なら何回でも Renew することができる | `vault  token create` | あり  | あり | できる | なし
Orphan  |  挙動は通常の Token とほぼ同じだが、Parent token がいない | `vault token create -orphan`
Periodic Token  | MaxTTL が設定されないので、Period 期間内に Renew すれば延々と利用できる | `vault token create -period`
Counting Token  | 利用できる回数に制限をもたせる　| `vault token create -use-limit`

上記に列挙したもの以外でも、Service token は Option が豊富にあり、それらを組み合わせることで用途に合わせた Token を作成できます。使用できる Option などは[API ドキュメント](https://www.vaultproject.io/api/auth/token/index.html)を参照ください。


### まとめ

Token は Vault とやり取りをするために必須のものです。また用途に合わせて挙動を設定できます。Token のライフサイクルを理解することで、より安全に Secret の管理・運用が可能になります。

Token は、 `vault token create`などで静的なものを作成し、Client へ配布する方法もありますが、各 Auth method で認証して動的に作成・配布する手段が推奨されます。

Auth method によって、設定できる Option が違ったりもするので、そのあたりは各ドキュメントを参照いただければと思います。

---

## AppRole による認証

ここまではトークン発行の権限を持つユーザ(今回の場合は root)を使ってトークンを発行してきました。

Vault では信頼する認証プロバイダで認証をし適切なトークンを発行するといったワークフローを簡単に実現できます。

Vault では以下のような認証プロバイダに対応しています。

* AppRole
* AliCloud
* AWS
* Azure
* GCP
* JWT/OIDC
* Kubernetes
* GitHub
* Okta
* LDAP

GitHub と OIDC を試してみたい方はすでに丁寧なインストラクションがあるので参考リンクを確認してみてください。ここでは AppRole を試してみます。AppRole は他の認証メソッド同様トークンを取得するための手段です。LDAP や他の認証方法が人による操作を前提としている一方 AppRole はマシンやアプリによる操作が前提とされており、自動化のワークフローに組み込みやすくなっています。

ワークフローの例は以下のようなイメージです。

![](https://learn.hashicorp.com/assets/images/vault-approle-workflow.png)

ref: [https://learn.hashicorp.com/vault/identity-access-management/iam-authentication](https://learn.hashicorp.com/vault/identity-access-management/iam-authentication)

AppRole で認証するためには`Role ID`と`Secret ID`という二つの値が必要で、username と password のようなイメージです。各 AppRole はポリシーに紐付き、AppRole で承認されるとクライアントにポリシーに基づいた権限のトークンが発行されます。

まずはポリシーを作ってみましょう。今回は先ほど作った`kv`のデータにアクセスできるようなポリシーを作ってみます。

```shell
$ cat > my-approle-policy.hcl <<EOF
path "kv/*" {
  capabilities = [ "read", "list", "create", "update", "delete"]
}
EOF
```

```shell
$ VAULT_TOKEN=$ROOT_TOKEN vault policy write my-approle path/to/my-approle-policy.hcl
```

`approle`を`enable`にし、`my-approle`のポリシーに基づいた AppRole を一つ作成します。

```console
$ VAULT_TOKEN=$ROOT_TOKEN vault auth enable approle
$ VAULT_TOKEN=$ROOT_TOKEN vault write -f auth/approle/role/my-approle policies=my-approle
$ VAULT_TOKEN=$ROOT_TOKEN vault read auth/approle/role/my-approle

Key                      Value
---                      -----
bind_secret_id           true
bound_cidr_list          <nil>
local_secret_ids         false
period                   0s
policies                 [my-approle]
secret_id_bound_cidrs    <nil>
secret_id_num_uses       0
secret_id_ttl            0s
token_bound_cidrs        <nil>
token_max_ttl            0s
token_num_uses           0
token_ttl                0s
token_type               default
```

これで AppRole の作成は完了です。次に`Role ID`を取得します。

```console
$ VAULT_TOKEN=$ROOT_TOKEN vault read auth/approle/role/my-approle/role-id
Key        Value
---        -----
role_id    a25b3148-7b95-57bf-bc5d-cb72ffc08e68
```

次に`Secret ID`を取得しますが、いくつかの方法があります。

一つは`push`と呼ばれる方法で、カスタムの値を指定するパターンです。

```console
$ VAULT_TOKEN=$ROOT_TOKEN vault write -f auth/approle/role/my-approle/custom-secret-id secret_id=ZeCletlb
Key                   Value
---                   -----
secret_id             ZeCletlb
secret_id_accessor    c2b12a4a-0fbf-45ce-b135-be2c1d829b06
```

push 型はカスタムの値をして出来ますが、Vault 以外のサーバ、アプリやツールなど Secret ID を発行する側に Secret ID を知らせてしまうことになるため、通常使用しません。`pull`と呼ばれる方法が一般的です。

```console
$ VAULT_TOKEN=$ROOT_TOKEN vault write -f auth/approle/role/my-approle/secret-id
Key                   Value
---                   -----
secret_id             1cef3c1e-feca-99d8-ecd4-7a17ca997919
secret_id_accessor    f620512c-e9e9-4f84-bbf6-9f4d484ff2bc
```

この場合、クライアントに値を持たせることがなく Secret ID の発行が可能となりよりセキュアです。

これらを使って認証し、トークンを取得してみましょう。

```console
$ VAULT_TOKEN=$ROOT_TOKEN vault write auth/approle/login role_id="a25b3148-7b95-57bf-bc5d-cb72ffc08e68" secret_id="1cef3c1e-feca-99d8-ecd4-7a17ca997919"
Key                     Value
---                     -----
token                   s.nEolH5Pjqf3207KljT9xoamS
token_accessor          bzRhTIXZ2GzggmDFiuDUgWJy
token_duration          768h
token_renewable         true
token_policies          ["default" "my-approle"]
identity_policies       []
policies                ["default" "my-approle"]
token_meta_role_name    my-approle-policy
```

AppRole により認証され、発行されたトークンを試してみましょう。

```shell
$ export MY_TOKEN=s.nEolH5Pjqf3207KljT9xoamS
```

```shell
VAULT_TOKEN=$ROOT_TOKEN vault secrets enable -version=2 kv
VAULT_TOKEN=$ROOT_TOKEN vault kv put kv/iam password=p@SSW0d
```


```console
$ VAULT_TOKEN=$MY_TOKEN vault kv get kv/iam
====== Metadata ======
Key              Value
---              -----
created_time     2019-09-05T02:02:17.120801Z
deletion_time    n/a
destroyed        false
version          1

====== Data ======
Key         Value
---         -----
password    p@SSW0d2

$ VAULT_TOKEN=$MY_TOKEN vault read database/roles/role-demoapp
Error reading database/roles/role-demoapp: Error making API request.

URL: GET http://127.0.0.1:8200/v1/database/roles/role-demoapp
Code: 403. Errors:

* 1 error occurred:
  * permission denied
```

ロールで定義した通り、AppRole で発行したトークンは KV に対してのみアクセス権限があることがわかるでしょう。

今回は AppRole の基本的な使い方を試しましたが、より実践的にどのように扱うかは[こちらの記事](https://blog.kabuctl.run/?p=94)に記載しておきましたので、本ハンズオン終了後、一読してみてください。

### 参考リンク
* [AppRole API Document](https://www.vaultproject.io/api/auth/approle/index.html)
* [AppRole Auth Method](https://www.vaultproject.io/docs/auth/approle.html)
* [Auth0 を使った OIDC 認証](https://learn.hashicorp.com/vault/operations/oidc-auth)
* [GitHub を使った認証](https://learn.hashicorp.com/vault/getting-started/authentication)

---

## Vault AWS auth demo

Original project: [Vault Agent with AWS](https://learn.hashicorp.com/vault/identity-access-management/vault-agent-aws)

---
### 概要

Vault の AWS での Authentication のデモになります。
デモの実行については、この Repo を Clone して[こちらの Asset](assets/auth_aws)をご使用ください。

AWS auth method については、[こちら](https://www.vaultproject.io/docs/auth/aws.html)を参照ください。

AWS auth method には２つのタイプがあります。`iam`と`ec2`の２種類です。
`iam`method では、IAM クレデンシャルでサインされた特別な AWS リクエストに対して認証を行います。IAM クレデンシャルは IAM instance profile や Lambda などで自動的に作成されるので、AWS 上のほぼ全てのサービスに対して利用できます。

`ec2`method は、AWS が EC2 インスタンスに自動的に付与するメタデータを用いて認証を行います。よって、この認証方法は EC2 のインスタンスにしか利用できません。

`ec2`method は`iam`method の登場の前に開発されたもので、現在のベスト・プラクティスとしてはより柔軟かつ高度なアクセスコントロールのある`iam`method を推奨しています。

このデモでは`iam`method を用いています。

### Demo setup

1.
まずは、`terraform.tfvars.example`を`terraform.tfvars`と変名して、中身を環境に合わせて変更してください。
変更してほしいもの：
* key_name
* aws_region
* availabiliy_zones

```hcl
#-------------------
# Required: こちらを各自の環境に合わせて変更ください
#-------------------

# SSH key name to access EC2 instances. This should already exist in the AWS Region
key_name = "MY_EC2_KEY_NAME"

# AWS region & AZs
aws_region = "ap-northeast-1"
availability_zones = "ap-northeast-1a"

#-----------------------------------------------
# Optional: To overwrite the default settings
#-----------------------------------------------

# All resources will be tagged with this (default is 'vault-agent-demo')
environment_name = "vault-agent-demo"

# Instance size (default is t2.micro)
instance_type = "t2.micro"

# Number of Vault servers to provision (default is 1)
vault_server_count = 1
```

2.
Terraform でプロビジョニングします。AWS のクレデンシャルを環境変数などに追加するのを忘れないでください。

```shell
$ export AWS_ACCESS_KEY_ID=xxxxxxxxxxxx
$ export AWS_SECRET_ACCESS_KEY=xxxxxxxxxxxx

$ terraform init

$ terraform plan

# Output provides the SSH instruction
$ terraform apply -auto-approve
```

3. 以下のようなアウトプットが表示され、EC2 に２つのインスタンスが出来上がっていれば成功です。

```console
Apply complete! Resources: 20 added, 0 changed, 0 destroyed.

Outputs:

endpoints =
Vault Server IP (public):  3.112.22.241
Vault Server IP (private): 10.0.101.67

For example:
   ssh -i masa.pem ubuntu@3.112.22.241

Vault Client IP (public):  13.115.119.242
Vault Client IP (private): 10.0.101.96

For example:
   ssh -i masa.pem ubuntu@13.115.119.242

Vault Client IAM Role ARN: arn:aws:iam::753278538983:role/masa-vault-auth-vault-client-role
```

ここでは 2 つのインスタンスを作成しています。
一つは、Vault server でもう一つは Vault client です。AWS 認証を行なう Vault Server はどこに立ち上げても構いませんが（GCP や Azure でも可）、認証される側の Client は AWS 上のインスタンスやサービスである必用があります（IAM ロールが付随している必用があるため）。

これでデモのセットアップは完了です。

### Vault server のセットアップ

まず、上記アウトプットに表示される Vault server へ ssh で入ります。
そして Vault が立ち上がっているか確認してください。

```console
$ ssh ubuntu@3.112.22.241
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-1054-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Dec 13 02:12:26 UTC 2019

  System load:  0.0               Processes:           89
  Usage of /:   21.0% of 7.69GB   Users logged in:     0
  Memory usage: 28%               IP address for eth0: 10.0.101.67
  Swap usage:   0%


39 packages can be updated.
15 updates are security updates.


Last login: Fri Dec 13 02:07:30 2019 from 126.140.246.218
-bash: warning: setlocale: LC_ALL: cannot change locale (ja_JP.UTF-8)
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

ubuntu@ip-10-0-101-67:~$ which vault
/usr/local/bin/vault
ubuntu@ip-10-0-101-67:~$ vault version
Vault v1.3.0
ubuntu@ip-10-0-101-67:~$ vault status
Key                      Value
---                      -----
Recovery Seal Type       awskms
Initialized              false
Sealed                   true
Total Recovery Shares    0
Threshold                0
Unseal Progress          0/0
Unseal Nonce             n/a
Version                  n/a
HA Enabled               true
ubuntu@ip-10-0-101-67:~$
```

`vault status`コマンドでエラーがでなければ Vault は正常に起動しています。ただ、この状態｀Initialized｀が False であり、`Sealed`は true になっています。つまり、Vault は起動しているが、まだ初期化がされておらず、Seal 状態であるということです。

それでは、次に Vault の初期化を行います。通常の Vault では、初期化をすると Shamir の分散鍵が生成され、それを用いて Unseal します。このデモでは AWS の KMS を用いて**Auto unseal**を行います。Auto Unseal の設定方法は、Server 上の`/etc/vault.d/vault.hcl`を参照ください。

```console
ubuntu@ip-10-0-101-67:~$ vault operator init
Recovery Key 1: 2bxJ0k7+lpoK8o6MAj7ebecIzh9V5d2n9L0GfWyUJjmn
Recovery Key 2: ElH9q/dkglVjFG8mfIZbriM8zbo1C1/JWH12j1R1L45j
Recovery Key 3: c9THb228rV++VUCTkyDjMUw0IG1LyKiaUa3ZmJzyq9oM
Recovery Key 4: EdhT6w6QKGCxtmuU8HSFbcSA/FXYYSHJ//fRF8UiD2+E
Recovery Key 5: s0APWYiXE6KMadHbwCbBWuTzL8CCUa5WnZOW5obGjM6k

Initial Root Token: s.Vfj4S1Wx5bFY5xms5eF751pr

Success! Vault is initialized

Recovery key initialized with 5 key shares and a key threshold of 3. Please
securely distribute the key shares printed above.
```

ここで表示される**Initial Root Token**の値を必ずメモしてください。次に Vault の状態を確認します。

```console
ubuntu@ip-10-0-101-67:~$ vault status
Key                      Value
---                      -----
Recovery Seal Type       shamir
Initialized              true
Sealed                   false
Total Recovery Shares    5
Threshold                3
Version                  1.3.0
Cluster Name             vault-cluster-918e85f6
Cluster ID               b135ecb7-0328-a361-8781-d9db57a876b5
HA Enabled               true
HA Cluster               https://10.0.101.67:8201
HA Mode                  active
ubuntu@ip-10-0-101-67:~$
```

`vault operator init`で初期化されると、**Auto unseal**のおかげで自動的に Vault が Unseal 状態になることが確認できます。(*Sealed = false*)

Vault の Backend storage には Consul が設定されています。

```console
ubuntu@ip-10-0-101-67:~$ consul members
Node            Address           Status  Type    Build  Protocol  DC   Segment
ip-10-0-101-67  10.0.101.67:8301  alive   server  1.6.2  2         dc1  <all>
ip-10-0-101-96  10.0.101.96:8301  alive   client  1.6.2  2         dc1  <default>
ubuntu@ip-10-0-101-67:~$
```

consul server が storage として使われ、それとは別に Vault client 側でも consul が client としてクラスタが構築されています。

次にデモ用にシークレットエンジンと Auth method を設定します。
ホームディレクトリにある`aws_auth.sh`を見てください。

```shell
vault secrets enable -path="secret" kv
vault kv put secret/myapp/config ttl='30s' username='appuser' password='suP3rsec(et!'

echo "path \"secret/myapp/*\" {
    capabilities = [\"read\", \"list\"]
}" | vault policy write myapp -

vault auth enable aws
vault write -force auth/aws/config/client

vault write auth/aws/role/dev-role-iam auth_type=iam bound_iam_principal_arn="arn:aws:iam::753278538983:role/masa-vault-auth-vault-client-role" policies=myapp ttl=24h
```

このスクリプトでは、Vault の K/V シークレットエンジンをマウントし、`secret/myapp/config`にシークレット情報を書き込んでいます。そして、そのシークレットにだけアクセス可能な**policy**を作成します。
さらに、AWS auth method の認証も設定しています。Vault 上に`dev-role-iam`という Role を作成し、ここで指定した IAM ロールの Client に対して、作成した`myapp`という policy を付与します。

このデモでは、Vault server に紐付けられた IAM ロール（この例では、_*arn:aws:iam::753278538983:role/masa-vault-auth-vault-client-role*)を用いて AWS auth method を設定しています。もし別の IAM ロールや IAM ユーザーの権限で認証を行いたい場合は、以下のように個別に設定することも可能です。

```console
$ vault write auth/aws/config/client secret_key=vCtSM8ZUEQ3mOFVlYPBQkf2sO6F/W7a5TVzrl3Oj access_key=VKIAJBRHKH6EVTTNXDHA
```

またその場合、認証用の IAM ポリシーは最低限以下の権限を与えてください。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "iam:GetInstanceProfile",
        "iam:GetUser",
        "iam:GetRole"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["sts:AssumeRole"],
      "Resource": [
        "arn:aws:iam::<AccountId>:role/<VaultRole>"
      ]
    }
  ]
}
```

それでは、スクリプトを実行してみます。Vault コマンドの実行には、まず権限のある Token を用いて login する必要があります。上記の`vault operator init`の際に作成された**Initial Root Token**でログインした上でスクリプトを実行します。

```console
ubuntu@ip-10-0-101-67:~$ vault login s.Vfj4S1Wx5bFY5xms5eF751pr
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.Vfj4S1Wx5bFY5xms5eF751pr
token_accessor       dqYhASeH7qdTh0acArOE8Cgu
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]

ubuntu@ip-10-0-101-67:~$ ./aws_auth.sh
Success! Enabled the kv secrets engine at: secret/
Success! Data written to: secret/myapp/config
Success! Uploaded policy: myapp
Success! Enabled aws auth method at: aws/
Success! Data written to: auth/aws/config/client
Success! Data written to: auth/aws/role/dev-role-iam
ubuntu@ip-10-0-101-67:~$
```

念の為、シークレットがちゃんと書き込まれたか確認します。Root トークンでログインしているので、問題なく読み出しはできるはずです。

```console
ubuntu@ip-10-0-101-67:~$ vault read secret/myapp/config
Key                 Value
---                 -----
refresh_interval    30s
password            suP3rsec(et!
ttl                 30s
username            appuser
ubuntu@ip-10-0-101-67:~$
```

これで Vault server 側の設定は終わりです。

### Vault client のセットアップ

それでは、Vault client 側から AWS 認証で Vault にアクセスし、シークレットの読み出しができるか確認してみましょう。

まず、Vault client に ssh でログインします。もし、Vault client の IP アドレスが分からなくなった場合は、`terraform output`コマンドで確認してください。

```console
$ terraform output
endpoints =
Vault Server IP (public):  3.112.22.241
Vault Server IP (private): 10.0.101.67

For example:
   ssh -i masa.pem ubuntu@3.112.22.241

Vault Client IP (public):  13.115.119.242
Vault Client IP (private): 10.0.101.96

For example:
   ssh -i masa.pem ubuntu@13.115.119.242

Vault Client IAM Role ARN: arn:aws:iam::753646501470:role/masa-vault-auth-vault-client-role
```

ログインします。

```console
$ ssh ubuntu@13.115.119.242
The authenticity of host '13.115.119.242 (13.115.119.242)' can't be established.
ECDSA key fingerprint is SHA256:UYqchHgw3mg1x9QEGz1OY/eyD00Not8UI5Ptr2H1lVc.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '13.115.119.242' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-1054-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Dec 13 02:54:11 UTC 2019

  System load:  0.0               Processes:           88
  Usage of /:   21.0% of 7.69GB   Users logged in:     0
  Memory usage: 18%               IP address for eth0: 10.0.101.96
  Swap usage:   0%

36 packages can be updated.
15 updates are security updates.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

-bash: warning: setlocale: LC_ALL: cannot change locale (ja_JP.UTF-8)
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

ubuntu@ip-10-0-101-96:~$
```

`vault status`コマンドを叩いて、Vault server つながっているか確認します。ちなみに Vault server は**VAULT_ADDR**という環境変数で指定されています。

```console
ubuntu@ip-10-0-101-96:~$ vault status
Key                      Value
---                      -----
Recovery Seal Type       shamir
Initialized              true
Sealed                   false
Total Recovery Shares    5
Threshold                3
Version                  1.3.0
Cluster Name             vault-cluster-918e85f6
Cluster ID               b135ecb7-0328-a361-8781-d9db57a876b5
HA Enabled               true
HA Cluster               https://10.0.101.67:8201
HA Mode                  active

ubuntu@ip-10-0-101-96:~$ echo $VAULT_ADDR
http://10.0.101.67:8200
```

この状態でシークレットが読み出せるか試してみます。

```console
ubuntu@ip-10-0-101-96:~$ vault read secret/myapp/config
Error reading secret/myapp/config: Error making API request.

URL: GET http://10.0.101.67:8200/v1/secret/myapp/config
Code: 400. Errors:

* missing client token
```

まだ認証をしていないので、Token が無くエラーになります。
それでは、認証をしてみます。認証は`vault login`コマンドを使用します。

`vault login -method=aws role=dev-role-iam`

`-method=aws`で AWS 認証を行うことを指定します。
｀role=dev-role-iam`で Vault 上のどの Role の Token を取得するか指定します。それでは実行してみましょう。

```console
ubuntu@ip-10-0-101-96:~$ vault login -method=aws role=dev-role-iam
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                                Value
---                                -----
token                              s.3BiCdXIBpmRf68iFi1wXnj6i
token_accessor                     7zJFSroWIDCNeDvfZhphhe1o
token_duration                     24h
token_renewable                    true
token_policies                     ["default" "myapp"]
identity_policies                  []
policies                           ["default" "myapp"]
token_meta_canonical_arn           arn:aws:iam::753646501470:role/masa-vault-auth-vault-client-role
token_meta_client_arn              arn:aws:sts::753646501470:assumed-role/masa-vault-auth-vault-client-role/i-0b0810e63d1d081ec
token_meta_inferred_entity_id      n/a
token_meta_inferred_entity_type    n/a
token_meta_account_id              753646501470
token_meta_auth_type               iam
token_meta_client_user_id          AROA266GU7ZPL673XWQ72
token_meta_inferred_aws_region     n/a
token_meta_role_id                 32a51eb5-6448-4222-c5e0-400709344741
ubuntu@ip-10-0-101-96:~$
```

認証が成功し、`token                              s.3BiCdXIBpmRf68iFi1wXnj6i`が返ってきました。

ここで再度、シークレットの読み出しをしてみます。

```console
ubuntu@ip-10-0-101-96:~$ vault read secret/myapp/config
Key                 Value
---                 -----
refresh_interval    30s
password            suP3rsec(et!
ttl                 30s
username            appuser
```

今回は読み出しに成功しました。

### Takeaways

デモで実行したとおり、AWS 認証を使うと AWS 上のサービスやインスタンスで使用される IAM ロールを用いて簡単に Vault にアクセスすることができます。
これにより AWS 上で動くインスタンやサービスは、アプリケーション内に認証用のシークレットを保管する必要がなくなり、また Vault 認証用のメカニズムも非常に簡単に導入することができます。

---

## AWS のシークレットエンジンを試す

AWS シークレットエンジンでは IAM ポリシーの定義に基づいた AWS のキーを動的に発行することが可能です。AWS のキー発行のワークフローをシンプルにし、TTL などを設定することでよりセキュアに利用できます。

サポートしているクレデンシャルタイプは下記の三つです。

* IAM user (Access Key & Secret Key)
* Assumed Role
* Federation Token

### IAM ユーザの動的発行

まずシークレットエンジンを enable にします。

```shell
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault secrets enable aws
```

次に Vault が AWS の API を実行するために必要なキーを登録します。

```shell
$ vault write aws/config/root \
    access_key=************ \
    secret_key=************ \
    region=ap-northeast-1
```

`access_key`, `secret_key`, `region`はご自身の環境に合わせたものに書き換えてください。ここでは必ずしも AWS の Aadmin ユーザを登録する必要はなく、ロールやユーザを発行できるユーザであれば大丈夫です。

次にロールを登録します。このロールが Vault から払い出されるユーザの権限と紐付きます。ロールは複数登録することが可能です。今回はまずは`credential_type`に`iam_user`を指定しています。

```shell
$ vault write aws/roles/my-role \
    credential_type=iam_user \
    policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::*"
    }
  ]
}
EOF
```

別端末を開いて`watch`コマンドでユーザのリストを監視します。

```console
$ watch -n 1 aws iam list-users

{
    "Users": [
        {
            "UserName": "tykaburagi",
            "Path": "/",
            "CreateDate": "2019-06-12T07:13:45Z",
            "UserId": "****************",
            "Arn": "****************"
        }
    ]
}
```

>aws cli にログイン出来ていない場合、以下のコマンドでログインしてください。
>
>```console
>$ aws configure
>AWS Access Key ID [****************]: ****************
>AWS Secret Access Key [****************]: ****************
>Default region name [ap-northeast-1]:
>Default output format [json]:
>```

>watch が入っていない場合、以下のコマンドで監視してください。
>```shell
>while true; do aws iam list-users; echo; sleep 1;done
>```
>Windows などで実行できない場合は手動で実行して下さい。

ロールを使って AWS のキーを発行してみましょう。

```console
$ vault read aws/creds/my-role

Key                Value
---                -----
lease_id           aws/creds/my-role/f3e92392-7d9c-09c8-c921-575d62fe80d8
lease_duration     768h
lease_renewable    true
access_key         ************
secret_key         ************
security_token     <nil>
```

この`watch`の出力結果を見るとユーザが増えていることがわかります。`lease_id`はあとで使うのでメモしておいてください。

```json
{
    "Users": [
        {
            "UserName": "tykaburagi",
            "Path": "/",
            "CreateDate": "2019-06-12T07:13:45Z",
            "UserId": "****************",
            "Arn": "****************"
        },
        {
		    "UserName": "vault-root-my-role-1566109640-4907",
		    "Path": "/",
		    "CreateDate": "2019-08-18T06:27:24Z",
		    "UserId": "AIDAZLVKZYEN6HOTBA74D",
		    "Arn": "arn:aws:iam::643529556251:user/vault-root-my-role-1566109640-4907"
        }
    ]
}
```

このユーザを使って動作を確認してみましょう。

```console
$ aws configure

AWS Access Key ID [****************62E7]: ****************
AWS Secret Access Key [****************WF35]: ****************
Default region name [ap-northeast-1]:
Default output format [json]:
```

Vault から払い出されたユーザのシークレットを入力して下さい。新しい端末を立ち上げて以下のコマンドを実行します。

```console
$ aws ec2 describe-instances

An error occurred (UnauthorizedOperation) when calling the DescribeInstances operation: You are not authorized to perform this operation.

$ aws s3 ls
2019-08-16 21:41:34 github-image-tkaburagi
2019-05-26 23:31:14 vault-enterprise-tkaburagi
2019-03-08 21:37:21 web-terraform-state-tykaburagi
```

Role に設定した通り S3 に対する操作のみ可能なことがわかります。

### Revoke を試す

aws cli のユーザを元のユーザに切り替えておきます。

```console
$ aws configure

AWS Access Key ID [****************62E7]: ****************
AWS Secret Access Key [****************WF35]: ****************
Default region name [ap-northeast-1]:
Default output format [json]:

$ watch -n 1 aws iam list-users

{
    "Users": [
        {
            "UserName": "tykaburagi",
            "Path": "/",
            "CreateDate": "2019-06-12T07:13:45Z",
            "UserId": "****************",
            "Arn": "****************"
        },
        {
		    "UserName": "vault-root-my-role-1566109640-4907",
		    "Path": "/",
		    "CreateDate": "2019-08-18T06:27:24Z",
		    "UserId": "AIDAZLVKZYEN6HOTBA74D",
		    "Arn": "arn:aws:iam::643529556251:user/vault-root-my-role-1566109640-4907"
        }
    ]
}
```

シンプルな手順でユーザが発行できることがわかりましたが、次は Revoke(破棄)を試してみます。Revoke にはマニュアルと自動の 2 通りの方法があります。

まずはマニュアルでの実行手順です。`vault read aws/creds/my-role`を実行した際に発行された`lease_id`をコピーしてください。

```shell
$ vault lease revoke aws/creds/my-role/<LEASE_ID>
```

`watch`の実行結果を見るとユーザが削除されているでしょう。

```json
{
    "Users": [
        {
            "UserName": "tykaburagi",
            "Path": "/",
            "CreateDate": "2019-06-12T07:13:45Z",
            "UserId": "****************",
            "Arn": "****************"
        }
    ]
}
```

次に自動 Revoke です。デフォルトでは TTL が`765h`になっています。これは数分にしてみましょう。

```shell
vault write aws/config/lease lease=2m lease_max=10m
```

```console
$ vault read aws/config/lease

Key          Value
---          -----
lease        2m0s
lease_max    10m0s
```

それではこの状態でユーザを発行します。

```console
$ vault read aws/creds/my-role

Key                Value
---                -----
lease_id           aws/creds/my-role/agnda2uyVWKso4E3HoWlPqY8
lease_duration     2m
lease_renewable    true
access_key         ****************
secret_key         ****************
security_token     <nil>
```

`watch`の実行結果を見るとユーザが増えています。今度は 2 分後にこのユーザは自動で削除されます。

```json
{
    "Users": [
        {
            "UserName": "tykaburagi",
            "Path": "/",
            "CreateDate": "2019-06-12T07:13:45Z",
            "UserId": "****************",
            "Arn": "****************"
        },
        {
            "UserName": "vault-root-my-role-1566111715-4258",
            "Path": "/",
            "CreateDate": "2019-08-18T07:01:56Z",
            "UserId": "AIDAZLVKZYENUIPMHTSYJ",
            "Arn": "arn:aws:iam::643529556251:user/vault-root-my-role-1566111715-4258"
        }
    ]
}
```

2 分後、再度見てみるとユーザが削除されていることがわかるでしょう。

```json
{
    "Users": [
        {
            "UserName": "tykaburagi",
            "Path": "/",
            "CreateDate": "2019-06-12T07:13:45Z",
            "UserId": "****************",
            "Arn": "****************"
        }
    ]
}
```

### 参考リンク
* [AWS Secret Engine](https://www.vaultproject.io/docs/secrets/aws/index.html)
* [AWS Secret Engine API](https://www.vaultproject.io/api/secret/aws/index.html)

---

## Vault を PKI エンジンとして扱う

Vault で扱うことの出来る「シークレット」は多岐に渡ります。今まで扱ってきた Cloud のユーザアカウントやデータベースのパスワード、静的なユーザ名とシークレットなどの他にサーバ証明書も Vault で扱うことが出来ます。

これを使うことで従来多くの時間を割いていた証明書の発行を短いサイクルで行うことが可能です。Vault には`PKI Secret Engine`というシークレットエンジンが用意されており、これを利用することで Vault が認証局として機能して動的に証明書を発行します。

`Root CA`, `Intermediate CA`の両方を扱うことが出来、外部の`Root CA`と連携をし`Intermediate CA`である Vault をインターフェースに発行することも出来ますし、Vault を`Root CA`と`Intermediate CA`両方の役割をさせて`self-singed`な証明書を発行することも出来ます。

ここでは両方の役割を Vault に持たせるパターンを試してみます。

### PKI として設定する

まずは PKI Secret Engine を有効化します。ここでは Vault を Root、Intermediate の両方として扱うため、別のパスでそれぞれ Enable していきます。

```shell
$ vault secrets enable -path="pki_root" pki
$ vault secrets enable -path="pki_intermediate" pki
```

これ以降`pki_root`をルート CA、`pki_intermediate`を中間 CA として扱います。ドメインは仮で`vault-handson.lab`とします。

`/pki/root/generate/:type`のエンドポイントで自己署名のルート CA 証明書とプライベートキーを発行します。

```shell
$ export DOMAIN=vault-handson.lab

$ vault write pki_root/root/generate/internal common_name="${DOMAIN} Root CA" ttl=24h > ca_root.crt.pem
```

以下のコマンドで Root CA が作成されたことが確認できます。

```console
$ curl -s --header "X-Vault-Token: <VAULT_TOKEN>" http://127.0.0.1:8200/v1/pki_root/ca/pem
-----BEGIN CERTIFICATE-----
MIIDrDCCApSgAwIBAgIUQneY8K+YH9M5DOuFVDVxHwM2CG8wDQYJKoZIhvcNAQEL
BQAwJDEiMCAGA1UEAxMZdmF1bHQtaGFuZHNvbi5sYWIgUm9vdCBDQTAeFw0xOTEy
~~~~~
-----END CERTIFICATE-----
```

次に証明書 wp 発行するエンドポイントと証明書失効リストを配信するためのエンドポイントを設定します。

```shell
$ vault write pki_root/config/urls \
issuing_certificates="http://127.0.0.1:8200/v1/pki_root/ca" \
crl_distribution_points="http://127.0.0.1:8200/v1/pki_root/crl"
```

次に Intermediate 側の設定です。まずは CSR(証明書署名リクエスト)の作成です。`intermediate/generate/:type`のエンドポイントです。

```shell
$ vault write -format=json pki_intermediate/intermediate/generate/internal \
common_name="${DOMAIN} Intermediate Authority" ttl="12h" \
| jq -r '.data.csr' > pki_intermediate.csr
```

`pki_intermediate.csr`ファイルを見ると CSR が生成されていることがわかるでしょう。次にこの CSR に対して、先ほど作った Root CA を使ってサインをしていきます。

```shell
$ vault write -format=json pki_root/root/sign-intermediate \
csr=@pki_intermediate.csr \
format=pem_bundle ttl="12h" \
| jq -r '.data.certificate' > ca_intermediate.cert.pem
```

Root CA によって CSR がサインされ、中間 CA としての証明書が発行されました。証明書を検証してみましょう。

``` shell
$ openssl x509 -in ca_intermediate.cert.pem -text -noout
```

`CA Issuers - URI:http://127.0.0.1:8200/v1/pki_root/ca`となっており、Vault の Root CA で発行されたことがわかります。


この証明書を中間 CA にインポートします。

```shell
$ vault write pki_intermediate/intermediate/set-signed \
certificate=@ca_intermediate.cert.pem
```

以下のコマンドで確認してみましょう。

```console
$ curl -s --header "X-Vault-Token: <VAULT_TOKEN>" http://127.0.0.1:8200/v1/pki_intermediate/ca/pem

-----BEGIN CERTIFICATE-----
MIIDxDCCAqygAwIBAgIUY/zc2qyWWEaXZdEhSezI2qTJWoswDQYJKoZIhvcNAQEL
BQAwJDEiMCAGA1UEAxMZdmF1bHQtaGFuZHNvbi5sYWIgUm9vdCBDQTAeFw0xOTEy
~~~~
-----END CERTIFICATE-----
```

### 証明書を発行する

これで Root CA と Intermediate CA の準備が出来ました。最後に証明書を発行していきます。発行する前にロールを定義する必要があります。

ロールとは、発行する証明書に与える権限の論理名のようなイメージです。

* 発行を許可するドメイン
* サブドメインの利用可否
* ワイルドカードドメインの利用可否

などを設定します。下記は`vault-handson.lab`が利用でき、サブドメインを許可する例です。

```shell
$ vault write pki_intermediate/roles/vault-dot-lab \
allowed_domains="vault-handson.lab" allow_subdomains=true max_ttl="12h"
```

最後にこのロールを使って証明書を発行しましょう。

```shell
$ vault write pki_intermediate/issue/vault-dot-lab \
common_name="contents.vault-handson.lab" ttl="24h"
```

`vault-handson.lab`のドメインの`contents`というサブドメインで発行をしています。出力される各シークレットを例えば以下のように

* ca_chain => ca-chain.crt.pem
* private_key => private.key.pem
* certificate => server.crt.pem

保存し、これをサーバ、コンテナやロードバランサにセットすることで証明書として扱うことが出来ます。

例えば AWS にインポートするときは以下のように実行します。

```
aws acm import-certificate \
--certificate file://./server.crt.pem \
--private-key file://./private.key.pem \
--certificate-chain file://./ca-chain.crt.pem
```

最後に certificate の出力結果を`server.crt.pem`として保存をして証明書を検証してみます。

```shell
openssl x509 -in server.cert.pem  -text -noout
```

正しく発行されているでしょう。Vault ではこのように Root CA、Intermediate CA として Vault を扱う、もしくは既存の Root CA と Vault 上の Intermediate CA を連携させて、TTL 付きの証明書を権限に応じて動的に、迅速に発行することが可能です。

### 参考リンク
* [PKI Secret Engine](https://www.vaultproject.io/docs/secrets/pki/index.html)
* [PKI Secret Engine API](https://www.vaultproject.io/api/secret/pki/index.html)
* [PKI Roles](https://www.vaultproject.io/api/secret/pki/index.html#create-update-role)

---

## Transit シークレットエンジンで Vault を Encryption as a Sevice として使う

これまで Vault のシークレット管理の機能を扱ってきましたが、Vault の二つ目のユースケースは`Data Protection`です。その中でも API ドリブンな Encryption の機能を使って Vault を暗号化としてのサービスとして扱う`Encryption as a Service (EaaS)`は非常に多く採用されているユースケースです。

ここではそれを実現する Transit の機能と、実際のアプリを使った利用イメージを扱います。

### Transit を有効化する。

その他のシークレットエンジンと同様、EaaS を利用する際は`Transit`というシークレットエンジンを`enabled`にします。

```console
$ vault secrets enable -path=transit transit
Success! Enabled the transit secrets engine at: transit/

$ vault secrets list
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_e3aa0798    per-token private secret storage
database/     database     database_603dc42e     n/a
identity/     identity     identity_86c0240d     identity store
kv/           kv           kv_20084de2           n/a
sys/          system       system_ae51ee57       system endpoints used for control, policy and debugging
transit/      transit      transit_ec14846c      n/a
```

Transit が有効になりました。Transit には大きく

* 暗号化
* 復号化
* キーローテーション

の機能があります。

### 初めての暗号化と復号化

早速データを暗号化してみましょう。Transit で暗号化するためには Plaintext は base64 で暗号化する必要があります。macOS であればターミナルから実行可能ですし、[こちら](https://kujirahand.com/web-tools/Base64.php)の Web サイトでもエンコードができます。

`myimportantpassword`というパスワードを暗号化してみます。

```console
$ base64 <<< "myimportantpassword"
bXlpbXBvcnRhbnRwYXNzd29yZAo=
```

これを`transit/encrypt/`のエンドポイントを使って暗号化キーを作り、暗号化します。`my-encrypt-key`は暗号化キーの名前です。

```console
$ vault write transit/encrypt/my-encrypt-key plaintext=bXlpbXBvcnRhbnRwYXNzd29yZAo=
Key           Value
---           -----
ciphertext    vault:v1:WputNlwLdegpFARr+OL8Az/UmDRCWsVL3ytVf/AUc9tFHt4YD1NOnfd4iSocUfG5
```

```shell
$ export CTEXT_V1=vault:v1:WputNlwLdegpFARr+OL8Az/UmDRCWsVL3ytVf/AUc9tFHt4YD1NOnfd4iSocUfG5
```
この暗号化の機能は base64 にさえ変換してしまえば画像など様々な形式のデータを暗号化することができます。

次に復号化をしてみます。復号化は`transit/decrypt/`のエンドポイントを使います。

```console
$ vault write transit/decrypt/my-encrypt-key ciphertext=$CTEXT_V1
Key          Value
---          -----
---          -----
plaintext    bXlpbXBvcnRhbnRwYXNzd29yZAo=
```

`plaintext`として Base64 のコードが表示されました。これをデコードしてパスワードを取り出してみます。

```console
$ base64 --decode <<< "bXlpbXBvcnRhbnRwYXNzd29yZAo="
myimportantpassword
```

無事に復号化できました。

暗号化キーは様々なアルゴリズムをサポートしており、`type`で指定可能です。

* aes256-gcm96 (Default)
* chacha20-poly1305 
* ed25519 
* ecdsa-p256 
* rsa-2048 
* rsa-4096

#### キーローテーション

暗号化、復号化のキーはどれだけ強力なアルゴリズムを使っても時間をかければ必ず解読出来てしまいます。そのため環境を最大限にセキュアに保つためにはキー自体をローテーションさせ、長く使わず短いサイクルでリニューアルすることが大切です。

Transit では`transit/keys/<KEYNAME>/rotate`と`transit/rewrap/<KEYNMAME>`というエンドポイントで簡単に実現できます。

`rotate`はキーの更新、`rewarp`は古いデータを新しいキーで再暗号化するためのエンドポイントです。

```console
$ vault write -f transit/keys/my-encrypt-key/rotate
$ vault read transit/keys/my-encrypt-key
Key                       Value
---                       -----
allow_plaintext_backup    false
deletion_allowed          false
derived                   false
exportable                false
keys                      map[2:1563181079 1:1563181077]
latest_version            2
min_available_version     0
min_decryption_version    1
min_encryption_version    0
name                      my-encrypt-key
supports_decryption       true
supports_derivation       true
supports_encryption       true
supports_signing          false
type                      aes256-gcm96
```

バージョンが 2 に変わりました。`min_decryption_version`はこのデータが復号化できる最小のキーのバージョンを示しています。まずはこの状態で新しいデータを暗号化してみましょう。

```console
$ base64 <<< "myimportantpassword-v2"
bXlpbXBvcnRhbnRwYXNzd29yZC12Mgo=

$ vault write transit/encrypt/my-encrypt-key plaintext=bXlpbXBvcnRhbnRwYXNzd29yZC12Mgo=

Key           Value
---           -----
ciphertext    vault:v2:93WEsl7Q7UM/eWHGZP+N9PmOEqXPYpnpVeBx21APu7pT1MOCJElJ7AkbiNgdr0gVOALw
```

```shell
$ export CTEXT_V2=vault:v2:93WEsl7Q7UM/eWHGZP+N9PmOEqXPYpnpVeBx21APu7pT1MOCJElJ7AkbiNgdr0gVOALw
```

新しいデータは v2 のキーで暗号化復号化され、それ以前のデータは古いキーで復号化されます。v1 と v2 で暗号化したデータをそれぞれ復号化してみます。

```console
$ vault write transit/decrypt/my-encrypt-key ciphertext=$CTEXT_V1

Key          Value
---          -----
plaintext    bXlpbXBvcnRhbnRwYXNzd29yZAo=

$ vault write transit/decrypt/my-encrypt-key ciphertext=$CTEXT_V2

Key          Value
---          -----
plaintext    bXlpbXBvcnRhbnRwYXNzd29yZC12Mgo=
```

V1, V2 のデータ共に複合化可能です。この状態でいずれ v2 の新しいキーに全てのデータを移行したいです。そのためには`rewrap`という操作を行い、古いデータの更新(再暗号化)を行います。`ciphertext`には v1 のデータを入れてください。

```console
$ vault write transit/rewrap/my-encrypt-key ciphertext=$CTEXT_V1

Key           Value
---           -----
ciphertext    vault:v2:pymUK9PJQ3KYXSw7uNj/lcTMOwfNav2t3pP52jAuQWQ6bTHNd9n/3tX4Zdc/IPLt
```
これで v1 で暗号化したデータを v2 で暗号化しました。次に、`min_decryption_version`を更新し v1 のキーを無効化し、利用できないようにします。

```shell
export CTEXT_V1_V2=vault:v2:pymUK9PJQ3KYXSw7uNj/lcTMOwfNav2t3pP52jAuQWQ6bTHNd9n/3tX4Zdc/IPLt
```

```console
$ vault write  transit/keys/my-encrypt-key/config min_decryption_version=2

Success! Data written to: transit/keys/my-encrypt-key/config

$ vault write transit/decrypt/my-encrypt-key ciphertext=$CTEXT_V1_V2

Key          Value
---          -----
plaintext    bXlpbXBvcnRhbnRwYXNzd29yZAo=

$ vault write transit/decrypt/my-encrypt-key ciphertext=$CTEXT_V1
Error writing data to transit/decrypt/my-encrypt-key: Error making API request.

URL: PUT http://127.0.0.1:8200/v1/transit/decrypt/my-encrypt-key
Code: 400. Errors:

* ciphertext or signature version is disallowed by policy (too old)
```

v1 のデータは復号化出来なくなり、v1 のキーが無効になっていることがわかります。

### Convergent 暗号化を試す

次に Convergent(収束)暗号化を試してみます。Convergent は一般的な暗号化の手法で、特定のキーを利用し同一のプレインテキストで暗号化されたものは毎回同一の暗号文を返すというものです。

Vault の暗号化はデフォルトでは同じ平文であっても毎回別の暗号文が生成されます。ところが暗号化したいが重複データを避けたい場合や暗号データを検索したいような場合、同じ平文は同じ暗号文で返して欲しい際があります。

Vault では暗号化キーを生成する際にこの Convergent 暗号化のパラメータを指定することで実現可能です。

まずは新しいキーを生成してみましょう。

```shell
$ vault write transit/keys/convergent-key type="chacha20-poly1305" convergent_encryption=true derived=true
```

Convergent に対応しているタイプのアルゴリズムを指定しています。この他にもあるので[こちら](https://www.vaultproject.io/api/secret/transit/index.html#type)で確認してみてください。

```console
$ vault read transit/keys/convergent-key
Key                              Value
---                              -----
allow_plaintext_backup           false
convergent_encryption            true
convergent_encryption_version    -1
deletion_allowed                 false
derived                          true
exportable                       false
kdf                              hkdf_sha256
keys                             map[1:1579319629]
latest_version                   1
min_available_version            0
min_decryption_version           1
min_encryption_version           0
name                             convergent-key
supports_decryption              true
supports_derivation              true
supports_encryption              true
supports_signing                 false
type                             chacha20-poly1305
```

Convernt が true になっているキーが生成したことがわかります。`derived`は`Key Derivation Function`を有効化するためのパラメータで Vault では Convergent を有効化する際に必須となります。

これによってクライアントが同一の暗号文を保持したとしても`context`パラメータを指定しないと復号化が不可能となり、より安全にデータを扱うことができます。

```console
$ vault write transit/encrypt/convergent-key plaintext=$(base64 <<< "myimportantpassword") context=$(base64 <<< "c2FtcGxxxx9udGV4dA")

Key           Value
---           -----
ciphertext    vault:v1:NuH3WBB956hNZOnPYZqo5lb86bZ5LN1BTKlmuZ78ZGzB2HYdcl9iAbh5hdxCC/1k
```

`Value`で出力される暗号文をコピーし、復号化してみましょう。

```console
$ base64 --decode <<< $(vault write -format=json transit/decrypt/convergent-key ciphertext="vault:v1:NuH3WBB956hNZOnPYZqo5lb86bZ5LN1BTKlmuZ78ZGzB2HYdcl9iAbh5hdxCC/1k" context=$(base64 <<< "c2FtcGxxxx9udGV4dA") | jq -r '.data.plaintext')

myimportantpassword
```

復号化出来ました。試しに`context`に別の値を入れてみましょう。

```console
$ base64 --decode <<< $(vault write -format=json transit/decrypt/convergent-key ciphertext="vault:v1:NuH3WBB956hNZOnPYZqo5lb86bZ5LN1BTKlmuZ78ZGzB2HYdcl9iAbh5hdxCC/1k" context=$(base64 <<< "samplecontext") | jq -r '.data.plaintext')

Error writing data to transit/decrypt/convergent-key: Error making API request.

URL: PUT http://127.0.0.1:8200/v1/transit/decrypt/convergent-key
Code: 400. Errors:

* invalid ciphertext: unable to decrypt
```

エラーになり、復号化出来ないはずです。このようにキーへのアクセス権や暗号化文を保持していても`context`を持っていないと復号化することが出来ず、データを守ることが出来ます。

最後に試しに Convergent が設定されているか再度同じ平文を暗号化してみます。

```console
$ vault write transit/encrypt/convergent-key plaintext=$(base64 <<< "myimportantpassword") context=$(base64 <<< "c2FtcGxxxx9udGV4dA")

Key           Value
---           -----
ciphertext    vault:v1:NuH3WBB956hNZOnPYZqo5lb86bZ5LN1BTKlmuZ78ZGzB2HYdcl9iAbh5hdxCC/1k
```

先ほどと同じ暗号文が返されるはずです。

余裕のある方は以下の内容を前の手順を振り返りながら試してみてください。

* `convergent-key`で別の平文を使って暗号化
* `convergent-key`を`rotate`して同じ平文`myimportantpassword`を暗号化
* 別の Convergent を生成して同じ平文`myimportantpassword`を暗号化

### 実際のアプリで使ってみる

次に利用イメージをもう少し理解しやすくするため、Spring のアプリで Transit を利用してみます。アプリのレポジトリを clone し起動します。

まずデータベースにテーブルを作ります。MySQL にログインし、以下のコマンドを発行します。

この手順を完了するには[Java 12](https://www.oracle.com/technetwork/java/javase/downloads/jdk12-downloads-5295953.html)が必要です。

```mysql
use handson;
create table users (id varchar(50), username varchar(50), password varchar(200), email varchar(50), address varchar(50), creditcard varchar(200));
```

次にロールの設定をします。ロールは二つ作成します。

* MySQL データベースとやりとりしてデータの select, insert をするための`database/*`配下のロール
* Vault とやりとりして Transit で暗号化復号化をするための`auth/*`配下のロール

まずはデータベース側です。コンフィグをアップデートし、`role-demoapp`というロールを許可します。

```shell
$ vault write database/config/mysql-handson-db \
  plugin_name=mysql-legacy-database-plugin \
  connection_url="{{username}}:{{password}}@tcp(127.0.0.1:3306)/" \
  allowed_roles="role-handson","role-handson-2","role-handson-3","role-demoapp" \
  username="root" \
  password="rooooot"
```

ロールを作成します。`handson.users`のテーブルに対して`SELECT`, `INSERT`の権限のあるロールです。

```shell
$ vault write database/roles/role-demoapp \
  db_name=mysql-handson-db \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT,INSERT,UPDATE ON handson.users TO '{{name}}'@'%';" \
  default_ttl="5h" \
  max_ttl="5h"
```

動作を確認しておきましょう。ここで生成したユーザ名パスワードは利用せず、実際にこの操作はアプリから実施することになります。

```console
$ vault read database/creds/role-demoapp

Key                Value
---                -----
lease_id           database/creds/role-demoapp/GwOQKPDCIJS1K1Z626RdrQlW
lease_duration     5h
lease_renewable    true
password           A1a-4VU2FVBp5HdIJGvz
username           v-role-FWRN0zpOp
```

次に Vault 認証用のロールです。ここで作るポリシーは`AppRole`の認証で付与されるトークンの権限となります。以下のようにポリシーの定義ファイルを作成してください。

```hcl
$ cat > policy-vault.hcl <<EOF
# Enable transit secrets engine
path "sys/mounts/transit" {
  capabilities = [ "create", "read", "update", "delete", "list" ]
}

# To read enabled secrets engines
path "sys/mounts" {
  capabilities = [ "read" ]
}

# Manage the transit secrets engine
path "transit/*" {
  capabilities = [ "create", "read", "update", "delete", "list" ]
}
EOF
```

AppRole が有効になっていない方は下記のコマンドで有効化しましょう。
```shell
$ vault auth enable approle
```

```console
$ vault policy write vault-policy policy-vault.hcl
$ vault write auth/approle/role/vault-approle policies=vault-policy period=1h
```

これで準備は完了です。アプリをクローンして、起動してみましょう。`YOUR_ROOT_TOKEN`はご自身の Root Token です。

```console
$ export ROOT_TOKEN=<YOUR_ROOT_TOKEN>
$ git clone https://github.com/tkaburagi/spring-vault-transit-demo
$ cd spring-vault-transit-demo
$ sed "s|VAULT_TOKEN=|VAULT_TOKEN=$ROOT_TOKEN|g" set-env-local.sh > my-set-env-local.sh
$ cat my-set-env-local.sh
$ source my-set-env-local.sh
$ ./mvnw clean package -DskipTests
$ java -jar target/demo-0.0.1-SNAPSHOT.jar
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.4.RELEASE)

2019-07-15 21:50:19.739  INFO 7226 --- [           main] com.example.demo.VaultDemoApplication    : Starting VaultDemoApplication v0.0.1-SNAPSHOT on Takayukis-MacBook-Pro.local with PID 7226 (/Users/kabu/hashicorp/intellij/springboot-vault-transit/target/demo-0.0.1-SNAPSHOT.jar started by kabu in /Users/kabu/hashicorp/intellij/springboot-vault-transit)
2019-07-15 21:50:19.741  INFO 7226 --- [           main] com.example.demo.VaultDemoApplication    : No active profile set, falling back to default profiles: default
```

コードの説明は後ほどしますが、このアプリには 4 つのエンドポイントがあります。

まずはパラメータで渡された文字列を暗号化、復号化する単純なエンドポイントです。

```console
$ curl -G http://localhost:8080/api/v1/transit/encrypt -d "ptext=hellotransit" 
vault:v1:Wa87EPlyIe0xaF8R+725/j8XRB18cbM8PfGLjM0jmlRCaVD0FmGJQg==

$ curl -G http://localhost:8080/api/v1/transit/decrypt --data-urlencode "ctext=vault:v1:9ZNILrEWKswi+lHhzGRXHfN0sY+idGEIHQZ4IVeLRNey/pNLibKZ6Q=="
hellotransit                            
```

次にデータを暗号化し、データベースにデータを保存するエンドポイントです。`api/v1/encrypt/add-user`に平文でデータを渡すと暗号化され、データベースに暗号刺されたデータが insert されます。

```console
$ curl http://localhost:8080/api/v1/encrypt/add-user -d username="Takayuki Kaburagi" -d password="PqssWOrd" -d address="Yokohama" --data-urlencode creditcard="9999-8888-6666-6666" --data-urlencode email="t.kaburagi@me.com"

{"id":"db0bbb62-fdfd-4e2e-a4db-1e5e32e36761","username":"Takayuki Kaburagi","password":"vault:v1:aRtAJK+ED8ap2vM5f9ba8eL0VvnjD+Akw8ag2eHLYNucXfRx","email":"h.kaburagi@me.com","address":"Yokohama","creditcard":"vault:v1:LYpkecFI4bY6c7I8a3fB47d0oHNf6bPL/6VTc14g+zgEVg47EoRjKWTJeYeaisw="}
```

データベースで確認してみましょう。

```mysql
mysql> select * from users;
+--------------------------------------+-----------------+-----------------------------------------------------------+-------------------+----------+---------------------------------------------------------------------------+
| id                                   | username        | password                                                  | email             | address  | creditcard                                                                |
+--------------------------------------+-----------------+-----------------------------------------------------------+-------------------+----------+---------------------------------------------------------------------------+
| db0bbb62-fdfd-4e2e-a4db-1e5e32e36761 | Takayuki Kaburagi | vault:v1:aRtAJK+ED8ap2vM5f9ba8eL0VvnjD+Akw8ag2eHLYNucXfRx | t.kaburagi@me.com | Yokohama | vault:v1:LYpkecFI4bY6c7I8a3fB47d0oHNf6bPL/6VTc14g+zgEVg47EoRjKWTJeYeaisw= |
+--------------------------------------+-----------------+-----------------------------------------------------------+-------------------+----------+---------------------------------------------------------------------------+
```

暗号されたデータが保存されていることがわかります。次にデータを取り出すためのエンドポイントです。`api/v1/plain/get-use`ではデータをそのまま取り出します。上の`uuid`の値をメモしてください。

```console
$ curl -G "http://localhost:8080/api/v1/non-decrypt/get-user" -d uuid=d87b7a21-0a33-4e64-a05d-60065eed71a9 | jq

{
  "id": "db0bbb62-fdfd-4e2e-a4db-1e5e32e36761",
  "username": "Hiroki Kaburagi",
  "password": "vault:v1:aRtAJK+ED8ap2vM5f9ba8eL0VvnjD+Akw8ag2eHLYNucXfRx",
  "email": "h.kaburagi@me.com",
  "address": "Yokohama",
  "creditcard": "vault:v1:LYpkecFI4bY6c7I8a3fB47d0oHNf6bPL/6VTc14g+zgEVg47EoRjKWTJeYeaisw="
```

この場合、データは暗号化されたままなのでアプリ側で復号の処理を実装する必要があります。Vault の場合、それを Vault に委託することが可能です。`api/v1/decrypt/get-user`を使います。

```console
$ curl -G "http://localhost:8080/api/v1/decrypt/get-user" -d uuid=db0bbb62-fdfd-4e2e-a4db-1e5e32e36761 | jq

{
  "id": "db0bbb62-fdfd-4e2e-a4db-1e5e32e36761",
  "username": "Takayuki Kaburagi",
  "password": "PqssWOrd",
  "email": "t.kaburagi@me.com",
  "address": "Yokohama",
  "creditcard": "9999-8888-6666-6666"
  ```

Vault に復号化し、アプリのデータとして利用することが出来るようになりました。このように Vault ではシークレット管理だけでなく暗号化の処理をサービスとして扱えるようにするような使い方をすることができます。


最後にキーのローテーションと Rewrap をしてみます。`get-keys`のエンドポイントでアプリからキーの情報が取り出せるようになっています。

```console
$ curl -G http://localhost:8080/api/v1/get-keys | jq

{
  "name": [
    "springdemo"
  ],
  "type": "aes256-gcm96",
  "latest_version": 1,
  "min_decrypt_version": 1
}
```

まずキーをローテーションします。

```console
$ vault write -f transit/keys/springdemo/rotate
$ curl -G http://localhost:8080/api/v1/get-keys | jq

{
  "name": [
    "springdemo"
  ],
  "type": "aes256-gcm96",
  "latest_version": 2,
  "min_decrypt_version": 1
}
```

新しいデータを投入してみましょう。

```shell
curl http://localhost:8080/api/v1/encrypt/add-user -d username="Yusuke Kaburagi" -d password="PqssWOrd" -d address="Tokyo" --data-urlencode creditcard="9999-8888-6666-6666" --data-urlencode email="yusuke@locahost"
```

v1, v2 のデータが両方入っていることがわかります。

```shell
mysql> select * from users;
```

v1 のデータを v2 に Rewrap してみます。このアプリでは`api/v1/rewrap`のエンドポイントで実現しています。

```shell
curl -G http://localhost:8080/api/v1/rewrap -d uuid=<OLD DATA'S UUID> | jq
```

データを見ると v2 に更新されているでしょう。

```shell
mysql> select * from users;
```

あとは同様に`min_decryption_version`を bump すれば完了です。

```shell
$ vault write  transit/keys/springdemo/config min_decryption_version=2
$ curl -G http://localhost:8080/api/v1/get-keys | jq

{
  "name": [
    "springdemo"
  ],
  "type": "aes256-gcm96",
  "latest_version": 2,
  "min_decrypt_version": 2
```

### 参考リンク
* [Transit](https://www.vaultproject.io/docs/secrets/transit/index.html)
* [API Document](https://www.vaultproject.io/api/secret/transit/index.html)
* [Spring Cloud Vault](https://cloud.spring.io/spring-cloud-vault/)
* [Spring Vault](https://projects.spring.io/spring-vault/)

---

## Transform Secret Engine を試す

`Transform Secret Engine`は`Format Preserving Encryption(FPE)`と`Masking`を実現するためのシークレットエンジンです。

`Transit Secret Engine`ではランダムな値を用いて暗号化を実現しましたが、

`FPE`とは、例えば`1234-5678-8765-4321`のような入力値に対して、`ASKT-THN3-KWt9-HHOA`のようにフォーマットを維持したまま暗号化する機能です。

`Masking`とは、`1234-5678-8765-4321`のような入力値に対して、`****-****-****-****`のように値をマスキングする機能です。

`FPE`を利用することで、データサイズを変更やデータベースのスキーマの変更することなく暗号化を実現することが可能となります。

`Transform Secret Engine`は Enterprise 版のみ有効な機能です。利用の際は[トライアルのライセンス](https://www.hashicorp.com/products/vault/trial/)や Entperprise の正式なライセンスで機能をアクティベーションする必要があります。

ライセンスのセットの仕方は[こちら](https://www.vaultproject.io/api-docs/system/license)を参考にしてみてください。

### Transformation の 4 つのリソース

`Transform Secret Engine`では 4 つのリソースを利用して上記のような機能を実現します。

* `Roles`: Transformation を行うためのロール。暗号化する際のエンドポイントとなり、ACL の設定をする際にも利用される。
* `Alphabets`: 置換される平文、および暗号化された後の暗号文に含まれる UTF-8 の文字列の定義する。
* `Templates`: 実際に入力値を暗号化する際に利用するテンプレート。暗号化で使用する`Alphabets`や暗号する値の`Parttern`(フォーマット)などを指定する。
* `Transformation`: 利用可能な`Roles`, 利用する`Templates`, `Type`などを定義する。

また暗号化のアルゴリズムには`NIST`によって認定されている、`AES-FF3-1`を採用しています。

### FPE を実際に使ってみる

4 つのリソースを意識しながら実際にまずはあらかじめ用意されているパターンで試してみたいと思います。

まずは有効化しましょう。Secret Engine の名前は`transform`です。

```shell
$ vault secrets enable transform
```

`Alphabets`と`Templates`はデフォルトのものが用意されています。各リソースを確認してみましょう。

```console
$ vault list transform/alphabet
Keys
----
builtin/alphalower
builtin/alphanumeric
builtin/alphanumericlower
builtin/alphanumericupper
builtin/alphaupper
builtin/numeric

$ vault list transform/template
Keys
----
builtin/creditcardnumber
builtin/socialsecuritynumber
```

この中`builtin/creditcardnumber`のテンプレートを使ってみます。このテンプレートは入力されたクレジットカード番号をランダムの数字で暗号化するものです。

まず、ロールを定義します。

```shell
$ vault write transform/role/my-transform-role \
transformations=first-transform \
```

この`my-transform-role`が暗号化をするときのエンドポイントの末尾となります。

`builtin/creditcardnumber`を利用した Transformation を作成してみましょう。`transform/transformation/<Transform Name>`がエンドポイントです。

```shell
$ vault write transform/transformation/first-transform \
type=fpe \
template=builtin/creditcardnumber \
allowed_roles=my-transform-role \
tweak_source=internal
```

* `type`は現在は`fpe`, `masking`のどちらかの選択です。
* `tweak_source`は暗号化をする際にツイーク値を持たせるか否かのパラメータです。ここでは`internal`とし、内部的に持たせるものとしています。

作成した`Transformation`を見てみましょう。

```console
$ vault read transform/transformation/first-transform

Key              Value
---              -----
allowed_roles    [my-transform-role]
templates        [builtin/creditcardnumber]
tweak_source     internal
type             fpe
```

さて、これを利用して暗号化する際は`transform/encode/<Role Name>`のエンドポイントを実行し、平文を渡します。

```console
$ vault write transform/encode/my-transform-role value=1234-4321-5678-8765

Key              Value
---              -----
encoded_value    1166-5682-8535-1071
```

以上のようにランダムな数字に暗号化されました。このとき、Vault が特定の値と値をマッピングしているわけではなく、常に`AES-FF3-1`アルゴリズムを利用して暗号化がなされています。

次に複合化してみます。

```console
$ vault write transform/decode/my-transform-role value=1166-5682-8535-1071

Key              Value
---              -----
decoded_value    1234-4321-5678-8765
```

正しい値が取り出せるでしょう。

### 自作の Transformation を利用する

次に自作の`Alphabet`と`Template`を作って暗号化をしてみましょう。ここでは Email アドレスを暗号化する`Transformation`を作ってみます。

まず`Alphabet`を作成します。ここでは置換する文字列と暗号で利用する文字列両方を指定します。

```shell
$ vault write transform/alphabet/localemailaddress \
alphabet=".@0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
```

次に`Template`です。

```shell
$ vault write transform/template/email-template \
type=regex \
pattern='([0-9A-Za-z]{1,100})@.*' \
alphabet=localemailaddress
```
テンプレートに設定する項目は下記の通りです。

* `type`: 現状は regrex のみサポート
* `pattern`: フォーマットの正規表現
* `alphabet`: 利用するアルファベット


最後に`Transformation`です。

```shell
$ vault write transform/transformations/fpe/email \
template=email-template \
allowed_roles=my-transform-role \
tweak_source=internal
```

今回はトランスフォームの名前に`email`、テンプレートに`email-template`をしてしています。

```shell
vault write transform/role/my-transform-role \
transformations=first-transform,email
```

このトランスフォームを`my-transform-role`ロールで利用可能に設定します。

それぞれの設定を確認しておきましょう。

```console
$ vault read transform/alphabet/localemailaddress
Key         Value
---         -----
alphabet    0123456789.@abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ

$ vault read transform/template/email-template
Key         Value
---         -----
alphabet    localemailaddress
pattern     ([0-9A-Za-z]{1,100})@.*
type        regex

$ vault read transform/transformation/email
Key              Value
---              -----
allowed_roles    [my-transform-role]
templates        [email-template]
tweak_source     internal
type             fpe
```

これを利用して暗号化をしてみましょう。ここで`value`で入力できる値は`localemailaddress`で指定しているアルファベットのみですので注意してください。`_`,`-`などは使えません。(もちろんアルファベットに追加すれば利用できます。)

```console
$ vault write transform/encode/my-transform-role \
value="email@kabuctl.run" \
transformation=email

Key              Value
---              -----
encoded_value    wmeph@kabuctl.run
```

複合化してみます。

```console
$ vault write transform/decode/my-transform-role \
value="wmeph@kabuctl.run" \
transformation=email

Key              Value
---              -----
decoded_value    email@kabuctl.run
```

最後に、`@`以下も暗号するように設定してみます。

テンプレートを以下のように書き直します。

```shell
$ vault write transform/template/email-template \
type=regex \
pattern='([0-9A-Za-z]{1,100})@(.*)\.(.*)' \
alphabet=localemailaddress
```

暗号化と複合化を試していましょう。`@`以下も暗号化されていることがわかるでしょう。

```console
$ vault write transform/encode/my-transform-role \
value="takayuki@kabucorp.com" \
transformation=email

Key              Value
---              -----
encoded_value    XE6TPQy7@kvcnrgrp.xce
```

```console
$ vault write transform/decode/my-transform-role \
value="XE6TPQy7@kvcnrgrp.xce" \
transformation=email
Key              Value
---              -----
decoded_value    takayuki@kabucorp.com
```

### Tweak 値を利用する

ここまで Tweak 値を利用せず暗号化を実施してきました。より安全にデータを守るためにはツイーク値というランダムの値を暗号文とセットで持たせることができます。

ツイーク値を利用することで複合化の際に、暗号キーへのアクセスに合わせてツイーク値を要求することができます。

一番最初に作った`first-transform`で試してみましょう。

```console
$ vault write transform/encode/my-transform-role value=1111-2222-3333-4444
Key              Value
---              -----
encoded_value    1606-8311-1961-4492

$ vault write transform/encode/my-transform-role value=1111-2222-3333-4444
Key              Value
---              -----
encoded_value    1606-8311-1961-4492


$ vault write transform/decode/my-transform-role value=1606-8311-1961-4492
Key              Value
---              -----
decoded_value    1111-2222-3333-4444
```

暗号化された値は同一の値を返し、複合化可能です。

次に Tweak 値を生成するモードに変更します。`tweak_source=generated`です。

```shell
$ vault write transform/transformation/first-transform \
type=fpe \
template=builtin/creditcardnumber \
allowed_roles=my-transform-role \
tweak_source=generated
```

この状態で暗号化を実施してみましょう。

```console
$ vault write transform/encode/my-transform-role value=1111-2222-3333-4444
Key              Value
---              -----
encoded_value    2620-2046-9436-7135
tweak            3T+WJ0yG9Q==

$ vault write transform/encode/my-transform-role value=1111-2222-3333-4444
Key              Value
---              -----
encoded_value    5776-7465-2375-4346
tweak            Jv/kQc9YuQ==
```

実行ごとに別々の値が生成され、それぞれに Tweak 値が生成されていることがわかるでしょう。
このモードの際、正しく値を取り出すに暗号文に合わせて Tweak 値が必ず必要です。

まず Tweak 値を入力せずに試してみます。`value`には上の 2 回目に生成された`encoded_value`を入れてください。

```console
$ vault write transform/decode/my-transform-role value=5776-7465-2375-4346
Error writing data to transform/decode/my-transform-role: Error making API request.

URL: PUT http://127.0.0.1:8200/v1/transform/decode/my-transform-role
Code: 400. Errors:

* incorrect tweak size provided: 0
```

次に Tweak 値を入れて試してみます。`tweak`には上の 2 回目に生成された`tweak`を入れてください。

```console
$ vault write transform/decode/my-transform-role value=5776-7465-2375-4346 tweak=Jv/kQc9YuQ==
Key              Value
---              -----
decoded_value    1111-2222-3333-4444
```

正しく復元できました。最後に Tweak 値に不正確な値を入れてみましょう。`tweak`には上の 1 回目に生成された`tweak`を入れてください。

```console
$ vault write transform/decode/my-transform-role value=5776-7465-2375-4346 tweak=3T+WJ0yG9Q==
Key              Value
---              -----
8534-4499-6272-2127
```

正しい値が返ってこないはずです。このように Tweak を利用することでより高度にデータの暗号化を行うことが可能です。

### Masking を試す

最後に`Masking`を試してみましょう。`Masking`は`one-way encryption`と呼んでいますが、その名の通り、マスキングしたデータは複合化することはできません。

文字通り「データを隠すため」の機能です。そのため、オペミスを防ぐため、`FPE`のタイプのものから`Masking`への変更はできないようになっています。`FPE`は複合化が前提となっているユースケースで利用するからです。

`Transformation`を以下のように作成します。

```shell
$ vault write transform/transformations/masking/masking-email \
template=email-template \
allowed_roles=my-transform-role \
tweak_source=internal
```

確認してみましょう。

```console
$ vault read transform/transformation/masking-email
Key                  Value
---                  -----
allowed_roles        [my-transform-role]
masking_character    42
templates            [email-template]
type                 masking
```

```shell
vault write transform/role/my-transform-role \
transformations=first-transform,email,masking-email
```

これを利用してデータをマスキングしてみます。

```console
$ vault write transform/encode/my-transform-role value="email@kabuctl.com" transformation=masking-email
Key              Value
---              -----
encoded_value    *****@*******.***
```

このようにマスキングされた値が返ってきます。

ちなみに、マスキングする文字列は変更できます。`masking_character=#`を追加してみましょう。

```shell
$ vault write transform/transformations/masking/masking-email \
template=email-template \
allowed_roles=my-transform-role \
tweak_source=internal \
masking_character=#
```

再度データをマスキングしてみます。

```console
$ vault write transform/encode/my-transform-role value="email@kabuctl.com" transformation=masking-email
Key              Value
---              -----
encoded_value    #####@#######.###
```

変更が反映されました。

この機能は例えば Web ブラウザや ATM の画面に実際の値を出したくない際や、ログに PII のデータを出力させたくない時に利用できます。

正規表現を変更することで一部の値のみマスキングすることも可能です。最後にこれを試してみましょう。

アットマーク前の最初と最後の文字を除いた文字列のみマスキングするような表現をしています。

```shell
$ vault write transform/template/email-template \
type=regex \
pattern='.([0-9A-Za-z]{1,100}).@.*' \
alphabet=localemailaddress
```

```console
$ vault write transform/encode/my-transform-role value="takayukikaburagi@kabuctl.com" transformation=masking-email
Key              Value
---              -----
encoded_value    t##############i@kabuctl.com
```

以上で一通りの`Transform Secret Engine`の機能を試すことができました。`Alphabets`と`Templates`の`Patterne`の正規表現を利用することで様々なデータの Transformation を実現することができます。

また、時間のある方はこちらの[サンプルアプリ](https://github.com/tkaburagi/vault-transformation-demo)で実際の Web アプリから利用することを試してみてください。

### 参考リンク
* [Transform Doc](https://www.vaultproject.io/docs/secrets/transform)
* [Transform API](https://www.vaultproject.io/api-docs/secret/transform)
* [Blog Post](https://www.hashicorp.com/blog/transform-secrets-engine/)
* [Tutorial](https://learn.hashicorp.com/vault/adp/transform)

---

## Response Wrapping を使ってシークレットをセキュアに渡して取得する。

Vault から生成されたシークレットを利用する際、シークレットの受け渡しは非常にセンシティブな作業です。その際`Cubbyhole Response Wrapping`という機能を利用し、トークンを一回限りのトークンでラップして受け渡すような運用が可能です。

その際重要になってくる`Cubbyhole`というシークレットエンジンをまずは使ってみたいと思います。

### Cubbyhole

`Cubbyhole`はロッカーや安全な場所という意味で、このシークレットエンジンは他とは違い Vault のトークンに必ず一つ割り当てられ、そのバックエンドは他のいかなる強力な権限を持つトークンからも見ることができません。ルートトークンからも他の Cubbyhole は見ることができません。また、Cubbyhole に格納されたデータはトークンの TTL が切れたり、Revoke されると同時に消滅します。

まずは試してみましょう。

TTL が 15 分のトークンを作ってみます。前の手順で作った`my-first-policy.hcl`を以下のように変更して write してトークンを作ります。

```hcl
path "database/roles/+" {
  capabilities = ["list","create", "read"]
}

path "database/roles/role-handson" {
  capabilities = ["deny"]
}

path "sys/*" {
  capabilities = ["read", "list"]
}
```

```console
$ vault policy write my-policy path/to/my-first-policy.hcl
$ vault token create -policy=my-policy -ttl=15m

Key                  Value
---                  -----
token                s.vz9bwNR7LRtTYiTqo3KxO9aV
token_accessor       FRotEsEBaUxvB0xV84P4vhnH
token_duration       5m
token_renewable      true
token_policies       ["default" "my-policy"]
identity_policies    []
policies             ["default" "my-policy"]
```

このトークンを使って`cubbyhole`にデータを投入します。

```console
$ VAULT_TOKEN=<TOKEN_ABOVE> vault secrets list
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_e3aa0798    per-token private secret storage
database/     database     database_603dc42e     n/a
identity/     identity     identity_86c0240d     identity store
kv/           kv           kv_20084de2           n/a
sys/          system       system_ae51ee57       system endpoints used for control, policy and debugging
transit/      transit      transit_ec14846c      n/a

$ VAULT_TOKEN=s.vz9bwNR7LRtTYiTqo3KxO9aV vault write cubbyhole/my-cubbyhole-secret foo=bar

$ VAULT_TOKEN=s.vz9bwNR7LRtTYiTqo3KxO9aV vault read cubbyhole/my-cubbyhole-secret
Key    Value
---    -----
foo    bar
```

次にルートトークンからこのデータを参照してみましょう。

```console
$ export ROOT_TOKEN=<YOUR_ROOT_TOKEN>
$ VAULT_TOKEN=$ROOT_TOKEN vault list cubbyhole/

$ VAULT_TOKEN=$ROOT_TOKEN vault read cubbyhole/my-cubbyhole-secret 
No value found at cubbyhole/my-cubbyhole-secret
```

データは参照できません。15 分後ログインしようとするとアクセスできず cubbyhole 内のデータも抹消されます。

以上が Cubbyhole です。`Response Wrapping`は内部的に Cubbyhole を使ってセキュアなクレデンシャルの受け渡しを実現します。

### Response Wrapping のワークフロー

Response Wrapping のワークフローは少し複雑です。

1. 実際に利用するクレデンシャルを発行する際に、一時トークン(Wrapping Token)を同時発行します。
2. クレデンシャルは Wrapping Token の`cubbyhole/response`内に保存されます。
3. クライアントは Wapping Token を使って`unwrap`という処理を行い、`cubbyhole/response`内のクレデンシャルを取り出します。
4. 一度利用された`Wrappgin Token`は即座に無効化され 2 度とクレデンシャルは取得できなくなります。

このような感じです。試してみましょう。ゆっくりやって欲しいので TTL は長めの 1 時間にします。まずクレデンシャルの発行です。クレデンシャルは Vault から発行できるシークレットであればなんでも OK です。

ここでは先ほど使った AppRole のシークレット ID を発行してみましょう。まず通常だとこのような結果になります。

```console
$ vault write -f auth/approle/role/my-approle/secret-id
Key                   Value
---                   -----
secret_id             f2f32284-5f39-8347-9278-1b879acedd98
secret_id_accessor    d61156ac-f797-ab7e-5024-a95584f78458
```

次は`-wrap-ttl`のオプションを使ってラッピングトークンを発行します。

```console
$ vault write -wrap-ttl=1h  -f auth/approle/role/my-approle/secret-id
Key                              Value
---                              -----
wrapping_token:                  s.3OzsP31vNPiqoksrtqY0nSmV
wrapping_accessor:               hiw2JzGYV7yKVE4TchiBBwUd
wrapping_token_ttl:              1h
wrapping_token_creation_time:    2019-07-17 21:43:50.763833 +0900 JST
wrapping_token_creation_path:    auth/approle/role/my-approle/secret-id
```

このラッピングトークンは 1 時間有効ですが、一度使うと抹消されます。`unwrap`という操作がアンラップし、Secret ID を取り出してみましょう。

```console
$ vault unwrap <WRAPPING_TOKEN>
Key                   Value
---                   -----
secret_id             eb757e5e-fe36-44c6-7b68-f0e19c692a27
secret_id_accessor    ffe9a83b-9dec-fd86-3d9d-085390d98776
```

Secret ID を取得できました。もう一度 unwrap してみます。

```console
$ vault unwrap <WRAPPING_TOKEN>
Error unwrapping: Error making API request.

URL: PUT http://127.0.0.1:8200/v1/sys/wrapping/unwrap
Code: 400. Errors:

* wrapping token is not valid or does not exist

$ vault token lookup <WRAPPING_TOKEN>
Error looking up token: Error making API request.

URL: POST http://127.0.0.1:8200/v1/auth/token/lookup
Code: 403. Errors:

* bad token
```

unwrap もできず、トークンを Lookup してもエラーが返りトークンが無効になっていることがわかります。このように TTL 内でも一度利用され、クレデンシャルが取得されると 2 度と利用できません。

余裕のある方はもう一度同じ手順でラッピングトークンを作り、今度はそのトークンの`cubbyhole/response`にアクセスしてトークンが保存されていることを確認しましょう。

```console
$ vault write -wrap-ttl=1h  -f auth/approle/role/my-approle/secret-id
$ VAULT_TOKEN=<WRAPPING_TOKEN> vault read cubbyhole/response -format=json
{
  "request_id": "e09bbe6e-d0de-4270-c386-471ce95f9d67",
  "lease_id": "",
  "lease_duration": 0,
  "renewable": false,
  "data": {
    "response": "{\"request_id\":\"b461f7d7-aec4-5543-735a-62118084c69f\",\"lease_id\":\"\",\"renewable\":false,\"lease_duration\":0,\"data\":{\"secret_id\":\"1a532c44-8d7c-84ee-ce3a-4e743a717a42\",\"secret_id_accessor\":\"0479662c-1b5c-fde1-3214-3c645baffe7c\"},\"wrap_info\":null,\"warnings\":null,\"auth\":null}"
  },
  "warnings": [
    "Reading from 'cubbyhole/response' is deprecated. Please use sys/wrapping/unwrap to unwrap responses, as it provides additional security checks and other benefits."
  ]
}
```

JSON のレスポンスでラッピングトークンの`cubbyhole/response`内に`secret_id`が格納されていることがわかります。`read`を行っても`unwrap`と同様、ラッピングトークンが無効になります。

```console
$ vault token lookup <WRAPPING_TOKEN>
Error looking up token: Error making API request.

URL: POST http://127.0.0.1:8200/v1/auth/token/lookup
Code: 403. Errors:

* bad token
```

`Response Wrapping`は AppRole 以外にもトークンなど様々なシークレットに利用することができます。これを利用することでアプリなどからシークレットを取得する際も特権ユーザのトークンを記述したり、Secret ID を直で記述することなくセキュアにシークレットを取り出すことができます。また、人にシークレットを渡す際も Wrapping Token のみを渡して取得してもらうことでより安全なシークレット管理が可能になります。

### 参考リンク
* [Cubbyhole Secret Engine](https://www.vaultproject.io/docs/secrets/cubbyhole/index.html)
* [Cubbyhole API Document](https://www.vaultproject.io/api/secret/cubbyhole/index.html)
* [Response Wrapping](https://www.vaultproject.io/docs/concepts/response-wrapping.html)
