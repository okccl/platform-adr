# Bootstrap 設計書

---

## 1. 概要

プラットフォームの bootstrap は、コンポーネント間の依存関係により単純な一斉 apply では正しく起動しない。単一の ArgoCD App-of-Apps と `sync-wave` アノテーションによる起動順序制御、およびカスタムヘルスチェックによる wave 進行の保証を組み合わせて、再現性のある bootstrap を実現する。Makefile は ArgoCD が管理できない操作（クラスタ作成・CNI インストール・CoreDNS 設定・ArgoCD 自身のインストール）のみを担う。

---

## 2. Bootstrap フロー

`make bootstrap` が実行するステップ。

| ステップ | Make ターゲット | 内容 |
|---|---|---|
| 1 | `cluster-create` | k3d クラスタ作成 |
| 2 | `install-cilium` | Cilium インストール（ArgoCD 起動前に CNI が必要） |
| 3 | `fix-coredns` | CoreDNS 設定（後述） |
| 4 | `bootstrap-argocd` | cert-manager/argocd namespace 作成・CA 投入・ArgoCD インストール・認証情報投入 |
| 5 | `bootstrap-sync` | ルート App apply → Keycloak 起動まで待機 |
| 6 | `bootstrap-apps` | user-apps-infra（Namespace・AppProject）の稼働確認後に apps-root App apply |

`bootstrap-sync` の詳細：

```
1. root App を apply（kubectl apply -f root.yaml）
2. kubectl annotate で即時リフレッシュをトリガー（auto-sync 待機を回避）
3. Envoy Gateway Pod が Ready になるまで待機
   （argocd.platform.local アクセスに必要）
4. argocd.platform.local 経由でログイン
5. wave-5 ArgoCD 自己管理 sync の完了を待機（Application health status を kubectl で直接監視）
6. [マイルストーン①] ArgoCD 起動完了を表示（URL・admin パスワード）
7. keycloak pod が Ready になるまで待機（kubectl wait ループ）
8. [マイルストーン②] bootstrap 完了を表示（ArgoCD URL・Keycloak URL）
```

### 2.1 CoreDNS 設定

`make fix-coredns`（ステップ 3）で以下の 2 つを CoreDNS ConfigMap に書き込む。bootstrap 前の時点で実行するため、Envoy Gateway SVC が存在しない状態でも事前に書き込み可能。

**NodeHosts — `host.k3d.internal` の登録**

MinIO コンテナ（`minio-external`）への疎通のため、k3d ネットワークのゲートウェイ IP を `host.k3d.internal` として登録する。

**Corefile — `*.platform.local` のリライトルール**

クラスタ内 Pod が `keycloak.platform.local` などを解決できるよう、Envoy Gateway の Service 名へリライトする。

```
rewrite name keycloak.platform.local  envoy-eg.envoy-gateway-system.svc.cluster.local
rewrite name argocd.platform.local    envoy-eg.envoy-gateway-system.svc.cluster.local
rewrite name backstage.platform.local envoy-eg.envoy-gateway-system.svc.cluster.local
```

Service 名は `platform/gateway/config/envoy-proxy-config.yaml`（EnvoyProxy リソース）で `envoy-eg` に固定しているため、bootstrap 前の時点で確定した名前を記述できる。

---

## 3. ルート App 構成

### 3.1 Application 定義

`platform-gitops/platform/group-roots/root.yaml` に単一の App-of-Apps を定義する。

- ソース: `platform/applications/` 以下を再帰的に読み込み
- 自動 Sync: prune・selfHeal 有効
- wave 失敗時のリトライ: 最大 5 回、10s 起点で指数バックオフ（上限 60s）

### 3.2 ディレクトリ構造

```
platform/applications/   # 全 Application を直下にフラット配置
```

---

## 4. Wave 設計

グループ間の wave 番号は 10 刻みで分離し、グループをまたぐ依存関係が wave 制御で確実に解消されるようにしている。

### 4.1 Wave 0–5（ネットワーク基盤）

| Wave | Application | 依存根拠 |
|---|---|---|
| 0 | storage | 依存なし（local-path StorageClass） |
| 1 | cert-manager | 依存なし |
| 1 | cilium | 依存なし |
| 1 | envoy-gateway-crds | 依存なし（CRD 先行インストール） |
| 1 | prometheus-operator-crds | 依存なし（CRD 先行インストール。cert-manager 等の早期 wave App が ServiceMonitor を作成できるようにする） |
| 2 | envoy-gateway | envoy-gateway-crds（CRD 依存） |
| 3 | cert-manager-config | cert-manager webhook（CA Issuer・Certificate） |
| 4 | gateway-config | envoy-gateway（Gateway/EnvoyProxy）+ cert-manager-config（TLS 証明書） |
| 5 | argocd | 自己管理 Helm sync により全コンポーネント再起動。完了まで wave 10 に進まない |

