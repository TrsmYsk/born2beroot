*This project has been created as part of the 42 curriculum by yterashi.*

# Born2beRoot

## Description

Born2beRoot は、仮想マシン上に Linux サーバーを構築し、仮想化、Linux のシステム管理、ユーザー管理、ストレージ管理、ネットワーク、認証、アクセス制御、ログ管理などの基礎を学ぶ 42 のプロジェクトです。

本プロジェクトでは VirtualBox 上に Debian をインストールし、GUI を使用しない最小構成のサーバーを構築しました。

主な構成は以下の通りです。

- OS: Debian
- 仮想化ソフトウェア: VirtualBox
- 仮想ディスク: 20 GB
- ストレージ管理: 暗号化 + LVM
- SSH: OpenSSH Server / TCP port 4242
- Firewall: UFW
- Mandatory Access Control: AppArmor
- 権限管理: sudo
- 認証・パスワード品質管理: PAM / libpam-pwquality
- 定期実行: cron
- 監視スクリプト: `/usr/local/bin/monitoring.sh`
- Hostname: `yterashi42`

### OS に Debian を選択した理由

Debian は安定性を重視した Linux ディストリビューションであり、サーバー用途でも広く利用されています。

また、`apt` / `dpkg` によるパッケージ管理が分かりやすく、公式ドキュメントや利用者向けの情報も豊富です。

Born2beRoot では Linux システム管理の基本を理解することを優先したため、比較的シンプルに構成を理解しやすい Debian を選択しました。

### ディスク構成

20 GB の仮想ディスクを用意し、暗号化領域の内部に LVM を構成しました。

LVM は主に以下の3層でストレージを管理します。

```text
Physical Volume (PV)
        ↓
Volume Group (VG)
        ↓
Logical Volume (LV)
```

LVM を利用することで、物理パーティションを直接固定的に使う場合よりも柔軟にストレージ領域を管理できます。

本環境では root、home、swap などを Logical Volume として管理しています。

### セキュリティ方針

サーバーのセキュリティを高めるため、以下の設定を行いました。

- root による SSH ログインを禁止
- SSH の待受ポートを 4242 に設定
- UFW により 4242 番ポートのみ許可
- AppArmor を起動時から有効化
- 強力なパスワードポリシーを設定
- sudo の認証失敗回数を3回までに制限
- sudo 認証失敗時のカスタムメッセージを設定
- sudo のイベントログを `/var/log/sudo/sudo.log` に保存
- sudo の入力・出力を I/O ログとして `/var/log/sudo/` 配下に保存
- `sudoreplay` により記録された sudo セッションを確認可能に設定
- sudo 実行時に TTY を要求
- sudo 実行時の `secure_path` を制限

### ユーザー管理

root とは別に通常ユーザー `yterashi` を作成し、`sudo` グループと `user42` グループに所属させています。

通常作業は一般ユーザーで行い、管理者権限が必要な場合のみ `sudo` を使用します。

これは、必要な操作に必要な権限だけを与える「最小権限の原則」を意識した構成です。

---

## インストール・利用している主要サービス / 機構

本プロジェクトでは、最小構成のサーバーを維持するため、必要なサービス・管理機構のみを利用しています。

### OpenSSH Server

OpenSSH Server は、SSH（Secure Shell）を利用してネットワーク越しにサーバーへ安全にログインするためのサービスです。

サーバー側では `sshd` が接続を待ち受けます。本環境では TCP port `4242` のみを使用し、root ユーザーによる SSH ログインは禁止しています。

確認例:

```bash
systemctl is-active ssh
systemctl is-enabled ssh
sudo sshd -T | grep -E '^(port|permitrootlogin)'
sudo ss -ltnp | grep 4242
```

ホスト側からの接続例:

```bash
ssh -p 4242 yterashi@127.0.0.1
```

### UFW

UFW（Uncomplicated Firewall）は、Linux の Firewall ルールを簡潔に管理するためのツールです。

本環境では UFW を有効にし、SSH に使用する TCP port `4242` のみを許可しています。

確認例:

```bash
sudo ufw status verbose
systemctl is-enabled ufw
```

ルールの追加例:

```bash
sudo ufw allow 4242/tcp
```

このコマンドは TCP の 4242 番ポートへの接続を許可します。

ルールの削除は、追加時と同じルールを `delete` とともに指定できます。

```bash
sudo ufw delete allow 4242/tcp
```

また、ルール番号を確認して番号指定で削除することもできます。

