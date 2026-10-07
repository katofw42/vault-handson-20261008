# HashiCorp Vault Workshop (AWS ハンズオン)

[Vault](https://www.vaultproject.io/) は HashiCorp が中心に開発をするシークレット管理ツールです。Vault を利用することで既存の Static なシークレット管理のみならず、クラウド、データベースや SSH などの様々なシークレットを動的に発行することができます。Vault はマルチプラットフォームでかつ全ての機能を HTTP API で提供しているため、環境やクライアントを問わず利用することができます。

本ドキュメントは、Vault を AWS 環境で一通り使い倒すことを目的とした一本道のハンズオンです。Vault のセットアップから始まり、AWS 権限付与の土台づくり、アプリケーションからの認証、シークレットの読み書き、AWS の動的クレデンシャル発行、テナントと権限の設計、Terraform 連携、そして Day2 の運用までを順に扱います。各セクションは前のセクションの状態を引き継ぐ前提で記述しているため、できるだけ順番に進めることをお勧めします。

> このハンズオンは **Amazon Linux 2023** 上の **Vault Enterprise (v2.1.2)** を前提にしています。Namespace や Sentinel、Secret Sync といった Vault Enterprise の機能を利用するため、有効な Enterprise ライセンスが必要です。コマンドや出力例はすべてこの環境で実際に確認したものを掲載しています。

## Pre-requisite

* 環境
	* Amazon Linux 2023 の動作する環境 (本ハンズオンでは EC2 インスタンスを使用)
	* EC2 には `HandsonRole` というインスタンスプロファイルを割り当て、コンソールアクセス用に SSM (`AmazonSSMManagedInstanceCore`) を持たせておきます。以降の AWS 権限付与はこのロールを起点にします。

* ソフトウェア
	* Vault Enterprise v2.1.2 (本ハンズオンの手順でインストールします)
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
2. [Vault への AWS 権限付与](#vault-への-aws-権限付与)
	- [実行手順: インスタンスプロファイルから AssumeRole で権限を渡す](#実行手順-インスタンスプロファイルから-assumerole-で権限を渡す)
3. [アプリからの利用 (Auth Method)](#アプリからの利用-auth-method)
	- [AWS Auth](#aws-auth)
	- [AppRole](#approle)
4. [Static Secret Engine](#static-secret-engine)
	- [Vault CLI 経由での読み書き](#vault-cli-経由での読み書き)
	- [Vault API 経由での読み書き](#vault-api-経由での読み書き)
	- [バージョニング](#バージョニング)
	- [Secret Sync による AWS Secrets Manager への反映](#secret-sync-による-aws-secrets-manager-への反映)
5. [AWS Secret Engine](#aws-secret-engine)
	- [Assumed Role によるクレデンシャルの動的発行](#assumed-role-によるクレデンシャルの動的発行)
	- [ポリシーで TTL が異なるアクセスキー発行](#ポリシーで-ttl-が異なるアクセスキー発行)
	- [強制 Revoke](#強制-revoke)
6. [テナントと権限設計](#テナントと権限設計)
	- [Namespace でテナントを分離する](#namespace-でテナントを分離する)
	- [Policy を作成して割り当てる](#policy-を作成して割り当てる)
	- [Sentinel による制御](#sentinel-による制御)
7. [Terraform 連携](#terraform-連携)
	- [動的クレデンシャルによる apply](#動的クレデンシャルによる-apply)
	- [ephemeral リソースで state にシークレットを残さない](#ephemeral-リソースで-state-にシークレットを残さない)
8. [Day2 運用](#day2-運用)
	- [バックアップ (スナップショットの取得)](#バックアップ-スナップショットの取得)
	- [リストア (スナップショットからの復元)](#リストア-スナップショットからの復元)
9. [クリーンアップ (削除手順)](#クリーンアップ-削除手順)
	- [AWS 側の後片付け](#aws-側の後片付け)
	- [EC2 インスタンスの削除](#ec2-インスタンスの削除)

---

## Vault セットアップ

ここではまず Vault のインストールと起動、`init` / `unseal` / `seal` といったライフサイクルの操作、クラウドの鍵管理サービスを使った Auto Unseal、監査ログを記録する Audit Device、そして以降のハンズオンで使うシークレットエンジンの有効化までを扱います。

このハンズオンは **Amazon Linux 2023** 上の **Vault Enterprise (v2.1.2)** を前提にしています。手元の作業端末からは SSH で Amazon Linux 2023 の EC2 インスタンスにログインし、その上で作業を進めてください。

### Vault のインストール

Amazon Linux 2023 では、HashiCorp の公式 yum リポジトリを追加すれば`dnf`一発で Vault Enterprise をインストールできます。まず EC2 にログインしてから、以下を実行します。

```console
$ sudo dnf install -y dnf-plugins-core
$ sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
$ sudo dnf install -y vault-enterprise
```

Community 版 (`vault`) ではなく **`vault-enterprise`** パッケージを指定している点に注意してください。特定のバージョンを固定したい場合は`vault-enterprise-2.1.2+ent-1`のようにバージョンを付けて指定します。

インストールが終わったら、バージョンを確認します。

```console
$ vault version
Vault v2.1.2+ent (8ee5bcac416a4b1020c4c64027690db129456746), built 2026-10-06T15:32:01Z
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
Version                 2.1.2+ent
Build Date              2026-10-06T15:32:01Z
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
Version            2.1.2+ent
Build Date         2026-10-06T15:32:01Z
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
Version                 2.1.2+ent
Build Date              2026-10-06T15:32:01Z
Storage Type            raft
Cluster Name            vault-cluster-2d0535cc
Cluster ID              99b81265-528f-d8c8-255a-6f8d24e3ef33
Removed From Cluster    false
HA Enabled              true
HA Cluster              n/a
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
Version                 2.1.2+ent
Build Date              2026-10-06T15:32:01Z
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
Version                  2.1.2+ent
Build Date               2026-10-06T15:32:01Z
Storage Type             raft
HA Enabled               true
```

`Seal Type`が`awskms`になり、`vault operator init`で初期化されると **Auto Unseal**のおかげで自動的に Vault が Unseal 状態になることが確認できます (*Sealed = false*)。`Recovery Key`は`unseal`には使いませんが、Root Token の再生成などの重要な操作で必要になるため大切に保管してください。

> AWS KMS を使う場合、Vault が稼働するインスタンスに`kms:Encrypt`, `kms:Decrypt`, `kms:DescribeKey`の権限を持つ IAM ロールを付与しておく必要があります。

### Audit Device を設定する

Vault への全てのリクエストとレスポンスを記録しておくことは、監査やインシデント調査の観点で非常に重要です。Vault では **Audit Device** を有効化することで、誰がいつどのパスにアクセスしたかを漏れなく記録できます。シークレットの値そのものはハッシュ化されて記録されるため、ログから生のシークレットが漏れることはありません。

ファイルに出力する Audit Device を有効化してみます。ログの出力先には、Vault を実行する`vault`ユーザが書き込めるディレクトリを用意します。ここでは`/var/log/vault/`を使います。

> **注意:** パッケージ版の`vault.service`は systemd の`PrivateTmp=yes`で動くため、`/tmp`配下はサービス専用の隔離された領域になります。`file_path=/tmp/vault-audit.log`を指定するとログはホストの`/tmp`からは見えない場所に書かれてしまい、後述の`tail`で参照できません。そのため`/tmp`以外のパス (ここでは`/var/log/vault/`) を使います。

```console
$ sudo mkdir -p /var/log/vault
$ sudo chown vault:vault /var/log/vault

$ vault audit enable file file_path=/var/log/vault/audit.log
Success! Enabled the file audit device at: file/

$ vault audit list
Path     Type    Description
----     ----    -----------
file/    file    n/a
```

有効化すると、以降の全ての操作がログに記録されます。試しに何かリクエストを投げてからログを覗いてみましょう。ログファイルは`vault`ユーザ所有 (パーミッション 600) で作られるため、参照には`sudo`を使います。

```console
$ vault secrets list > /dev/null
$ sudo tail -n 1 /var/log/vault/audit.log | jq
{
  "time": "2026-10-07T17:50:54.652730286Z",
  "type": "response",
  "auth": {
    "client_token": "hmac-sha256:...",
    "policies": ["root"]
  },
  "request": {
    "operation": "read",
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
agent-registry/    agent_registry    agent-registry_78657e75    agent registry
aws/               aws               aws_745b3153               n/a
cubbyhole/         cubbyhole         cubbyhole_a9fbdd5c         per-token private secret storage
identity/          identity          identity_2689cb7d          identity store
kv/                kv                kv_e5f087d7                n/a
sys/               system            system_02822775            system endpoints used for control, policy and debugging
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

## Vault への AWS 権限付与

この章では、後続の [AWS Secret Engine](#aws-secret-engine) や [Terraform 連携](#terraform-連携) で Vault が AWS を操作するための権限を、先に用意しておきます。Vault 自身が AWS の API を呼び出して IAM ユーザやアクセスキー、一時クレデンシャルを動的に発行するため、その土台となる権限付与をこの段階で済ませておくと、以降の章がスムーズに進みます。

Vault に AWS 権限を与える方法は大きく 2 通りです。

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

### 実行手順: インスタンスプロファイルから AssumeRole で権限を渡す

このハンズオンでは、より本番に近く、かつアクセスキーを一切保持しない方法を使います。Vault は EC2 上で動いているため、**この EC2 のインスタンスプロファイル (`HandsonRole`) を使って対象リソース用のロールを AssumeRole し、そのロールの権限でシークレットを払い出す** 構成にします。静的なアクセスキーを Vault に登録する必要がなく、権限は AssumeRole 先のロールに集約できます。

構成は以下の 2 つで決まります。

1. **AssumeRole される側のロールの許可ポリシー** — Vault 経由で払い出す操作の実体 (ここでは対象 VPC にサブネットを作成する権限など) を定義します。
2. **そのロールの信頼ポリシー (trust policy)** — 誰が AssumeRole できるか。ここでは Vault が動く EC2 のインスタンスロール (`HandsonRole`) を信頼元に指定します。

このハンズオンでは、**あらかじめ作成済みの VPC に対して、Vault 経由で払い出した権限でサブネットを作成する** というシナリオを題材にします。そのため対象の VPC は事前に用意しておいてください。

> **プレースホルダの置き換えが必要です。** 以降の JSON やコマンドに出てくる `<ACCOUNT_ID>` (AWS アカウント ID) と `<VPC_ID>` (サブネットを作成する対象の既存 VPC の ID、例: `vpc-xxxxxxxx`) は、ご自身の環境の値に置き換えてください。リージョンも必要に応じて読み替えてください (本書では `ap-northeast-1` を使用)。

まず、対象ロールに付与する **許可ポリシー** です。この例では、指定した既存 VPC (`<VPC_ID>`) にサブネットを作成・削除でき、作成時のタグ付けと可視化のための Describe 系を許可しています。`Resource` と `Condition` を絞ることで、払い出される権限を必要最小限に限定しています。サブネットの削除 (`ec2:DeleteSubnet`) と `ec2:DescribeNetworkInterfaces` を含めているのは、[Terraform 連携](#terraform-連携) の `terraform destroy` で作成したサブネットを後片付けできるようにするためです。

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "CreateSubnetOnlyInTargetVPC",
            "Effect": "Allow",
            "Action": "ec2:CreateSubnet",
            "Resource": "arn:aws:ec2:ap-northeast-1:<ACCOUNT_ID>:vpc/<VPC_ID>"
        },
        {
            "Sid": "CreateSubnetResource",
            "Effect": "Allow",
            "Action": "ec2:CreateSubnet",
            "Resource": "arn:aws:ec2:ap-northeast-1:<ACCOUNT_ID>:subnet/*"
        },
        {
            "Sid": "TagOnlyWhenCreatingSubnet",
            "Effect": "Allow",
            "Action": "ec2:CreateTags",
            "Resource": "arn:aws:ec2:ap-northeast-1:<ACCOUNT_ID>:subnet/*",
            "Condition": {
                "StringEquals": {
                    "ec2:CreateAction": "CreateSubnet"
                }
            }
        },
        {
            "Sid": "DeleteSubnetResource",
            "Effect": "Allow",
            "Action": "ec2:DeleteSubnet",
            "Resource": "arn:aws:ec2:ap-northeast-1:<ACCOUNT_ID>:subnet/*"
        },
        {
            "Sid": "ReadOnlyForVisibility",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeSubnets",
                "ec2:DescribeVpcs",
                "ec2:DescribeAvailabilityZones",
                "ec2:DescribeNetworkInterfaces"
            ],
            "Resource": "*"
        }
    ]
}
```

次に、同じ対象ロールの **信頼ポリシー** です。Vault が動く EC2 のインスタンスロール `HandsonRole` だけが、このロールを AssumeRole できるようにします。これにより「この EC2 上の Vault」以外はこのロールを引き受けられません。

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<ACCOUNT_ID>:role/HandsonRole"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

対象ロール (ここでは `handson-assume-role` とします) を上記 2 つのポリシーで作成します。お手元の IAM 変更権限を持つプロファイルで実行してください。許可ポリシーを `permissions.json`、信頼ポリシーを `trust.json` に保存しておきます。

```console
$ aws iam create-role \
    --role-name handson-assume-role \
    --assume-role-policy-document file://trust.json

$ aws iam put-role-policy \
    --role-name handson-assume-role \
    --policy-name handson-assume-permissions \
    --policy-document file://permissions.json
```

あわせて、Vault が動く EC2 のインスタンスロール (`HandsonRole`) 側にも、この対象ロールを AssumeRole できる権限が必要です。信頼ポリシーと許可ポリシーは両方そろって初めて AssumeRole が成立します。

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/handson-assume-role"
        }
    ]
}
```

これで、Vault は static なアクセスキーを持たずに、インスタンスプロファイルから `handson-assume-role` を AssumeRole してシークレット (この例では一時的なクレデンシャル) を払い出せます。この対象ロールの ARN は、[AWS Secret Engine](#aws-secret-engine) の章でロールの `role_arns` として指定します。

> なお、[Static Secret Engine](#static-secret-engine) の Secret Sync で AWS Secrets Manager へ同期する場合は、`HandsonRole` 側に Secrets Manager を操作する権限も必要です。以下を追加しておきます。
>
> ```json
> {
>     "Version": "2012-10-17",
>     "Statement": [
>         {
>             "Sid": "VaultSecretSync",
>             "Effect": "Allow",
>             "Action": [
>                 "secretsmanager:CreateSecret",
>                 "secretsmanager:UpdateSecret",
>                 "secretsmanager:PutSecretValue",
>                 "secretsmanager:TagResource",
>                 "secretsmanager:DeleteSecret",
>                 "secretsmanager:DescribeSecret",
>                 "secretsmanager:ListSecrets",
>                 "secretsmanager:GetSecretValue"
>             ],
>             "Resource": "*"
>         }
>     ]
> }
> ```

### 参考リンク
* [AWS Secret Engine](https://developer.hashicorp.com/vault/docs/secrets/aws)
* [AssumeRole でのクレデンシャル発行](https://developer.hashicorp.com/vault/docs/secrets/aws#sts-assumerole)
* [IAM ロールの信頼ポリシー](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_terms-and-concepts.html)

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

AWS auth method を使うと、AWS 上で動くインスタンスやサービスが、AWS が発行する認証情報をそのまま使って Vault にログインできます。アプリの中に Vault 用のクレデンシャルを別途埋め込む必要がなくなるのが大きな利点です。

AWS auth method には `iam` と `ec2` の 2 つのタイプがあります。

* **`iam` method** — IAM クレデンシャルで署名した AWS リクエストに対して認証します。IAM ロールは EC2 のインスタンスプロファイルや Lambda などで自動的に利用できるため、AWS 上のほぼ全てのサービスに適用できます。より柔軟なアクセス制御ができるため、現在のベストプラクティスとしては基本的にこちらが推奨されます。
* **`ec2` method** — AWS が各 EC2 インスタンスに付与する **インスタンスアイデンティティドキュメント** (メタデータ) を使って認証します。EC2 インスタンスでしか使えませんが、追加のクレデンシャルを一切持たずに「この EC2 である」ことだけで認証できるのが特徴です。

ここでは、用意済みの EC2 1 台だけで完結する形で、**`ec2` method**（インスタンスメタデータによるログイン）を実際に動かして確認します。Vault サーバと認証されるクライアントは同じ EC2 上にあり、この EC2 が自分自身のインスタンスアイデンティティで Vault にログインします。

#### 事前準備: Vault 側の設定

まず、ec2 method のログインを成立させるために必要な Vault 側の設定を行います。root token でログインした状態で実行してください。

ec2 method では、ログイン時に Vault が AWS に問い合わせてインスタンスの正当性を検証します。そのため **Vault の実行環境 (この EC2 のインスタンスロール) に `ec2:DescribeInstances` の権限が必要** です。IAM ロールを束縛条件に使う場合は `iam:GetInstanceProfile` も必要になります。以下のインラインポリシーを、この EC2 のインスタンスロールに付与しておきます。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VaultAwsEc2Auth",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "iam:GetInstanceProfile"
      ],
      "Resource": "*"
    }
  ]
}
```

> 権限が不足している場合、ログイン時に `failed to verify instance ID: ... UnauthorizedOperation ... ec2:DescribeInstances` というエラーになります。

次に、aws 認証メソッドを有効化し、Vault が AWS API を呼ぶためのクライアント設定を行います。この EC2 はインスタンスロールを持っているため、`auth/aws/config/client` に static なアクセスキーを渡す必要はありません (Vault がインスタンスロールの一時クレデンシャルを自動的に利用します)。

```console
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault auth enable aws
Success! Enabled aws auth method at: aws/

$ vault write -f auth/aws/config/client
Success! Data written to: auth/aws/config/client
```

認証に成功したトークンへ割り当てるポリシーを作成します。ここでは `kv` を読めるだけの `ec2-demo` を用意します。

```console
$ echo 'path "kv/*" { capabilities = ["read","list"] }' | vault policy write ec2-demo -
Success! Uploaded policy: ec2-demo
```

最後に、ec2 method のロールを作成します。`auth_type=ec2` を指定し、どのインスタンスを認証対象とするかを束縛条件で絞ります。ここではインスタンスプロファイルの ARN で束縛しています (他にも `bound_ami_id`・`bound_vpc_id`・`bound_account_id` などが使えます)。`<ACCOUNT_ID>` はご自身の AWS アカウント ID に置き換えてください。

```console
$ vault write auth/aws/role/ec2-role \
    auth_type=ec2 \
    bound_iam_instance_profile_arn="arn:aws:iam::<ACCOUNT_ID>:instance-profile/*" \
    policies=ec2-demo \
    ttl=1h
Success! Data written to: auth/aws/role/ec2-role
```

これで Vault 側の事前設定は完了です。整理すると、ec2 method のログインに必要な事前設定は以下の 4 つです。

1. `vault auth enable aws` — aws 認証メソッドの有効化
2. `vault write -f auth/aws/config/client` — Vault が AWS を照会するためのクライアント設定
3. ログイン後に付与するポリシーの作成
4. `auth_type=ec2` のロール作成 (束縛条件を 1 つ以上指定)

加えて AWS 側では、前述のとおり Vault の実行ロールに `ec2:DescribeInstances` 権限が必要です。

#### メタデータを使ってログインする

それでは、この EC2 のインスタンスメタデータを使ってログインします。EC2 のメタデータサービス (IMDSv2) から、インスタンスアイデンティティドキュメントの **PKCS#7 署名** を取得し、それを `auth/aws/login` に渡します。

```console
$ TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
$ PKCS7=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/dynamic/instance-identity/pkcs7 | tr -d '\n')
$ NONCE=$(uuidgen)

$ vault write auth/aws/login role=ec2-role pkcs7="$PKCS7" nonce="$NONCE"
Key                            Value
---                            -----
token                          hvs.CAESIKQIrfE2F-lr3Kq5HmHZz66d4r1lJqehAsYIIW3oe4Ex...
token_accessor                 4OQKRMDsn6UjfprHU1ZyMnAP
token_duration                 1h
token_renewable                true
token_policies                 ["default" "ec2-demo"]
identity_policies              []
policies                       ["default" "ec2-demo"]
token_meta_account_id          730335563172
token_meta_auth_type           ec2
token_meta_role                ec2-role
token_meta_role_tag_max_ttl    0s
```

ログインに成功し、`ec2-demo` ポリシーの付いたトークンが発行されました。`token_meta_auth_type` が `ec2` になっており、この EC2 がメタデータの署名だけで認証されたことがわかります。アクセスキーやパスワードを一切渡していない点に注目してください。

> `nonce` は再認証を防ぐための値です。同じインスタンスで再度ログインする際は、初回と同じ nonce を使う必要があります (省略すると Vault が生成し、`token_meta` 経由では返りません)。クライアント側で nonce を保持する運用が前提です。
>
> なお、CLI ヘルパーの `vault login -method=aws` は既定で `iam` method を使うため、`ec2` method のロールに対しては `auth method iam not allowed for role ...` というエラーになります。ec2 method では上記のように `vault write auth/aws/login` でメタデータの PKCS#7 を直接渡す形が確実です。

発行されたトークンで、ポリシーどおり `kv` が読めることを確認できます。これで、AWS 上の EC2 が「自分が何者か」をメタデータで証明するだけで Vault にログインし、権限に応じたシークレットへアクセスできることが確認できました。

### AppRole

AWS 認証は AWS 上のワークロードに最適ですが、オンプレミスの CI/CD やコンテナなど、IAM ロールを持たないマシンから Vault を使いたい場合もあります。そうしたケースで広く使われるのが **AppRole** です。LDAP や他の認証方法が人による操作を前提としている一方、AppRole はマシンやアプリによる操作が前提とされており、自動化のワークフローに組み込みやすくなっています。

ワークフローの例は以下のようなイメージです。

![](https://learn.hashicorp.com/assets/images/vault-approle-workflow.png)

ref: [https://learn.hashicorp.com/vault/identity-access-management/iam-authentication](https://learn.hashicorp.com/vault/identity-access-management/iam-authentication)

AppRole で認証するためには`Role ID`と`Secret ID`という二つの値が必要で、username と password のようなイメージです。各 AppRole はポリシーに紐付き、AppRole で承認されるとクライアントにポリシーに基づいた権限のトークンが発行されます。

まず、この AppRole に紐付けるポリシーを用意します。ここでは`kv`を読み書きできる`app-policy`をその場で作成します。

```console
$ vault policy write app-policy - <<EOF
path "kv/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
EOF
Success! Uploaded policy: app-policy
```

`approle`を`enable`にし、`app-policy`のポリシーに基づいた AppRole を一つ作成します。

```console
$ vault auth enable approle
$ vault write -f auth/approle/role/my-approle policies=app-policy
$ vault read auth/approle/role/my-approle

Key                        Value
---                        -----
alias_metadata             map[]
bind_secret_id             true
local_secret_ids           false
policies                   [app-policy]
secret_id_bound_cidrs      <nil>
secret_id_num_uses         0
secret_id_ttl              0s
token_bound_cidrs          []
token_explicit_max_ttl     0s
token_max_ttl              0s
token_no_default_policy    false
token_num_uses             0
token_period               0s
token_policies             [app-policy]
token_ttl                  0s
token_type                 default
```

これで AppRole の作成は完了です。次に`Role ID`を取得します。

```console
$ vault read auth/approle/role/my-approle/role-id
Key        Value
---        -----
role_id    2db44579-2255-8d2c-331a-40412419c072
```

次に`Secret ID`を取得しますが、いくつかの方法があります。

一つは`push`と呼ばれる方法で、カスタムの値を指定するパターンです。

```console
$ vault write -f auth/approle/role/my-approle/custom-secret-id secret_id=ZeCletlb
Key                   Value
---                   -----
secret_id             ZeCletlb
secret_id_accessor    7032e201-1ece-c288-d378-903bdc21a0b5
secret_id_num_uses    0
secret_id_ttl         0s
```

push 型はカスタムの値を指定できますが、Vault 以外のサーバ、アプリやツールなど Secret ID を発行する側に Secret ID を知らせてしまうことになるため、通常使用しません。`pull`と呼ばれる方法が一般的です。

```console
$ vault write -f auth/approle/role/my-approle/secret-id
Key                   Value
---                   -----
secret_id             4bed29dc-1956-337e-41a8-9708a233a5d7
secret_id_accessor    0762cc61-cfc9-7021-e50c-1cc70708391f
secret_id_num_uses    0
secret_id_ttl         0s
```

この場合、クライアントに値を持たせることがなく Secret ID の発行が可能となりよりセキュアです。

これらを使って認証し、トークンを取得してみましょう。

```console
$ vault write auth/approle/login role_id="2db44579-2255-8d2c-331a-40412419c072" secret_id="4bed29dc-1956-337e-41a8-9708a233a5d7"
Key                     Value
---                     -----
token                   hvs.CAESIL4wRXQFVjU4L9WPqrNfX5XEW5hogPuTFEQkkKOfu4WgG...
token_accessor          60YewuJwztluACX4ZpR4waFQ
token_duration          768h
token_renewable         true
token_policies          ["app-policy" "default"]
identity_policies       []
policies                ["app-policy" "default"]
token_meta_role_name    my-approle
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
== Secret Path ==
kv/data/iam

======= Metadata =======
Key                Value
---                -----
created_time       2026-10-07T17:09:28.123943402Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

====== Data ======
Key         Value
---         -----
name        kabu
password    passwd
```

KV v2 では、実データの上に`Secret Path`やバージョンなどの`Metadata`が表示されます。特定のフィールドだけを取り出すこともできます。アプリのスクリプトからパスワードだけを抜き出したいときなどに便利です。

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
curl -H "X-Vault-Request: true" -H "X-Vault-Token: $(vault print token)" http://127.0.0.1:8200/v1/kv/data/iam
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

Secret Sync は利用前に機能を **アクティベート** する必要があります。まず一度だけ以下を実行します。

```console
$ vault write -f sys/activation-flags/secrets-sync/activate
Key            Value
---            -----
activated      [secrets-sync]
unactivated    [enable-scim force-identity-deduplication secrets-import]
```

次に、同期先となる AWS Secrets Manager の宛先 (destination) を登録します。[Vault への AWS 権限付与](#vault-への-aws-権限付与) のとおり、この EC2 はインスタンスプロファイル (`HandsonRole`) を持っているため、ここでも **静的なアクセスキーは不要** です (リージョンのみ指定)。`HandsonRole` には Secrets Manager を操作する権限 (`secretsmanager:CreateSecret` / `PutSecretValue` / `TagResource` など) を付与しておきます。

```console
$ vault write sys/sync/destinations/aws-sm/my-dest region=ap-northeast-1
Key                   Value
---                   -----
connection_details    map[region:ap-northeast-1]
name                  my-dest
options               map[custom_tags:map[] granularity_level:secret-path secret_name_template:vault/{{ .MountAccessor }}/{{ .SecretPath }}]
type                  aws-sm
```

次に、同期したい KV のシークレットをこの宛先に関連付け (associate) します。

```console
$ vault write sys/sync/destinations/aws-sm/my-dest/associations/set \
    mount=kv \
    secret_name=iam
Key                        Value
---                        -----
associated_secrets         map[kv_b8d0d6f4/iam:map[... external_name:vault/kv_b8d0d6f4/iam ... sync_status:SYNCED ...]]
store_name                 my-dest
store_type                 aws-sm
sync_operation_counters    map[SYNCED:1]
```

`sync_status` が `SYNCED` になれば同期完了です。AWS Secrets Manager 側に対応するシークレットが作成されます。シークレット名は既定で `vault/<MountAccessor>/<SecretPath>` というテンプレートで決まります。AWS CLI で確認してみましょう。

```console
$ aws secretsmanager list-secrets --region ap-northeast-1 --query 'SecretList[].Name'
[
    "vault/kv_b8d0d6f4/iam"
]
```

Vault 側で`kv/iam`を更新すると、この AWS Secrets Manager のシークレットにも自動的に新しい値が反映されます。これにより、Vault を正とした運用を崩さずに、AWS ネイティブなシークレット参照とも共存できます。

### 参考リンク
* [Vault KV Secret Engine](https://www.vaultproject.io/docs/secrets/kv/kv-v2.html)
* [KV Secret Engine API](https://www.vaultproject.io/api/secret/kv/index.html)
* [Secret Sync](https://developer.hashicorp.com/vault/docs/sync)

---

## AWS Secret Engine

AWS シークレットエンジンでは、AWS の権限を Vault 経由で動的に、かつ短命なクレデンシャルとして払い出せます。発行のたびにユニークで有効期限付きのクレデンシャルになるため、長期間有効なキーを配り回す必要がなくなります。

サポートしているクレデンシャルタイプは下記の三つです。

* IAM User — IAM ユーザとアクセスキーをその都度作成・削除する
* Assumed Role — 既存の IAM ロールを STS で AssumeRole し、一時クレデンシャルを払い出す
* Federation Token — フェデレーショントークンを払い出す

本ハンズオンでは、[Vault への AWS 権限付与](#vault-への-aws-権限付与) で用意した構成に合わせて **Assumed Role** を使います。Vault は EC2 のインスタンスプロファイル (`HandsonRole`) の権限で対象ロール (ドキュメント上は `handson-assume-role`) を AssumeRole し、その権限スコープの一時クレデンシャルを払い出します。IAM ユーザを都度作成する `iam_user` と違い、`iam:CreateUser` などの強い権限が不要で、権限は AssumeRole 先のロールに集約できるのが利点です。

### Assumed Role によるクレデンシャルの動的発行

[Vault セットアップ](#vault-セットアップ) で`aws`エンジンは有効化済みですが、未実施の場合はここで enable にします。

```shell
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault secrets enable aws
```

次に、Vault が AWS を操作するためのクライアント設定を行います。[Vault への AWS 権限付与](#vault-への-aws-権限付与) のとおり、この EC2 はインスタンスプロファイル (`HandsonRole`) を持っているため、`aws/config/root` に **静的なアクセスキーを渡す必要はありません**。リージョンだけ指定すれば、Vault はインスタンスロールの一時クレデンシャルを使って AWS を呼び出します。

```console
$ vault write aws/config/root region=ap-northeast-1
Success! Data written to: aws/config/root
```

続いて、AssumeRole 先のロールを紐付けた Vault ロールを作成します。`credential_type=assumed_role` を指定し、`role_arns` に [Vault への AWS 権限付与](#vault-への-aws-権限付与) で作成した対象ロールの ARN を指定します。

```console
$ vault write aws/roles/subnet-role \
    credential_type=assumed_role \
    role_arns="arn:aws:iam::<ACCOUNT_ID>:role/handson-assume-role"
Success! Data written to: aws/roles/subnet-role
```

> `<ACCOUNT_ID>` はご自身の AWS アカウント ID に置き換えてください。この対象ロールは、VPC へのサブネット作成権限など「払い出したい権限」を許可ポリシーに持ち、信頼ポリシーで `HandsonRole` からの AssumeRole を許可している必要があります (詳細は [Vault への AWS 権限付与](#vault-への-aws-権限付与) を参照)。

それでは、このロールを使ってクレデンシャルを発行してみましょう。`ttl` で有効期限を指定できます。

```console
$ vault read aws/creds/subnet-role ttl=15m
Key                Value
---                -----
lease_id           aws/creds/subnet-role/XPtUG66uD5wgdTY4MOxHZ6Pt
lease_duration     14m59s
lease_renewable    false
access_key         ASIA2UC3EJWSF43YTZU2
arn                arn:aws:sts::730335563172:assumed-role/handson-assume-role/vault-root-subnet-role-1791393377-uE1dJZxsiK7bLyDyyuOo
secret_key         5ruahqxcBQzW/LCt7H2jRaBUcvpt8NpkxIt2TN42
security_token     IQoJb3JpZ2luX2VjE...(省略)...
ttl                14m59s
```

`access_key` が `ASIA` から始まる STS の一時クレデンシャルになっており、`arn` が `assumed-role/handson-assume-role/...` になっていることがわかります。Vault が対象ロールを AssumeRole して払い出した証拠です。

払い出したクレデンシャルで、対象ロールに許可された操作 (事前作成した VPC へのサブネット作成) が行えることを確認してみましょう。別端末で環境変数にセットして実行します (IAM の反映に数秒かかることがあります)。`<VPC_ID>` は [Vault への AWS 権限付与](#vault-への-aws-権限付与) で対象にした既存 VPC の ID に置き換えてください。

```console
$ export AWS_ACCESS_KEY_ID=ASIA2UC3EJWSF43YTZU2
$ export AWS_SECRET_ACCESS_KEY=5ruahqxcBQzW/LCt7H2jRaBUcvpt8NpkxIt2TN42
$ export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjE...(省略)...

# 許可されている操作: 事前作成済みの VPC にサブネットを作成できる
$ aws ec2 create-subnet \
    --vpc-id <VPC_ID> \
    --cidr-block 10.0.100.0/24 \
    --region ap-northeast-1 \
    --query 'Subnet.SubnetId'
"subnet-0a1b2c3d4e5f67890"

# 許可されていない操作: DescribeInstances は拒否される
$ aws ec2 describe-instances --region ap-northeast-1
An error occurred (UnauthorizedOperation) when calling the DescribeInstances operation: You are not authorized to perform this operation. ... is not authorized to perform: ec2:DescribeInstances ...
```

対象ロールの許可ポリシーどおり、事前作成した VPC へのサブネット作成はできる一方で、許可していない `DescribeInstances` は拒否されました。Vault が払い出すクレデンシャルの権限が、AssumeRole 先ロールのスコープに正しく閉じていることが確認できます。

> **TTL の注意点:** `assumed_role` の実体は STS の `AssumeRole` であり、STS の仕様上 **最小 TTL は 15 分 (900 秒)** です。`ttl=2m` のように 15 分未満を指定すると `Code: 400` のエラーになります。短命運用でも 15 分が下限となる点に注意してください。

### ポリシーで TTL が異なるアクセスキー発行

ロールは複数登録できるため、「用途ごとに権限と TTL を変えて使い分ける」運用が可能です。ロールには `default_sts_ttl` (発行時のデフォルト TTL) と `max_sts_ttl` (renew できる最大 TTL) を設定できます。ここでは短命運用向けに、デフォルト 15 分・最大 1 時間のロールを追加してみます。

```console
$ vault write aws/roles/subnet-role-short \
    credential_type=assumed_role \
    role_arns="arn:aws:iam::<ACCOUNT_ID>:role/handson-assume-role" \
    default_sts_ttl=15m \
    max_sts_ttl=1h
Success! Data written to: aws/roles/subnet-role-short
```

発行してみると、`lease_duration` がロールに設定した 15 分になります。

```console
$ vault read aws/creds/subnet-role-short
Key                Value
---                -----
lease_id           aws/creds/subnet-role-short/OWLnwcl0F9onQQ1ER1C3Q3Lk
lease_duration     14m59s
access_key         ASIA2UC3EJWS...
arn                arn:aws:sts::730335563172:assumed-role/handson-assume-role/vault-root-subnet-role-short-...
...
```

同じ AssumeRole 先でも、ロールごとに TTL のプロファイルを分けておくことで、「短時間だけ使う用途」と「もう少し長く使う用途」をクライアント側の都合で選べます。TTL を短くするほど、万が一クレデンシャルが漏れても有効な時間が限られるため、より安全です。

### 強制 Revoke

発行済みのクレデンシャルを即座に無効化したいケースもあります。発行時の `lease_id` を指定すれば、個別に revoke できます。

```shell
$ vault lease revoke aws/creds/subnet-role/<LEASE_ID>
```

インシデント発生時など、特定のロールから発行した全てのクレデンシャルをまとめて強制的に失効させたい場合は、プレフィックス指定の `-prefix` を使います。

```console
$ vault lease revoke -prefix -force aws/creds/subnet-role
Warning! Force-removing leases can cause Vault to become out of sync with
secret engines!
All revocation operations queued successfully!
```

`-prefix` は指定したパス配下の全てのリースを対象にし、`-force` は Vault 側でリースを削除する際に AWS 側の失効に失敗しても強制的に進めるオプションです。これにより、`subnet-role` から発行した全てのクレデンシャルを一括で無効化できます。

> `assumed_role` で払い出した STS の一時クレデンシャルは、revoke しても STS トークン自体は TTL 満了まで有効なまま、という点に注意してください (STS の仕様で即時失効はできません)。Vault 側のリース管理からは外れるため、`iam_user` タイプのように AWS 上のエンティティ (ユーザ) を即削除する強制力はありません。即時失効が必要な要件では `iam_user` タイプの採用も検討します。

動的シークレットの TTL と revoke を組み合わせることで、「使うときだけ発行し、不要になったら破棄する」というセキュアな運用が実現できます。

### 参考リンク
* [AWS Secret Engine](https://developer.hashicorp.com/vault/docs/secrets/aws)
* [AWS Secret Engine API](https://developer.hashicorp.com/vault/api-docs/secret/aws)
* [Lease, Renew, and Revoke](https://developer.hashicorp.com/vault/docs/concepts/lease)

---

## テナントと権限設計

Vault を組織で共有する際は、チームや環境ごとにテナントを分離し、それぞれに最低限の権限だけを与える設計が重要です。ここでは Namespace によるテナント分離、Policy による権限定義と割り当て、そして Sentinel によるより高度なガバナンスを扱います。

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

Policy は Vault のコンフィグレーションと同様`HCL`で記述します。`path`で対象のエンドポイントを、`capabilities`でそのエンドポイントに対する権限を指定します。ここでは例として`kv`と`aws`のエンドポイントを操作できる`demo-policy`を作ってみます。

```shell
$ cat > demo-policy.hcl <<EOF
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
$ vault policy write demo-policy demo-policy.hcl
Success! Uploaded policy: demo-policy

$ vault policy list           
demo-policy
default
root

$ vault policy read demo-policy
path "kv/*" {
  capabilities = [ "read", "list", "create", "update", "delete" ]
}

path "aws/creds/*" {
  capabilities = [ "read" ]
}
```

新しいポリシーができました。このポリシーと紐づけられたトークンは`kv`への読み書きと`aws/creds`からのクレデンシャル発行の権限を与えられます。ではトークンを発行して、割り当ての動作を確認してみます。

```console
$ vault token create -policy=demo-policy 
Key                  Value
---                  -----
token                hvs.CAESIG...
token_accessor       LfQCnqPOJHGqO8TplfSjTNFs
token_duration       768h
token_renewable      true
token_policies       ["default" "demo-policy"]
identity_policies    []
policies             ["default" "demo-policy"]
```

発行したトークンを環境変数にセットして、権限の範囲を確かめます。

```shell
$ export DEMO_TOKEN=hvs.CAESIG...
```

```console
$ VAULT_TOKEN=$DEMO_TOKEN vault kv put kv/myapp password=p@SSW0d
Success! Data written to: kv/myapp

$ VAULT_TOKEN=$DEMO_TOKEN vault policy list
Error making API request.

URL: GET http://127.0.0.1:8200/v1/sys/policies/acl?list=true
Code: 403. Errors:

* permission denied
```

ポリシーに設定した通り、`kv`への書き込みは成功しますが、権限を与えていない`sys/policies`の操作はエラーになります。`deny by default`というルールのもと、明示的に許可したもの以外は全て`deny`となります。この「必要な権限だけを与える」設計が、Vault を安全に運用する基本です。

### Sentinel による制御

> Sentinel は **Vault Enterprise / HCP Vault** でのみ利用できる Policy as Code のフレームワークです。

Policy (ACL) が「どのパスにアクセスできるか」を制御するのに対し、Sentinel は「どういう条件のときに操作を許可するか」という、ACL だけでは表現できないロジックベースのガバナンスをコードで記述できます。接続元 IP やトークンの発行時刻、リクエスト内容などを条件にできるため、ゼロトラストやインシデント対応といった Enterprise のセキュリティ要件を Vault 側で強制できます。

Sentinel ポリシーには、評価対象によって 2 種類があります。

* **EGP (Endpoint Governing Policy)** — 特定の **パス** に紐付く。ログインパスなど未認証のパスにも適用できる
* **RGP (Role Governing Policy)** — 特定の **トークン / Identity エンティティ / グループ** に紐付く

さらに適用の強さに応じて 3 つのモードがあります。

* `advisory` — 違反しても警告を出すだけで操作は通す
* `soft-mandatory` — 原則ブロックするが、root 権限で上書きできる
* `hard-mandatory` — 例外なくブロックする

ここでは Enterprise で特に需要の高い 2 つのユースケースを、実際に動かして確認します。1 つはインシデント対応のための **ブレークグラス (一斉トークン失効)**、もう 1 つはネットワーク統制のための **ログイン元 IP の制限** です。

> **補足:** Sentinel の`print()`によるデバッグ出力は、ポリシー評価が **失敗したときだけ** サーバログに出力されます。成功時には出力されない点に注意してください。

#### ユースケース 1: ブレークグラス (一斉トークン失効)

トークンや生成済みシークレットが漏洩した可能性が判明したとき、「ある時刻より前に発行されたトークンを一斉に無効化したい」という要件が生まれます。全トークンを個別に revoke するのは時間がかかり、漏洩していないトークンまで巻き込んでしまいます。Sentinel なら、トークンの発行時刻 (`token.creation_time`) を条件に、カットオフ時刻より前のトークンだけを一括で遮断できます。

まず、検証用に「古いトークン」と、カットオフ時刻をはさんだ「新しいトークン」を用意します。

```console
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ OLD_TOKEN=$(vault token create -policy=default -ttl=60m -field=token)   # カットオフ前に発行
$ CUTOFF=$(date -u +%Y-%m-%dT%H:%M:%SZ)                                   # この時刻を基準にする
$ NEW_TOKEN=$(vault token create -policy=default -ttl=60m -field=token)   # カットオフ後に発行
```

次にブレークグラス用の EGP を書きます。`rule when not request.unauthenticated`で認証済みリクエストだけを対象にし、トークンの発行時刻がカットオフより後 (= 漏洩に関与していない) の場合のみ許可します。`<CUTOFF>`は上で取得した時刻に置き換えてください。

```python
import "time"

main = rule when not request.unauthenticated {
    time.load(token.creation_time).unix > time.load("<CUTOFF>").unix
}
```

全パスに適用したいので、`paths="*"`・`hard-mandatory`で登録します。

```console
$ vault write sys/policies/egp/break-glass \
    policy=@break-glass.sentinel \
    paths="*" \
    enforcement_level="hard-mandatory"
Success! Data written to: sys/policies/egp/break-glass
```

では古いトークンで操作してみます。カットオフより前に発行されているため、拒否されます。

```console
$ VAULT_TOKEN=$OLD_TOKEN vault token lookup
Error looking up token: Error making API request.

URL: GET http://127.0.0.1:8200/v1/auth/token/lookup-self
Code: 403. Errors:

* 2 errors occurred:
	* egp standard policy "root/break-glass" evaluation resulted in denial.
The specific error was:
<nil>
A trace of the execution for policy "root/break-glass" is available:
Result: false
Description: <none>
Rule "main" (root/break-glass:2:1) = false
	* permission denied
```

一方、カットオフより後に発行された新しいトークンは問題なく通ります。

```console
$ VAULT_TOKEN=$NEW_TOKEN vault token lookup
Key                 Value
---                 -----
display_name        token
policies            [default]
ttl                 59m
...
```

このように、トークンを個別に revoke することなく、発行時刻を境に「疑わしいトークンだけ」を即座に遮断できます。対応が済んだらポリシーを削除して通常運用に戻します。

```console
$ vault delete sys/policies/egp/break-glass
Success! Data deleted (if it existed) at: sys/policies/egp/break-glass
```

#### ユースケース 2: ログイン元 IP を制限する

次はネットワーク統制です。「社内ネットワーク (特定の CIDR) からのログインしか認めない」という要件を、`sockaddr`インポートを使って認証メソッドのログインパスに適用します。ここでは`userpass`認証メソッドで検証します。

まず検証用の認証メソッドとユーザを用意します。

```console
$ vault auth enable userpass
Success! Enabled userpass auth method at: userpass/

$ vault write auth/userpass/users/alice password=pass policies=default
Success! Data written to: auth/userpass/users/alice
```

次に、許可する CIDR を`10.0.0.0/8`に限定する EGP を書きます。`request.connection.remote_addr` (接続元 IP) がその範囲に含まれるかを`sockaddr.is_contained`で判定し、`rule when`でログインパスのときだけ評価します。

```python
import "sockaddr"
import "strings"

# 社内ネットワークとして許可する CIDR
allowed_cidr = "10.0.0.0/8"

cidrcheck = rule {
    sockaddr.is_contained(allowed_cidr, request.connection.remote_addr)
}

main = rule when strings.has_prefix(request.path, "auth/userpass/login") {
    cidrcheck
}
```

ログインパスに紐付けて登録します。

```console
$ vault write sys/policies/egp/userpass-cidr \
    policy=@userpass-cidr.sentinel \
    paths="auth/userpass/login/*" \
    enforcement_level="hard-mandatory"
Success! Data written to: sys/policies/egp/userpass-cidr
```

このハンズオン環境では Vault へローカル (`127.0.0.1`) から接続しているため、許可 CIDR の`10.0.0.0/8`には含まれません。ログインを試すと拒否されます。

```console
$ vault login -method=userpass username=alice password=pass
Error authenticating: Error making API request.

URL: PUT http://127.0.0.1:8200/v1/auth/userpass/login/alice
Code: 400. Errors:

* 2 errors occurred:
	* egp standard policy "root/userpass-cidr" evaluation resulted in denial.
The specific error was:
<nil>
	* permission denied
```

逆に、許可 CIDR に自分の接続元を含めれば通ります。`allowed_cidr`を`127.0.0.1/32`に変えて同じパスに上書き登録し、再度ログインしてみましょう。

```console
$ vault write sys/policies/egp/userpass-cidr \
    policy=@userpass-cidr-local.sentinel \
    paths="auth/userpass/login/*" \
    enforcement_level="hard-mandatory"
Success! Data written to: sys/policies/egp/userpass-cidr

$ vault login -method=userpass username=alice password=pass
Success! You are now authenticated. The token information displayed below
is already stored in the token helper.

Key                    Value
---                    -----
token                  hvs.CAESI....
token_policies         ["default"]
...
```

同じユーザ・同じ認証情報でも、接続元 IP が許可範囲外ならログイン自体が成立しません。ACL では表現できない「どこからアクセスしているか」という条件を、Sentinel なら認証の段階で強制できます。確認が済んだらポリシーを削除しておきます。

```console
$ vault delete sys/policies/egp/userpass-cidr
Success! Data deleted (if it existed) at: sys/policies/egp/userpass-cidr
```

Sentinel を使うと、このように Policy (ACL) だけでは表現しきれない組織のコンプライアンス要件やインシデント対応のロジックを、Vault 側でコードとして強制できます。

### 参考リンク
* [Namespaces](https://developer.hashicorp.com/vault/docs/enterprise/namespaces)
* [Policies](https://www.vaultproject.io/docs/concepts/policies.html)
* [Policy API Document](https://www.vaultproject.io/api/system/policy.html)
* [Sentinel](https://developer.hashicorp.com/vault/docs/enterprise/sentinel)

---

## Terraform 連携

ここまで Vault を CLI や API から使ってきましたが、インフラをコードで管理する Terraform と組み合わせると、Vault の価値はさらに高まります。ここでは Terraform から Vault の動的クレデンシャルを使って AWS に`apply`する方法と、Terraform 1.10 以降の **ephemeral** リソースを使ってシークレットを state に残さずに利用する方法を扱います。

### Terraform のインストール

Terraform も HashiCorp の公式リポジトリから`dnf`でインストールできます ([Vault セットアップ](#vault-セットアップ) でリポジトリを追加済みなら、追加手順は不要です)。

```console
$ sudo dnf install -y dnf-plugins-core
$ sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
$ sudo dnf install -y terraform

$ terraform version
Terraform v1.16.5
on linux_arm64
```

### 動的クレデンシャルによる apply

Terraform で AWS にリソースを作る際、通常は長期間有効なアクセスキーを環境変数や`provider`ブロックに書きます。これはキーの管理と漏洩リスクという課題を抱えています。そこで、[AWS Secret Engine](#aws-secret-engine) で設定した動的クレデンシャル (`subnet-role`) を Terraform から読み出し、その短命なクレデンシャルで AWS プロバイダを認証させます。ここでは [Vault への AWS 権限付与](#vault-への-aws-権限付与) で用意した権限の範囲で、**事前作成済みの VPC にサブネットを 1 つ作成** してみます。

作業用ディレクトリに`main.tf`を作成します。`<VPC_ID>` はサブネットを作成する既存 VPC の ID に置き換えてください。`cidr_block` は対象 VPC の CIDR 範囲内の空きレンジにしてください。

```hcl
terraform {
  required_version = ">= 1.10.0"
  required_providers {
    vault = { source = "hashicorp/vault" }
    aws   = { source = "hashicorp/aws" }
  }
}

provider "vault" {}

# AWS Secret Engine の assumed_role ロール (subnet-role) から
# STS の一時クレデンシャルを動的に発行する
data "vault_aws_access_credentials" "creds" {
  backend = "aws"
  role    = "subnet-role"
  type    = "sts"
}

# 発行された短命なクレデンシャルで AWS プロバイダを認証する
# assumed_role は STS の一時クレデンシャルのため session token も渡す
provider "aws" {
  region     = "ap-northeast-1"
  access_key = data.vault_aws_access_credentials.creds.access_key
  secret_key = data.vault_aws_access_credentials.creds.secret_key
  token      = data.vault_aws_access_credentials.creds.security_token
}

# 事前作成済みの VPC にサブネットを作成する
resource "aws_subnet" "handson" {
  vpc_id            = "<VPC_ID>"
  cidr_block        = "10.0.110.0/24"
  availability_zone = "ap-northeast-1a"
  tags = { Name = "vault-handson-subnet" }
}

output "subnet_id" {
  value = aws_subnet.handson.id
}
```

`VAULT_ADDR` と `VAULT_TOKEN` を設定して`init`→`apply`します。

```console
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ export VAULT_TOKEN=$(vault print token)

$ terraform init
...
Terraform has been successfully initialized!

$ terraform apply -auto-approve
data.vault_aws_access_credentials.creds: Reading...
data.vault_aws_access_credentials.creds: Read complete after 1s [id=...]
...
aws_subnet.handson: Creating...
aws_subnet.handson: Creation complete after 1s [id=subnet-051fbd108f81654ea]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

subnet_id = "subnet-051fbd108f81654ea"
```

実行の裏側では、[AWS Secret Engine](#aws-secret-engine) の章で見たのと同じように、Vault が `handson-assume-role` を AssumeRole した STS の一時クレデンシャルが払い出され、そのクレデンシャルでサブネットが作成されます。リースが切れるとクレデンシャルは自動的に失効します。これにより、Terraform の実行ごとにユニークで短命なクレデンシャルが使われ、長期間有効なキーを一切保持する必要がなくなります。

> **`terraform destroy` について:** [Vault への AWS 権限付与](#vault-への-aws-権限付与) で `handson-assume-role` に `ec2:DeleteSubnet` を含めているため、同じ Vault 動的クレデンシャルで `terraform destroy` を実行すれば、作成したサブネットを削除できます。手動で作成したサブネットの削除も含め、まとめた手順は [クリーンアップ (削除手順)](#クリーンアップ-削除手順) を参照してください。

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
-rw-------. 1 ec2-user ec2-user 534K Oct  7 17:31 vault-20261007.snap
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
== Secret Path ==
kv/data/iam

======= Metadata =======
Key                Value
---                -----
created_time       2026-10-07T17:09:28.123943402Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            6

====== Data ======
Key         Value
---         -----
name        kabu
password    passwd
```

取得時点のデータが復元されていることが確認できれば完了です。バックアップは「取得できること」だけでなく「正しくリストアできること」までを定期的に検証して初めて意味を持ちます。半期に一度など、リストアのリハーサルを運用手順に組み込んでおきましょう。

### 参考リンク
* [vault operator raft snapshot](https://developer.hashicorp.com/vault/docs/commands/operator/raft)
* [Integrated Storage](https://developer.hashicorp.com/vault/docs/concepts/integrated-storage)
* [Automated Integrated Storage Snapshots](https://developer.hashicorp.com/vault/docs/enterprise/automated-integrated-storage-snapshots)

---

## クリーンアップ (削除手順)

ハンズオンで作成したリソースを削除します。課金や権限の残存を避けるため、不要になったら必ず後片付けをしてください。

**Vault 本体の中の設定 (シークレットエンジン・認証メソッド・ポリシー・Sentinel・Secret Sync など) は、個別に消す必要はありません。** 本ハンズオンの Vault は専用の EC2 インスタンス上で動いており、最後にインスタンスごと破棄するため、Vault 内部の後片付けは不要です。残るのは **AWS 側に作られたリソース** と **EC2 インスタンス本体** です。

### AWS 側の後片付け

#### 1. 作成したサブネットを削除する

[Terraform 連携](#terraform-連携) で `handson-assume-role` に `ec2:DeleteSubnet` を含めているため、Terraform で作成したサブネットは **同じ Vault 動的クレデンシャルで `terraform destroy`** できます。

```console
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ export VAULT_TOKEN=$(vault print token)
$ cd /path/to/tf-subnet
$ terraform destroy -auto-approve
```

手動の `aws ec2 create-subnet` で作成したサブネットが残っている場合は、個別に削除します。発行した Vault クレデンシャル (または削除権限を持つプロファイル) で実行してください。

```console
$ aws ec2 delete-subnet --subnet-id <SUBNET_ID> --region ap-northeast-1
```

#### 2. IAM ロールとインラインポリシーを削除する

手動のマネジメントコンソールでも削除できるよう、**削除対象の名前を明示** します。以下の名前のものを探して削除してください (IAM 変更権限を持つプロファイルで実行)。

**削除するロール:**

* **`handson-assume-role`** — [Vault への AWS 権限付与](#vault-への-aws-権限付与) で作成した AssumeRole 先の対象ロール。**まるごと削除します。**

**`handson-assume-role` に紐づくインラインポリシー:**

* **`handson-assume-permissions`** — サブネット作成/削除などの許可ポリシー (ロールを削除すれば一緒に消えます)

**`HandsonRole` (EC2 のインスタンスロール) に追加したインラインポリシー 3 つ:**

* **`VaultAwsEc2Auth`** — AWS Auth (ec2 method) 用 (`ec2:DescribeInstances` ほか)
* **`VaultAssumeSubnetRole`** — `handson-assume-role` を AssumeRole するための許可
* **`VaultSecretSync`** — Secret Sync の Secrets Manager 操作用

> **`HandsonRole` 本体は削除しないでください。** これは EC2 のコンソールアクセス (SSM) に使っているロールです。ハンズオンで追加した上記 3 つのインラインポリシーだけを外します。

CLI で削除する場合は以下のとおりです (`<...>` の名前は上記のものです)。

```console
# handson-assume-role: インラインポリシーを外してからロールを削除
$ aws iam delete-role-policy --role-name handson-assume-role --policy-name handson-assume-permissions
$ aws iam delete-role --role-name handson-assume-role

# HandsonRole に追加した 3 つのインラインポリシーを外す (ロール自体は残す)
$ aws iam delete-role-policy --role-name HandsonRole --policy-name VaultAwsEc2Auth
$ aws iam delete-role-policy --role-name HandsonRole --policy-name VaultAssumeSubnetRole
$ aws iam delete-role-policy --role-name HandsonRole --policy-name VaultSecretSync
```

> Secret Sync を試した場合は、AWS Secrets Manager 側に `vault/...` という名前のシークレットが作成されています。不要であれば `aws secretsmanager delete-secret` で合わせて削除してください。

### EC2 インスタンスの削除

最後に、Vault が動いていた **EC2 インスタンスを終了 (terminate)** します。これにより Vault 本体・設定・データ・スナップショット・鍵ファイルを含め、インスタンス内のものはすべて消えます。インスタンスプロファイルの関連付けも自動的に解除されます。

```console
$ aws ec2 terminate-instances --instance-ids <INSTANCE_ID> --region ap-northeast-1
```

> Vault の `init` 時に生成した Unseal Key / Root Token を、スクリーンショットや別ファイルなどインスタンス外に控えている場合は、それらも忘れずに破棄してください。
