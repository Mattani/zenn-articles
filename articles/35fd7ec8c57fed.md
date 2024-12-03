---
title: "Elevateを使用してRedmineサーバをCentOS7からRockyLinuxにアップデートしてみた（後編）"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Redmine","CentOS", "RockyLinux", "ELevate"]
published: true
---

## はじめに

この記事は[前編](https://zenn.dev/mattani/articles/8cfe87aee8a8cd)の続きです。前編では、AlmaLinuxから公開されているElevateというツールを使ってRedmineサーバのCentOSをRockyLinuxにアップデートするところまで実施しました。しかし、ElevateでOSはアップグレードできたものの、すぐにRedmineが動く状態ではありませんでした。（詳細は前編参照）その後、問題を順にクリアしてRedmineサーバとして動作させるところまで、この記事で書きます。

（前提条件）

* OSアップデート前はCentOS7.9でしたが、ElevateによりCentOSがRockyLinuxにアップデート済み
* rootアカウントでログインできている
以下、プロンプトの#はrootアカウント、$はpostgresアカウントです。

## 現状の整理

* postgreSQLのバージョンが9から10に上がっており、DBを起動しようとしても失敗する
* Apache httpdは起動しているがRedmineは動いていない
* Ansible, certbotはアンインストールされた状態になっている
* EPELリポジトリは参照可能な状態になっている

## 復旧作業
### アクション(1)　postgreSQL DBのリカバリ

#### 古いDBのディレクトリを退避

postgreSQLユーザに`su`し、古いDBのディレクトリを退避します。

```bash
# su - postgres
$ ls -l
$ ls
backups  data  initdb.log
$ mv data data.bak
$ ls
backups  data.bak  initdb.log
$ exit
#
```

#### DBを初期化

DBを初期化します。これで新たな`var/lib/pgsql/data`ができます

```bash
# /usr/bin/postgresql-setup --initdb --unit postgresql
 * Initializing database in '/var/lib/pgsql/data'
 * Initialized, logs are in /var/lib/pgsql/initdb_postgresql.log
```

#### DBサーバを起動

systemctlでpostgresサービスを起動します

```bash
# systemctl start postgresql
# systemctl status postgresql
● postgresql.service - PostgreSQL database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql.service; enabled; vendor >
   Active: active (running) since Tue 2024-11-12 02:09:21 JST; 4min 32s ago
　　・・・
```

#### Redmine用のDBユーザー、DBを作成

DBを初期化し、DBユーザーもDBもない状態になっています。以下のコマンドで、Redmine用のDBユーザーを作成します。このとき入力するパスワードは、 `/var/lib/redmine/config/database.yml`に書いてあるものにします。また、redmine用のDBも以下のようにして作成します。

```bash
# sudo -u postgres createuser -P redmine
Enter password for new role:
Enter it again:
# sudo -u postgres createdb -E UTF-8 -l ja_JP.UTF-8 -O redmine -T template0 redmine
#
```

#### DBをリストア

[前編](https://zenn.dev/mattani/articles/8cfe87aee8a8cd#5.-postgresql-(postgresql-server)-has-been-detected-on-your-system)のときに、DBダンプを取得していますので、それをここでリストアします。リストア後、postgresユーザからrootユーザに戻ります。この状態で、`systemctl status postgresql`で確認すると、postgresqlはちゃんと動作しているようです。

```bash
# su - postgres
$ psql -d redmine < dumpfile.sql
　（出力は割愛）
$ exit
# systemctl status postgresql
● postgresql.service - PostgreSQL database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2024-11-13 01:41:35 JST; 10min ago
  Process: 10421 ExecStartPre=/usr/libexec/postgresql-check-db-dir postgresql (code=exited, status=0/SUCCE>
 Main PID: 10423 (postmaster)
    Tasks: 8 (limit: 12382)
   Memory: 40.2M
```

#### DBマイグレーションしてみる

DBマイグレーションしてみると、Rake abort!になりますね。エラー内容をみると、rubyからopenssl.soが見えていないようです。次項目で、Rubyの環境をリカバリしていきます。

```bash
# cd /var/lib/redmine
# RAILS_ENV=production bundle exec rake db:migrate
rake aborted!
LoadError: libssl.so.10: cannot open shared object file: No such file or directory - /usr/local/lib/ruby/2.6.0/x86_64-linux/openssl.so
・・・
```

### アクション(2)　Rubyの環境をリカバリ

#### rbenv環境を作成

私の場合、最近は初期構築のときからrbenvを入れるのですが、CentOS時代はrbenvを入れていませんでした。Rubyの環境をつくるのはrbenvを使うのが簡単なので、まずはrbenvの環境を作っていきます。

```bash
# git clone https://github.com/rbenv/rbenv.git /usr/local/rbenv
Cloning into '/usr/local/rbenv'...
remote: Enumerating objects: 3334, done.
remote: Counting objects: 100% (484/484), done.
remote: Compressing objects: 100% (261/261), done.
remote: Total 3334 (delta 277), reused 372 (delta 207), pack-reused 2850 (from 1)
Receiving objects: 100% (3334/3334), 683.04 KiB | 17.97 MiB/s, done.
Resolving deltas: 100% (2067/2067), done.
# vi ~/.bashrc
```

```txt:.bashrcに追加する内容
# rbenv initialization
export RBENV_ROOT="/usr/local/rbenv"
export PATH="$RBENV_ROOT/bin:$PATH"
eval "$(rbenv init -)"
```

.bashrcを読み込みます（ログインしなおしてもOK）

```bash
# . ~/.bashrc
```

Ruby-buildプラグインをインストールします

```bash
# git clone https://github.com/rbenv/ruby-build.git "$(rbenv root)"/plugins/ruby-build
Cloning into '/usr/local/rbenv/plugins/ruby-build'...
remote: Enumerating objects: 16564, done.
remote: Counting objects: 100% (4833/4833), done.
remote: Compressing objects: 100% (399/399), done.
remote: Total 16564 (delta 4615), reused 4558 (delta 4422), pack-reused 11731 (from 1)
Receiving objects: 100% (16564/16564), 3.19 MiB | 3.26 MiB/s, done.
Resolving deltas: 100% (11778/11778), done.
```

rbenv環境ができたか確認します。無事rbenvがインストールできているようです。

```bash
# curl -fsSL https://github.com/rbenv/rbenv-installer/raw/HEAD/bin/rbenv-doctor | bash
```

![rbenv-doctorの実行結果](/images/rbenv-doctor-result.png)

rbenvを使ってRuby 2.6.5をインストールします。インストールが終わったら、`rbenv global 2.6.5`で使用するRubyのバージョンをグローバル設定しておきます。
:::message
Ruby2.6.5は表示されているとおりEOLになっているので、本来なら新しいバージョンのRubyをインストールすべきですが、この記事ではElevateによるOSアップグレードがテーマなので、それ以外はもともとの環境と同じようにする選択をしています
:::

![rbenv-install-2.6.5](/images/rbenv-install-2.6.5.png)

ここで、`rbenv rehash`を実行し、この状態でRubyがOpensslを認識しているか確認してみます。ちゃんとOpenSSL 1.1.1kを認識してますね。

```bash
# rbenv rehash
# ruby -r openssl -e 'p OpenSSL::OPENSSL_LIBRARY_VERSION'
"OpenSSL 1.1.1k  FIPS 25 Mar 2021"
```

再度DBマイグレーションを実施してみるとこんどはRake abort!にはなりませんでしたが、多くのgemでextensions are not builtというエラーになります。一番最後にでているとおり、`bundle install`が必要なようです
![alt text](/images/db-migrate-result.png)

### アクション(3)　bundle install（gemの入れ直し）

Rubyを入れなおしているので、過去にいれたgemは消して、改めて0から`bundle install`します。参考として`bundle config`の情報も載せておきます。

```bash
# cd /var/lib/redmine
# bundle config
Settings are listed in order of priority. The top value will be used.
path
Set for your local app (/var/lib/redmine/.bundle/config): "vendor/bundle"

without
Set for your local app (/var/lib/redmine/.bundle/config): [:development, :test]
# rm -rf Gemfile.lock vendor/bundle
# bundle install
（出力は割愛）
```

出力は割愛しますが、`bundle install`は成功し、無事gemがインストールできたようですので、再度マイグレーションを実行してみます。

```bash
# RAILS_ENV=production bundle exec rake db:migrate
rake aborted!
NameError: uninitialized constant Nokogiri::HTML4
・・・
```

### アクション(4)　マイグレーションのエラーを修復

マイグレーションで出たエラーを修復していきます。このエラーは見覚えありますね。以前に[Qiitaに記事](https://qiita.com/Mattani/items/c4bcc95a72b69d1c0489) 書いたものです。
上記記事のとおり、`Gemfile.local`を作成し、`bundle update`して再度マイグレーション実行してみます。

```txt:/var/lib/redmine/Gemfile.local
gem 'loofah', '~> 2.20.0'
```

```bash
# bundle update
（出力は割愛）
# RAILS_ENV=production bundle exec rake db:migrate
svn: E155036: Please see the 'svn upgrade' command
svn: E155036: The working copy at '/var/lib/redmine'
is too old (format 29) to work with client version '1.10.2 (r1835932)' (expects format 31). You need to upgrade the working copy first.

rake aborted!
```

こんどは別のエラーでました。なにやらsvnのフォーマットが古いと言っているようです。エラーメッセージに従い、svnのフォーマットをアップグレードしてみます。

```bash
# svn upgrade /var/lib/redmine
Upgraded '.'
# RAILS_ENV=production bundle exec rake db:migrate
rake aborted!
LoadError: cannot load such file -- blankslate
・・・
```

また別のエラーでました。またまた見覚えあるエラーですね。これは2024年6月にRedmine.tokyoで発表したトラブル事例です。[資料はこちら](https://speakerdeck.com/mattani/hmatsutani)
対策は`Gemfile.local`に１行追記することです。

```txt:/var/lib/redmine/Gemfile.local
gem 'loofah', '~> 2.20.0'
gem 'builder', '~> 3.2.4'
```

```bash
# bundle update
（出力は割愛）
# RAILS_ENV=production bundle exec rake db:migrate
rake aborted!
PG::ConnectionBad: FATAL:  Ident authentication failed for user "redmine"
・・・
```

また別のエラーでました。DBでuser "redmine"の認証が失敗しているようです。どうもpostgresqlがアップデートされたときに、`pg_hba.conf`に設定していた内容が失われているようです。postgresユーザになり、`data/pg_hba.conf`に追記します

```txt:/var/lib/pgsql/data/pg_hba.conf
# If you want to allow non-local connections, you need to add more
# "host" records.  In that case you will also need to make PostgreSQL
# listen on a non-local interface via the listen_addresses
# configuration parameter, or via the -i or -h command line switches.
# (以下の２行を追記します)
host    redmine         redmine         127.0.0.1/32            md5
host    redmine         redmine         ::1/128                 md5
```

これで、マイグレーションを実行してもエラーは発生しなくなりました。
（実際のマイグレーションによるDB変更はなにも起こりませんでしたが笑）
```bash
# RAILS_ENV=production bundle exec rake db:migrate
#
```

ここまで実行し、Redmineが動いているか確認してみると、Forbiddenのエラーが出ました。RailsアプリケーションとしてのRedmineが認識されていないようです。

![Forbidden](/images/forbidden.png)

### アクション(5)　Passengerのインストール

そういえば上記でRubyの環境をインストールしなおしているので、PassengerのインストールとApache用モジュールのビルドも当然必要になりますね。以下のとおり実行していきます。Apache用モジュールのビルドが終わったらApache用設定内容を確認してApacheの設定を変更します。

```bash
# gem install passenger -N
Fetching rack-3.1.8.gem
Fetching rackup-2.2.1.gem
Fetching rake-13.2.1.gem
Fetching passenger-6.0.23.gem
Successfully installed rack-3.1.8
Successfully installed rackup-2.2.1
Successfully installed rake-13.2.1
Building native extensions. This could take a while...
Successfully installed passenger-6.0.23
```

:::message
gem install のオプション`--no-rdoc --no-ri`で紹介されている記事が多いですが、これらのオプション指定は古いです。現在は`-N`を使うのがよいようです。
参考：[gem installで-no-riや-no-rdocを-Nにすべき理由を解説!](https://freelance.shiftinc.jp/column/ruby--no-ri)
:::

```bash
# passenger-install-apache2-module --auto --languages ruby
（出力は割愛）
# passenger-install-apache2-module --snippet
LoadModule passenger_module /usr/local/rbenv/versions/2.6.5/lib/ruby/gems/2.6.0/gems/passenger-6.0.23/buildout/apache2/mod_passenger.so
<IfModule mod_passenger.c>
  PassengerRoot /usr/local/rbenv/versions/2.6.5/lib/ruby/gems/2.6.0/gems/passenger-6.0.23
  PassengerDefaultRuby /usr/local/rbenv/versions/2.6.5/bin/ruby
</IfModule>
```

`passenger-install-apache2-module --snippet`の結果を`/etc/httpd/conf.d/redmine.conf`に書き込みます。書き込み終わったらhttpdを再起動。

```bash
# systemctl restart httpd
```

これでブラウザからアクセスし、ようやくOSをRocky Linuxにした環境で、Redmineが無事復旧できたことを確認しました。

## まとめ

ここまでの作業でようやくCentOS上で動作していたRedmineのOSを更改し、Rocky Linux上でRedmineが動作させることができました。

（復旧作業のまとめ）

* アクション(1)　postgreSQL　DBのリカバリ
* アクション(2)　Rubyの環境をリカバリ
* アクション(3)　bundle install（gemの入れ直し）
* アクション(4)　マイグレーションのエラーを修復
* アクション(5)　Passengerのインストール

改めて復旧作業を見返してみると・・・結局新しいサーバにRedmineをゼロからインストールしなおしているのとたいして変わらない作業になっています😅
遭遇した問題については、他のサーバ構築などで経験があれば対処がすぐにわかるものもあるとおもいますが、慣れていなければひとつひとつ対処を調査する必要があり、なかなか手間がかかります。また環境によっては違う問題が発生する可能性もあります。

ElevateはCentOSからほかのRHEL8系のOSにアップグレードできる大変すばらしいツールです。OSのアップグレード自体は非常に簡単でした。ただ、実際にやってみると、アプリケーションを同じように動く状態にするにはそれなりに大変なリカバリ作業が必要なことがわかりました。

新しいVM上で構築するのであれば、[Ansibleスクリプト](https://github.com/Mattani/redmine-rocky-ansible)で楽に構築できることも考えれば、頑張って１つのVM上でOSをアップグレードするよりも、別のVM上で新規構築したほうがよっぽど早いと感じました。

この経験が、これからElevateでアップグレード作業する方や、Elevateでアップグレードするか別VMで構築するか迷っている方の参考になれば幸いです。