```bash
sudo ufw status numbered
sudo ufw delete <rule_number>
```

例:

```bash
sudo ufw status numbered
sudo ufw delete 2
```

Born2beRoot では不要なポートを開けず、最終的に 4242/tcp のみが許可されていることを確認します。

### AppArmor

AppArmor は Linux の Mandatory Access Control（MAC）機構です。

通常の owner / group / permission によるアクセス制御に加え、アプリケーションごとの profile に基づいてアクセス可能なリソースを制限できます。

本環境では Debian の要件に従って、AppArmor を起動時から有効にしています。

確認例:

```bash
cat /sys/module/apparmor/parameters/enabled
systemctl is-active apparmor
systemctl is-enabled apparmor
sudo aa-status
```

### cron

cron は、指定した時刻・間隔でコマンドやスクリプトを自動実行するためのスケジューラです。

本環境では `/usr/local/bin/monitoring.sh` をサーバー起動時と10分ごとに実行します。

root の crontab:

```cron
@reboot /usr/local/bin/monitoring.sh
*/10 * * * * /usr/local/bin/monitoring.sh
```

確認例:

```bash
systemctl is-active cron
systemctl is-enabled cron
sudo crontab -l
```

### sudo

`sudo` は、許可された一般ユーザーが必要なコマンドだけを別ユーザー、通常は root の権限で実行するための仕組みです。

本環境では `/etc/sudoers.d/born2beroot` で主に以下を設定しています。

```text
Defaults passwd_tries=3
Defaults badpass_message="Incorrect password"
Defaults logfile="/var/log/sudo/sudo.log"
Defaults log_input,log_output
Defaults iolog_dir="/var/log/sudo"
Defaults requiretty
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
```

`logfile` は、誰が・いつ・どの sudo コマンドを実行したかというイベントログを記録します。

`log_input` / `log_output` と `iolog_dir` は、sudo セッションの入出力を I/O ログとして保存します。この I/O ログは `sudoreplay` で一覧表示・再生できます。

確認例:

```bash
sudo visudo -c
sudo -l
sudo cat /var/log/sudo/sudo.log
sudo find /var/log/sudo -type f
sudo sudoreplay -d /var/log/sudo -l
```

### PAM / libpam-pwquality

PAM（Pluggable Authentication Modules）は、Linux のサービスが共通して利用できる認証フレームワークです。

本プロジェクトでは `pam_pwquality` を利用して、パスワード変更時に長さ、文字種、連続文字、ユーザー名、旧パスワードとの差などを検査します。

主な設定ファイル:

```text
/etc/pam.d/common-password
/etc/security/pwquality.conf
```

確認例:

```bash
grep pwquality /etc/pam.d/common-password
sudo grep -Ev '^[[:space:]]*#|^[[:space:]]*$' /etc/security/pwquality.conf
```

### monitoring.sh

`/usr/local/bin/monitoring.sh` は Bash で作成したサーバー監視スクリプトです。

`wall` を利用して、サーバー起動時および10分ごとにログイン中の端末へ以下の情報を表示します。

- OS architecture と kernel version
- Physical CPU 数
- Virtual CPU 数
- RAM 使用量・使用率
- Disk 使用量・使用率
- CPU 使用率
- 最終 reboot 時刻
- LVM が有効かどうか
- Active connection 数
- サーバーを利用しているユーザー数
- IPv4 address と MAC address
- sudo で実行された command 数

手動確認:

```bash
sudo /usr/local/bin/monitoring.sh
```

---

## Technology Comparison

### Debian vs Rocky Linux

#### Debian

メリット:

- 安定性を重視している
- `apt` / `dpkg` によるパッケージ管理が分かりやすい
- ドキュメントや利用者向け情報が豊富
- Linux / system administration 初学者でも比較的扱いやすい

デメリット:

- Stable では最新バージョンのソフトウェア導入が遅い場合がある
- RHEL 系の企業環境とは管理方法やツールが異なる

#### Rocky Linux

Rocky Linux は Red Hat Enterprise Linux（RHEL）との互換性を重視した Linux ディストリビューションです。

メリット:

- RHEL 系のサーバー管理を学べる
- Enterprise Linux に近い環境を構築できる
- SELinux や firewalld など RHEL 系で一般的な仕組みを学べる

デメリット:

- Debian と比較して初学者には設定が複雑になりやすい
- SELinux など理解すべき概念が増える

本プロジェクトでは、Linux システム管理の基本を理解することを優先して Debian を選択しました。

