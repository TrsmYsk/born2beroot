*This project has been created as part of the 42 curriculum by yterashi.*

# Born2beRoot

## Description

Born2beRoot は、仮想マシン上に Linux サーバーを構築し、Linux のシステム管理、仮想化、ユーザー管理、セキュリティ、ネットワーク、ストレージ管理などの基礎を学ぶことを目的とした 42 のプロジェクトです。

本プロジェクトでは VirtualBox 上に Debian をインストールし、GUI を使用しない最小構成のサーバーを構築しました。

主な構成は以下の通りです。

- OS: Debian
- 仮想化ソフトウェア: VirtualBox
- 仮想ディスク: 20 GB
- ストレージ管理: 暗号化 + LVM
- SSH: TCP port 4242
- Firewall: UFW
- Mandatory Access Control: AppArmor
- 権限管理: sudo
- 定期処理: cron
- 監視スクリプト: `/usr/local/bin/monitoring.sh`
- Hostname: `yterashi42`

### OS に Debian を選択した理由

Debian は安定性を重視した Linux ディストリビューションであり、サーバー用途で広く利用されています。

また、パッケージ管理に `apt` を利用でき、設定方法やドキュメントも豊富です。

Born2beRoot を通して Linux のシステム管理を学ぶことが目的であるため、比較的シンプルで理解しやすい Debian を選択しました。

### ディスク構成

20 GB の仮想ディスクを用意し、暗号化領域の内部に LVM を構成しました。

LVM は以下の3層でストレージを管理します。

```text
Physical Volume (PV)
        ↓
Volume Group (VG)
        ↓
Logical Volume (LV)
```

LVM を利用することで、通常の固定パーティションより柔軟にストレージ領域を管理できます。

本環境では root、home、swap などを論理ボリュームとして管理しています。

### セキュリティ

サーバーのセキュリティを高めるため、以下の設定を行いました。

- root による SSH ログインを禁止
- SSH の待受ポートを 4242 に変更
- UFW により 4242 番ポートのみ許可
- AppArmor を有効化
- 強力なパスワードポリシーを設定
- sudo の試行回数を制限
- sudo の入出力をログとして記録
- sudo 実行時の `secure_path` を設定
- sudo 実行時に TTY を要求

### ユーザー管理

root とは別に通常ユーザー `yterashi` を作成し、以下のグループに所属させています。

```text
sudo
user42
```

通常の作業は一般ユーザーで行い、管理者権限が必要な処理だけ `sudo` を利用します。

これは必要な場合にだけ高い権限を使用する「最小権限の原則」を意識した構成です。

---

## Technology Comparison

### Debian vs Rocky Linux

#### Debian

メリット:

- 安定性が高い
- `apt` によるパッケージ管理が分かりやすい
- 情報やドキュメントが豊富
- Linux 初学者でも比較的扱いやすい

デメリット:

- 最新ソフトウェアの導入が遅い場合がある
- RHEL 系の企業環境とは管理方法が異なる

#### Rocky Linux

Rocky Linux は Red Hat Enterprise Linux（RHEL）との互換性を重視した Linux ディストリビューションです。

メリット:

- RHEL 系のサーバー管理を学べる
- Enterprise Linux に近い環境を構築できる
- SELinux を標準的に利用する

デメリット:

- Debian と比較して初学者には設定が複雑になりやすい
- SELinux や firewalld など理解すべき概念が増える

本プロジェクトでは、Linux システム管理の基本を理解することを優先して Debian を選択しました。

---

### AppArmor vs SELinux

AppArmor と SELinux は、どちらも通常の Linux のファイルパーミッションに加えてアクセス制御を行う Mandatory Access Control（MAC）の仕組みです。

#### AppArmor

AppArmor は主にファイルパスとアプリケーションごとの profile に基づいてアクセスを制御します。

メリット:

- 設定が比較的理解しやすい
- ファイルパスを基準にルールを記述できる
- Debian / Ubuntu 系で利用しやすい

デメリット:

- SELinux と比較すると複雑なセキュリティポリシーの表現力が限定される場合がある

#### SELinux

SELinux はプロセスやファイルに security label / type を設定し、policy に基づいてアクセスを制御します。

メリット:

- 非常に細かいアクセス制御が可能
- 大規模・複雑なサーバー環境にも対応しやすい

デメリット:

- security context や type enforcement などの概念が必要
- 設定やトラブルシューティングが複雑

本環境では Debian を利用しているため AppArmor を使用しています。

---

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

本環境では SSH 用の TCP port 4242 のみを許可しています。

```bash
sudo ufw status verbose
```

#### firewalld

firewalld は zone という概念を使用してネットワークごとの信頼レベルを管理できます。

メリット:

- 複雑なネットワーク構成に対応しやすい
- zone 単位でルールを整理できる
- 動的な Firewall 設定に適している

デメリット:

- UFW と比較して設定概念が複雑

Rocky Linux では firewalld が利用されます。

---

### VirtualBox vs UTM

どちらも仮想マシンを作成・実行するためのソフトウェアです。

#### VirtualBox

VirtualBox は Oracle が提供する仮想化ソフトウェアです。

メリット:

