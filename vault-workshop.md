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
- [LDAP による認証](#ldap-による認証)
- [AppRole による認証](#approle-による認証)
- [Vault AWS auth demo](#vault-aws-auth-demo)
- [GCP Auth Method を利用してクライアントを認証する](#gcp-auth-method-を利用してクライアントを認証する)
- [Kubernetes 連携を試す](#kubernetes-連携を試す)
- [多要素認証(Multi Factor Authentication)を試す](#多要素認証multi-factor-authenticationを試す)
- [AWS のシークレットエンジンを試す](#aws-のシークレットエンジンを試す)
- [Azure のシークレットエンジンを試す](#azure-のシークレットエンジンを試す)
- [GCP のシークレットエンジンを試す](#gcp-のシークレットエンジンを試す)
- [Vault を PKI エンジンとして扱う](#vault-を-pki-エンジンとして扱う)
- [Transit シークレットエンジンで Vault を Encryption as a Sevice として使う](#transit-シークレットエンジンで-vault-を-encryption-as-a-sevice-として使う)
- [SSH シークレットエンジンを使ってワンタイム SSH パスワードを利用する](#ssh-シークレットエンジンを使ってワンタイム-ssh-パスワードを利用する)
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

## LDAP による認証

ここでは LDAP を用いた認証を行ってみます。


### LDAP サーバーの準備

ここでは、[OpenLDAP コンテナ](https://github.com/osixia/docker-openldap)を用いて Copy&Paste しながら、LDAP 連携の挙動を確認していきます。

#### OpenLDAP コンテナの起動

まず以下のコマンドで LDAP コンテナを起動します。

```shell
$ docker run -d --name openldap -p 389:389 -e LDAP_ORGANIZATION="Example corp" -e LDAP_DOMAIN="example.org" -e LDAP_ADMIN_PASSWORD="admin" osixia/openldap:1.5.0
```

`docker ps`などでコンテナが起動したことを確認ください。

#### OpenLDAP 環境の設定

LDAP に、`it`部門と、`security`部門を想定したグループを作成し、`it_people_1`と`it_people_2`のユーザを設定します。

まず、以下のコマンドで、LDIF ファイルを作成します。

```shell
$ cat > users.ldif <<'EOF'
# ── ユーザー用 OU
dn: ou=people,dc=example,dc=org
objectClass: organizationalUnit
ou: people

# ── IT 部署ユーザー
dn: uid=it_people_1,ou=people,dc=example,dc=org
objectClass: inetOrgPerson
cn: IT Person One
sn: One
uid: it_people_1
userPassword: pass-it

# ── セキュリティ部署ユーザー
dn: uid=security_people_1,ou=people,dc=example,dc=org
objectClass: inetOrgPerson
cn: Security Person One
sn: One
uid: security_people_1
userPassword: pass-sec

# ── グループ用 OU
dn: ou=groups,dc=example,dc=org
objectClass: organizationalUnit
ou: groups

# ── IT グループ（メンバー：it_people_1）
dn: cn=it,ou=groups,dc=example,dc=org
objectClass: groupOfNames
cn: it
member: uid=it_people_1,ou=people,dc=example,dc=org

# ── セキュリティグループ（メンバー：security_people_1）
dn: cn=security,ou=groups,dc=example,dc=org
objectClass: groupOfNames
cn: security
member: uid=security_people_1,ou=people,dc=example,dc=org
EOF
```

users.ldif ファイルが作成された事を確認したら、稼働中のコンテナに LDIF ファイルのコピーを行います。

```shell
% docker cp users.ldif openldap:/tmp/users.ldif
```

結果として以下のような内容が表示されます。

```shell
Successfully copied 3.07kB to openldap:/tmp/users.ldif
```

LDIF ファイルをコピーしたら、設定ディレクトリにコピーして LDAP に設定を反映します。

```shell
$ docker exec -it openldap ldapadd -x -D "cn=admin,dc=example,dc=org" -w admin -f /tmp/users.ldif
```

結果として以下のような内容が表示されます。


```shell

adding new entry "ou=people,dc=example,dc=org"

adding new entry "uid=it_people_1,ou=people,dc=example,dc=org"

adding new entry "uid=security_people_1,ou=people,dc=example,dc=org"

adding new entry "ou=groups,dc=example,dc=org"

adding new entry "cn=it,ou=groups,dc=example,dc=org"

adding new entry "cn=security,ou=groups,dc=example,dc=org"
```

もし投入に失敗して途中から投入をリトライする場合には `-c` オプションが必要となります。
修正して設定を LDAP 設定をやりなおす場合、再び LDIF ファイルをコピーした上で、以下のコマンドを試してください。

```shell
$ dokcer exec -it openldap ldapadd -c -x -D "cn=admin,dc=example,dc=org" -w admin -f /tmp/users.ldif
```


#### OpenLDAP コンテナとの通信確認
LDAP サーバーとの通信を確認するには、以下のコマンドを叩いてエントリーが取得できることを確認ください。

コマンドとしては、 `-D` には、バインド DN として admin ユーザで LDAP にログインする事を指定しています。検索クエリとして設定されいてる `(&(A)(B))` は論理積（AND 条件）で、 `(obujectClass=groupOfNames)` でグループオブジェクトである事と、 `(cn=valut-users)` でグループ名が it（もしくは security）であることを指定して検索しています。

以下では IT グループに所属しているユーザー一覧を表示します。
`-b` には検索起点となるベース DN を設定 `"dc=example,dc=org"` として example.com を指定しています。

```shell
$ docker exec -it openldap ldapsearch -x -D "cn=admin,dc=example,dc=org" -w admin -b "dc=example,dc=org" "(&(objectClass=groupOfNames)(cn=it))" -LLL cn member
```

結果として以下のような内容が表示されます。

```shell
dn: cn=it,ou=groups,dc=example,dc=org
cn: it
member: uid=it_people_1,ou=people,dc=example,dc=org
```

以下では Security グループに所属しているユーザー一覧を表示します。

```shell
$ docker exec -it openldap ldapsearch -x -D "cn=admin,dc=example,dc=org" -w admin -b "dc=example,dc=org" "(&(objectClass=groupOfNames)(cn=security))" -LLL cn member
```

結果として以下のような内容が表示されます。

```shell
dn: cn=security,ou=groups,dc=example,dc=org
cn: security
member: uid=security_people_1,ou=people,dc=example,dc=org
```


### LDAP auth method の設定

次に Vault 側で LDAP auth method を設定します。

```shell
$ vault auth enable -path=ldap ldap
```

結果として以下のような内容が表示されます。

```shell
Success! Enabled ldap auth method at: ldap/
```

Vault に、LDAP の設定内容に沿った設定を投入します。コマンドの前に解説すると以下の様な内容となっています。

```
# LDAPサーバーのURL
url      = "ldap://127.0.0.1:389"

# LDAPに接続するための管理者アカウント(DN)とパスワード
binddn   = "cn=admin,dc=example,dc=org"
bindpass = "admin"

# ユーザーが格納されているベースDN
userdn   = "ou=people,dc=example,dc=org"

# Vaultにログインする際に使用するユーザー属性
# ここではLDAP設定上の "uid" をログイン名として利用する
userattr = "uid"

# グループが格納されているベースDN
groupdn  = "ou=groups,dc=example,dc=org"

# グループに属しているかを確認するフィルタ
# {{.UserDN}} がログイン中のユーザーDNに置き換えられる
groupfilter = "(&(objectClass=groupOfNames)(member={{.UserDN}}))"

# VaultがLDAP設定上のグループ名として利用する属性
groupattr   = "cn"

# TLS証明書の検証を無効化（テスト用途のみ）
insecure_tls = true
```

上記の内容を vault に設定するコマンドは以下の通りです。
LDAP サーバーとの通信に必要な設定を行っています。

```shell
$ vault write auth/ldap/config -<< EOH
{
"url":"ldap://127.0.0.1:389",
"binddn":"cn=admin,dc=example,dc=org",
"bindpass":"admin",
"userdn":"ou=people,dc=example,dc=org",
"userattr":"uid",
"groupdn":"ou=groups,dc=example,dc=org",
"groupfilter":"(&(objectClass=groupOfNames)(member={{.UserDN}}))",
"groupattr":"cn",
"insecure_tls":true
}
EOH
```

結果として以下のような内容が表示されます。

```shell
Success! Data written to: auth/ldap/config

```

`vault auth list`コマンドで LDAP 認証が作成されていることを確認ください。

```console
$ vault auth list
```

結果として以下のような内容が表示されます。

```console
Path      Type     Accessor               Description                Version
----      ----     --------               -----------                -------
ldap/     ldap     auth_ldap_d161dcdc     n/a                        n/a
token/    token    auth_token_3817a250    token based credentials    n/a
```


### シークレットを準備

次にこのワークショップで用いるシークレットを準備します。Secret engine は KV エンジンを使用します。もし、まだ設定していない場合は以下のコマンドで KV を有効化してください。

```shell
$ vault secrets enable -path=secret kv
```

これにより、Vault 上の/secret という Path に KV エンジンがマウントされます。

すでに `secret` が設定されているかどうかは、以下のコマンドで確認できます。

```shell
% vault secrets list
```

結果として以下のような内容が表示されます。

```shell
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_ea802eb5    per-token private secret storage
identity/     identity     identity_db97ced7     identity store
secret/       kv           kv_826e0694           key/value secret storage
sys/          system       system_814a0e43       system endpoints used for control, policy and debugging

```


KV エンジンに IT 部門用のシークレットを書き込みます。

```shell
% vault kv put secret/ldap/it password="foo"
```

結果として以下のような内容が表示されます。

```shell
=== Secret Path ===
secret/data/ldap/it

======= Metadata =======
Key                Value
---                -----
created_time       2025-09-09T10:38:29.024046Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

KV エンジンにセキュリティ部門用のシークレットを書き込みます。

```shell
 % vault kv put secret/ldap/security password="bar"
```

結果として以下のような内容が表示されます。


```shell
====== Secret Path ======
secret/data/ldap/security

======= Metadata =======
Key                Value
---                -----
created_time       2025-09-09T10:38:46.759949Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

IT グループ向け、Security グループ向けの２つのシークレットが書き込まれました。


### Policy の設定

次に、これらのシークレットへのアクセスを許可するための Policy を準備します。


#### Policy の中身
ここでは以下の IT グループ向けと Security グループ向けの２種類の Policy を使用します。

まず IT 部門用の Policy ファイルを作成します。

```shell
$ cat > it_policy.hcl <<'EOF'
# Policy for IT people
path "secret/data/ldap" {
	capabilities = [ "list" ]
}

# For KV v2 ACL rules
# see https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2/upgrade
path "secret/data/ldap/it" {
	capabilities = [ "create", "read", "update", "delete", "list" ]
}

EOF
```

次に Security 部門用の Policy ファイルを作成します。

```shell
$ cat > security_policy.hcl <<'EOF'
# Policy for security people

path "secret/data/ldap" {
	capabilities = [ "list" ]
}

# For KV v2 ACL rules
# see https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2/upgrade
path "secret/data/ldap/security" {
	capabilities = [ "create", "read", "update", "delete", "list" ]
}
EOF
```

それぞれの Policy に、secret/ldap 以下（secret/data/ldap 以下）のそれぞれのシークレットへのアクセス権限が明示的に記載されています。

path に `secret/data` から始まっている点について詳しく知りたい場合は、
https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2/upgrade
を確認してください。


#### Policy の設定とグループへの適用

Policy が準備できたら、その Policy を LDAP 上のグループと紐付けます。

まず、IT 部門の Policy を設定します。

```shell
$ vault policy write it_policy it_policy.hcl
```

結果として以下のような内容が表示されます。

```shell
Success! Uploaded policy: it_policy
```

次に、Security 部門の Policy を設定します。

```shell
$ vault policy write security_policy security_policy.hcl
```

結果として以下のような内容が表示されます。

```shell
Success! Uploaded policy: security_policy
```

設定された IT 部門用の Policy を LDAP 上の it グループと紐づけます。

```shell
$ vault write auth/ldap/groups/it policies=it_policy
```

結果として以下のような内容が表示されます。

```shell
Success! Data written to: auth/ldap/groups/it
```

設定された security 部門用の Policy を LDAP 上の security グループと紐づけます。

```shell
$ vault write auth/ldap/groups/security policies=security_policy
```

結果として以下のような内容が表示されます。

```shell
Success! Data written to: auth/ldap/groups/security
```


`vault policy write` で Policy を書き込み、 `vault write auth/ldap/groups/<グループ名>` でグループと Policy を紐付けました。



### LDAP 認証を利用したログイン

これで Vault を通じて LDAP 認証を行う準備が整いました。

まず、IT 部門のメンバー `it_people_1` でログインしてみます。

```shell
% vault login -method=ldap -path=ldap username=it_people_1 password=pass-it
```

結果として以下のような内容が表示されます。

```shell
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                    Value
---                    -----
token                  hvs.CAESILL....
token_accessor         BxhAnmJI3EOKP2URyPOabZp8
token_duration         768h
token_renewable        true
token_policies         ["default" "it_policy"]
identity_policies      []
policies               ["default" "it_policy"]
token_meta_username    it_people_1
```


無事にログインされ Token が返ってきています。token_policies に"it_policy"が設定されていることを確認ください。

上記の token を以降の確認作業に利用するため、保管してください。


次に、Security 部門のメンバー `security_people_1` でログインしてみます。

```shell
% vault login -method=ldap -path=ldap username=security_people_1 password=pass-sec
```

結果として以下のような内容が表示されます。

```shell
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                    Value
---                    -----
token                  hvs.CAESIB............
token_accessor         yZZm6fucxvbF4qUNrbge9aVX
token_duration         768h
token_renewable        true
token_policies         ["default" "security_policy"]
identity_policies      []
policies               ["default" "security_policy"]
token_meta_username    security_people_1
```

こちらも無事にログインされ Token が返ってきています。token_policies に"security_policy"が設定されていることを確認ください。

同じく、上記の token を以降の確認作業に利用するため、保管してください。

また、`vault token lookup` で現在の Token を確認できます。

ここからの動作確認のため、それぞれのログインの結果出力にある token を環境変数にいれておきます。

（Token は出力内容にあわせて差し替えてください）

```shell
$ #IT部門ユーザのtoken
$ export IT_TOKEN=hvs.CAESILL....
$ #security部門ユーザのtoken
$ export SECURITY_TOKEN=hvs.CAESIB......
```

この後の作業では、これらの Token を切り替えて作業していきます。Token の切り替えは、 `vault login <Token値>` で行います。

```shell
$ vault login hvs.CAESILL....
$ # 上記のITグループユーザのTokenを使用
```

Token は環境変数 `VAULT_TOKEN` からも設定できます。よって、以下のような切り替えも可能です。

```shell
$ VAULT_TOKEN=$IT_TOKEN vault <コマンド>  # コマンドをITトークンで実行
$ VAULT_TOKEN=$SECURITY_TOKEN vault <コマンド>  # コマンドをSecurityトークンで実行
```


### シークレットの取得

それではシークレットの取得をしてみましょう。

まず、Policy によれば IT グループのユーザーは `secret/ldap/it` にアクセスができるはずです。
（policy ファイルでは `secret/data/ldap/it` を設定していますが問題なく見えるはずです。）

```shell
$ VAULT_TOKEN=$IT_TOKEN vault kv get /secret/ldap/it
```

`secret/data/ldap/it` が表示されていることが確認できます。

```shell
=== Secret Path ===
secret/data/ldap/it

======= Metadata =======
Key                Value
---                -----
created_time       2025-09-09T11:09:11.314132Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

====== Data ======
Key         Value
---         -----
password    foo
```

同様に Security グループのユーザは、 `secret/ldap/security` にアクセスできます。
（policy ファイルでは `secret/data/ldap/security` を設定しています）

```shell
$ VAULT_TOKEN=$SECURITY_TOKEN vault kv get /secret/ldap/security
```

`secret/data/ldap/security` が表示されていることが確認できます。

```shell
====== Secret Path ======
secret/data/ldap/security

======= Metadata =======
Key                Value
---                -----
created_time       2025-09-09T11:09:25.600965Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

====== Data ======
Key         Value
---         -----
password    bar
```

ここで、IT 部門ユーザで、security の secret の取得を試してみます。

```shell
$ VAULT_TOKEN=$IT_TOKEN vault kv get /secret/ldap/security
```

以下のように `permisson denied` が表示されて取得ができません。
policy がうまく動作しています。
IT グループの Policy では　`secret/ldap/security` へのアクセス権限がないので無事にはじかれました。


```shell

Error reading secret/data/ldap/security: Error making API request.

URL: GET http://127.0.0.1:8200/v1/secret/data/ldap/security
Code: 403. Errors:

* 1 error occurred:
	* permission denied
```

```shell
$ VAULT_TOKEN=$SECURITY_TOKEN vault kv get /secret/ldap/it
```

同様にセキュリティ部門ユーザは `secret/ldap/it` へのアクセス権限がないのではじかれています。
こうして、Policy の動作を確認することができました。

```shell
Error reading secret/data/ldap/it: Error making API request.

URL: GET http://127.0.0.1:8200/v1/secret/data/ldap/it
Code: 403. Errors:

* 1 error occurred:
	* permission denied
```

### 参考リンク
* [LDAP auth method](https://www.vaultproject.io/docs/auth/ldap.html
)
* [API doc](https://www.vaultproject.io/api/auth/ldap/index.html)

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

## GCP Auth Method を利用してクライアントを認証する

Table of Contents                                                                                                          
=================                                                                                                                                                                                
  * [Vault 事前準備](#vault-事前準備)                                                
  * [GCP 事前準備](#gcp事前準備)
  * [GCP Auth Method の設定 (IAM 編)](#gcp-auth-methodの設定-iam編)
  * [GCP Auth Method の設定 (GCE 編)](#gcp-auth-methodの設定-gce編)
  * [参考リンク](#参考リンク)

GCP Auth Method では GCP の`IAM Service Account`, `Google Compute Engine Instances`を利用してクライアントを認証することができます。

* `Service Account`は IAM Service Account を利用しての認証
* `Google Compute Engine Instances`は GCE インスタンスのメタデータを利用して認証

このハンズオンでは GCP アカウントが必要です。[こちら](https://cloud.google.com/free/)からアカウントを作成していください。

### Vault 事前準備

まずは Vault 上のデータとポリシーを準備します。Root Token でログインし、以下のコマンドを実行してください。

```sh
$ vault kv put kv/cred-1 name=user-1 password=passwd
$ vault kv put kv/cred-2 name=user-2 password=dwssap
```

次にポリシーを作成します。

```sh
$ vault policy write read-cred-1 -<<EOF
path "kv/cred-1" {
  capabilities = [ "read" ]
}
EOF
```

このポリシーは後ほどクライアントが利用する GCP Service Account との紐付けを行い、GCP Auth でログインしたユーザに与える Vault の権限になります。

ここでは`KV Secret Engine`の`kv/cred-1`の read のみ出来る権限として設定しています。

### GCP 事前準備

GCP 側の設定です。

まず、トップ画面の検索ボックスから`IAM Service Account Credentials API`と検索し`Enable`をクリックして API を有効化します。

次に GCP を使って認証をするために Vault 側に設定する Service Account の発行です。Vault はこのシークレットを利用して GCP へ認証を依頼します。

GCP のコンソールにログインして、`Navigation Menu`から`IAM&Admin` -> `Service accounts`と進んでください。

`CREATE SERVICE ACCOUNT`をクリックして、名前に`vault-server`と入力してロールの選択に移ります。

* `Compute Viewer`
* `Service Account Key Admin`

の二つを選択し、`CREATE`してください。

`CONTIUNE`で進んだら、`CREATE KEY`で JSON のキーを発行します。ダウンロードされたキーは`.gcp-vault-auth-config-key.json`にリネームします。

```sh
$ mv /path/to/***********.json ~/.gcp-vault-auth-config-key.json
```

次に Vault にログインするクライアント側の Service Account を発行します。

GCP のコンソールにログインして、`Navigation Menu`から`IAM&Admin` -> `Service accounts`と進んでください。

`CREATE SERVICE ACCOUNT`をクリックして、名前に`vault-client`と入力してロールの選択に移ります。

* `Service Account Token Creator`

を選択し、`CREATE`してください。

`CONTIUNE`で進んだら、`CREATE KEY`で JSON のキーを発行します。ダウンロードされたキーは`.gcp-vault-client-key.json`にリネームします。

```sh
$ mv /path/to/***********.json ~/.gcp-vault-client-key.json
```

これで GCP 側の準備は完了です。

### GCP Auth Method の設定 (IAM 編)

[こちら](https://www.vaultproject.io/docs/auth/gcp#iam-login)がワークフローです。

最後に`GCP Auth Method`の設定を行います。

GCP 認証を有効化し、Vault 用の Service Account をセットします。Vault はこの Service Account を利用して GCP へ認証を依頼します。

```sh
$ vault auth enable gcp
$ vault write auth/gcp/config credentials=@.gcp-vault-auth-config-key.json
```

`role`を作成します。`read-cred-1`のポリシーを先ほど発行した Service Account にバインドします。これでこの Service Account を使ってログインしたユーザに`read-cred-1`で設定した Vault の権限を与えることができます。

`GCP_PRJ`にご自身の GCP プロジェクト名をセットしてください。

```sh
$ GCP_PRJ=se-kabu
$ vault write auth/gcp/role/read-cred \
    type="iam" \
    policies="read-cred-1" \
    bound_service_accounts="vault-client@${GCP_PRJ}.iam.gserviceaccount.com"
```

これで設定は完了です。ログインしてみましょう。ログインには

* CLI Helper を使って認証に必要な JWT を取得して Vault にリクエストする(IAM のみ有効)
* CLI 使って別で生成した JWT を使ってリクエストする
* API を実行する

の 3 パターンがあります。今回は CLI Helper を使ってみます。

```sh
$ vault login -method=gcp \
    role="read-cred" \
    service_account="vault-client@${GCP_PRJ}.iam.gserviceaccount.com" \
    project="${GCP_PRJ}" \
    jwt_exp="15m" \
    credentials=@.gcp-vault-client.key.json
```

これでログインができました。以降のリクエストはここで発行されたトークンを使って実行されます。トークンの権限を試してみましょう。

```console
$ vault kv get kv/cred-1
====== Data ======
Key         Value
---         -----
name        user-1
password    passwd

$ vault kv get kv/cred-2
Error reading kv/cred-2: Error making API request.

URL: GET http://127.0.0.1:8200/v1/kv/cred-2
Code: 403. Errors:

* 1 error occurred:
	* permission denied
```

ロールとポリシーで設定したように今回作成した Service Account では`kv/cred-1`を read するためのポリシーがバインドされているため設定通りに動作していることがわかります。

### GCP Auth Method の設定 (GCE 編)

次に GCE のメタデータを利用して認証するパターンを試してみます。

手順の前に以下の GCE インスタンスを立ち上げてください。

* Service Account: `vault-client`
* Zone: `asia-northeast1-b`
* Label: `foo:bar`
* SSH ログインが有効
* curl コマンドが利用可能
* インターネットアクセス可能

GCE インスタンスが立ち上がったら、Vault 側の設定を加えます。まずは Root Token でログインし直してください。

```sh
$ vault login
```

次に GCE 認証用のロールを定義します。

```sh
$ ZONES=asia-northeast1-b
$ LABELS=foo:bar
$ vault write auth/gcp/role/read-cred-gce \
    type="gce" \
    policies="read-cred-1" \
    bound_projects=${GCP_PRJ} \
    bound_zones=${ZONES} \
    bound_labels=${LABELS}
```

先ほどは Service Account で認証しましたが、今回は GCE のメタデータを利用します。その他にも以下のメタデータをセットできます。

* GCE インスタンスに付与される`Service Account`
* `Instance Group`
* `Region`

各パラメタータをリスト型で設定できるため複数の値を入れることもできます。

これで Vault 側の設定は完了です。

次に GCE インスタンスに SSH で入り、次のコマンドを実行してください。

```sh
$ ROLE=read-cred-gce

$ curl \
  --header "Metadata-Flavor: Google" \
  --get \
  --data-urlencode "audience=http://vault/${ROLE}" \
  --data-urlencode "format=full" \
  "http://metadata/computeMetadata/v1/instance/service-accounts/default/identity"
```

インスタンスのメタデータサーバから JWT の発行を依頼しています。これは GCE インスタンス上からのみ有効なリクエストです。

発行された JWT をコピーしてローカルの端末に戻ります。先ほど発行された JWT を使ってログインしてみましょう。(**今回は Vault がローカルマシンで起動している前提のためローカルから実行しますが、通常は GCE からリーチできる所に配置し GCE インスタンスから利用します。**)

こちらのワークフローがわかりやすいです。(refer: https://petersouter.xyz/demonstrating-the-gce-auth-method-for-vault/)
<kbd>
  <img src="https://petersouter.xyz/images/2018/07/jwt_gcp_explanation.png">
</kbd> 

```sh
$ JWT=<COPIED_TOKEN>
$ VTOKEN=$(vault write -field=token auth/gcp/login \
        role="read-cred-gce" \
        jwt=${JWT})
$ echo ${VTOKEN}
```

トークンが発行されたはずです。

```console
$ vault token lookup ${VTOKEN}
Key                 Value
---                 -----
accessor            29wEzEdQ3O3yxgDXqzAfKD8G
creation_time       1591159208
creation_ttl        768h
display_name        gcp-terraform
entity_id           796c99f7-718c-1023-69fb-1da9da0b0f01
expire_time         2020-07-05T13:40:08.801533+09:00
explicit_max_ttl    0s
id                  s.uwhaAkaS9A7C2FVAGOIe1SKA
issue_time          2020-06-03T13:40:08.801538+09:00
meta                map[instance_creation_timestamp:1591159129 instance_id:6418544221929256035 instance_name:terraform project_id:se-kabu project_number:707116064532 role:read-cred-gce service_account_email:vault-client@se-kabu.iam.gserviceaccount.com service_account_id:101585660406385936575 zone:asia-northeast1-b]
num_uses            0
orphan              true
path                auth/gcp/login
policies            [default read-cred-1]
renewable           true
ttl                 767h59m50s
type                service
```

`read-cred-1`のポリシーが付与されていることがわかるでしょう。

このトークンを使ってログインして先ほどと同様にテストしてみます。

```console
$ vault login ${VTOKEN}
$ vault kv get kv/cred-1
====== Data ======
Key         Value
---         -----
name        user-1
password    passwd

$ vault kv get kv/cred-2
Error reading kv/cred-2: Error making API request.

URL: GET http://127.0.0.1:8200/v1/kv/cred-2
Code: 403. Errors:

* 1 error occurred:
    * permission denied
```

設定した通りの権限となっているでしょう。

このように GCE インスタンスが`GCE Metadata Server`と連携をし Signed JWT を取得し、それを利用して Vault の認証することができます。

これを利用することで GCE インスタンスのメタ情報をもとに GCE インスタンスに Vault のアクセス権限を付与することが可能です。

### 参考リンク
* [GCP Auth Method](https://www.vaultproject.io/docs/auth/gcp)
* [GCP Auth Mehotd API](https://www.vaultproject.io/api/auth/gcp)
* [Generating JWTs](https://www.vaultproject.io/docs/auth/gcp#generating-jwts)
* [Demonstrating the GCE Auth method for Vault](https://petersouter.xyz/demonstrating-the-gce-auth-method-for-vault/)

---

## Kubernetes 連携を試す

Vault と Kubernetes は様々な形で連携できます。例えば、

* Vault を K8s の Pod として稼働させる
* K8s 上の Pod から Vault の動的シークレットを取得する

などです。

ここでは以下のような構成で試してみます。
![](https://github-image-tkaburagi.s3-ap-northeast-1.amazonaws.com/vault-workshop/Screen+Shot+2019-08-19+at+15.01.12.png)
* Rails <-> Postgres で Rails からデータを取得
* Vault <-> Postgres で Postgres のシークレットを発行、更新
* Rails <-> Vault で Postgres のシークレットを取得
* Vault <-> K8s でサービスアカウントの連携

### install Vault on K8s

まずは Vault を Kuberenetes 上にデプロイしてみます。[前日アナウンスされた](https://www.hashicorp.com/blog/announcing-the-vault-helm-chart)Helm でのインストールです。今回は練習のため K8s 上にデプロイしますが、2019 年 8 月 19 日現在 Enterprise 版ではサポート対象外の構成となります。近い将来サポート対象になるはずです。

インストールは簡単です。minikube が起動していることを確認してください。

```shell
$ git clone https://github.com/hashicorp/vault-helm.git
$ helm init
$ helm install ./vault-helm --name=vault
```

インストールが完了したら別の端末を立ち上げてポートフォワードします。

```shell
$ kubectl port-forward vault-0 8200:8200
```

```shell
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault operator init
$ vault operator unseal <UNSEAL_KEY>
```

Unseal は 3 回繰り返してください。初期化の手順は[こちら](https://github.com/hashicorp-japan/vault-workshop/blob/master/contents/hello-vault.md#vault%E3%81%AE%E5%88%9D%E6%9C%9F%E5%8C%96%E5%87%A6%E7%90%86)を参考にして下さい。

```console
$ vault status
Key                      Value
---                      -----
Recovery Seal Type       shamir
Initialized              true
Sealed                   false
Total Recovery Shares    5
Threshold                3
Version                  1.1.3
Cluster Name             vault-cluster-26e7cd60
Cluster ID               6c2c7a37-c0c5-c9dd-e118-9b38fa8fb920
HA Enabled               false
```

`Sealed`が`false`になっていれば OK です。データベースシークレットエンジンを有効化しておきましょう。

```shell
$ vault login <ROOT_TOKEN>
$ vault secrets enable database
```

次に Postgres を K8s 上にインストールします。

### install Postgres on K8s

次に Postgres をインストールします。

```shell
helm install --name postgres \
             --set image.repository=postgres \
             --set image.tag=10.6 \
             --set postgresqlDataDir=/data/pgdata \
             --set persistence.mountPath=/data/ \
             stable/postgresql
```

Pod の起動を確認しましょう。

```console
$ kubectl get pods
NAME                                           READY   STATUS    RESTARTS   AGE
postgres-postgresql-0                          1/1     Running   1          7d8h
vault-0                                        0/1     Running   1          7d12h
```

Postgres ユーザのパスワードを設定します。

```shell
$ kubectl exec -it postgres-postgresql-0 -- psql -U postgres
```

```shell
$ ALTER USER postgres WITH PASSWORD 'postgres';
$ quit
```

### Vault - Postgres 間の連携設定

前に実施した MySQL と同様、Postgres のシークレットを払い出すための設定を K8s 上の Vault に行っていきます。まずは Config の設定です。

```shell
$ vault write database/config/postgres \
plugin_name=postgresql-database-plugin \
allowed_roles="postgres-role" \
connection_url="postgresql://postgres:postgres@postgres-postgresql.default.svc.cluster.local:5432/postgres?sslmode=disable"
```

次に`postgres-role`の設定です。

```shell
$ vault write database/roles/postgres-role \
db_name=postgres \
creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA PUBLIC TO \"{{name}}\";" \
default_ttl="1h" \
max_ttl="24h"
```

このロールを使ってシークレットを払い出します。

```shell
$ vault read -format json database/creds/postgres-role
```

<details><summary>出力結果の例</summary>

```json
{
  "request_id": "98843e81-cb6d-10cc-a7a8-96754fcebbe1",
  "lease_id": "database/creds/postgres-role/RMTEnlBPjOQt4JKql7Qsm4Z3",
  "lease_duration": 3600,
  "renewable": true,
  "data": {
    "password": "A1a-3ViKAGli3CCm4VUm",
    "username": "v-root-postgres-bSpP7p8SwwCNENAFyTfK-1566227063"
  },
  "warnings": null
}
```
</details>

最後にポリシーの設定を行います。このポリシーは K8s 上の Pod のアプリから取得するトークンに紐づくポリシーです。つまり、アプリケーションに与える権限となります。

```shell
$ cat > postgres-policy.hcl <<EOF
path "database/creds/postgres-role" {
  capabilities = ["read"]
}
path "sys/leases/renew" {
  capabilities = ["create"]
}
path "sys/leases/revoke" {
  capabilities = ["update"]
}
EOF
```

```shell
$ vault policy write postgres-policy postgres-policy.hcl
```

### Kubernetes 側の設定

次は K8s の設定です。`Service Account`と[TokenReview API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#tokenreview-v1-authentication-k8s-io)を使ってサービスアカウントに認証するための`Cluster Role Binding`を作ります。

```shell
$ cat > postgres-serviceaccount.yml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: postgres-vault
  namespace: default
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: postgres-vault
EOF
```

`ClusterRole`の`system:auth-delegator`は K8s がデフォルトで持っているロールです。 このロールを`postgres-vault`のサービスアカウントにマッピングして ReviewToken API を使った認証認可の権限を与えます。

```shell
$ kubectl apply -f postgres-serviceaccount.yml
```

### Vault Kubernetes Auth Method の設定

次に(ここから本題です)Vault の K8s Auth Method の設定を行います。これは Kubernetes のサービスアカウントトークンを利用して Vault 認証するための設定です。これによって K8s 上の Pod からサービスアカウントを使って Vault から Postgres のシークレットを取得することが可能になります。

Vault の K8s 認証メソッドを有効化しておきます。

```shell
$ vault auth enable kubernetes
```

次に Kubernetes の認証の設定を行います。以下の情報が必要です。

* `kubernetes_ca_cert`
  * TLS クライアントが K8s API を使うための証明書
* `token_reviewer_jwt`
  * TokenReview API にアクセスするために使用されるサービスアカウントトークン

これらを取得するために以下のコマンドを実行してください。
```
$ export VAULT_SA_NAME=$(kubectl get sa postgres-vault -o jsonpath="{.secrets[*]['name']}")
$ export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data.token}" | base64 --decode; echo)
$ export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)
$ export K8S_HOST=$(kubectl exec -it vault-0 -- sh -c 'echo $KUBERNETES_SERVICE_HOST')
```

<details><summary>`kubectl get sa postgres-vault -o json`の例</summary>

```json
{
    "apiVersion": "v1",
    "kind": "ServiceAccount",
    "metadata": {
        "annotations": {
            "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"ServiceAccount\",\"metadata\":{\"annotations\":{},\"name\":\"postgres-vault\",\"namespace\":\"default\"}}\n"
        },
        "creationTimestamp": "2019-08-12T07:04:38Z",
        "name": "postgres-vault",
        "namespace": "default",
        "resourceVersion": "26321",
        "selfLink": "/api/v1/namespaces/default/serviceaccounts/postgres-vault",
        "uid": "7309c23d-bccf-11e9-9fcc-0800275363d6"
    },
    "secrets": [
        {
            "name": "postgres-vault-token-ts9x4"
        }
    ]
}
```
</details>

<details><summary>`kubectl get secret $VAULT_SA_NAME -o json`の例</summary>

```json
{
    "apiVersion": "v1",
    "data": {
        "ca.crt": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM1ekNDQWMrZ0F3SUJBZ0lCQVRBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwdGFXNXAKYTNWaVpVTkJNQjRYRFRFNU1EZ3hNREE0TVRVeE9Wb1hEVEk1TURnd09EQTRNVFV4T1Zvd0ZURVRNQkVHQTFVRQpBeE1LYldsdWFXdDFZbVZEUVRDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBS1FSCnNxRUV4SjZYU1QxZWNzeU9QVWlrQ1F4Rnd0bGcvbTc4RjVpUXI2clRndEpDRlZqTnhQZXdRaEtvcWlCLzg2WXYKT1UybFFyVEpycTFHazFpdFlDdXRlcjZtY0xLYk9wSHZYb01MUUtkZXlXRzdtcUEwb0pFY2xvWFZSNjI5V0pSSwplZ1FZb3B2Wk5IVVdYTnQxNEFjTjFDS1F0WmVhd3JqTHc0SXk2R3BvWHFLa0o3Q052QU45NEpQT0J3T25GbjUrCnMyNFhnd29yWnI4YWVOcUZNdGVWWkllMk9qS2c5dnJmb1Zvc3NNeTB1NmdwR3N4Q2tMUFFUeGthQnNwMW1KRWEKZHFFR082R2x2QTdYR1NocGR3RHk4UFM5b2ltVzVEQkhrUWFOWDYxQ1JzYlpLaVVhZXFtOCtJNllvYWxGNEdIegptV0hJRnhEcyttem43V2VUbHBVQ0F3RUFBYU5DTUVBd0RnWURWUjBQQVFIL0JBUURBZ0trTUIwR0ExVWRKUVFXCk1CUUdDQ3NHQVFVRkJ3TUNCZ2dyQmdFRkJRY0RBVEFQQmdOVkhSTUJBZjhFQlRBREFRSC9NQTBHQ1NxR1NJYjMKRFFFQkN3VUFBNElCQVFBZnA0VkQ4Sk1PTjdtYmxJaExmRmJTNmdFUUVuMit6eHU0dC9UZTIwVmhLenVHTlBsTwpyM09YV2trWWZBdVFHM3R1NGVDZ2pWR3NWeElPZndxVUswVEhRZ0tnNHFxRXdSdmEwNEhTK0ZKZk1yM1ZCZVFCCjhNbjNGSnErVS8wa3hrNmdWKy96WDVQa1BqalhJaHNmVlVGRzNsVjUydnIyeVIwQ0xIRnRkRkI2TzY3L2h0MVMKWFZsRExWQk1vVDVWbENlTDNjS2srZU4yUGZ2ZVlWRmxBbjlNemRqL2dLYjBVUFF1OFVLNDhyS0U5Ri96Tk5HZgp3T0FoVDI0VVVNUklxTVlNWXdsSEpOaVZqWEtBa05Eb3FFSVdYVHUwUW5uMFc1blUwR0RpNXEycVA5UDA1NWJWCnZkUG8zU3l3Ulk3YzU3UkJveHBpWVVxd1pjT2JZdk5WNC95UAotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==",
        "namespace": "ZGVmYXVsdA==",
        "token": "ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklpSjkuZXlKcGMzTWlPaUpyZFdKbGNtNWxkR1Z6TDNObGNuWnBZMlZoWTJOdmRXNTBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5dVlXMWxjM0JoWTJVaU9pSmtaV1poZFd4MElpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl6WldOeVpYUXVibUZ0WlNJNkluQnZjM1JuY21WekxYWmhkV3gwTFhSdmEyVnVMWFJ6T1hnMElpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl6WlhKMmFXTmxMV0ZqWTI5MWJuUXVibUZ0WlNJNkluQnZjM1JuY21WekxYWmhkV3gwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXpaWEoyYVdObExXRmpZMjkxYm5RdWRXbGtJam9pTnpNd09XTXlNMlF0WW1OalppMHhNV1U1TFRsbVkyTXRNRGd3TURJM05UTTJNMlEySWl3aWMzVmlJam9pYzNsemRHVnRPbk5sY25acFkyVmhZMk52ZFc1ME9tUmxabUYxYkhRNmNHOXpkR2R5WlhNdGRtRjFiSFFpZlEuWGF0TktwVmpFNnJLZ3JzN2t3TTE3ODU1UXhBLVc2b0lRTWlUZW9rQmJEZjRfd3EwcWJzN2pwSEJnRlRVMkdORl9DTWkwSmlHa0loUFJ1X3NIUTkzaHVwdDBJWFFFZzNURFR2OXFsVFdlN1hQUFRaLXhqdUpRUFhxNENHMzd1R2x1T1J4UmdWWktVM3FaSnV4WlZINmNSdjJMeF9aZDN5MFppN09EUkZWaEJtLUtpeFN1czdFeHdjd3BUQlRoQ3EtSm12c25od3ZmOFlHeDdEdUUtQ3FndEdjMXJnZ2N1djd1WC1BbnZKRXRiLTVwdEgwZWlVX3F4dXNHb2c5LW5iMXRnYTR3dHJfd1V4bWZCMFAxZWp0S2hUSm1ZZ3dIdi1zYlNYTGVLZ3VLNVRLYXBMSV9EaWFXdnRDb3RVS0VmM3JibXdBM28xSlBZMGlMbFdGTVJrNmdB"
    },
    "kind": "Secret",
    "metadata": {
        "annotations": {
            "kubernetes.io/service-account.name": "postgres-vault",
            "kubernetes.io/service-account.uid": "7309c23d-bccf-11e9-9fcc-0800275363d6"
        },
        "creationTimestamp": "2019-08-12T07:04:38Z",
        "name": "postgres-vault-token-ts9x4",
        "namespace": "default",
        "resourceVersion": "26320",
        "selfLink": "/api/v1/namespaces/default/secrets/postgres-vault-token-ts9x4",
        "uid": "730db7e6-bccf-11e9-9fcc-0800275363d6"
    },
    "type": "kubernetes.io/service-account-token"
}
```
</details>

取得した値を使って認証の設定を行います。これは Vault が Kubernetes に接続するための設定です。`kubernetes_host`で設定したエンドポイントに対して取得した`token_reviewer_jwt`で認証します。

```shell
$ vault write auth/kubernetes/config \
  token_reviewer_jwt="$SA_JWT_TOKEN" \
  kubernetes_host="https://$K8S_HOST:443" \
  kubernetes_ca_cert="$SA_CA_CRT"
```

サービスアカウントロールにアタッチされるロールを作ります。`default`ネームスペースの先ほど作った`postgres-vault`サービスアカウントを認可して、Vault 上で作った`postgres-policy`の権限を付与しています。

```shell
$ vault write auth/kubernetes/role/postgres \
    bound_service_account_names=postgres-vault \
    bound_service_account_namespaces=default \
    policies=postgres-policy \
    ttl=24h
```

つまりここで`postgres-vault`で認証されたクライアントに対して下記のように作った権限を与えるという意味です。

```hcl
path "database/creds/postgres-role" {
  capabilities = ["read"]
}
path "sys/leases/renew" {
  capabilities = ["create"]
}
path "sys/leases/revoke" {
  capabilities = ["update"]
}
```

### Temporarily の Pod で試す

アプリで利用する前に Temporarily の Pod を立てて一連の流れをテストしてみましょう。`postgres-vault`のサービスアカウントを設定した Pod を一つ起動してみます。

```
$ kubectl run tmp --rm -i --tty --serviceaccount=postgres-vault --image alpine
```

ログインできたら

* サービスアカウントトークンを fetch して
* Vault にログインし、
* Vault が発行したトークンを使って、
* Postgres のシークレットを利用して

みます。

Apline に必要なパッケージをインストールして、サービスアカウントトークンを取得して fetch します

```shell
$ apk update
$ apk add curl postgresql-client jq
$ KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

次にこれを利用して Kubernetes Auth Method を使って Vault にログインします。ログインには`auth/kubernetes/login`のエンドポイントを使います。

```shell
$ VAULT_K8S_LOGIN=$(curl --request POST --data '{"jwt": "'"$KUBE_TOKEN"'", "role": "postgres"}' http://vault.default.svc.cluster.local:8200/v1/auth/kubernetes/login)
```

ログイン情報を確認しておきましょう。トークンが発行され、認証されたクライアントに`postgres-policy`が割り当てられていることがわかります。

```shell
$ echo $VAULT_K8S_LOGIN | jq
```

<details><summary>出力結果</summary>

```json
{
  "request_id": "071f4939-26d5-ef37-0311-30dc64b804d7",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": null,
  "wrap_info": null,
  "warnings": null,
  "auth": {
    "client_token": "s.z79s26iFrza2LxvQBy6Xyscs",
    "accessor": "6T7Xt14PsltZZh1TLVffOePb",
    "policies": [
      "default",
      "postgres-policy"
    ],
    "token_policies": [
      "default",
      "postgres-policy"
    ],
    "metadata": {
      "role": "postgres",
      "service_account_name": "postgres-vault",
      "service_account_namespace": "default",
      "service_account_secret_name": "postgres-vault-token-ts9x4",
      "service_account_uid": "7309c23d-bccf-11e9-9fcc-0800275363d6"
    },
    "lease_duration": 86400,
    "renewable": true,
    "entity_id": "a2b587be-d518-c4f5-fff5-1ba0fecedbbe",
    "token_type": "service",
    "orphan": true
  }
}
```
</details>

`.auth.client_token`が Vault のトークンなのでこれを取得します。

```
$ X_VAULT_TOKEN=$(echo $VAULT_K8S_LOGIN | jq -r '.auth.client_token')
```

次に、このトークンを使って Vault の API をコールして Postgres のシークレットを生成しましょう。`database/creds/ROLE_NAME`がエンドポイントです。

```
$ POSTGRES_CREDS=$(curl --header "X-Vault-Token: $X_VAULT_TOKEN" http://vault.default.svc.cluster.local:8200/v1/database/creds/postgres-role)
```

確認します。

```
$ echo $POSTGRES_CREDS | jq

{
  "request_id": "d31b68f9-54f1-0ec2-8cc9-bbd31fc7d3f5",
  "lease_id": "database/creds/postgres-role/I0ImdJrSVDoQ7UlMLfWuHDaN",
  "renewable": true,
  "lease_duration": 3600,
  "data": {
    "password": "A1a-WfWDZtqLzuid4Ili",
    "username": "v-kubernet-postgres-qkxMrIGHABBwr40HJStX-1565595560"
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null
}
```

このユーザを使って Postgres を利用しています。

```
$ PGUSER=$(echo $POSTGRES_CREDS | jq -r '.data.username')
$ export PGPASSWORD=$(echo $POSTGRES_CREDS | jq -r '.data.password')
$ psql -h postgres-postgresql -U $PGUSER postgres -c 'SELECT * FROM pg_catalog.pg_tables;'
```

テーブルが表示され正しくシークレットが発行できることが確認できるはずです。

<kbd>
  <img src="https://miro.medium.com/max/700/1*qfojP76kbu-L7rYDMTx8JQ.png">
</kbd>

Pod から Vault のシークレットを使ってシークレットを発行する一連の手順を確認しました。

### 実際の Web アプリの Pod から利用してみる

次はいよいよ Rails のアプリから Vault を経由して Postgres を扱ってみます。

以下の Yaml を任意のディレクトリに作成してください。

<details><summary>vault-rails.yml</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault-dynamic-secrets-rails
  labels:
    app: vault-dynamic-secrets-rails
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vault-dynamic-secrets-rails
  template:
    metadata:
      labels:
        app: vault-dynamic-secrets-rails
    spec:
      serviceAccountName: postgres-vault
      initContainers:
        - name: vault-init
          image: everpeace/curl-jq
          command:
            - "sh"
            - "-c"
            - >
              KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token);
              curl --request POST --data '{"jwt": "'"$KUBE_TOKEN"'", "role": "postgres"}' http://vault.default.svc.cluster.local:8200/v1/auth/kubernetes/login | jq -j '.auth.client_token' > /etc/vault/token;
              X_VAULT_TOKEN=$(cat /etc/vault/token);
              curl --header "X-Vault-Token: $X_VAULT_TOKEN" http://vault.default.svc.cluster.local:8200/v1/database/creds/postgres-role > /etc/app/creds.json;
          volumeMounts:
            - name: app-creds
              mountPath: /etc/app
            - name: vault-token
              mountPath: /etc/vault
      containers:
        - name: rails
          image: gmaliar/vault-dynamic-secrets-rails:0.0.1
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
          resources:
            limits:
              memory: "150Mi"
              cpu: "200m"
          volumeMounts:
            - name: app-creds
              mountPath: /etc/app
            - name: vault-token
              mountPath: /etc/vault
        - name: vault-manager
          image: everpeace/curl-jq
          command:
            - "sh"
            - "-c"
            - >
              X_VAULT_TOKEN=$(cat /etc/vault/token);
              VAULT_LEASE_ID=$(cat /etc/app/creds.json | jq -j '.lease_id');
              while true; do
                curl --request PUT --header "X-Vault-Token: $X_VAULT_TOKEN" --data '{"lease_id": "'"$VAULT_LEASE_ID"'", "increment": 3600}' http://vault.default.svc.cluster.local:8200/v1/sys/leases/renew;
                sleep 3600;
              done
          lifecycle:
            preStop:
              exec:
                command:
                  - "sh"
                  - "-c"
                  - >
                    X_VAULT_TOKEN=$(cat /etc/vault/token);
                    VAULT_LEASE_ID=$(cat /etc/app/creds.json | jq -j '.lease_id');
                    curl --request PUT --header "X-Vault-Token: $X_VAULT_TOKEN" --data '{"lease_id": "'"$VAULT_LEASE_ID"'"}' http://vault.default.svc.cluster.local:8200/v1/sys/leases/revoke;
          volumeMounts:
            - name: app-creds
              mountPath: /etc/app
            - name: vault-token
              mountPath: /etc/vault
      volumes:
        - name: app-creds
          emptyDir: {}
        - name: vault-token
          emptyDir: {}
```
</details>

Apply します。

```shell
$ kubectl apply -f vault-rails.yml
```

Pod 名を取得しましょう。

```shell
$ kubectl get pod -l app=vault-dymanic-secrets-rails -o wide
```

Pod 名を引数に Port foward の設定を行います。

```shell
$ port-forward <POD_NAME_1> 3001:3000
$ port-forward <POD_NAME_2> 3002:3000
```

ブラウザでアクセスすると Postgres のユーザ名とパスワードが Pod ごとに発行されていることがわかるでしょう。

<kbd>
  <img src="https://miro.medium.com/max/700/1*1wq5AFgky7JDDsKM0EOdmg.png">
</kbd>

<kbd>
  <img src="https://miro.medium.com/max/700/1*kERi7ESQ6oWUeIEs-5f11A.png">
</kbd>

### 参考リンク
* [Kubernetes with Vault](https://www.vaultproject.io/docs/platform/k8s/index.html)
* [Kubernetes Auth Method](https://www.vaultproject.io/docs/auth/kubernetes.html)
* [Kubernetes Auth Method API](https://www.vaultproject.io/api/auth/kubernetes/index.html)
* [Helm Chart for Vault](https://github.com/hashicorp/vault-helm)
* [Helm Chart for Postgres](https://github.com/helm/charts/tree/master/stable/postgresql)
* [Sample App Blog](https://medium.com/@gmaliar/dynamic-secrets-on-kubernetes-pods-using-vault-35d9094d169)

---

## 多要素認証(Multi Factor Authentication)を試す

Table of Contents
=================

  * [Policy の設定を行う](#policyの設定を行う)
  * [TOTP 用のバーコードと URL を発行する](#totp用のバーコードとurlを発行する)
  * [Vault を利用して TOTP を発行する](#vaultを利用してtotpを発行する)
  * [Google Authenticator を利用して TOTP を発行する](#google-authenticatorを利用してtotpを発行する)
  * [参考リンク](#参考リンク)

HashiCorp Vault Enterprise では`Multi Factor Authentication(MFA)`
を利用して、Vault の API コール時に多要素での認証を設定することが可能です。

以下のような認証方式を入れることができます。

* TOTP
* Okta
* Duo
* PingID

今回は TOTP を使って時間制限付きのワンタイムパスワードを多要素認証として利用し、「Vault Token を作るための API の実行」に対して多要素認証を掛けたいと思います。

* GitHub で認証
* 認証されたユーザに Vault Token を作るための権限を付与
* API 実行時に`TOTP`を入力し実行

という流れです。

`Multi Factor Authentication`は Enterprise 版のみ有効な機能です。利用の際は[トライアルのライセンス](https://www.hashicorp.com/products/vault/trial/)や Entperprise の正式なライセンスで機能をアクティベーションする必要があります。

ライセンスのセットの仕方は[こちら](https://www.vaultproject.io/api-docs/system/license)を参考にしてみてください。

### Policy の設定を行う

多要素認証を扱うには Vault の Policy 設定に`mfa_methods`を付与します。`mfa_methods`で指定する値は多要素認証の方法を定義しますが、こちらを事前に定義します。

```sh
$ vault write sys/mfa/method/totp/my_totp \
    issuer=Vault \
    period=90 \
    algorithm=SHA256 \
    digits=8
```

このコマンドを実行することで TOTP MFA の設定が完了です。

* `issuer`は任意の発行者名
* `period`は TOTP の生存期間
* `algorithm`は TOTP を生成する際のアルゴリズム
* `digits`は TOTP の文字列数です。

これをポリシーの`mfa_methods`に下記のように指定します。

```hcl
path "auth/token/create" {
  capabilities = ["create"]
  mfa_methods  = ["my_totp"]
}
```

`auth/token/create`のエンドポイントに`write`処理を実行するための権限で`mfa_methods`として`my_totp`をセットしています。

以下のコマンドでセットしてみましょう。

```sh
$ vault policy write totp-policy -<<EOF
path "auth/token/create" {
  capabilities = [ "read", "list", "create", "update", "delete"]
  mfa_methods  = ["my_totp"]
}
EOF
```

### GitHub 認証の設定

GitHub の認証には`Organization`, `Team`と`GitHub用のAPI Token`が必要です。それぞれ事前に作成してください。

ここでは

* `Organization` = `hashicorp-japan`
* `Team` = `vault-token-creation`

としています。

以下のコマンドで GitHub 認証を有効化します。

```sh
$ ORG=<YOUR_ORG_NAME>
$ vault auth enable github
$ vault write auth/github/config organization=${ORG}
$ vault write auth/github/map/teams/vault-token-creation value=totp-policy
```

`hashicorp-japan`内の`vault-token-creation`に所属しているユーザが認証されると、先ほど作成した`totp-policy`が付与されます。

このポリシー先ほど設定した通り`auth/token/create`の`write`処理だけの権限を持ち、実行時に多要素認証を求められます。

それではログインしてみましょう。

```sh
$ vault login -method=github
```

GitHub のトークンを入力すると認証が成功し以下のように出力されるはずです。

```
GitHub Personal Access Token (will be hidden):
WARNING! The VAULT_TOKEN environment variable is set! This takes precedence
over the value set by this command. To use the value set by this command,
unset the VAULT_TOKEN environment variable or set it to the token displayed
below.

Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                    Value
---                    -----
token                  s.H4Nj8UqTXsZWGmWstkczVnMF
token_accessor         UtSrKvGDhHctNbpRCtdgu6BI
token_duration         768h
token_renewable        true
token_policies         ["default" "totp-policy"]
identity_policies      []
policies               ["default" "totp-policy"]
token_meta_org         hashicorp-japan
token_meta_username    tkaburagi
```

トークンが一つ作成され、ポリシーが設定セットされているはずです。このトークンをテストしてみましょう。

```console
$ VTOKEN=s.H4Nj8UqTXsZWGmWstkczVnMF
$ VAULT_TOKEN=${VTOKEN} vault read sys/mounts

Error listing secrets engines: Error making API request.

URL: GET http://127.0.0.1:8200/v1/sys/mounts
Code: 403. Errors:

* 1 error occurred:
	* permission denied

$ VAULT_TOKEN=${VTOKEN} vault write -f auth/token/create

Error writing data to http://127.0.0.1:8200/v1/auth/token/create: Error making API request.

URL: PUT http://127.0.0.1:8200/v1/http:/127.0.0.1:8200/v1/auth/token/create
Code: 403. Errors:

* 1 error occurred:
	* permission denied
```

いずれのリクエストも`permission denied`となるはずです。

* `sys/mounts`にはそもそも権限がないこと
* `auth/token/create`には権限があるが、MFA がポリシー上要求されているがセットされていないこと

が原因です。

次に TOTP の発行を有効にし MFA を使って API を実行してみましょう。

### TOTP 用のバーコードと URL を発行する

次に TOTP 用のバーコードと URL を発行します。

以下のコマンドで先ほど発行した Vault トークンの`entity_id`を取得し、`sys/mfa/method/totp/my_totp/admin-generate`のエンドポイントを実行し`barcode`, `url`を取得します。

```sh
$ vault write sys/mfa/method/totp/my_totp/admin-generate \
    entity_id=$(vault token lookup -format=json ${VTOKEN} | jq -r '.data.entity_id')
```

ここで出力された`barcode`や`url`は外部の TOTP コードの Gnerator にセットすることで TOTP を発行することができます。

`barcode`と`url`を保存し、まずは`barcode`を利用して`Google Authenticator`を利用する例を紹介します。


### Google Authenticator を利用して TOTP を発行する

まずは Google Authentiation を試してみます。AppStore もしくは Google Play からインストールをした上で実行してください。

```sh
$ base64 --decode <<< <TOTP_BARCODE> > barcode-totp.png
```

以下のような QR コードが生成されているはずです。

<kbd>
  <img src="https://blog-kabuctl-run.s3-ap-northeast-1.amazonaws.com/20200426/mfa-2.png">
</kbd>

これを Google Authenticator でスキャンすると、以下のように TOTP が発行されていることがわかるでしょう。

<kbd>
  <img src="https://blog-kabuctl-run.s3-ap-northeast-1.amazonaws.com/20200426/mfa-1.jpg">
</kbd>

この値をメモして API を再度実行してみます。MFA を利用する際は`-mfa my_totp:<TOTP>`のパラメータをセットし API を実行します。

```console
$ VAULT_TOKEN=${VTOKEN} vault write -mfa my_totp:83501485 -f auth/token/create

Key                  Value
---                  -----
token                s.5nWNLFCMX9ZAcfsdBHFo184L
token_accessor       rPHpz2xsLsEdGsof50enlXWe
token_duration       768h
token_renewable      true
token_policies       ["default" "totp-policy"]
identity_policies    []
policies             ["default" "totp-policy"]
```

トークンが発行できるはずです。試しに適当な TOTP をセットしてみます。

```console
$ VAULT_TOKEN=${VTOKEN} vault write -mfa my_totp:99999999 -f auth/token/create

Error writing data to auth/token/create: Error making API request.

URL: PUT http://127.0.0.1:8200/v1/auth/token/create
Code: 403. Errors:

* 1 error occurred:
	* permission denied
```

エラーとなり実行できないはずです。次に`my_totp:`に正しい値を入れて別のエンドポイントを実行してみましょう。

```console
$ VAULT_TOKEN=s${VTOKEN} vault write -mfa my_totp:60390043 -f sys/monts

Error writing data to sys/monts: Error making API request.

URL: PUT http://127.0.0.1:8200/v1/sys/monts
Code: 403. Errors:

* 1 error occurred:
	* permission denied
```

権限でセットした通り、正しい TOTP を持っていても`auth/token/create`以外のエンドポイントは実行できません。

### Vault を利用して TOTP を発行する

ここまで Google Autenticator を利用してきましたが、最後に Vault 自身を Generator として利用する方法を紹介します。

先ほど保存した`url`をコピーしましょう。

```sh
$ vault secrets enable totp
$ vault write totp/keys/my-key \
    url="<TOTP_URL>"
```

これで Vault 自体がキーを発行できるようになり、外部のサービスを使う必要がありません。

```console
$ vault read totp/code/my-key

Key     Value
---     -----
code    93346290
```

```console
$ VAULT_TOKEN=${VTOKEN} vault write -mfa my_totp:93346290 -f auth/token/create

Key                  Value
---                  -----
token                s.Ev5Gbtmwtvxb9QmTJYJ0UJ3T
token_accessor       KjhI3CBlnfJJunVYNOBRlHgw
token_duration       768h
token_renewable      true
token_policies       ["default" "totp-policy"]
identity_policies    []
policies             ["default" "totp-policy"]
```

同じように TOTP を利用することができました。

以上のように、Vault Enterprise の MFA 機能を利用することで特定のエンドポイントに対してより安全な多要素認証を入れることができます。

今回は TOTP の例でしたが、Okta などその他の認証基盤と連携させることも可能です。

### 参考リンク
* [Vault Enterprise MFA Support](https://www.vaultproject.io/docs/enterprise/mfa)
* [TOTP MFA](https://www.vaultproject.io/docs/enterprise/mfa/mfa-totp)
* [MFA API](https://www.vaultproject.io/api-docs/system/mfa)
* [TOTP MFA API](https://www.vaultproject.io/api-docs/system/mfa/totp)
* [Learn: GitHub Auth Method](https://learn.hashicorp.com/vault/getting-started/authentication)
* [TOTP Secret Engine](https://www.vaultproject.io/docs/secrets/totp)

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

## Azure のシークレットエンジンを試す

Azure シークレットエンジンではロールの定義に基づいた Azure の Service Principal を動的に発行することが可能です。Azure のキー発行のワークフローをシンプルにし、TTL などを設定することでよりセキュアに利用できます。

### Azure のセットアップ

[こちら](https://github.com/hashicorp-japan/vault-workshop/blob/master/assets/azure-guide.md)を参考に Azure のセットアップを行なって下さい。

### IAM ユーザの動的発行

まずシークレットエンジンを enable にします。

```shell
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault secrets enable azure
```

次に Vault が Azure の API を実行するために必要なキーを登録します。

```shell
export SUB_ID="***********"
export TENANT_ID="***********"
export CLIENT_ID="***********"
export CLIENT_SECRET="***********"

$ vault write azure/config \
        subscription_id="${SUB_ID}" \
        client_id="${CLIENT_ID}" \
        client_secret="${CLIENT_SECRET}" \
        tenant_id="${TENANT_ID}"
```

`subscription_id`, `client_id`, `client_secret`, `tenant_id`はご自身の環境に合わせたものに書き換えてください。ここでは必ずしも Azure の Admin ユーザを登録する必要はなく、ロールやユーザを発行できるユーザであれば大丈夫です。

次にロールを登録します。このロールが Vault から払い出されるユーザの権限と紐付きます。ロールは複数登録することが可能です。今回はまずは全てのリソースに対する Read Only のロールを作成しています。

```shell
$ vault write azure/roles/reader azure_roles=-<<EOF
    [
      {
        "role_name": "Reader",
        "scope": "/subscriptions/${SUB_ID}/resourceGroups/vault-resource-group"
      }
    ]
EOF
```

別端末を開いて`watch`コマンドでユーザのリストを監視します。

```console
$ export TENANT_ID="***********"
$ watch -n 1 'az ad sp list --query "[].{id:appId, tenant:appOwnerTenantId}" | grep -B 1 ${TENANT_ID}'

    "id": "4c4411ee-9654-4acf-b242-*******************",
    "tenant": "a67e8730-8fe4-453a-a239-e62d4df0a815"
--
    "id": "80343551-a8cf-494a-9b40-*******************",
    "tenant": "a67e8730-8fe4-453a-a239-e62d4df0a815"
--
```

>az cli にログイン出来ていない場合、以下のコマンドでログインしてください。
>
>```shell
>$  az login --service-principal \
>    -u "${CLIENT_ID}" \
>    -p "${CLIENT_SECRET}" \
>    --tenant "${TENANT_ID}"
>```

>Windows などで実行できない場合は手動で実行して下さい。

元の端末に戻り、ロールを使って Azure のシークレットを発行してみましょう。

```console
$ vault read azure/creds/reader

Key                Value
---                -----
lease_id           azure/creds/reader/OXL8Ua8oGnvQukcEU8taA3Ni
lease_duration     768h
lease_renewable    true
client_id          *******************
client_secret      *******************
```

この`watch`の出力結果を見るとユーザが増えていることがわかります。`lease_id`はあとで使うのでメモしておいてください。
`client_id`と`client_secret`もメモしておいて下さい。

```
    "id": "4c4411ee-9654-4acf-b242-*******************",
    "tenant": "a67e8730-8fe4-453a-a239-e62d4df0a815"
--
    "id": "80343551-a8cf-494a-9b40-*******************",
    "tenant": "a67e8730-8fe4-453a-a239-e62d4df0a815"
--
    "id": "70788f51-a1ff-472e-8b74-*******************",
    "tenant": "a67e8730-8fe4-453a-a239-e62d4df0a815"
--
```

このユーザを使って動作を確認してみましょう。

`-u`: 先ほど生成した`client_id` 
`-p`: 先ほど生成した`client_secret`

をそれぞれ入力します。

```shell
$  az login --service-principal \
 -u "*****" \
 -p "*****" \
 --tenant ${TENANT_ID}
```

新しい端末を立ち上げて以下のコマンドを実行します。

```console
$ az network vnet list

$ az storage account list

$ az vm list

$ az storage account create -n $(openssl rand -hex 10) -g vault-resource-group
The client '****************' with object id '****************' does not have authorization to perform action 'Microsoft.Storage/storageAccounts/write' over scope '/subscriptions/****************/resourceGroups/vault-resource-group/providers/Microsoft.Storage/storageAccounts/ea00f5eb2a503375d265' or the scope is invalid. If access was recently granted, please refresh your credentials.
```

Role に設定した通り Read のオペレーションを行うことができますが、Create などその他の操作を行うことが出来ないことがわかるでしょう。

### Revoke を試す

az cli のユーザを元のユーザに切り替えておきます。`watch`を実行している端末を一度`ctrl+c`で抜けて以下のコマンドでユーザでログインをし直します。

```shell
$  az login --service-principal \
    -u "${CLIENT_ID}" \
    -p "${CLIENT_SECRET}" \
    --tenant "${TENANT_ID}"
```
```console
$ watch -n 1 'az ad sp list --query "[].{id:appId, tenant:appOwnerTenantId}" | grep -B 1 ${TENANT_ID}'

    "id": "4c4411ee-9654-4acf-b242-*******************",
    "tenant": "a67e8730-8fe4-453a-a239-e62d4df0a815"
--
    "id": "80343551-a8cf-494a-9b40-*******************",
    "tenant": "a67e8730-8fe4-453a-a239-e62d4df0a815"
--
    "id": "70788f51-a1ff-472e-8b74-*******************",
    "tenant": "a67e8730-8fe4-453a-a239-e62d4df0a815"
--
```

シンプルな手順でユーザが発行できることがわかりましたが、次は Revoke(破棄)を試してみます。Revoke にはマニュアルと自動の 2 通りの方法があります。

まずはマニュアルでの実行手順です。`vault read azure/creds/reader`を実行した際に発行された`lease_id`をコピーしてください。

```shell
$ vault lease revoke azure/creds/reader/<LEASE_ID>
```

`watch`の実行結果を見ると一つユーザが削除されているでしょう。

```
    "id": "4c4411ee-9654-4acf-b242-*******************",
    "tenant": "a67e8730-8fe4-453a-a239-e62d4df0a815"
--
    "id": "80343551-a8cf-494a-9b40-*******************",
    "tenant": "a67e8730-8fe4-453a-a239-e62d4df0a815"
--
```

次に自動 Revoke です。デフォルトでは TTL が`765h`になっています。これは数分にしてみましょう。

```shell
vault write azure/roles/reader ttl=2m max_ttl=10m azure_roles=-<<EOF
    [
      {
        "role_name": "Reader",
        "scope": "/subscriptions/${SUB_ID}/resourceGroups/vault-resource-group"
      }
    ]
EOF
```

```console
$ vault read azure/roles/reader
Key                      Value
---                      -----
application_object_id    n/a
azure_groups             <nil>
azure_roles              [map[role_id:/subscriptions/6343b729-dfc6-4798-898b-b8eb9c9f4afb/providers/Microsoft.Authorization/roleDefinitions/acdd72a7-3385-48ef-bd42-f606fba81ae7 role_name:Reader scope:/subscriptions/6343b729-dfc6-4798-898b-b8eb9c9f4afb/resourceGroups/vault-resource-group]]
max_ttl                  10m
ttl                      2m
```

それではこの状態でユーザを発行します。

```console
$ vault read azure/creds/reader
Key                Value
---                -----
lease_id           azure/creds/reader/ct6a4XdSvmiV1d9zcfAVy2cS
lease_duration     2m
lease_renewable    true
client_id          ****************
client_secret      ****************
```

`watch`の実行結果を見るとユーザが増えています。今度は 2 分後にこのユーザは自動で削除されます。

```
    "id": "4c4411ee-9654-4acf-b242-*******************",
    "tenant": "a67e8730-8fe4-453a-a239-e62d4df0a815"
--
    "id": "80343551-a8cf-494a-9b40-*******************",
    "tenant": "a67e8730-8fe4-453a-a239-e62d4df0a815"
--
    "id": "70788f51-a1ff-472e-8b74-*******************",
    "tenant": "a67e8730-8fe4-453a-a239-e62d4df0a815"
--
```

2 分後、再度見てみるとユーザが削除されていることがわかるでしょう。

```json
    "id": "4c4411ee-9654-4acf-b242-*******************",
    "tenant": "a67e8730-8fe4-453a-a239-e62d4df0a815"
--
    "id": "80343551-a8cf-494a-9b40-*******************",
    "tenant": "a67e8730-8fe4-453a-a239-e62d4df0a815"
--
```

自動 Revoke には今回のように`ttl`で時間として指定したり`user_limit`というパラメータで回数で指定し、その回数使用されたら自動で破棄することなども可能です。

ここまで試したように Vault では以下のようなことが可能となり、安全に Azure のシークレットを扱うことが出来ます。

1. 発行するシークレットを Vault 経由で動的に発行が可能
2. 発行する際に必要な権限のみを付与してクライアントに提供することが可能
3. TTL や user_limit を設定し動的にシークレットを Revoke することが可能

このような機能で設定ファイルに静的に記述したり、長い間同じキーを複数クライアントで使い続けるなどの危険な運用を回避することが出来ます。

### 参考リンク
* [Azure Secret Engine](https://www.vaultproject.io/docs/secrets/azure/index.html)
* [Azure Secret Engine API](https://www.vaultproject.io/api/secret/azure/index.html)

---

## GCP のシークレットエンジンを試す

GCP シークレットエンジンではロールセットの定義に基づいた GCP のキーを動的に発行することが可能です。GCP のキー発行のワークフローをシンプルにし、TTL などを設定することでよりセキュアに利用できます。

サポートしているクレデンシャルタイプは下記の三つです。

* Service Account Key
* Token

このハンズオンでは GCP アカウントが必要です。[こちら](https://cloud.google.com/free/)からアカウントを作成していください。

### セットアップ

GCP シークレットエンジンを扱うためのセットアップを行います。

まずは Service Account の作成です。この Service Account は Vault に登録するためのもので、この Servie Account のキーを利用して実際にプロジェクト等で利用するキーを発行します。

GCP のコンソールにログインして、`Navigation Menu`から`IAM&Admin` -> `Service accounts`と進んでください。

`CREATE SERVICE ACCOUNT`をクリックして、任意の名前を入力したら`CREATE`してください。`Select a role`で`Project` -> `Owner`を選んでください。

`CONTIUNE`で進んだら、`CREATE KEY`で JSON のキーを発行します。**このキーはプロジェクトの Owner の権限を持つので絶対に外部に漏らさないように保管してください。**

次に API を有効化します。Vault は上記で発行したキーを使って GCP のシークレットを払い出すわけですがそのためには`IAM`と`Cloud Resource Manager`の API を有効にする必要があります。

[https://console.developers.google.com/apis/dashboard](https://console.developers.google.com/apis/dashboard)にアクセスして`+ ENABLE APIS AND SERVICES`をクリックします。

次の画面で検索ボックスから`IAM`, `Cloud Resource Mananger`と`Compute`を探してそれぞれ`Enable`ボタンで有効化します。

次に Vault のセットアップです。以下のコマンドで GCP シークレットエンジンを有効化しましょう。

```shell
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault secrets enable gcp
```

最後に`gcloud`コマンドをインストールします。[こちら](https://cloud.google.com/sdk/)からインストールしてください。

これでセットアップは完了です。この後ロールセットに基づいた GCP のキーを動的に生成していきます。

### Vault の設定

次に Vault 側の設定です。Vault 側には

* GCP API を実行してキーを発行するための Vault 側へのキーの登録
* Vault が発行するキーに付与する権限

の設定が必要です。

まずは先ほど発行した Service Account のキーを Vault のコンフィグとして登録します。`KEY_JSON.json`は自身のファイル名に置き換えてください。


```shell
$ vault write gcp/config credentials=@KEY_JSON.json
```

Vault は内部的にこの鍵を使って GCP の API を実行して動的に新しいキーを発行して行きます。その際、発行するキーにどのような権限を与えるかを指定する必要があります。その設定を`gcp/roleset`で定義します。

`project`は`Project ID`を入力します。

> Project ID が分からない方は
> `gloud auth login`コマンドを実行してログインしてください。
> ログインが成功するとターミナル上に ID が表示されます。


`secret_type`には`service_account_key`もしくは`token`のいずれかを入力します。

ここでは Service Account を使ってみましょう。`bindings`は権限の設定です。まずは自身のプロジェクトに対して`viewer`の権限を設定してみましょう。

```shell
$ cat << EOF > mybindings.hcl
resource "//cloudresourcemanager.googleapis.com/projects/<PROJECT_ID>" {
  roles = ["roles/viewer"]
}
EOF
```

これを`gcp/roleset/ROLE_NAME`の`bindings`で指定します。ここでは`pj-viewer`という名前でロールセットを作成します。

```shell
$ vault write gcp/roleset/pj-viewer \
    project="peak-elevator-237302" \
    secret_type="service_account_key"  \
    bindings=@mybindings.hcl
```

作成したロールセットの一覧は以下のように確認します。

```console
$ vault list gcp/rolesets
Keys
----
pj-viewer
```

内容を確認したい時は read します。

```console
$ vault read gcp/roleset/pj-viewer
Key                      Value
---                      -----
bindings                 map[//cloudresourcemanager.googleapis.com/projects/peak-elevator-237302:[roles/viewer]]
project                  peak-elevator-237302
secret_type              service_account_key
service_account_email    vaultpj-viewer-1575172760@peak-elevator-237302.iam.gserviceaccount.com
```

これ以降`pj-viewer`を指定してキーを作成すると`bindings`に定義された権限のキーが発行されます。

これで Vault 側の設定は完了です。

### キーを発行する

それでは実際にキーを発行してみましょう。

```console
$ vault read -format=json gcp/key/pj-viewer | jq -r '.data.private_key_data' > gcp.key.encoded
```

`gcp.key.encoded`にエンコードされたキーが出力されているでしょう。

```console
$ base64 -D gcp.key.encoded > gcp.key
```

gcp.key の中身を確認してください。これでキーが作成されました。

このキーを使ってログインしてみましょう。

```shell
$ gcloud auth activate-service-account  --key-file=gcp.key
```

ログインしたら権限を試してみましょう。

```shell
$ gcloud iam service-accounts list

$ gcloud compute disk-types list

$ gcloud compute disk-types describe local-ssd
```

これらは`viewer`の権限で実行することが出来るでしょう。一方以下のコマンドはエラーが発生するはずです。

```console
$ gcloud iam service-accounts create auser-vault-handson
ERROR: (gcloud.iam.service-accounts.create) User [vaultpj-viewer-1575172760@peak-elevator-237302.iam.gserviceaccount.com] does not have permission to access project [peak-elevator-237302] (or it may not exist): Permission iam.serviceAccounts.create is required to perform this operation on project projects/peak-elevator-237302.
```

### 動的なシークレットを試す

ここまででキーを発行してきましたが、Vault の特徴の一つは動的なシークレット管理です。Vault から発行するシークレットは全て TTL を付与します。

デフォルトだと 768h の有効期限ですが、最低限の期間に設定し、それ以降は無効にすることがベストです。まずは準備をします。

```console
$ gcloud iam service-accounts list
NAME                                                                EMAIL                                                                        DISABLED
admin                                                               
Service account for Vault secrets backend role set pj-viewer        vaultpj-viewer-1575172760@peak-elevator-237302.iam.gserviceaccount.com       False
```

`vaultpj-viewer-***@***iam.gserviceaccount.com`をコピーして別のターミナルを立ち上げ、以下のコマンドでキーのリストを監視します。

```console
$ watch -n 1 iam service-accounts keys list --iam-account=vaultpj-viewer-***@***iam.gserviceaccount.com
KEY_ID                                    CREATED_AT            EXPIRES_AT
10c1eec7578c74dc06d1312613fe02bc4dc0ebe3  2019-12-01T03:59:28Z  9999-12-31T23:59:59Z
```

#### 手動でのシークレット破棄を試す

次は Revoke(破棄)を試してみます。Revoke にはマニュアルと自動の 2 通りの方法があります。

まずは手動を試してみます。先ほどと同様にシークレットを発行します。`lease_id`はあとで使うのでメモしておいてください。これを使ってシークレットの`renew`, `revoke`などのライフサイクルを管理します。

```console
$ vault read gcp/key/pj-viewer
Key                 Value
---                 -----
lease_id            gcp/key/pj-viewer/1UC2B3DckozLyNHX5K4Ks4h9
lease_duration      768h
lease_renewable     true
key_algorithm       KEY_ALG_RSA_2048
key_type            TYPE_GOOGLE_CREDENTIALS_FILE
private_key_data
```

`watch`の内容を見るとキーが発行されているでしょう。

```
KEY_ID                                    CREATED_AT            EXPIRES_AT
10c1eec7578c74dc06d1312613fe02bc4dc0ebe3  2019-12-01T03:59:28Z  9999-12-31T23:59:59Z
a07f4979f3e8a40350418325002b1956cc46cf48  2019-12-01T04:02:58Z  2029-11-28T04:02:58Z
```

これを削除してみます。先ほどメモした Lease ID を引数に、`revoke`コマンドを実行するだけです。

```shell
$ vault lease revoke <LEASE_ID>
```

`watch`の内容を見るとキーが破棄されていることがわかります。

```
KEY_ID                                    CREATED_AT            EXPIRES_AT
10c1eec7578c74dc06d1312613fe02bc4dc0ebe3  2019-12-01T03:59:28Z  9999-12-31T23:59:59Z
```

次に TTL に基づいた自動での破棄を試してみます。

#### 自動でのシークレット破棄を試す

`watch`の出力はそのままにしておいてください。TTL の設定をしてみましょう。

```console
$ vault write gcp/config ttl=2m max_ttl=10m
$ vault read gcp/config

Key        Value
---        -----
max_ttl    10m
ttl        2m
```

TTL を 2 分に設定しました。`max_ttl`は`renew`というオペレーションで延長できる最大の有効期限です。

```console
$ vault read gcp/key/pj-viewer
Key                 Value
---                 -----
lease_id            gcp/key/pj-viewer/XWUOnUut8QYauHiBRSXHXs9o
lease_duration      2m
lease_renewable     true
key_algorithm       KEY_ALG_RSA_2048
key_type            TYPE_GOOGLE_CREDENTIALS_FILE
private_key_data
```

`watch`の端末を見るとキーが一つ発行されていることがわかるでしょう。

```console
KEY_ID                                    CREATED_AT            EXPIRES_AT
10c1eec7578c74dc06d1312613fe02bc4dc0ebe3  2019-12-01T03:59:28Z  9999-12-31T23:59:59Z
3aeebad8ea118e648b909a1ebefb80205e9df29e  2019-12-01T04:00:52Z  9999-12-31T23:59:59Z
```

このキーは 2 分後に Vault から自動で削除されます。2 分後ターミナルを確認してください。

```console
KEY_ID                                    CREATED_AT            EXPIRES_AT
10c1eec7578c74dc06d1312613fe02bc4dc0ebe3  2019-12-01T05:28:30Z  2029-11-28T05:28:30Z
```

ユーザが削除されていることがわかるでしょう。

### 参考リンク
* [GCP Secret Engine](https://www.vaultproject.io/docs/secrets/gcp/index.html)
* [Role Set Bindings](https://www.vaultproject.io/docs/secrets/gcp/index.html#roleset-bindings)
* [GCP Secret Engine API](https://www.vaultproject.io/api/secret/gcp/index.html)

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

## SSH シークレットエンジンを使ってワンタイム SSH パスワードを利用する

Vault の SSH シークレットエンジンでは SSH を使ったマシンへのアクセスのセキュアな認証認可を提供します。Vault を利用することで複雑な SSH クレデンシャルのライフサイクル管理のワークフローをよりシンプルにセキュアに実施できます。SSH シークレットエンジンの主な機能は以下の二つです。

* Singed SSH Certificates
* One-time SSH Paswords

ここでは両方を簡単に試してみます。

### Vagrant で VM を起動

Vagrant を使って Ubuntu OS の VM を一つ起動してみましょう。適当なディレクトリを作ります。

```shell
$ mkdir -p ~/vagrant/ubuntu
$ cd ~/vagrant/ubuntu
$ vagrant box add ubuntu14.04 https://cloud-images.ubuntu.com/vagrant/trusty/current/trusty-server-cloudimg-amd64-vagrant-disk1.box
$ vagrant init ubuntu14.04
```

`Vagrantfile`の以下の行のコメントは外してください。

```
# config.vm.network "public_network"
```

```console
$ vagrant up

Bringing machine 'default' up with 'virtualbox' provider...
==> default: Importing base box 'ubuntu14.04'...
==> default: Matching MAC address for NAT networking...
==> default: Setting the name of the VM: ubuntu_default_1564553548030_13754
==> default: Clearing any previously set forwarded ports...
==> default: Clearing any previously set network interfaces...
==> default: Available bridged network interfaces:
1) en0: Wi-Fi (Wireless)
2) ap1
3) p2p0
4) awdl0
5) en2: Thunderbolt 2
6) en4: Thunderbolt 4
7) en1: Thunderbolt 1
8) en3: Thunderbolt 3
9) bridge0
10) en5: USB Ethernet(?)
==> default: When choosing an interface, it is usually the one that is
==> default: being used to connect to the internet.
    default: Which interface should the network bridge to? 1
==> default: Preparing network interfaces based on configuration...
    default: Adapter 1: nat
    default: Adapter 2: bridged
==> default: Forwarding ports...
    default: 22 (guest) => 2222 (host) (adapter 1)
==> default: Booting VM...
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 127.0.0.1:2222
    default: SSH username: vagrant
    default: SSH auth method: private key


    default:
    default: Vagrant insecure key detected. Vagrant will automatically replace
    default: this with a newly generated keypair for better security.
    default:
    default: Inserting generated public key within guest...
    default: Removing insecure key from the guest if it's present...
    default: Key inserted! Disconnecting and reconnecting using new SSH key...
==> default: Machine booted and ready!
==> default: Checking for guest additions in VM...
    default: The guest additions on this VM do not match the installed version of
    default: VirtualBox! In most cases this is fine, but in rare cases it can
    default: prevent things such as shared folders from working properly. If you see
    default: shared folder errors, please make sure the guest additions within the
    default: virtual machine match the version of VirtualBox you have installed on
    default: your host and reload your VM.
    default:
    default: Guest Additions Version: 4.3.40
    default: VirtualBox Version: 6.0
==> default: Configuring and enabling network interfaces...
==> default: Mounting shared folders...
    default: /vagrant => /Users/kabu/vagrant/ubuntu
```

起動しました。今後この VM のことを「ホスト」、ローカルマシンのことを「クライアント」と呼びます。

### Vault の起動

ローカルに立ち上がっている Vault とホストを通信させるため、Vault のコンフィグを以下のように変更し再起動します。`Ctrl+C`で Vault の端末を止めて以下のようにファイルを変更してください。

```hcl
storage "file" {
   path = "/path/to/vault-oss-data"
}

listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = 1
}

listener "tcp" {
  address     = "192.168.11.2:8200"
  tls_disable = 1
}

api_addr = "http://192.168.11.2:8200"

ui = true
```

`192.168.11.2:8200`の IP はぞれぞれの環境に合わせてください。これで再度起動します。

```shell
$ vault server -config=path/to/vault-local-config.hcl start
```

### Singed SSH Certificates

#### クライアントキーサイン

公開鍵認証の場合サーバに対する秘密鍵を共有したり、キーペアをそれぞれに持たせて運用が複雑化する課題がありますが CA 認証によりこれらを解決でき、かつ Vault が CA として機能することで非常にシンプルなワークフローで実現できます。公開鍵認証と CA 認証の一般的な流れは下記の通りです。

[公開鍵認証]
<kbd>
  <img src="https://github-image-tkaburagi.s3-ap-northeast-1.amazonaws.com/vault-workshop/puflow.png">
</kbd>

[CA 認証]
<kbd>
  <img src="https://github-image-tkaburagi.s3-ap-northeast-1.amazonaws.com/vault-workshop/caflow-with-number.png">
</kbd>

まずいつものように Vault の SSH シークレットエンジンを有効化しましょう。

```shell
vault secrets enable -path=ssh ssh
```

#### CA の設定

図の①の手順です。

まず、Vault を CA として設定します。`generate_signing_key`を指定することでこのタイミングでサイン用のキーペアーを発行できます。既存にある場合は`private_key`, `public_key`のパラメータの引数としてセットします。

ここでは`generate_signing_key`を付与して生成してみます。

```shell
vault write ssh/config/ca generate_signing_key=true
```

Public Key のみが出力されるでしょう。

次に Vault 上に、SSH でログインするユーザに与える権限と認証のタイプを指定します。

```shell
$ export VAULT_ADDR="http://192.168.11.2:8200"
```

```
$ vault write ssh/roles/my-role -<<"EOH"
{
  "allow_user_certificates": true,
  "allowed_users": "ubuntu",
  "default_extensions": [
    {
      "permit-pty": ""
    }
  ],
  "key_type": "ca",
  "default_user": "ubuntu",
  "ttl": "30m0s"
}
EOH
```

`key_type`は他に`otp`と`dynamic`を指定することができ、`otp`はこのあと扱います。

#### サイン用公開鍵の配布

図の②の手順です。

次に、この公開鍵をターゲットとなるホスト(VM)の SSH コンフィグレーションに配布します。(図の②)

ホストに`vagrant ssh`でログインします。以下のコマンドで Vault で生成した public_key を取得します。

```shell
$ sudo curl -o /etc/ssh/trusted-user-ca-keys.pem http://192.168.11.2:8200/v1/ssh/public_key
```

`192.168.11.2`はローカルの Vault のアドレスです。

```console
$ cat /etc/ssh/trusted-user-ca-keys.pem
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABA.....
```

これを`sshd_config`に配備します。

```console
$ sudo sed -i '$a TrustedUserCAKeys /etc/ssh/trusted-user-ca-keys.pem' /etc/ssh/sshd_config
$ sudo service ssh restart
```

今回は Vault サーバとクライアントがローカルマシンで同居しているのでわかりづらいかもしれませんが、Vault を起動しているローカルマシンで以下を実行します。

#### クライアント側の設定

クライアントでキーペアを作成します。図の③の手順です。
```shell
$ ssh-keygen -t 
```

生成された公開鍵を Vault(CA)に渡してサインのリクエストをし、サイン済みの公開鍵(証明書)を発行します。

```shell
$ vault write -field=signed_key ssh/sign/my-role \
    public_key=@$HOME/.ssh/id_rsa.pub > signed-cert.pub
```

出力される`signed_key`がそれに当たります。

証明書の内容を確認してみましょう。

```shell
ssh-keygen -Lf ~/.ssh/signed-cert.pub
```

`Valid`の欄が Role で指定した 30mins の TTL になっているはずです。

最後にこれを使って SSH でサーバにアクセスしてみましょう。

```shell
ssh -i signed-cert.pub -i ~/.ssh/id_rsa ubuntu@192.168.11.9
```

`ubuntu`ユーザでログインできるはずです。

この証明書は 30 分後に無効となりログインが不可能になります。また、`revoke`を行うことで明示的に無効にできます。

このように CA 認証を簡単に利用することができるため、環境やチームが拡大するにつれて複雑化する運用を効率化することができます。

### One-time SSH Paswords

次は SSH シークレットエンジンの二つ目のユースケースであるワンタイムパスワード(OTP)を扱います。

クラウド化が進むにつれ、オンラインセキュリティの重要度は増しています。アタッカーが VM へのアクセスを入手するとあらゆるデータやサービスへのアクセスを許すこととなり、SSH キーの扱いは非常に重要です。

ワークフローは下記の通りです。

* 認証された Vault のクライアントが Vault に対して OTP の発行を依頼
* Vault により OTP の発行と提供
* クライアントは SSH コネクションの間その OTP を利用しホストに接続
* クライアントがホストに接続したことを確認すると Vault がそのパスワードを消去

まずはホスト側の準備をします。Vault の OTP を利用するには全てのホスト側で[`vault-ssh-helper`](https://github.com/hashicorp/vault-ssh-helper)をインストールする必要があります。

ホストに`vagrant ssh`でログインして以下のコマンドを実行します。

```shell
$ wget https://releases.hashicorp.com/vault-ssh-helper/0.1.4/vault-ssh-helper_0.1.4_linux_amd64.zip
$ sudo unzip -q vault-ssh-helper_0.1.4_linux_amd64.zip -d /usr/local/bin
$ sudo chmod 0755 /usr/local/bin/vault-ssh-helper
$ sudo chown root:root /usr/local/bin/vault-ssh-helper
```

`/etc/vault-ssh-helper.d/config.hcl`のファイルを作り、下記のように編集します。

```hcl
vault_addr = "http://<LOCAL_VAULT_ADDR>:8200"
ssh_mount_point = "ssh"
allowed_roles = "*"
```

`/etc/pam.d/sshd`を編集します。変更部分は`Standard Un*x authentication.`から数行ですが、ミスを防ぐために[こちらを参照して](https://github.com/tkaburagi/vault-configs/blob/master/sshd)全て上書きしてください。

次に`/etc/ssh/sshd_config`の最後の行に以下を追加して sshd を再起動します。

```
ChallengeResponseAuthentication yes
PasswordAuthentication no
UsePAM yes
```

```shell
$ sudo service ssh restart
```

次に Vault 側の設定です。

先ほど有効にした`ssh`エンドポイントを利用します。`ssh/roles/<NAME>`のエンドポイントでロールを作ります。

```shell
$ vault write ssh/roles/otp_key_role key_type=otp \
        default_user=ubuntu \
        cidr_list=0.0.0.0/0
```

次にクライアントトークンに紐付けるポリシーを設定します。クラアイントは上記で定義したロールの OTP を発行するだけなので、`ssh/roles/otp_key_role`に対する権限だけあれば OK です。

`vault-ssh-otp-policy.hcl`のファイルを作り下記のように編集します。

```hcl
path "ssh/creds/otp_key_role" {
  capabilities = [ "create","update","read" ]
}
```

```console
$ vault policy write ssh-otp-client-policy path/to/vault-ssh-otp-policy.hcl
$ vault token create -policy=ssh-otp-client-policy -ttl=15m

Key                  Value
---                  -----
token                s.9nifyTTs49Mu73HMZffWBFVU
token_accessor       fkIZ2hWU0dMyQ2CbgaK55xfk
token_duration       15m
token_renewable      true
token_policies       ["default" "ssh-otp-client-policy"]
identity_policies    []
policies             ["default" "ssh-otp-client-policy"]
```

ここからはクライアントの手順です。クライアントは上記のトークンを使って、OTP の発行を Vault に依頼します。

```console
$ VAULT_TOKEN=s.9nifyTTs49Mu73HMZffWBFVU vault write ssh/creds/otp_key_role ip=<HOST's IP>

Key                Value
---                -----
lease_id           ssh/creds/otp_key_role/HmJV2vzyGf7bT2Mh6oB7qeyo
lease_duration     768h
lease_renewable    false
ip                 192.168.11.9
key                49ae44c0-298f-02d2-c8a3-2c1d8fbaee1c
key_type           otp
port               22
username           ubuntu
```

ubuntu ユーザ用の OTP が発行されました。これでログインしてみます。`key`がパスワードです。

```console
$ ssh ubuntu@192.168.11.9
ubuntu@192.168.11.9's password: 49ae44c0-298f-02d2-c8a3-2c1d8fbaee1c

Welcome to Ubuntu 14.04.6 LTS (GNU/Linux 3.13.0-170-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

  System information as of Thu Aug  1 03:02:18 UTC 2019

  System load:  0.05              Processes:           79
  Usage of /:   3.6% of 39.34GB   Users logged in:     1
  Memory usage: 27%               IP address for eth0: 10.0.2.15
  Swap usage:   0%                IP address for eth1: 192.168.11.9

  Graph this data and manage this system at:
    https://landscape.canonical.com/

New release '16.04.6 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Thu Aug  1 03:02:18 2019 from 192.168.11.2
ubuntu@vagrant-ubuntu-trusty-64:~$ whoami
ubuntu
ubuntu@vagrant-ubuntu-trusty-64:~$ exit
```

`exit`を実行して抜けてみましょう。再度ログインをしてみます。

```console
$ ssh ubuntu@192.168.11.9
ubuntu@192.168.11.9's password: 49ae44c0-298f-02d2-c8a3-2c1d8fbaee1c
Permission denied, please try again.
ubuntu@192.168.11.9's password: 49ae44c0-298f-02d2-c8a3-2c1d8fbaee1c
Permission denied, please try again.
```

Vault により無効化されました。

### 参考リンク
* [SSH Secret Engine](https://www.vaultproject.io/docs/secrets/ssh/index.html)
* [API Document](https://www.vaultproject.io/api/secret/ssh/index.html)
* [Vault SSH Helper](https://github.com/hashicorp/vault-ssh-helper)
* [公開鍵認証と CA 認証](http://kontany.net/blog/?p=211)

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
