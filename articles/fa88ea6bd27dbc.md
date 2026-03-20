---
title: "RedmineでSSOログイン実現するプラグインの比較"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Redmine, プラグイン, SSO]
published: true
---

## はじめに

RedmineをSSO認証して使いたいというニーズは古くからありました。

Redmineは従来からLDAP/Active Directory連携でユーザ管理を一元化でき、いわばSSOの原型となる仕組みを備えています。企業システムではその後SAMLが普及し、近年はOAuth2/OIDCといったよりモダンな認証方式への移行が進んでいます。

なお、[Redmine 6.1でOAuth2対応](https://blog.redmine.jp/articles/updates/2025/06/)が追加されましたが、これはAPIプロバイダ向けの機能であり、ユーザのログイン（認証）に使えるものではありません。

本記事は、RedmineでSSOを実現するために、SAMLプラグインおよびOAuth2/OIDCのプラグインについて比較してみたものです。

注：記事中の最終コミット日時や、未対応等の情報は、記事執筆時点での情報です。今後のバージョンで追加される可能性はあります。また導入の容易さについては、私の主観で書いております。ご了承ください。

## SAMLプラグインについて

以下は主要な SAML 関連プラグインの比較です。一般論として、SAML は IdP とのメタデータ交換(XML)、証明書管理、SLO（シングルログアウト）やAttributeのマッピング等、設定項目が多く、プロバイダと協調した検証が必要になるため導入のハードルは高めです。

| プラグイン | OSS/商用 | 現在の更新状況 | Redmine 5 以降対応 | 導入の容易さ |
| --- | --- | --- | --- | --- |
| [AlphaNodes/redmine_saml](https://github.com/AlphaNodes/redmine_saml) | OSS(GPL v2) | アーカイブ済み（read-only、アーカイブ日: 2025-10-09） — 以降はメンテされていない可能性大 | 要検証（README に一部 Redmine 6 向けの記述があるが、アーカイブのため利用は注意） | 難 |
| [chrodriguez/redmine_omniauth_saml](https://github.com/chrodriguez/redmine_omniauth_saml) → [davintoo/redmine_omniauth_saml](https://github.com/davintoo/redmine_omniauth_saml) | OSS(GPL v2) | chrodriguezはアーカイブ済み（2021-05-17）。davintooは最終コミット 2022-12-21 | フォーク上で Redmine 5 対応の痕跡あるが、Redmine 6は2024/11リリースであり、対応していない可能性大 | 中〜難 |
| [Lychee SAML認証](https://lychee-redmine.jp/plugin/saml) | 商用 | 現在も提供されている | 対応可能と考えられる（詳細はベンダに問い合わせ要） | 中（サポート費用が発生） |

## OAuth2/OIDCプラグインについて

一般論として、OAuth2は“認可フレームワーク”であり、認証用途には本来向きません。OIDC は OAuth2 を拡張して“認証”を標準化したもので、ID Token（JWT）でユーザーの身元が保証されます。ともにJSONでの実現でありXMLを使用するSAMLよりはシンプルであるといえます。

| プラグイン | OSS/商用 | 現在の更新状況 | Redmine 5 以降対応 | 導入の容易さ |
| --- | --- | --- | --- | --- |
| [planio-gmbh/omniauth-redmine-oauth2](https://github.com/planio-gmbh/omniauth-redmine-oauth2) | OSS(MIT) | 最終コミット: 2020-08-25（活動は停滞） | （古いため未対応の可能性大） | 難～中 |
| [Contargo/redmine_oidc](https://github.com/Contargo/redmine_oidc) →[jirutka/redmine_oidc](https://github.com/jirutka/redmine_oidc) | OSS(GPL v2) | Contargo 最終コミット: 2024-05-14 / jirutka 最終コミット: 2025-03-24 | Redmine 5は対応。jirutka フォークは Redmine 6 対応をうたっている | 中～容易 |
| [enricohuang/redmine_oidc](https://github.com/enricohuang/redmine_oidc) | OSS(MIT) | 最終コミット: 2026-02-16（新規リリース） | README に Redmine 5.0+/6.0+ 対応明記 | 中〜容易 |
| [kontron/redmine_oauth](https://github.com/kontron/redmine_oauth) | OSS(GPL v3) | 最終コミット: 2026-03-06（v4.0.4 リリース） | v3.0.1でRedmine 6.1対応の記述あり | 中（OAuth2/OIDC両対応・機能多のため比較的難かも） |

### 機能比較（OAuth2/OIDC）

| プラグイン | 認証方式 | Discovery / JWKS | PKCE | ユーザ同期 / 自動作成 | 備考 |
| --- | --- | --- | --- | --- | --- |
| [planio-gmbh/omniauth-redmine-oauth2](https://github.com/planio-gmbh/omniauth-redmine-oauth2) | OAuth2 (OmniAuth strategy) | ー | ー | 手動（自動作成の記載なし） | OAuth2クライアント/古い。リポジトリは活動停滞 |
| [Contargo/redmine_oidc](https://github.com/Contargo/redmine_oidc) →[jirutka/redmine_oidc](https://github.com/jirutka/redmine_oidc) | OIDC | 対応 | 未対応 | 自動登録・メール一致のオプションあり（マッピング機能あり） | Keycloak 想定の設定例あり、openid_connect ライブラリ利用 |
| [enricohuang/redmine_oidc](https://github.com/enricohuang/redmine_oidc) | OIDC | 対応 | 未対応 | 自動登録・自己リンク・メール一致など豊富なユーザ管理機能あり | Self-service linking、MIT ライセンス |
| [kontron/redmine_oauth](https://github.com/kontron/redmine_oauth) | OAuth2 / OIDC 両対応 | 対応(OIDCのみ) | 対応済み（3.0.2（2025-01-09）以降） | Redmine の Self-registration に依存する挙動（設定次第で登録/リンク） | 多プロバイダ対応、ドキュメント豊富 |

## まとめ

本記事で各プラグインの方式や実現している機能を調査・整理した結果、2026年現在、RedmineでSSOするのであれば、 `OIDC` に対応したプラグインを選ぶのが現実的だと思います。

- SAMLプラグイン: XML ベースで IdP 側とのメタデータ交換や証明書管理が必要になり、プロバイダと協調した検証が不可欠であるため導入ハードルが高い。運用面でも設定ミスで挙動が複雑になりやすい。
- OAuth2プラグイン: 本来は「認可フレームワーク」であり、認証（SSO）用途にそのまま流用すると挙動やセキュリティ設計で工夫や追加実装が必要になりがちで扱いが難しい。
- OIDCプラグイン: OAuth2 を認証用途に拡張した仕様で、Discovery/JWKS 等によりプロバイダ連携がシンプルに行えるため、短時間で試せる（実装・運用ともに比較的容易）。

あとは、プラグインを導入するにあたって、活発に更新されていることも重要なため `enricohuang/redmine_oidc` または`jirutka/redmine_oidc`がまず試してみるプラグインの有力候補となります。`Contargo/redmine_oidc`は、Redmine 6 に対応していないようですが`jirutka/redmine_oidc`フォークでRedmine6対応しているため、こちらが実質的な後継とみてよさそうです。
`kontron/redmine_oauth`は、機能としては現時点では一番充実しているように見えます。PKCEに対応しているものも、現時点ではこれだけです。設定が複雑になる可能性はありますが、用途によってはこちらのほうがよいかもしれません。

## 気づき・今後について

本記事執筆のうえで、私自身、SAML、OAuth2、OIDCや各プラグインの実装方式について調査してかなり理解が進みました。
本記事は、あくまで机上での事前調査によるもので、実際に使ってみたわけではありません。実際にやってみたらまたご紹介しようと思います。

これからRedmineのSSO認証を検討する方の一助になれば幸いです。

## 用語

以下は本記事で使用する主要な用語の簡潔な説明です。

- **SSO (Single Sign-On)**: 一度の認証で複数のサービスやアプリケーションにアクセスできる仕組み。ユーザーの利便性向上と認証管理の一元化が主な目的で、IdP（Identity Provider）を介して各サービスの認証を委任する形で実現される。
- **SAML (Security Assertion Markup Language)**: XML ベースの認証／認可プロトコル。IdP（Identity Provider）と SP（Service Provider）間でアサーション（認証情報）やメタデータ、証明書を交換して SSO を実現する。メタデータや証明書管理が必要で、設定・検証の手間がかかる。
- **OAuth2**: リソースへのアクセスを第三者に委任するための「認可フレームワーク」。認証そのものを定義しないため、単体で SSO の認証層として使うには注意が必要（設計上の補完や追加実装が必要）。
- **OIDC (OpenID Connect)**: OAuth2 の上に乗る「認証のための仕様」。Authorization Code + Discovery + ID Token（JWT）を用い、プロバイダ連携が比較的簡単にできるため、近年の SSO 実装で採用されやすい。
- **Discovery**: OIDC の `/.well-known/openid-configuration` でプロバイダのエンドポイント（認可・トークン・userinfo など）を自動取得する仕組み。
- **JWKS**: JSON Web Key Set。プロバイダが公開する公開鍵の集合で、ID Token（JWT）の署名検証に使う。
- **PKCE**: Proof Key for Code Exchange。認可コードフローで認可コードの乗っ取りを防ぐための拡張（クライアント側でコードチャレンジを生成して検証する）。
- **SLO（Single Logout）**: アプリケーションでログアウトするとIdPでもセッション終了、IdPでログアウトするとアプリケーションもログアウトする、というように、IdP とアプリケーション間でログアウト状態を同期・連携させる。
- **ID Token**: OIDC が発行する JWT 形式のトークンで、ユーザーの認証情報（sub, email 等）を含む。
- **JWT (JSON Web Token)**: コンパクトな URL セーフな JSON ベースのトークン形式。ヘッダ・ペイロード・署名の3部構成で、署名により改ざん検知が可能。ID Token やアクセストークンで広く使われる。
