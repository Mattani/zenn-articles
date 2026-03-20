---
title: "RedmineでSSOログイン実現するプラグインの比較"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Redmine, プラグイン, SSO]
published: false
---

## はじめに

RedmineをSSO認証して使いたいというニーズは古くからありました。

Redmineは従来からLDAP/Active Directory連携でユーザ管理を一元化でき、いわばSSOの原型となる仕組みを備えています。企業システムではその後SAMLが普及し、近年はOAuth2/OIDCといったよりモダンな認証方式への移行が進んでいます。

なお、[Redmine 6.1でOAuth2対応](https://blog.redmine.jp/articles/updates/2025/06/)が追加されましたが、これはAPIプロバイダ向けの機能であり、ユーザのログイン（認証）に使えるものではありません。

本記事は、RedmineでSSOを実現するために、SAMLプラグインおよびOAuth2/OIDCのプラグインについて比較してみたものです。

注：記事中の最終コミット日時や、未対応等の情報は、記事執筆時点での情報です。今後のバージョンで追加される可能性はあります。また導入の容易さについては、私の主観で書いております。ご了承ください。

### SAMLプラグインについて

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
| [planio-gmbh/omniauth-redmine-oauth2](https://github.com/planio-gmbh/omniauth-redmine-oauth2) | OSS(MIT) | 最終コミット: 2020-08-25（活動は停滞） | 要検証（古いため未対応の可能性大） | 難（開発活性が低いため導入ハードル高い） |
| [Contargo/redmine_oidc](https://github.com/Contargo/redmine_oidc) | OSS(GPL v2) | 最終コミット: 2024-05-14（Redmine 5 対応テストあり） | Redmine 5 対応の痕跡あり（要検証） | 中（設定項目は多めだがドキュメント充実） |
| [enricohuang/redmine_oidc](https://github.com/enricohuang/redmine_oidc) | OSS(MIT) | 最終コミット: 2026-02-16（新規リリース・Redmine 5/6 対応表記あり） | README に Redmine 5.0+/6.0+ 対応明記 | 中〜容易（最近のリリースで互換性が明示されている） |
| [kontron/redmine_oauth](https://github.com/kontron/redmine_oauth) | OSS(GPL v3) | 最終コミット: 2026-03-06（v4.0.4 リリース） | 要検証（READMEに豊富な導入手順あり） | 中〜容易（活発でドキュメント充実） |

### 機能比較（OAuth2/OIDC）

| プラグイン | 認証方式 | Discovery / JWKS | PKCE | ユーザ同期 / 自動作成 | 備考 |
| --- | --- | --- | ---: | --- | --- |
| [planio-gmbh/omniauth-redmine-oauth2](https://github.com/planio-gmbh/omniauth-redmine-oauth2) | OAuth2 (OmniAuth strategy) | ー | ー | 手動（自動作成の記載なし） | OAuth2クライアント/古い。リポジトリは活動停滞 |
| [Contargo/redmine_oidc](https://github.com/Contargo/redmine_oidc) | OIDC | 対応 | 未対応 | 自動登録・メール一致のオプションあり（マッピング機能あり） | Keycloak 想定の設定例あり、openid_connect ライブラリ利用 |
| [enricohuang/redmine_oidc](https://github.com/enricohuang/redmine_oidc) | OIDC | 対応 | 未記載 | 自動登録・自己リンク・メール一致など豊富なユーザ管理機能あり | Self-service linking、MIT ライセンス |
| [kontron/redmine_oauth](https://github.com/kontron/redmine_oauth) | OAuth2 / OIDC 両対応 | 対応(OIDCのみ) | 対応済み（3.0.2（2025-01-09）以降） | Redmine の Self-registration に依存する挙動（設定次第で登録/リンク） | 多プロバイダ対応、ドキュメント豊富 |

## まとめ

今回の整理の結果、Redmine SSOを試すなら、OICD認証に対応したものを選ぶのがよいとおもいます（ enricohuang/redmine_oidcか、kontron/redmine_oauth ）
Contargoは、Redmine 6に対応していないっぽいのが残念、というところですね。

kontron/redmine_oauth は、OAuth2 / OIDC両方対応であり、個別の機能としては一番充実しているようです。一方、そのせいで実装が複雑な印象をうけます。上でも書きましたが、OAuth2は認可フレームワークで、そのままではログインには使えないはず。

## 気づき・今後について

本記事執筆のうえで、私自身、SAML、OAuth2、OIDCや各プラグインの実装方式について調査してかなり理解が進みました。

本記事は、あくまで机上での調査によるもので、実際に使って書いたものではありません。

OIDCについては、比較的簡単に対応できそうなので、実際にやってみたらまたご紹介しようとおもいます。

これからRedmineのSSO認証を検討する方の一助になれば幸いです。