### 4.2 Wave 10–16（認証・シークレット基盤）

| Wave | Application | 依存根拠 |
|---|---|---|
| 10 | external-secrets | 依存なし |
| 10 | kube-prometheus-stack | 依存なし |
| 10 | kyverno | 依存なし |
| 11 | cnpg | 依存なし（webhook は Wave 10 完了後に有効化） |
| 11 | platform-secrets-sources | external-secrets（ESO CRD + ClusterSecretStore） |
| 12 | kyverno-policies | kyverno webhook — ESO webhook endpoint 登録待ちのバッファ |
| 13 | external-secrets-config | platform-secrets-sources（ClusterSecretStore）— config/generators/namespaces の multi-source で全 ExternalSecret を一括管理 |
| 14 | keycloak-db | cnpg webhook + external-secrets-config（wave 13 で DB Secret 生成済み） |
| 15 | keycloak | keycloak-db（DB 接続先） |
| 16 | keycloak-config-cli | keycloak（設定投入先） |
| 16 | keycloak-routes | keycloak（HTTPRoute のバックエンド） |

### 4.3 Wave 20–23（その他プラットフォームコンポーネント）

| Wave | Application | 依存根拠 |
|---|---|---|
| 20 | alloy | 依存なし（loki/tempo は起動後に接続） |
| 20 | argo-rollouts | 依存なし |
| 20 | crossplane | 依存なし |
| 20 | gateway-routes | 依存なし（バックエンドは前グループで起動済み） |
| 20 | goldilocks | 依存なし |
| 20 | keda | 依存なし |
| 20 | loki | 依存なし |
| 20 | tempo | 依存なし |
| 20 | vpa | 依存なし |
| 21 | backstage-db | cnpg webhook + external-secrets-config（wave 12 で DB Secret 生成済み） |
| 21 | crossplane-config | crossplane |
| 21 | grafana-dashboards | kube-prometheus-stack（ConfigMap 自動検出） |
| 21 | platform-alerts | kube-prometheus-stack（PrometheusRule） |
| 21 | user-apps-infra | 依存なし（sample-app Namespace + sample-apps AppProject）→完了後にbootstrap-apps開始 |
| 22 | backstage | backstage-db（DB 接続先） |
| 23 | trivy-operator | 依存なし（スキャン開始を他コンポーネント起動後に後回し） |

---

## 5. カスタムヘルスチェック

`platform/argocd/values.yaml` に定義。wave の進行条件として ArgoCD が使用する。

| リソース種別 | 保証する内容 |
|---|---|
| `argoproj.io/Application` | 新規作成直後の Application は `status.health.status` が nil のため、ArgoCD はデフォルトで Unknown と判定して wave をブロックしない。このチェックにより nil/空 status を Progressing として扱い、Application が真に Healthy になるまで次の wave に進まないことを保証する |
| `gateway.networking.k8s.io/Gateway` | Envoy proxy が実際に起動してトラフィックを処理できる状態か（`Programmed: True`）。wave 4 完了判定に使用。 |
| `admissionregistration.k8s.io/ValidatingWebhookConfiguration` | webhook CA bundle の注入完了。cert-manager・CNPG・ESO・Kyverno の webhook が呼び出し可能な状態になるまで次の wave に進まない |
| `external-secrets.io/ClusterSecretStore` | ESO が外部プロバイダー（kubernetes-store）へ接続できているか |
| `external-secrets.io/ExternalSecret` | ESO が Secret を正常に生成できているか |
| `postgresql.cnpg.io/Cluster` | CNPG Cluster が `Cluster in healthy state` に達したか。ヘルスチェック未定義の場合 ArgoCD はリソース作成直後に Healthy と判定するため、DB が未起動のまま依存コンポーネント（keycloak・backstage）の wave に進んでしまう |
| `apiextensions.k8s.io/CustomResourceDefinition` | CRD が `Established=True` になるまで Progressing として扱う。組み込みチェックは `NamesAccepted=False` を Degraded と判定するが、apply 直後の一時的な状態で wave を止めないよう Progressing に変更している |
