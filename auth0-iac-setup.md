# Auth0 構成管理運用ガイド

本プロジェクトでは **Auth0 Deploy CLI** を導入し、本番・テスト・ローカル環境のAuth0設定をコード（IaC）で管理します。
これにより、環境ごとの設定差異（Configuration Drift）を防ぎ、開発者がコマンド一つで自身の検証用テナントを構築・リセットできる環境を提供します。

## 1. ディレクトリ構成

リポジトリのルートに以下の構成を作成します。

```text
auth0-iac/
├── tenants/
│   ├── tenant.yaml           # 【マスター】全環境共通の構成定義
│   └── reset.yaml            # 【初期化用】テナントを空にするための定義
├── configs/
│   ├── prod.json             # 本番環境用マッピング定義（CI/CDで利用）
│   ├── test.json             # テスト環境用マッピング定義
│   └── local.json.example    # ローカル環境用テンプレート（開発者がコピーして利用）
├── .gitignore                # secretsを含むjsonを除外
└── package.json              # 実行スクリプト
```

## 2. セットアップ手順

### 前提条件
* Node.js がインストールされていること

### 初回インストール
プロジェクト直下で以下を実行します。

```bash
npm install auth0-deploy-cli --save-dev
```

### Auth0側の準備（各環境共通）
対象となるAuth0テナント（本番・テスト・個人のローカル用）それぞれで、デプロイ実行用のアプリケーションを作成する必要があります。

1.  **Auth0 Dashboard > Applications > Create Application**
    * **Name:** `Auth0 Deploy CLI` (任意)
    * **Type:** Machine to Machine Applications
2.  **APIの選択:** `Auth0 Management API` を選択
3.  **Permissions (Scopes):** `All` を選択（すべてのリソースを管理するため）
4.  作成後、`Settings` タブで **Client ID** と **Client Secret** を控えておく。

---

## 3. 設定ファイルの記述

### A. マスター定義ファイル (`tenants/tenant.yaml`)
環境ごとに異なる値（URLやSecret）は `##KEY##` の形式で変数化します。

```yaml
rules:
  - name: "Add-Role-To-Token"
    script: "./rules/add-role.js"
    enabled: true

clients:
  - name: "My App"
    app_type: regular_web
    callbacks:
      - "##APP_CALLBACK_URL##"  # 環境変数
    allowed_logout_urls:
      - "##APP_LOGOUT_URL##"    # 環境変数
```

### B. 環境ごとの設定ファイル (`configs/*.json`)

**開発者用テンプレート (`configs/local.json.example`)**
開発者はこれを `local.json` にリネームして、自分のテナント情報を記述します。

```json
{
  "AUTH0_DOMAIN": "dev-user.jp.auth0.com",
  "AUTH0_CLIENT_ID": "YOUR_DEPLOY_APP_CLIENT_ID",
  "AUTH0_CLIENT_SECRET": "YOUR_DEPLOY_APP_CLIENT_SECRET",
  "AUTH0_ALLOW_DELETE": true,
  "AUTH0_KEYWORD_REPLACE_MAPPINGS": {
    "##APP_CALLBACK_URL##": "http://localhost:3000/callback",
    "##APP_LOGOUT_URL##": "http://localhost:3000"
  },
  "AUTH0_EXCLUDED_CLIENTS": [
    "YOUR_DEPLOY_APP_CLIENT_ID" 
  ],
  "EXCLUDED_PROPS": {
    "clients": [ "Auth0 Deploy CLI" ],
    "connections": [ "Username-Password-Authentication" ]
  }
}
```

* `AUTH0_ALLOW_DELETE: true`: 定義ファイル（yaml）に存在しないリソースを削除します。
* `AUTH0_EXCLUDED_CLIENTS` / `EXCLUDED_PROPS`:
    * **重要:** デプロイ実行に使っているクライアント（自分自身）を除外しないと、実行中に削除されてエラーになります。
    * ログイン確認用のDB接続などを残したい場合も `connections` に記述します。

### C. 初期化用ファイル (`tenants/reset.yaml`)
テナントをリセットするための最小限の定義です。中身を空に近い状態にすることで、インポート時に既存リソースを削除させます。

```yaml
tenant:
  enabled_locales:
    - en
    - ja
  flags: {}
# 他のリソース（rules, clients等）を書かないことで、
# AUTH0_ALLOW_DELETE: true と組み合わせた時に全削除を実行させる
```

---

## 4. 実行コマンド（package.json）

`package.json` に以下のスクリプトを定義します。

```json
{
  "scripts": {
    "auth0:deploy:prod": "a0deploy import -c configs/prod.json -i tenants/tenant.yaml",
    "auth0:deploy:test": "a0deploy import -c configs/test.json -i tenants/tenant.yaml",
    "auth0:deploy:local": "a0deploy import -c configs/local.json -i tenants/tenant.yaml",
    "auth0:reset:local": "a0deploy import -c configs/local.json -i tenants/reset.yaml"
  }
}
```

## 5. 開発者の運用フロー

### ローカル環境の構築（初回・変更時）
リポジトリの最新設定を自分のテナントに反映します。

```bash
# configs/local.json の設定に従い、自分のテナントを構築
npm run auth0:deploy:local
```

### ローカル環境の初期化（リセット）
設定をいじりすぎて壊れた場合や、まっさらな状態からテストしたい場合に実行します。
テナントの設定がほぼ全て削除され、初期状態に戻ります（`EXCLUDED_PROPS` で指定したものは残ります）。

```bash
# 1. 自分のテナントを更地にする
npm run auth0:reset:local

# 2. その後、再度構築コマンドを叩いて最新の状態にする
npm run auth0:deploy:local
```

## 6. 注意事項

1.  **機密情報の管理:**
    * `configs/local.json` や `configs/prod.json` には Client Secret が含まれるため、**絶対にGitにコミットしないでください**。
    * `.gitignore` に `configs/*.json` を追加し、`*.json.example` のみを管理します。
2.  **本番反映:**
    * 本番への適用（`auth0:deploy:prod`）は、GitHub Actions等のCI/CDパイプライン経由で行うことを推奨します。
