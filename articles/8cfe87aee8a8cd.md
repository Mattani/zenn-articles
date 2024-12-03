---
title: "Elevateを使用してRedmineサーバをCentOS7からRockyLinuxにアップデートしてみた（前編）"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Redmine","CentOS", "RockyLinux", "ELevate"]
published: true
---

## はじめに

2024年6月30日でCentOSがサポート終了になりましたが、CentOSを使っているサーバを更改しなきゃだけど、まだできていない、という方多いのではないでしょうか。すでにyumリポジトリも閉じていてアップデート等一切できない状態になっていて、さすがにこれはまずいな、という状況になっています。

仮想サーバで運用しているものは、一般には後継OSのVMを立ち上げてアプリケーションを再インストールし、必要なデータ移行を行って切り替えるのが一般的かとおもいます。

一方、AlmaLinuxにより、ELevateというツールが公開されているので、これを使ってCentOS7からAlmaLinux8等にアップデートする、という方法があります。しかし、OSをアップデートしてもアプリケーションがどうなるかとか、やはり心配ですよね。

そこで、すでに運用しているRedmineサーバでELevateを使用してOSをアップデートしてみるとどうなるか、やってみました。

なお、このツールはCentOSからAlmaLinuxにアップデートするだけでなく、そのほかのメジャーなディストリビューションへの移行にも対応しています。
私はRockyLinuxにアップデートすることにしました。自分のディストリビューション以外への移行にも対応するなんて、AlmaLinuxの懐の深さに感謝いたします😭

![ELvate](/images/ELevateNG.png)

（前提条件）rootアカウントでログインできていること
以下、プロンプトの#はrootアカウントです。

## 更改前の情報

OSはCentOS7.8、カーネルのバージョンは以下のとおりです。
カーネルについては、この後の手順内でyum updateするので実際にはCentOS7.9、```3.10.0-1160.119.1.el7.x86_64```からのアップデートになります。

```bash:（アップデート前）
# cat /etc/redhat-release
CentOS Linux release 7.8.2003 (Core)

# uname -r
3.10.0-1127.18.2.el7.x86_64
```

以下に示すとおり、少々Redmineが古いバージョンですが、Redmineのバージョンアップは今回の記事のスコープ外とします。

![Redmine_info](/images/Redmine_info.png)

## ELevateによるOSアップデートの概要