- Linux、Windows、macOS などで利用できる
- 利用者が多く情報が豊富
- 仮想ネットワークや仮想ディスクの設定が行いやすい

デメリット:

- Apple Silicon 環境などでは制約がある場合がある

本プロジェクトでは VirtualBox を利用しました。

#### UTM

UTM は特に macOS / Apple Silicon 環境で利用される仮想化ソフトウェアです。

メリット:

- Apple Silicon との相性がよい
- ARM 環境でも利用しやすい
- QEMU を利用した仮想化・エミュレーションが可能

デメリット:

- VirtualBox と操作方法や仮想ディスク形式が異なる
- x86 Linux PC では VirtualBox の方が一般的な場合がある

---

## apt vs aptitude

`apt` と `aptitude` は、どちらも Debian のパッケージ管理に利用できるツールです。

### apt

現在の Debian で一般的に使用されるパッケージ管理コマンドです。

```bash
sudo apt update
sudo apt install <package>
sudo apt remove <package>
```

シンプルで日常的なパッケージ管理に適しています。

### aptitude

`aptitude` はコマンドラインに加えて対話型インターフェースも持つパッケージ管理ツールです。

複雑な依存関係について複数の解決案を提示できる場合があります。

通常のシステム管理では `apt` を使用し、必要に応じて `aptitude` を利用できます。

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

VirtualBox の NAT port forwarding を利用している環境では、host から以下のように接続できます。

```bash
ssh -p 4242 yterashi@127.0.0.1
```

SSH の実効設定は以下で確認できます。

```bash
sudo sshd -T | grep -E '^(port|permitrootlogin)'
```

期待する設定:

```text
port 4242
permitrootlogin no
```

### Firewall の確認

```bash
sudo ufw status verbose
```

UFW が active であり、SSH 用の 4242 番ポートのみ許可されていることを確認します。

### AppArmor の確認

```bash
systemctl is-active apparmor
systemctl is-enabled apparmor
sudo aa-status
```

### sudo の確認

```bash
sudo visudo -c
sudo -l
sudo env | grep '^PATH='
sudo find /var/log/sudo -type f
```

### パスワードポリシーの確認

```bash
grep -E '^PASS_(MAX|MIN|WARN)_DAYS' /etc/login.defs
sudo chage -l yterashi
sudo chage -l root
```

パスワード強度設定:

```bash
sudo grep -Ev '^[[:space:]]*#|^[[:space:]]*$' /etc/security/pwquality.conf
```

PAM 設定:

```bash
grep pwquality /etc/pam.d/common-password
```

### LVM の確認

```bash
lsblk
sudo pvs
sudo vgs
sudo lvs
```

### cron の確認

```bash
systemctl status cron
sudo crontab -l
```

root の crontab では `monitoring.sh` を起動時および10分ごとに実行します。

```cron
@reboot /usr/local/bin/monitoring.sh
*/10 * * * * /usr/local/bin/monitoring.sh
```

---

## Monitoring

`/usr/local/bin/monitoring.sh` にサーバーの状態を表示する Bash script を配置しています。

この script は `cron` によって起動時および10分ごとに実行され、`wall` を利用してログイン中のすべての terminal に情報を送信します。

表示する主な情報は以下です。

- OS architecture と kernel version
- Physical CPU 数
- Virtual CPU 数
- RAM 使用量・使用率
- Disk 使用量・使用率
- CPU 使用率
- 最終 reboot 時刻
- LVM の状態
- TCP connection 数
- Login user 数
- IPv4 address
- MAC address
- sudo command 実行回数

cron service の確認:

```bash
systemctl is-active cron
```

monitoring script の手動実行:

```bash
sudo /usr/local/bin/monitoring.sh
```

---

## Useful Commands

### User の作成

```bash
sudo adduser new_user
```

### Group の作成

```bash
sudo groupadd new_group
```

### User を Group に追加

```bash
sudo usermod -aG new_group new_user
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

```bash
sudo hostnamectl set-hostname newhostname
```

---

## Signature

提出時には VirtualBox の仮想ディスク `.vdi` ファイルから SHA-1 hash を取得し、その値を `signature.txt` に保存します。

例:

```bash
sha1sum Born2beRoot.vdi
```

`signature.txt` には SHA-1 の値のみを記載します。

仮想マシンを再度起動すると `.vdi` の内容が変化し、SHA-1 signature も変化する可能性があるため、最終 signature の生成後は注意が必要です。

仮想ディスクそのものは Git repository には追加しません。

---

## Resources

このプロジェクトでは、以下の資料・コマンドの manual を参考にしました。

- Born2beRoot Subject Version 5.2
- Debian documentation
- VirtualBox documentation
- AppArmor documentation
- OpenSSH documentation
- `man ssh`
- `man sshd`
- `man sshd_config`
- `man sudo`
- `man sudoers`
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
- 設定内容を確認するための Linux コマンドの調査
- Born2beRoot の peer evaluation で想定される質問の整理
- README の構成と文章の整理

AI が提示した内容をそのまま利用するのではなく、実際の仮想マシン上でコマンドを実行し、設定ファイルやサービスの状態を確認しながら内容を検証しました。
