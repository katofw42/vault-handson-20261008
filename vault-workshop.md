# HashiCorp Vault Workshop (AWS ハンズオン)

[Vault](https://www.vaultproject.io/) は HashiCorp が中心に開発をするシークレット管理ツールです。Vault を利用することで既存の Static なシークレット管理のみならず、クラウド、データベースや SSH などの様々なシークレットを動的に発行することができます。Vault はマルチプラットフォームでかつ全ての機能を HTTP API で提供しているため、環境やクライアントを問わず利用することができます。

本ドキュメントは、Vault を AWS 環境で一通り使い倒すことを目的とした一本道のハンズオンです。Vault のセットアップから始まり、テナントと権限の設計、アプリケーションからの認証、シークレットの読み書き、AWS の動的クレデンシャル発行、Terraform 連携、そして Day2 の運用までを順に扱います。各セクションは前のセクションの状態を引き継ぐ前提で記述しているため、できるだけ順番に進めることをお勧めします。

> このハンズオンは **Amazon Linux 2023** 上の **Vault Enterprise (v2.1.1)** を前提にしています。Namespace や Sentinel、Secret Sync といった Vault Enterprise の機能を利用するため、有効な Enterprise ライセンスが必要です。コマンドや出力例はすべてこの環境で実際に確認したものを掲載しています。

## Pre-requisite

* 環境
	* Amazon Linux 2023 の動作する環境 (本ハンズオンでは EC2 インスタンスを使用)

* ソフトウェア
	* Vault Enterprise v2.1.1 (本ハンズオンの手順でインストールします)
	* Terraform


* ライセンス / クレデンシャル
	* Vault Enterprise ライセンス (`vault.hclic`)
	* AWS (IAM ユーザまたはロールを作成できる権限)

## お勧めの進め方

本ハンズオンは以下の流れで構成されています。上から順に進めてください。

1. [Vault セットアップ](#vault-セットアップ)
	- [Vault のインストール](#vault-のインストール)
	- [Enterprise ライセンスの配置](#enterprise-ライセンスの配置)
	- [Vault のコンフィグレーション](#vault-のコンフィグレーション)
	- [Vault の初期化処理 (init / unseal)](#vault-の初期化処理-init--unseal)
	- [seal を試す](#seal-を試す)
	- [Auto Unseal (参考手順)](#auto-unseal-参考手順)
	- [Audit Device を設定する](#audit-device-を設定する)
	- [各種シークレットエンジンの有効化](#各種シークレットエンジンの有効化)
2. [テナントと権限設計](#テナントと権限設計)
	- [Namespace でテナントを分離する](#namespace-でテナントを分離する)
	- [Policy を作成して割り当てる](#policy-を作成して割り当てる)
	- [Sentinel による制御](#sentinel-による制御)
	- [Vault への AWS 権限付与](#vault-への-aws-権限付与)
3. [アプリからの利用 (Auth Method)](#アプリからの利用-auth-method)
	- [AWS Auth](#aws-auth)
	- [AppRole](#approle)
4. [Static Secret Engine](#static-secret-engine)
	- [Vault CLI 経由での読み書き](#vault-cli-経由での読み書き)
	- [Vault API 経由での読み書き](#vault-api-経由での読み書き)
	- [バージョニング](#バージョニング)
	- [Secret Sync による AWS Secrets Manager への反映](#secret-sync-による-aws-secrets-manager-への反映)
5. [AWS Secret Engine](#aws-secret-engine)
	- [IAM ユーザの動的発行](#iam-ユーザの動的発行)
	- [ポリシーで TTL が異なるアクセスキー発行](#ポリシーで-ttl-が異なるアクセスキー発行)
	- [強制 Revoke](#強制-revoke)
6. [Terraform 連携](#terraform-連携)
	- [動的クレデンシャルによる apply](#動的クレデンシャルによる-apply)
	- [ephemeral リソースで state にシークレットを残さない](#ephemeral-リソースで-state-にシークレットを残さない)
7. [Day2 運用](#day2-運用)
	- [バックアップ (スナップショットの取得)](#バックアップ-スナップショットの取得)
	- [リストア (スナップショットからの復元)](#リストア-スナップショットからの復元)

## 目次

- [Vault セットアップ](#vault-セットアップ)
- [テナントと権限設計](#テナントと権限設計)
- [アプリからの利用 (Auth Method)](#アプリからの利用-auth-method)
- [Static Secret Engine](#static-secret-engine)
- [AWS Secret Engine](#aws-secret-engine)
- [Terraform 連携](#terraform-連携)
- [Day2 運用](#day2-運用)

---

## Vault セットアップ

ここではまず Vault のインストールと起動、`init` / `unseal` / `seal` といったライフサイクルの操作、クラウドの鍵管理サービスを使った Auto Unseal、監査ログを記録する Audit Device、そして以降のハンズオンで使うシークレットエンジンの有効化までを扱います。

このハンズオンは **Amazon Linux 2023** 上の **Vault Enterprise (v2.1.1)** を前提にしています。手元の作業端末からは SSH で Amazon Linux 2023 の EC2 インスタンスにログインし、その上で作業を進めてください。

### Vault のインストール

Amazon Linux 2023 では、HashiCorp の公式 yum リポジトリを追加すれば`dnf`一発で Vault Enterprise をインストールできます。まず EC2 にログインしてから、以下を実行します。

```console
$ sudo dnf install -y dnf-plugins-core
$ sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
$ sudo dnf install -y vault-enterprise
```

Community 版 (`vault`) ではなく **`vault-enterprise`** パッケージを指定している点に注意してください。特定のバージョンを固定したい場合は`vault-enterprise-2.1.1+ent-1`のようにバージョンを付けて指定します。

インストールが終わったら、バージョンを確認します。

```console
$ vault version
Vault v2.1.1+ent (99f338575c1f226671bd863309439596fac1f67e), built 2026-09-15T21:39:40Z
```

`+ent`が付いているものが Enterprise バイナリです。パッケージインストールでは、設定ファイルや systemd のサービス定義もあわせて配置されます。

* バイナリ: `/usr/bin/vault`
* 設定ファイル: `/etc/vault.d/vault.hcl`
* systemd ユニット: `vault.service` (実行ユーザは`vault`)

これでインストールは完了です。

### Enterprise ライセンスの配置

Vault Enterprise を起動するにはライセンスが必要です。ライセンスは文字列 (`02MV4UU4...`のような長い 1 行) で配布されるので、これを Vault が読み込めるファイルとして保存します。パッケージの設定ファイルでも参照されている `/etc/vault.d/vault.hclic` に置くのが標準的です。

配布されたライセンス文字列を、EC2 上で以下のように書き込みます。`<ここにライセンス文字列>` の部分を実際の値に置き換えてください。

```console
$ sudo tee /etc/vault.d/vault.hclic >/dev/null <<'EOF'
<ここにライセンス文字列>
EOF
$ sudo chown vault:vault /etc/vault.d/vault.hclic
$ sudo chmod 640 /etc/vault.d/vault.hclic
```

ライセンスの読み込ませ方には環境変数 (`VAULT_LICENSE` / `VAULT_LICENSE_PATH`) を使う方法もありますが、本ハンズオンでは次のコンフィグで`license_path`を指定する **ライセンスの自動ロード (autoloading)** を使います。配置したライセンスが正しく読み込まれているかは、後ほど起動後に`vault read sys/license/status`で確認します。

### Vault のコンフィグレーション

次に Vault のコンフィグレーションを作成します。Vault のコンフィグレーションは`HashiCorp Configuration Language`で記述し、パッケージインストールの場合は`/etc/vault.d/vault.hcl`を編集します。ここでは本ハンズオン向けに、以下の内容で上書きします。

```console
$ sudo tee /etc/vault.d/vault.hcl >/dev/null <<'EOF'
ui = true
disable_mlock = true

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-1"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = true
}

license_path = "/etc/vault.d/vault.hclic"

api_addr     = "http://127.0.0.1:8200"
cluster_addr = "http://127.0.0.1:8201"
EOF
```

ポイントを押さえておきましょう。

* `storage "raft"` — Vault 推奨の **Integrated Storage (Raft)** を使います。データは`/opt/vault/data`に保存されます (パッケージが用意するディレクトリです)。
* `disable_mlock = true` — **Vault 1.20 以降、Integrated Storage を使う場合は`disable_mlock`を`true`/`false`で明示的に指定することが必須**になりました。省略すると起動時に`disable_mlock must be configured 'true' or 'false'`というエラーで失敗します。
* `listener "tcp"` — ハンズオンを簡単にするため TLS を無効 (`tls_disable = true`) にして HTTP で待ち受けます。本番では必ず TLS を有効にしてください。
* `license_path` — 先ほど配置した Enterprise ライセンスを自動ロードします。

編集したら systemd サービスとして Vault を起動し、ブート時にも自動起動するよう有効化します。

```console
$ sudo systemctl enable --now vault
$ systemctl is-active vault
active
```

起動したら、CLI の接続先を環境変数で指定して状態を確認します。

```console
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault status
Key                     Value
---                     -----
Seal Type               shamir
Initialized             false
Sealed                  true
Total Shares            0
Threshold               0
Unseal Progress         0/0
Unseal Nonce            n/a
Version                 2.1.1+ent
Build Date              2026-09-15T21:39:40Z
Storage Type            raft
Removed From Cluster    false
HA Enabled              true
```

`Initialized`が`false`、`Sealed`が`true`になっています。つまり Vault は起動しているが、まだ初期化がされておらず、Seal 状態であるということです。Vault を利用するまでに`init`と`unseal`という処理が必要です。

### Vault の初期化処理 (init / unseal)

まずは初期化です。`-key-shares`で生成する Unseal Key の数、`-key-threshold`で unseal に必要なキーの数を指定します (ここではそれぞれ 5 と 3)。生成されるキーとトークンは二度と再表示されないため、必ずファイルに保存してください。

```console
$ vault operator init -key-shares=5 -key-threshold=3 > vault-keys.txt
$ cat vault-keys.txt
Unseal Key 1: OxRA2iGFNs2UAfsBfTGHfauDGaFJqJYhActQh4tSxW05
Unseal Key 2: 7ik3chhr73SZsYMa+z50BH16YhkR42h3fviSbWabzldA
Unseal Key 3: I5xKJ6XnXvWF1leS4hEt3fZzFK8PfeG2eF+v8Z84fPH8
Unseal Key 4: c+G6Ny66MCoH1XSD8d33vHsR+eSk/OXJsi/ylggxRdon
Unseal Key 5: NeoD92Of0Wfl9rMlk0lFzq37nQz8aLdTw9JtEzBoaiAO

Initial Root Token: hvs.BhwdXXr0BANd68gmzAEtMAkQ
```

init の処理をすると、Vault を`unseal`するためのキーと`Initial Root Token`が生成されます。Vault 1.10 以降、トークンは`hvs.`から始まる形式になっています。試しにこの状態でシークレット一覧を見ようとすると、`sealed`のためエラーになります。

```console
$ vault secrets list
Error listing secrets engines: Error making API request.

URL: GET http://127.0.0.1:8200/v1/sys/mounts
Code: 503. Errors:

* Vault is sealed
```

Vault では`sealed`という状態になっているといかに強力な権限のあるトークンを使ったとしてもいかなる操作も受け付けません。`unseal`の処理は`Unseal Key`を使います。

指定した通り 5 つのキーのうち 3 つのキーが集まると`unseal`されます。これは[シャミアの秘密鍵分散法](http://ohta-lab.jp/users/mitsugu/research/SSS/main.html)という仕組みで、1 人に全ての鍵を集中させないための設計です。5 つの`Unseal Key`の任意の 3 つを使ってみましょう。`vault operator unseal`コマンドを 3 度実行します (引数を省略すると対話的にキー入力を求められます)。

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
Version            2.1.1+ent
Build Date         2026-09-15T21:39:40Z
Storage Type       raft
HA Enabled         true

$ vault operator unseal
...(省略)...
Unseal Progress    2/3

$ vault operator unseal
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            5
Threshold               3
Version                 2.1.1+ent
Build Date              2026-09-15T21:39:40Z
Storage Type            raft
Cluster Name            vault-cluster-8b2814c3
Cluster ID              4808da7d-9205-e82b-3c80-71598f0fb77c
HA Enabled              true
HA Mode                 standby
Active Node Address     <none>
Raft Committed Index    59
Raft Applied Index      59
``` 

3 回目の出力で`Sealed`が`false`に変化したことがわかるでしょう。この状態で`Initial Root Token`を使ってログインします。

```console
$ vault login hvs.BhwdXXr0BANd68gmzAEtMAkQ
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                hvs.BhwdXXr0BANd68gmzAEtMAkQ
token_accessor       8rtGY0cJ2m0hFqj9r2hqkz1S
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

これでログインは成功です。あわせて、[Enterprise ライセンスの配置](#enterprise-ライセンスの配置) で設定したライセンスが正しく自動ロードされているかを確認しておきましょう。

```console
$ vault read sys/license/status
Key                   Value
---                   -----
autoloaded            map[...features:[... Namespaces Sentinel ... Secrets Sync Automated Snapshots ...] expiration_time:2026-12-17T00:00:00Z ...]
autoloading_used      true
persisted_autoload    map[...]
```

`autoloading_used`が`true`になっており、`features`に`Namespaces`や`Sentinel`、`Secrets Sync`といった本ハンズオンで使う Enterprise 機能が含まれていれば OK です。以降の章ではこの環境を使ってハンズオンを進めていきます。

### seal を試す

`unseal`の逆の操作が`seal`です。インシデント対応などで Vault を即座にロックダウンしたいとき、`vault operator seal`を実行すると Vault はメモリ上のマスターキーを破棄し、再び封印された状態に戻ります。この操作は`sys/seal`への権限を持つトークンであれば一度の実行で完了します。

```console
$ vault operator seal
Success! Vault is sealed.

$ vault status
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  true
Total Shares            5
Threshold               3
Unseal Progress         0/3
Unseal Nonce            n/a
Version                 2.1.1+ent
Build Date              2026-09-15T21:39:40Z
Storage Type            raft
Removed From Cluster    false
HA Enabled              true
```

`Sealed`が`true`に戻り、再度いかなる操作も受け付けなくなりました。元に戻すには先ほどと同様`vault operator unseal`を 3 回実行します。手元で試した場合は、ここで unseal して先に進んでください。

### Auto Unseal (参考手順)

本番運用で`unseal`の鍵を 3 人がかりで毎回入力するのは現実的ではありません。そこで Vault はクラウドの鍵管理サービス (AWS KMS など) を使って自動的に`unseal`する **Auto Unseal** に対応しています。Shamir の鍵の代わりにマスターキーをクラウドの鍵で暗号化し、起動時に自動で復号して unseal します。

AWS KMS を使う場合、コンフィグに`seal "awskms"`スタンザを追加します。

```hcl
seal "awskms" {
  region     = "ap-northeast-1"
  kms_key_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

この設定で起動して`vault operator init`を実行すると、通常の`Unseal Key`の代わりに`Recovery Key`が生成され、起動と同時に自動で unseal されます。

```console
$ vault operator init
Recovery Key 1: 2bxJ0k7+lpoK8o6MAj7ebecIzh9V5d2n9L0GfWyUJjmn
Recovery Key 2: ElH9q/dkglVjFG8mfIZbriM8zbo1C1/JWH12j1R1L45j
Recovery Key 3: c9THb228rV++VUCTkyDjMUw0IG1LyKiaUa3ZmJzyq9oM
Recovery Key 4: EdhT6w6QKGCxtmuU8HSFbcSA/FXYYSHJ//fRF8UiD2+E
Recovery Key 5: s0APWYiXE6KMadHbwCbBWuTzL8CCUa5WnZOW5obGjM6k

Initial Root Token: hvs.Vfj4S1Wx5bFY5xms5eF751pr

Success! Vault is initialized
```

```console
$ vault status
Key                      Value
---                      -----
Recovery Seal Type       shamir
Seal Type                awskms
Initialized              true
Sealed                   false
Total Recovery Shares    5
Threshold                3
Version                  2.1.1+ent
Build Date               2026-09-15T21:39:40Z
Storage Type             raft
HA Enabled               true
```

`Seal Type`が`awskms`になり、`vault operator init`で初期化されると **Auto Unseal**のおかげで自動的に Vault が Unseal 状態になることが確認できます (*Sealed = false*)。`Recovery Key`は`unseal`には使いませんが、Root Token の再生成などの重要な操作で必要になるため大切に保管してください。

> AWS KMS を使う場合、Vault が稼働するインスタンスに`kms:Encrypt`, `kms:Decrypt`, `kms:DescribeKey`の権限を持つ IAM ロールを付与しておく必要があります。

### Audit Device を設定する

Vault への全てのリクエストとレスポンスを記録しておくことは、監査やインシデント調査の観点で非常に重要です。Vault では **Audit Device** を有効化することで、誰がいつどのパスにアクセスしたかを漏れなく記録できます。シークレットの値そのものはハッシュ化されて記録されるため、ログから生のシークレットが漏れることはありません。

ファイルに出力する Audit Device を有効化してみます。

```console
$ vault audit enable file file_path=/tmp/vault-audit.log
Success! Enabled the file audit device at: file/

$ vault audit list
Path     Type    Description
----     ----    -----------
file/    file    n/a
```

有効化すると、以降の全ての操作がログに記録されます。試しに何かリクエストを投げてからログを覗いてみましょう。

```console
$ vault secrets list > /dev/null
$ tail -n 1 /tmp/vault-audit.log | jq
{
  "time": "2026-10-07T02:11:38.123456Z",
  "type": "response",
  "auth": {
    "client_token": "hmac-sha256:...",
    "policies": ["root"]
  },
  "request": {
    "operation": "list",
    "path": "sys/mounts"
  }
}
```

`client_token`が`hmac-sha256:`から始まる値になっており、ハッシュ化されて記録されていることがわかります。Audit Device は複数同時に有効化でき、1 つでも書き込みに失敗すると Vault はリクエストをブロックします。これは「監査ログが残らない操作は許さない」という設計思想によるものです。本番では`syslog`や`socket`タイプを使ってログ基盤に集約するのが一般的です。

### 各種シークレットエンジンの有効化

Vault では機能ごとに **シークレットエンジン** を有効化 (`enable`) して利用します。本ハンズオンで使うエンジンをここでまとめて有効化しておきましょう。`enable`は強い権限を必要とするため、ここでは Root Token を使います。

```console
$ export VAULT_ADDR="http://127.0.0.1:8200"

$ vault secrets enable -path=kv -version=2 kv
Success! Enabled the kv secrets engine at: kv/

$ vault secrets enable aws
Success! Enabled the aws secrets engine at: aws/
```

有効化したエンジンの一覧は`vault secrets list`で確認できます。

```console
$ vault secrets list
Path               Type              Accessor                   Description
----               ----              --------                   -----------
agent-registry/    agent_registry    agent-registry_eeb36c2a    agent registry
aws/               aws               aws_7dc02279               n/a
cubbyhole/         cubbyhole         cubbyhole_21473bac         per-token private secret storage
identity/          identity          identity_b0a1517a          identity store
kv/                kv                kv_b8d0d6f4                n/a
sys/               system            system_570ec64b            system endpoints used for control, policy and debugging
```

`kv/`と`aws/`がそれぞれ API のエンドポイントとしてマウントされました。以降の章ではこれらのパスを使ってシークレットを扱っていきます。不要になったエンジンは`vault secrets disable <path>`で無効化できます。

### 参考リンク
* [Install Vault (Linux パッケージ)](https://developer.hashicorp.com/vault/install)
* [Enterprise ライセンスの管理](https://developer.hashicorp.com/vault/docs/enterprise/license)
* [Server Configuration](https://developer.hashicorp.com/vault/docs/configuration)
* [Seal / Unseal](https://developer.hashicorp.com/vault/docs/concepts/seal)
* [Auto Unseal with AWS KMS](https://developer.hashicorp.com/vault/docs/configuration/seal/awskms)
* [Integrated Storage (Raft)](https://developer.hashicorp.com/vault/docs/concepts/integrated-storage)
* [Audit Devices](https://developer.hashicorp.com/vault/docs/audit)

---

## テナントと権限設計

Vault を組織で共有する際は、チームや環境ごとにテナントを分離し、それぞれに最低限の権限だけを与える設計が重要です。ここでは Namespace によるテナント分離、Policy による権限定義と割り当て、そして Sentinel によるより高度なガバナンスを扱います。最後に、後続の章で使う AWS 操作用の権限を Vault に付与します。

### Namespace でテナントを分離する

> Namespace は **Vault Enterprise / HCP Vault** でのみ利用できる機能です。

Namespace は Vault の中に独立した「仮想の Vault」を作る機能です。各 Namespace は独自のポリシー、認証メソッド、シークレットエンジン、トークンを持ち、互いに干渉しません。これにより 1 つの Vault クラスタを複数チームでセキュアに共有できます。

ここでは`workshop`という Namespace を作ってみます。

```console
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault namespace create workshop
Key                  Value
---                  -----
custom_metadata      map[]
id                   abCD12
path                 workshop/

$ vault namespace list
Keys
----
workshop/
```

作成した Namespace を使うには、`-namespace`オプションか`VAULT_NAMESPACE`環境変数を指定します。以降のコマンドをこの Namespace に対して実行してみましょう。

```console
$ export VAULT_NAMESPACE=workshop
$ vault secrets enable -path=kv -version=2 kv
Success! Enabled the kv secrets engine at: kv/
```

この`kv`はルート Namespace の`kv`とは完全に別物です。チームごとに Namespace を切ることで、「他チームのシークレットを誤って参照・上書きしてしまう」といった事故を構造的に防げます。本ハンズオンでは以降ルート Namespace で進めるため、一旦この環境変数は外しておきます。

```console
$ unset VAULT_NAMESPACE
```

### Policy を作成して割り当てる

ここまで Root Token を利用して様々な操作をしてきましたが、実際の運用では強力な権限を持つ Root Token は保持をせずに必要な時のみ生成します。通常、最低限の権限のユーザを作成し Vault を利用していきます。権限は **Policy** で定義し、トークンや認証ロールに割り当てます。

まず、プリセットされるポリシー一覧を確認してみましょう。ポリシーを管理するエンドポイントは`sys/policy`と`sys/policies`です。`sys`のエンドポイントには[その他にも様々な機能](https://www.vaultproject.io/api/system/index.html)が用意されています。

```console
$ vault policy list
default
root
```

Policy は Vault のコンフィグレーションと同様`HCL`で記述します。`path`で対象のエンドポイントを、`capabilities`でそのエンドポイントに対する権限を指定します。ここでは後続の章で使う`kv`と`aws`のエンドポイントを操作できるポリシーを作ってみます。

```shell
$ cd /path/to/vault-workshop
$ cat > app-policy.hcl <<EOF
path "kv/*" {
  capabilities = [ "read", "list", "create", "update", "delete" ]
}

path "aws/creds/*" {
  capabilities = [ "read" ]
}
EOF
```

作ったら`vault policy write`のコマンドでポリシーを作成します。ポリシーの作成は Root Token で実施します。

```console
$ vault policy write app-policy app-policy.hcl
Success! Uploaded policy: app-policy

$ vault policy list           
app-policy
default
root

$ vault policy read app-policy
path "kv/*" {
  capabilities = [ "read", "list", "create", "update", "delete" ]
}

path "aws/creds/*" {
  capabilities = [ "read" ]
}
```

新しいポリシーができました。このポリシーと紐づけられたトークンは`kv`への読み書きと`aws/creds`からのクレデンシャル発行の権限を与えられます。ではトークンを発行して、割り当ての動作を確認してみます。

```console
$ vault token create -policy=app-policy 
Key                  Value
---                  -----
token                s.bA9M42W41G7tF90REMDCtMeO
token_accessor       LfQCnqPOJHGqO8TplfSjTNFs
token_duration       768h
token_renewable      true
token_policies       ["default" "app-policy"]
identity_policies    []
policies             ["default" "app-policy"]
```

発行したトークンを環境変数にセットして、権限の範囲を確かめます。

```shell
$ export APP_TOKEN=s.bA9M42W41G7tF90REMDCtMeO
```

```console
$ VAULT_TOKEN=$APP_TOKEN vault kv put kv/myapp password=p@SSW0d
Success! Data written to: kv/myapp

$ VAULT_TOKEN=$APP_TOKEN vault policy list
Error making API request.

URL: GET http://127.0.0.1:8200/v1/sys/policies/acl?list=true
Code: 403. Errors:

* permission denied
```

ポリシーに設定した通り、`kv`への書き込みは成功しますが、権限を与えていない`sys/policies`の操作はエラーになります。`deny by default`というルールのもと、明示的に許可したもの以外は全て`deny`となります。この「必要な権限だけを与える」設計が、Vault を安全に運用する基本です。

### Sentinel による制御

> Sentinel は **Vault Enterprise / HCP Vault** でのみ利用できる Policy as Code のフレームワークです。

Policy が「どのパスにアクセスできるか」を制御するのに対し、Sentinel は「どういう条件のときに操作を許可するか」という、より高度なガバナンスをコードで表現できます。例えば「平日の業務時間内しか本番シークレットを読めない」「特定の CIDR からのアクセスに限る」といったルールです。

Sentinel ポリシーには適用の強さに応じて 3 つのモードがあります。

* `advisory` — 違反しても警告を出すだけで操作は通す
* `soft-mandatory` — 原則ブロックするが、root 権限で上書きできる
* `hard-mandatory` — 例外なくブロックする

例として、業務時間 (平日 9〜18 時) 以外は書き込みを拒否する EGP (Endpoint Governing Policy) を書いてみます。

```python
import "time"

workday = rule {
    time.now.weekday > 0 and time.now.weekday < 6
}

workhour = rule {
    time.now.hour >= 9 and time.now.hour < 18
}

main = rule {
    workday and workhour
}
```

このポリシーを特定のパスに紐付けて適用します。

```console
$ vault write sys/policies/egp/business-hours \
    policy=@business-hours.sentinel \
    paths="kv/*" \
    enforcement_level="soft-mandatory"
Success! Data written to: sys/policies/egp/business-hours
```

これにより、業務時間外に`kv/*`への操作が行われると、`soft-mandatory`の設定に従ってブロックされます (root では上書き可能)。Sentinel を使うと、Policy だけでは表現しきれない組織のコンプライアンス要件を Vault 側で強制できます。

### Vault への AWS 権限付与

後続の AWS Secret Engine や Terraform 連携では、Vault 自身が AWS の API を呼び出して IAM ユーザやアクセスキーを動的に発行します。そのため、Vault に AWS を操作するための権限を与えておく必要があります。

与え方は大きく 2 通りです。

* Vault が稼働するインスタンスに IAM ロールを付与する (EC2 / ECS などでの推奨)
* Vault に直接 IAM ユーザのアクセスキーを登録する (手元で試す場合など)

ここでは手元で試す前提で、後者の方法を使います。Vault に登録する IAM ユーザには、必ずしも Admin 権限は必要なく、IAM ユーザやアクセスキーを発行・削除できる最低限の権限で十分です。以下のようなポリシーが目安です。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:AttachUserPolicy",
        "iam:CreateAccessKey",
        "iam:CreateUser",
        "iam:DeleteAccessKey",
        "iam:DeleteUser",
        "iam:DeleteUserPolicy",
        "iam:DetachUserPolicy",
        "iam:ListAccessKeys",
        "iam:ListAttachedUserPolicies",
        "iam:ListGroupsForUser",
        "iam:ListUserPolicies",
        "iam:PutUserPolicy",
        "iam:AddUserToGroup",
        "iam:RemoveUserFromGroup"
      ],
      "Resource": ["arn:aws:iam::<ACCOUNT_ID>:user/vault-*"]
    }
  ]
}
```

`Resource`を`vault-*`に絞ることで、Vault が発行するユーザ (後述の通り`vault-`から始まる名前になります) 以外には影響しないようにしています。このユーザのアクセスキーは [AWS Secret Engine](#aws-secret-engine) の章で Vault に登録します。

### 参考リンク
* [Namespaces](https://developer.hashicorp.com/vault/docs/enterprise/namespaces)
* [Policies](https://www.vaultproject.io/docs/concepts/policies.html)
* [Policy API Document](https://www.vaultproject.io/api/system/policy.html)
* [Sentinel](https://developer.hashicorp.com/vault/docs/enterprise/sentinel)

---

## アプリからの利用 (Auth Method)

ここまではトークン発行の権限を持つユーザ (今回の場合は root) を使ってトークンを発行してきました。実際にはアプリケーションやインスタンスが Vault を利用する際、信頼する認証プロバイダで認証をし適切なトークンを発行するというワークフローを組みます。これを担うのが **Auth Method** です。

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

ここでは AWS 上で動くワークロードを想定した **AWS Auth**、そして CI/CD やアプリなどマシン向けの汎用的な手段である **AppRole** の 2 つを扱います。各ロールはポリシーに紐付き、認証に成功するとクライアントにポリシーに基づいた権限のトークンが発行されます。

### AWS Auth

Vault の AWS での Authentication のデモになります。
デモの実行については、この Repo を Clone して[こちらの Asset](assets/auth_aws)をご使用ください。

AWS auth method については、[こちら](https://www.vaultproject.io/docs/auth/aws.html)を参照ください。

AWS auth method には２つのタイプがあります。`iam`と`ec2`の２種類です。
`iam`method では、IAM クレデンシャルでサインされた特別な AWS リクエストに対して認証を行います。IAM クレデンシャルは IAM instance profile や Lambda などで自動的に作成されるので、AWS 上のほぼ全てのサービスに対して利用できます。

`ec2`method は、AWS が EC2 インスタンスに自動的に付与するメタデータを用いて認証を行います。よって、この認証方法は EC2 のインスタンスにしか利用できません。

`ec2`method は`iam`method の登場の前に開発されたもので、現在のベスト・プラクティスとしてはより柔軟かつ高度なアクセスコントロールのある`iam`method を推奨しています。

このデモでは`iam`method を用いています。

#### Demo setup

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

#### Vault server のセットアップ

まず、上記アウトプットに表示される Vault server へ ssh で入ります。
そして Vault が立ち上がっているか確認してください。

```console
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
```

`vault status`コマンドでエラーがでなければ Vault は正常に起動しています。ただ、この状態｀Initialized｀が False であり、`Sealed`は true になっています。つまり、Vault は起動しているが、まだ初期化がされておらず、Seal 状態であるということです。

それでは、次に Vault の初期化を行います。このデモでは [Vault セットアップ](#vault-セットアップ) で説明した AWS KMS による **Auto Unseal** を使っています。Auto Unseal の設定方法は、Server 上の`/etc/vault.d/vault.hcl`を参照ください。

```console
ubuntu@ip-10-0-101-67:~$ vault operator init
Recovery Key 1: 2bxJ0k7+lpoK8o6MAj7ebecIzh9V5d2n9L0GfWyUJjmn
...(省略)...
Initial Root Token: s.Vfj4S1Wx5bFY5xms5eF751pr

Success! Vault is initialized
```

ここで表示される**Initial Root Token**の値を必ずメモしてください。`vault operator init`で初期化されると、**Auto unseal**のおかげで自動的に Vault が Unseal 状態になります (*Sealed = false*)。

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

このデモでは、Vault server に紐付けられた IAM ロールを用いて AWS auth method を設定しています。もし別の IAM ロールや IAM ユーザーの権限で認証を行いたい場合は、以下のように個別に設定することも可能です。

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
Success! You are now authenticated.

ubuntu@ip-10-0-101-67:~$ ./aws_auth.sh
Success! Enabled the kv secrets engine at: secret/
Success! Data written to: secret/myapp/config
Success! Uploaded policy: myapp
Success! Enabled aws auth method at: aws/
Success! Data written to: auth/aws/config/client
Success! Data written to: auth/aws/role/dev-role-iam
```

これで Vault server 側の設定は終わりです。

#### Vault client のセットアップ

それでは、Vault client 側から AWS 認証で Vault にアクセスし、シークレットの読み出しができるか確認してみましょう。

まず、Vault client に ssh でログインします。もし、Vault client の IP アドレスが分からなくなった場合は、`terraform output`コマンドで確認してください。`vault status`コマンドを叩いて、Vault server につながっているか確認します。ちなみに Vault server は**VAULT_ADDR**という環境変数で指定されています。

この状態でシークレットが読み出せるか試してみます。

```console
ubuntu@ip-10-0-101-96:~$ vault read secret/myapp/config
Error reading secret/myapp/config: Error making API request.

URL: GET http://10.0.101.67:8200/v1/secret/myapp/config
Code: 400. Errors:

* missing client token
```

まだ認証をしていないので、Token が無くエラーになります。
それでは、認証をしてみます。認証は`vault login`コマンドを使用します。`-method=aws`で AWS 認証を行うことを指定し、`role=dev-role-iam`で Vault 上のどの Role の Token を取得するか指定します。それでは実行してみましょう。

```console
ubuntu@ip-10-0-101-96:~$ vault login -method=aws role=dev-role-iam
Success! You are now authenticated. The token information displayed below
is already stored in the token helper.

Key                                Value
---                                -----
token                              s.3BiCdXIBpmRf68iFi1wXnj6i
token_duration                     24h
token_renewable                    true
token_policies                     ["default" "myapp"]
policies                           ["default" "myapp"]
token_meta_canonical_arn           arn:aws:iam::753646501470:role/masa-vault-auth-vault-client-role
token_meta_auth_type               iam
```

認証が成功し、トークンが返ってきました。ここで再度、シークレットの読み出しをしてみます。

```console
ubuntu@ip-10-0-101-96:~$ vault read secret/myapp/config
Key                 Value
---                 -----
refresh_interval    30s
password            suP3rsec(et!
ttl                 30s
username            appuser
```

今回は読み出しに成功しました。AWS 認証を使うと AWS 上のサービスやインスタンスで使用される IAM ロールを用いて簡単に Vault にアクセスすることができます。これにより AWS 上で動くインスタンスやサービスは、アプリケーション内に認証用のシークレットを保管する必要がなくなり、また Vault 認証用のメカニズムも非常に簡単に導入することができます。

### AppRole

AWS 認証は AWS 上のワークロードに最適ですが、オンプレミスの CI/CD やコンテナなど、IAM ロールを持たないマシンから Vault を使いたい場合もあります。そうしたケースで広く使われるのが **AppRole** です。LDAP や他の認証方法が人による操作を前提としている一方、AppRole はマシンやアプリによる操作が前提とされており、自動化のワークフローに組み込みやすくなっています。

ワークフローの例は以下のようなイメージです。

![](https://learn.hashicorp.com/assets/images/vault-approle-workflow.png)

ref: [https://learn.hashicorp.com/vault/identity-access-management/iam-authentication](https://learn.hashicorp.com/vault/identity-access-management/iam-authentication)

AppRole で認証するためには`Role ID`と`Secret ID`という二つの値が必要で、username と password のようなイメージです。各 AppRole はポリシーに紐付き、AppRole で承認されるとクライアントにポリシーに基づいた権限のトークンが発行されます。

ここでは [テナントと権限設計](#テナントと権限設計) で作成した`app-policy`を再利用します。`approle`を`enable`にし、`app-policy`のポリシーに基づいた AppRole を一つ作成します。

```console
$ VAULT_TOKEN=$ROOT_TOKEN vault auth enable approle
$ VAULT_TOKEN=$ROOT_TOKEN vault write -f auth/approle/role/my-approle policies=app-policy
$ VAULT_TOKEN=$ROOT_TOKEN vault read auth/approle/role/my-approle

Key                      Value
---                      -----
bind_secret_id           true
local_secret_ids         false
period                   0s
policies                 [app-policy]
secret_id_num_uses       0
secret_id_ttl            0s
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

push 型はカスタムの値を指定できますが、Vault 以外のサーバ、アプリやツールなど Secret ID を発行する側に Secret ID を知らせてしまうことになるため、通常使用しません。`pull`と呼ばれる方法が一般的です。

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
$ vault write auth/approle/login role_id="a25b3148-7b95-57bf-bc5d-cb72ffc08e68" secret_id="1cef3c1e-feca-99d8-ecd4-7a17ca997919"
Key                     Value
---                     -----
token                   s.nEolH5Pjqf3207KljT9xoamS
token_duration          768h
token_renewable         true
token_policies          ["default" "app-policy"]
policies                ["default" "app-policy"]
```

AppRole により認証され、`app-policy`の権限を持ったトークンが発行されました。発行されたトークンを使うと、ポリシーで許可された`kv`や`aws/creds`にアクセスできます。

今回は AppRole の基本的な使い方を試しましたが、より実践的にどのように扱うかは[こちらの記事](https://blog.kabuctl.run/?p=94)に記載しておきましたので、本ハンズオン終了後、一読してみてください。

### 参考リンク
* [AWS Auth Method](https://www.vaultproject.io/docs/auth/aws.html)
* [AppRole Auth Method](https://www.vaultproject.io/docs/auth/approle.html)
* [AppRole API Document](https://www.vaultproject.io/api/auth/approle/index.html)
* [Auth0 を使った OIDC 認証](https://learn.hashicorp.com/vault/operations/oidc-auth)

---

## Static Secret Engine

Vault の最も基本的なユースケースが、静的なシークレット (パスワードや API キーなど) を安全に保管し、必要なときに取り出す Static Secret です。ここでは [Vault セットアップ](#vault-セットアップ) で有効化した KV v2 シークレットエンジンを使い、CLI と API の両方からの読み書き、バージョニング、そして Vault Enterprise / HCP Vault の Secret Sync による AWS Secrets Manager への反映までを扱います。

### Vault CLI 経由での読み書き

まずは CLI でデータを put してみましょう。KV v2 では`vault kv`サブコマンドを使います。

```console
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault kv put kv/iam name=kabu password=passwd
$ vault kv get kv/iam                                            
====== Data ======
Key         Value
---         -----
name        kabu
password    passwd
```

特定のフィールドだけを取り出すこともできます。アプリのスクリプトからパスワードだけを抜き出したいときなどに便利です。

```console
$ vault kv get -field=password kv/iam
passwd
```

最後にデータを削除します。KV v2 ではデータ本体とメタデータが分かれているため、完全に消す場合は両方を削除します。

```console
$ vault kv delete kv/iam
$ vault kv metadata delete kv/iam
```

### Vault API 経由での読み書き

Vault の CLI は API への HTTPS のアクセスをラップしているため、全ての CLI での操作は API への curl のリクエストに変換できます。`-output-curl-string`を使うだけです。

```console
$ vault kv get -output-curl-string kv/iam
curl -H "X-Vault-Token: $(vault print token)" http://127.0.0.1:8200/v1/kv/data/iam
```

アプリなどのクライアントから Vault の API を呼ぶ時などに、記述方法に迷った時に便利です。実際に API 経由で書き込みと読み出しをしてみましょう。KV v2 では、データを`data`というキーでラップして送る点に注意してください。

```console
$ curl -s \
    -H "X-Vault-Token: $(vault print token)" \
    -X POST \
    -d '{"data":{"name":"kabu","password":"passwd"}}' \
    http://127.0.0.1:8200/v1/kv/data/iam | jq
{
  "request_id": "15a27428-e566-186b-3a47-b66c727f5f02",
  "data": {
    "created_time": "2026-10-07T02:20:57.871216Z",
    "deletion_time": "",
    "destroyed": false,
    "version": 1
  }
}
```

読み出しも試してみます。レスポンスの`data.data`に実際のシークレットが入っています。

```console
$ curl -s \
    -H "X-Vault-Token: $(vault print token)" \
    http://127.0.0.1:8200/v1/kv/data/iam | jq -r '.data.data'
{
  "name": "kabu",
  "password": "passwd"
}
```

このように Vault の機能は全て HTTP API で提供されているため、どの言語のアプリからでも同じように利用できます。

### バージョニング

KV v2 の大きな特徴がバージョニングです。同じキーに対して書き込みを行うと、過去のデータは上書きされず世代として保持されます。データを上書きしてバージョン 2 を作ってみます。

```console
$ vault kv put kv/iam name=kabu password=passwd
Key              Value
---              -----
created_time     2026-10-07T06:00:44.023139Z
destroyed        false
version          1

$ vault kv put kv/iam name=kabu-2 password=passwd
Key              Value
---              -----
created_time     2026-10-07T06:08:03.871067Z
destroyed        false
version          2
```

データが上書きされてバージョン 2 のデータが生成されました。古いバージョンのデータは`-version`オプションを付与することで参照できます。

```console
$ vault kv get -version=1 kv/iam
====== Data ======
Key         Value
---         -----
name        kabu
password    passwd
```

誤って上書きしてしまった場合でも、過去のバージョンを参照できるため復旧が容易です。古いバージョンのデータを完全に削除する際は`destroy`を使います。

```console
$ vault kv destroy -versions=1 kv/iam
Success! Data written to: kv/destroy/iam
```

もう一つ、更新時に注意したいのが`put`と`patch`の違いです。`put`はキーの集合ごと上書きするため、指定し忘れたキーは消えてしまいます。一部のフィールドだけを更新したい場合は`patch`を使います。

```console
$ vault kv patch kv/iam password=passwd-2
$ vault kv get kv/iam
====== Data ======
Key         Value
---         -----
name        kabu-2
password    passwd-2
```

`name`を指定していないにもかかわらず残っていることがわかります。`patch`を使うとデータの一部のみを安全に更新できます。

### Secret Sync による AWS Secrets Manager への反映

> Secret Sync は **Vault Enterprise / HCP Vault** で利用できる機能です。

Vault に保管した Static Secret を、AWS Secrets Manager など外部のシークレットストアに自動で同期する機能が **Secret Sync** です。Vault を一元的な source of truth としながら、どうしても AWS Secrets Manager を参照する必要があるアプリやマネージドサービスにも値を届けられます。Vault 側でシークレットを更新すると、同期先にも自動で反映されます。

まず、同期先となる AWS Secrets Manager の宛先 (destination) を登録します。Vault が AWS Secrets Manager を操作するためのアクセスキーを指定します。

```console
$ vault write sys/sync/destinations/aws-sm/my-dest \
    access_key_id=$AWS_ACCESS_KEY_ID \
    secret_access_key=$AWS_SECRET_ACCESS_KEY \
    region=ap-northeast-1
Success! Data written to: sys/sync/destinations/aws-sm/my-dest
```

次に、同期したい KV のシークレットをこの宛先に関連付け (associate) します。

```console
$ vault write sys/sync/destinations/aws-sm/my-dest/associations/set \
    mount=kv \
    secret_name=iam
Key                Value
---                -----
associated_secrets map[kv_12159ddb/iam:map[...]]
```

関連付けが完了すると、AWS Secrets Manager 側に対応するシークレットが作成されます。AWS CLI で確認してみましょう。

```console
$ aws secretsmanager list-secrets --query 'SecretList[].Name'
[
    "vault/kv/iam"
]
```

Vault 側で`kv/iam`を更新すると、この AWS Secrets Manager のシークレットにも自動的に新しい値が反映されます。これにより、Vault を正とした運用を崩さずに、AWS ネイティブなシークレット参照とも共存できます。

### 参考リンク
* [Vault KV Secret Engine](https://www.vaultproject.io/docs/secrets/kv/kv-v2.html)
* [KV Secret Engine API](https://www.vaultproject.io/api/secret/kv/index.html)
* [Secret Sync](https://developer.hashicorp.com/vault/docs/sync)

---

## AWS Secret Engine

AWS シークレットエンジンでは IAM ポリシーの定義に基づいた AWS のキーを動的に発行することが可能です。AWS のキー発行のワークフローをシンプルにし、TTL などを設定することでよりセキュアに利用できます。

サポートしているクレデンシャルタイプは下記の三つです。

* IAM user (Access Key & Secret Key)
* Assumed Role
* Federation Token

### IAM ユーザの動的発行

[Vault セットアップ](#vault-セットアップ) で`aws`エンジンは有効化済みですが、未実施の場合はここで enable にします。

```shell
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault secrets enable aws
```

次に Vault が AWS の API を実行するために必要なキーを登録します。ここで使うのは [テナントと権限設計](#テナントと権限設計) で用意した、IAM ユーザを発行・削除できる権限を持つキーです。

```shell
$ vault write aws/config/root \
    access_key=************ \
    secret_key=************ \
    region=ap-northeast-1
```

`access_key`, `secret_key`, `region`はご自身の環境に合わせたものに書き換えてください。ここでは必ずしも AWS の Admin ユーザを登録する必要はなく、ロールやユーザを発行できるユーザであれば大丈夫です。

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

この`watch`の出力結果を見るとユーザが増えていることがわかります。発行されるユーザ名は`vault-`から始まることに注目してください。`lease_id`はあとで使うのでメモしておいてください。

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

このユーザを使って動作を確認してみましょう。別の端末で`aws configure`に Vault から払い出されたキーを設定し、以下のコマンドを実行します。

```console
$ aws ec2 describe-instances

An error occurred (UnauthorizedOperation) when calling the DescribeInstances operation: You are not authorized to perform this operation.

$ aws s3 ls
2019-08-16 21:41:34 github-image-tkaburagi
2019-05-26 23:31:14 vault-enterprise-tkaburagi
2019-03-08 21:37:21 web-terraform-state-tykaburagi
```

Role に設定した通り S3 に対する操作のみ可能なことがわかります。

### ポリシーで TTL が異なるアクセスキー発行

ロールは複数登録できるため、「用途ごとに権限と TTL の異なるキーを使い分ける」運用が可能です。先ほどの`my-role`は長めのデフォルト TTL でしたが、ここでは短命な使い捨てキー用に、別の TTL を持つロールを追加してみます。ロールには`default_sts_ttl`と`max_sts_ttl`で TTL を設定できます。

```shell
$ vault write aws/roles/my-role-short \
    credential_type=iam_user \
    default_sts_ttl=2m \
    max_sts_ttl=10m \
    policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::*"
    }
  ]
}
EOF
```

`my-role`は従来通りの TTL でフルの S3 権限を、`my-role-short`は 2 分の短い TTL で`ListBucket`だけを許可するロールです。このように同じエンジンの中でポリシーと TTL を変えたロールを並べ、クライアントや用途に応じて使い分けます。短い TTL のキーを発行してみます。

```console
$ vault read aws/creds/my-role-short

Key                Value
---                -----
lease_id           aws/creds/my-role-short/agnda2uyVWKso4E3HoWlPqY8
lease_duration     2m
lease_renewable    true
access_key         ****************
secret_key         ****************
security_token     <nil>
```

`lease_duration`がロールに設定した 2 分になっています。TTL を短くするほど、万が一キーが漏れても有効な時間が限られるため、より安全です。`watch`で見ていると、このユーザは 2 分後に自動で削除されます。

### 強制 Revoke

発行済みのクレデンシャルを即座に無効化したいケースもあります。Revoke にはマニュアルと自動の 2 通りの方法があります。

まずはマニュアルでの実行手順です。`vault read aws/creds/my-role`を実行した際に発行された`lease_id`をコピーしてください。

```shell
$ vault lease revoke aws/creds/my-role/<LEASE_ID>
```

`watch`の実行結果を見るとユーザが削除されているでしょう。インシデント発生時など、特定のロールから発行した全てのクレデンシャルをまとめて強制的に失効させたい場合は、プレフィックス指定の`-prefix`を使います。

```console
$ vault lease revoke -prefix -force aws/creds/my-role
All revocation operations queued successfully!
```

`-prefix`は指定したパス配下の全てのリースを対象にし、`-force`は Vault 側でリースを削除する際に AWS 側の削除に失敗しても強制的に進めるオプションです。これにより、`my-role`から発行された全てのアクセスキーを一括で無効化できます。`watch`の出力から、`vault-`で始まるユーザが全て消えていることを確認してください。

次に自動 Revoke です。デフォルトでは TTL が`768h`になっています。これはエンジン全体のデフォルトとして数分にしてみましょう。

```shell
$ vault write aws/config/lease lease=2m lease_max=10m
```

```console
$ vault read aws/config/lease

Key          Value
---          -----
lease        2m0s
lease_max    10m0s
```

この状態でユーザを発行すると、TTL 経過後に Vault が自動的にユーザを削除します。動的シークレットの TTL と強制 Revoke を組み合わせることで、「使うときだけ発行し、不要になったら即座に消す」というセキュアな運用が実現できます。

### 参考リンク
* [AWS Secret Engine](https://www.vaultproject.io/docs/secrets/aws/index.html)
* [AWS Secret Engine API](https://www.vaultproject.io/api/secret/aws/index.html)
* [Lease, Renew, and Revoke](https://developer.hashicorp.com/vault/docs/concepts/lease)

---

## Terraform 連携

ここまで Vault を CLI や API から使ってきましたが、インフラをコードで管理する Terraform と組み合わせると、Vault の価値はさらに高まります。ここでは Terraform から Vault の動的クレデンシャルを使って AWS に`apply`する方法と、Terraform 1.10 以降の **ephemeral** リソースを使ってシークレットを state に残さずに利用する方法を扱います。

### 動的クレデンシャルによる apply

Terraform で AWS にリソースを作る際、通常は長期間有効なアクセスキーを環境変数や`provider`ブロックに書きます。これはキーの管理と漏洩リスクという課題を抱えています。そこで、[AWS Secret Engine](#aws-secret-engine) で設定した動的クレデンシャルを Terraform から読み出し、その短命なキーで AWS プロバイダを認証させます。

まず Vault プロバイダと AWS プロバイダを定義します。Vault プロバイダは`VAULT_ADDR`と`VAULT_TOKEN`環境変数を参照します。

```hcl
provider "vault" {}

# AWS Secret Engine からアクセスキーを動的に発行する
data "vault_aws_access_credentials" "creds" {
  backend = "aws"
  role    = "my-role"
}

# 発行された短命なキーで AWS プロバイダを認証する
provider "aws" {
  region     = "ap-northeast-1"
  access_key = data.vault_aws_access_credentials.creds.access_key
  secret_key = data.vault_aws_access_credentials.creds.secret_key
}
```

この状態で`apply`を実行すると、Terraform はまず Vault から IAM ユーザを 1 つ発行し、そのキーを使って AWS にリソースを作成します。

```console
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ export VAULT_TOKEN=$(vault print token)

$ terraform init
$ terraform apply -auto-approve
data.vault_aws_access_credentials.creds: Reading...
data.vault_aws_access_credentials.creds: Read complete after 6s

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

実行の裏側では、[AWS Secret Engine](#aws-secret-engine) の章で見たのと同じように`vault-`で始まる IAM ユーザが一時的に作られ、`apply`が終わってリースが切れると自動的に削除されます。これにより、Terraform の実行ごとにユニークで短命なクレデンシャルが使われ、長期間有効なキーを一切保持する必要がなくなります。

> HCP Terraform / Terraform Enterprise では、ワークスペースと Vault の間に信頼関係を結び、ワークロードアイデンティティで Vault を認証する **Vault-backed dynamic credentials** が利用できます。この場合`VAULT_TOKEN`すら保持する必要がなくなります。

### ephemeral リソースで state にシークレットを残さない

動的クレデンシャルで AWS 認証そのものは安全になりましたが、もう一つの課題が **Terraform の state ファイル** です。従来の`data`ソースで Vault からシークレットを読むと、その値が state ファイルに平文で書き込まれてしまいます。state を安全に扱っていても、シークレットが複数箇所に散らばるのは好ましくありません。

これを解決するのが Terraform 1.10 で導入された **ephemeral** (エフェメラル) な仕組みです。ephemeral リソースは実行中のみメモリ上に存在し、その値は plan や state のどちらにも保存されません。Vault プロバイダは`vault_kv_secret_v2`などの ephemeral リソースを提供しています。

```hcl
terraform {
  required_version = ">= 1.10.0"
}

# ephemeral: 読み出した値は state にも plan にも残らない
ephemeral "vault_kv_secret_v2" "db" {
  mount = "kv"
  name  = "iam"
}
```

ephemeral リソースの値は、同じく state に残らない **write-only** 引数にのみ渡せます。例えば RDS のパスワードを設定する例は以下のようになります。

```hcl
resource "aws_db_instance" "app" {
  # ... 省略 ...
  username = "admin"

  # write-only 引数: state に保存されない
  password_wo         = ephemeral.vault_kv_secret_v2.db.data["password"]
  password_wo_version = 1
}
```

`password_wo`は書き込み専用の引数で、Terraform は値を適用はしますが state には記録しません。`password_wo_version`を変更したときだけ再適用されます。

```console
$ terraform apply -auto-approve
ephemeral.vault_kv_secret_v2.db: Opening...
ephemeral.vault_kv_secret_v2.db: Opening complete after 1s
...
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

$ terraform show -json | jq '.values.root_module.resources[].values.password_wo'
null
```

`terraform show`で state を覗いてもパスワードは`null`で、値がどこにも永続化されていないことがわかります。動的クレデンシャルで「認証」を、ephemeral で「シークレットの受け渡し」を、それぞれ state に残さず安全に扱えるようになります。

### 参考リンク
* [Vault Provider: vault_aws_access_credentials](https://registry.terraform.io/providers/hashicorp/vault/latest/docs/data-sources/aws_access_credentials)
* [Ephemeral values in Terraform](https://www.hashicorp.com/blog/ephemeral-values-in-terraform)
* [Vault-backed dynamic credentials in HCP Terraform](https://developer.hashicorp.com/terraform/cloud-docs/dynamic-provider-credentials/vault-backed)

---

## Day2 運用

Vault を本番で動かし始めると、構築よりも日々の運用 (Day2) の比重が大きくなります。その中でも最も重要なのが、万一に備えたバックアップとリストアです。ここでは Vault の推奨ストレージである Integrated Storage (Raft) を前提に、スナップショットの取得と復元を扱います。

### バックアップ (スナップショットの取得)

Integrated Storage を使っている場合、Vault はクラスタ全体の状態を 1 つのスナップショットファイルとして出力できます。これには KV のデータ、ポリシー、Auth Method やシークレットエンジンの設定などが全て含まれます。取得は`vault operator raft snapshot save`コマンド一つで完了します。

```console
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault operator raft snapshot save vault-$(date +%Y%m%d).snap

$ ls -lh vault-*.snap
-rw-------  1 user  staff   1.2M 10  7 11:30 vault-20261007.snap
```

スナップショットの取得にはクラスタ全体を読み出す権限が必要なため、専用のポリシーを用意してトークンを割り当てるのが一般的です。

```hcl
path "sys/storage/raft/snapshot" {
  capabilities = ["read"]
}
```

本番では、このコマンドを cron や自動化ツールで定期実行し、取得したスナップショットを S3 などのオブジェクトストレージに保管します。スナップショット自体は Vault のマスターキーで暗号化された状態ではないため、保管先のアクセス制御と暗号化を必ず行ってください。

> Vault Enterprise / HCP Vault では、スナップショットを指定したスケジュールで自動取得し、S3 や GCS へ自動保管する **Automated Snapshots** が利用できます。手動運用に比べて取り逃しのリスクを下げられます。

### リストア (スナップショットからの復元)

取得したスナップショットから復元するには`vault operator raft snapshot restore`を使います。復元は現在のクラスタのデータを上書きするため、実行には細心の注意が必要です。

```console
$ vault operator raft snapshot restore vault-20261007.snap
```

> **注意:** リストアは現在の Vault のデータを丸ごと置き換える破壊的な操作です。誤って本番クラスタに別環境のスナップショットを流し込むと取り返しがつきません。必ず対象のクラスタと、スナップショットの取得元・取得時刻を確認してから実行してください。可能であれば、まず検証用クラスタで復元の手順と内容を確認することをお勧めします。

復元後、Auto Unseal を使っている場合は自動的に unseal されますが、Shamir の鍵を使っている場合はスナップショット取得時点の`Unseal Key`で改めて unseal する必要があります。復元が完了したら、代表的なシークレットを読み出せるか、ポリシーや Auth Method が意図通りに残っているかを確認します。

```console
$ vault status
Key             Value
---             -----
Sealed          false
...

$ vault kv get kv/iam
====== Data ======
Key         Value
---         -----
name        kabu-2
password    passwd-2
```

取得時点のデータが復元されていることが確認できれば完了です。バックアップは「取得できること」だけでなく「正しくリストアできること」までを定期的に検証して初めて意味を持ちます。半期に一度など、リストアのリハーサルを運用手順に組み込んでおきましょう。

### 参考リンク
* [vault operator raft snapshot](https://developer.hashicorp.com/vault/docs/commands/operator/raft)
* [Integrated Storage](https://developer.hashicorp.com/vault/docs/concepts/integrated-storage)
* [Automated Integrated Storage Snapshots](https://developer.hashicorp.com/vault/docs/enterprise/automated-integrated-storage-snapshots)