詳細は [公式サイト](https://wiki.almalinux.org/elevate/) を見ていただければとおもいますが、ざっくりいうと以下の流れとなります
公式サイトにも書かれていますが、事前にバックアップやスナップショットを取得するなどしておきましょう。

1.事前準備

* CentOSのリポジトリをAlmaLinuxが提供するURLに変更し、OSをアップデートする
* leappをインストール

2.環境の事前チェックと問題の解消

* leapp preupgradeの実行
* 発見されたエラー／ワーニングの対処方法を検討
* エラー／ワーニングを解消する、または対応方法をアンサーファイルに書きこむ
* 再度leapp preupgradeを実行して解消されたか確認する

3.アップグレードを実行

以下、実際にこの環境でアップグレードを実行していきます。

## 事前準備

[AlmaLinuxによるquick-start-guide](https://wiki.almalinux.org/elevate/ELevate-quickstart-guide.html)を見て、コマンドを実行していきます。

```bash
# sudo curl -o /etc/yum.repos.d/CentOS-Base.repo https://el7.repo.almalinux.org/centos/CentOS-Base.repo
# sudo yum update -y
# sudo reboot
（出力結果は割愛）
```

再起動後、再度Terminalを起動して、作業を継続します。今回はAlmaLinuxではなくRockyLinuxにアップグレードするのでleapp-data-rockyをインストールします。

```bash
# uname -r
3.10.0-1160.119.1.el7.x86_64
# cat /etc/redhat-release
CentOS Linux release 7.9.2009 (Core)
# sudo yum install -y http://repo.almalinux.org/elevate/elevate-release-latest-el$(rpm --eval %rhel).noarch.rpm
(出力結果は割愛)
# sudo yum install -y leapp-upgrade leapp-data-rocky
(出力結果は割愛)
```

## 環境の事前チェックと問題の解消

```bash
# sudo leapp preupgrade
```

大量に表示される途中表示は割愛しますが、最後に以下のようなレポートが表示されます。表示されているとおり、アップグレードの詳細ログは```/var/log/leapp/leapp-preupgrade.log```にも出力されています。

![preupgradeのエラーレポート](/images/leapp_preupgrade_error_report.png)

実際に表示される内容は、それぞれの実行環境によって異なると思いますが、ここでは私の環境で表示されたエラーについて、一つ一つ対処をしていきます。

### 問題点の確認と対処詳細

#### 1. Leapp detected loaded kernel drivers which have been removed in RHEL 8. Upgrade cannot proceed

```txt:/var/log/leapp/leapp-report.txt(抜粋,適宜改行をいれて見やすくしています)
Risk Factor: high (inhibitor)
Title: Leapp detected loaded kernel drivers which have been removed in RHEL 8. 
Upgrade cannot proceed.
Summary: Support for the following RHEL 7 device drivers has been removed in RHEL 8:
     - pata_acpi
```

RHEL8で削除されたカーネルドライバを検知した、とのことです。レポートの詳細を確認すると、```pata_acpi```というモジュールであることがわかります。
このドライバは不要なので削除します。

```bash
# sudo rmmod pata_acpi
```

#### 2. Possible problems with remote login using root account

```txt:/var/log/leapp/leapp-report.txt(抜粋,適宜改行をいれて見やすくしています)
Risk Factor: high (inhibitor)
Title: Possible problems with remote login using root account
Summary: OpenSSH configuration file does not explicitly state the option
 PermitRootLogin in sshd_config file, which will default in RHEL8 to
 "prohibit-password".
Remediation: [hint] If you depend on remote root logins using passwords,
consider setting up a different user for remote administration or adding 
"PermitRootLogin yes" to sshd_config. 
If this change is ok for you, add explicit "PermitRootLogin prohibit-password" 
to your sshd_config to ignore this inhibitor
```

ルートアカウントでのリモートログインが許可されている（RHEL8ではデフォルトでは```prohibit-passwor```になっている）とのことです。
私の環境ではrootで最初からログインすることはなく、一般ユーザからsuする使い方をしているので、明示的にprohibit-passwordに設定します。

```bash:/etc/ssh/sshd_config
(#PermitRootLogin yesという記述があるのでその下に以下を追加します)
PermitRootLogin prohibit-password
```

#### 3. Missing required answers in the answer file

```txt:/var/log/leapp/leapp-report.txt(抜粋,適宜改行をいれて見やすくしています)
Risk Factor: high (inhibitor)
Title: Missing required answers in the answer file
Summary: One or more sections in answerfile are missing user choices:
 remove_pam_pkcs11_module_check.confirm
For more information consult https://red.ht/leapp-dialogs.
Related links:
    - Leapp upgrade fail with error 
      "Inhibitor: Missing required answers in the answer file": https://access.redhat.com/solutions/7035321
Remediation: [hint] Please register user choices with leapp answer cli command
or by manually editing the answerfile.
[command] leapp answer --section remove_pam_pkcs11_module_check.confirm=True
```

出力のとおり、アンサーファイルに必要な回答がなされていない、ということです。出力で指示されているとおり、```leapp answer --section remove_pam_pkcs11_module_check.confirm=True```を実行するか、手動でアンサーファイルを編集します

```bash:/var/log/leapp/answerfile
[remove_pam_pkcs11_module_check]
# Title:              None
# Reason:             Confirmation
# =================== remove_pam_pkcs11_module_check.confirm ==================
# Label:              Disable pam_pkcs11 module in PAM configuration? If no, the upgrade process will be interrupted.
# Description:        PAM module pam_pkcs11 is no longer available in RHEL-8 since it was replaced by SSSD.
# Reason:             Leaving this module in PAM configuration may lock out the system.
# Type:               bool
# Default:            None
# Available choices: True/False
# Unanswered question. Uncomment the following line with your answer
confirm = True
```

上記が対応必須な問題点であり、これでpreupgradeをもう一度実行すればinhibited problemsのところが解消され、レポートが黄色表示になりました。

![preupgrade再実行](/images/leapp_preupgrade-warning_report.png)

### その他のリスクの確認

#### 1. Difference in Python versions and support in RHEL 8

```txt:/var/log/leapp/leapp-report.txt(抜粋,適宜改行をいれて見やすくしています)
Risk Factor: high
Title: Difference in Python versions and support in RHEL 8
Summary: In RHEL 8, there is no 'python' command.Python 3
 (backward incompatible) is the primary Python version
and Python 2 is available with limited support and limited set of packages.
If you no longer require Python 2 packages following the upgrade,
please remove them. Read more here: https://red.ht/rhel-8-python
Related links:
    - Difference in Python versions and support in RHEL 8:
     https://red.ht/rhel-8-python
Remediation: [hint] Please run "alternatives --set python /usr/bin/python3" after upgrade

```

RHEL8にはPythonコマンドはなく、Python3になっている。Python2は限定的に利用可能なだけ。Python2が必要ないのであればアップグレードを検討するべき。
→Python2はCentOSではシステムパッケージに依存している可能性があるため、拙速に削除するのは怖い。ヒントとしても表示されているとおり、アップグレード後に対応することとする。

[アップグレード後の対応へ移動](#アップグレード後の対応)

#### 2. GRUB2 core will be automatically updated during the upgrade

```txt:/var/log/leapp/leapp-report.txt(抜粋,適宜改行をいれて見やすくしています)
Risk Factor: high
Title: GRUB2 core will be automatically updated during the upgrade
Summary: On legacy (BIOS) systems, GRUB2 core (located in the gap between the MBR 
and the first partition) cannot be updated during the rpm transaction
and Leapp has to initiate the update running "grub2-install" after the transaction. 
No action is needed before the upgrade. After the upgrade, 
it is recommended to check the GRUB configuration.
```

GRUB2（ブートローダ）はOSアップグレードの際に自動的にアップデートされます、とのこと。アップグレード前には特にアクション不要。アップデート後にGRUB2設定を確認する。

[アップグレード後の対応へ移動](#アップグレード後の対応)

#### 3. Detected custom leapp actors or files

```txt:/var/log/leapp/leapp-report.txt(抜粋,適宜改行をいれて見やすくしています)
Risk Factor: high
Title: Detected custom leapp actors or files.
Summary: We have detected installed custom actors or files on the system. 
These can be provided e.g. by third party vendors, Red Hat consultants, 
or can be created by users to customize the upgrade (e.g. to migrate custom applications). 
This is allowed and appreciated. However Red Hat is not responsible for any issues caused by these custom leapp actors. 
Note that upgrade tooling is under agile development which could require more frequent update of custom actors.
The list of custom leapp actors and files:
    - /usr/share/leapp-repository/repositories/system_upgrade/common/files/rpm-gpg/8/RPM-GPG-KEY-Rocky-8
Related links:
    - Customizing your Red Hat Enterprise Linux in-place upgrade: https://red.ht/customize-rhel-upgrade
Remediation: [hint] In case of any issues connected to custom or third party actors, contact vendor of such actors. 
Also we suggest to ensure the installed custom leapp actors are up to date, compatible with the installed packages.
```

カスタムleappアクターを検出した、とのこと。RPM-GPG-KEY-Rocky-8がその対象になっているようです。
このファイルは、leappの手順どおりにGPG keyをインストールしたことによるものですので無視します。
想定されるファイルなので警告出さないでほしいところですが、とりあえずアクション不要とおもいます。

#### 4. Packages not signed by Red Hat found on the system

```txt:/var/log/leapp/leapp-report.txt(抜粋,適宜改行をいれて見やすくしています)
Risk Factor: high
Title: Packages not signed by Red Hat found on the system
Summary: The following packages have not been signed by Red Hat and may be removed 
during the upgrade process in case Red Hat-signed packages to be removed 
during the upgrade depend on them:
- AwsAgent
- ansible
- certbot
- epel-release
- python-ndg_httpsclient
- python-requests-toolbelt
- python-zope-component
- python-zope-event
- python2-acme
- python2-certbot
- python2-certbot-apache
- python2-configargparse
- python2-distro
- python2-future
- python2-httplib2
- python2-jmespath
- python2-josepy
- python2-mock
- python2-parsedatetime
- python2-pyrfc3339
- python2-six
```

これらのファイルはRedHatが署名したものではない、アップグレードによって削除される可能性がある、とのこと。
AwsAgentはAWSのエージェントですね。私の環境では意図的にいれたものではないので特に気にしないことにします。ansibleはRedmineをインストールする際に自分でいれたもの。certbotはLet's EncryptのSSL証明書更新ツール。epel-releaseはepelのリポジトリのファイルで自分で入れたものです。pythonやpython2で始まるものは、特に意識して入れたものではなくディストリビューションにもとから入っているものですが、前の項目でもpython3がデフォルトになっており、アップグレードで削除されても別にいいかとおもいます。
ということで、アップデート後にAnsible,certbot,epel-releaseが問題ないか確認することにします。

[アップグレード後の対応へ移動](#アップグレード後の対応)

#### 5. PostgreSQL (postgresql-server) has been detected on your system

```txt:/var/log/leapp/leapp-report.txt(抜粋,適宜改行をいれて見やすくしています)
Risk Factor: medium
Title: PostgreSQL (postgresql-server) has been detected on your system
Summary: PostgreSQL server component will be upgraded. 
Since RHEL-8 includes PostgreSQL server 10 by default, which is incompatible with 9.2 included in RHEL-7,
it is necessary to proceed with additional steps 
for the complete upgrade of the PostgreSQL data.
Related links:
    - Migrating to a RHEL 8 version of PostgreSQL: https://red.ht/rhel-8-migrate-postgresql-server
Remediation: [hint] Back up your data before proceeding with the upgrade and follow steps in the documentation section 
"Migrating to a RHEL 8 version of PostgreSQL" after the upgrade.
```

PostgreSQLが検出されたとのこと。リスクファクターは`Medium`ですが、RedmineサーバとしてはPostgreSQLが動かなくなるのは致命的なのでちゃんと対応しないといけない項目です。
移行のために必要になるとおもわれるので、きちんとDBのバックアップはとっておきましょう。（DBのバックアップ方法は[Redmine.JPの記事「データのバックアップ方法」](https://redmine.jp/faq/system_management/backup/)をご参照ください）

:::message alert
ここでとったDBのバックアップは後で使いますので、必ず取得してください
:::

[アップグレード後の対応へ移動](#アップグレード後の対応)

#### 6. chrony using default configuration

```txt:/var/log/leapp/leapp-report.txt(抜粋,適宜改行をいれて見やすくしています)
Risk Factor: medium
Title: chrony using default configuration
Summary: default chrony configuration in RHEL8 uses leapsectz directive, 
which cannot be used with leap smearing NTP servers, 
and uses a single pool directive instead of four server directives
```

RHELではchronyはデフォルトでleapsectzディレクティブを使う。この場合、leap smearing NTPサーバを使うことはできない。４つのserverディレクティブの代わりに1津のpoolディレクティブを使う、とのことです。
NTPサーバの設定まわりの記述だということがわかりますね。アップデート後にchronyの設定がどうなっているか、確認することとします。

[アップグレード後の対応へ移動](#アップグレード後の対応)

#### 7. Module pam_pkcs11 will be removed from PAM configuration

```txt:/var/log/leapp/leapp-report.txt(抜粋,適宜改行をいれて見やすくしています)
Risk Factor: medium
Title: Module pam_pkcs11 will be removed from PAM configuration
Summary: Module pam_pkcs11 was surpassed by SSSD and therefore it was removed from RHEL-8. 
Keeping it in PAM configuration may lock out the system thus it will be automatically 
removed from PAM configuration before upgrading to RHEL-8. 
Please switch to SSSD to recover the functionality of pam_pkcs11.
Remediation: [hint] Configure SSSD to replace pam_pkcs11
```

pam_pkcs11というのは、PCKS#11スマートカードを使用してユーザ名とパスワードを検証するモジュールとのことです。私の環境では使っていないので気にしませんが、使っている場合はpam_pkcs11の代わりにSSSDを使え、と書かれてますね。

### 事前準備まとめ

さて、ここまでで`leapp preupgrade`で表示された問題の対処とリスクを確認できました。私の環境ではこれだけでしたが、別の環境では違う問題やリスクが表示される可能性はあります。ただ、leapp-reportはとてもわかりやすく書かれているので、読めば何を対応するべきなのかだいたいわかる、という印象でした。
また、`leapp preupgrade`は何回でも実行できるので、いろいろ試して結果が変わるかどうかを確かめるのもよいとおもいます。

## アップグレード

`sudo leapp upgrade`が終わったら、再起動します。再起動は、環境にもよりますが、すこし時間がかかる場合があります。
（公式ドキュメントにも書いてあるとおりです）

### アップグレードの実行

:::message alert
ここから実際にアップグレードの実行です。最悪の場合ここまでもどれるように、実行前にスナップショットをとりました。
:::

```bash
# sudo leapp upgrade
# reboot
```

### アップグレードできたか確認

ちゃんとRocky Linuxにアップグレードできました。

```bash
$ cat /etc/redhat-release
Rocky Linux release 8.10 (Green Obsidian)
$ cat /etc/os-release
NAME="Rocky Linux"
VERSION="8.10 (Green Obsidian)"
ID="rocky"
ID_LIKE="rhel centos fedora"
VERSION_ID="8.10"
PLATFORM_ID="platform:el8"
PRETTY_NAME="Rocky Linux 8.10 (Green Obsidian)"
ANSI_COLOR="0;32"
LOGO="fedora-logo-icon"
CPE_NAME="cpe:/o:rocky:rocky:8:GA"
HOME_URL="https://rockylinux.org/"
BUG_REPORT_URL="https://bugs.rockylinux.org/"
SUPPORT_END="2029-05-31"
ROCKY_SUPPORT_PRODUCT="Rocky-Linux-8"
ROCKY_SUPPORT_PRODUCT_VERSION="8.10"
REDHAT_SUPPORT_PRODUCT="Rocky Linux"
REDHAT_SUPPORT_PRODUCT_VERSION="8.10"
$ rpm -qa | grep el7 # for 7 to 8 upgrade
leapp-0.18.0-1.el7.noarch
python2-parsedatetime-2.4-6.el7.noarch
python-zope-event-4.0.3-2.el7.noarch
python2-leapp-0.18.0-1.el7.noarch
python2-distro-1.5.0-1.el7.noarch
python2-httplib2-0.18.1-3.el7.noarch
kernel-3.10.0-1127.18.2.el7.x86_64
leapp-upgrade-el7toel8-0.21.0-2.el7.elevate.3.noarch
python2-jmespath-0.9.4-2.el7.noarch
python2-pyrfc3339-1.1-3.el7.noarch
kernel-3.10.0-1160.119.1.el7.x86_64
python2-future-0.18.2-2.el7.noarch
leapp-data-rocky-0.4-7.el7.20240827.noarch
python2-configargparse-0.11.0-2.el7.noarch
elevate-release-1.0-2.el7.noarch
```

## アップグレード後の対応

OSとしては無事アップグレードできていることがわかりましたが、もともと`preupgrade`のときに後で対応する必要がある、とチェックしておいた項目の対応を実行します。

### Pythonコマンドのシンボリックリンクを変更

```bash:OSアップグレード後(RockyLinux8.10)
# which python
/usr/bin/which: no python in (/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin)
# which python2
/usr/bin/python2
# which python3
/usr/bin/python3
```

アップグレード前は`python`コマンドは`python2`へのシンボリックリンクになっていましたが、そのシンボリックリンクはなくなっており、`/usr/bin/python2`と`/usr/bin/python3`は存在しています。leapp-report.txtで提案されていたとおり、alternativesコマンドを使用してpython3をデフォルトのpythonコマンドにします。

```bash:OSアップグレード後(RockyLinux8.10)
# alternatives --set python /usr/bin/python3
# which python
/usr/bin/python
# python --version
Python 3.6.8
```

### GRUB2の設定を確認

以下、GRUB2の設定を確認しました。
詳細はわからないというのが実際のところですが、デュアルブートしようとしているわけでもないですし、無事RockyLinuxが起動できているのでよしとします。

:::details 確認結果詳細

```txt:/boot/grub2/grub.cfg(OSアップグレード後(RockyLinux8.10))
#
# DO NOT EDIT THIS FILE
#
# It is automatically generated by grub2-mkconfig using templates
# from /etc/grub.d and settings from /etc/default/grub
#

### BEGIN /etc/grub.d/00_header ###
set pager=1

if [ -f ${config_directory}/grubenv ]; then
  load_env -f ${config_directory}/grubenv
elif [ -s $prefix/grubenv ]; then
  load_env
fi
if [ "${next_entry}" ] ; then
   set default="${next_entry}"
   set next_entry=
   save_env next_entry
   set boot_once=true
else
   set default="0"
fi

if [ x"${feature_menuentry_id}" = xy ]; then
  menuentry_id_option="--id"
else
  menuentry_id_option=""
fi

export menuentry_id_option

if [ "${prev_saved_entry}" ]; then
  set saved_entry="${prev_saved_entry}"
  save_env saved_entry
  set prev_saved_entry=
  save_env prev_saved_entry
  set boot_once=true
fi

function savedefault {
  if [ -z "${boot_once}" ]; then
    saved_entry="${chosen}"
    save_env saved_entry
  fi
}

function load_video {
  if [ x$feature_all_video_module = xy ]; then
    insmod all_video
  else
    insmod efi_gop
    insmod efi_uga
    insmod ieee1275_fb
    insmod vbe
    insmod vga
    insmod video_bochs
    insmod video_cirrus
  fi
}

if [ x$feature_default_font_path = xy ] ; then
   font=unicode
else
insmod part_msdos
insmod xfs
set root='hd0,msdos1'
if [ x$feature_platform_search_hint = xy ]; then
  search --no-floppy --fs-uuid --set=root --hint='hd0,msdos1'  2f64394d-a847-4c1a-bda9-e658edc017ad
else
  search --no-floppy --fs-uuid --set=root 2f64394d-a847-4c1a-bda9-e658edc017ad
fi
    font="/usr/share/grub/unicode.pf2"
fi

if loadfont $font ; then
  set gfxmode=auto
  load_video
  insmod gfxterm
fi
terminal_output gfxterm
if [ x$feature_timeout_style = xy ] ; then
  set timeout_style=menu
  set timeout=0
# Fallback normal timeout code in case the timeout_style feature is
# unavailable.
else
  set timeout=0
fi
### END /etc/grub.d/00_header ###

### BEGIN /etc/grub.d/00_tuned ###
set tuned_params=""
set tuned_initrd=""
### END /etc/grub.d/00_tuned ###

### BEGIN /etc/grub.d/01_users ###
if [ -f ${prefix}/user.cfg ]; then
  source ${prefix}/user.cfg
  if [ -n "${GRUB2_PASSWORD}" ]; then
    set superusers="root"
    export superusers
    password_pbkdf2 root ${GRUB2_PASSWORD}
  fi
fi
### END /etc/grub.d/01_users ###

### BEGIN /etc/grub.d/08_fallback_counting ###
insmod increment
# Check if boot_counter exists and boot_success=0 to activate this behaviour.
if [ -n "${boot_counter}" -a "${boot_success}" = "0" ]; then
  # if countdown has ended, choose to boot rollback deployment,
  # i.e. default=1 on OSTree-based systems.
  if  [ "${boot_counter}" = "0" -o "${boot_counter}" = "-1" ]; then
    set default=1
    set boot_counter=-1
  # otherwise decrement boot_counter
  else
    decrement boot_counter
  fi
  save_env boot_counter
fi
### END /etc/grub.d/08_fallback_counting ###

### BEGIN /etc/grub.d/10_linux ###
insmod part_msdos
insmod xfs
set root='hd0,msdos1'
if [ x$feature_platform_search_hint = xy ]; then
  search --no-floppy --fs-uuid --set=root --hint='hd0,msdos1'  2f64394d-a847-4c1a-bda9-e658edc017ad
else
  search --no-floppy --fs-uuid --set=root 2f64394d-a847-4c1a-bda9-e658edc017ad
fi
insmod part_msdos
insmod xfs
set boot='hd0,msdos1'
if [ x$feature_platform_search_hint = xy ]; then
  search --no-floppy --fs-uuid --set=boot --hint='hd0,msdos1'  2f64394d-a847-4c1a-bda9-e658edc017ad
else
  search --no-floppy --fs-uuid --set=boot 2f64394d-a847-4c1a-bda9-e658edc017ad
fi

# This section was generated by a script. Do not modify the generated file - all changes
# will be lost the next time file is regenerated. Instead edit the BootLoaderSpec files.
#
# The blscfg command parses the BootLoaderSpec files stored in /boot/loader/entries and
# populates the boot menu. Please refer to the Boot Loader Specification documentation
# for the files format: https://www.freedesktop.org/wiki/Specifications/BootLoaderSpec/.

# The kernelopts variable should be defined in the grubenv file. But to ensure that menu
# entries populated from BootLoaderSpec files that use this variable work correctly even
# without a grubenv file, define a fallback kernelopts variable if this has not been set.
#
# The kernelopts variable in the grubenv file can be modified using the grubby tool or by
# executing the grub2-mkconfig tool. For the latter, the values of the GRUB_CMDLINE_LINUX
# and GRUB_CMDLINE_LINUX_DEFAULT options from /etc/default/grub file are used to set both
# the kernelopts variable in the grubenv file and the fallback kernelopts variable.
if [ -z "${kernelopts}" ]; then
  set kernelopts="root=UUID=2f64394d-a847-4c1a-bda9-e658edc017ad ro  "
fi

insmod blscfg
blscfg
### END /etc/grub.d/10_linux ###

### BEGIN /etc/grub.d/10_reset_boot_success ###
# Hiding the menu is ok if last boot was ok or if this is a first boot attempt to boot the entry
if [ "${boot_success}" = "1" -o "${boot_indeterminate}" = "1" ]; then
  set menu_hide_ok=1
else
  set menu_hide_ok=0
fi
# Reset boot_indeterminate after a successful boot
if [ "${boot_success}" = "1" ] ; then
  set boot_indeterminate=0
# Avoid boot_indeterminate causing the menu to be hidden more then once
elif [ "${boot_indeterminate}" = "1" ]; then
  set boot_indeterminate=2
fi
# Reset boot_success for current boot
set boot_success=0
save_env boot_success boot_indeterminate
### END /etc/grub.d/10_reset_boot_success ###

### BEGIN /etc/grub.d/12_menu_auto_hide ###
if [ x$feature_timeout_style = xy ] ; then
  if [ "${menu_show_once}" ]; then
    unset menu_show_once
    save_env menu_show_once
    set timeout_style=menu
    set timeout=60
  elif [ "${menu_auto_hide}" -a "${menu_hide_ok}" = "1" ]; then
    set orig_timeout_style=${timeout_style}
    set orig_timeout=${timeout}
    if [ "${fastboot}" = "1" ]; then
      # timeout_style=menu + timeout=0 avoids the countdown code keypress check
      set timeout_style=menu
      set timeout=0
    else
      set timeout_style=hidden
      set timeout=1
    fi
  fi
fi
### END /etc/grub.d/12_menu_auto_hide ###

### BEGIN /etc/grub.d/20_linux_xen ###
### END /etc/grub.d/20_linux_xen ###

### BEGIN /etc/grub.d/20_ppc_terminfo ###
### END /etc/grub.d/20_ppc_terminfo ###

### BEGIN /etc/grub.d/30_os-prober ###
### END /etc/grub.d/30_os-prober ###

### BEGIN /etc/grub.d/30_uefi-firmware ###
### END /etc/grub.d/30_uefi-firmware ###

### BEGIN /etc/grub.d/40_custom ###
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.
### END /etc/grub.d/40_custom ###

### BEGIN /etc/grub.d/41_custom ###
if [ -f  ${config_directory}/custom.cfg ]; then
  source ${config_directory}/custom.cfg
elif [ -z "${config_directory}" -a -f  $prefix/custom.cfg ]; then
  source $prefix/custom.cfg;
fi
### END /etc/grub.d/41_custom ###
```

冒頭に、このファイルを編集するな、と書かれてますね。
正直、このファイルが何をやっているのか、細かいことはわかりません。
このファイルは`grub2-mkconfig`というコマンドで生成するようです。
このコマンドを実行した結果と、/boot/grub2/grub.cfgを比較してみました

```txt:/boot/grub2/grub.cfgとgrub2-mkconfigコマンドの出力の差分
# diff /boot/grub2/grub.cfg grub.cfg
79a80,82
>   set locale_dir=$prefix/locale
>   set lang=en_US
>   insmod gettext
```

差分はLANGに関することだけで、特に違いはないようですね。
なお、以下のとおり、GRUB2ではなくGRUBの設定ファイルをみると、CentOSのメニューが残ったままになっています。

```txt:/boot/grub/menu.lst(OSアップグレード後(RockyLinux8.10))
default=0
timeout=0


title CentOS Linux (3.10.0-1160.119.1.el7.x86_64) 7 (Core)
        root (hd0,0)
        kernel /boot/vmlinuz-3.10.0-1160.119.1.el7.x86_64 ro root=UUID=2f64394d-a847-4c1a-bda9-e658edc017ad console=hvc0 LANG=en_US.UTF-8
        initrd /boot/initramfs-3.10.0-1160.119.1.el7.x86_64.img
title CentOS Linux (3.10.0-1127.18.2.el7.x86_64) 7 (Core)
        root (hd0,0)
        kernel /boot/vmlinuz-3.10.0-1127.18.2.el7.x86_64 ro root=UUID=2f64394d-a847-4c1a-bda9-e658edc017ad console=hvc0 LANG=en_US.UTF-8
        initrd /boot/initramfs-3.10.0-1127.18.2.el7.x86_64.img
title CentOS Linux (3.10.0-1127.13.1.el7.x86_64) 7 (Core)
        root (hd0,0)
        kernel /boot/vmlinuz-3.10.0-1127.13.1.el7.x86_64 ro root=UUID=2f64394d-a847-4c1a-bda9-e658edc017ad console=hvc0 LANG=en_US.UTF-8
        initrd /boot/initramfs-3.10.0-1127.13.1.el7.x86_64.img
title CentOS Linux (3.10.0-1127.10.1.el7.x86_64) 7 (Core)
        root (hd0,0)
        kernel /boot/vmlinuz-3.10.0-1127.10.1.el7.x86_64 ro root=UUID=2f64394d-a847-4c1a-bda9-e658edc017ad console=hvc0 LANG=en_US.UTF-8
        initrd /boot/initramfs-3.10.0-1127.10.1.el7.x86_64.img
title CentOS Linux (3.10.0-1062.12.1.el7.x86_64) 7 (Core)
        root (hd0,0)
        kernel /boot/vmlinuz-3.10.0-1062.12.1.el7.x86_64 ro root=UUID=2f64394d-a847-4c1a-bda9-e658edc017ad console=hvc0 LANG=en_US.UTF-8
        initrd /boot/initramfs-3.10.0-1062.12.1.el7.x86_64.img
```

パッケージは以下のとおりGRUB2のほうがインストールされており、GRUBの設定を見に行くわけではないとおもいます。

```bash
# rpm -qa | grep grub
grub2-common-2.02-158.el8_10.rocky.0.1.noarch
grub2-tools-2.02-158.el8_10.rocky.0.1.x86_64
grub2-tools-efi-2.02-158.el8_10.rocky.0.1.x86_64
grub2-tools-minimal-2.02-158.el8_10.rocky.0.1.x86_64
grubby-8.40-49.el8.x86_64
grub2-tools-extra-2.02-158.el8_10.rocky.0.1.x86_64
grub2-pc-2.02-158.el8_10.rocky.0.1.x86_64
grub2-pc-modules-2.02-158.el8_10.rocky.0.1.noarch
```

:::

### Ansible,certbot,epel-releaseが問題ないか確認

確認した結果、ansible、certbotはインストールされていない状態になっていますね。必要に応じてインストールする必要がありそうです。epelはインストールされており、バージョンが上がっています。

:::details 確認結果詳細

```bash:OSアップグレード前(CentOS7.9)
# rpm -qa | grep ansible
ansible-2.9.27-1.el7.noarch
# rpm -qa | grep certbot
python2-certbot-1.11.0-2.el7.noarch
certbot-1.11.0-2.el7.noarch
python2-certbot-apache-1.11.0-1.el7.noarch
# rpm -qa | grep epel
epel-release-7-14.noarch
```

```bash:OSアップグレード後(RockyLinux8.10)
# rpm -qa | grep ansible
# rpm -qa | grep certbot
# rpm -qa | grep epel
epel-release-8-18.el8.noarch
```

:::

### postgreSQLの確認

確認した結果、CentOSの環境ではpostgreSQL9.2がインストールされていたところ、RockyLinuxでは10.23になっています。これによりプロセスは起動しようとしてもエラーがでて起動できない状態です。これについての対処は後編で実施することにします。

:::details postgreSQLの確認詳細

```bash:OSアップグレード後(RockyLinux8.10)
# rpm -qa | grep postgres
postgresql-10.23-4.module+el8.9.0+1734+74bd286c.x86_64
postgresql-server-10.23-4.module+el8.9.0+1734+74bd286c.x86_64
# systemctl status postgresql
● postgresql.service - PostgreSQL database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Sun 2024-11-10 23:02:52 JST; 2h 18min ago
　　・・・
# systemctl start postgresql
Job for postgresql.service failed because the control process exited with error code.
See "systemctl status postgresql.service" and "journalctl -xe" for details.
```

```txt:(参考)OSアップグレード前
# rpm -qa | grep postgres
postgresql-server-9.2.24-9.el7_9.x86_64
postgresql-libs-9.2.24-9.el7_9.x86_64
postgresql-9.2.24-9.el7_9.x86_64
postgresql-devel-9.2.24-9.el7_9.x86_64
```

:::

### Chronyの設定を確認

確認した結果、CentOSのときとは異なるpoolディレクティブを使用するようになっており、きちんとNTPサーバとの接続も確認できるので問題ないようです。

:::details 確認結果詳細

```txt:OSアップグレード後の/etc/chrony.conf冒頭部分
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
pool 2.rocky.pool.ntp.org iburst
```

```bash（NTPサーバとの接続状態を確認）
# chronyc sources
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^- ap-northeast-1.clearnet.>     2  10   377   753   +395us[ +395us] +/-   23ms
^* mail1.marinecat.net           2  10   377   803   +218us[ +264us] +/- 2374us
^- time.cloudflare.com           3  10   377   756   +745us[ +745us] +/-   60ms
^- time.cloudflare.com           3  10   377   758  +1182us[+1182us] +/-   61ms
```

```txt:(参考)OSアップグレード前の/etc/chrony.conf冒頭部分
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst
```

:::

### （おまけ）httpdの確認

preupgradeの時点では特にリスクとして表示されていませんでしたが、Redmineが動作するためにはhttpdが動作している必要があります。確認の結果、こちらは問題なく動作しているようです。

:::details 確認結果詳細

```bash
# systemctl status httpd
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2024-11-10 23:02:52 JST; 2h 46min ago
     Docs: man:httpd.service(8)
 Main PID: 889 (httpd)
   Status: "Total requests: 11; Idle/Busy workers 100/0;Requests/sec: 0.0011; Bytes served/sec:   2 B/sec"
    Tasks: 278 (limit: 12382)
   Memory: 47.9M
   CGroup: /system.slice/httpd.service
           tq 889 /usr/sbin/httpd -DFOREGROUND
           tq 914 /usr/sbin/httpd -DFOREGROUND
           tq 920 /usr/sbin/httpd -DFOREGROUND
           tq 921 /usr/sbin/httpd -DFOREGROUND
           tq 922 /usr/sbin/httpd -DFOREGROUND
           mq2242 /usr/sbin/httpd -DFOREGROUND

Nov 10 23:02:52 ip-172-31-44-172.ap-northeast-1.compute.internal systemd[1]: Starting The Apache HTTP Server...
Nov 10 23:02:52 ip-172-31-44-172.ap-northeast-1.compute.internal systemd[1]: Started The Apache HTTP Server.
Nov 10 23:02:52 ip-172-31-44-172.ap-northeast-1.compute.internal httpd[889]: Server configured, listening on: port 443, port 80
```

:::

## まとめ

長くなってしまったのでこの記事はここまでとします。結果、Redmineはまだ動いていない状態です。復旧させるために何をしたか、別記事でまとめますのですこしお待ちください。

* AlmaLinuxにより提供されたElevateツールにより、RedmineサーバのCentOS7.9をRockyLinux8.10に無事アップグレードすることができた。
* PostgreSQLのバージョンが上がってしまっており、プロセスが起動できていない。よって、Redmineも当然動かない状態になっている。
* ansibleやcertbotなど、個別にインストールしていたツール類はアンインストールされるようだ。
* httpdは問題なく動作しているようだ。

## 参考

* [ELevate公式サイト(AlmaLinux)](https://almalinux.org/elevate/)
