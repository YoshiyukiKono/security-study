# OIDCClient と JWKS

OIDCClient と JWKS は、認証・認可の世界、特に **OpenID Connect (OIDC)** や **JSON Web Token (JWT)** を理解する上で重要な要素です。

Rancher や Kubernetes、GitHub Actions、AIサービスなどでも頻繁に登場します。

---

## 全体像

```
ユーザー
   │
   ▼
OIDC Client
   │
   │ 「ログインしてください」
   ▼
Identity Provider
(Keycloak, Microsoft Entra ID, Google...)
   │
   │ 署名付きJWT(ID Token)
   ▼
OIDC Client
   │
   │ JWKSで署名を確認
   ▼
「このトークンは本物」
```

---

# OIDC Clientとは

OIDC Clientとは

> **OpenID Connectを利用してログインするアプリケーション**

です。

例えば

* Rancher
* Grafana
* Argo CD
* 自作Webアプリ

などがOIDC Clientになります。

つまり

> 「私は認証そのものはやらないので、認証サーバーに任せます」

という立場です。

---

例えば Rancherなら

```
ユーザー
    ↓
Rancher
    ↓
Keycloakへリダイレクト
```

Keycloakでログインすると

```
ID Token(JWT)
```

が返ってきます。

RancherはそのJWTを信用してログインさせます。

---

# Identity Provider(IdP)

OIDC Clientが信頼する認証サーバーです。

代表例

* Keycloak
* Microsoft Entra ID
* Google の認証
* Okta

---

# JWTとは

ログインすると

```
eyJhbGciOiJSUzI1Ni...
```

のような長い文字列が返ります。

これは

```
Header
Payload
Signature
```

の3つで構成されています。

Payloadには例えば

```
ユーザー名

メール

有効期限

Role
```

などが入っています。

しかし

> Payloadだけでは改ざん可能

です。

そこで

```
Signature
```

が付きます。

---

# ではJWKSとは？

JWKSは

**JSON Web Key Set**

の略です。

つまり

> JWTを検証するための公開鍵一覧

です。

例えば

```
https://id.example.com/.well-known/jwks.json
```

を開くと

```json
{
  "keys": [
    {
      "kid":"123",
      "kty":"RSA",
      "n":"...",
      "e":"AQAB"
    }
  ]
}
```

のようなJSONが返ります。

---

## 何のため？

JWTには

```
Signature
```

があります。

これは

```
秘密鍵
```

で署名されています。

一方、

```
公開鍵
```

で

> 本当にこの秘密鍵で署名されたか？

を確認できます。

その公開鍵が

```
JWKS
```

です。

---

## イメージ

```
JWT

「私はKeycloakが発行しました」

        │

        ▼

OIDC Client

「本当？」

        │

        ▼

JWKSを見る

公開鍵で署名検証

        │

        ▼

OK
```

---

# 署名の流れ

```
Keycloak

秘密鍵
   │
   ▼
JWTに署名

-------------------

Rancher

JWT受信

↓

JWKS取得

↓

公開鍵で検証

↓

ログイン成功
```

---

# なぜ公開鍵を配る？

秘密鍵は絶対に外へ出せません。

```
秘密鍵
```

↓

署名専用

```
公開鍵
```

↓

誰でも取得可能

だから

```
秘密鍵は守られる
```

一方

```
誰でも検証できる
```

という仕組みになります。

---

# kidとは？

JWTのHeaderを見ると

```json
{
  "alg":"RS256",
  "kid":"abc123"
}
```

があります。

これは

```
どの公開鍵を使えばよいか
```

を表します。

JWKSには

```
kid=abc123
```

の公開鍵があります。

OIDC Clientは

```
kid一致

↓

その公開鍵で検証
```

を行います。

---

# Rancherでも使われる

例えば RancherをOIDC接続すると

```
Rancher
    │
    ▼
Keycloak

↓

JWT取得

↓

JWKS取得

↓

署名検証

↓

Role Mapping

↓

ログイン
```

という流れになります。

---

## まとめ

* **OIDC Client**：認証を外部のIdPに任せるアプリケーション（例：Rancher、Grafana、Argo CD）
* **JWT**：ログイン後に発行される、ユーザー情報を含む署名付きトークン
* **JWKS**：JWTの署名を検証するための**公開鍵の一覧**
* **秘密鍵**はIdPだけが保持し、**公開鍵**はJWKSとして公開されるため、安全にトークンの真正性を検証できる

Rancher や Kubernetes の文脈では、「OIDC Client が IdP から受け取った JWT を、JWKS を使って検証してからユーザーを認証する」という理解をしておくと、OIDC連携の設定やトラブルシュートがしやすくなります。