### AppArmor vs SELinux

AppArmor と SELinux は、どちらも通常の Linux の DAC（Discretionary Access Control）に加えてアクセス制御を行う Mandatory Access Control（MAC）の仕組みです。

#### AppArmor

AppArmor は主にアプリケーションごとの profile とファイルパスを基準にアクセスを制御します。

メリット:

- 設定を比較的理解しやすい
- ファイルパスを基準にルールを記述できる
- Debian / Ubuntu 系で利用しやすい

デメリット:

- SELinux と比較すると複雑なセキュリティポリシーの表現力が限定される場合がある

#### SELinux

SELinux はプロセスやファイルに security label / type を付与し、policy に基づいてアクセスを制御します。

メリット:

- 非常に細かいアクセス制御が可能
- 複雑なサーバー環境にも対応しやすい

デメリット:

- security context、domain、type enforcement などの概念が必要
- 設定やトラブルシューティングが複雑

本環境では Debian を利用しているため AppArmor を使用しています。

### UFW vs firewalld

どちらも Linux の Firewall 設定を管理するためのツールです。

#### UFW

UFW は Uncomplicated Firewall の略で、シンプルな Firewall 設定を目的としています。

メリット:

- コマンドが簡潔
- 設定内容を理解しやすい
- 小規模なサーバーの Firewall 管理に適している

デメリット:

- 複雑なネットワーク構成では firewalld より柔軟性が低い場合がある

#### firewalld

firewalld は zone という概念を利用してネットワークごとの信頼レベルを管理できます。

メリット:

- 複雑なネットワーク構成に対応しやすい
- zone 単位でルールを整理できる
- 動的な Firewall 設定に適している

デメリット:

- UFW と比較して設定概念が複雑

Rocky Linux では firewalld を使用します。

### VirtualBox vs UTM

どちらも仮想マシンを作成・実行するための仮想化ソフトウェアです。

#### VirtualBox

VirtualBox は Oracle が提供する仮想化ソフトウェアです。

メリット:

- Linux、Windows、macOS などで利用できる
- 利用者が多く情報が豊富
- 仮想ネットワークや仮想ディスクを GUI から管理しやすい

デメリット:

- Apple Silicon 環境などでは制約がある場合がある

本プロジェクトでは VirtualBox を利用しました。

#### UTM

UTM は特に macOS / Apple Silicon 環境で利用される仮想化ソフトウェアです。

メリット:

- Apple Silicon 環境で利用しやすい
- QEMU を利用した仮想化・エミュレーションが可能

デメリット:

- VirtualBox と操作方法や仮想ディスク形式が異なる
- x86 Linux PC では VirtualBox の方が一般的な場合がある

---

## apt vs aptitude

`apt` と `aptitude` は、どちらも Debian のパッケージ管理に利用できるツールです。

### apt

`apt` は、現在の Debian で日常的なパッケージ管理に広く使われるコマンドです。

```bash
sudo apt update
sudo apt install <package>
sudo apt remove <package>
```

シンプルで、一般的なインストール・更新・削除に適しています。

### aptitude

`aptitude` はコマンドライン操作に加えて対話型インターフェースも持つパッケージ管理ツールです。

依存関係に問題がある場合、複数の解決案を提示できることがあります。

通常のシステム管理では `apt` を利用し、必要に応じて `aptitude` を利用できます。

---

## Instructions

### Virtual Machine の起動

VirtualBox を起動し、`Born2beRoot` 仮想マシンを起動します。

OS 起動後、通常ユーザーとしてログインします。

```text
login: yterashi
```

### SSH 接続

SSH server は TCP port 4242 で動作します。

VirtualBox の NAT port forwarding を利用している場合、ホストから以下のように接続できます。

```bash
ssh -p 4242 yterashi@127.0.0.1
```

SSH の実効設定確認:

```bash
sudo sshd -T | grep -E '^(port|permitrootlogin)'
```

期待する設定:

```text
port 4242
permitrootlogin no
```

レビュー時に新しいユーザーを作成した場合も、そのユーザーが SSH 側で明示的に拒否されていなければ、通常の Linux ユーザーとして同じ SSH サーバーを利用してログインできます。

### Firewall の確認

```bash
sudo ufw status verbose
```

UFW が active であり、不要な port が開いていないことを確認します。

### AppArmor の確認

```bash
cat /sys/module/apparmor/parameters/enabled
systemctl is-active apparmor
systemctl is-enabled apparmor
sudo aa-status
```

### sudo の確認

```bash
sudo visudo -c
sudo -l
sudo env | grep '^PATH='
sudo cat /var/log/sudo/sudo.log
sudo find /var/log/sudo -type f
sudo sudoreplay -d /var/log/sudo -l
```

### パスワードポリシーの確認

有効期限:

```bash
grep -E '^PASS_(MAX_DAYS|MIN_DAYS|WARN_AGE)' /etc/login.defs
sudo chage -l yterashi
sudo chage -l root
```

パスワード品質:

```bash
sudo grep -Ev '^[[:space:]]*#|^[[:space:]]*$' /etc/security/pwquality.conf
grep pwquality /etc/pam.d/common-password
```

### LVM / ディスク構成の確認

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
sudo pvs
sudo vgs
sudo lvs
```

### cron の確認

```bash
systemctl is-active cron
systemctl is-enabled cron
sudo crontab -l
```

### monitoring.sh の停止・再開

スクリプト自体を編集せず、cron service を停止することで定期実行を止められます。

```bash
sudo systemctl stop cron
```

再開:

```bash
sudo systemctl start cron
```

---

## Useful Commands

### User の作成

```bash
sudo adduser new_user
```

### Group の作成

```bash
sudo addgroup new_group
```

### User を Group に追加

```bash
sudo adduser new_user new_group
```

確認:

```bash
id new_user
groups new_user
```

### User の削除

```bash
sudo deluser --remove-home new_user
```

### Group の削除

```bash
sudo delgroup new_group
```

### Hostname の確認

```bash
hostname
```

### Hostname の変更

hostname を変更する場合は、`/etc/hosts` の `127.0.1.1` に対応する名前も一致させ、ローカルの名前解決に不整合が生じないようにします。

```bash
sudo nano /etc/hosts
sudo hostnamectl set-hostname newhostname
```

---

## Signature

提出時には VirtualBox の仮想ディスク `.vdi` ファイルから SHA-1 hash を取得し、その値を `signature.txt` に保存します。

Linux での例:

```bash
sha1sum Born2beRoot.vdi
```

`sha1sum` の出力は、SHA-1 値とファイル名の2列になります。

```text
<SHA-1 hash>  Born2beRoot.vdi
```

`signature.txt` には SHA-1 値だけを保存するため、`awk` で1列目を取り出します。

```bash
sha1sum Born2beRoot.vdi | awk '{print $1}'
```

そのまま `signature.txt` に書き込む場合:

```bash
sha1sum Born2beRoot.vdi | awk '{print $1}' > signature.txt
```

保存された内容を確認します。

```bash
cat signature.txt
```

`signature.txt` には40桁の SHA-1 値のみを記載します。

VM を起動すると仮想ディスクの内容が変化して signature が変わる可能性があるため、提出用 signature を取得した後は VM の扱いに注意します。

仮想マシン本体や `.vdi` ファイルは Git repository には追加しません。

各 evaluation は snapshot が存在しない状態から開始し、evaluation 用 snapshot は評価手順に従って作成・削除します。

---

## Resources

このプロジェクトでは、以下の資料や manual を参考にしました。

- Born2beRoot Subject Version 5.2
- Debian documentation
- VirtualBox documentation
- OpenSSH documentation
- AppArmor documentation
- sudo / sudoers / sudoreplay documentation
- libpam-pwquality documentation
- `man ssh`
- `man sshd`
- `man sshd_config`
- `man sudo`
- `man sudoers`
- `man sudoreplay`
- `man ufw`
- `man crontab`
- `man systemctl`
- `man passwd`
- `man chage`
- `man lsblk`
- `man lvm`

### AI の利用について

AI は主に以下の用途で利用しました。

- Linux のシステム管理に関する概念の理解
- Debian と Rocky Linux の違いの整理
- AppArmor と SELinux の違いの理解
- UFW と firewalld の違いの理解
- VirtualBox と UTM の違いの整理
- LVM、SSH、sudo、PAM、cron などの概念の学習
- sudo のイベントログ、I/O ログ、`sudoreplay` の仕組みの確認
- 設定内容を確認するための Linux コマンドの調査
- Born2beRoot の peer evaluation で想定される質問・操作の整理
- README の構成と文章の整理

AI が提示した内容をそのまま利用するのではなく、実際の仮想マシン上でコマンドを実行し、設定ファイル、サービス状態、ログ、SSH 接続などを確認しながら内容を検証しました。
